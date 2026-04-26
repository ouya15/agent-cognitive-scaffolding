# AI Agent Cognitive Scaffolding

> 从一次 OpenClaw 运维故障中提炼的 AI Agent 认知脚手架设计方法论。

## 项目背景

2026 年 4 月 25 日深夜，本地 OpenClaw（AI Agent Gateway）故障修复过程中，三个 AI Agent 先后接手修复，连续失败多次。最终修复后，我们将这次事件的教训系统化为一套 **AI Agent 认知脚手架**（Cognitive Scaffolding）设计方法，并实现了完整的 `openclaw-ops` Skill。

这次经历暴露了当前 AI Agent 在运维场景中的五个根本认知缺陷，也展示了**"为 AI 设计决策框架"**这一新思路的有效性。

## 核心理念

**与其抱怨 AI 不够聪明，不如给它搭一副能让它聪明起来的脚手架。**

认知脚手架不是替 AI 做决策，而是为它的决策过程提供结构化支撑：

| 原则 | 含义 | 解决什么缺陷 |
|------|------|-------------|
| 状态显式化 | 不允许 Agent 假设任何状态，必须通过命令实际检查 | 不诊断就下药 |
| 决策外部化 | 将决策规则写在 Skill 中，不依赖 Agent 的"推理能力" | 边界推理不可靠 |
| 验证强制化 | 每个操作后必须验证结果 | 执行完不验证 |
| 失败预设化 | 预先定义失败场景和替代策略 | 重复同一策略期待不同结果 |

## 方法论溯源

本项目的 Skill 设计融合了四种经典工程方法，每种都针对特定的 AI 认知缺陷：

- **Hoare Logic** {P} C {Q} —— 前置/后置条件验证，解决"执行了命令就以为完成了任务"
- **FMEA** 故障模式与影响分析 —— 预设故障模式与响应方案，解决"遇到故障无预案"
- **OODA Loop** 观察-判断-决策-行动 —— 强制性 Phase 0 状态诊断，解决"不诊断就下药"
- **Kubernetes 声明式控制器** —— 期望状态 vs 实际状态对比，解决"用假设代替验证"

## 项目结构

```
agent-cognitive-scaffolding/
├── README.md                          # 项目说明
├── docs/
│   ├── 01-incident-record.md          # 完整事件实录（10 阶段，不美化不遗漏）
│   ├── 02-blog-post.md               # 技术博客文章（面向面试官，展示元思考能力）
│   ├── 03-skill-redesign-dialogue.md # Skill 重设计对话实录（含 AI 内部思维链）
│   └── 04-cross-review-report.md     # 四份文档交叉评审报告
└── skills/
    └── openclaw-ops/
        └── SKILL.md                   # OpenClaw 运维 Skill（458 行 AI 决策引擎）
```

## 关键文档导航

### [01-事件完整实录](docs/01-incident-record.md)

严格按时间线记录的完整事件过程，包括：
- 10 个阶段的完整时间线（23:12 ~ 01:00）
- 三个 Agent 的行为轨迹与失误分析
- 5 个根因的根因分析
- 用户的 4 次关键干预角色分析
- 对 AI Agent 开发者/使用者/人机协作模式的启示

### [02-技术博客文章](docs/02-blog-post.md)

标题："当 AI 修了三遍还没修好：从一次深夜故障谈 Agent 的认知脚手架设计"

第一人称叙事，面向面试官读者，展示：
- 元知识能力（知道 AI 为什么修不好）
- 系统自动化能力（从一次性修复到可复用知识资产）
- 概括能力（从具体事件到可迁移方法论）

### [03-Skill 重设计对话实录](docs/03-skill-redesign-dialogue.md)

记录了从"面向 AI 的思维"提示到第二版 Skill 交付的完整对话，包括：
- AI 对第一版 5 个结构性问题的自我审视
- 4 个关键设计决策的推理过程
- 3 次认知跃迁的分析

### [04-交叉评审报告](docs/04-cross-review-report.md)

对四份 MD 文档的完整性与一致性交叉评审，包括：
- 5 个维度的检查结果
- 4 个需要修复的不一致问题
- 修复清单

## openclaw-ops Skill

`skills/openclaw-ops/SKILL.md` 是本项目的核心产出——一个 458 行的 AI 决策引擎。

### 结构概览

| 模块 | 内容 | 解决的问题 |
|------|------|-----------|
| Design Philosophy | 4 条 AI 反模式禁令 | 信息误读为指令、重复策略 |
| Architecture Layers | 5 层模型（+ domain 双层状态） | 跨层操作盲区 |
| Phase 0 | 6 维度状态诊断（CLI / Gateway / Port / Version / Package / Domain） | 不诊断就下药 |
| 8 个 Playbooks | P1 HEALTHY ~ P8 DOMAIN_DEREGISTERED | 开放推理 → 分类查表 |
| Global Invariants | 6 条操作后强制验证 | 宣布成功但实际没成功 |
| Failure Patterns | 7 种失败模式 × 计数 × 强制切换 | 重复同一策略期待不同结果 |
| Extension Lifecycle | 3 位置清理规则 | 残留扩展导致启动超时 |

### 关键设计决策

- **"两次失败必须换路"规则**：同一策略失败 2 次，强制切换，禁止第 3 次尝试
- **信息 vs 指令显式区分**：`"Update available"` 是 INFO，不是 TODO
- **永远通过应用层操作**：ALWAYS 用 `openclaw update`，NEVER 用 `npm update/install`
- **静态配置 ≠ 动态注册**：Layer 3（LaunchAgent）的 plist 文件存在不等于服务在 domain 中有效注册

## 演进时间线

| 日期 | 里程碑 |
|------|--------|
| 2026-04-25 | OpenClaw 故障发生，三个 Agent 先后修复 |
| 2026-04-26 | 第一版 Skill（148 行）→ 第二版 Skill（387 行）→ 博客/实录/对话链产出 |
| 2026-04-26 | 交叉评审，修复 4 个不一致问题 |
| 2026-04-26 | 发现 `stop → bootout → domain 卸载` 问题 |
| 2026-04-26 | Skill 扩展到 458 行，新增 Phase 0 State F、Playbook P8、P1c 双模式 |
| 2026-04-27 | 项目初始化 |

## 后续研究方向

1. **不完备状态处理机制**：当 Phase 0 诊断结果不匹配任何已知 Playbook 时的兜底策略
2. **运行时状态聚类**：从 Agent 的执行数据中自动发现新的故障模式簇
3. **失败驱动的 Playbook 进化**：自动从 "同一策略失败两次" 事件生成候选条目
4. **形式化推导**：用程序分析技术自动推导工具的状态空间和状态转移

## 许可证

本项目旨在记录和分享一次真实的 AI Agent 协作经验，内容可自由用于教育和研究目的。
