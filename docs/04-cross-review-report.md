# 四份 MD 文档交叉评审报告

> 评审时间：2026-04-26
> 评审对象：
> 1. `openclaw-incident-record.md` —— 事故完整实录
> 2. `blog-ai-agent-cognitive-scaffolding.md` —— 面向面试官的技术博客
> 3. `skill-redesign-dialogue-chain.md` —— Skill 重设计对话实录
> 4. `SKILL.md` —— openclaw-ops Skill 本体（`~/.qoderwork/skills/openclaw-ops/SKILL.md`）

---

## 一、一致性检查结论

| 检查维度 | 结果 | 说明 |
|---------|------|------|
| 版本号时间线 | ✅ 一致 | 三份文档均记录：v2026.3.24 → v2026.4.8 → v2026.4.23 |
| 事件时间范围 | ✅ 一致 | 约 23:12 ~ 01:00，总时长约 1 小时 48 分钟 |
| Skill 版本信息 | ✅ 一致 | v1 = 148 行，v2 = 387 行，路径一致 |
| Agent 分工与失误 | ✅ 一致 | Agent 1/2/3 的行为描述在三份文档中无矛盾 |
| 最终修复步骤 | ✅ 一致 | 清理 → 重装 → install → start → 验证 |
| 方法论溯源 | ✅ 一致 | Hoare Logic / FMEA / OODA / K8s 四源一致 |

---

## 二、发现的不一致问题（按严重程度排序）

### 🔴 问题 1：事故记录中 Global Invariants 不完整

**位置**：`openclaw-incident-record.md` 第 507-512 行

**现状**：只列出 3 条不变量：
- CLI 入口可用
- Gateway 可达
- 包完整性

**SKILL.md 实际有 5 条**（第 340-350 行）：
1. `openclaw --version` 返回版本号
2. `lsof -i :18789` 显示 LISTEN 进程
3. `curl http://127.0.0.1:18789/` 返回 HTTP 200
4. `openclaw status` 显示 "Gateway ... reachable"
5. gateway.err.log 中无 "Invalid config"

**影响**：事故记录作为"最完整的实录"，却在关键设计要素上缺了两条，与 SKILL.md 和博客文章不一致。

**修复**：补充至 5 条，与 SKILL.md 完全一致。

---

### 🔴 问题 2：事故记录中 5-Layer 架构缺失

**位置**：`openclaw-incident-record.md` 全文

**现状**：事故记录在第 16 行的组件表中提到了 LaunchAgent、CLI、npm 等组件，但没有明确给出 SKILL.md 中定义的 5 层架构（Layer 0~4）。

**对比**：
- 博客文章第 6.4 节提到了"LaunchAgent 配置指向正确路径"，隐含了分层概念
- 对话链完整展示了 5 层架构摘要
- SKILL.md 开篇就是 Architecture Layers 一节

**影响**：事故记录是"面向完整复盘"的文档，缺少架构分层描述会降低读者对后续"跨层操作"失误的理解。

**修复**：在"涉及的关键组件"后补充 5-Layer 架构说明。

---

### 🟡 问题 3：博客文章中 status 输出格式简化，削弱因果逻辑

**位置**：`blog-ai-agent-cognitive-scaffolding.md` 第 14 行

**现状**：博客写的是：
```
Update available: 2026.4.23
```

**事故记录中的实际输出**（第 118 行）：
```
Update: available · pnpm · npm update 2026.4.23
```

**问题**：博客省略了 `"npm update"` 字样——而 Agent 1 正是因为看到 `"npm update"` 这几个字才直接执行了 `npm update -g`。省略这一关键细节会削弱"信息误读为指令"这一核心论点的说服力。

**修复**：保留博客的简洁风格，但在正文中补充说明实际输出包含 `"npm update"` 字样。

---

### 🟡 问题 4：对话链中 SKILL.md 内容标注为"完整内容"实为摘要

**位置**：`skill-redesign-dialogue-chain.md` 第 201 行、第 205-317 行

**现状**：标注为"第二版 Skill 完整内容"，但实际只展示了：
- Design Philosophy
- Architecture Layers（无代码块格式）
- Phase 0（命令以行内代码展示，非代码块）
- Playbook Decision Tree（只有标签和一句话描述，无完整 ACT/VERIFY 内容）
- Global Invariants（简化版）
- Failure Patterns（简化版）

**实际 SKILL.md 有 387 行**，包含每个 Playbook 的完整 DIAGNOSE → ACT → VERIFY 块、详细的命令代码块、Extension Lifecycle 等。

**修复**：将标题和说明文字改为"核心结构摘要"，避免误导。

---

### 🟢 问题 5：认知缺陷数量 4 vs 5（抽象层次不同，非错误）

**位置**：
- 事故记录第 10 阶段：4 个认知缺陷（无状态幻觉、信息误读、副作用盲区、规划-执行分离失败）
- 博客文章：5 个缺陷（上述 4 个 + 上下文继承断层）

**分析**：
- 事故记录中，"上下文继承策略不继承教训"被放在第 5 阶段的第 5 个根因中
- 第 10 阶段将 5 个根因抽象为 4 个"通用认知缺陷"，其中"上下文继承断层"被归入"无状态幻觉"
- 博客文章保留了 5 个缺陷的原始形式，因为叙事更流畅

**结论**：这不是事实错误，是抽象粒度不同。无需修改，但在评审报告中注明。

---

### 🟢 问题 6：博客中未提及 extensions 目录清理细节

**位置**：`blog-ai-agent-cognitive-scaffolding.md`

**现状**：博客提到"移除一个不用的频道配置"，但没有说明残留 extensions 目录导致启动超时这一关键细节。

**分析**：博客面向面试官，需要控制篇幅。这个细节在事故记录和 SKILL.md 中都有完整记录。这不是必须修复的不一致，但建议在博客中至少提及"残留文件"这一概念，以展示对边界情况的考虑。

**修复**：在博客"修好了，然后呢？"一节中补充一句关于 extensions 清理的简要说明。

---

## 三、完整性检查

| 文档 | 内容覆盖 | 缺失项 |
|------|---------|--------|
| 事故记录 | 时间线、根因、修复步骤、Skill 演进、范式讨论 | 5-Layer 架构、完整 Global Invariants |
| 博客文章 | 叙事、缺陷分析、方法论、认知分工 | extensions 清理细节、status 输出完整格式 |
| 对话链 | 用户提示、AI 思维链、设计决策、认知跃迁 | 无显著缺失（但 SKILL.md 摘要标注需修正） |
| SKILL.md | 完整决策引擎（Phase 0 / 7 Playbooks / Invariants / Failure Patterns / Extension Lifecycle） | 无 |

---

## 四、修复清单

1. **openclaw-incident-record.md**：补充 5-Layer 架构说明；补充 Global Invariants 至 5 条
2. **blog-ai-agent-cognitive-scaffolding.md**：补充 status 输出中的 `"npm update"` 字样说明；补充 extensions 残留问题
3. **skill-redesign-dialogue-chain.md**：将"完整内容"改为"核心结构摘要"
4. **SKILL.md**：无需修改

---

> 评审完成。共发现 4 个需要修复的问题（2 个重要、2 个中等），1 个抽象层次差异（无需修改），1 个建议性补充。
