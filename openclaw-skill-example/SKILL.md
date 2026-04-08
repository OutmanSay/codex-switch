---
name: codex-account-switch
description: ChatGPT Plus 多账号切换。当用户说"GPT-5.4 额度用完了"、"切换 codex 账号"、"换一个 ChatGPT 账号"、"codex 退费了 / 欠费了"时触发，使用本 skill 列出可用账号并切换到指定账号。
context: fork
metadata:
  openclaw:
    emoji: "🔀"
    requires:
      bins: ["bash"]
---

# codex-account-switch

ChatGPT Plus 多账号切换，通过 `~/.local/bin/codex-switch` 工具管理多个 Codex CLI 的 OAuth 凭证，并同步到 OpenClaw 每个 agent 的独立 auth-profiles 缓存。

## 背景

OpenClaw 的 `openai-codex/gpt-5.4` 模型在**两个独立位置**存着 OAuth token，切账号必须同时更新才能真正生效：

1. **`~/.codex/auth.json`** — Codex CLI 自己用的（用户在 terminal 里跑 `codex "..."` 时读这个）
2. **`~/.openclaw/agents/<agent>/agent/auth-profiles.json`** — **每个 OpenClaw agent 有自己独立的一份**，存在 `profiles["openai-codex:default"]` 里
   - 覆盖 5 个 agent：main / chill / code / luna / think
   - codex agent 没有这个文件
   - **OpenClaw 通过 🦞 / bot 调 gpt-5.4 时，读的是这里，不是 `~/.codex/auth.json`**

**历史坑**（2026-04-07 到 04-08 花了 3+ 小时定位）：早期的 `codex-switch` 只改 `~/.codex/auth.json`，完全没碰 `auth-profiles.json`，导致所有通过 🦞 / bot 触发的调用一直烧旧账号的 quota，而 terminal 里跑 `codex` 又正常。2026-04-08 升级了 `codex-switch` 把这个坑彻底堵上。

## When to Use

- 用户说"GPT-5.4 额度用完了 / 没额度了 / quota exhausted"
- 用户说"换一个 codex / ChatGPT 账号"
- 用户说"codex 欠费了 / 报错了 / 调不通"
- 用户说"切到另一个 ChatGPT Plus"
- 用户直接说邮箱前缀："用 xxx 账号"（xxx 是某个邮箱前缀）
- 用户说"gpt-5.4 一直在烧 xxx 账号但我想用 yyy 账号" → 几乎肯定是这个 skill

## When NOT to Use

- 用户说"切换模型"但没提 GPT-5.4 或 ChatGPT 账号 → 用 session 级别的 `/model` 切换，不是这个 skill
- 用户说"切换 bot" → 是建议用户自己在 Telegram 切 bot，不是这个 skill
- 需要新登录第三个账号 → 这个 skill 不处理登录，告诉用户手动跑 `codex logout && codex login && codex-switch save <name>`（save 会自动同步到 OpenClaw 所有 agent）

## 核心命令

### 查看当前账号 + 所有存档

```bash
codex-switch
```

输出示例：
```
当前账号: alice@example.com (mode: chatgpt)

已保存的账号:
  alice                 alice@example.com
  bob                   bob@example.com
```

### 切换到指定账号（最常用）

```bash
codex-switch <name>
```

其中 `<name>` 是存档文件 `auth-<name>.json` 的 `<name>` 部分（一般是邮箱本地部分，即 @ 前面的内容）。

**这条命令会自动做 5 件事**（2026-04-08 之后）：
1. `auto_sync_current`：把当前 auth.json 存档回 `auth-<current>.json`（保留 refresh 后的最新 token）
2. `kill_running_codex`：杀掉所有名为 `codex` 的进程（pgrep -x，精确匹配），避免内存缓存污染
3. `cp auth-<target>.json → auth.json`：切换 Codex CLI 层
4. **`sync_openclaw_profiles`**：把新 `access / refresh / expires / accountId` 写进 5 个 agent 的 `auth-profiles.json`（原子写 + 自动备份为 `.bak-<timestamp>`）
5. **`restart_openclaw_gateway`**：根据调用场景决定是否重启 gateway
   - **Terminal 调用**（`[ -t 1 ]` 为真）：强制 `launchctl bootout + bootstrap`，约 10 秒 downtime
   - **Sub-agent 调用**（非交互，通过本 skill）：**跳过 restart**，依赖 gateway 的 auth-profiles.json 热加载机制（约 60 秒内生效）
   - 设置 `CODEX_SWITCH_FORCE_RESTART=1` 可以强制重启

### 保存当前账号

```bash
codex-switch save <name>
```

把当前 `auth.json` 存档为 `auth-<name>.json`，同时**自动同步到 OpenClaw 的 5 个 auth-profiles.json**。`codex login` 登新账号后必须跑这个，否则 OpenClaw agents 还是用的老 token。

### 强制重新同步（debug 用）

```bash
codex-switch sync-openclaw
```

如果怀疑 OpenClaw 的 auth-profiles 跟 `~/.codex/auth.json` 不一致，可以跑这个强制同步。会问你是否重启 gateway。

## 工作流程

当用户触发本 skill 时，sub-agent 按以下步骤执行：

1. **查看当前状态**：跑 `codex-switch`，获取当前账号 + 所有可用账号列表
2. **判断用户意图**：
   - 如果用户指定了具体账号名 → 直接跑 `codex-switch <name>`
   - 如果只说"换另一个" → 从列表里挑一个**不是当前账号**的，跑 `codex-switch <name>`
   - 如果只有一个存档账号 → 告诉用户"只有一个账号存档，需要手动 `codex logout && codex login` 登录新账号并 `codex-switch save <name>` 保存"
3. **确认切换**：脚本输出里会有 `✓ 已切换到账号 'xxx'` 和 `[openclaw-sync] ✓ 已同步 5 个 agent profile`
4. **告诉用户**：
   - 切换成功 → "已切换到 xxx@xxx.com。OpenClaw 的 5 个 agent profile 都已同步，gateway 会在约 60 秒内热加载新 token。你可以继续用 🦞，对话不会中断。"
   - 如果 sync 或 auth.json 切换失败，把脚本完整输出原样返回给用户

## 重要提示

### 切换后的生效机制（重点 — 2026-04-08 重写）

- **OpenClaw 调 GPT-5.4 的完整路径**：
  ```
  🦞 / chill / code / luna / think
    → openai-codex provider
    → 读 ~/.openclaw/agents/<agent>/agent/auth-profiles.json 里 profiles["openai-codex:default"]
    → 用里面的 access_token 直接调 OpenAI API
    → access 快过期时用 refresh_token 刷新，写回同一个文件
  ```
- **Codex CLI 路径**（terminal 里跑 `codex "..."`）：
  ```
  codex → 读 ~/.codex/auth.json → 调 ChatGPT
  ```
- **两条路径凭证互不共享**，本 skill 的 `codex-switch <name>` 同时更新两边

### 生效时序

- 对话 / session / 记忆层面：**完全无感**，sub-agent 的 `sessionKey`、历史消息、记忆都保留
- API 层面：
  - Terminal 调用：`codex login` / `codex-switch` 后立即生效
  - Sub-agent 调用：gateway 热加载 auth-profiles.json 约 60 秒内生效（实测通常更快）
  - 如果切换过程中正好有 agent 请求在飞，可能还是用旧 token 完成那一次，下一次请求才用新 token

### 常见陷阱

1. **bot session 的 model override 跟 codex 账号无关**：用户如果说"bot1 切到 GPT-5.4"，那是 session 模型切换；说"切 codex 账号"才是本 skill
2. **`codex login` 后必须跑 `codex-switch save <name>`**：否则 OpenClaw 的 auth-profiles 不会更新，新登录的账号只对 terminal 里的 `codex` 命令生效，对 🦞 无效
3. **两个账号的额度独立**：切换后不会合并额度，只是轮流用
4. **正在运行的 codex 命令会被中断**：如果用户正在 codex REPL 里做事，切换会强制杀掉它（`pkill -x codex`），需要用户手动重新打开 codex。触发本 skill 前可以先提醒"这会打断你当前的 codex terminal 会话"（但不会打断 🦞 等 agent 对话）
5. **debug 账号不对的方法**：跑下面这个 snippet 看所有 agent 当前挂的是哪个账号
   ```bash
   python3 -c "
   import json, base64, os
   for a in ['main','chill','code','luna','think']:
       p = os.path.expanduser(f'~/.openclaw/agents/{a}/agent/auth-profiles.json')
       try:
           d = json.load(open(p))
           tok = d['profiles']['openai-codex:default']['access']
           pl = json.loads(base64.urlsafe_b64decode(tok.split('.')[1] + '==='))
           email = pl.get('https://api.openai.com/profile', {}).get('email', '?')
           print(f'{a:6} {email}')
       except Exception as e: print(f'{a}: {e}')
   "
   ```

## 参考

- 工具源码：`~/.local/bin/codex-switch`
- Codex CLI auth 存储：`~/.codex/auth.json`（当前）+ `~/.codex/auth-*.json`（存档）
- OpenClaw agent auth 存储：`~/.openclaw/agents/<name>/agent/auth-profiles.json`（每个 agent 独立）
- OpenClaw 的 codex provider 配置：`openclaw.json` 里 `auth.profiles.openai-codex:default`（mode=oauth，不需要改）
- 双缓存 bug 的完整 post-mortem：memory/`project_openclaw_codex_auth_dual_cache.md`
