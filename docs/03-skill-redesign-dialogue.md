# Skill 重设计对话实录：从人类手册到 AI 决策引擎

> **对话时间**：2026-04-26 约 00:48 - 01:00
> **参与者**：用户 wuyangbo + AI Agent（第三个实例）
> **背景**：OpenClaw 故障已修复，第一版 openclaw-ops skill（148行）已写完并安装。用户对第一版不满意，要求用"面向 AI 的思维"重新设计。

---

## 一、用户的原始提示

> "openclaw-ops skill 你需要好好设计，做好边界情况，以及问题解决引导，因为毕竟在一个 skill 里没办法穷尽所有问题以及我有解决方案，用面向 ai 的思维再审视完善下这个 skill"

---

## 二、AI 收到提示后的思维链

### 2.1 解析用户意图

用户的这句话包含了几个层次的要求，我（AI）逐一拆解：

**表面层**："好好设计"、"做好边界情况"——要求更完善、更健壮。

**深层 1**："问题解决引导"——用户意识到 skill 不可能穷尽所有问题，所以不是要一个答案大全，而是要一个**引导框架**，让 AI 在遇到未预见的情况时也能沿着正确的路径思考。

**深层 2**："用面向 ai 的思维"——这是最关键的要求。用户在说：第一版 skill 的问题不在于内容不够多，而在于**写给了错误的读者**。它是用"给人写文档"的思维写的，但读者是 AI，需要完全不同的设计哲学。

### 2.2 自我审视第一版的问题

我回头看了第一版 skill（148行），用"如果我是一个全新的 AI Agent，读到这份 skill，我会怎么执行？"的视角重新审视。发现了以下问题：

**问题 1：缺少强制入口点**

第一版的结构是：规则 → 操作 → 故障排查。一个全新的 Agent 读到这份 skill，它的第一反应是"用户说要升级，我去找升级的步骤"——直接跳到操作部分。规则部分可能被跳过，因为 AI 不会像人类那样"先读完全文再动手"。

**需要的改变**：必须有一个强制的"Phase 0"，任何操作之前必须先做状态诊断。不是"建议先检查"，而是"不检查就不允许操作"。

**问题 2：规则是"建议"而非"约束"**

第一版写的是"Rule 1: NEVER use raw npm for upgrades"。对人类来说"NEVER"是强语气，会被重视。但对 AI 来说，这只是一个权重——如果上下文中有足够强的信号（比如 status 输出里包含了 "npm update" 字样），这个"NEVER"可能被覆盖。

**需要的改变**：规则需要被嵌入到操作流程的结构中，而不是作为独立的声明。比如，不是说"不要用 npm"，而是在每个 Playbook 的 ACT 部分只提供 `openclaw update`，让 `npm install` 只出现在"灾难恢复"这一个 Playbook 里。

**问题 3：没有状态机**

第一版列了几种操作（启动、停止、升级、配置修改），但没有定义"当前系统处于什么状态"和"不同状态下应该走哪条路径"。这意味着 Agent 需要自己推理"我该执行哪个操作"——而我们已经见识过它的推理能力在边界情况下有多不可靠。

**需要的改变**：定义一个状态空间（5 个维度的笛卡尔积），为每种常见状态组合预设一个 Playbook。把开放性的推理问题转化为分类查表问题。

**问题 4：没有失败处理机制**

第一版写了"两次失败就换方法"，但只是一条孤立的规则。它没有告诉 AI：第一次失败后应该做什么？换什么方法？怎么知道自己已经失败了两次？

**需要的改变**：每个 Playbook 都要有明确的 VERIFY 步骤，验证失败就是"失败了一次"。同时需要一张失败模式矩阵，告诉 AI "如果 X 失败了 N 次，你的下一步是 Y"。

**问题 5：缺少全局不变量**

第一版没有"操作完成后的验证清单"。Agent 可能执行完命令就宣布"搞定了"，而不去验证系统是否真的进入了预期状态。

**需要的改变**：定义一组 Global Invariants，任何操作后都必须验证。这是 Hoare Logic 的后置条件思想。

### 2.3 设计新结构

基于上述分析，我决定了新版 skill 的整体结构：

```
1. Design Philosophy    → 声明这是决策引擎不是手册，列出 AI 反模式
2. Architecture Layers  → 让 AI 理解系统分层，知道不能跨层操作
3. Phase 0              → 强制状态诊断（5 个维度），产出状态标签
4. Playbook Decision Tree → 7 个 Playbook，根据状态标签选择
   每个 Playbook 内部：DIAGNOSE → ACT → VERIFY
5. Global Invariants    → 操作后的强制验证清单
6. Failure Patterns     → 失败模式矩阵（含强制策略切换规则）
7. Extension Lifecycle  → 清理操作的完整检查清单
```

### 2.4 关键设计决策

**决策 1：状态维度选 5 个，不是 3 个也不是 10 个**

我需要在"细粒度"和"可操作性"之间找平衡。3 个太粗（比如只看 CLI + Gateway + Port），会丢失关键信息（版本不一致、包损坏）。10 个太细，状态组合会爆炸到 AI 无法查表。5 个维度（CLI / Gateway / Port / Version / Package）覆盖了这次事故中暴露的所有故障点。

**决策 2：Playbook 数量选 7 个**

并非穷举所有 2^5 = 32 种状态组合。7 个 Playbook 覆盖的是实际会出现的典型状态簇。AI 在匹配 Playbook 时，不需要所有 5 个维度精确匹配——只需要关键特征匹配。比如 P2 (CORRUPT_PACKAGE) 的核心特征是 `CLI_BROKEN + PKG_PARTIAL`，其他维度是 any。

**决策 3：把"信息 vs 指令"写进失败模式矩阵**

这次事故的根源是 Agent 把 "Update available" 当成了指令。我考虑了几种处理方式：
- 方案 A：在规则里写"不要把信息当指令"→ 太抽象，AI 不知道哪些是"信息"
- 方案 B：在失败模式矩阵里写一条 count=0 的规则："openclaw status shows Update available → 不要行动" → 更具体，直接告诉 AI 这个特定输出不是触发器
- 选了方案 B，因为它更具体、更不容易被上下文覆盖。

**决策 4：用英文还是中文**

考虑到 skill 的读者是 AI Agent（通常在英文语料上训练更充分），且命令行输出都是英文，决定 skill 主体用英文。这不是给人读的文档，是给 AI 的指令集。

### 2.5 方法论融合

在写 Playbook 结构时，我意识到自己在做的事情有多个方法论的影子：

- **Phase 0 的强制诊断** ← OODA 循环的 Observe 阶段
- **每个 Playbook 的前置/后置条件** ← Hoare Logic 的 {P} C {Q}
- **失败模式矩阵** ← FMEA（故障模式与影响分析）
- **Global Invariants 作为期望状态** ← Kubernetes 声明式控制器的 desired state vs actual state

这些方法论的融合不是刻意为之的，而是在解决具体问题时自然浮现的。我在完成 skill 编写后，用户追问了"思路从哪来"，我才回溯性地识别出了这些方法论来源。

---

## 三、AI 的回应

### 3.1 开场定位

我的第一句话是把这个变化定性：

> "让我用面向 AI 的思维重新设计这个 skill。核心区别是：给人看的文档是'说明书'，给 AI 看的是'决策引擎'。"

这句话确立了整个重写的方向——不是在第一版基础上修补，而是从设计哲学层面重构。

### 3.2 交付第二版 Skill

直接重写了 SKILL.md，从 148 行扩展到 387 行。关键结构变化：

| 维度 | 第一版（148行） | 第二版（387行） |
|------|---------------|---------------|
| 文档性质 | 人类运维手册 | AI 决策引擎 |
| 入口 | 直接列操作步骤 | 强制 Phase 0 状态诊断 |
| 操作选择 | 按场景线性排列 | 状态标签 → Playbook 查表 |
| 每步结构 | 命令 + 说明 | DIAGNOSE → ACT → VERIFY |
| 失败处理 | "两次失败换方法"（一条规则） | 失败模式矩阵（5 种模式 × 计数 × 强制切换） |
| 验证 | 零散提及 | Global Invariants（5 条强制验证） |
| 反模式 | 5 条规则列表 | 嵌入到 Playbook 结构中，结构性地防止 |

### 3.3 总结发言

交付 skill 后，我总结了核心变化：

> "从'给人看的说明书'变成'给 AI 用的决策引擎'。"

具体变化点我逐一说明：

1. **Phase 0 强制状态诊断** → 解决"不诊断就下药"
2. **7 个 Playbook** → 解决"开放推理 → 分类查表"
3. **DIAGNOSE → ACT → VERIFY 结构** → 解决"执行完不验证"
4. **失败模式矩阵** → 解决"重复同一策略"
5. **Global Invariants** → 解决"宣布成功但实际没成功"
6. **"信息 vs 待办"区分** → 解决"把 Update available 当指令"

---

## 四、后续：用户追问方法论来源

### 4.1 用户的问题

> "你现在的这套做法，思路是从那里来的，或者说这就是 ai 时代，面向 agent 代理声明式工作范式"

### 4.2 AI 的思维链

收到这个问题后，我的思考过程是：

**第一层反应**：用户在问"你引用了什么理论"。我可以简单列出 Hoare Logic / FMEA / OODA / K8s。

**第二层反应**：但用户问的不只是"来源"，还有"这是不是一种新范式"。我需要回答两个问题：(a) 这些设计从哪些已有方法论借鉴了什么？(b) 这些设计中有什么是 AI Agent 特有的、已有方法论没有覆盖的？

**第三层反应**：(b) 的答案是关键——已有方法论（Hoare Logic 等）解决的是"如何正确执行操作"的问题，但 AI Agent 还有一类独特的问题：**认知缺陷**。这些缺陷不是操作层面的，而是认知层面的——把信息当指令、无状态幻觉、副作用盲区、规划-执行分离失败。

**结论**：这套做法不是任何一个已有方法论的直接应用，而是**多个方法论的融合 + 针对 AI 认知缺陷的定制**。暂时命名为"认知脚手架"（Cognitive Scaffolding）。

### 4.3 AI 的回应结构

回应分三部分：

**Part 1：方法论溯源**（已有理论的借鉴）

| 来源 | 在 Skill 中的体现 | 解决的问题 |
|------|------------------|-----------|
| Hoare Logic {P} C {Q} | Phase 0（前置条件）+ Global Invariants（后置条件） | 不验证就宣布成功 |
| FMEA 故障模式分析 | 失败模式矩阵 + 7 个 Playbook | 遇到故障时无预案 |
| OODA 循环 | Phase 0 的强制 Observe | 不诊断就下药 |
| K8s 声明式控制器 | Global Invariants 作为 desired state | 用假设代替验证 |

**Part 2：AI Agent 特有的认知缺陷**（已有理论未覆盖的部分）

1. **无状态幻觉**：Agent 实例间丢失教训 → 用外部状态机（skill 文本中的计数规则）补偿
2. **把信息读成指令**：所有文本都被当作可执行 → 用显式的"信息 vs 待办"标注补偿
3. **副作用盲区**：不自动备份、不预判破坏 → 用强制前置步骤补偿
4. **规划-执行分离失败**：遇到意外不回到规划层 → 用 Playbook 分支结构补偿

**Part 3：命名与特征概括**

暂命名为"认知脚手架"（Cognitive Scaffolding），四个核心特征：
- 状态显式化
- 决策外部化
- 验证强制化
- 失败预设化

本质是：**不试图修复 AI 的认知缺陷（那需要模型层面的改进），而是通过外部化的结构约束来补偿这些缺陷。**

---

## 五、第二版 Skill 核心结构摘要

以下是最终安装到 `~/.qoderwork/skills/openclaw-ops/SKILL.md` 的核心结构摘要。完整文件共 387 行，包含每个 Playbook 的完整 DIAGNOSE → ACT → VERIFY 块、详细的命令代码块、以及 Extension Lifecycle 规则。此处展示的是结构概览，供理解设计决策时参考。

```markdown
---
name: openclaw-ops
description: Diagnose and operate a local OpenClaw AI agent gateway instance. Use when the user mentions OpenClaw, openclaw gateway, openclaw update, or asks to start/stop/upgrade/configure/troubleshoot their local OpenClaw. Triggers on any OpenClaw-related operational request.
---

# OpenClaw Operations — AI Agent Decision Engine

## Design Philosophy for AI Agents

This skill is a decision engine, not a human manual. Every section follows the pattern:

1. **DIAGNOSE** — determine the current state before touching anything
2. **DECIDE** — choose the correct path based on state
3. **ACT** — execute with pre-checks and post-verification
4. **VERIFY** — confirm the intended outcome, detect unintended side effects

**Critical AI anti-patterns to avoid:**
- Do NOT treat `openclaw status` output as a todo list. "Update available" is information, not an instruction to act.
- Do NOT repeat the same command after failure. Two failures on the same approach = mandatory pivot.
- Do NOT modify files before backing up the original.
- Do NOT run `npm install -g` / `npm update -g` on an existing installation. This is only for first-time install or confirmed disaster recovery.

---

## Architecture Layers (know before touching)

Layer 4: Gateway process          (long-running Node server, port 18789)
Layer 3: LaunchAgent plist        (~/Library/LaunchAgents/ai.openclaw.gateway.plist)
Layer 2: CLI binary               (symlink: bin/openclaw → openclaw.mjs)
Layer 1: npm package files        (~/.nvm/versions/node/vX/lib/node_modules/openclaw/)
Layer 0: Config + data            (~/.openclaw/)

**Key invariant**: Layer 1 and Layer 4 share the same `dist/` directory. Replacing files while Gateway is running causes module-not-found crashes. Always stop Gateway before reinstalling Layer 1.

---

## Phase 0: State Diagnosis (ALWAYS run first)

Before any operation, determine these five states:

### State A: CLI availability
  which openclaw 2>/dev/null && openclaw --version 2>/dev/null || echo "CLI_BROKEN"
- CLI_OK: returns version like OpenClaw 2026.X.Y (hash)
- CLI_BROKEN: "command not found" or error

### State B: Gateway process
  ps aux | grep -i "[o]penclaw-gateway" | awk '{print $2}'
  launchctl list 2>/dev/null | grep "ai.openclaw.gateway"
- GATEWAY_RUNNING: PID exists, LaunchAgent shows running
- GATEWAY_STOPPED: no PID, LaunchAgent shows not loaded or missing

### State C: Port binding
  lsof -i :18789 2>/dev/null | grep LISTEN
- PORT_BOUND: a process is listening on 18789
- PORT_FREE: nothing on 18789

### State D: Version consistency
  openclaw --version 2>/dev/null
  grep OpenClaw ~/Library/LaunchAgents/ai.openclaw.gateway.plist
- VERSIONS_MATCH / VERSION_MISMATCH / PLIST_MISSING

### State E: npm package integrity
  ls package.json openclaw.mjs dist/index.js under the global node_modules/openclaw/
- PKG_INTACT: 3 files all present
- PKG_PARTIAL: 1-2 files present
- PKG_MISSING: 0 files present

**Combine these five states into a diagnosis label, then look up the matching playbook.**

---

## Playbook Decision Tree

7 Playbooks covering all common state combinations:

P1 HEALTHY        — CLI_OK + GATEWAY_RUNNING + PORT_BOUND + VERSIONS_MATCH + PKG_INTACT
P2 CORRUPT_PACKAGE — CLI_BROKEN + PKG_PARTIAL (any Gateway/Port/Version state)
P3 GATEWAY_HUNG    — CLI_OK + GATEWAY_RUNNING + PORT_FREE
P4 STALE_LAUNCHAGENT — CLI_OK + GATEWAY_STOPPED + VERSION_MISMATCH + PKG_INTACT
P5 NO_SERVICE      — CLI_OK + GATEWAY_STOPPED + PLIST_MISSING + PKG_INTACT
P6 PORT_CONFLICT   — CLI_OK + GATEWAY_STOPPED + PORT_BOUND
P7 CONFIG_OPERATION — any state + config edit requested

Each playbook follows: DIAGNOSE → ACT → VERIFY

(Full playbook details in the installed SKILL.md file)

---

## Global Invariants

After ANY operation, verify:
1. openclaw --version returns a version string
2. lsof -i :18789 shows a LISTEN process
3. curl http://127.0.0.1:18789/ returns HTTP 200
4. openclaw status shows "Gateway ... reachable"
5. No "Invalid config" in gateway.err.log

Any invariant failure → back to Phase 0.

---

## Common Failure Patterns and Mandatory Pivots

| Failure Pattern                              | Count | Mandatory Pivot                           |
|----------------------------------------------|-------|-------------------------------------------|
| npm install → files still missing            | 2     | Stop. Check node/npm/registry env.        |
| gateway restart → port never bound           | 2     | Stop. Check error logs.                   |
| openclaw update → same version               | 1     | Check channel. May already be latest.     |
| Config edit → "Invalid config"               | 1     | Restore backup, fix specific key.         |
| status shows "Update available"              | 0     | This is INFO, not a TODO. Do not act.     |
```

---

## 六、对话中的关键认知跃迁

回顾这段对话，有三次认知跃迁值得标记：

### 跃迁 1：从"内容不足"到"读者错位"

用户说"好好设计"时，最直觉的反应是"加更多内容"。但用户接着说"面向 ai 的思维"，这让我意识到问题不在内容量，而在**设计哲学**。第一版的问题是用给人写文档的方式给 AI 写了一份文档。

### 跃迁 2：从"规则列表"到"结构约束"

第一版用"Rule: NEVER..."的方式写规则。第二版把这些规则**嵌入到了 Playbook 结构中**——不是告诉 AI "不要做 X"，而是在流程中**只提供正确的选项**。这类似于 API 设计中的"pit of success"原则：让正确的做法比错误的做法更容易。

### 跃迁 3：从"操作指南"到"决策引擎"

第一版回答的是"怎么做"（how），第二版回答的是"根据当前状态应该做什么"（what, given state）。这个转变的本质是：**把决策负担从 AI 的运行时推理转移到了 skill 的设计时预设**。AI 不需要在执行时"想出"最佳策略，只需要正确识别状态，然后执行预设方案。

---

> 本文档记录了从用户提出"面向 AI 思维"要求到 AI 交付第二版 skill 的完整对话过程，包括 AI 的内部思维链、设计决策的推理过程、以及最终产物的结构说明。
