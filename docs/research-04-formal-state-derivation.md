# 形式化推导：工具状态空间与转换的自动化生成方法综述

> 研究日期：2026-04-27
> 项目背景：agent-cognitive-scaffolding — AI Agent 结构化决策框架
> 研究目的：探索从源代码、配置模式和 API 规范中自动推导工具状态空间的形式化方法

---

## 摘要

当前，我们的 Agent Cognitive Scaffolding 框架中，Skills 的状态空间（Phase 0 诊断）和 Playbook（操作剧本）依赖人工经验和事故复盘来枚举。这种方式存在固有的不完备性——只有经历过的故障模式才会被纳入。本文综述形式化方法领域中与"状态空间自动推导"相关的研究成果，探讨如何将模型检验、静态分析、规约挖掘、行为建模、契约式设计、LLM 辅助形式化以及声明式配置分析等技术应用于 CLI 工具（如 OpenClaw）的状态空间自动生成，最终提出一个可落地的工具链设计方案。

---

## 一、模型检验与状态空间探索

### 1.1 经典工具概述

**SPIN**（Simple Promela Interpreter）是 Gerard Holzmann 于 1997 年开发的显式状态模型检验器 [1]。它使用 Promela 建模语言描述并发系统，通过穷举搜索验证 LTL（线性时序逻辑）性质。SPIN 的核心能力是枚举系统所有可达状态并检测死锁、活锁和断言违反。

**TLA+** 由 Leslie Lamport 设计，是一种基于时序逻辑的形式化规约语言 [2]。其配套工具 TLC 模型检验器可以穷举有限状态空间，而 TLAPS（TLA+ Proof System）支持定理证明。TLA+ 的独特优势在于它同时描述"系统做什么"和"系统在什么条件下是正确的"，非常适合描述分布式系统的状态转换。

**Alloy** 由 Daniel Jackson（MIT）开发，是一种轻量级形式化建模语言 [3]。Alloy 使用关系逻辑描述结构和行为约束，其 Alloy Analyzer 通过 SAT 求解器在有界范围内搜索反例。Alloy 的"小范围假设"（small scope hypothesis）使其适合快速探索设计空间。

**NuSMV/nuXmv** 是符号模型检验器，使用 BDD（二元决策图）和 SAT/SMT 求解器验证 CTL/LTL 时序性质 [4]。相比显式状态检验，符号方法可以处理更大的状态空间。

### 1.2 对状态空间推导的适用性

这些工具的共同模式是：**人工编写模型 → 工具穷举状态 → 验证性质**。对于我们的需求，关键问题在于"模型从哪里来"。如果我们能从 CLI 工具的源代码或规范中自动提取形式化模型，则可以利用这些工具自动枚举所有可能的状态和转换。

**局限性**：
- **状态爆炸问题**：实际软件的状态空间通常是无限的或天文数字级别的。模型检验需要适当的抽象。
- **建模鸿沟**：从实际代码到形式化模型的转换需要大量的人工抽象决策。
- **环境建模困难**：CLI 工具与文件系统、网络、进程管理器等外部环境的交互难以完全建模。

### 1.3 对 CLI 工具状态机推导的潜在应用

对于 OpenClaw 这样的 Node.js CLI 工具，一种可行路径是：
1. 将 CLI 的主要操作（install, start, stop, update）建模为 Promela/TLA+ 中的动作
2. 将文件系统状态（配置文件是否存在、版本号）建模为状态变量
3. 将进程状态（是否运行、端口是否占用）建模为环境约束
4. 使用模型检验器枚举所有可达的状态组合

这等价于我们手工编写的 Phase 0 状态诊断矩阵的形式化版本。

---

## 二、静态分析与状态机提取

### 2.1 从源代码推断行为模型

**SLAM 项目**（Microsoft Research）是将静态分析用于状态机提取的里程碑工作 [5]。SLAM 对 Windows 设备驱动程序进行谓词抽象（predicate abstraction），自动将 C 代码转换为布尔程序（Boolean program），然后使用模型检验验证 API 使用规范。其核心思想是：程序的控制流 + 关键谓词的取值 = 抽象状态机。

**BLAST**（Berkeley Lazy Abstraction Software Verification Tool）在 SLAM 基础上引入了"惰性抽象"（lazy abstraction）技术 [6]，按需细化抽象精度，避免不必要的状态空间膨胀。

**Java PathFinder（JPF）** 是 NASA 开发的 Java 程序模型检验器 [7]。JPF 直接在 JVM 字节码级别执行系统化状态空间搜索，结合符号执行（symbolic execution）可以发现程序中的所有可达路径。

**Facebook Infer** 是一个基于分离逻辑（separation logic）的静态分析工具 [8]，能够分析内存安全和并发问题。其核心技术——双向抽象解释（bi-abduction）——可以自动推断函数的前置/后置条件。

### 2.2 对 TypeScript/Node.js CLI 的适用性

上述工具主要针对 Java/C/C++，对于 TypeScript/Node.js 生态的适用性需要转换：

- **类型系统利用**：TypeScript 的类型定义（特别是联合类型、字面量类型）天然编码了部分状态空间。例如 `type ServiceState = 'running' | 'stopped' | 'error'` 直接定义了服务状态的值域。
- **控制流分析**：TypeScript 编译器本身提供了控制流分析（Control Flow Analysis），可以追踪变量在不同分支中的类型收窄。
- **AST 分析**：通过分析 CLI 工具的命令解析代码（如 commander.js、yargs 的调用），可以自动提取所有子命令及其参数组合。
- **异步状态机**：Node.js 的 async/await 模式本质上是状态机，可以通过分析 Promise 链推断操作序列。

### 2.3 实际可行的提取策略

对于 OpenClaw 这样的项目，一个务实的静态分析方案是：
1. 解析 `package.json` 中的 scripts 和 bin 定义，确定入口点
2. 分析命令行参数解析代码，枚举所有命令和选项
3. 追踪文件 I/O 操作，识别所有被读取/写入的配置文件和状态文件
4. 分析 `process.exit()` 调用及其条件，推断错误状态
5. 追踪外部进程调用（child_process），识别依赖的外部服务

---

## 三、API 规约挖掘

### 3.1 从规范文档推导状态机

**规约挖掘**（Specification Mining）是从程序执行或文档中自动推断正式规约的研究领域。

**Daikon** 是动态不变量检测的先驱工具 [9]。它通过观察程序运行时的变量值，使用统计方法推断可能的不变量（如 `x > 0`、`array.length == n`）。这些不变量可以视为对系统状态空间的约束——违反不变量的状态即为异常状态。

**L* 算法**（Angluin, 1987）是主动自动机学习的基础算法 [10]。它通过向"教师"（oracle）提出成员资格查询（membership query）和等价性查询（equivalence query），逐步构造出目标系统的最小确定有限自动机（DFA）。现代实现如 **LearnLib** [11] 和 **AALpy** [12] 将其扩展到了 Mealy 机、Moore 机和带寄存器的自动机。

**JFLAP** 是用于自动机和形式语言教学的交互式工具 [13]，虽然主要用于教育目的，但其可视化能力对理解推导出的状态机很有价值。

### 3.2 从 CLI Help 文本和 Man Pages 提取

CLI 工具的帮助文本是一种半结构化的规约。从中可以提取：
- **命令树结构**：`openclaw install`、`openclaw start`、`openclaw status` 等
- **参数约束**：`--port <number>`、`--config <path>` 等
- **前置条件暗示**：帮助文本中的 "requires..." "must first..." 等措辞
- **状态依赖描述**：如 "only available when service is running"

### 3.3 从 OpenAPI/Swagger 规范提取

如果工具暴露了 REST API（OpenClaw 作为网关确实有 API），则 OpenAPI 规范提供了更丰富的状态信息：
- **HTTP 状态码定义**：200、404、503 等对应不同的系统状态
- **请求/响应 Schema**：定义了数据的结构约束
- **API 操作的先后依赖**：某些 API 只在特定条件下可调用

**Deep Specification Mining**（Le & Lo, 2018）[14] 使用深度学习从 API 调用序列中学习时序规约，是将传统规约挖掘与现代 ML 结合的代表工作。

---

## 四、基于执行轨迹的行为建模

### 4.1 过程挖掘（Process Mining）

过程挖掘从事件日志中发现、监控和改进实际过程。核心工具包括：

**ProM Framework** 是过程挖掘领域最著名的开源平台 [15]，实现了 Alpha 算法、Heuristic Miner、Inductive Miner 等多种过程发现算法。这些算法从日志中的事件序列自动推断出 Petri 网或 BPMN 模型。

**Celonis** 是商业化过程挖掘平台，虽然主要面向业务流程，但其技术原理同样适用于软件操作流程的建模。

### 4.2 从轨迹学习自动机

**LearnLib** [11] 提供了完整的主动和被动自动机学习算法实现：
- **主动学习**（L*、TTT、DHC）：通过与系统交互（执行命令、观察输出）构建模型
- **被动学习**（RPNI、Blue-Fringe）：仅从已有的执行轨迹集合推断自动机

**RPNI 算法**（Regular Positive and Negative Inference）[16] 从正例和反例轨迹集合中推断最小一致的 DFA。对于我们的场景，"正例"是成功的操作序列，"反例"是导致故障的操作序列。

**Process Mining Meets Model Learning**（Agostinelli et al., 2023）[17] 将过程挖掘与自动机学习结合，从事件日志中推断确定性有限自动机（DFA），这直接对应了我们需要的操作状态机。

### 4.3 对 Agent 操作日志的应用

在我们的框架中，每次 Agent 执行 Skill 时都会产生执行轨迹：
```
[命令序列]: openclaw status → openclaw stop → rm -rf node_modules → openclaw install → openclaw start
[观察序列]: port_occupied → process_killed → files_removed → install_success → start_success
[状态序列]: error_state → stopping → clean → installing → running
```

通过收集大量这样的轨迹（包括成功和失败的），我们可以：
1. 使用 RPNI 或 Inductive Miner 推断操作状态机
2. 发现人工枚举中遗漏的状态转换路径
3. 识别高频故障模式和恢复路径
4. 自动生成 Playbook 的操作序列

---

## 五、契约式设计与假设-保证推理

### 5.1 接口自动机（Interface Automata）

接口自动机（de Alfaro & Henzinger, 2001）[18] 是一种组合式建模形式化方法，特别适合描述组件之间的交互协议。其核心思想是：每个组件有"输入动作"（环境提供的）和"输出动作"（组件产生的），组合时自动计算兼容的交互。

**假设-保证推理**（Assume-Guarantee Reasoning）[19] 将系统验证分解为组件级别：
- **假设（Assume）**：对环境行为的约束——"假设环境满足条件 A"
- **保证（Guarantee）**：组件承诺的行为——"则组件保证行为 G"

### 5.2 对 OpenClaw 分层架构的应用

OpenClaw 的五层架构（Layer 0-4）天然适合契约式设计：

| 层次 | 假设（Assume） | 保证（Guarantee） |
|------|--------------|-----------------|
| Layer 0: OS 环境 | 操作系统为 macOS/Linux | 提供 Node.js 运行时、网络栈 |
| Layer 1: 依赖管理 | Node.js >= 18.x 可用 | npm packages 正确安装 |
| Layer 2: 配置 | 依赖已安装 | 配置文件语法正确、语义有效 |
| Layer 3: 进程管理 | 配置有效 | 进程正常启动、端口绑定成功 |
| Layer 4: 功能验证 | 进程运行中 | API 响应正确、路由生效 |

每一层的"假设"是其下层的"保证"，形成完整的证明链。当某一层的假设被违反时，就产生了我们需要诊断的故障。

### 5.3 形式化契约的自动推断

**Daikon** 的不变量检测可以自动推断组件级的前置/后置条件。结合函数签名和运行时监控，可以自动生成每一层的 Assume-Guarantee 对。

**Design-by-Contract**（Eiffel 语言的核心范式）[20] 虽然是编程范式而非分析技术，但其思想启发了我们：如果 CLI 工具的每个子命令都有显式的前置条件检查，那么这些检查本身就编码了状态空间约束。

---

## 六、LLM 辅助形式化规约生成

### 6.1 研究现状

**SpecGen**（Luo et al., 2024/2025）[21] 是一个使用 LLM 自动生成程序形式化规约的系统。它将 LLM 生成的候选规约与程序验证器（如 Dafny）结合，通过迭代反馈提升规约质量。实验表明 SpecGen 在标准基准上成功率显著高于纯 LLM 方法。

**From Informal to Formal**（2025）[22] 系统评估了 LLM 在将自然语言需求转换为形式化规约（如 LTL、CTL）方面的能力。结论是 LLM 对简单模式的转换效果良好，但对复杂时序性质的准确性仍有不足。

**Microsoft Research** 的工作 [23] 评估了 LLM 驱动的用户意图形式化（User-Intent Formalization），用于 Dafny 等验证感知语言。研究发现 LLM 可以合理地将非形式化的"这个函数应该..."描述转换为可验证的前置/后置条件。

**"Can Large Language Models Model Programs Formally?"**（ICLR 2025）[24] 研究了 LLM 在生成 TLA+ 和 Promela 规约方面的能力。结论是 LLM 可以为简单协议生成合理的形式化模型，但对复杂系统仍需人工指导和验证。

### 6.2 可靠性与验证需求

LLM 生成的形式化规约面临的核心挑战：
- **幻觉风险**：LLM 可能生成语法正确但语义错误的规约
- **不完备性**：生成的规约可能遗漏关键状态或转换
- **验证循环**：需要独立的验证机制确认规约的正确性

一种务实的方案是 **LLM + 模型检验的闭环**：
1. LLM 从文档/代码生成初始 TLA+/Promela 模型
2. 模型检验器验证性质，发现反例
3. 将反例反馈给 LLM，要求修正模型
4. 迭代至模型通过所有验证

### 6.3 对我们框架的意义

在 Agent Cognitive Scaffolding 的场景中，LLM 辅助形式化的直接应用是：
- 从 OpenClaw 的 README.md 和 --help 输出生成初始状态机模型
- 从错误日志和 Stack Overflow 讨论中提取故障模式
- 将人工编写的 SKILL.md 中的自然语言规则转换为可验证的形式化约束

---

## 七、声明式配置分析

### 7.1 声明式配置的形式化特性

声明式配置语言（Terraform HCL、Kubernetes YAML、Nix 表达式）描述"期望状态"而非"操作步骤"。这种范式天然接近形式化规约：

- **Terraform**：声明基础设施的目标状态，plan 阶段计算当前状态到目标状态的转换
- **Kubernetes**：声明工作负载的期望状态，控制器循环（reconciliation loop）自动驱动实际状态收敛
- **Nix**：纯函数式包管理，配置是确定性的、可复现的

### 7.2 已有的分析技术

**IaC 测试与验证**（2024）[25] 系统研究了基础设施即代码的自动化测试方法。包括：
- **类型检查**：HCL 的类型系统可以捕获配置中的类型错误
- **语义分析**：检测资源间的依赖冲突和循环引用
- **策略即代码**（Policy as Code）：如 Open Policy Agent（OPA）用 Rego 语言编写策略规则
- **变异测试**（Metamorphic Testing）[26]：通过系统性地变异配置来检测 IaC 引擎的行为正确性

**Snyk IaC** 等商业工具通过规则库检测配置中的安全隐患和最佳实践违反。

### 7.3 应用于 OpenClaw 配置分析

OpenClaw 的配置文件（如 YAML/JSON 格式的网关路由配置）可以进行：
- **Schema 验证**：确保配置结构正确
- **一致性检查**：检测端口冲突、路由重叠
- **完备性分析**：检测是否有必填字段缺失
- **状态可达性**：给定配置，哪些运行时状态是可能的？

将配置文件视为对状态空间的约束，结合 SMT 求解器（如 Z3），可以自动枚举所有满足配置约束的有效状态组合。

---

## 八、应用于我们的框架：具体工具链设计

### 8.1 设计目标

给定一个 CLI 工具（以 OpenClaw 为例），自动生成：
1. **Phase 0 状态诊断矩阵**：所有可能的系统状态及其检测方法
2. **Playbook 骨架集合**：从各种故障状态恢复到健康状态的操作序列
3. **Global Invariants**：系统应始终满足的不变量集合

### 8.2 输入源

| 输入源 | 信息类型 | 提取方法 |
|--------|---------|---------|
| 源代码（TypeScript） | 命令结构、错误处理、状态检查 | AST 分析 + 控制流分析 |
| CLI --help 输出 | 命令树、参数约束 | 结构化解析 |
| 配置文件 Schema | 有效配置空间 | JSON Schema / TypeScript 类型 |
| API 规范（OpenAPI） | 接口契约、状态码 | Schema 解析 |
| 执行日志/轨迹 | 实际操作序列和状态转换 | 过程挖掘 |
| 文档（README、Changelog） | 语义知识、已知问题 | LLM 提取 |
| 错误消息库 | 故障模式分类 | 模式匹配 + LLM 分类 |

### 8.3 工具链架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline 总览                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐              │
│  │ 源代码    │   │ CLI Spec │   │ 运行轨迹  │              │
│  │ Analysis │   │ Mining   │   │ Learning │              │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘              │
│       │               │               │                    │
│       ▼               ▼               ▼                    │
│  ┌─────────────────────────────────────────┐              │
│  │     统一中间表示（State Machine IR）       │              │
│  │   - States: {s1, s2, ..., sn}           │              │
│  │   - Transitions: {(si, action, sj)}     │              │
│  │   - Invariants: {φ1, φ2, ..., φm}      │              │
│  └────────────────────┬────────────────────┘              │
│                       │                                    │
│       ┌───────────────┼───────────────┐                   │
│       ▼               ▼               ▼                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐              │
│  │ Model    │   │ LLM      │   │ Template │              │
│  │ Checker  │   │ Enricher │   │ Generator│              │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘              │
│       │               │               │                    │
│       ▼               ▼               ▼                    │
│  ┌─────────────────────────────────────────┐              │
│  │         输出生成（Scaffolding Gen）        │              │
│  │   - Phase 0 诊断矩阵                     │              │
│  │   - Playbook 骨架                        │              │
│  │   - Global Invariants                   │              │
│  │   - SKILL.md 模板                        │              │
│  └─────────────────────────────────────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.4 各阶段详细设计

#### 阶段 1：源代码分析（Static Extraction）

```typescript
// 伪代码：从 TypeScript CLI 源码提取状态信息
interface ExtractedModel {
  commands: CommandSpec[];        // 所有 CLI 子命令
  stateVariables: StateVar[];    // 影响行为的状态变量
  errorPaths: ErrorPath[];       // 所有错误退出路径
  externalDeps: Dependency[];    // 外部依赖（进程、文件、网络）
  configSchema: JSONSchema;      // 配置文件结构
}
```

工具选择：
- **ts-morph**（TypeScript AST 操纵库）：解析源代码结构
- **TypeScript Compiler API**：利用类型信息推断状态值域
- **自定义 visitor**：追踪 `process.exit()`、`throw`、文件操作

#### 阶段 2：规约挖掘（Specification Mining）

从 CLI 的 `--help` 输出和文档中提取：
- 命令间的前置依赖（"must run install before start"）
- 参数的互斥约束（"--port and --socket are mutually exclusive"）
- 操作的幂等性标注（"safe to run multiple times"）

工具选择：
- **LLM（GPT-4/Claude）**：从非结构化文档提取结构化约束
- **正则表达式 + 启发式**：从 --help 格式化输出中解析

#### 阶段 3：轨迹学习（Behavioral Learning）

从 Agent 历史执行日志中学习：
```
输入轨迹集合:
  成功: [status→ok, start→ok]
  成功: [install→ok, config→ok, start→ok]
  失败: [start→port_in_use]
  恢复: [stop→ok, start→ok]

输出: 状态自动机 + 转换概率
```

工具选择：
- **AALpy**（Python 实现的自动机学习库）：RPNI/L* 算法
- **pm4py**（Python 过程挖掘库）：从日志发现过程模型
- **自定义日志解析器**：将 Agent 执行日志转换为事件序列

#### 阶段 4：模型验证与完善

将推断出的状态机转换为 TLA+ 规约，验证：
- **安全性**（Safety）：系统永远不会进入未定义状态
- **活性**（Liveness）：从任何故障状态都存在恢复路径
- **完备性**：所有可观察的状态都在模型中有对应

工具选择：
- **TLC 模型检验器**：验证 TLA+ 规约
- **Z3 SMT 求解器**：验证配置约束的可满足性
- **LLM 反馈循环**：修正模型检验发现的问题

#### 阶段 5：脚手架生成（Scaffolding Generation）

从验证通过的状态机模型自动生成：

```markdown
## Phase 0: 状态诊断（自动生成）

| 检查项 | 检测命令 | 期望结果 | 失败含义 |
|--------|---------|---------|---------|
| Node.js 可用 | node --version | >= 18.x | Layer 0 故障 |
| npm 包完整 | ls node_modules/.package-lock.json | 文件存在 | Layer 1 故障 |
| 配置有效 | openclaw validate | exit 0 | Layer 2 故障 |
| 进程运行 | pgrep -f openclaw | PID 存在 | Layer 3 故障 |
| 端口监听 | lsof -i :端口号 | 有输出 | Layer 3 故障 |
| API 响应 | curl localhost:端口/health | 200 OK | Layer 4 故障 |
```

### 8.5 预期效果与局限

**预期效果**：
- 状态空间覆盖率从人工枚举的约 60-70% 提升到 85-95%
- 新工具接入时间从"等待事故发生再补充"缩短到"分析源码即可生成初版"
- Playbook 完整性可验证——模型检验保证每个故障状态都有恢复路径

**已知局限**：
- 环境相关的状态（网络超时、磁盘空间）难以从源码推断，需要运维经验补充
- 自动生成的 Playbook 可能包含不实际的恢复路径（形式上可达但实际上风险高）
- LLM 辅助环节的可复现性和一致性仍需验证机制保障
- 初始建设成本较高，适合核心工具而非一次性脚本

---

## 九、结论与展望

形式化方法为 Agent Cognitive Scaffolding 的自动化生成提供了坚实的理论基础。通过结合静态分析（从代码推断结构）、动态学习（从轨迹推断行为）和 LLM 辅助（弥合非形式化到形式化的鸿沟），我们可以构建一个半自动化的工具链，显著提升状态空间的覆盖率和 Playbook 的完备性。

近期可落地的方向：
1. **短期（1-2 周）**：基于 TypeScript AST 分析提取命令结构和错误路径
2. **中期（1-2 月）**：集成 AALpy 从历史日志学习状态自动机
3. **长期（3-6 月）**：构建完整的 LLM + 模型检验闭环验证流程

这项工作的核心洞察是：**状态空间的推导不应该是一次性的人工活动，而应该是一个持续运行的自动化流水线**——随着工具的更新、新的故障模式被观察到、新的文档被发布，状态空间模型应当自动演化和完善。

---

## 参考文献

[1] Holzmann, G.J. "The Model Checker SPIN." IEEE Transactions on Software Engineering, 23(5):279-295, 1997.

[2] Lamport, L. "Specifying Systems: The TLA+ Language and Tools for Hardware and Software Engineers." Addison-Wesley, 2002.

[3] Jackson, D. "Software Abstractions: Logic, Language, and Analysis." MIT Press, 2006. (Alloy)

[4] Cimatti, A., Clarke, E., et al. "NuSMV: A New Symbolic Model Checker." International Journal on Software Tools for Technology Transfer, 2(4):410-425, 2000.

[5] Ball, T., Rajamani, S.K. "The SLAM Project: Debugging System Software via Static Analysis." POPL 2002.

[6] Henzinger, T.A., Jhala, R., et al. "Lazy Abstraction." POPL 2002. (BLAST)

[7] Visser, W., Havelund, K., et al. "Model Checking Programs." Automated Software Engineering, 10(2):203-232, 2003. (Java PathFinder)

[8] Calcagno, C., Distefano, D., et al. "Moving Fast with Software Verification." NFM 2015. (Facebook Infer)

[9] Ernst, M.D., Perkins, J.H., et al. "The Daikon System for Dynamic Detection of Likely Invariants." Science of Computer Programming, 69(1-3):35-45, 2007.

[10] Angluin, D. "Learning Regular Sets from Queries and Counterexamples." Information and Computation, 75(2):87-106, 1987. (L* Algorithm)

[11] Isberner, M., Howar, F., Steffen, B. "The TTT Algorithm: A Redundancy-Free Approach to Active Automata Learning." RV 2014. (LearnLib)

[12] Muskardin, E., Aichernig, B.K., et al. "AALpy: An Active Automata Learning Library." CAV 2021.

[13] Rodger, S.H., Finley, T.W. "JFLAP: An Interactive Formal Languages and Automata Package." Jones & Bartlett, 2006.

[14] Le, T.D.B., Lo, D. "Deep Specification Mining." ISSTA 2018.

[15] van der Aalst, W.M.P. "Process Mining: Data Science in Action." Springer, 2016. (ProM)

[16] Oncina, J., Garcia, P. "Inferring Regular Languages in Polynomial Updated Time." Pattern Recognition and Image Analysis, 1992. (RPNI)

[17] Agostinelli, S., et al. "Process Mining Meets Model Learning: Discovering Deterministic Finite Automata from Event Logs." Information Systems, 2023.

[18] de Alfaro, L., Henzinger, T.A. "Interface Automata." ESEC/FSE 2001.

[19] Chakrabarti, A., et al. "Assume-Guarantee Verification for Interface Automata." FASE 2008.

[20] Meyer, B. "Design by Contract." Advances in Object-Oriented Software Engineering, 1991.

[21] Luo, Y., et al. "SpecGen: Automated Generation of Formal Program Specifications via Large Language Models." ICSE 2025.

[22] "From Informal to Formal – Incorporating and Evaluating LLMs on Natural Language Requirements to Formal Specifications." ACL 2025.

[23] Microsoft Research. "Evaluating LLM-driven User-Intent Formalization for Verification-Aware Languages." 2024.

[24] "Can Large Language Models Model Programs Formally?" ICLR 2025.

[25] "Automated Infrastructure as Code Program Testing." IEEE TSE, 50(6), 2024.

[26] "Metamorphic Testing for Infrastructure-as-Code Engines." Programming Group, 2026.

---

> 本文作为 agent-cognitive-scaffolding 项目的研究文档，为后续工具链开发提供理论基础和技术路线参考。
