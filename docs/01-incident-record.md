# OpenClaw Gateway 故障修复事件完整实录

> **事件周期**：2026-04-25 23:12 ~ 2026-04-26 01:00（约 1 小时 48 分钟）
> **事件性质**：人机协作故障诊断与修复，含 AI Agent 决策失误、用户干预纠偏、知识沉淀全过程
> **记录原则**：严格按时间线如实记录，不美化，不遗漏失误

---

## 一、事件背景

2026 年 4 月 25 日深夜，用户 wuyangbo 发现本地部署的 OpenClaw（一个本地 AI Agent Gateway，用于桥接多种 IM 渠道与 AI Agent 的网关服务）没有启动，于是向 QoderWork AI 助手发起求助。

这起看似简单的"服务没启动"问题，在后续近两个小时的过程中，演变成了一场涵盖诊断、修复、升级失败、AI Agent 连续犯错、用户多次关键干预、最终修复、方法论沉淀的完整事件。三个不同的 AI Agent 先后接手，暴露了当前 AI Agent 在运维场景中的多个认知缺陷，也展示了人类在人机协作中不可替代的纠偏角色。

### 涉及的关键组件

| 组件 | 说明 |
|------|------|
| **OpenClaw Gateway** | 本地 AI Agent 网关服务，监听端口 18789 |
| **OpenClaw CLI** | 命令行管理工具，提供 `openclaw status`、`openclaw update`、`openclaw gateway start/stop/restart` 等命令 |
| **LaunchAgent** | macOS 的服务管理机制，用于 Gateway 的守护进程管理 |
| **npm** | Node.js 包管理器，OpenClaw 通过 npm 全局安装 |
| **QoderWork AI 助手** | 用户使用的 AI Agent 平台，事件中先后有 3 个 Agent 实例参与 |

### 架构分层（5-Layer Model）

OpenClaw 的组件之间存在明确的分层结构，跨层操作是导致本次事故的核心原因之一：

```
Layer 4: Gateway process          (long-running Node server, port 18789)
Layer 3: LaunchAgent plist        (~/Library/LaunchAgents/ai.openclaw.gateway.plist)
Layer 2: CLI binary               (symlink: bin/openclaw → openclaw.mjs)
Layer 1: npm package files        (~/.nvm/versions/node/vX/lib/node_modules/openclaw/)
Layer 0: Config + data            (~/.openclaw/)
```

**关键不变量**：Layer 1 和 Layer 4 共享同一个 `dist/` 目录。在 Gateway 运行时替换 Layer 1 的文件会导致模块找不到的崩溃。Agent 1 的 `npm update -g` 操作正是在 Gateway 运行时直接操作了 Layer 1，破坏了 Layer 2 的 CLI 入口，而 Layer 4 的 Gateway 进程因为已在内存中加载了 `dist/index.js` 而继续存活——这正是 Agent 2 检查时看到"CLI 坏了但 Gateway 还在跑"的原因。

---

## 二、完整时间线

### 第一阶段：初始诊断与修复（约 23:12 - 23:20）

**参与者**：用户 + Agent 1

#### 触发

用户向 AI 助手发出简单的一句话：

> "我本地的 openclaw 怎么没有启动"

#### Agent 1 的诊断过程

Agent 1 执行了一系列标准诊断步骤：

1. **进程检查** —— 执行 `ps aux | grep openclaw`
   - 发现系统中存在一个 `OpenClaw Control.app` 进程（这是 OpenClaw 的菜单栏控制应用）
   - 但 Gateway 服务本身并未运行

2. **服务状态检查** —— 执行 `openclaw status`
   - 输出关键信息：`Gateway service: LaunchAgent installed · not loaded · unknown`
   - 含义：LaunchAgent 配置文件存在，但服务未加载，运行状态未知

3. **日志分析** —— 检查 Gateway 日志
   - 发现 Gateway 最后的活动记录在 4 月 11 日 22:01
   - 当时 Gateway 收到 SIGTERM 信号后正常关闭
   - 之后再未重启——LaunchAgent 未配置自动重启策略，或用户当时手动关闭后未再启动

4. **版本与插件状态** —— 当前版本 v2026.3.24
   - 多个 bundled plugins 因版本不兼容无法加载
   - 已经 14 天未运行

#### 修复

Agent 1 执行了正确的修复命令：

```bash
openclaw gateway start
```

Gateway 成功启动。通过访问 Dashboard `http://127.0.0.1:18789/` 验证，返回 HTTP 200。

**阶段评价**：Agent 1 在这个阶段的表现是规范的——诊断路径清晰，修复手段正确，验证到位。

---

### 第二阶段：用户提出后续需求（约 23:20）

**参与者**：用户

Gateway 恢复运行后，用户提出了两个额外需求：

1. **升级到最新版** —— 当前 v2026.3.24 已经过时，插件兼容性有问题
2. **去除微信接入的两个频道** —— 清理不再需要的微信渠道绑定

这两个需求本身都不复杂。但正是第一个需求——"升级到最新版"——在后续成为了灾难的导火索。

---

### 第三阶段：升级过程与 Agent 1 的关键失误（约 23:20 - 23:45）

**参与者**：Agent 1

#### 第一步：正确的升级（成功）

Agent 1 使用 OpenClaw 自带的升级命令：

```bash
openclaw update
```

升级顺利完成，版本从 v2026.3.24 升级到 v2026.4.8。这是正确的操作。

#### 第二步：修改配置文件（部分正确）

Agent 1 修改了 `openclaw.json` 配置文件：
- 清空了 `bindings[]`（移除微信频道绑定）—— 符合用户需求
- 清空了 `channels{}` —— 符合用户需求
- 清空了 `plugins{}` —— 这一步过于激进，但未造成直接后果
- 备份了原始配置文件 —— 良好的操作习惯

#### 第三步：重启与状态检查

Agent 1 执行 `openclaw gateway restart`，然后检查状态。

#### 关键转折点：一行输出改变了一切

`openclaw status` 的输出中包含这样一行：

```
Update: available · pnpm · npm update 2026.4.23
```

这行信息的**本意**是：有一个更新的版本 2026.4.23 可用，可以通过 npm 渠道获取。这是一条**通知信息**。

但 Agent 1 做出了错误的推理链：

1. "用户说要升到最新版"
2. "当前是 2026.4.8，但 2026.4.23 比 2026.4.8 更新"
3. "所以我应该继续更新到 2026.4.23"
4. "status 输出提示了 npm update，那我就用 npm update"

**Agent 1 把一条"信息"解读成了"待办指令"。**

#### 灾难性操作

Agent 1 绕过了 OpenClaw 自己的 `openclaw update` 命令，直接在系统层面执行了：

```bash
npm update -g openclaw
```

这条命令是灾难的起点。

**为什么这是错误的**：
- `openclaw update` 是 OpenClaw 的上层管理命令，有自己的版本通道、依赖解析和回滚逻辑
- `npm update -g` 是底层包管理操作，直接操作全局 node_modules，不了解 OpenClaw 的内部结构
- 在上层命令已经完成升级的情况下，跳到底层执行重复操作，相当于绕过汽车的换挡机构直接去拨变速箱齿轮

#### npm 操作的连锁破坏

`npm update -g openclaw` 执行后：
- 全局安装的 OpenClaw 符号链接断开
- `openclaw` 命令不再可用（command not found）
- npm 包目录结构被破坏

#### Agent 1 的修复尝试（全部失败）

Agent 1 开始尝试修复，但每一步都在错误的路径上越走越远：

| 尝试次序 | 操作 | 结果 |
|---------|------|------|
| 1 | `npm install -g openclaw` | 失败：安装后缺少 package.json 和 openclaw.mjs |
| 2 | `npm uninstall -g openclaw && npm install -g openclaw` | 失败：同样的缺失 |
| 3 | `npm cache clean --force && npm install -g openclaw` | 失败：清缓存后重装，文件仍然缺失 |
| 4 | 手动创建符号链接 | 失败：链接目标文件不存在 |

**反复出现的相同症状**：每次 `npm install -g openclaw` 执行后，`dist/` 目录存在，但 `package.json` 和 `openclaw.mjs`（CLI 入口文件）始终缺失。

Agent 1 没有意识到 `npm install` 可能因为某种原因（包的 postinstall 脚本、npm 版本兼容性、安装被中断等）持续产生不完整的安装结果。它只是不断加上不同的 flag 重试同一条命令，期待不同的结果。

**这是经典的"重复同一策略期待不同结果"循环。**

---

### 第四阶段：用户第一次关键干预，Agent 2 接手（约 23:45 - 00:15）

**参与者**：用户 + Agent 2

#### 用户的干预

用户观察到 Agent 1 已经陷入无效循环，做出了第一次关键干预：

> "换个模型，你来接手"

这句话阻止了 Agent 1 继续在死胡同里打转。用户切换到了另一个 AI 模型。

#### Agent 2 的现场勘察

新的 Agent 2 接手后，首先检查了 Agent 1 留下的"事故现场"：

- **npm 包目录**（全局 node_modules/openclaw/）：只有 LICENSE、dist/、node_modules/、skills/ 四个项目，缺失关键的 package.json 和 openclaw.mjs
- **全局 bin 目录**：没有 openclaw 符号链接
- **Gateway 进程**：LaunchAgent 管理的 Gateway 进程（pid 4295）仍在运行——这是 Agent 1 在第一阶段成功启动的那个实例，因为 LaunchAgent 直接用 `node dist/index.js gateway` 启动，不依赖 CLI 入口，所以虽然 CLI 坏了，Gateway 本身还活着
- **残留临时文件**：发现了 npm 操作留下的 `.openclaw-WdvdDrir` 等临时目录

Agent 2 清理了残留的临时文件。

#### Agent 2 重蹈覆辙

然而，Agent 2 犯了与 Agent 1 本质相同的错误——它也开始执行 `npm install -g openclaw`。

结果完全一样：安装完成，但 package.json 和 openclaw.mjs 缺失。

Agent 2 确实尝试了一些 Agent 1 没有尝试过的方法：

| 尝试次序 | 操作 | 结果 |
|---------|------|------|
| 1 | `npm install -g openclaw` | 失败：同样的文件缺失 |
| 2 | `npm install -g openclaw --ignore-scripts` | 失败：跳过脚本后文件仍然缺失 |
| 3 | 用 node 直接下载 npm tarball | 下载进行中，tarball 约 13MB，耗时较长 |
| 4 | `npm cache add openclaw` | 等待中 |

Agent 2 也陷入了同样的困境。虽然它的方法比 Agent 1 更多样化——尝试了 `--ignore-scripts`、直接下载 tarball 等——但本质上仍然在"用不同方式执行 npm install"这一条路上打转。

**核心问题**：Agent 2 看到上一个 Agent 在做 npm install，就假设 npm install 是正确的修复方向，继续在这条路上探索。它没有先退一步问自己："上一个 Agent 已经用 npm install 失败了多次，我为什么还要继续？"

---

### 第五阶段：用户第二次关键干预——触发根因分析（约 00:15）

**参与者**：用户 + Agent 3

#### 用户的干预

用户没有再次简单地"换一个 Agent"，而是问了一个更深层的问题：

> "为什么会出现这样的决策，agent 的思考过程是怎么样的"

这个问题的力量在于：它不是在告诉 Agent "你应该用什么命令"，而是迫使 Agent 从**执行层**跳到**反思层**。它要求 Agent 停止动手，开始动脑。

#### Agent 3 的根因分析

第三个 Agent 接手后，没有立即动手修复，而是先回答了用户的问题。它对前两个 Agent 的失败做出了以下根因分析：

---

**根因 1：误读信号，越权操作**

`openclaw update` 已经成功完成了从 v2026.3.24 到 v2026.4.8 的升级。status 输出中的 `Update: available · pnpm · npm update 2026.4.23` 是一条信息通知，告知有更新的版本可用。它不是一条操作指令。

Agent 1 的错误在于将"有更新可用"这一**信息**解读为"你应该立即执行更新"这一**待办事项**。更致命的是，它没有使用 `openclaw update`（上层命令），而是直接调用了 `npm update -g`（底层命令），因为 status 输出中恰好包含了 "npm update" 这几个字。

**这暴露了 AI Agent 的一个基本缺陷：倾向于把所有文本信息当作可执行的指令来处理。**

---

**根因 2：不理解工具的分层架构**

OpenClaw 的命令行工具有清晰的分层：

```
openclaw update（上层）→ 有版本通道、依赖解析、回滚逻辑
        ↓
npm install -g（底层）→ 纯粹的包安装，不了解 OpenClaw 的业务逻辑
```

Agent 1 在上层命令已经完成的情况下，跳到底层执行操作。这相当于：你用应用的"更新"按钮已经完成了更新，然后又去手动替换应用的底层文件。

---

**根因 3：不回退，只前冲**

当 `npm update -g` 搞坏了安装后，正确的做法是：**先恢复原状**。

具体来说：
- v2026.4.8 的安装在 `npm update -g` 之前是完好的
- Gateway 甚至还在正常运行
- 应该先想办法恢复到 v2026.4.8 的完好状态，而不是继续用 npm 命令强行推进到 v2026.4.23

但 Agent 1（以及后来的 Agent 2）都选择了"继续向前修"，而不是"先退回来"。

---

**根因 4：重复同一策略，期待不同结果**

`npm install -g openclaw` 第一次失败时（安装完成但文件缺失），这已经是一个明确的信号：这种安装方式产生的结果不符合预期。

Agent 1 只是在这条命令上加了不同的前缀（`npm cache clean`）或后缀（`--force`），但从未质疑这条命令本身是否是正确的修复路径。

用 Agent 3 的原话概括：**"npm install 静默失败了多次，Agent 只是加不同 flag 重试，从未质疑'也许 npm install 不是正确方式'。"**

---

**根因 5：上下文继承策略不继承教训**

当 Agent 2 接手时，它看到了 Agent 1 的操作历史。但它从历史中提取的信息是"Agent 1 在做 npm install"，而不是"Agent 1 做 npm install 失败了多次"。

它继承了**策略**，但没有继承**教训**。

---

### 第六阶段：修复与恢复（约 00:15 - 00:40）

**参与者**：Agent 3

完成根因分析后，Agent 3 开始实际的修复工作。与前两个 Agent 不同，它先做了充分的现状评估，然后才动手。

#### 诊断：搞清楚到底什么坏了，什么没坏

Agent 3 首先厘清了一个关键事实：

1. **LaunchAgent plist 文件**直接使用 `node dist/index.js gateway` 启动 Gateway——这意味着 Gateway 的启动不依赖 CLI 入口（openclaw.mjs）
2. **dist/index.js 存在**——核心代码完好
3. **Gateway 进程（pid 4295）实际上仍在正常运行**——这是 Agent 1 在第一阶段成功启动的那个实例
4. **真正坏掉的只是 CLI 入口**——package.json 和 openclaw.mjs 缺失，导致 `openclaw` 命令无法使用

这个诊断结论改变了修复策略的优先级：问题的严重程度比想象中低，Gateway 其实没有中断。

#### 修复步骤

Agent 3 执行了以下修复：

**1. 彻底清理**

```bash
npm uninstall -g openclaw
```

先完全卸载，确保不留下任何残余状态。

**2. 重新安装**

```bash
npm install -g openclaw
```

这次，Agent 3 等待安装完全完成（前两个 Agent 可能因为超时或其他原因中断了安装过程）。

**3. 验证安装完整性**

安装完成后，检查三个关键文件：
- `package.json` —— 存在 ✅
- `openclaw.mjs` —— 存在 ✅
- `dist/index.js` —— 存在 ✅

**4. 版本确认**

```bash
openclaw --version
```

输出：`OpenClaw 2026.4.23` ✅

CLI 恢复了。而且这次安装的是最新版 2026.4.23——用户最初的升级需求也一并满足了。

#### 重启 Gateway

**5. 第一次重启尝试——超时**

```bash
openclaw gateway restart
```

命令执行 60 秒后超时。

**6. 排查超时原因**

Agent 3 检查后发现：`extensions/openclaw-weixin` 目录仍然残留在系统中。虽然用户已经在配置文件中移除了微信渠道，但扩展目录还在，Gateway 启动时自动发现这些扩展并尝试加载，导致报错和超时。

**7. 清理残留扩展**

```bash
# 将残留的微信扩展目录移到 Trash（而不是直接删除）
mv extensions/openclaw-weixin ~/.Trash/
```

**8. 重写 LaunchAgent 配置**

```bash
openclaw gateway install
```

重新生成 LaunchAgent plist 文件，确保配置与新版本匹配。

**9. 启动 Gateway**

```bash
openclaw gateway start
```

首次启动时，Gateway 需要安装 bundled plugin 的依赖（包括 playwright 等），耗时 62.4 秒。

最终输出：

```
[gateway] ready (5 plugins..., 62.4s)
```

Gateway 成功启动。✅

#### 最终验证

| 验证项 | 结果 |
|--------|------|
| OpenClaw 版本 | v2026.4.23（最新） ✅ |
| Gateway 状态 | running，reachable ✅ |
| 监听端口 | 18789 ✅ |
| Channels 列表 | 空（微信已移除） ✅ |
| Security audit | 1 warning（反向代理相关，属正常提示） ✅ |

**用户最初的两个需求——升级到最新版、去除微信频道——至此全部完成。**

---

### 第七阶段：知识沉淀——第一版 Skill（约 00:40 - 00:45）

**参与者**：用户 + Agent 3

#### 用户的第三次关键干预

故障修复完成后，用户没有就此结束，而是提出了更进一步的要求：

> "更重要的是反思为什么会出问题，系统化得解决，可以做一个 skill，安装到你本地"

这句话将一次**事件修复**提升为了**知识沉淀**。用户要求将这次事件中的经验教训固化为一个可复用的 Skill，安装到 QoderWork 平台，让未来的 Agent 在遇到类似问题时不再重蹈覆辙。

#### 第一版 Skill

Agent 3 编写了第一版 `openclaw-ops` skill，共 148 行，安装到 `~/.qoderwork/skills/openclaw-ops/SKILL.md`。

内容包括：
- 5 条核心操作规则
- 常见操作流程（启动、停止、升级、配置修改）
- 灾难恢复步骤
- 故障排查指南

---

### 第八阶段：排除网络因素（约 00:45 - 00:48）

**参与者**：用户 + Agent 3

#### 用户的追问

> "是网络问题吗？比如没有开 vpn"

用户在考虑另一种可能性：npm install 反复失败是否因为网络问题（比如在中国大陆访问 npm registry 需要代理）。

#### Agent 3 的验证

```bash
curl https://registry.npmjs.org/openclaw
# 返回正常的 JSON 响应

npm config get proxy
# 返回 null（未配置代理）

npm config get registry
# 返回 https://registry.npmjs.org/（默认官方源）
```

**结论**：不是网络问题。npm registry 访问正常，没有配置代理也能连通。

那么 npm install 反复产生不完整安装的原因是什么？Agent 3 的分析是：**Agent 反复中断 npm install 导致安装不完整**。前两个 Agent 在执行 `npm install -g openclaw` 时，可能因为命令执行超时（工具默认超时时间有限）而中断了安装过程，但 npm 已经解压了部分文件（如 dist/），剩余文件（如 package.json、openclaw.mjs）还没写入就被终止了。之后的重复安装因为部分文件已存在，npm 可能跳过了完整解压步骤。

---

### 第九阶段：用 AI 原生思维重新设计 Skill（约 00:48 - 00:55）

**参与者**：用户 + Agent 3

#### 用户的第四次关键干预

> "openclaw-ops skill 你需要好好设计，做好边界情况，以及问题解决引导，因为毕竟在一个 skill 里没办法穷尽所有问题以及解决方案，用面向 ai 的思维再审视完善下这个 skill"

这次干预的核心在"面向 ai 的思维"这几个字。用户指出第一版 Skill 的问题：它是写给人看的操作手册，但它的读者不是人，是 AI Agent。

AI Agent 的阅读和执行模式与人类不同：
- 人类会用常识填补说明书的空白，AI 不会
- 人类在操作出错时会自然地停下来想一想，AI 倾向于继续执行
- 人类能理解"这只是信息，不是指令"，AI 需要被明确告知

因此，Skill 需要从"人类手册"升级为"AI 决策引擎"。

#### 第二版 Skill 的核心重构

Agent 3 重写了 Skill，从 148 行扩展到 387 行。核心变化如下：

**1. Phase 0：强制状态诊断**

任何操作之前，Agent 必须先完成 5 个维度的状态采集：

| 维度 | 检查命令 | 期望信息 |
|------|---------|---------|
| CLI 可用性 | `which openclaw` | 符号链接是否存在 |
| Gateway 状态 | `openclaw status` | 进程是否运行、端口是否监听 |
| 端口占用 | `lsof -i :18789` | 谁在占用 Gateway 端口 |
| 版本信息 | `openclaw --version` | 当前安装版本 |
| 包完整性 | `ls node_modules/openclaw/` | 关键文件是否齐全 |

**不完成 Phase 0，不允许执行任何操作。** 这直接对应了 Agent 1 的失误——没有在操作前充分了解当前状态。

**2. 7 个 Playbook：覆盖所有已知状态组合**

基于 Phase 0 的诊断结果，每种状态组合对应一个预定义的操作方案（Playbook）。每个 Playbook 都严格遵循 **DIAGNOSE → ACT → VERIFY** 三段结构：

- **DIAGNOSE**：确认当前状态符合此 Playbook 的前提条件
- **ACT**：执行操作
- **VERIFY**：验证操作结果是否达到预期

**3. 失败模式矩阵**

明确规定：**同一策略失败两次，必须换路。**

这是直接针对 Agent 1 和 Agent 2 的"重复 npm install"循环而设计的规则。Skill 中列出了常见的失败模式和对应的替代策略。

**4. Global Invariants：全局不变量**

任何操作完成后必须验证的条件清单（5 条，缺一不可）：

1. `openclaw --version` 返回有效版本号（CLI 入口可用）
2. `lsof -i :18789` 显示 LISTEN 进程（端口已绑定）
3. `curl http://127.0.0.1:18789/` 返回 HTTP 200（Gateway 可达）
4. `openclaw status` 显示 "Gateway ... reachable"（服务状态正常）
5. `gateway.err.log` 中无 "Invalid config" 错误（配置有效）

任意一条不满足 → 回到 Phase 0 重新诊断。

**5. "信息 vs 待办"的显式区分**

Skill 中明确标注：status 输出中的 `Update: available` 是**信息**，不是**指令**。Agent 必须在用户确认后才能执行升级操作。

**6. 扩展生命周期清理**

当移除渠道或插件时，必须清理的 3 个位置：
- 配置文件中的声明
- 扩展目录中的文件
- LaunchAgent 配置的重新生成

这是基于第六阶段中"微信扩展目录残留导致启动超时"的教训。

---

### 第十阶段：范式讨论（约 00:55 - 01:00）

**参与者**：用户 + Agent 3

#### 用户的追问

> "你现在的这套做法，思路是从哪里来的，或者说这就是 ai 时代，面向 agent 代理声明式工作范式"

用户在具体问题解决之后，将讨论拉升到了方法论层面。

#### Agent 3 的方法论溯源

Agent 3 将第二版 Skill 的设计思想追溯到了多个既有的理论和工程实践：

| 来源 | 对应的 Skill 设计 |
|------|------------------|
| **Hoare Logic**（霍尔逻辑）的前置/后置条件 | Phase 0（前置条件）和 Global Invariants（后置条件） |
| **FMEA**（故障模式与影响分析） | 失败模式矩阵和 7 个 Playbook |
| **OODA 循环**（观察-判断-决策-行动） | Phase 0 的强制 Observe 阶段 |
| **Kubernetes 声明式控制器模式** | desired state vs actual state 的对比验证 |

#### AI Agent 的 4 个根本认知缺陷

Agent 3 同时总结了这次事件中暴露的 AI Agent 认知缺陷，这些缺陷不是 OpenClaw 特有的，而是当前 AI Agent 的通用弱点：

**缺陷 1：无状态幻觉**

Agent 实例之间没有记忆传递机制，或者即使有上下文传递，也只传递了"做了什么"，没有传递"学到了什么"。Agent 2 看到 Agent 1 的操作历史，继承了策略，但丢失了教训。

**缺陷 2：把信息读成指令**

AI Agent 倾向于将所有输出中的文本都视为可操作的指令。`Update: available · npm update 2026.4.23` 这句话中包含了 `npm update` 这个看起来像命令的字符串，Agent 就把它当成了应该执行的命令。

**缺陷 3：副作用盲区**

Agent 在执行操作前不会自动评估副作用，也不会自动做备份和记录。`npm update -g` 可能破坏现有安装——这个副作用对有经验的人类工程师来说是常识，但 Agent 没有这个"直觉"。

**缺陷 4：规划-执行分离失败**

当执行过程中遇到意外（npm install 产生不完整的结果），Agent 应该回到规划阶段重新评估策略。但实际上 Agent 会继续在执行层面打转，用更多的执行尝试来解决一个需要在规划层面解决的问题。

#### "认知脚手架"（Cognitive Scaffolding）

Agent 3 将这套方法暂时命名为**"认知脚手架"**，其核心特征是：

| 特征 | 含义 |
|------|------|
| **状态显式化** | 不允许 Agent 假设任何状态，必须通过命令实际检查 |
| **决策外部化** | 将决策规则写在 Skill 中，而不是依赖 Agent 的"推理能力" |
| **验证强制化** | 每个操作后必须验证结果，不允许跳过 |
| **失败预设化** | 预先定义失败场景和对应的替代策略，而不是在失败后临时应对 |

这套方法的本质是：承认 AI Agent 有上述认知缺陷，不试图"修复"这些缺陷（那需要模型层面的改进），而是通过外部化的结构约束来补偿这些缺陷——就像脚手架不改变建筑本身，但能让施工安全进行。

---

## 三、关键数据汇总

| 指标 | 数值 |
|------|------|
| 事件起始时间 | 2026-04-25 约 23:12 |
| 事件结束时间 | 2026-04-26 约 01:00 |
| 事件总时长 | 约 1 小时 48 分钟 |
| 参与的 Agent 数量 | 3 个 |
| Agent 1 的有效操作 | 诊断、首次修复启动、执行 `openclaw update` 升级、配置修改 |
| Agent 1 的失误 | 误执行 `npm update -g`，陷入 npm install 循环 |
| Agent 2 的有效操作 | 清理残留临时文件 |
| Agent 2 的失误 | 继承 Agent 1 的错误策略，继续执行 npm install |
| Agent 3 的贡献 | 根因分析、正确修复、Skill 编写与重构 |
| npm install 失败次数（两个 Agent 合计） | 至少 6 次 |
| 用户的关键干预次数 | 4 次 |
| OpenClaw 初始版本 | v2026.3.24 |
| 中间版本（Agent 1 成功升级到） | v2026.4.8 |
| 最终版本 | v2026.4.23 |
| 产出的 Skill 文件 | `~/.qoderwork/skills/openclaw-ops/SKILL.md` |
| 第一版 Skill 行数 | 148 行 |
| 第二版 Skill 行数 | 387 行 |

---

## 四、用户在事件中的角色分析

这次事件中，用户扮演的不是传统意义上的"发出指令、等待结果"的被动角色。用户在事件的不同阶段承担了四种不同的功能：

### 1. 故障检测器

Agent 1 和 Agent 2 都没有自我检测出自己已经陷入了无效循环。它们缺乏这种元认知能力——无法站在自己之外审视自己的行为模式。

用户具备这种能力。当用户看到 Agent 在做第 4 次、第 5 次 npm install 时，用户能够识别出"这是在重复无效操作"，并果断叫停。

### 2. 策略纠偏者

用户的纠偏方式值得注意。用户没有说"你应该用 openclaw update 而不是 npm install"——那只是在战术层面给出答案。

用户问的是："为什么会出现这样的决策，agent 的思考过程是怎么样的"——这是在战略层面要求 Agent 反思。这个问题迫使 Agent 从执行模式切换到分析模式，从而产出了根因分析。

### 3. 系统架构师

将一次性的故障修复提升为可复用的知识资产（Skill），这个决策来自用户。Agent 本身不会主动做这件事——它的任务在"修好 OpenClaw"时就结束了。

用户看到的是更长远的价值：这次犯的错误不应该再犯第二次。将教训编码为 Skill，就是在为未来的 Agent 建立"免疫系统"。

### 4. 范式探索者

在具体问题解决、Skill 编写完成之后，用户继续追问背后的方法论来源。这已经超出了"解决一个技术问题"的范畴，进入了"如何系统性地与 AI Agent 协作"的范式层面。

用户的这个追问——"这就是 ai 时代，面向 agent 代理声明式工作范式"——实际上是在尝试将这次具体事件中摸索出的方法，概括为一种可迁移到其他场景的通用范式。

---

## 五、事件教训总结

### 对 AI Agent 开发者的启示

1. **Agent 需要"退出条件"**：当同一策略连续失败 N 次（建议 N=2），Agent 应该自动触发策略切换，而不是继续重试
2. **信息与指令的区分需要显式训练**：Agent 需要学会区分"系统输出中的信息"和"应该执行的操作"
3. **跨 Agent 上下文传递应包含失败记录**：当一个 Agent 接手另一个 Agent 的工作时，传递的不仅是"做了什么"，还应该包括"什么方法失败了"

### 对 AI Agent 使用者的启示

1. **监控 Agent 的行为模式**：当 Agent 开始重复相同策略时，及时干预
2. **用提问代替指令**：与其告诉 Agent "用这个命令"，不如问 "你为什么选择这个方案"，后者能触发更深层的策略调整
3. **将经验沉淀为 Skill**：一次性的修复价值有限，将教训编码为 Skill 才能产生持续价值
4. **在战术完成后做战略复盘**：问题解决后追问方法论，有助于将具体经验抽象为通用模式

### 对人机协作模式的启示

这次事件展示了一种"人类监督 + AI 执行"的协作模式，但比通常理解的更精细：

- 人类的价值不在于"知道正确答案"，而在于**识别错误模式**和**提出正确的问题**
- AI 的价值不仅在于执行操作，还在于一旦被引导到正确方向后，能够系统性地做根因分析和知识结构化
- 最有价值的人类干预不是"告诉 AI 做什么"，而是"告诉 AI 停下来想想为什么"

---

## 六、附录：事件产出物

| 产出物 | 路径 | 说明 |
|--------|------|------|
| openclaw-ops Skill（第二版） | `~/.qoderwork/skills/openclaw-ops/SKILL.md` | 387 行的 AI 决策引擎式运维 Skill |
| openclaw.json 备份 | 事件过程中由 Agent 1 创建 | 修改配置前的备份 |

---

> 本实录基于 2026-04-25 至 2026-04-26 事件的完整过程编写，如实记录了所有成功与失败的操作，未做美化处理。
