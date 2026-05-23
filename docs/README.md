# Hermes Agent 详解

> **Hermes Agent** 是由 [Nous Research](https://nousresearch.com) 构建的自我进化 AI 助手。它是目前唯一一个内置学习循环的 Agent——能够从经验中创造技能、在使用中自我改进、主动保存知识、搜索历史对话，并在多个会话中逐步加深对用户的理解。

---

## 一、项目概述

Hermes Agent 是一个**模块化、多平台的 AI Agent 框架**，核心特点是：

| 特性 | 说明 |
|------|------|
| **自我进化** | 内置 curator 系统，Agent 能从复杂任务中自主创建技能，并持续优化 |
| **多平台接入** | 支持 CLI、Telegram、Discord、Slack、WhatsApp、Signal、Email 等多种界面 |
| **持久记忆** | 通过 Honcho 实现用户建模，FTS5 会话搜索实现跨会话记忆 |
| **灵活部署** | 可运行在本地、Docker、SSH、Modal、Daytona、Vercel 等多种后端 |
| **模型无关** | 支持 OpenRouter、NVIDIA NIM、Xiaomi MiMo、Kimi、MiniMax、HuggingFace、OpenAI 等多种模型 |

---

## 二、核心架构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 — 核心对话循环 (~12k LOC)
├── model_tools.py        # 工具编排，工具发现与调用
├── toolsets.py           # 工具集定义
├── cli.py                # HermesCLI 类 — 交互式 CLI 编排器 (~11k LOC)
├── hermes_state.py       # SessionDB — SQLite 会话存储 (FTS5 搜索)
├── hermes_cli/          # CLI 子命令、设置向导、插件加载器、皮肤引擎
├── agent/               # Agent 内部实现（Provider 适配器、记忆、缓存、压缩等）
├── tools/               # 工具实现 — 通过 tools/registry.py 自动发现
│   └── environments/    # 终端后端（local, docker, ssh, modal, daytona, singularity）
├── gateway/             # 消息网关 — 支持多平台接入
│   └── platforms/       # 各平台适配器（telegram, discord, slack, whatsapp...）
├── plugins/             # 插件系统（记忆提供者、上下文引擎、模型提供者等）
├── skills/              # 内置技能
├── optional-skills/     # 可选技能（需单独安装）
├── ui-tui/              # Ink (React) 终端 UI — hermes --tui
├── tui_gateway/         # Python JSON-RPC 后端（为 TUI 提供服务）
├── cron/                # 定时任务调度器
└── tests/               # pytest 测试套件
```

### 核心文件说明

| 文件 | 职责 |
|------|------|
| `run_agent.py` | AIAgent 类的核心对话循环，处理工具调用、迭代控制、预算跟踪 |
| `model_tools.py` | 工具编排：发现内置工具、处理函数调用 |
| `hermes_state.py` | SQLite 会话数据库，FTS5 全文搜索支持会话检索 |
| `hermes_cli/` | CLI 入口，包含设置向导、插件加载、皮肤系统 |

---

## 三、命令行界面

安装后，通过 `hermes` 命令启动：

```bash
hermes              # 启动交互式 CLI
hermes model        # 选择 LLM 提供者和模型
hermes tools        # 配置启用的工具集
hermes config set   # 设置配置项
hermes gateway      # 启动消息网关（Telegram、Discord 等）
hermes setup        # 运行完整设置向导
hermes update       # 更新到最新版本
hermes doctor       # 诊断问题
```

### 常用斜杠命令

| 命令 | 功能 |
|------|------|
| `/new`, `/reset` | 开始新对话 |
| `/model [provider:model]` | 切换模型 |
| `/personality [name]` | 设置人格 |
| `/retry`, `/undo` | 重试或撤销上一轮 |
| `/compress`, `/usage` | 压缩上下文 / 查看使用量 |
| `/skills` | 查看可用技能 |
| `/stop` | 停止当前工作（消息平台） |

---

## 四、平台与消息网关

Hermes 支持多种消息平台，通过 `hermes gateway` 命令统一管理：

| 平台 | 说明 |
|------|------|
| Telegram | 常用即时通讯 |
| Discord | 社区机器人 |
| Slack | 企业协作工具 |
| WhatsApp | 移动端消息 |
| Signal | 隐私优先 |
| Email | 邮件通知 |
| Home Assistant | 智能家居 |
| Matrix | 去中心化聊天 |
| DingTalk | 钉钉 |
| Feishu | 飞书 |

所有平台共用同一个 Agent 实例，实现**跨平台对话连续性**。

---

## 五、技能系统

Hermes 拥有强大的技能（Skills）系统：

### 5.1 内置技能 (`skills/`)

开箱即用，涵盖 GitHub、MLOps 等领域。

### 5.2 可选技能 (`optional-skills/`)

需要手动安装：
```bash
hermes skills install official/<category>/<skill>
```

### 5.3 自动进化

- **Curator 系统**：后台自动追踪 Agent 创建的技能使用情况
- **自动归档**：长期未使用的技能会被自动归档，保留在 `~/.hermes/skills/.archive/`
- **技能自改进**：技能在使用过程中会根据反馈持续优化

### 5.4 技能创作标准

新技能必须满足：
- 描述 ≤ 60 字符，一句话结尾用句号
- SKILL.md 使用标准 frontmatter 格式
- 脚本放在 `scripts/` 目录
- 测试文件位于 `tests/skills/test_<skill>_skill.py`

---

## 六、工具系统

### 6.1 工具集 (Toolsets)

工具按功能分组为工具集，定义在 `toolsets.py`：

| 工具集 | 功能 |
|--------|------|
| `terminal` | 本地/远程 shell 命令执行 |
| `browser` | 网页浏览与交互 |
| `file` | 文件操作（读、写、搜索） |
| `code_execution` | Python/JS 代码执行 |
| `delegation` | 委托子 Agent 并行工作 |
| `memory` | 记忆存储与检索 |
| `search` | 网络搜索 |
| `vision` | 图像分析与生成 |
| `tts` | 语音合成 |
| `cronjob` | 定时任务 |
| `messaging` | 消息发送 |

### 6.2 添加自定义工具

**方式一：插件（推荐）**
```
~/.hermes/plugins/<name>/plugin.yaml
~/.hermes/plugins/<name>/__init__.py
```

**方式二：内置工具（需贡献代码）**
1. 创建 `tools/your_tool.py`
2. 在 `toolsets.py` 中注册到工具集

---

## 七、定时任务 (Cron)

Hermes 内置调度器，支持自然语言创建定时任务：

```bash
hermes cron list    # 列出所有定时任务
hermes cron add    # 添加新任务
hermes cron edit   # 编辑任务
hermes cron remove # 删除任务
```

### 支持的时间格式

| 格式 | 示例 |
|------|------|
| 持续时间 | `"30m"`, `"2h"`, `"1d"` |
| 自然语言 | `"every monday 9am"` |
| Cron 表达式 | `"0 9 * * *"` |
| ISO 时间戳 | `"2026-06-01T09:00:00Z"` |

---

## 八、多实例与配置文件

### 8.1 配置文件位置

- `~/.hermes/config.yaml` — 主配置
- `~/.hermes/.env` — API 密钥等敏感信息
- `~/.hermes/logs/` — 日志文件

### 8.2 多配置文件（Profiles）

Hermes 支持多个完全隔离的实例：

```bash
hermes -p <profile_name>  # 使用指定配置
hermes profile list       # 列出所有配置
hermes profile create     # 创建新配置
```

每个 Profile 拥有独立的 `HERMES_HOME` 目录（配置、记忆、会话、技能等完全隔离）。

---

## 九、启动方式详解

Hermes Agent 支持多种启动方式，适应不同场景和平台。

### 方式一：交互式 CLI（最常用）

直接启动交互式对话界面。

```bash
hermes
```

| 依赖 | 说明 |
|------|------|
| Python 3.11+ | 核心运行环境 |
| uv | 包管理器（自动安装） |
| API Key | `.env` 中配置（如 `OPENROUTER_API_KEY`） |

**准备步骤：**
1. 运行 `hermes setup` 配置 API Key
2. 运行 `hermes model` 选择模型
3. 运行 `hermes tools` 配置工具集

---

### 方式二：开发模式（从源码）

适合贡献者或需要修改代码的用户。

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh
./hermes
```

| 依赖 | 说明 |
|------|------|
| Git | 代码版本控制 |
| curl | 下载安装脚本 |
| Python 3.11+ | 运行时 |
| uv | 自动化安装脚本会安装 |
| Node.js | TUI 前端构建 |

**准备步骤：**
1. 克隆代码仓库
2. 运行 `setup-hermes.sh` 自动完成：
   - 安装/定位 uv
   - 创建 Python venv
   - 安装所有依赖（`.[all]`）
   - 同步技能到 `~/.hermes/skills/`
   - 创建 `hermes` 命令软链接
3. 运行 `hermes setup` 配置 API Key

---

### 方式三：Docker 容器（推荐生产环境）

隔离环境，一键部署。

```bash
# 构建镜像
docker build -t hermes-agent .

# 运行容器
HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d
```

| 依赖 | 说明 |
|------|------|
| Docker | 容器运行时 |
| docker-compose | 容器编排 |

**准备步骤：**
1. 确保 `~/.hermes` 目录存在（挂载到 `/opt/data`）
2. 配置 `~/.hermes/.env` 写入 API Key
3. 配置 `~/.hermes/config.yaml`（可选）

**Dockerfile 关键特性：**
- 基于 `debian:13.4`
- 使用 `gosu` 实现权限降级
- 预装 uv、nodejs、npm、ripgrep、ffmpeg 等
- TUI 和 Dashboard 已预构建
- 支持 `HERMES_UID`/`HERMES_GID` 映射主机用户

**docker-compose 服务：**
| 服务 | 端口 | 说明 |
|------|------|------|
| gateway | - | 消息网关（Telegram、Discord 等） |
| dashboard | 9119 | Web 界面 |

**环境变量：**
```bash
HERMES_UID=$(id -u)        # 主机用户 UID
HERMES_GID=$(id -g)        # 主机用户 GID
HERMES_DASHBOARD=1         # 启动 dashboard
HERMES_DASHBOARD_HOST=0.0.0.0  # dashboard 监听地址
HERMES_DASHBOARD_PORT=9119     # dashboard 端口
```

---

### 方式四：Docker 直接运行

无需 docker-compose，单容器启动。

```bash
# 基础运行
docker run -v ~/hermes_data:/opt/data \
  -e OPENROUTER_API_KEY=sk-xxx \
  nousresearch/hermes-agent

# 带 Dashboard
docker run -v ~/hermes_data:/opt/data \
  -e OPENROUTER_API_KEY=sk-xxx \
  -e HERMES_DASHBOARD=1 \
  -p 9119:9119 \
  nousresearch/hermes-agent

# 交互式 Shell
docker run -v ~/hermes_data:/opt/data \
  -e OPENROUTER_API_KEY=sk-xxx \
  -it nousresearch/hermes-agent bash
```

| 依赖 | 说明 |
|------|------|
| Docker | 容器运行时 |

**准备步骤：**
1. 准备宿主机数据目录 `~/hermes_data`
2. 配置 `~/hermes_data/.env`（包含 API Key）
3. 配置 `~/hermes_data/config.yaml`（可选）

---

### 方式五：消息网关模式

将 Hermes 接入即时通讯平台，实现远程对话。

```bash
# 方式 A：使用 systemd 服务
hermes gateway install
hermes gateway start

# 方式 B：Docker 模式
docker run ... nousresearch/hermes-agent gateway run
```

| 依赖 | 说明 |
|------|------|
| 消息平台账号 | Telegram Bot / Discord Bot 等 |
| API Key | 平台 Bot Token |
| 网络配置 | 允许 Webhook 回调 |

**准备步骤：**
1. 在目标平台创建 Bot 获取 Token
2. 运行 `hermes gateway setup`
3. 配置平台连接信息
4. 运行 `hermes gateway start`

**支持的平台：**
- Telegram、Discord、Slack、WhatsApp、Signal
- Email、SMS、Matrix、Home Assistant
- DingTalk、Feishu、钉钉、企业微信

---

### 方式六：独立 Python 模块

在 Python 代码中直接使用 AIAgent 类。

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-xxx",
    model="anthropic/claude-opus-4.6"
)
response = agent.run_conversation("Hello, how are you?")
print(response)
```

| 依赖 | 说明 |
|------|------|
| Python 3.11+ | 运行环境 |
| openai | API 客户端库 |
| API Key | LLM 服务 API Key |

---

### 方式七：TUI 模式（富文本界面）

带语法高亮和更好可视化的终端界面。

```bash
hermes --tui
# 或
HERMES_TUI=1 hermes
```

| 依赖 | 说明 |
|------|------|
| Node.js + npm | Ink TUI 前端构建 |
| Python + rich | 终端渲染 |

**注意：** Windows 原生不支持 PTY，WSL2 或 Docker 环境下使用更佳。

---

### 方式八：Dashboard Web 界面

浏览器中的图形化界面。

```bash
hermes dashboard
# 访问 http://localhost:9119
```

| 依赖 | 说明 |
|------|------|
| FastAPI | Web 框架 |
| Uvicorn | ASGI 服务器 |
| Node.js | 前端构建 |

**Docker 启动 Dashboard：**
```bash
HERMES_DASHBOARD=1 docker compose up -d
# 访问 http://localhost:9119
```

---

### 方式九：定时任务（Cron）

无人值守的自动化任务。

```bash
hermes cron add "every monday 9am" --prompt "生成周报"
hermes cron list
```

| 依赖 | 说明 |
|------|------|
| croniter | Cron 表达式解析 |
| 消息平台（可选） | 任务结果通知 |

**准备：** 配置好消息平台后，任务结果可推送到 Telegram/Discord 等。

---

### 方式十：Windows PowerShell

Windows 原生安装（早期测试阶段）。

```powershell
# PowerShell 安装
iex (irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1)

# 启动
hermes
```

| 依赖 | 说明 |
|------|------|
| Windows 10+ | 操作系统 |
| PowerShell 5+ | 脚本执行 |
| MinGit | 捆绑的 Git Bash |

**注意：** Windows 原生支持仍为早期测试版，WSL2 更稳定。

---

## 十、配置文件速查

| 路径 | 说明 |
|------|------|
| `~/.hermes/config.yaml` | 主配置文件 |
| `~/.hermes/.env` | API 密钥（敏感信息） |
| `~/.hermes/logs/` | 日志目录 |
| `~/.hermes/sessions/` | 会话数据 |
| `~/.hermes/skills/` | 技能目录 |
| `~/.hermes/memories/` | 记忆存储 |
| `~/.hermes/skins/` | 自定义皮肤 |
| `~/.hermes/plugins/` | 插件目录 |

---

## 十一、依赖管理体系

Hermes Agent 采用**两层依赖管理策略**：核心依赖 + 按需加载的可选依赖。

### 11.1 核心依赖（必须安装）

核心依赖在 `setup-hermes.sh` 或 `pip install -e ".[all]"` 时安装，是运行 Hermes 的最低要求。

| 包 | 版本 | 说明 |
|---|------|------|
| openai | 2.24.0 | OpenAI 兼容 API 客户端 |
| python-dotenv | 1.2.2 | .env 文件读取 |
| fire | 0.7.1 | 配置管理 |
| httpx[socks] | 0.28.1 | HTTP 客户端 |
| rich | 14.3.3 | 终端美化输出 |
| tenacity | 9.1.4 | 重试机制 |
| pyyaml | 6.0.3 | YAML 配置解析 |
| requests | 2.33.0 | HTTP 请求库（CVE 修复） |
| jinja2 | 3.1.6 | 模板引擎 |
| pydantic | 2.13.4 | 数据验证 |
| prompt_toolkit | 3.0.52 | 交互式 CLI |
| croniter | 6.0.0 | Cron 表达式解析 |
| PyJWT[crypto] | 2.12.1 | JWT 认证 |
| psutil | 7.2.2 | 进程管理 |

**安装核心依赖：**
```bash
# 方式一：setup 脚本（推荐）
./setup-hermes.sh

# 方式二：uv（开发）
uv sync --extra all

# 方式三：pip
pip install -e ".[all]"
```

---

### 11.2 可选依赖（按需加载）

Hermes Agent 使用 **Lazy Loading 机制**，大部分可选依赖在首次使用时自动安装，无需事前声明。

#### 11.2.1 Extra 依赖组

通过 `pip install -e ".[extra_name]"` 安装：

| Extra | 功能 | 包 |
|-------|------|-----|
| `cli` | 交互式菜单 | simple-term-menu |
| `dev` | 开发测试 | pytest, debugpy, ruff, mcp |
| `pty` | 终端支持 | ptyprocess (Linux) / pywinpty (Win) |
| `mcp` | MCP 协议支持 | mcp |
| `honcho` | Honcho 记忆 | honcho-ai |
| `acp` | ACP 协议 | agent-client-protocol |
| `homeassistant` | Home Assistant | aiohttp |
| `sms` | SMS 支持 | aiohttp |
| `google` | Google 工作区 | google-api-python-client 等 |
| `web` | Dashboard | fastapi, uvicorn |
| `youtube` | YouTube 字幕 | youtube-transcript-api |

#### 11.2.2 Lazy Loading 依赖（首次使用时自动安装）

以下依赖在首次使用时由 `tools/lazy_deps.py` 自动检测并安装：

| Feature Key | 功能 | 依赖包 |
|-------------|------|--------|
| `provider.anthropic` | Anthropic 原生 SDK | anthropic==0.87.0 |
| `provider.bedrock` | AWS Bedrock | boto3==1.42.89 |
| `provider.azure_identity` | Azure Entra ID | azure-identity==1.25.3 |
| `search.exa` | Exa 搜索 | exa-py==2.10.2 |
| `search.firecrawl` | Firecrawl 搜索 | firecrawl-py==4.17.0 |
| `search.parallel` | 并行搜索 | parallel-web==0.4.2 |
| `tts.edge` | Edge TTS | edge-tts==7.2.7 |
| `tts.elevenlabs` | ElevenLabs TTS | elevenlabs==1.59.0 |
| `stt.faster_whisper` | 本地语音识别 | faster-whisper, sounddevice, numpy |
| `image.fal` | 图像生成 | fal-client==0.13.1 |
| `memory.honcho` | Honcho 记忆 | honcho-ai==2.0.1 |
| `memory.hindsight` | Hindsight 记忆 | hindsight-client==0.6.1 |
| `platform.telegram` | Telegram 接入 | python-telegram-bot |
| `platform.discord` | Discord 接入 | discord.py, brotlicffi |
| `platform.slack` | Slack 接入 | slack-bolt, slack-sdk |
| `platform.matrix` | Matrix 接入 | mautrix |
| `platform.dingtalk` | 钉钉接入 | dingtalk-stream |
| `platform.feishu` | 飞书接入 | lark-oapi |
| `terminal.modal` | Modal 后端 | modal |
| `terminal.daytona` | Daytona 后端 | daytona |
| `terminal.vercel` | Vercel 后端 | vercel |
| `skill.google_workspace` | Google 工作区技能 | google-api-python-client 等 |
| `skill.youtube` | YouTube 技能 | youtube-transcript-api |
| `tool.acp` | ACP 工具 | agent-client-protocol |
| `tool.dashboard` | Dashboard | fastapi, uvicorn |

**Lazy Loading 安全机制：**
- 仅安装白名单中的包（`LAZY_DEPS` 字典）
- 安装到当前 venv，不影响系统 Python
- 可通过 `security.allow_lazy_installs: false` 禁用

---

### 11.3 安装方式对比

| 场景 | 推荐方式 | 命令 |
|------|---------|------|
| 最小化安装（仅 CLI） | 默认 core | `pip install -e "."` |
| 完整功能 | `[all]` | `uv sync --extra all` |
| 特定平台（如 Telegram） | Lazy Loading | 首次使用 Telegram 时自动安装 |
| 开发调试 | `[dev]` | `pip install -e ".[dev]"` |
| Docker 生产环境 | 预构建镜像 | `docker compose up -d` |

---

### 11.4 依赖版本锁定策略

Hermes Agent 采用**精确版本锁定**策略防止供应链攻击：

| 来源 | 策略 | 示例 |
|------|------|------|
| PyPI 包 | `==exact` 精确锁定 | `openai==2.24.0` |
| Git URL | 提交 SHA | `git+https://...@<sha>` |
| GitHub Actions | SHA + 注释 | `actions/checkout@sha # v4` |

**为什么不用范围锁定？**
> 2026-05-12，PyPI 上的 `mistralai` 包被植入恶意代码（Mini Shai-Hulud 蠕虫）。精确锁定确保只有经过代码审查的新版本才能到达用户。

---

### 11.5 管理命令

```bash
# 查看已安装的依赖
pip list | grep hermes

# 更新 hermes（保留依赖）
hermes update

# 查看工具配置
hermes tools

# 手动安装可选依赖
uv pip install -e ".[telegram]"  # Telegram 平台

# 查看 lazy deps 状态
# 首次使用某功能时自动安装，查看日志确认
hermes logs --level info
```

---

### 11.6 Docker 环境依赖

Docker 镜像中已预装所有依赖：

```dockerfile
# 基础环境
FROM debian:13.4

# 预装系统包
RUN apt-get install -y build-essential curl nodejs npm python3 ripgrep ffmpeg gcc python3-dev libffi-dev procps git openssh-client docker-cli tini

# Python 依赖（通过 uv sync --frozen 安装）
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --extra all --extra messaging
```

---

## 十二、相关资源

| 资源 | 链接 |
|------|------|
| 官方文档 | https://hermes-agent.nousresearch.com/docs/ |
| Discord 社区 | https://discord.gg/NousResearch |
| Skills Hub | https://agentskills.io |
| GitHub Issues | https://github.com/NousResearch/hermes-agent/issues |

---

## 十三、全量 Provider 配置指南

### 13.1 Provider 总览

| Provider | 模型示例 | 端点 | 环境变量 |
|----------|----------|------|---------|
| `openrouter` | claude-opus-4.6, gpt-4o | https://openrouter.ai/api/v1 | `OPENROUTER_API_KEY` |
| `openai-codex` | gpt-4o, o1 | OpenAI 官方 | OAuth (hermes auth) |
| `anthropic` | claude-opus-4.6, claude-sonnet-4 | https://api.anthropic.com | `ANTHROPIC_API_KEY` |
| `bedrock` | Claude 3 on AWS | AWS 区域端点 | AWS IAM / boto3 |
| `azure-foundry` | GPT-4o on Azure | Azure OpenAI 端点 | `AZURE_API_KEY` 或 Entra ID |
| `gemini` | gemini-2.5-pro | https://generativelanguage.googleapis.com/v1beta/openai | `GOOGLE_API_KEY` |
| `deepseek` | deepseek-chat, deepseek-coder | https://api.deepseek.com | `DEEPSEEK_API_KEY` |
| `kimi-coding` | kimi-k2.5, moonshot-v1-128k | https://api.kimi.com/coding/v1 | `KIMI_API_KEY` |
| `kimi-coding-cn` | Kimi 中国版 | https://api.moonshot.cn/v1 | `KIMI_CN_API_KEY` |
| `minimax` | MiniMax-M2.7 | https://api.minimax.io/anthropic | `MINIMAX_API_KEY` |
| `minimax-cn` | MiniMax-M2.7 | https://api.minimaxi.com/anthropic | `MINIMAX_CN_API_KEY` |
| `novita` | novita-* | https://api.novita.ai/v1 | `NOVITA_API_KEY` |
| `nvidia` | nemotron-4-340b | https://build.nvidia.com | `NVIDIA_API_KEY` |
| `xiaomi` | MiMo | https://api.xiaomi.com | `XIAOMI_API_KEY` |
| `xai` | grok-2, grok-2-mini | https://api.x.ai/v1 | `XAI_API_KEY` |
| `zai` | GLM-4-Plus, GLM-4V | https://api.z.ai/api/paas/v4 | `ZAI_API_KEY` |
| `alibaba` | qwen-turbo, qwen-plus | 阿里云 DashScope | `ALIBABA_API_KEY` |
| `qwen-oauth` | Qwen OAuth | 阿里云 OAuth | OAuth Flow |
| `stepfun` | step-1v, step-1o | https://api.stepfun.com | `STEPFUN_API_KEY` |
| `arcee` | trinity-mini, trinity-large | https://api.arcee.ai | `ARCEEAI_API_KEY` |
| `ollama-cloud` | llama3.1, qwen2.5 | https://ollama.com/v1 | `OLLAMA_API_KEY` |
| `huggingface` | inference endpoints | https://api-inference.huggingface.co | `HF_TOKEN` |
| `ai-gateway` | Vercel AI Gateway | https://api.aifoundation.com | `AI_GATEWAY_API_KEY` |
| `opencode-zen` | GPT/GLM/Gemini/Kimi/MiniMax | https://opencode.ai/zen/v1 | `OPENCODE_ZEN_API_KEY` |
| `custom` | 任意 OpenAI 兼容端点 | 用户自定义 | 通过 `key_env` 或 `api_key` 配置 |

---

### 13.2 MiniMax 配置示例（已验证）

**config.yaml:**
```yaml
model:
  provider: "minimax-cn"
  default: "MiniMax-M2.7"
  base_url: "https://api.minimaxi.com/v1"
```

**.env:**
```
MINIMAX_CN_API_KEY=sk-cp-你的API密钥
```

---

### 13.3 OpenRouter（多模型聚合）

**config.yaml:**
```yaml
model:
  provider: "openrouter"
  default: "anthropic/claude-opus-4.6"
  base_url: "https://openrouter.ai/api/v1"
```

**.env:**
```
OPENROUTER_API_KEY=sk-or-v1-你的API密钥
```

---

### 13.4 OpenAI (含 Codex)

**config.yaml:**
```yaml
model:
  provider: "openai-codex"
  default: "gpt-4o"
```

**.env:**
```
# 需要运行 hermes auth 进行 OAuth 认证
# 或手动设置 ANTHROPIC_API_KEY 等效字段
```

---

### 13.5 Anthropic (原生)

**config.yaml:**
```yaml
model:
  provider: "anthropic"
  default: "claude-opus-4.6"
  base_url: "https://api.anthropic.com"
```

**.env:**
```
ANTHROPIC_API_KEY=sk-ant-你的API密钥
```

---

### 13.6 Google Gemini

**config.yaml:**
```yaml
model:
  provider: "gemini"
  default: "gemini-2.5-pro"
  base_url: "https://generativelanguage.googleapis.com/v1beta/openai"
```

**.env:**
```
GOOGLE_API_KEY=你的Google AI Studio密钥
# 或 GEMINI_API_KEY=你的密钥
```

---

### 13.7 DeepSeek

**config.yaml:**
```yaml
model:
  provider: "deepseek"
  default: "deepseek-chat"
  base_url: "https://api.deepseek.com"
```

**.env:**
```
DEEPSEEK_API_KEY=sk-你的API密钥
```

---

### 13.8 Kimi / Moonshot

**config.yaml:**
```yaml
model:
  provider: "kimi-coding"
  default: "kimi-k2.5"
  base_url: "https://api.kimi.com/coding/v1"
```

**.env:**
```
KIMI_API_KEY=sk-kimi-你的API密钥
```

**中国版 (moonshot.cn):**
```yaml
model:
  provider: "kimi-coding-cn"
  default: "moonshot-v1-128k"
  base_url: "https://api.moonshot.cn/v1"
```

**.env:**
```
KIMI_CN_API_KEY=sk-kimi-cn-你的API密钥
```

---

### 13.9 AWS Bedrock

**config.yaml:**
```yaml
model:
  provider: "bedrock"
  default: "anthropic.claude-3-5-sonnet-20241022-v1:0"
  # base_url 通常不需要，boto3 自动解析区域端点
```

**.env:**
```
# AWS 凭证通过以下任一方式配置：
# 1. aws configure (AWS Access Key + Secret)
# 2. IAM Role (EC2/ECS/Lambda)
# 3. 环境变量 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
# 4. boto3 默认凭证链
```

---

### 13.10 Azure OpenAI Foundry

**config.yaml:**
```yaml
model:
  provider: "azure-foundry"
  default: "gpt-4o"
  base_url: "https://你的资源.openai.azure.com/openai/v1"
  auth_mode: "entra_id"  # Entra ID / Managed Identity / az login
```

**.env:**
```
# Entra ID 认证不需要 API Key
# 密钥认证使用:
AZURE_API_KEY=你的密钥
```

---

### 13.11 自定义端点 (Custom)

适用于 Ollama、本地 vLLM、第三方兼容 API。

**config.yaml:**
```yaml
model:
  provider: "custom"
  default: "你的模型名"
  base_url: "http://localhost:11434/v1"
custom_providers:
  - name: "my-local"
    base_url: "http://localhost:11434/v1"
    key_env: "CUSTOM_API_KEY"  # 推荐：引用环境变量
    # api_key: "直接写密钥"    # 不推荐：明文密钥
```

**.env:**
```
CUSTOM_API_KEY=ollama 或空（本地无需认证）
```

**常见本地端点:**
| 服务 | base_url |
|------|----------|
| Ollama | `http://localhost:11434/v1` |
| vLLM | `http://localhost:8000/v1` |
| LM Studio | `http://localhost:1234/v1` |
| LocalAI | `http://localhost:8080/v1` |

---

### 13.12 Novita AI

**config.yaml:**
```yaml
model:
  provider: "novita"
  default: "novita-ai/your-model"
  base_url: "https://api.novita.ai/v1"
```

**.env:**
```
NOVITA_API_KEY=你的API密钥
```

---

### 13.13 NVIDIA NIM

**config.yaml:**
```yaml
model:
  provider: "nvidia"
  default: "nemotron-4-340b-instruct"
  base_url: "https://integrate.api.nvidia.com/v1"
```

**.env:**
```
NVIDIA_API_KEY=nvapi-你的密钥
```

---

### 13.14 Xiaomi MiMo

**config.yaml:**
```yaml
model:
  provider: "xiaomi"
  default: "MiMo-7B"
  base_url: "https://api.xiaomi.com"
```

**.env:**
```
XIAOMI_API_KEY=你的API密钥
```

---

### 13.15 xAI (Grok)

**config.yaml:**
```yaml
model:
  provider: "xai"
  default: "grok-2"
  base_url: "https://api.x.ai/v1"
```

**.env:**
```
XAI_API_KEY=你的API密钥
```

---

### 13.16 z.ai / GLM

**config.yaml:**
```yaml
model:
  provider: "zai"
  default: "GLM-4-Plus"
  base_url: "https://api.z.ai/api/paas/v4"
```

**.env:**
```
ZAI_API_KEY=你的API密钥
```

---

### 13.17 Hugging Face Inference

**config.yaml:**
```yaml
model:
  provider: "huggingface"
  default: "meta-llama/Llama-3-70b-instruct"
  base_url: "https://api-inference.huggingface.co/v1"
```

**.env:**
```
HF_TOKEN=hf_你的密钥
```

---

### 13.18 配置优先级

Hermes 配置 provider 的优先级（从高到低）：

1. **config.yaml 中 `model` 下的直接配置**（`provider`, `default`, `base_url`）
2. **config.yaml 中 `custom_providers` 定义**（通过 `key_env` 引用环境变量）
3. **环境变量**（`.env` 文件中的 `*_API_KEY`）
4. **命令行参数**（`hermes model` 切换）

---

### 13.19 常用配置命令

```bash
# 查看当前模型配置
hermes model

# 切换 provider
hermes config set model.provider minimax-cn

# 设置默认模型
hermes config set model.default MiniMax-M2.7

# 设置 API Key
hermes config set MINIMAX_CN_API_KEY 你的密钥

# 查看完整配置
hermes config show

# 诊断配置问题
hermes doctor
```

---

### 13.20 安全建议

| 建议 | 说明 |
|------|------|
| 使用 `key_env` | 优先使用 `key_env` 引用环境变量，而非明文 `api_key` |
| 保护 .env 文件 | `chmod 600 ~/.hermes/.env` |
| 加入 .gitignore | 确保 `~/.hermes/` 不被提交到版本控制 |
| 生产环境 | 考虑使用密钥管理服务（AWS Secrets Manager、Azure Key Vault 等） |

---

## 十四、快速参考

```bash
# 启动 CLI
hermes

# 配置模型
hermes model

# 启动 Telegram Bot
hermes gateway setup
hermes gateway start

# 安装技能
hermes skills install official/github/repo-insights

# 定时任务
hermes cron add "every day 9am" --prompt "给我写日报"

# 查看日志
hermes logs --follow
```

### 13.21 Batch Runner 批处理命令

Hermes 提供 `batch_runner.py` 用于批量处理 QA 任务，支持两种运行方式。

---

#### 方式一：源码版本

直接从 hermes-agent 源码目录运行：

```bash
/home/tony/projects/tut_hermes/hermes-agent-main/venv/bin/python \
  /home/tony/projects/tut_hermes/hermes-agent-main/batch_runner.py \
  --dataset_file=/home/tony/projects/tut_hermes/hermes-agent-main/datagen-config-examples/questions.jsonl \
  --batch_size=2 \
  --run_name=qa_test \
  --base_url=https://api.minimaxi.com/v1 \
  --api_key=$MINIMAX_CN_API_KEY \
  --model=MiniMax-M2.7
```

| 要求 | 说明 |
|------|------|
| 源码目录 | `hermes-agent-main/` 下有 `batch_runner.py` |
| Python 环境 | 源码目录下的 venv（`venv/bin/python`） |
| 配置文件 | `~/.hermes/config.yaml` + `~/.hermes/.env` |

---

#### 方式二：Whl 库版本（已验证）

在 hermes_pkg 目录下安装后运行：

```bash
cd /home/tony/projects/tut_hermes/hermes_pkg

# 安装 hermes-agent 到当前环境
pip install -e .

# 运行批处理
python .venv/lib/python3.12/site-packages/batch_runner.py \
  --dataset_file=/home/tony/projects/tut_hermes/hermes-agent-main/datagen-config-examples/questions.jsonl \
  --batch_size=2 \
  --run_name=qa_test \
  --base_url=https://api.minimaxi.com/v1 \
  --api_key=$MINIMAX_CN_API_KEY \
  --model=MiniMax-M2.7
```

**前置准备：**

1. **配置模型**（`~/.hermes/config.yaml`）：
   ```yaml
   model:
     provider: "minimax-cn"
     default: "MiniMax-M2.7"
     base_url: "https://api.minimaxi.com/v1"
   ```

2. **配置 API Key**（`~/.hermes/.env`）：
   ```
   MINIMAX_CN_API_KEY=sk-cp-你的密钥
   ```

---

**通用参数说明：**

| 参数 | 说明 |
|------|------|
| `--dataset_file` | JSONL 数据文件路径，每行包含 `{"prompt": "问题内容"}` |
| `--batch_size` | 每批处理的 prompt 数量 |
| `--run_name` | 任务名称，用于生成输出目录 |
| `--base_url` | API 端点 URL |
| `--api_key` | API 密钥（推荐用 `$MINIMAX_CN_API_KEY` 环境变量） |
| `--model` | 模型名称 |
| `--num_workers` | 并行工作进程数（默认 4） |
| `--resume` | 从 checkpoint 恢复中断的任务 |

**输入数据格式 (JSONL)：**
```jsonl
{"prompt": "问题1：什么是人工智能？"}
{"prompt": "问题2：机器学习和深度学习有什么区别？"}
{"prompt": "问题3：解释监督学习的概念"}
```

**输出目录结构：**
```
data/<run_name>/
├── batch_0.jsonl      # 第 1 批轨迹
├── batch_1.jsonl      # 第 2 批轨迹
├── trajectories.jsonl  # 合并后的完整轨迹
├── checkpoint.json     # 断点记录（恢复用）
└── statistics.json     # 统计汇总
```

**Checkpoint 机制说明：**

- 每次运行结束后保存 `checkpoint.json`，记录已完成的所有 prompt 索引
- 下次运行时自动跳过已完成的批次（显示 "Already completed (skipping)"）
- **注意**：如果上一次运行中断导致无输出但 checkpoint 已记录，会误认为已完成
- 如需重新处理，使用新 `--run_name`（会创建新目录），或手动删除 checkpoint.json

**输出格式（每行一个完整轨迹）：**
```json
{
  "prompt_index": 0,
  "conversations": [
    {"role": "user", "content": "问题：什么是AI？"},
    {"role": "assistant", "content": "...", "reasoning": "..."},
    {"role": "assistant", "content": null, "tool_calls": [...]},
    {"role": "tool", "content": "...", "tool_call_id": "..."}
  ],
  "metadata": {"batch_num": 0, "timestamp": "2026-05-23T..."},
  "completed": true,
  "api_calls": 5,
  "toolsets_used": ["terminal", "memory"],
  "tool_stats": {"terminal": {"count": 1, "success": 1, "failure": 0}}
}
```

**常见问题排查：**

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| "Already completed" 但无输出 | 上次中断，checkpoint 有记录但未生成轨迹 | 删除 checkpoint.json 或用新 run_name |
| `Could not import tool module` | 缺少可选依赖（如 websockets） | `pip install websockets` 或忽略（不影响核心功能） |
| 0 entries in trajectories.jsonl | 数据集为空或格式错误 | 确认 JSONL 每行有 `{"prompt": ...}` |

---

*本文档由 Hermes Agent 项目分析生成，涵盖核心功能、架构设计与使用指南。*