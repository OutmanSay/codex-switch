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

ChatGPT Plus 多账号切换，通过 `~/.local/bin/codex-switch` 工具管理多个 Codex CLI 的 OAuth 凭证。

## 背景

OpenClaw 的 `openai-codex/gpt-5.4` 模型走的是 Codex CLI 路径，由 Codex CLI 读取 `~/.codex/auth.json` 里的 OAuth token 访问 ChatGPT Plus。**Codex CLI 本身不支持多账号**，但我们通过备份/替换 `auth.json` 实现了多账号切换。

当前用户有多个 Plus 账号，每个都用自己邮箱前缀命名存档在 `~/.codex/auth-<email-local-part>.json`。

## When to Use

- 用户说"GPT-5.4 额度用完了 / 没额度了 / quota exhausted"
- 用户说"换一个 codex / ChatGPT 账号"
- 用户说"codex 欠费了 / 报错了 / 调不通"
- 用户说"切到另一个 ChatGPT Plus"
- 用户直接说邮箱前缀："用 xxx 账号"（xxx 是某个邮箱前缀）

## When NOT to Use

- 用户说"切换模型"但没提 GPT-5.4 或 ChatGPT 账号 → 用 session 级别的 `/model` 切换，不是这个 skill
- 用户说"切换 bot" → 是建议用户自己在 Telegram 切 bot，不是这个 skill
- 需要新登录第三个账号 → 这个 skill 不处理登录，告诉用户手动跑 `codex logout && codex login && codex-switch save <name>`

## 核心命令

### 查看当前账号 + 所有存档

```bash
codex-switch
```

输出示例：
```
当前账号: alice@example.com (mode: chatgpt)

已保存的账号:
  alice           alice@example.com
  bob             bob@example.com
```

### 切换到指定账号

```bash
codex-switch <name>
```

其中 `<name>` 是存档文件 `auth-<name>.json` 的 `<name>` 部分（一般是邮箱本地部分，即 @ 前面的内容）。

### 保存当前账号

```bash
codex-switch save <name>
```

## 工作流程

当用户触发本 skill 时，sub-agent 按以下步骤执行：

1. **查看当前状态**：跑 `codex-switch`，获取当前账号 + 所有可用账号列表
2. **判断用户意图**：
   - 如果用户指定了具体账号名 → 直接跑 `codex-switch <name>`
   - 如果只说"换另一个" → 从列表里挑一个**不是当前账号**的，跑 `codex-switch <name>`
   - 如果只有一个存档账号 → 告诉用户"只有一个账号存档，需要手动 `codex logout && codex login` 登录新账号并 `codex-switch save <name>` 保存"
3. **确认切换**：再跑一次 `codex-switch`（不带参数）显示新的当前账号
4. **告诉用户**：
   - 切换成功 → "已切换到 xxx@xxx.com，OpenClaw 下次调 GPT-5.4 会用新账号"
   - **不需要重启 OpenClaw gateway**（Codex CLI 每次调用都重新读 auth.json，切换立刻生效）

## 重要提示

### 切换后的生效机制

- OpenClaw 调 GPT-5.4 路径：`main → openai-codex provider → spawn codex CLI → 读 auth.json → 调 ChatGPT`
- 每次都是 spawn 新进程，所以**切换立刻生效**，不需要重启 gateway
- 但**已有的 session 里在跑的请求**不会被影响，要下一次 prompt 才会用新账号

### 常见陷阱

1. **bot session 的 model override 跟 codex 账号无关**：用户如果说"bot1 切到 GPT-5.4"，那是 session 模型切换；说"切 codex 账号"才是本 skill
2. **Codex CLI 进程缓存**：理论上可能有短暂的 token 缓存，如果切换后立刻报错，建议等 10 秒重试
3. **两个账号的额度独立**：切换后不会合并额度，只是轮流用

## 参考

- 工具源码：`~/.local/bin/codex-switch`
- Codex auth 存储：`~/.codex/auth.json` (当前) + `~/.codex/auth-*.json` (存档)
- OpenClaw 的 codex provider：`auth.profiles.openai-codex:default`（mode=oauth，不需要改）
