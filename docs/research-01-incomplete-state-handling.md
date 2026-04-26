# 研究报告：不完备状态处理机制

> AI Agent 在遭遇未知/未枚举状态时的检测、降级与兜底策略综述

**作者**：agent-cognitive-scaffolding 项目组  
**日期**：2026-04-27  
**状态**：初稿

---

## 摘要

在 agent-cognitive-scaffolding 项目中，我们设计了基于 Phase 0 状态诊断 → Playbook 查表的结构化决策框架。然而，现实世界的状态空间是开放的——Agent 必然会遭遇不在任何已知 Playbook 中的状态组合。本报告从六个学科视角综述"不完备状态处理"的理论基础与工程实践，并为框架的 `P_UNKNOWN` 兜底机制提出具体设计建议。

---

## 1. 开放世界识别与分布外检测（OOD Detection）

### 1.1 问题定义

传统分类器假设测试时输入的分布与训练时一致（closed-world assumption）。开放世界识别（Open Set Recognition, OSR）和分布外检测（Out-of-Distribution Detection, OOD Detection）研究的核心问题是：**如何判断一个输入不属于已知类别？**

这与我们的框架高度对应：Phase 0 诊断产出一个状态向量（CLI 状态、Gateway 状态、端口状态、版本状态、包状态、Domain 状态），然后需要判断该向量是否匹配已知的 8 个 Playbook。

### 1.2 主要技术路线

**（1）基于 Softmax 置信度的方法**

Hendrycks & Gimpel (2017) 提出了 Maximum Softmax Probability (MSP) 基线方法：将分类器输出的最大 softmax 概率作为置信度度量，低于阈值则判定为 OOD。该方法简单但存在神经网络对 OOD 样本过度自信的问题。

**（2）能量函数方法**

Liu et al. (2020) "Energy-based Out-of-Distribution Detection" 提出用能量分数（energy score）替代 softmax 概率。能量分数 $E(x) = -\log \sum_i e^{f_i(x)}$ 对 OOD 样本的分离效果优于 softmax。该方法无需重新训练模型，可直接应用于已有分类器。

**（3）基于特征空间距离的方法**

Lee et al. (2018) 提出 Mahalanobis Distance-based 方法：在特征空间中拟合每个类别的高斯分布，计算测试样本到最近类别中心的马氏距离。距离超过阈值则判定为 OOD。Sun et al. (2022) 的 KNN-based 方法进一步简化，使用 k-近邻距离直接判定。

**（4）基于相对角度的方法**

NeurIPS 2025 新近工作提出使用特征向量之间的相对角度作为 OOD 度量，在高维空间中角度比欧氏距离更稳定。

### 1.3 对我们框架的启示

| OOD 技术概念 | 映射到 Agent 框架 |
|---|---|
| Softmax 置信度 | 状态向量与每个 Playbook 条件的匹配度评分 |
| 能量分数 | 所有 Playbook 匹配度的综合衡量（越低 = 越不匹配任何已知模式） |
| 马氏距离 | 状态向量到"典型故障模式簇"的距离 |
| OOD 阈值 | 触发 P_UNKNOWN 的置信度门槛 |

**关键洞察**：我们不需要训练神经网络。Phase 0 的状态空间是结构化的、有限维度的。可以将每个 Playbook 表示为状态空间中的一个"区域"（通过其前置条件定义），然后用简单的规则匹配度量是否有样本落在所有已知区域之外。

---

## 2. 安全关键系统中的优雅降级

### 2.1 航空领域：飞行包线保护（Envelope Protection）

现代飞行控制系统（如 Airbus 的 Normal Law → Alternate Law → Direct Law 三级降级）是优雅降级的经典范例：

- **Normal Law**：全自动保护，计算机阻止飞行员做出可能超出包线的操作
- **Alternate Law**：部分传感器失效时，减少自动保护，给飞行员更多控制权
- **Direct Law**：多重故障时，完全取消自动保护，飞行员直接控制舵面

这一设计理念的核心是：**系统越不确定自身的状态理解是否正确，就越应该退让控制权给人类**。

### 2.2 核能领域：纵深防御（Defense in Depth）

IAEA 的纵深防御原则包含五个层次：

1. **预防**：设计阻止异常发生
2. **监控**：检测并修正偏差
3. **安全系统**：阻止事故升级
4. **事故管理**：限制后果
5. **场外响应**：减轻辐射后果

每一层都假设前一层可能失效。这是一种"不信任前一层完美"的设计哲学。

### 2.3 汽车领域：安全状态（Safe State）

ISO 26262 功能安全标准定义了"安全状态"概念：当系统无法确定正确行为时，必须进入一个已知安全的状态。例如：

- 自动驾驶遇到无法理解的场景 → 靠边停车（安全状态）
- ABS 控制器检测到传感器矛盾 → 退化为传统制动（降级但安全的状态）

### 2.4 共同设计模式

从安全关键系统中提取的设计模式：

| 模式 | 含义 | 对 Agent 框架的映射 |
|------|------|-------------------|
| **安全状态（Safe State）** | 系统能退回到的已知安全配置 | Agent 停止操作，保持现状，上报人类 |
| **看门狗定时器（Watchdog Timer）** | 超时无响应自动触发保护 | Playbook 执行超时自动中止 |
| **投票系统（Voting）** | 多个独立通道交叉验证 | 多维度状态诊断交叉确认 |
| **包线保护（Envelope Protection）** | 限制操作在安全范围内 | 禁止 Agent 在不确定状态下执行破坏性操作 |
| **渐进降级（Progressive Degradation）** | 逐步减少功能而非突然失效 | Normal → Conservative → Manual 三级模式 |

### 2.5 Nancy Leveson 的 Safety-III 思想

MIT 的 Nancy Leveson 在其 Safety-III 框架中强调：传统安全分析（故障树、事件树）假设组件失效是离散事件，但现代系统的事故往往源于**组件间的意外交互**——每个组件都在正常工作，但组合行为超出设计预期。

这正是我们面临的问题：Phase 0 的每个维度可能都不异常，但其组合可能是前所未见的。Safety-III 的应对策略是**持续监控系统级约束是否被违反**，而不是枚举所有可能的组件故障组合。

---

## 3. 异常处理层次结构

### 3.1 编程语言的异常处理模型

异常处理的核心设计决策是**粒度**和**传播方向**：

- **Java 检查异常**：强制调用者处理或向上传播，编译期保证
- **Go 的错误值**：显式返回错误，强制在每个调用点决策
- **Erlang 的 "let it crash"**：不在出错处处理，而是由 supervisor 树监控并重启

Erlang/OTP 的 supervisor 树模型与我们的框架最为相关：

```
Application Supervisor
├── Worker 1 (try a strategy)
├── Worker 2 (try another strategy)  
└── Escalation Supervisor (when all workers fail)
    └── Human notification
```

### 3.2 分布式系统的弹性模式

**（1）熔断器模式（Circuit Breaker）**

Michael Nygard 在 *Release It!* (2007) 中系统化的模式：

- **Closed**：正常请求通过
- **Open**：故障超过阈值，直接返回失败，不再尝试
- **Half-Open**：定期试探性发送请求，判断是否恢复

对 Agent 框架的映射：当某个修复策略连续失败 N 次，"熔断"该策略，不再尝试，转向其他策略或上报。我们现有的"两次失败必须换路"规则本质上就是一个 threshold=2 的熔断器。

**（2）舱壁模式（Bulkhead）**

将系统隔离为独立舱室，一个舱室的故障不影响其他舱室。对 Agent 框架：不同层次（CLI / Gateway / Port / Domain）的问题应该隔离处理，一个层次的诊断失败不应阻止其他层次的诊断。

**（3）后备模式（Fallback）**

当主路径失败时使用降级但可用的备选方案。对 Agent 框架：当最优 Playbook 不适用时，不是直接失败，而是退化到更保守但更安全的通用流程。

### 3.3 分层升级策略

综合以上模式，异常处理的最佳实践是建立清晰的升级层次：

```
Level 0: 自动重试（瞬时故障）
Level 1: 策略切换（同类替代方案）
Level 2: 降级执行（减少功能但保持核心目标）
Level 3: 安全停止（保持现状，不做进一步操作）
Level 4: 人类升级（上报并等待指令）
```

每一层都有明确的**进入条件**（什么情况下从上一层升级到本层）和**退出条件**（什么情况下可以降回上一层）。

---

## 4. 人类在环路中的兜底机制

### 4.1 何时升级给人类

研究表明（Parasuraman et al., 2000 "A model for types and levels of human interaction with automation"），自动化程度与人类信任之间存在非线性关系。自动化系统需要在两个极端之间取得平衡：

- **过度升级（Over-escalation）**→ 警报疲劳（Alert Fatigue）
- **不足升级（Under-escalation）**→ 遗漏关键问题

### 4.2 警报疲劳的研究

ACM 2024 的研究 "Towards Human-AI Teaming to Mitigate Alert Fatigue in Security Operations Centers" 指出：

- 安全运营中心（SOC）分析师每天面临数千条警报，误报率高达 90%+
- 导致分析师忽略真实警报的概率显著增加
- 解决方案包括：警报聚合、上下文富化、动态优先级调整

### 4.3 升级触发条件的设计原则

基于文献综述，有效的升级触发应满足以下条件：

| 维度 | 触发条件 | 具体到我们的框架 |
|------|---------|----------------|
| **置信度** | 状态匹配置信度低于阈值 | Phase 0 诊断结果不满足任何 Playbook 前置条件 |
| **影响范围** | 操作可能造成不可逆损害 | 涉及数据删除、配置覆写等破坏性操作 |
| **尝试耗尽** | 所有已知策略均已失败 | "两次失败换路" 规则触发后仍无可用策略 |
| **时间约束** | 在限定时间内未能解决 | 执行超过看门狗时间窗口 |
| **新颖性** | 遇到从未见过的状态组合 | 状态向量不在任何已知 Playbook 的前置条件范围内 |

### 4.4 升级信息的质量要求

仅仅"我不知道该怎么办"是不够的。有效的人类升级应包含：

1. **当前状态快照**：Phase 0 诊断的完整输出
2. **已尝试的策略及结果**：Agent 之前做了什么、为什么失败
3. **不确定性来源**：Agent 无法确定的具体方面
4. **建议的候选行动**：即使不确定，也提供带置信度的建议
5. **风险评估**：如果不处理，可能的后果和时间窗口

### 4.5 避免过度升级的机制

- **分级信息通道**：紧急升级（阻塞等待回复）vs. 异步通知（FYI，不阻塞继续）
- **升级冷却期**：同一类型的升级在 N 分钟内只发送一次
- **自适应阈值**：根据人类历史响应模式动态调整升级阈值——如果人类经常批准某类操作，逐渐降低该类操作的升级频率

---

## 5. 保形预测与不确定性量化

### 5.1 保形预测（Conformal Prediction）基础

保形预测（Vovk et al., 2005）是一种无分布假设的不确定性量化框架。其核心思想：

给定置信水平 $1-\alpha$，为每个测试样本输出一个**预测集合**（prediction set），保证真实标签以至少 $1-\alpha$ 的概率落入该集合。

关键性质：
- **覆盖率保证**：$P(Y \in C(X)) \geq 1-\alpha$，这是理论保证而非经验估计
- **无分布假设**：只需要数据的可交换性（exchangeability），不需要参数分布假设
- **与模型无关**：可以包裹任何底层预测模型

### 5.2 对 Agent 决策的映射

将保形预测应用到 Playbook 选择中：

- **校准集**：历史上遇到的（状态向量, 正确 Playbook）对
- **非一致性分数（Nonconformity Score）**：衡量新状态向量与各 Playbook 的"不一致程度"
- **预测集合**：对新状态，输出"可能适用的 Playbook 集合"

三种情况：
1. 预测集合 = {P3}：高置信度，直接执行 P3
2. 预测集合 = {P3, P5}：中等置信度，需要进一步诊断区分，或征询人类
3. 预测集合 = ∅（空集）或预测集合 = 全集：极低置信度，触发 P_UNKNOWN

### 5.3 决策论基础

Xu & Gao (2025) "Decision Theoretic Foundations for Conformal Prediction" 将保形预测与决策论结合：不同的误分类错误有不同的代价，预测集合的大小应该反映错误的代价。

对我们的框架：不同 Playbook 的误执行代价不同。例如：
- 误执行 P1（HEALTHY → 不操作）：代价低，最多浪费时间
- 误执行 P7（CORRUPTION → 完全重装）：代价高，可能丢失配置

因此，高代价 Playbook 需要更高的置信度阈值才能触发。

### 5.4 实用的置信度框架

不需要完整实现保形预测，可以借鉴其思想设计简化版本：

```
confidence_score = min(
    match_score(state, playbook.preconditions),  # 前置条件匹配度
    distinctiveness_score(state, other_playbooks) # 与其他 Playbook 的区分度
)

if confidence_score >= HIGH_THRESHOLD:
    execute(playbook)
elif confidence_score >= LOW_THRESHOLD:
    execute_with_extra_verification(playbook)
else:
    trigger_P_UNKNOWN()
```

---

## 6. 应用到我们的框架：P_UNKNOWN 设计

### 6.1 设计原则

综合以上五个领域的研究，P_UNKNOWN 应遵循以下原则：

1. **显式检测**：不是"剩余"（所有 Playbook 都不匹配时默认触发），而是有积极的检测逻辑
2. **安全优先**：在不确定时选择保守行动，而非冒险执行可能错误的 Playbook
3. **信息最大化**：在触发人类升级之前，收集尽可能多的诊断信息
4. **可进化**：P_UNKNOWN 处理的case应被记录，成为未来新 Playbook 的种子

### 6.2 P_UNKNOWN 的分层响应设计

```yaml
P_UNKNOWN:
  trigger_conditions:
    - no_playbook_match: "Phase 0 状态向量不满足任何 P1-P8 的前置条件"
    - ambiguous_match: "状态向量同时满足多个互斥 Playbook 的前置条件"
    - partial_diagnosis_failure: "Phase 0 某些维度的诊断命令本身失败"
  
  response_levels:
    L0_extended_diagnosis:
      description: "扩展诊断——收集更多信息"
      actions:
        - collect_system_logs: "收集最近 100 行相关日志"
        - check_related_services: "检查上下游依赖服务状态"
        - timestamp_correlation: "关联时间线——最近发生了什么变化？"
        - resource_check: "检查磁盘/内存/CPU 是否有资源瓶颈"
      exit_condition: "扩展诊断后重新执行 Phase 0，看是否能匹配已知 Playbook"
      timeout: "60 seconds"
    
    L1_conservative_action:
      description: "保守操作——只执行确定安全的步骤"
      precondition: "L0 未能解决状态模糊性"
      allowed_actions:
        - restart_service: "如果服务不在运行状态，尝试标准启动"
        - verify_config: "验证配置文件的完整性和语法正确性"
      forbidden_actions:
        - uninstall: "禁止卸载"
        - overwrite_config: "禁止覆写配置"
        - force_operations: "禁止任何 --force 参数的操作"
      exit_condition: "保守操作后重新验证——状态是否改善？"
      max_attempts: 1
    
    L2_safe_stop:
      description: "安全停止——保持当前状态不恶化"
      precondition: "L1 保守操作未改善状态，或无法确定任何安全操作"
      actions:
        - preserve_state: "记录当前完整状态快照"
        - no_further_changes: "不做任何修改"
        - prepare_escalation_report: "准备人类升级报告"
      
    L3_human_escalation:
      description: "人类升级——提供完整上下文等待指令"
      report_contents:
        - phase0_full_output: "完整的 Phase 0 诊断结果"
        - extended_diagnosis_results: "L0 扩展诊断收集的信息"
        - attempted_actions: "L1 中尝试的操作及结果"
        - state_diff: "与上次已知正常状态的差异"
        - hypothesis: "Agent 的最佳猜测（附带置信度）"
        - risk_assessment: "不处理的风险评估"
      escalation_format: |
        ⚠️ P_UNKNOWN 触发：遇到未知状态组合
        
        【当前状态】
        {phase0_output}
        
        【不匹配原因】
        已知 Playbook 均不匹配，具体原因：{mismatch_reasons}
        
        【已采集的额外信息】
        {extended_diagnosis}
        
        【已尝试的操作】
        {attempted_actions_with_results}
        
        【我的最佳猜测】
        最接近的 Playbook: {closest_playbook} (置信度: {confidence}%)
        不确定之处: {uncertainty_sources}
        
        【请求】
        请指示下一步操作，或确认是否可执行 {closest_playbook}
```

### 6.3 置信度评分机制

为每个 Playbook 定义一个结构化的匹配函数：

```yaml
playbook_matching:
  scoring_method: "weighted_condition_match"
  
  # Phase 0 各维度的权重（反映诊断重要性）
  dimension_weights:
    cli_status: 0.15
    gateway_status: 0.25
    port_status: 0.20
    version_status: 0.10
    package_status: 0.15
    domain_status: 0.15
  
  # 匹配度阈值
  thresholds:
    high_confidence: 0.90   # 直接执行
    medium_confidence: 0.70  # 执行但增加额外验证步骤
    low_confidence: 0.50     # 进入 P_UNKNOWN L0（扩展诊断）
    no_match: 0.50           # 低于此值认为不匹配
  
  # 破坏性操作的额外置信度要求
  destructive_action_penalty:
    P7_CORRUPTION: "+0.15"    # 需要 0.90 + 0.15 = 实质上需要完美匹配
    P6_REINSTALL: "+0.10"
```

### 6.4 状态记忆与 Playbook 进化

P_UNKNOWN 不应仅是一个静态的兜底机制，还应是框架进化的驱动力：

```yaml
evolution_mechanism:
  logging:
    - 每次 P_UNKNOWN 触发时记录完整状态向量
    - 记录人类最终的解决方案
    - 记录从触发到解决的完整操作序列
  
  pattern_detection:
    - 当同一状态模式触发 P_UNKNOWN 超过 N 次（建议 N=3）
    - 自动提议创建新的 Playbook
    - 候选 Playbook 基于历史解决方案的共性提取
  
  playbook_proposal_format: |
    📋 候选新 Playbook 提议
    
    触发次数: {count}
    共同状态模式: {common_state_pattern}
    历史解决方案: {solutions_summary}
    建议的 Playbook 名称: P9_{suggested_name}
    建议的操作流程: {suggested_steps}
    
    是否批准加入 Skill？[需要人类确认]
```

### 6.5 与现有框架的集成点

P_UNKNOWN 与现有 Skill 架构的集成：

```
Phase 0 诊断
    │
    ├─ 匹配 P1-P8 → 执行对应 Playbook
    │
    ├─ 多重匹配 → P_UNKNOWN (ambiguous_match)
    │   └─ 取最高置信度的 Playbook
    │       ├─ 置信度差 > 0.2 → 执行最高的（有明确优胜者）
    │       └─ 置信度差 < 0.2 → L0 扩展诊断
    │
    └─ 无匹配 → P_UNKNOWN (no_playbook_match)
        └─ L0 → L1 → L2 → L3 分层响应
```

### 6.6 全局不变量扩展

在现有 6 条 Global Invariants 基础上增加 P_UNKNOWN 相关不变量：

```yaml
additional_invariants:
  - INV-7: "NEVER execute a destructive action when confidence < HIGH_THRESHOLD"
  - INV-8: "ALWAYS log the full state vector when P_UNKNOWN triggers"
  - INV-9: "NEVER attempt more than 1 conservative action in L1 without re-diagnosis"
  - INV-10: "ALWAYS provide hypothesis and confidence when escalating to human"
```

---

## 7. 对比分析与理论综合

### 7.1 各领域方法的对比

| 领域 | 核心思想 | 适用层面 | 我们采用的方面 |
|------|---------|---------|--------------|
| OOD 检测 | 度量输入与已知分布的距离 | 检测层 | 置信度评分机制 |
| 安全关键系统 | 未知时退化而非冒进 | 响应层 | 分层降级设计 |
| 异常处理 | 分层升级与隔离 | 架构层 | L0-L3 层次结构 |
| HITL | 平衡升级频率与信息质量 | 交互层 | 升级报告格式 |
| 保形预测 | 校准化的不确定性表达 | 决策层 | 阈值与代价敏感匹配 |

### 7.2 方法论整合

这五个领域并非独立，而是形成一个完整的处理管道：

```
检测 → 评估 → 响应 → 升级 → 进化

OOD检测     保形预测    安全关键     HITL      异常处理
(识别异常) → (量化不确定) → (分层降级) → (人类介入) → (记录学习)
```

### 7.3 已知局限性

1. **状态空间的组合爆炸**：6 个维度，每个维度可能有多种取值，组合数很大。但实际中大部分组合在物理上不可能出现，有效状态空间远小于理论上界。

2. **阈值的冷启动问题**：初始阶段缺乏历史数据来校准置信度阈值。建议：开始时使用保守阈值（偏向多升级），随着数据积累逐步调优。

3. **"未知的未知"**：P_UNKNOWN 处理的是"已知的未知"（我们知道我们不知道）。真正的"未知的未知"（我们不知道我们不知道）可能表现为 Phase 0 本身的诊断命令失效——连状态都无法获取。这需要更底层的健壮性设计（如 L0 中的 `partial_diagnosis_failure` 处理）。

---

## 8. 参考文献

### 8.1 OOD 检测与开放世界识别

- Hendrycks, D., & Gimpel, K. (2017). "A Baseline for Detecting Misclassified and Out-of-Distribution Examples in Neural Networks." *ICLR 2017.*
- Liu, W., Wang, X., Owens, J., & Li, Y. (2020). "Energy-based Out-of-Distribution Detection." *NeurIPS 2020.*
- Lee, K., Lee, K., Lee, H., & Shin, J. (2018). "A Simple Unified Framework for Detecting Out-of-Distribution Samples and Adversarial Attacks." *NeurIPS 2018.*
- Sun, Y., Ming, Y., Zhu, X., & Li, Y. (2022). "Out-of-Distribution Detection with Deep Nearest Neighbors." *ICML 2022.*
- Scheirer, W. J., de Rezende Rocha, A., Sapkota, A., & Boult, T. E. (2013). "Toward Open Set Recognition." *IEEE TPAMI.*
- ACM Computing Surveys (2025). "Out-of-Distribution Detection: A Task-Oriented Survey of Recent Advances."

### 8.2 安全关键系统与优雅降级

- Leveson, N. G. (2020). "Safety III: A Systems Approach to Safety and Resilience." MIT.
- ISO 26262:2018. "Road vehicles — Functional safety."
- IAEA Safety Standards. "Defence in Depth in Nuclear Safety." INSAG-10.
- Traverse, P., Lacaze, I., & Souyris, J. (2004). "Airbus Fly-by-Wire: A Total Approach to Dependability." *IFIP International Conference on Dependable Systems.*
- Koopman, P. (1997). "Automatic Graceful Degradation for Distributed Embedded Systems." Carnegie Mellon University.
- EASA (2025). European Plan for Aviation Safety, Volume III.

### 8.3 异常处理与分布式弹性

- Nygard, M. T. (2007). *Release It! Design and Deploy Production-Ready Software.* Pragmatic Bookshelf.
- Microsoft Azure Architecture Center. "Circuit Breaker Pattern."
- Armstrong, J. (2003). "Making reliable distributed systems in the presence of software errors." PhD thesis, Royal Institute of Technology, Stockholm. (Erlang/OTP supervisor 设计)
- Temporal.io (2025). "Error handling in distributed systems: A guide to resilience patterns."

### 8.4 人类在环与警报管理

- Parasuraman, R., Sheridan, T. B., & Wickens, C. D. (2000). "A model for types and levels of human interaction with automation." *IEEE Transactions on Systems, Man, and Cybernetics.*
- ACM (2024). "Towards Human-AI Teaming to Mitigate Alert Fatigue in Security Operations Centers."
- ScienceDirect (2024). "Too much of a good thing: How varying levels of automation impact trust and performance."

### 8.5 保形预测与不确定性量化

- Vovk, V., Gammerman, A., & Shafer, G. (2005). *Algorithmic Learning in a Random World.* Springer.
- Xu, Z., & Gao, R. (2025). "Decision Theoretic Foundations for Conformal Prediction." *ICML 2025.*
- Angelopoulos, A. N., & Bates, S. (2023). "Conformal Prediction: A Gentle Introduction." *Foundations and Trends in Machine Learning.*
- Columbia University (2025). "Conformal prediction has been argued to provide meaningful uncertainty estimates." arXiv:2503.11709.

### 8.6 AI Agent 可靠性

- Platform Engineering (2025). "The Agent Reliability Score: What Your AI Platform Must Guarantee Before Agents Go Live."
- arXiv (2026). "Runtime Governance for AI Agents: Policies on Paths." arXiv:2603.16586.
- ACM (2025). "INSYTE: A Classification Framework for Traditional to Agentic AI."

---

## 9. 后续工作

1. **原型实现**：在 `openclaw-ops` Skill 中实际实现 P_UNKNOWN 的 L0-L3 层次
2. **模拟测试**：构造不匹配任何已知 Playbook 的状态组合，测试 P_UNKNOWN 的响应
3. **阈值校准**：收集实际运行数据后，调优置信度阈值
4. **进化机制验证**：验证从 P_UNKNOWN 日志中是否能有效提议新 Playbook
5. **通用化**：将 P_UNKNOWN 模式抽象为可复用的 Skill 模板组件

---

## 10. 结论

不完备状态处理不是一个"附加功能"，而是认知脚手架框架的**结构性必需品**。正如 Leveson 所言：事故不是因为组件故障，而是因为系统遇到了设计者未预见的状态组合。

我们的 P_UNKNOWN 设计综合了五个领域的最佳实践：

- 从 **OOD 检测** 借鉴了"主动识别异常"的思想——不是被动地"都不匹配所以是未知"，而是积极地度量"与已知的距离"
- 从 **安全关键系统** 借鉴了"不确定时退让"的原则——置信度越低，允许的操作越保守
- 从 **异常处理** 借鉴了"分层升级"的架构——L0 到 L3，每层有明确的进入/退出条件
- 从 **HITL 研究** 借鉴了"高质量升级"的设计——不是简单地说"我不会"，而是提供丰富的上下文和假设
- 从 **保形预测** 借鉴了"校准化不确定性"的理念——置信度不是随意设定的数字，而是有理论基础的度量

最终目标是让 Agent 在面对未知时，表现得像一个合格的初级工程师：承认不确定性、采取保守行动、收集信息、然后清晰地向高级工程师上报——而不是自作主张地冒险操作，也不是在第一时间就放弃并毫无信息地求助。
