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

```
Layer 4: Gateway process          (long-running Node server, port 18789)
Layer 3: LaunchAgent plist        (~/Library/LaunchAgents/ai.openclaw.gateway.plist)
Layer 2: CLI binary               (symlink: bin/openclaw → openclaw.mjs)
Layer 1: npm package files        (~/.nvm/versions/node/vX/lib/node_modules/openclaw/)
Layer 0: Config + data            (~/.openclaw/)
```

**Key invariant**: Layer 1 and Layer 4 share the same `dist/` directory. Replacing files while Gateway is running causes module-not-found crashes. Always stop Gateway before reinstalling Layer 1.

**Layer 3 has TWO states** — this is a common blind spot:
- **Static**: plist file exists on disk (`~/Library/LaunchAgents/ai.openclaw.gateway.plist`)
- **Dynamic**: service is registered in the active launchd domain (`launchctl print gui/$(id -u)/ai.openclaw.gateway`)

`openclaw gateway stop` may fall back to `launchctl bootout`, which removes the dynamic registration while leaving the static plist intact. The result: `RunAtLoad=true` is in the plist, but launchd ignores it because the service is no longer in the domain. **Always verify both states.**

---

## Phase 0: State Diagnosis (ALWAYS run first)

Before any operation, determine these five states:

### State A: CLI availability
```bash
which openclaw 2>/dev/null && openclaw --version 2>/dev/null || echo "CLI_BROKEN"
```
- **CLI_OK**: returns version like `OpenClaw 2026.X.Y (hash)`
- **CLI_BROKEN**: "command not found" or error

### State B: Gateway process
```bash
ps aux | grep -i "[o]penclaw-gateway" | awk '{print $2}'
launchctl list 2>/dev/null | grep "ai.openclaw.gateway"
```
- **GATEWAY_RUNNING**: PID exists, LaunchAgent shows `running`
- **GATEWAY_STOPPED**: no PID, LaunchAgent shows `not loaded` or missing

### State C: Port binding
```bash
lsof -i :18789 2>/dev/null | grep LISTEN
```
- **PORT_BOUND**: a process is listening on 18789
- **PORT_FREE**: nothing on 18789

### State D: Version consistency
```bash
openclaw --version 2>/dev/null
# Then read LaunchAgent plist:
grep -o 'OpenClaw.*)' ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null || echo "PLIST_MISSING"
```
- **VERSIONS_MATCH**: CLI version == plist version (or both present and equal)
- **VERSION_MISMATCH**: CLI version != plist version, or one is missing
- **PLIST_MISSING**: no LaunchAgent plist at all

### State E: npm package integrity
```bash
PKG="~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw"
ls "$PKG/package.json" "$PKG/openclaw.mjs" "$PKG/dist/index.js" 2>/dev/null | wc -l
```
- **PKG_INTACT**: 3 files all present
- **PKG_PARTIAL**: 1-2 files present (common: dist/ exists but package.json or openclaw.mjs missing)
- **PKG_MISSING**: 0 files present

### State F: LaunchAgent domain registration
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "DOMAIN_UNLOADED"
```
- **DOMAIN_LOADED**: service is registered in the active launchd domain
- **DOMAIN_UNLOADED**: plist exists but service was bootout'd (common after `openclaw gateway stop` fallback). `RunAtLoad` and `KeepAlive` are ignored until re-bootstrap.

**Combine these six states into a diagnosis label, then look up the matching playbook below.**

---

## Playbook Decision Tree

### Playbook P1: "CLI_OK + GATEWAY_RUNNING + PORT_BOUND + VERSIONS_MATCH + PKG_INTACT"
**Label: HEALTHY**

This is the baseline good state. User requests fall into:

#### P1a — User wants to upgrade
```
ACT:
  openclaw update
  # Wait for completion. Do NOT interrupt.
  # If update reports success:
  openclaw gateway install   # rewrite LaunchAgent plist
  openclaw gateway restart
  # Wait 60-120s for first-boot plugin deps install.
  # Watch logs: tail -f ~/.openclaw/logs/gateway.log
  # Stop waiting when you see: [gateway] ready (N plugins..., Xs)

VERIFY:
  openclaw status | grep -E "Gateway|Update"
  lsof -i :18789
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
```

#### P1b — User wants to remove a channel/plugin
```
PRE-CHECK:
  ls ~/.openclaw/extensions/   # note which extensions exist

ACT:
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)
  # Edit openclaw.json:
  #   - Remove matching entries from bindings[]
  #   - Remove matching keys from channels{}
  #   - Remove matching keys from plugins.entries{}
  # If the extension directory still exists:
  mv ~/.openclaw/extensions/<name> ~/.Trash/<name>-backup-$(date +%Y%m%d)
  openclaw gateway restart

VERIFY:
  openclaw status | grep -A5 "Channels"
  # Expected: table is empty or does not list the removed channel
```

#### P1c — User wants to stop Gateway

**CRITICAL**: There are two ways to stop Gateway with very different side effects.

**Option A — Temporary stop (preserves auto-start on next login):**
```
ACT:
  launchctl stop gui/$(id -u)/ai.openclaw.gateway

WHY: This only stops the process. The service stays registered in the launchd domain.
  On next user login, launchd will see the registered service with RunAtLoad=true
  and automatically start it. KeepAlive=true will also auto-restart if the process
  crashes.

VERIFY:
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_OK" || echo "DOMAIN_LOST"
  lsof -i :18789                  # should be empty
```

**Option B — Full stop (may remove auto-start; use only when permanently disabling):**
```
ACT:
  openclaw gateway stop

WARNING: If launchctl stop fails internally, openclaw falls back to launchctl bootout,
  which REMOVES the service from the launchd domain. The plist file remains, but
  launchd will NOT auto-start the service on next login. The user must manually run
  `openclaw gateway start` to re-bootstrap.

VERIFY:
  launchctl list | grep openclaw   # should show not loaded
  lsof -i :18789                  # should be empty
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "WARNING: domain still loaded" || echo "DOMAIN_UNLOADED (expected)"
```

#### P1d — User wants to start Gateway
```
ACT:
  openclaw gateway start

VERIFY:
  sleep 10
  lsof -i :18789                  # should show node LISTEN
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
```

---

### Playbook P2: "CLI_BROKEN + (any Gateway/Port/Version state) + PKG_PARTIAL"
**Label: CORRUPT_PACKAGE**

The npm package is partially installed (common after interrupted install). Dist files may exist but package.json or openclaw.mjs is missing.

```
DIAGNOSE deeper:
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/
  # Look for:
  #   - .openclaw-* (temp dir residue) → sign of interrupted install
  #   - dist/ without package.json → partial extraction
  #   - package.json without openclaw.mjs → postinstall script failure

ACT:
  # Step 1: If Gateway is running, note its PID (it holds working code in memory)
  ps aux | grep "[o]penclaw-gateway"
  
  # Step 2: Clean reinstall (only for disaster recovery — this is the exception to Rule 1)
  source ~/.nvm/nvm.sh && nvm use default
  npm uninstall -g openclaw
  # Verify clean:
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw 2>/dev/null && echo "STILL THERE" || echo "CLEAN"
  # If "STILL THERE", manually remove:
  rm -rf ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw*
  rm -f ~/.nvm/versions/node/$(node -v)/bin/openclaw
  
  # Step 3: Fresh install — DO NOT interrupt, DO NOT chain other commands
  npm install -g openclaw
  # This is a ~13MB package with 1700+ files. It takes time.
  # If bash tool times out, wait and verify separately.

VERIFY (must check ALL three):
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/package.json
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/openclaw.mjs
  ls ~/.nvm/versions/node/$(node -v)/lib/node_modules/openclaw/dist/index.js
  openclaw --version
  # Expected: version string, no errors

POST-RECOVERY:
  openclaw gateway install
  openclaw gateway start
  # Wait for "ready" in logs
```

**IF reinstall fails twice** (npm install produces errors or files still missing):
- Pivot to: check nvm/node environment (`nvm list`, `node -v`)
- Pivot to: check npm registry reachability (`curl -I https://registry.npmjs.org`)
- Pivot to: check for stale npm cache (`npm cache clean --force`)
- Do NOT run `npm install -g openclaw` a third time without changing the approach.

---

### Playbook P3: "CLI_OK + GATEWAY_RUNNING + PORT_FREE + (any PKG state)"
**Label: GATEWAY_HUNG**

Gateway process exists but is not binding to the port. Common causes:
1. Startup blocked by plugin dependency installation (normal for first boot after upgrade)
2. Config validation error (fatal)
3. Version mismatch — old Gateway binary with new dist files

```
DIAGNOSE:
  tail -30 ~/.openclaw/logs/gateway.err.log
  tail -30 ~/.openclaw/logs/gateway.log

ACT based on error:
  
  If error log shows "Invalid config" with plugin version mismatches:
    → Remove stale extension dirs from ~/.openclaw/extensions/
    → openclaw gateway restart
    
  If log shows "[plugins] installed bundled runtime deps" but nothing after:
    → This is normal first-boot behavior. WAIT 60-120s more.
    → Do NOT restart. Check again with `tail -f`.
    
  If log shows "Cannot find module ..." with hashed filenames:
    → dist/ files were replaced while Gateway was running (Layer 1/Layer 4 collision)
    → openclaw gateway stop
    → openclaw gateway install   # rewrite plist with correct paths
    → openclaw gateway start
    
  If log shows no errors but Gateway just says "starting..." forever:
    → openclaw gateway stop
    → openclaw gateway install
    → openclaw gateway start
    → Wait with `tail -f`.

VERIFY:
  After restart, wait up to 120s, then:
  lsof -i :18789
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: 200
```

---

### Playbook P4: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + VERSION_MISMATCH + PKG_INTACT"
**Label: STALE_LAUNCHAGENT**

CLI is on a new version but LaunchAgent plist still points to old paths. This happens after `openclaw update` without `openclaw gateway install`.

```
ACT:
  openclaw gateway install   # rewrites plist
  openclaw gateway start

VERIFY:
  grep "dist/index.js" ~/Library/LaunchAgents/ai.openclaw.gateway.plist
  # Should point to current node version's path
  lsof -i :18789
```

---

### Playbook P5: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + PLIST_MISSING + PKG_INTACT"
**Label: NO_SERVICE**

OpenClaw is installed but LaunchAgent was never created (or was deleted).

```
ACT:
  openclaw gateway install
  openclaw gateway start

VERIFY:
  launchctl list | grep openclaw
  lsof -i :18789
```

---

### Playbook P6: "CLI_OK + GATEWAY_STOPPED + PORT_BOUND + (any other state)"
**Label: PORT_CONFLICT**

Port 18789 is occupied by something else (another OpenClaw instance, or another service).

```
DIAGNOSE:
  lsof -i :18789   # find the process
  # If it's another openclaw-gateway process:
  ps aux | grep "[o]penclaw-gateway"

ACT:
  # Kill the stale process
  kill -9 <PID>
  # Then start fresh:
  openclaw gateway start

VERIFY:
  lsof -i :18789 | grep LISTEN
  # Should show the new Gateway PID
```

---

### Playbook P7: "any state + Config edit requested"
**Label: CONFIG_OPERATION**

```
MANDATORY PRE-CHECK:
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)

ACT:
  # Edit the file. Rules for specific edits:
  # - Remove channel: delete from bindings[], channels{}, AND plugins.entries{}
  # - Remove plugin: delete from plugins.entries{}, AND move extension dir to trash
  # - Change model provider: update in models.providers{}
  # - Never leave dangling references (e.g., a binding pointing to a deleted channel)

POST-EDIT VERIFY:
  # Validate JSON syntax:
  node -e "JSON.parse(require('fs').readFileSync(process.env.HOME+'/.openclaw/openclaw.json'))" && echo "JSON_VALID" || echo "JSON_INVALID"
  # If invalid, restore from backup immediately.

APPLY:
  openclaw gateway restart
  # If restart fails with "Invalid config", check error log for the specific key,
  # fix it, and restart again.
```

---

### Playbook P8: "CLI_OK + GATEWAY_STOPPED + PORT_FREE + PLIST_EXISTS + VERSIONS_MATCH + PKG_INTACT + DOMAIN_UNLOADED"
**Label: DOMAIN_DEREGISTERED**

The LaunchAgent plist exists and is correctly configured, but the service has been removed from the active launchd domain (typically by `launchctl bootout` during `openclaw gateway stop` fallback). The user may report "Gateway doesn't auto-start after reboot/login" even though everything looks correct.

```
DIAGNOSE:
  # Confirm the mismatch: plist exists but domain is empty
  ls ~/Library/LaunchAgents/ai.openclaw.gateway.plist && echo "PLIST_EXISTS" || echo "PLIST_MISSING"
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "DOMAIN_UNLOADED"
  # Expected: PLIST_EXISTS + DOMAIN_UNLOADED

ACT:
  openclaw gateway start
  # This re-bootstraps the service into the launchd domain.
  # The output should say something like: "re-bootstrapped launchd service"

VERIFY:
  launchctl print gui/$(id -u)/ai.openclaw.gateway 2>/dev/null | grep -q "state =" && echo "DOMAIN_LOADED" || echo "STILL_UNLOADED"
  ps aux | grep -i "[o]penclaw-gateway" | awk '{print $2}'
  lsof -i :18789 2>/dev/null | grep LISTEN
  curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
  Expected: DOMAIN_LOADED + PID exists + LISTEN + 200
```

---

### Playbook P9: "User wants to install an external channel/plugin (e.g., WeChat)"
**Label: EXTERNAL_PLUGIN_INSTALL**

OpenClaw supports third-party plugins (e.g., WeChat via `@tencent-weixin/openclaw-weixin`) that are not in the stock plugin list. These require explicit install and QR-code login.

```
DIAGNOSE:
  # Check if the plugin is already installed:
  openclaw plugins list 2>/dev/null | grep -i <plugin-name>
  # Check if the package exists on npm:
  npm view <npm-spec> 2>/dev/null | head -5

ACT:
  # Step 1: Backup config
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)

  # Step 2: Install the plugin (NOT npx — use openclaw's plugin installer)
  openclaw plugins install <npm-spec>
  # Note: "npx -y <pkg>" will fail with "could not determine executable to run"
  # because channel plugins are not standalone CLIs.

  # Step 3: Check security audit results
  openclaw security audit --deep 2>&1 | grep -A10 <plugin-name>
  # WARNING: The audit may flag "[potential-exfiltration]" patterns.
  # For channel plugins (e.g., WeChat), file read + network send is NORMAL
  # (e.g., encrypting and uploading images to WeChat CDN). These are often
  # false positives. Review the actual code to confirm.

  # Step 4: QR code login (if the plugin requires it)
  openclaw channels login --channel <channel-id>
  # IMPORTANT: QR codes expire after ~5 minutes. If the login process times out,
  # re-run the command to get a fresh QR code.

  # Step 5: Restart Gateway to load the new channel
  openclaw gateway restart

VERIFY:
  openclaw plugins list 2>/dev/null | grep -i <plugin-name>
  # Expected: shows "loaded" status
  sleep 15
  openclaw status 2>&1 | grep -i <channel-name>
  # Expected: channel appears in the status output
```

**Post-install configuration notes:**
- The plugin installer automatically updates `openclaw.json` (adds `plugins.entries` and `plugins.installs`). A `.bak` backup is also created by the installer itself.
- After install, you'll see "plugins.allow is empty; discovered non-bundled plugins may auto-load" in logs. This is a **warning, not an error**. To silence it, add the plugin ID to `plugins.allow` in config.
- If the gateway log shows "invalid channels.start channel" after login, it means the login credentials were saved but the gateway hasn't picked them up yet. A `gateway restart` fixes this.

**Uninstalling an external plugin:**
```
# Use the same cleanup procedure as P1b:
# 1. Remove from bindings[], channels{}, plugins.entries{}, plugins.installs{}
# 2. Move ~/.openclaw/extensions/<name> to trash
# 3. openclaw gateway restart
```

---

## Global Invariants (must hold after any operation)

After ANY start/restart/upgrade operation, verify these before declaring success:

```
1. openclaw --version returns a version string (no errors)
2. lsof -i :18789 shows a LISTEN process
3. curl http://127.0.0.1:18789/ returns HTTP 200
4. openclaw status shows "Gateway ... reachable"
5. No "Invalid config" in gateway.err.log
6. launchctl print gui/$(id -u)/ai.openclaw.gateway shows "state = running" or "state = sleeping"
   (NOT "SERVICE_NOT_IN_DOMAIN" — that indicates bootout happened)
```

If any invariant fails, do not declare success. Go back to State Diagnosis (Phase 0) and pick the matching playbook.

**Special case for stop operations**: Invariant 6 may intentionally show DOMAIN_UNLOADED if the user requested a permanent stop via `openclaw gateway stop`. In that case, verify invariants 1-5 only, and note the domain state separately.

---

## Common Failure Patterns and Mandatory Pivots

| Failure Pattern | Count | Mandatory Pivot |
|----------------|-------|----------------|
| `npm install -g openclaw` → files still missing | 2 | Stop. Check `node -v`, `npm -v`, npm registry reachability, nvm env. Do not retry install. |
| `openclaw gateway restart` → port never bound | 2 | Stop. Check `tail -30 ~/.openclaw/logs/gateway.err.log`. Likely config error or version mismatch. |
| `openclaw update` → same version after completion | 1 | Check `openclaw status` Update line. The version on your channel may already be latest. This is not a failure. |
| Config edit → "Invalid config" on restart | 1 | Restore from backup, re-read the error message carefully, fix the specific key, restart. |
| `openclaw status` shows "Update available" | 0 (do not act) | This is INFORMATION, not a TODO. Only upgrade if user explicitly asked. |
| `openclaw gateway stop` → service no longer auto-starts after reboot | 1 | Re-bootstrap with `openclaw gateway start`. For temporary stops, prefer `launchctl stop gui/$(id -u)/ai.openclaw.gateway` to avoid bootout. |
| User reports "Gateway won't start after login" but plist exists and `openclaw status` looks OK | 1 | Check State F (domain registration). Likely P8 DOMAIN_DEREGISTERED. |
| `npx -y <plugin-pkg>` → "could not determine executable to run" | 1 | Use `openclaw plugins install <npm-spec>` instead. Channel plugins are not standalone CLIs. |
| Plugin install → security audit shows "[potential-exfiltration]" | 0 (do not panic) | For channel plugins, file read + network send is normal (e.g., uploading images to WeChat CDN). Review code to confirm it's a false positive. |
| `openclaw channels login` → QR code expired | 1 | Re-run the login command to get a fresh QR code. Codes expire after ~5 minutes. |
| QR login succeeds but gateway shows "invalid channels.start channel" | 1 | Credentials saved but gateway hasn't loaded them. Run `openclaw gateway restart`. |

---

## Extension Lifecycle (channels and plugins)

When removing a channel or plugin, you MUST clean up ALL of these locations. Leaving any behind causes auto-discovery errors:

```
1. ~/.openclaw/openclaw.json:
   - bindings[] entries referencing the channel
   - channels{} key for the channel
   - plugins.entries{} key for the plugin
   - plugins.installs{} record for the plugin (optional but recommended)

2. ~/.openclaw/extensions/:
   - Directory matching the plugin name
   - Move to trash, never rm -rf

3. ~/.openclaw/agents/*/sessions/:
   - Old session files may reference the channel (harmless but can be cleaned)
```

After cleanup, restart Gateway. If you see "plugins.allow is empty; discovered non-bundled plugins may auto-load" in logs, that's a warning, not an error. To silence it, set `plugins.allow` to an explicit list in config.
