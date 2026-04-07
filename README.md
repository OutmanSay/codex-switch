# codex-switch

ChatGPT Plus 多账号管理工具，用 bash 实现 [OpenAI Codex CLI](https://github.com/openai/codex) 的账号切换。

> Codex CLI 本身只支持单账号（一份 OAuth token 存在 `~/.codex/auth.json`）。
> 这个工具通过备份/替换 token 文件实现多账号轮换，适合有多个 ChatGPT Plus 账号想轮流用的场景。

## 为什么需要这个

- ChatGPT Plus 账号 GPT-5.4 有使用额度限制
- 有多个 Plus 账号时，想在 Codex CLI 之间切换必须 `codex logout` + `codex login` 重新走 OAuth，每次都要打开浏览器
- 希望像 `kubectl config use-context` 一样的体验——一条命令就切

## 工作原理

```
时刻 1：codex login A 账号       → auth.json 存 A 的 token
时刻 2：codex-switch save alice  → auth-alice.json 备份 A
时刻 3：codex logout + codex login B 账号
时刻 4：codex-switch save bob    → auth-bob.json 备份 B
时刻 5：codex-switch alice       → 复制 auth-alice.json → auth.json（切回 A）
时刻 6：codex-switch bob         → 复制 auth-bob.json → auth.json（切回 B）
```

Codex CLI 每次调用都重新读 `auth.json`，所以**切换立刻生效**，不需要重启任何服务。

## 安装

```bash
# 把脚本放到 PATH 可及的位置
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/OutmanSay/codex-switch/main/codex-switch -o ~/.local/bin/codex-switch
chmod +x ~/.local/bin/codex-switch

# 确保 ~/.local/bin 在 PATH 里
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc  # 或 ~/.bashrc
source ~/.zshrc
```

或者直接 clone：

```bash
git clone https://github.com/OutmanSay/codex-switch.git
cd codex-switch
install -m 755 codex-switch ~/.local/bin/codex-switch
```

## 使用

### 第一次使用 — 保存第一个账号

```bash
# 假设你已经 codex login 登录了账号 A
codex-switch save alice
```

输出：
```
✓ 当前账号已保存为 'alice'
当前账号: alice@example.com (mode: chatgpt)
```

### 添加第二个账号

```bash
# 登出当前账号
codex logout

# 登录第二个账号（会打开浏览器）
codex login

# 保存为 bob
codex-switch save bob
```

### 日常切换

```bash
# 查看当前账号 + 所有存档
codex-switch

# 切到 alice
codex-switch alice

# 切到 bob
codex-switch bob
```

输出示例：
```
$ codex-switch
当前账号: alice@example.com (mode: chatgpt)

已保存的账号:
  alice           alice@example.com
  bob             bob@example.com
```

## 命名建议

用邮箱的本地部分（@ 前面）作为存档名，方便记忆和识别：
- `alice@gmail.com` → `codex-switch save alice`
- `bob@example.com` → `codex-switch save bob`
- `work@company.com` → `codex-switch save work`

## 自动同步机制

Codex CLI 在使用过程中会**自动刷新 access_token**（用 refresh_token 换新的），并回写到 `auth.json`。

如果不处理这个问题，每个账号的存档文件会逐渐老化——虽然 `refresh_token` 还能用，但 `access_token` 是过期的快照。

**`codex-switch` 在切换前会自动同步当前账号的最新 token 到对应的存档**：

```bash
$ codex-switch bob
  (auto-sync: 当前账号 'alice' 的最新 token 已同步回存档)
✓ 已切换到账号 'bob'
当前账号: bob@example.com (mode: chatgpt)
```

这保证了每个账号存档始终是最新刷新后的 token。

## 文件位置

- 工具：`~/.local/bin/codex-switch`
- 当前 Codex 登录：`~/.codex/auth.json`（Codex CLI 标准位置）
- 账号存档：`~/.codex/auth-<name>.json`
- 灾难恢复备份：`~/.codex/auth.json.before-switch`（每次切换前自动创建）

## 与 OpenClaw 集成

如果你用 [OpenClaw](https://github.com/openclaw/openclaw) 通过 Codex CLI 调用 GPT-5.4，这个工具可以无缝接入。

仓库的 `openclaw-skill-example/SKILL.md` 包含一个完整的 OpenClaw skill 示例，让 AI agent 能听懂 "切换 codex 账号"、"GPT-5.4 额度用完了" 等自然语言指令，自动帮你切换账号。

用法：
```bash
# 安装 skill 到 OpenClaw
mkdir -p ~/.openclaw/skills/codex-account-switch
cp openclaw-skill-example/SKILL.md ~/.openclaw/skills/codex-account-switch/SKILL.md
```

然后对话里说"切换 codex 账号到 alice"，agent 就会自动调用 `codex-switch alice`。

## 常见问题

**Q: 切换后需要重启 OpenClaw / 任何服务吗？**
不需要。Codex CLI 每次调用都重新读 `auth.json`，切换立刻生效。

**Q: 这会不会让 OpenAI 的反作弊系统检测到？**
每个账号都是你本人正常登录的，切换工具只是文件级管理，不会产生异常的登录/登出痕迹。本质上跟你手动 `codex logout && codex login` 切换一样，只是更方便。

**Q: refresh_token 过期了怎么办？**
过期后 `codex-switch <name>` 切过去会 codex CLI 用不了。这时需要对那个账号重新走一次完整登录：`codex logout && codex login && codex-switch save <same-name>`（覆盖旧存档）。

**Q: 支持 API key 模式吗？**
支持。`auth.json` 可以存 API key 或 OAuth token，`codex-switch` 只是替换整个文件，所以两种模式都兼容。

**Q: 能做成 Claude Code、Gemini CLI 的多账号切换吗？**
原理相同，替换对应 CLI 的 auth 文件就行。已有社区工具 [ccswitch](https://github.com/KyoriChuangShou/ccswitch) 做 Claude Code 多账号切换，思路一样。

## License

MIT

## 相关

- [OpenAI Codex CLI](https://github.com/openai/codex) — 官方 CLI
- [OpenClaw](https://github.com/openclaw/openclaw) — Agent 编排平台
- [ccswitch](https://github.com/KyoriChuangShou/ccswitch) — Claude Code 多账号切换工具
