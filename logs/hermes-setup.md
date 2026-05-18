# Hermes Agent 配置日志

> Week 1 · 任务 #6 · 课程工具准备（PoW）
> 日期：2026-05-18

## 选型

课程提供 3 个 learning agent 选项：Claude Code / Codex CLI / Hermes Agent。
开营仪式 Jacken 老师在 `00:54-01:02` 演示了 Hermes 接 Telegram + 自动同步课程任务的工作流。结合自己已有的 Claude Code 订阅，最终选择：

**Hermes Agent（本地）+ Claude Code 订阅做 LLM provider + Telegram 做对话入口**

理由：
- **零额外 LLM 成本**：Hermes 原生支持读 macOS Keychain 里的 `Claude Code-credentials` 条目（>=Claude Code 2.1.114），直接复用订阅 OAuth token
- **手机端可用**：通过 Telegram bot 远程访问，不绑定笔记本
- **跟 Claude Code 协作不冲突**：Claude Code 做 IDE 内 coding 助手，Hermes 做跨设备对话+任务管理，满足课程"高级挑战"要求（"分别用 Claude Code / Codex / Hermes 中至少两个工具处理同一个学习任务"）

## 安装步骤

```bash
# 1. 装 Hermes（官方 install.sh，预先 audit 过 - 2007 行，行业标准，仅在装系统包时用 sudo 并提示）
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh -o /tmp/hermes-install.sh
# (audit: set -e, unset PYTHONPATH/PYTHONHOME, sudo 仅在 brew/apt 处提示, rm -rf 仅清 tmp)
bash /tmp/hermes-install.sh --skip-setup

# 2. 创建 Telegram bot（@BotFather → /newbot）
# 3. 把 TELEGRAM_BOT_TOKEN 写到 ~/.hermes/.env (mode 600)
# 4. 配置 model 走 Claude Code 订阅
hermes config set model.default "anthropic/claude-opus-4-7"
hermes config set model.provider "anthropic"
hermes config set model.base_url ""    # 关键：清空 base_url，否则会被默认路由到 OpenRouter

# 5. 启动 gateway 配对
hermes gateway run &
# 在 TG bot 对话发 /start，bot 返回 pairing code
hermes pairing approve telegram <CODE>

# 6. 装成 launchd 自启服务
hermes gateway install
# → ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

## 验证（功能测试）

### 测试输入（Telegram）
> 帮我用一句话说明 AI Agent 和 Web3 智能合约的本质区别

### Hermes 回复（Claude Opus 4.7）
> AI Agent 是**概率性**的——基于语言模型推理，每次执行结果可能不同，擅长处理模糊和开放性任务；而 Web3 智能合约是**确定性**的——代码即法律，相同输入永远产生相同输出，追求的是无需信任的可验证执行。
>
> 一句话版本：**AI Agent 用"理解"做事，智能合约用"规则"做事。**

### 进程状态
```
launchctl list | grep hermes
# 98836  0  ai.hermes.gateway
```

## 踩坑记录

### 坑 1：base_url 残留 OpenRouter，model 调用 404
**症状**：bot 把 Next.js 框架的 HTML 404 页面源码作为模型回复 dump 到 TG（29 条连续消息，全是 `_next/static/chunks/*.js` 路径）。
**原因**：`hermes config set model.provider anthropic` **不会**自动清空 `model.base_url`，残留的 OpenRouter URL 接收了请求但没有 API key，返回 HTML 404。
**修复**：`hermes config set model.base_url ""`。
**教训**：切 provider 时要同时清/设 base_url；命令行 `--provider anthropic` 会覆盖 config，所以单次测试通的不代表 config 默认走得通。

### 坑 2：API key 不要贴聊天框
跟 AI Agent 配置 key 时，**输入要走 `read -s` 等隐藏输入或直接编辑 `~/.hermes/.env`**，不要把完整 key 复制粘贴到任何对话窗口。本次首发因为没注意这点，后续把 GLM key 弃用、只保留 TG bot token + Claude Code 订阅 OAuth（OAuth 自动 refresh，无单点泄露风险）。

## 关键约束

按 Jacken 老师在 `00:58` 的明文说明：
> "API Key 必须配到机器的环境变量里，不能通过聊天框传给 agent——我们尽量不要做这种把 API Key 暴露出来的行为。"

本次配置完全遵循：所有 secret 都在 `~/.hermes/.env` (mode 600)，不进 git，不进聊天上下文。

## 后续可做

- [ ] 配置 `hermes skills` 添加 ai-web3-school 专属 skill（自动同步 WCB 任务列表 → TG 通知）
- [ ] 接入 `hermes cron` 设置每日打卡提醒
- [ ] Hackathon 阶段考虑用 Hermes delegation 跑并行 sub-agents
