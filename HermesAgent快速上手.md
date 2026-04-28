# Hermes Agent 快速上手

> 5 分钟从零到跑起来的 Hermes Agent 实践指南。完整参考见 [Hermes Agent：实践者参考指南（2026）.md](./Hermes%20Agent%EF%BC%9A%E5%AE%9E%E8%B7%B5%E8%80%85%E5%8F%82%E8%80%83%E6%8C%87%E5%8D%97%EF%BC%882026%EF%BC%89.md)

---

## Hermes Agent 是什么？

一个开源、自我进化的 AI 智能体运行时。支持 CLI、Telegram/Discord/Slack/WhatsApp 等 17 个消息平台，兼容任何 OpenAI 兼容的 LLM 提供商（OpenRouter、Anthropic、DeepSeek、Kimi、通义千问、Ollama 等约 19 个）。

不是聊天封装器 —— 它能读写文件、执行命令、抓取网页、生成子智能体、创建技能、运行定时任务。

## 安装

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc    # 或 ~/.zshrc
hermes              # 开始聊天
```

**环境要求：** 只需 `git`。安装脚本自动处理 Python、Node.js、依赖项。

验证安装：

```bash
hermes version      # 查看版本
hermes doctor       # 诊断问题（出问题时优先跑这条）
hermes status       # 查看当前配置和认证状态
```

> 原生 Windows 不支持，请在 WSL2 中运行。

---

## 第一次配置：选一个提供商

Hermes 有 **3 条认证路径**，对号入座即可：

### 路径 1：API Key（最常用）

在 `~/.hermes/.env` 中配置密钥，Hermes 启动时自动读取。

**OpenRouter（一个 Key 访问几乎所有模型）：**

```bash
hermes config set OPENROUTER_API_KEY sk-or-v1-xxxxx
hermes chat --provider openrouter --model anthropic/claude-sonnet-4
```

**DeepSeek：**

```bash
hermes config set DEEPSEEK_API_KEY sk-xxxxx
hermes chat --provider deepseek --model deepseek-chat
```

**通义千问（阿里云）：**

```bash
hermes config set DASHSCOPE_API_KEY sk-xxxxx
hermes chat --provider alibaba --model qwen3.5-plus
```

### 路径 2：OAuth 登录

没有 API Key，但有 ChatGPT / Claude Pro 账号？用这个：

```bash
hermes model    # 交互式选择提供商
# 选 Anthropic、OpenAI Codex 或 GitHub Copilot
# 自动打开浏览器完成 OAuth，凭据存储在 ~/.hermes/auth.json
```

### 路径 3：本地模型（Ollama 等）

```bash
# 先启动 Ollama
ollama pull qwen2.5-coder:14b
OLLAMA_CONTEXT_LENGTH=32768 ollama serve    # 重要：提高默认的 4k 上下文

# 再配置 Hermes
hermes model    # 选 "Custom endpoint"
# 输入：http://localhost:11434/v1 → 无需 Key → qwen2.5-coder:14b
```

> **Ollama 陷阱：** 默认上下文只有 4096 tokens！务必设置 `OLLAMA_CONTEXT_LENGTH=32768`。
> **llama.cpp 陷阱：** 必须加 `--jinja` 参数，否则工具调用不生效。

---

## 常用命令速查

### 顶层命令

| 命令 | 用途 |
|------|------|
| `hermes` | 进入交互式对话 |
| `hermes chat -q "一句话提问"` | 一次性非交互提问 |
| `hermes model` | 切换默认提供商/模型 |
| `hermes config set KEY VAL` | 修改配置 |
| `hermes doctor` | 诊断问题 |
| `hermes status` | 查看当前状态 |
| `hermes dump` | 生成诊断报告（求助时粘贴） |
| `hermes gateway start` | 启动消息网关 |
| `hermes update` | 更新到最新版 |

### 会话中的 Slash 命令

| 命令 | 作用 |
|------|------|
| `/model openrouter:claude-sonnet-4` | 会话中途切换模型（不丢历史） |
| `/model deepseek:deepseek-chat` | 切换到 DeepSeek |
| `/model custom:qwen-2.5` | 切换到自定义端点 |
| `/help` | 显示所有可用命令 |
| `/new` | 开始新会话 |
| `/compress` | 手动压缩对话上下文 |
| `/yolo` | 跳过危险命令审批提示 |
| `/save` | 保存当前对话 |
| `/history` | 查看对话历史 |
| `/skills` | 管理技能 |
| `/tools list` | 查看已启用的工具 |
| `/btw 问题` | 临时提问（不调工具不持久化） |
| `/plan 需求` | 先生成计划再执行 |
| `/undo` | 撤销上一轮对话 |
| `/rollback` | 恢复文件系统检查点 |

### 模型切换语法

```bash
/model                                        # 查看当前模型
/model zai:glm-5                              # 指定 provider:model
/model custom:local:qwen-2.5                  # 命名自定义提供商
/model openrouter:anthropic/claude-sonnet-4   # 切回云端
```

---

## 关键配置文件

```
~/.hermes/
├── config.yaml    # 所有设置（模型、终端、压缩、记忆、工具包等）
├── .env           # 密钥（API Key、Bot Token）
├── SOUL.md        # Agent 身份定义（语气、风格）
├── skills/        # 技能库
├── memories/      # 持久化记忆
├── cron/          # 定时任务
├── sessions/      # 对话记录（SQLite + 全文搜索）
└── logs/          # 日志
```

- **非机密设置** → `config.yaml`
- **密钥** → `.env`
- `config.yaml` 中可用 `${VAR}` 引用 `.env` 中的变量

### 最简 config.yaml 示例

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4

# 本地 Ollama 版本：
# model:
#   provider: custom
#   default: qwen2.5-coder:14b
#   base_url: http://localhost:11434/v1
#   context_length: 32768

compression:
  enabled: true
  threshold: 0.50                    # 达到 50% 上下文时自动压缩

terminal:
  backend: local                      # 或 docker / ssh
  timeout: 180

toolsets:
  - web
  - terminal
  - file
  - skills
  - memory
  - todo
```

### 模型回退配置

当主提供商失败时（限流、服务器错误），自动切换到备用模型，不丢失对话：

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

---

## 常见陷阱与修复

### 1. 视觉/网页摘要/压缩悄悄失效

**现象：** 配置了 Anthropic，但图片分析、网页抓取不工作。

**原因：** 这些辅助任务默认使用 Gemini Flash（走 OpenRouter），而你只有 Anthropic Key。

**修复：** 在 `config.yaml` 中加：

```yaml
auxiliary:
  vision:      { provider: "main" }
  web_extract: { provider: "main" }

compression:
  summary_provider: "main"
```

### 2. 本地模型上下文只有 2048 tokens

**修复：** 在 `config.yaml` 中显式设置（Ollama 还需要 `OLLAMA_CONTEXT_LENGTH`）：

```yaml
model:
  context_length: 32768
```

### 3. 工具调用以文本形式出现，没有执行

本地服务器未启用工具调用功能：
- **llama.cpp：** 加 `--jinja` 参数
- **vLLM：** 加 `--enable-auto-tool-choice --tool-call-parser hermes`
- **SGLang：** 加 `--tool-call-parser qwen`

### 4. 响应在句子中间截断

SGLang 默认 `max_tokens=128`，在 `config.yaml` 中设置：

```yaml
model:
  max_tokens: 4096
```

---

## 提供商速查

| 提供商 | 认证方式 | 配置方法 |
|--------|---------|---------|
| OpenRouter | API Key | `OPENROUTER_API_KEY` |
| Anthropic | OAuth / API Key | `hermes model` 或 `ANTHROPIC_API_KEY` |
| DeepSeek | API Key | `DEEPSEEK_API_KEY` |
| 通义千问 | API Key | `DASHSCOPE_API_KEY` |
| Kimi | API Key | `KIMI_API_KEY` |
| 智谱 GLM | API Key | `GLM_API_KEY` |
| MiniMax | API Key | `MINIMAX_API_KEY` |
| Ollama | 无（本地） | `hermes model` → Custom endpoint |
| vLLM/SGLang | 无（本地） | `hermes model` → Custom endpoint |

---

## 进阶功能一览

- **消息网关：** Telegram、Discord、Slack、WhatsApp、Signal、微信、飞书、钉钉等 17 个平台
- **定时任务：** 对话式创建，如"每天早上 9 点检查 AI 新闻并 Telegram 推送" → `hermes cron list`
- **多 Profile：** `hermes profile create work` 创建隔离实例（工作与个人互不干扰）
- **技能系统：** Agent 自动从经验中创建技能；`hermes skills browse` 浏览技能市场
- **MCP 集成：** `hermes mcp add` 连接外部 MCP 服务器扩展工具
- **IDE 集成：** `hermes acp` 作为 ACP 服务器接入 VS Code / Zed / JetBrains
- **本地仪表板：** `hermes dashboard` 启动浏览器管理界面

---

## 出问题了怎么办？

```bash
hermes doctor           # 第一条诊断命令
hermes dump             # 生成脱敏诊断报告，贴到 GitHub Issue 或 Discord
hermes logs -f          # 实时查看日志
hermes logs --level WARNING --since 1h   # 最近 1 小时的警告
```
