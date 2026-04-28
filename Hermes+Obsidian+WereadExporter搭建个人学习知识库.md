# Hermes + Obsidian + WereadExporter 搭建个人学习知识库

> 用 AI Agent 武装本地笔记，实现知识获取、提炼、沉淀、回顾的全自动流水线。

---

## 我的部署环境

本文基于以下实际环境撰写：

| 项目 | 配置 |
|------|------|
| 操作系统 | macOS（本地开发机） |
| Obsidian Vault 路径 | `/Users/{user}/Documents/wiki/obsidian/` |
| Hermes 模型提供商 | DeepSeek V4 Flash（主），配置了 OpenRouter 作为 fallback |
| Hermes 代理 | CompShare (cp.compshare.cn) 中转 |
| WereadExporter 目录 | `~/Documents/hermes-tools/weread-exporter/` |
| Hermes 安装方式 | 官方 install.sh 一键安装，配置文件在 `~/.hermes/config.yaml` |
| 消息平台对接 | 已对接 Telegram / 微信（iLink Bot），支持手机端远程操作 |

> 如果你的环境不同（如 Windows、不同模型提供商），下文每个步骤都会标注跨平台差异。

---

## 为什么需要这三件套？

传统的个人知识管理碎片化严重：

- **微信读书**上划了很多笔记，导出麻烦，读完了就吃灰
- **Obsidian** 里记了一大堆笔记，但散落各处，缺乏关联
- **AI** 帮不上忙，因为笔记格式不统一，AI 看不懂你的知识结构

这套方案解决的问题：**让 AI 成为你知识库的编译器**，而你是架构师。

| 工具 | 角色 | 干什么 |
|------|------|--------|
| **WereadExporter** | 采集器 | 把微信读书上的书/笔记导出为 Markdown |
| **Obsidian** | 存储层 | 以本地 Markdown 文件组织知识网 |
| **Hermes Agent** | 大脑 | 理解、提炼、链接、审查你的知识库 |

三者结合后的工作流：

```
微信读书 → WereadExporter → raw/ 原始资料
                                      ↓
                            Hermes 读取、提炼
                                      ↓
                          wiki/ 结构化笔记 (双向链接)
                                      ↓
                            Hermes 查询、固话、审查
```

---

## 第一步：安装 WereadExporter — 微信读书导出工具

把微信读书上的笔记导出为 Markdown。

### 安装

```bash
git clone https://github.com/drunkdream/weread-exporter.git
cd weread-exporter
pip install -r requirements.txt
```

**macOS** 额外安装系统依赖：

```bash
brew install cairo pango
```

**Windows** 无需额外安装系统依赖 —— 工具已内置所需 DLL。直接在 cmd 或 PowerShell 中运行 pip 即可。

> **Windows 提示：** 如果遇到 `weasyprint` 相关报错，确保 Python 版本 >= 3.7，并安装 Visual C++ 红istributable。PDF 导出在 Windows 上工作正常，无需额外配置。

### 获取书籍 ID

打开 https://weread.qq.com/ ，搜索目标书籍，进入书籍介绍页。URL 中的最后一段就是书籍 ID：

```
https://weread.qq.com/web/bookDetail/08232ac0720befa90825d88
                                        ↑ 这就是 book ID
```

### 导出为 Markdown

```bash
python -m weread_exporter -b 08232ac0720befa90825d88 -o md
```

首次运行会打开浏览器，扫码登录微信读书。后续会话会自动复用 Cookie。

输出在项目根目录的 `cache/<book_id>/chapters/` 下，每章节一个 `.md` 文件。

> **提示：** 如果 Chrome 已经登录了微信读书，加 `--use-default-profile` 可以直接复用会话：
> ```bash
> python -m weread_exporter -b <book_id> -o md --use-default-profile
> ```

你也可以同时导出为其他格式：

```bash
python -m weread_exporter -b <book_id> -o epub -o pdf
```

### 导出笔记

微信读书自带笔记和划线功能。导出的 Markdown 中会包含高亮标注。把这些 raw 文件作为知识库的素材输入。

---

## 第二步：搭建 Obsidian 知识库 — 三层文件结构

参考 Andrej Karpathy 开源的 **LLM Wiki 方法**，按三层结构组织你的知识库。

### 创建知识库

用 Obsidian 打开一个本地文件夹（比如 `~/Documents/wiki/`），这就是你的 Vault。

> **Windows 路径提示：** Obsidian 在 Windows 上路径为 `C:\Users\{用户名}\Documents\wiki\`。Hermes 在 WSL2 中访问 Windows 文件通过 `/mnt/c/Users/{用户名}/Documents/wiki/`。推荐把 Vault 放在 WSL2 内部（如 `~/wiki/`）以避免跨文件系统性能问题。

### 目录结构

```
wiki/
├── _index.md              ← MOC 首页（所有内容的入口）
├── raw/                   ← 原始资源层：外部资料只读存放
│   └── weread-exports/    ← WereadExporter 导出的书籍正文
├── wiki/                  ← 维基文件层：AI 提炼的结构化笔记
│   ├── chapters/          ← 章节笔记（大主题分解）
│   ├── concepts/          ← 核心概念卡片（原子化提炼）
│   ├── people/            ← 人物/实体页面
│   ├── tags/              ← 标签索引
│   └── log.md             ← append-only 操作日志
└── CLAUDE.md              ← 模式与规则层：告诉 AI 怎么管理这个知识库
```

### 三层结构说明

**原始资源层 (raw/)**

存放 WereadExporter 导出的原始 Markdown、网页剪藏、PDF 等。原则：**只读不改**，保留原始出处。

**维基文件层 (wiki/)**

这是知识库的核心。AI 从原始资料中提取知识点，写成独立的 Markdown 文件，并互相建立 `[[双向链接]]`。

每个知识点独立成页。例如：

- `wiki/concepts/上下文管理.md` 会提到 `[[记忆系统]]` 和 `[[权限系统]]`
- `wiki/chapters/` 下的大章节笔记会引用多个概念笔记
- `wiki/people/` 下的角色笔记被相关技术笔记引用

**模式与规则层 (CLAUDE.md)**

配置文件，告诉 AI 知识库的目录结构、标签规范、命名约定。AI 在操作知识库时会自动读取这个文件。

### 具体的 Vault 结构示例

这是我在使用的真实结构（分析 Claude Code 源码的知识库）：

```
wiki/
├── _index.md              ← MOC 首页，表格列出所有章节和概念
├── chapters/              ← 11 个大章节
│   ├── 01 Claude Code 简介.md
│   ├── 03 System Prompt工程.md
│   ├── 04 工具系统：4个原语，59个工具.md
│   ├── 05 权限系统：信任是设计出来的.md
│   ├── 06 记忆系统：只记偏好，不记代码.md
│   ├── 07 上下文管理：长对话的生存术.md
│   ├── 08 搜索：为什么grep打败了RAG.md
│   ├── 09 多Agent架构：像公司一样运转.md
│   ├── 10 Feature Flags里的未来.md
│   ├── 11 两个Claude Code.md
│   └── 12 Harness Engineering方法论.md
├── concepts/              ← 7 个核心概念
│   ├── Concept System Prompt.md
│   ├── Concept 工具系统.md
│   ├── Concept 权限系统.md
│   ├── Concept 记忆系统.md
│   ├── Concept 上下文管理.md
│   ├── Concept 多Agent架构.md
│   └── Concept Harness Engineering.md
├── people/
│   ├── Anthropic.md
│   └── Capybara（模型内部代号）.md
└── tags/
    └── 标签索引.md
```

首页 `_index.md` 的写法参考：

```markdown
---
title: "知识库名称"
aliases: ["index", "首页", "MOC"]
tags: [元数据, index]
---

# 知识库名称

## 章节

| # | 章节 | 核心主题 |
|---|------|----------|
| 1 | [[01 标题]] | 简介 |

## 核心概念

| 概念 | 链接 |
|------|------|
| [[Concept 名称]] | 说明 |
```

---

## 第三步：配置 Hermes Agent — 让 AI 认识你的知识库

Hermes Agent 是一个开源的 AI 智能体运行时，能读写文件、执行命令、管理技能、运行定时任务。把它配置为知识库的"大脑"。

### 安装 Hermes

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.zshrc
```

> **Windows 用户注意：** Hermes 不支持原生 Windows，请在 **WSL2**（Windows Subsystem for Linux）中运行。
>
> 1. 安装 WSL2：[微软官方指南](https://learn.microsoft.com/zh-cn/windows/wsl/install)
> 2. 推荐发行版：Ubuntu 22.04 LTS 或更新版本
> 3. 在 WSL2 终端中执行上述安装命令
> 4. Obsidian Vault 建议放在 WSL2 内部（如 `~/wiki/`）而非 Windows 文件系统，避免跨 FS 读写性能损耗
> 5. WereadExporter 也安装在 WSL2 内部，Python 版本需 >= 3.7
>
> 验证安装和后续所有 Hermes 命令都在 WSL2 终端中执行。

### 配置模型提供商

我现在用的是 DeepSeek 在此简单首席配置：

```bash
hermes config set DEEPSEEK_API_KEY sk-xxxxx
hermes chat --provider deepseek --model deepseek-v4-flash
```

其它还可以使用 OpenRouter / 通义千问等 国内外服务商。

### 让 AI 学习你的知识库

启动 Hermes 后，先让它学习知识库的规则：

```
你先学习一下我的知识库结构。知识库在 /Users/{user}/Documents/wiki/obsidian/
先看 _index.md 和 CLAUDE.md（如果有的话），理解目录结构，
然后告诉我你理解了。
```

> **Windows WSL2 路径示例：** 如果 Vault 放在 WSL2 内部，路径为 `/home/{user}/wiki/obsidian/`。如果放在 Windows 文件系统通过 WSL2 访问，路径为 `/mnt/c/Users/{用户名}/Documents/wiki/obsidian/`。

Hermes 会读取文件、理解目录结构、记住你的知识库组织方式。

---

## 实战流程全解

### 场景一：微信读书读完一本书 → 提炼到知识库

**Step 1：导出**

```bash
cd ~/tools/weread-exporter
python -m weread_exporter -b <book_id> -o md
```

导出完成后，把 `cache/<book_id>/chapters/` 下的文件复制到知识库的 `raw/` 目录。

**Step 2：让 AI 提炼**

在 Hermes 中：

```
帮我把 raw/weread-exports/ 下的新资料处理一下。
先阅读每章内容，然后：
1. 识别核心概念，在 wiki/concepts/ 下创建独立页面
2. 识别重要人物/项目，在 wiki/people/ 下创建页面
3. 建立双向链接 —— 比如概念之间互相引用
4. 更新 _index.md，新页面加入表格
5. 在 log.md 中追加一条操作记录
6. 给每个页面打上合适的标签
```

Hermes 会执行一系列操作：读取 raw 文件 → 理解内容 → 创建/更新 wiki 页面 → 建立双向链接 → 更新索引。

**Step 3：检查成果**

切换到 Obsidian 的**关系图谱视图**，你会看到新节点出现在网络中，与已有知识产生关联。

---

### 场景二：日常研究后沉淀知识

遇到一篇好文章或解决了一个技术问题，直接在 Hermes 中说：

```
我刚研究透了 Hermes Agent 的回退机制，
帮我把它固化为一篇 wiki 笔记，放在 wiki/concepts/ 下。
引用已有的相关概念，并创建必要的反向链接。
```

AI 会创建新页面，自动检查已有知识库中是否有相关内容可以链接。

---

### 场景三：基于知识库提问

```
基于我的 wiki 知识库，帮我梳理一下：
Hermes 的工具系统和权限系统之间是怎么配合的？
```

Hermes 会翻阅内部 wiki，综合多篇笔记为你生成带引用的答案。

如果觉得回答不错，可以进一步说：

```
这个总结很好，把它保存为一篇新的 syntheses 笔记。
```

---

### 场景四：知识库体检（Lint）

定期对知识库进行审查：

```
对我的知识库进行 Lint 审查。
检查：孤立页面、断链、内容矛盾、可以合并的概念。
```

AI 会自动扫描全库，报告问题。你可以选择让它自动修复或手动决策。

---

## Hermes 技能库：几个实用 skill

Hermes 有 skill 系统，可以记住常用的知识库操作流程。以下是几个我常用的 skill：

**obsidian** — 读取、搜索、创建 Obsidian 笔记
- 需要查阅已有笔记时，用这个 skill 快速定位
- 创建新笔记时自动遵循目录结构和命名规范

**claude-code** — 委托 Claude Code 进行大型代码分析
- 需要深度分析源码时，交给 Claude Code
- 分析结果提炼到知识库中

**subagent-driven-development** — 用子 Agent 并行执行计划
- 复杂任务拆解，多个子 Agent 同时工作
- 适合大批量知识摄取场景

---

## 知识库管理原则

### 1. 原子化

每个知识点独立成页。一个概念一页、一个人物一页、一个工具一页。通过双向链接形成网络。

### 2. 不要堆资料

一口气让 AI 生成几百个词条但不看，没有意义。**你的阅读量决定了知识库的质量**，AI 只是加速整理。

### 3. 定期审查

知识库会不断长出新节点，也会产生孤立节点和冗余节点。每周做一次 Lint：

```
对我的知识库进行 Lint 审查。
```

### 4. 索引更新是硬约束

每次 AI 创建新页面，都必须在 `_index.md` 中追加一行。这是不可跳过的步骤。

### 5. 日志即时间线

`log.md` 记录每次操作的时间、内容、改动范围。这是知识库的演进史，也是回退的依据。

---

## 完整目录结构参考

```
Documents/wiki/
├── _index.md                  ← MOC 首页
├── CLAUDE.md                  ← Schema 规则（可选）
├── log.md                     ← 操作日志
├── raw/
│   └── weread-exports/
│       ├── 01-第一章.md       ← 书籍原始导出
│       ├── 02-第二章.md
│       └── ...
├── wiki/
│   ├── chapters/
│   │   ├── 01-大主题一.md
│   │   └── 02-大主题二.md
│   ├── concepts/
│   │   ├── 概念A.md
│   │   ├── 概念B.md
│   │   └── ...
│   ├── people/
│   │   ├── 人物A.md
│   │   └── 人物B.md
│   ├── sources/
│   │   └── 来源摘要记录.md
│   ├── syntheses/
│   │   └── 综合分析笔记.md
│   └── tags/
│       └── 标签索引.md
```

---

## 进阶扩展

### 移动端接入

Hermes 支持 17 个消息平台。配置 Telegram / Discord 后，你可以在手机上：

- 发送文章链接 → AI 自动采集并提炼到知识库
- 语音提问 → 基于知识库回答

配置方法：

```bash
hermes gateway start
```

### 定时任务

每天自动检查知识库健康状况：

```
帮我创建一个定时任务，每天早上 9 点检查并汇报知识库的健康状况。
```

Hermes 会创建一个 cron job，每天自动执行 Lint 并推送报告。

### MCP 集成

连接外部 AI 工具到 Hermes：

```bash
hermes mcp add --name my-tool --command "..."
```

---

## 常见问题

### WereadExporter 登录失败

Cookie 过期了，加 `--force-login` 重新登录：

```bash
python -m weread_exporter -b <book_id> -o md --force-login
```

### Chrome 版本不兼容

Chrome 136+ 可能不支持 `--use-default-profile`，去掉这个参数即可。

**Windows 额外注意：** Chrome 驱动路径可能不同。如果遇到 `ChromeDriver` 相关报错，确保 Chrome 已更新到最新稳定版，或在 WSL2 中安装 Chromium：

```bash
sudo apt install chromium-browser
```

### AI 不知道知识库存在

每次新会话或切换模型后，先让 AI 重新学习：

```
先学习我的知识库，知识库在 /Users/{user}/Documents/wiki/obsidian/
读完 _index.md 后告诉我你理解了。
```

**Windows WSL2 路径版本：**

```
先学习我的知识库，知识库在 /mnt/c/Users/{用户名}/Documents/wiki/obsidian/
读完 _index.md 后告诉我你理解了。
```

### 知识库页面越来越多，AI 找不到东西

设计好 `_index.md` 的索引表，AI 回答问题时先扫一眼索引，就能精确定位需要深度阅读的笔记。

---

## Windows 与 macOS 部署差异速查

| 项目 | macOS | Windows (WSL2) |
|------|-------|----------------|
| Hermes Agent | 原生支持，`install.sh` 直接安装 | 需先装 WSL2，在 WSL2 Ubuntu 中安装 |
| WereadExporter 系统依赖 | `brew install cairo pango` | 无需额外安装（内置 DLL） |
| Obsidian Vault 建议位置 | `~/Documents/wiki/` | WSL2 内部 `~/wiki/`（性能更优） |
| Hermes 配置文件路径 | `~/.hermes/config.yaml` | WSL2 内 `~/.hermes/config.yaml` |
| Obsidian 本体 | 原生 macOS App | 原生 Windows App + WSL2 读写同一目录 |
| Chrome 驱动 | 系统自带 | WSL2 内可能需 `sudo apt install chromium-browser` |
| 模型提供商配置 | 无差异 | 无差异 |

> **核心原则：** Obsidian 本体在原生系统上运行，Hermes + WereadExporter 在 WSL2 中运行。Vault 目录建议放在 WSL2 内部，Obsidian 通过 `\\wsl.localhost\Ubuntu\home\{user}\wiki\` 路径打开。这样避免跨文件系统 IO 性能损耗。

---

## 参考资料

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [Obsidian 官网](https://obsidian.md/zh/)
- [WereadExporter GitHub](https://github.com/drunkdream/weread-exporter)
- [Karpathy LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Hermes Agent 快速上手](./HermesAgent快速上手.md)
