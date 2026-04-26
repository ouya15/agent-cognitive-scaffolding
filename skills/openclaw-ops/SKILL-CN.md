---
name: openclaw-ops
description: 诊断并操作本地 OpenClaw AI Agent Gateway 实例。当用户提及 OpenClaw、openclaw gateway、openclaw update，或要求启动/停止/升级/配置/排查本地 OpenClaw 时触发。适用于任何 OpenClaw 相关的运维请求。
> 【设计说明】description 是 Agent 决定是否加载此 Skill 的依据。写得越精确，Agent 越不容易误触发或漏触发。
---

## OpenClaw Operations — AI Agent Decision Engine
## OpenClaw 运维 — AI Agent 决策引擎

### Design Philosophy for AI Agents
> 【设计理念：面向 AI Agent 的设计哲学】

This skill is a decision engine, not a human manual. Every section follows the pattern:
> 【这份 Skill 是一个"决策引擎"，不是给人看的操作手册。每个操作步骤都遵循以下模式：】

1. **DIAGNOSE** — determine the current state before touching anything
> 【1. 诊断：在动手之前，先确定系统当前处于什么状态】
2. **DECIDE** — choose the correct path based on state
> 【2. 决策：根据诊断结果选择正确的操作路径】
3. **ACT** — execute with pre-checks and post-verification
> 【3. 执行：在执行前后都进行检查和验证】
4. **VERIFY** — confirm the intended outcome, detect unintended side effects
> 【4. 验证：确认操作达到了预期效果，并检测是否有副作用】

**Critical AI anti-patterns to avoid:**
> 【AI Agent 必须避免的关键反模式——这些是我们从实际事故中总结的教训：】

- Do NOT treat `openclaw status` output as a todo list. "Update available" is information, not an instruction to act.
> 【不要把 status 命令的输出当成待办清单。"Update available"（有更新可用）是一条信息，不是让你立刻执行更新的指令。这是我们事故的核心原因——Agent 1 把"有新版本"这条通知当成了"你应该马上升级"。】
- Do NOT repeat the same command after failure. Two failures on the same approach = mandatory pivot.
> 【失败后不要重复执行同一条命令。同一策略失败两次，强制换路，禁止第三次尝试。】
- Do NOT modify files before backing up the original.
> 【修改任何文件之前，必须先备份原始文件。】
- Do NOT run `npm install -g` / `npm update -g` on an existing installation. This is only for first-time install or confirmed disaster recovery.
> 【不要在已有安装的基础上执行 npm install/update -g。这仅用于首次安装或确认的灾难恢复。】

---

## Architecture Layers (know before touching)
> 【架构分层：在动手之前必须了解的系统层次结构】

```
Layer 4: Gateway process          (long-running Node server, port 18789)
> 第4层：Gateway 进程 —— 长期运行的 Node.js 服务器，监听 18789 端口
Layer 3: LaunchAgent plist        (~/Library/LaunchAgents/ai.openclaw.gateway.plist)
> 第3层：LaunchAgent 配置 —— macOS 的系统服务配置文件（plist）
Layer 2: CLI binary               (symlink: bin/openclaw → openclaw.mjs)
> 第2层：CLI 命令行工具 —— 符号链接，指向 openclaw.mjs
Layer 1: npm package files        (~/.nvm/versions/node/vX/lib/node_modules/openclaw/)
> 第1层：npm 包文件 —— 全局 node_modules 中的 OpenClaw 安装目录
Layer 0: Config + data            (~/.openclaw/)
> 第0层：配置文件和数据 —— 用户的 OpenClaw 配置目录
```

**Key invariant**: Layer 1 and Layer 4 share the same `dist/` directory. Replacing files while Gateway is running causes module-not-found crashes. Always stop Gateway before reinstalling Layer 1.
> 【关键不变量：第1层（npm 包）和第4层（Gateway 进程）共享同一个 dist/ 目录。在 Gateway 运行时替换文件会导致"找不到模块"崩溃。重装第1层之前必须先停止 Gateway。】

**Layer 3 has TWO states** — this is a common blind spot:
> 【第3层有两种状态——这是一个常见的盲区，也是我们事故后才发现的问题：】
- **Static**: plist file exists on disk (`~/Library/LaunchAgents/ai.openclaw.gateway.plist`)
> 【静态状态：plist 文件存在于磁盘上。文件在，不等于服务在运行。】
- **Dynamic**: service is registered in the active launchd domain (`launchctl print gui/$(id -u)/ai.openclaw.gateway`)
> 【动态状态：服务被注册到当前登录会话的 launchd 域中。只有在这个状态下，RunAtLoad 和 KeepAlive 才会生效。】

`openclaw gateway stop` may fall back to `launchctl bootout`, which removes the dynamic registration while leaving the static plist intact. The result: `RunAtLoad=true` is in the plist, but launchd ignores it because the service is no longer in the domain. **Always verify both states.**
> 【`openclaw gateway stop` 命令在内部失败时会回退到 `launchctl bootout`，这会将服务从 launchd 域中移除，但 plist 文件保持不变。结果是：plist 里写着"开机自启"，但 launchd 已经不认识这个服务了。所以必须同时检查文件是否存在和域是否已注册。】

---

## Phase 0: State Diagnosis (ALWAYS run first)
> 【第0阶段：状态诊断 —— 任何操作之前必须首先执行】

Before any operation, determine these five states:
> 【在执行任何操作之前，必须确定以下六个维度的状态：】

### State A: CLI availability
> 【状态 A：CLI 命令行工具是否可用】
```bash
which openclaw 2>/dev/null && openclaw --version 2>/dev/null || echo "CLI_BROKEN"
# which 检查符号链接是否存在；openclaw --version 检查命令能否正常执行；都失败则输出 CLI_BROKEN
```
- **CLI_OK**: returns version like `OpenClaw 2026.X.Y (hash)`
> 【CLI 正常：返回版本号】
- **CLI_BROKEN**: "command not found" or error
> 【CLI 损坏：找不到命令或执行报错】

### State B: Gateway process
> 【状态 B：Gateway 进程是否在运行】
```bash
ps aux | grep -i "[o]penclaw-gateway" | awk '{print $2}'
# 检查是否有 openclaw-gateway 进程，输出 PID
launchctl list 2>/dev/null | grep "ai.openclaw.gateway"
# 检查 LaunchAgent 是否被 launchd 管理
```
- **GATEWAY_RUNNING**: PID exists, LaunchAgent shows `running`
> 【Gateway 正在运行：有进程，LaunchAgent 显示运行中】
- **GATEWAY_STOPPED**: no PID, LaunchAgent shows `not loaded` or missing
> 【Gateway 已停止：没有进程，LaunchAgent 未加载或不存】

### State C: Port binding
> 【状态 C：18789 端口是否被监听】
```bash
lsof -i :18789 2>/dev/null | grep LISTEN
# 检查是否有进程在监听 18789 端口
```
- **PORT_BOUND**: a process is listening on 18789
> 【端口已绑定：有进程在监听】
- **PORT_FREE**: nothing on 18789
> 【端口空闲：没有进程在监听】

### State D: Version consistency
> 【状态 D：版本一致性 —— CLI 版本和 LaunchAgent 配置中的版本是否一致】
```bash
openclaw --version 2>/dev/null
# 获取 CLI 报告的版本号
# Then read LaunchAgent plist:
grep -o 'OpenClaw.*)' ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null || echo "PLIST_MISSING"
# 从 LaunchAgent plist 中提取版本号；文件不存在则输出 PLIST_MISSING
```
- **VERSIONS_MATCH**: CLI version == plist version (or both present and equal)
> 【版本匹配：CLI 和 plist 中的版本号一致】
- **VERSION_MISMATCH**: CLI version != plist version, or one is missing
> 【版本不匹配：版本号不同，或其中一个缺失】
- **PLIST_MISSING**: no LaunchAgent plist at all
> 【plist 完全不存在】

### State E: npm package integrity
> 【状态 E：npm 包完整性 —— 关键文件是否齐全】
```bash
PKG="~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw"
ls "$PKG/package.json" "$PKG/openclaw.mjs" "$PKG/dist/index.js" 2>/dev/null | wc -l
# 检查三个关键文件是否存在：package.json（包描述）、openclaw.mjs（CLI 入口）、dist/index.js（核心代码）
# wc -l 统计存在的文件数量
```
- **PKG_INTACT**: 3 files all present
> 【包完整：三个文件都在】
- **PKG_PARTIAL**: 1-2 files present (common: dist/ exists but package.json or openclaw.mjs missing)
> 【包部分缺失：只有1-2个文件。最常见的情况是 dist/ 存在但 package.json 或 openclaw.mjs 缺失——这是 npm 安装被中断的典型症状】
- **PKG_MISSING**: 0 files present
> 【包完全缺失：0 个文件】

### State F: LaunchAgent domain registration
> 【状态 F：LaunchAgent 是否在 launchd 域中注册 —— 这是事后发现的关键维度】
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "DOMAIN_UNLOADED"
# 检查服务是否在当前登录会话的 launchd 域中处于活动状态。如果返回"DOMAIN_UNLOADED"，说明服务被 bootout 过。
```
- **DOMAIN_LOADED**: service is registered in the active launchd domain
> 【域已加载：服务在 launchd 域中，RunAtLoad 和 KeepAlive 生效】
- **DOMAIN_UNLOADED**: plist exists but service was bootout'd (common after `openclaw gateway stop` fallback). `RunAtLoad` and `KeepAlive` are ignored until re-bootstrap.
> 【域未加载：plist 文件在，但服务被从域中移除了。常见于执行了 `openclaw gateway stop` 之后。在这种状态下，即使 plist 里写了自动启动，launchd 也不会在开机后拉起服务。】

**Combine these six states into a diagnosis label, then look up the matching playbook below.**
> 【将以上六个维度的状态组合成一个诊断标签，然后在下方找到匹配的 Playbook（操作手册）。】

---

## Playbook Decision Tree
> 【Playbook 决策树 —— 根据诊断标签选择对应的操作方案】

### Playbook P1: "CLI_OK + GATEWAY_RUNNING + PORT_BOUND + VERSIONS_MATCH + PKG_INTACT"
> 【P1：所有维度都正常 —— 系统健康状态】
**Label: HEALTHY**
> 【标签：健康】

This is the baseline good state. User requests fall into:
> 【这是正常的基线状态。用户的请求会分为以下子场景：】

#### P1a — User wants to upgrade
> 【P1a：用户要求升级】
```
ACT:
  openclaw update
  # 使用 OpenClaw 自带的升级命令。不要用 npm update！
  # Wait for completion. Do NOT interrupt.
  # 等待完成，不要中断。升级可能需要几分钟。
  # If update reports success:
  # 升级成功后：
  openclaw gateway install   # rewrite LaunchAgent plist
  # 重新生成 LaunchAgent plist 文件，确保配置与新版本匹配
  openclaw gateway restart
  # 重启 Gateway
  # Wait 60-120s for first-boot plugin deps install.
  # 首次启动可能需要 60-120 秒安装插件依赖，耐心等待。
  # Watch logs: tail -f ~/.openclaw/logs/gateway.log
  # 通过日志观察进度
  # Stop waiting when you see: [gateway] ready (N plugins..., Xs)
  # 看到 "[gateway] ready" 即表示启动完成

VERIFY:
  openclaw status | grep -E "Gateway|Update"
  lsof -i :18789
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
> 【验证：status 显示 Gateway 可达，端口监听正常，HTTP 返回 200】
```

#### P1b — User wants to remove a channel/plugin
> 【P1b：用户要求移除频道或插件】
```
PRE-CHECK:
  ls ~/.openclaw/extensions/   # note which extensions exist
> 【预检查：先看看当前有哪些扩展目录】

ACT:
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)
  # 备份配置文件（日期格式：YYYYMMDD）
  # Edit openclaw.json:
  #   - Remove matching entries from bindings[]
  #   - Remove matching keys from channels{}
  #   - Remove matching keys from plugins.entries{}
  # 编辑配置文件：从 bindings 数组、channels 对象、plugins.entries 对象中删除对应条目
  # If the extension directory still exists:
  mv ~/.openclaw/extensions/<name> ~/.Trash/<name>-backup-$(date +%Y%m%d)
  # 如果扩展目录还在，移到废纸篓（不要直接删除）
  openclaw gateway restart
  # 重启 Gateway 使配置生效

VERIFY:
  openclaw status | grep -A5 "Channels"
  # Expected: table is empty or does not list the removed channel
> 【验证：Channels 列表中没有被移除的频道】
```

#### P1c — User wants to stop Gateway
> 【P1c：用户要求停止 Gateway】

**CRITICAL**: There are two ways to stop Gateway with very different side effects.
> 【关键：有两种停止方式，副作用完全不同。这是我们从"关机后无法自启"事故中总结的重要发现。】

**Option A — Temporary stop (preserves auto-start on next login):**
> 【方案 A —— 临时停止（保留下次登录后的自动启动）：推荐用于日常关闭】
```
ACT:
  launchctl stop gui/$(id -u)/ai.openclaw.gateway
> 【只停止进程，不改变 launchd 域中的注册状态】

WHY: This only stops the process. The service stays registered in the launchd domain.
  On next user login, launchd will see the registered service with RunAtLoad=true
  and automatically start it. KeepAlive=true will also auto-restart if the process
  crashes.
> 【为什么推荐方案 A：它只杀进程，服务保留在 launchd 域中。下次登录时 launchd 会自动启动，进程崩溃时 KeepAlive 会自动重启。】

VERIFY:
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_OK" || echo "DOMAIN_LOST"
  lsof -i :18789                  # should be empty
> 【验证：域仍然已加载，端口已释放】
```

**Option B — Full stop (may remove auto-start; use only when permanently disabling):**
> 【方案 B —— 完全停止（可能移除自动启动；仅在永久禁用时使用）】
```
ACT:
  openclaw gateway stop
> 【注意：这个命令内部可能触发 bootout！】

WARNING: If launchctl stop fails internally, openclaw falls back to launchctl bootout,
  which REMOVES the service from the launchd domain. The plist file remains, but
  launchd will NOT auto-start the service on next login. The user must manually run
  `openclaw gateway start` to re-bootstrap.
> 【警告：如果内部 launchctl stop 失败，openclaw 会回退到 bootout，将服务从域中移除。plist 文件还在，但下次登录不会自启。用户需要手动执行 `openclaw gateway start` 重新注册。】

VERIFY:
  launchctl list | grep openclaw   # should show not loaded
  lsof -i :18789                  # should be empty
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "WARNING: domain still loaded" || echo "DOMAIN_UNLOADED (expected)"
> 【验证：进程停止，端口释放，域已卸载（这是预期的）】
```

#### P1d — User wants to start Gateway
> 【P1d：用户要求启动 Gateway】
```
ACT:
  openclaw gateway start
> 【启动命令。如果服务未被注册到 domain，这个命令会同时执行 bootstrap】

VERIFY:
  sleep 10
  lsof -i :18789                  # should show node LISTEN
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
> 【验证：等待 10 秒后端口已监听，HTTP 返回 200】
```

---

### Playbook P2: "CLI_BROKEN + (any Gateway/Port/Version state) + PKG_PARTIAL"
> 【P2：CLI 损坏 + 包部分缺失 —— 包损坏状态】
**Label: CORRUPT_PACKAGE**
> 【标签：包损坏】

The npm package is partially installed (common after interrupted install). Dist files may exist but package.json or openclaw.mjs is missing.
> 【npm 包安装不完整（常见于安装被中断后）。dist/ 目录可能在，但 package.json 或 openclaw.mjs 入口文件缺失。】

```
DIAGNOSE deeper:
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/
  # Look for:
  #   - .openclaw-* (temp dir residue) → sign of interrupted install
  #   - dist/ without package.json → partial extraction
  #   - package.json without openclaw.mjs → postinstall script failure
> 【深入诊断：查看包目录，寻找中断安装的痕迹——临时目录残留、只有 dist 没有 package.json、只有 package.json 没有 openclaw.mjs】

ACT:
  # Step 1: If Gateway is running, note its PID (it holds working code in memory)
  ps aux | grep "[o]penclaw-gateway"
  # 第一步：如果 Gateway 还在运行，记下它的 PID——内存中有能用的代码
  # Step 2: Clean reinstall (only for disaster recovery — this is the exception to Rule 1)
  source ~/.nvm/nvm.sh && nvm use default
  # 第二步：确保 nvm 环境正确（这是很多 npm 问题的根源）
  npm uninstall -g openclaw
  # 完全卸载
  # Verify clean:
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw 2>/dev/null && echo "STILL THERE" || echo "CLEAN"
  # 验证是否卸载干净
  # If "STILL THERE", manually remove:
  rm -rf ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw*
  rm -f ~/.nvm/versions/node/$(node -v)/bin/openclaw
  # 如果还没卸载干净，手动删除
  # Step 3: Fresh install — DO NOT interrupt, DO NOT chain other commands
  npm install -g openclaw
  # 第三步：全新安装——不要中断，不要串联其他命令
  # This is a ~13MB package with 1700+ files. It takes time.
  # 这个包约 13MB，1700+ 个文件，需要时间。
  # If bash tool times out, wait and verify separately.
  # 如果工具超时，等待后再单独验证。

VERIFY (must check ALL three):
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/package.json
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/openclaw.mjs
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/dist/index.js
  openclaw --version
  # Expected: version string, no errors
> 【验证：三个关键文件都存在，版本号正常输出。这三个缺一不可。】

POST-RECOVERY:
  openclaw gateway install
  openclaw gateway start
  # Wait for "ready" in logs
> 【恢复后：重新安装服务配置，启动 Gateway】
```

**IF reinstall fails twice** (npm install produces errors or files still missing):
> 【如果重装失败两次：】
- Pivot to: check nvm/node environment (`nvm list`, `node -v`)
> 【换路：检查 nvm/node 环境】
- Pivot to: check npm registry reachability (`curl -I https://registry.npmjs.org`)
> 【换路：检查 npm registry 网络连通性】
- Pivot to: check for stale npm cache (`npm cache clean --force`)
> 【换路：清理 npm 缓存】
- Do NOT run `npm install -g openclaw` a third time without changing the approach.
> 【绝对不要在没换路的情况下第三次执行 npm install -g openclaw。】

---

### Playbook P3: "CLI_OK + GATEWAY_RUNNING + PORT_FREE + (any PKG state)"
> 【P3：CLI 正常 + 进程在运行 + 端口空闲 —— Gateway 挂起状态】
**Label: GATEWAY_HUNG**
> 【标签：Gateway 挂起/卡住】

Gateway process exists but is not binding to the port. Common causes:
> 【Gateway 进程存在但没有绑定端口。常见原因：】
1. Startup blocked by plugin dependency installation (normal for first boot after upgrade)
> 【1. 被插件依赖安装阻塞——升级后首次启动的正常现象，需要等待】
2. Config validation error (fatal)
> 【2. 配置验证错误——致命错误，需要修复配置】
3. Version mismatch — old Gateway binary with new dist files
> 【3. 版本不匹配——旧的 Gateway 二进制和新的 dist 文件不兼容】

```
DIAGNOSE:
  tail -30 ~/.openclaw/logs/gateway.err.log
  tail -30 ~/.openclaw/logs/gateway.log
> 【诊断：查看最近 30 行错误日志和运行日志】

ACT based on error:
> 【根据错误类型选择操作：】
  
  If error log shows "Invalid config" with plugin version mismatches:
    → Remove stale extension dirs from ~/.openclaw/extensions/
    → openclaw gateway restart
  # 如果日志显示"Invalid config"且伴随插件版本不匹配：删除残留的扩展目录，然后重启。
    
  If log shows "[plugins] installed bundled runtime deps" but nothing after:
    → This is normal first-boot behavior. WAIT 60-120s more.
    → Do NOT restart. Check again with `tail -f`.
  # 如果日志显示"安装了 bundled 依赖"之后就没有动静了：这是首次启动的正常行为，再等 60-120 秒。不要重启！用 tail -f 继续观察。
    
  If log shows "Cannot find module ..." with hashed filenames:
    → dist/ files were replaced while Gateway was running (Layer 1/Layer 4 collision)
    → openclaw gateway stop
    → openclaw gateway install   # rewrite plist with correct paths
    → openclaw gateway start
  # 如果日志显示"找不到模块"加哈希文件名：说明 Gateway 运行时 dist/ 被替换了（第1层和第4层冲突）。停止→重写配置→启动。
    
  If log shows no errors but Gateway just says "starting..." forever:
    → openclaw gateway stop
    → openclaw gateway install
    → openclaw gateway start
    → Wait with `tail -f`.
  # 如果没有错误但 Gateway 永远只显示"starting..."：停止→重写配置→启动→用 tail -f 观察。

VERIFY:
  After restart, wait up to 120s, then:
  lsof -i :18789
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
> 【验证：重启后最多等待 120 秒，端口应已监听，HTTP 返回 200】
```

---

### Playbook P4: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + VERSION_MISMATCH + PKG_INTACT"
> 【P4：CLI 正常 + 进程停止 + 版本不匹配 + 包完整 —— 过期 LaunchAgent】
**Label: STALE_LAUNCHAGENT**
> 【标签：LaunchAgent 配置过期】

CLI is on a new version but LaunchAgent plist still points to old paths. This happens after `openclaw update` without `openclaw gateway install`.
> 【CLI 已经是新版本，但 LaunchAgent plist 还指向旧路径。通常发生在 `openclaw update` 后没有执行 `openclaw gateway install`。】

```
ACT:
  openclaw gateway install   # rewrites plist
  openclaw gateway start
> 【操作：重写 plist 文件使其指向新版本的正确路径，然后启动】

VERIFY:
  grep "dist/index.js" ~/Library/LaunchAgents/ai.openclaw.gateway.plist
  # Should point to current node version's path
  lsof -i :18789
> 【验证：plist 中的路径对应当前 node 版本的路径，端口正常监听】
```

---

### Playbook P5: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + PLIST_MISSING + PKG_INTACT"
> 【P5：CLI 正常 + 进程停止 + plist 不存在 + 包完整 —— 无服务注册】
**Label: NO_SERVICE**
> 【标签：服务未注册】

OpenClaw is installed but LaunchAgent was never created (or was deleted).
> 【OpenClaw 已正确安装，但 LaunchAgent 从未创建（或被删除了）。】

```
ACT:
  openclaw gateway install
  openclaw gateway start
> 【操作：安装 LaunchAgent 服务配置，然后启动】

VERIFY:
  launchctl list | grep openclaw
  lsof -i :18789
> 【验证：launchd 中存在 openclaw 服务，端口正常监听】
```

---

### Playbook P6: "CLI_OK + GATEWAY_STOPPED + PORT_BOUND + (any other state)"
> 【P6：CLI 正常 + 进程停止 + 端口被占用 —— 端口冲突】
**Label: PORT_CONFLICT**
> 【标签：端口冲突】

Port 18789 is occupied by something else (another OpenClaw instance, or another service).
> 【18789 端口被其他进程占用（可能是另一个 OpenClaw 实例，或其他服务）。】

```
DIAGNOSE:
  lsof -i :18789   # find the process
  # If it's another openclaw-gateway process:
  ps aux | grep "[o]penclaw-gateway"
> 【诊断：找出谁在占用端口，判断是否是一个残留的 openclaw-gateway 进程】

ACT:
  # Kill the stale process
  kill -9 <PID>
  # Then start fresh:
  openclaw gateway start
> 【操作：杀掉残留进程，然后干净地启动】

VERIFY:
  lsof -i :18789 | grep LISTEN
  # Should show the new Gateway PID
> 【验证：端口由新的 Gateway 进程监听】
```

---

### Playbook P7: "any state + Config edit requested"
> 【P7：任何状态 + 用户要求编辑配置 —— 配置操作】
**Label: CONFIG_OPERATION**
> 【标签：配置操作】

```
MANDATORY PRE-CHECK:
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)
> 【强制预检查：备份配置文件（这是不可跳过的）】

ACT:
  # Edit the file. Rules for specific edits:
  # - Remove channel: delete from bindings[], channels{}, AND plugins.entries{}
  # - Remove plugin: delete from plugins.entries{}, AND move extension dir to trash
  # - Change model provider: update in models.providers{}
  # - Never leave dangling references (e.g., a binding pointing to a deleted channel)
> 【操作：编辑文件。规则：移除频道时同时删除 bindings、channels、plugins.entries 中的条目；移除插件时同时删除 entries 和扩展目录；修改模型提供商时更新 providers。永远不要留下悬空引用（比如指向已删除频道的绑定）。】

POST-EDIT VERIFY:
  # Validate JSON syntax:
  node -e "JSON.parse(require('fs').readFileSync(process.env.HOME+'/.openclaw/openclaw.json'))" && echo "JSON_VALID" || echo "JSON_INVALID"
  # If invalid, restore from backup immediately.
> 【编辑后验证：检查 JSON 语法是否合法。如果不合法，立即从备份恢复。】

APPLY:
  openclaw gateway restart
  # If restart fails with "Invalid config", check error log for the specific key,
  # fix it, and restart again.
> 【应用：重启 Gateway。如果重启失败并提示"Invalid config"，查看错误日志找到具体的 key，修复后再次重启。】
```

---

### Playbook P8: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + PLIST_EXISTS + VERSIONS_MATCH + PKG_INTACT + DOMAIN_UNLOADED"
> 【P8：一切看起来都正常，但服务不在 launchd 域中 —— 域注销状态】
**Label: DOMAIN_DEREGISTERED**
> 【标签：域注销】

The LaunchAgent plist exists and is correctly configured, but the service has been removed from the active launchd domain (typically by `launchctl bootout` during `openclaw gateway stop` fallback). The user may report "Gateway doesn't auto-start after reboot/login" even though everything looks correct.
> 【LaunchAgent plist 存在且配置正确，但服务已被从 launchd 域中移除（通常是 `openclaw gateway stop` 回退到 bootout 导致的）。用户可能反馈"重启/登录后 Gateway 没有自动启动"，尽管所有配置看起来都正常。这是我们实际经历的真实场景。】

```
DIAGNOSE:
  # Confirm the mismatch: plist exists but domain is empty
  ls ~/Library/LaunchAgents/ai.openclaw.gateway.plist && echo "PLIST_EXISTS" || echo "PLIST_MISSING"
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "DOMAIN_UNLOADED"
  # Expected: PLIST_EXISTS + DOMAIN_UNLOADED
> 【诊断：确认"文件存在但域为空"的不匹配状态】

ACT:
  openclaw gateway start
  # This re-bootstraps the service into the launchd domain.
  # The output should say something like: "re-bootstrapped launchd service"
> 【操作：执行 start 命令，它会自动重新 bootstrap 服务到 launchd 域。输出会显示"re-bootstrapped launchd service"。】

VERIFY:
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "STILL_UNLOADED"
  ps aux | grep -i "[o]penclaw-gateway" | awk '{print $2}'
  lsof -i :18789 2>/dev/null | grep LISTEN
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: DOMAIN_LOADED + PID exists + LISTEN + 200
> 【验证：域已加载 + 进程存在 + 端口监听 + HTTP 200，四项缺一不可】
```

---

## Global Invariants (must hold after any operation)
> 【全局不变量 —— 任何操作完成后必须验证的条件】

After ANY start/restart/upgrade operation, verify these before declaring success:
> 【任何启动/重启/升级操作完成后，在宣布"成功"之前，必须验证以下条件：】

```
1. openclaw --version returns a version string (no errors)
> 1. 版本号正常输出（无报错）—— CLI 入口可用
2. lsof -i :18789 shows a LISTEN process
> 2. 端口 18789 有 LISTEN 进程 —— 端口已绑定
3. curl http://127.0.0.1:18789/ returns HTTP 200
> 3. HTTP 返回 200 —— Gateway 可达
4. openclaw status shows "Gateway ... reachable"
> 4. status 显示 "Gateway ... reachable" —— 服务状态正常
5. No "Invalid config" in gateway.err.log
> 5. 错误日志中没有 "Invalid config" —— 配置有效
6. launchctl print gui/$(id -u)/ai.openclaw.gateway shows "state = running" or "state = sleeping"
   (NOT "SERVICE_NOT_IN_DOMAIN" — that indicates bootout happened)
> 6. launchd 域中服务状态为 running 或 sleeping（不是 SERVICE_NOT_IN_DOMAIN——那说明发生了 bootout）
```

If any invariant fails, do not declare success. Go back to State Diagnosis (Phase 0) and pick the matching playbook.
> 【任何不变量不满足，都不能宣布"成功"。回到 Phase 0 重新诊断，选择匹配的 Playbook。】

**Special case for stop operations**: Invariant 6 may intentionally show DOMAIN_UNLOADED if the user requested a permanent stop via `openclaw gateway stop`. In that case, verify invariants 1-5 only, and note the domain state separately.
> 【停止操作的特殊情况：如果用户通过 `openclaw gateway stop` 请求永久停止，不变量 6 可能会故意显示 DOMAIN_UNLOADED。这种情况下只需验证 1-5，并单独记录域状态。】

---

## Common Failure Patterns and Mandatory Pivots
> 【常见失败模式和强制换路规则】

| Failure Pattern | Count | Mandatory Pivot |
> | 失败模式 | 失败次数阈值 | 强制换路操作 |
| `npm install -g openclaw` → files still missing | 2 | Stop. Check `node -v`, `npm -v`, npm registry reachability, nvm env. Do not retry install. |
> | npm install 后文件仍然缺失 → 失败2次 | 停止。检查 node/npm 版本、npm registry 连通性、nvm 环境。不要再重试安装。 |
| `openclaw gateway restart` → port never bound | 2 | Stop. Check `tail -30 ~/.openclaw/logs/gateway.err.log`. Likely config error or version mismatch. |
> | restart 后端口从未绑定 → 失败2次 | 停止。查看错误日志。可能是配置错误或版本不匹配。 |
| `openclaw update` → same version after completion | 1 | Check `openclaw status` Update line. The version on your channel may already be latest. This is not a failure. |
> | update 后版本号不变 → 失败1次 | 检查 status 中的 Update 行。你所在通道的版本可能已经是最新。这不是失败。 |
| Config edit → "Invalid config" on restart | 1 | Restore from backup, re-read the error message carefully, fix the specific key, restart. |
> | 编辑配置后重启提示"Invalid config" → 失败1次 | 从备份恢复，仔细阅读错误信息，修复具体的 key，再次重启。 |
| `openclaw status` shows "Update available" | 0 (do not act) | This is INFORMATION, not a TODO. Only upgrade if user explicitly asked. |
> | status 显示"Update available" → 0 次（不要行动） | 这是信息，不是待办。只有用户明确要求时才升级。 |
| `openclaw gateway stop` → service no longer auto-starts after reboot | 1 | Re-bootstrap with `openclaw gateway start`. For temporary stops, prefer `launchctl stop gui/$(id -u)/ai.openclaw.gateway` to avoid bootout. |
> | stop 后重启不自启 → 失败1次 | 用 `openclaw gateway start` 重新 bootstrap。临时停止时优先用 `launchctl stop` 避免 bootout。 |
| User reports "Gateway won't start after login" but plist exists and `openclaw status` looks OK | 1 | Check State F (domain registration). Likely P8 DOMAIN_DEREGISTERED. |
> | 用户反馈"登录后不自启"但配置看起来正常 → 失败1次 | 检查状态 F（域注册）。很可能是 P8 DOMAIN_DEREGISTERED。 |

---

## Extension Lifecycle (channels and plugins)
> 【扩展生命周期管理——移除频道或插件时必须清理的所有位置】

When removing a channel or plugin, you MUST clean up ALL of these locations. Leaving any behind causes auto-discovery errors:
> 【移除频道或插件时，必须清理以下所有位置。留下任何一个都会导致自动发现错误：】

```
1. ~/.openclaw/openclaw.json:
   - bindings[] entries referencing the channel
   # 配置文件中：引用该频道的 bindings 数组条目
   - channels{} key for the channel
   # channels 对象中对应的 key
   - plugins.entries{} key for the plugin
   # plugins.entries 对象中对应的 key
   - plugins.installs{} record for the plugin (optional but recommended)
   # plugins.installs 中的安装记录（可选但推荐清理）

2. ~/.openclaw/extensions/:
   - Directory matching the plugin name
   # 扩展目录：与插件名称匹配的目录
   - Move to trash, never rm -rf
   # 移到废纸篓，绝对不要 rm -rf 直接删除

3. ~/.openclaw/agents/*/sessions/:
   - Old session files may reference the channel (harmless but can be cleaned)
   # Agent 会话文件：旧会话文件可能引用了该频道（无害但可清理）
```

After cleanup, restart Gateway. If you see "plugins.allow is empty; discovered non-bundled plugins may auto-load" in logs, that's a warning, not an error. To silence it, set `plugins.allow` to an explicit list in config.
> 【清理完成后，重启 Gateway。如果日志中出现"plugins.allow is empty"，这是一个警告（不是错误）。要消除它，在配置中设置 plugins.allow 为明确的插件列表。】
