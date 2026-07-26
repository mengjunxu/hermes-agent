# Hermes Agent 项目分析

## 一、项目概览

**Hermes Agent** 是 [Nous Research](https://nousresearch.com) 开发的自我进化型个人 AI 代理。

| 属性 | 值 |
|---|---|
| 版本 | 0.19.0 |
| 许可证 | MIT |
| 主语言 | Python 3.11–3.13 |
| TS/JS | TUI (Ink/React)、桌面应用 (Electron)、Web Dashboard |
| 包管理 | uv (Python), npm workspaces (JS/TS) |
| Python 源文件 | ~560 个（agent/tools/gateway/hermes_cli） |
| 测试文件 | ~2270 个（约 17k 测试用例） |
| 模型提供商插件 | 31 个 |
| 记忆后端插件 | 8 个 |
| 消息平台适配器 | 20+ 个 |

### 两个核心设计原则

1. **对话级 Prompt 缓存是神圣不可侵犯的。** 每轮对话复用缓存的 system prompt 前缀。任何在对话中途修改历史上下文、切换工具集、重建系统提示的操作都会使缓存失效，成倍增加成本。唯一的例外是上下文压缩。
2. **核心是窄腰，能力在边缘。** 每个模型工具在每次 API 调用时都会发送，因此新增核心工具的门槛极高。新能力应以 CLI 命令+技能、服务门控工具、插件、MCP 服务器的方式接入，而非直接写入核心。

---

## 二、架构分析

### 核心文件（骨架）

```
run_agent.py          → AIAgent 类 — 核心对话循环 (~12k 行)
cli.py                → HermesCLI 类 — 交互式 CLI 编排器 (~11k 行)
model_tools.py        → 工具编排层 + discover_builtin_tools() + handle_function_call()
toolsets.py           → 工具集定义 (TOOLSETS 字典 + _HERMES_CORE_TOOLS 列表)
hermes_state.py       → SessionDB — SQLite 会话存储 (FTS5 全文搜索)
hermes_constants.py   → get_hermes_home() — 多配置文件感知路径
hermes_logging.py     → 日志系统 (agent.log / errors.log / gateway.log)
```

### 对话循环

```
用户消息 → AIAgent.run_conversation()
  ├── 构建系统提示 (agent/prompt_builder.py)
  ├── 构建 API 参数 (模型、消息、工具、推理配置)
  ├── 调用 LLM (OpenAI 兼容 API)
  ├── 如果响应包含 tool_calls:
  │     ├── 通过工具注册表分发执行 (model_tools.py)
  │     ├── 将工具结果加入对话
  │     └── 循环回 LLM 调用
  ├── 如果是文本响应:
  │     ├── 持久化会话到 SQLite
  │     └── 返回最终响应
  └── 接近 token 上限时进行上下文压缩
```

### 文件依赖链

```
tools/registry.py  (无依赖 — 被所有工具文件导入)
       ↑
tools/*.py  (每个在导入时调用 registry.register())
       ↑
model_tools.py  (导入 tools/registry 并触发工具发现)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### 模块分析

#### `agent/` — 代理内核（~120+ Python 文件）

这是项目最核心的目录，包含：

| 子系统 | 关键文件 | 说明 |
|---|---|---|
| **系统提示构建** | `prompt_builder.py`, `system_prompt.py` | 组装身份、技能、上下文文件、记忆 |
| **上下文压缩** | `context_compressor.py`, `conversation_compression.py` | 接近 token 限制时自动摘要 |
| **辅助 LLM** | `auxiliary_client.py` | 摘要、视觉、嵌入、标题生成等辅助任务 |
| **模型适配器** | `anthropic_adapter.py`, `gemini_native_adapter.py`, `codex_responses_adapter.py`, `vertex_adapter.py`, `bedrock_adapter.py`, `azure_identity_adapter.py` | 多提供商原生 API 适配 |
| **凭据管理** | `credential_pool.py`, `credential_sources.py`, `credential_persistence.py` | 多账号轮换、配额跟踪 |
| **记忆系统** | `memory_manager.py`, `memory_provider.py` | 可插拔记忆后端 ABC + 编排器 |
| **技能管理** | `curator.py`, `skill_commands.py`, `skill_preprocessing.py`, `skill_utils.py` | 技能生命周期、后台维护 |
| **显示** | `display.py` | KawaiiSpinner 动画面孔、工具进度 |
| **模型元数据** | `model_metadata.py` | 上下文长度、token 估算 |
| **速率限制** | `rate_limit_tracker.py`, `nous_rate_guard.py` | 配额跟踪、断路器 |
| **迭代预算** | `iteration_budget.py` | 工具调用迭代次数管理 |
| **Mixture-of-Agents** | `moa_loop.py`, `moa_trace.py` | 多模型协作循环 |
| **学习图** | `learning_graph.py`, `learning_mutations.py` | 知识图谱式学习 |
| **图像/视频/语音** | `image_gen_provider.py`, `video_gen_provider.py`, `tts_provider.py`, `transcription_provider.py` | 多媒体生成 ABC + 注册表 |
| **Web 搜索** | `web_search_provider.py`, `web_search_registry.py` | 可插拔搜索后端 |
| **上下文引擎** | `context_engine.py` | 可插拔上下文处理 |
| **安全** | `file_safety.py`, `redact.py`, `ssl_guard.py`, `ssl_verify.py`, `secret_scope.py` | 文件安全、密钥脱敏、SSL 验证 |

#### `tools/` — 工具实现（~90+ Python 文件）

自注册式工具系统。关键工具：

| 类别 | 文件 | 说明 |
|---|---|---|
| **终端** | `terminal_tool.py`, `approval.py` | 终端编排、危险命令检测 |
| **文件操作** | `file_operations.py`, `file_tools.py`, `patch_parser.py`, `fuzzy_match.py` | 读写、搜索、补丁 |
| **Web** | `web_tools.py`, `url_safety.py` | web_search, web_extract |
| **浏览器** | `browser_tool.py`, `browser_cdp_tool.py`, `browser_supervisor.py`, `browser_camofox.py` | CDP 浏览器自动化 |
| **代码执行** | `code_execution_tool.py` | 沙箱 Python + RPC 工具调用 |
| **委派** | `delegate_tool.py`, `async_delegation.py` | 子代理生成、并行任务 |
| **记忆** | `memory_tool.py`, `todo_tool.py` | 代理级工具（被 run_agent.py 拦截） |
| **技能** | `skills_tool.py`, `skills_hub.py`, `skills_guard.py`, `skills_sync.py`, `skill_manager_tool.py` | 技能搜索、加载、管理、安全扫描 |
| **Cron** | `cronjob_tools.py` | 定时任务管理 |
| **Kanban** | `kanban_tools.py` | 多代理协作看板 |
| **MCP** | `mcp_tool.py`, `mcp_oauth.py`, `mcp_oauth_manager.py`, `mcp_stdio_watchdog.py` | MCP 服务器连接 |
| **视觉** | `vision_tools.py` | 多模态图像分析 |
| **TTS/STT** | `tts_tool.py`, `transcription_tools.py`, `voice_mode.py` | 语音合成/识别 |
| **图像/视频生成** | `image_generation_tool.py`, `video_generation_tool.py` | 多后端生成 |
| **终端环境** | `environments/` | local, docker, ssh, modal, daytona, singularity 六种后端 |

#### `gateway/` — 消息网关

```
gateway/
├── run.py          → GatewayRunner — 平台生命周期、消息路由、cron
├── config.py       → 平台配置解析
├── session.py      → 会话存储、上下文提示、重置策略
└── platforms/      → 平台适配器
    ├── telegram, discord, slack, whatsapp, signal, matrix,
    ├── mattermost, email, sms, dingtalk, wecom, weixin, feishu,
    ├── qqbot, bluebubbles, yuanbao, teams, line, irc, ntfy,
    ├── google_chat, homeassistant, simplex, photon, raft,
    ├── webhook, api_server
    └── ADDING_A_PLATFORM.md
```

网关有**双重消息守卫**：base adapter 排队消息 + gateway runner 拦截控制命令。

#### `hermes_cli/` — CLI 子命令和基础设施

包含：主入口 (`main.py`)、配置管理 (`config.py`)、设置向导 (`setup.py`)、认证 (`auth.py`)、模型选择 (`models.py`)、斜杠命令注册表 (`commands.py`)、皮肤引擎 (`skin_engine.py`)、插件加载器 (`plugins.py`)、看板 CLI (`kanban.py`)、策展人 CLI (`curator.py`)、Web 服务器 (`web_server.py`)、PTY 桥接 (`pty_bridge.py`)、Windows 网关 (`gateway_windows.py`) 等。

#### `plugins/` — 插件系统

| 插件类别 | 目录 | 说明 |
|---|---|---|
| **模型提供商** | `model-providers/` (31个) | openrouter, anthropic, gemini, deepseek, nvidia, bedrock, vertex, copilot, fireworks, minimax, xai 等 |
| **记忆后端** | `memory/` (8个) | honcho, mem0, supermemory, byterover, hindsight, holographic, openviking, retaindb |
| **平台适配器** | `platforms/` (20个) | telegram, discord, slack, whatsapp, matrix, signal, teams, line, irc 等 |
| **上下文引擎** | `context_engine/` | 可插拔上下文处理 |
| **图像生成** | `image_gen/` | 多后端图像生成 |
| **视频生成** | `video_gen/` | 多后端视频生成 |
| **Kanban** | `kanban/` | 多代理协作看板 + Web UI |
| **可观测性** | `observability/` | 指标/追踪/日志 |
| **其他** | `disk-cleanup/`, `google_meet/`, `spotify/`, `hermes-achievements/`, `security-guidance/`, `teams_pipeline/`, `browser/`, `web/`, `cron_providers/`, `dashboard_auth/` | 各种扩展功能 |

**重要政策：** 不再接受新的第三方产品插件进入树内。新插件必须以独立仓库发布。

#### `skills/` + `optional-skills/` — 技能系统

```
skills/ (内置，默认激活):
├── apple/             → macOS 专属 (iMessage, Apple Reminders)
├── autonomous-ai-agents/
├── computer-use/
├── creative/
├── data-science/
├── dogfood/
├── email/
├── github/
├── media/
├── mlops/
├── note-taking/
├── productivity/
├── research/
├── smart-home/
├── social-media/
├── software-development/
└── yuanbao/

optional-skills/ (官方可选，需手动安装):
├── autonomous-ai-agents/, blockchain/, communication/, creative/,
├── devops/, email/, health/, mcp/, migration/, mlops/,
├── productivity/, research/, security/, web-development/
```

#### `ui-tui/` + `tui_gateway/` — TUI 架构

```
hermes --tui
  └─ Node (Ink/React)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ 渲染对话、输入、提示、活动流
```

TypeScript 负责屏幕渲染，Python 负责会话、工具、模型调用、斜杠命令逻辑。

#### `apps/` — 桌面应用

- `apps/desktop/` — Electron + React + nanostore 渲染器，通过 JSON-RPC 与 `tui_gateway` 后端通信
- `apps/shared/` — 框架无关的 WebSocket/JSON-RPC 传输层 (`@hermes/shared`)
- `apps/bootstrap-installer/` — 引导安装器

#### `tests/` — 测试套件（~2270 文件）

```
tests/
├── agent/          → 代理内核测试
├── gateway/        → 网关测试
├── tools/          → 工具测试
├── hermes_cli/     → CLI 测试
├── skills/         → 技能测试
├── cron/           → 定时任务测试
├── e2e/            → 端到端测试
├── integration/    → 集成测试（需外部服务）
├── stress/         → 压力测试
├── state/          → 状态管理测试
├── plugins/        → 插件测试
├── providers/     → 提供商测试
├── acp/            → ACP 集成测试
├── dashboard/      → Dashboard 测试
├── docker/         → Docker 测试
├── tui_gateway/    → TUI 网关测试
├── fixtures/       → 测试夹具
├── fakes/          → 模拟对象
└── ...
```

---

## 三、技术栈分析

### Python 依赖

**核心依赖（全部精确锁定 `==X.Y.Z`）：**

| 包 | 版本 | 用途 |
|---|---|---|
| `openai` | 2.24.0 | LLM API 客户端 |
| `httpx[socks]` | 0.28.1 | HTTP 客户端 |
| `rich` | 14.3.3 | CLI 美化 |
| `prompt_toolkit` | 3.0.52 | 交互式输入 |
| `pydantic` | 2.13.4 | 数据验证 |
| `pyyaml` | 6.0.3 | 配置文件 |
| `croniter` | 6.0.0 | Cron 调度 |
| `psutil` | 7.2.2 | 跨平台进程管理 |
| `Pillow` | 12.2.0 | 图像处理 |
| `fastapi` | >=0.104.0,<1 | Web API |
| `uvicorn[standard]` | >=0.24.0,<1 | ASGI 服务器 |
| `websockets` | 15.0.1 | WebSocket |
| `fire` | 0.7.1 | CLI 框架 |
| `tenacity` | 9.1.4 | 重试逻辑 |

**可选依赖（按需安装，通过 `tools/lazy_deps.py` 延迟安装）：**
- 消息平台：`python-telegram-bot`, `discord.py`, `slack-bolt`, `mautrix` (Matrix) 等
- 搜索后端：`exa-py`, `firecrawl-py`, `parallel-web`
- TTS/STT：`edge-tts`, `elevenlabs`, `faster-whisper`, `mistralai`
- 图像生成：`fal-client`
- 云端终端：`modal`, `daytona`
- 云 LLM：`anthropic`, `boto3` (Bedrock), `google-auth` (Vertex)

### JS/TS 依赖

- **TUI：** Ink (React 终端渲染), nanostores
- **桌面：** Electron, `@assistant-ui/react`
- **Web Dashboard：** React, FastAPI (后端), xterm.js
- **浏览器工具：** `agent-browser`
- **引擎要求：** Node.js >=20.0.0

---

## 四、关键设计模式

### 1. 自注册工具系统

每个 `tools/*.py` 在导入时调用 `registry.register()`，无需手动维护导入列表。但工具名称必须添加到 `toolsets.py` 的工具集中才会暴露给代理。

### 2. 足迹阶梯（新能力决策优先级）

```
扩展现有代码 → CLI 命令 + 技能 → 服务门控工具(check_fn) → 插件 → MCP 服务器 → 新核心工具(最后手段)
```

### 3. 多配置文件（Profiles）

每个配置文件有独立的 `HERMES_HOME` 目录（配置、API 密钥、记忆、会话、技能等完全隔离）。所有代码必须使用 `get_hermes_home()` 而非硬编码 `~/.hermes`。

### 4. 三重配置加载器

| 加载器 | 使用场景 | 位置 |
|---|---|---|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | 子命令/设置向导 | `hermes_cli/config.py` |
| 直接 YAML 读取 | 网关运行时 | `gateway/run.py` |

### 5. 插件 ABC + 编排器模式

当 3+ 个 PR 尝试集成同类功能时，设计一个 ABC + 编排器，将已有的内置实现包装为第一个 provider，将竞争 PR 转为该接口的插件。

---

## 五、安全分析

| 防护层 | 实现方式 |
|---|---|
| **Shell 注入** | `shlex.quote()` 转义用户输入 |
| **危险命令检测** | `tools/approval.py` 正则模式 + 用户审批流 |
| **Cron 提示注入** | `tools/cronjob_tools.py` 扫描器阻止指令覆盖 |
| **写入黑名单** | 保护 `~/.ssh/authorized_keys`、`/etc/shadow` 等；`os.path.realpath()` 防符号链接绕过 |
| **技能安全扫描** | `tools/skills_guard.py` 扫描 Hub 安装的技能 |
| **代码执行沙箱** | `execute_code` 子进程剥离 API 密钥 |
| **容器加固** | Docker：丢弃所有 capabilities、禁止提权、PID 限制、tmpfs 大小限制 |
| **依赖供应链** | 核心依赖精确锁定 `==`、范围锁定 `>=floor,<next_major`、Git URL 锁定 SHA、GitHub Actions 锁定 SHA |
| **SSL 验证** | `agent/ssl_guard.py` + `agent/ssl_verify.py` |
| **密钥脱敏** | `agent/redact.py` + `agent/secret_scope.py` |

---

## 六、构建与测试命令

### 安装与运行

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
uv pip install -e ".[all,dev]"
npm install                    # JS/TS 依赖
hermes doctor                  # 诊断
hermes                         # 交互式 CLI
hermes --tui                   # TUI
hermes dashboard               # Web Dashboard
```

### 测试

```bash
scripts/run_tests.sh                              # 全量测试（CI 一致性）
scripts/run_tests.sh tests/gateway/               # 单目录
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # 单测试
```

**永远不要直接调用 `pytest`** — wrapper 强制：unset 凭据变量、TZ=UTC、LANG=C.UTF-8、`-n auto` xdist workers、每文件独立子进程。

### Lint / 类型检查

```bash
ruff check .            # 仅强制 PLW1514（必须指定 encoding=）
ty check                # 类型检查器
npm run --ws check      # JS/TS 全工作区检查
```

---

## 七、开发约定

### 提交规范

Conventional Commits：`<type>(<scope>): <description>`

- 类型：`fix`, `feat`, `docs`, `test`, `refactor`, `chore`
- 范围：`cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `security` 等

### 依赖锁定策略

| 来源 | 策略 |
|---|---|
| PyPI | `==X.Y.Z`（核心）或 `>=floor,<next_major`（范围） |
| Git URL | 完整 commit SHA |
| GitHub Actions | SHA + 版本注释 |
| CI-only pip | `==exact` |

### 测试铁律

- **永远用 `scripts/run_tests.sh`**
- 测试不得写入 `~/.hermes/` — 自动 fixture 重定向到临时目录
- **禁止快照式测试**（change-detector tests）— 断言不变量/行为，而非当前值快照
- **禁止在测试中读源代码文件** — 测试行为，不测试源码文本形状
- JS/TS 相关测试放在 `tests-js/`（vitest），不要放在 `tests/*.py`
- E2E 验证优先于 Mock（涉及解析链、配置传播、安全边界时）

### 跨平台铁律

- **禁止 `os.kill(pid, 0)`** — Windows 上会向整个控制台进程组发送 Ctrl+C。用 `psutil.pid_exists(pid)`
- shell out 前用 `shutil.which()` 检查工具是否存在
- `termios`/`fcntl` 用 `try/except (ImportError, NotImplementedError)` 保护
- 用 `pathlib.Path` 而非字符串拼接 `/`
- 用 `sys.executable` 调用 Python，永远不用 shebang
- 提交前运行 `scripts/check-windows-footguns.py`

### 关键陷阱

| 陷阱 | 说明 |
|---|---|
| 硬编码 `~/.hermes` | 破坏多配置文件。用 `get_hermes_home()` |
| `simple_term_menu` | 新代码禁止使用。用 `hermes_cli/curses_ui.py` |
| `\033[K` (ANSI erase) | prompt_toolkit 下泄漏为字面文本。用空格填充 |
| 工具 schema 跨引用 | 不要在描述中引用其他工具集的工具名 |
| 死代码接入 | 未经验证不要将未使用模块接入活跃路径 |
| 陈旧分支 squash merge | 会静默回退最近的修复 |

---

## 八、项目成熟度评估

### 优势

- 架构设计极为严谨，"窄核心+宽边缘"理念贯穿始终
- 安全意识强：精确依赖锁定、供应链攻击防范、多层防护
- 跨平台支持出色：Linux/macOS/Windows/WSL2/Termux 全覆盖
- 插件系统成熟：ABC + 编排器 + 延迟安装
- 测试基础设施完善：wrapper 保证 CI 一致性、子进程隔离
- 文档质量极高：`AGENTS.md` 是 AI 助手的最佳实践范例

### 挑战

- 核心文件过大（`run_agent.py` ~12k 行、`cli.py` ~11k 行），虽然在积极重构为 mixin/模块
- 依赖管理复杂（核心精确锁定 + 可选延迟安装），但这是安全权衡的合理结果
- 测试套件庞大（~2270 文件），但 wrapper 和并行化已很好处理

### 总体判断

这是一个架构成熟、安全意识强、扩展性设计优秀的项目。"窄核心+宽边缘"理念确保了核心的稳定性，同时通过插件/技能/MCP 保持了极强的扩展能力。
