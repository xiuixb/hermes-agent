# Hermes Agent 项目目录结构与文件说明

> 完整的三分类目录指南：核心代码逻辑（约 16 万行）、非核心代码逻辑（约 29 万行）、配置文件与文档。

---

## 分类一：核心代码逻辑（Core — 约 16 万行 / 35%）

本地运行 Agent 必需的最小模块集合。这些代码定义了"Agent 是什么、它怎么思考、它能做什么"。

### 1.1 agent/ — 核心 Agent 引擎（113 文件，72K 行）

Agent 运行时的心脏。所有决策逻辑、消息管理、工具调度、记忆处理都在这里。

| 文件 | 说描 |
|------|------|
| `__init__.py` | 包初始化 |
| `conversation_loop.py` | **主会话循环**（~4250 行）：单 turn 消息组装 → API 调用 → 工具分发 → 结果追加的主控制流 |
| `system_prompt.py` | **系统提示词组装**：三层拼接（stable / context / volatile），缓存管理 |
| `prompt_builder.py` | **提示词模板库**（84KB）：所有硬编码身份、引导、模型指令常量 |
| `prompt_caching.py` | **Anthropic 缓存控制**：在消息中注入 cache_control 断点以节省 ~75% 输入成本 |
| `context_engine.py` | **上下文文件发现**：扫描 AGENTS.md, .hermes.md, .cursorrules 等 |
| `context_compressor.py` | **上下文压缩器**：超长会话时压缩早期消息 |
| `context_references.py` | 上下文引用管理器 |
| `conversation_compression.py` | 会话压缩策略 |
| `manual_compression_feedback.py` | 手动压缩触发后的反馈处理 |
| `subdirectory_hints.py` | 会话中渐进发现子目录的 AGENTS.md |
| `turn_context.py` | **Turn 准备阶段**：每轮开始的参数重置、系统提示词恢复、preflight 压缩 |
| `turn_finalizer.py` | Turn 结束阶段：后处理、记忆审查触发 |
| `turn_retry_state.py` | Turn 重试状态跟踪 |
| `iteration_budget.py` | **迭代预算**：限制每轮最大 API 调用次数 |
| `agent_init.py` | Agent 实例初始化 |
| `agent_runtime_helpers.py` | 运行时辅助函数 |
| `chat_completion_helpers.py` | **build_api_kwargs()**：根据 api_mode 构建最终 API 请求参数 |
| `message_sanitization.py` | 消息清理：surrogate 字符、非 ASCII、图片剥离 |
| `error_classifier.py` | API 错误分类：识别 billing / rate-limit / context-too-large 等失败类型 |
| `retry_utils.py` | 重试工具（jittered backoff） |
| `tool_executor.py` | 工具调度执行器 |
| `tool_dispatch_helpers.py` | 工具分发辅助函数 |
| `tool_guardrails.py` | **工具安全护栏**：限制文件范围、路径安全、命令审批 |
| `tool_result_classification.py` | 工具结果分类（成功/错误/需要重试） |
| `file_safety.py` | **文件安全**：跨 profile 写入守卫、路径沙箱 |
| `memory_manager.py` | **记忆管理器**：外部记忆提供商（Honcho/RetainDB/Mem0 等）的抽象层 |
| `memory_provider.py` | 记忆提供商接口 |
| `insights.py` | 使用统计与分析 |
| `title_generator.py` | 会话标题自动生成 |
| `plugin_llm.py` | 插件 LLM 访问：插件调用 LLM 的受控通道 |
| `skill_utils.py` | 技能系统工具：YAML frontmatter 解析、条件匹配 |
| `skill_commands.py` | 技能系统命令处理 |
| `skill_bundles.py` | 技能包管理 |
| `skill_preprocessing.py` | 技能预处理 |
| `trajectory.py` | 轨迹存储：记录完整会话用于训练和改进 |
| `background_review.py` | **后台审查**：记忆/技能创建 nudge 的异步触发 |
| `markdown_tables.py` | Markdown 表格渲染 |
| `display.py` | 显示工具类（KawaiiSpinner 等） |
| `i18n.py` | 国际化支持 |
| `redact.py` | 敏感信息脱敏 |
| `rate_limit_tracker.py` | 速率限制追踪器 |
| `nous_rate_guard.py` | Nous Portal 速率限制守卫 |
| `credential_pool.py` | 凭证池（多 API key 轮换） |
| `credential_sources.py` | 凭证来源管理 |
| `credential_persistence.py` | 凭证持久化 |
| `credits_tracker.py` | 积分（credits）追踪 |
| `account_usage.py` | 账户用量统计 |
| `portal_tags.py` | Nous Portal 标签 |
| `curator.py` | Curator 模式：知识策展 |
| `curator_backup.py` | Curator 备份 |
| `browser_provider.py` | 浏览器提供商抽象 |
| `browser_registry.py` | 浏览器注册中心 |
| `image_gen_provider.py` | 图像生成提供商抽象 |
| `image_gen_registry.py` | 图像生成注册中心 |
| `image_routing.py` | 图像路由（多模态内容分发） |
| `video_gen_provider.py` | 视频生成提供商抽象 |
| `video_gen_registry.py` | 视频生成注册中心 |
| `web_search_provider.py` | Web 搜索提供商抽象 |
| `web_search_registry.py` | Web 搜索注册中心 |
| `transcription_provider.py` | 转录提供商抽象 |
| `transcription_registry.py` | 转录注册中心 |
| `tts_provider.py` | TTS 提供商抽象 |
| `tts_registry.py` | TTS 注册中心 |
| `model_metadata.py` | **模型元数据**：context_length 解析、token 估算 |
| `usage_pricing.py` | 使用量定价计算 |
| `lmstudio_reasoning.py` | LM Studio 推理配置 |
| `model_tools.py` | 模型相关工具（项目根级的 model_tools.py 引用的共享逻辑） |
| `models_dev.py` | 模型开发工具 |
| `async_utils.py` | 异步工具 |
| `auxiliary_client.py` | 辅助 LLM 客户端（子任务调用） |
| `copilot_acp_client.py` | GitHub Copilot ACP 客户端 |
| `codex_runtime.py` | Codex 运行时 |
| `codex_responses_adapter.py` | Codex Responses API 适配器 |
| `moonshot_schema.py` | Moonshot/Kimi schema 转换 |
| `gemini_schema.py` | Gemini schema 转换 |
| `gemini_native_adapter.py` | Gemini 原生 API 适配器 |
| `gemini_cloudcode_adapter.py` | Gemini Cloud Code 适配器 |
| `google_code_assist.py` | Google Code Assist 集成 |
| `google_oauth.py` | Google OAuth 身份认证 |
| `anthropic_adapter.py` | Anthropic Messages API 适配器 |
| `bedrock_adapter.py` | AWS Bedrock API 适配器 |
| `azure_identity_adapter.py` | Azure Identity 适配器 |
| `jiter_preload.py` | jiter JSON 解析器预加载 |
| `onboarding.py` | 新用户引导流程 |
| `process_bootstrap.py` | 进程启动引导（安全 stdio 等） |
| `runtime_cwd.py` | 运行时工作目录解析 |
| `shell_hooks.py` | Shell 钩子 |
| `think_scrubber.py` | 思考标记（\<think\>）清洗 |
| `stream_diag.py` | 流式传输诊断 |
| `model_cost_guard.py` | 模型成本守卫 |

**agent/ 子目录：**

| 子目录 | 说明 |
|--------|------|
| `agent/lsp/` (11 文件) | **LSP 集成**：Language Server Protocol 客户端、管理器、事件日志、工作区同步 |
| `agent/transports/` (11 文件) | **API 传输层**：chat_completions、anthropic、bedrock、codex、codex_app_server 等 |
| `agent/secret_sources/` (2 文件) | 密钥来源：Bitwarden 集成 |

---

### 1.2 tools/ — 工具系统（101 文件，76K 行）

Agent 的手和脚——每个工具都是一个独立模块，定义 Agent 能执行的操作。

| 文件 | 说明 |
|------|------|
| `__init__.py` | 包初始化 |
| `registry.py` | **工具注册中心**：声明式注册所有工具 |
| `terminal_tool.py` | **终端执行**：在本地/Docker/SSH/Modal/Daytona 后端执行命令 |
| `read_terminal_tool.py` | 读取终端输出 |
| `file_tools.py` | **文件操作**：read_file、write_file、edit_file、search_files |
| `file_operations.py` | 文件操作原子化（atomic replace） |
| `file_state.py` | 文件状态追踪 |
| `code_execution_tool.py` | **代码执行**：Python/JS/Shell 沙箱执行 |
| `memory_tool.py` | **记忆写入/读取**：持久化 MEMORY.md |
| `session_search_tool.py` | **会话搜索**：FTS5 全文搜索 + LLM 摘要跨会话回溯 |
| `skill_manager_tool.py` | **技能管理**：创建、修补、删除技能 |
| `skills_tool.py` | 技能列表/查看/加载 |
| `skills_guard.py` | 技能安全守卫 |
| `skills_sync.py` | 技能同步 |
| `skills_hub.py` | **技能市场**：agentskills.io 发布/安装 |
| `skills_ast_audit.py` | 技能 AST 审计 |
| `skill_provenance.py` | 技能来源追踪 |
| `skill_usage.py` | 技能使用统计 |
| `delegate_tool.py` | **子代理委托**：派生隔离的子代理执行并行任务 |
| `kanban_tools.py` | **看板工具**：kanban_show/create/block/complete/heartbeat |
| `browser_tool.py` | **浏览器控制**：Playwright 驱动的 Web 自动化 |
| `browser_cdp_tool.py` | Chrome DevTools Protocol 浏览器控制 |
| `browser_camofox.py` | CamoFox 浏览器（隐身模式） |
| `browser_camofox_state.py` | CamoFox 状态管理 |
| `browser_dialog_tool.py` | 浏览器对话框处理 |
| `browser_supervisor.py` | 浏览器监督器 |
| `computer_use_tool.py` | **桌面控制**（macOS）：驱动 AT-SPI 的 GUI 自动化 |
| `web_tools.py` | **Web 搜索/抓取**：Firecrawl / Exa / Parallel Web |
| `x_search_tool.py` | X.com 搜索 |
| `image_generation_tool.py` | **图像生成**：FAL / OpenAI / xAI 图像生成 |
| `video_generation_tool.py` | **视频生成** |
| `tts_tool.py` | **文本转语音**：Edge TTS / ElevenLabs / OpenAI / MiniMax |
| `transcription_tools.py` | **语音转文本**：Whisper 转录 |
| `vision_tools.py` | **视觉分析**：图片理解、截图分析 |
| `send_message_tool.py` | **消息发送**：通过网关向第三方发消息 |
| `todo_tool.py` | **TODO 管理**：跟踪任务状态 |
| `cronjob_tools.py` | **定时任务管理**：创建/编辑/删除 cron 作业 |
| `clarify_tool.py` | 主动向用户提问（CLI + 网关双向） |
| `clarify_gateway.py` | 网关上的 clarify 通道 |
| `interrupt.py` | 中断处理 |
| `mcp_tool.py` | **MCP 集成工具**：外部 MCP 服务器连接 |
| `mcp_oauth.py` | MCP OAuth 认证 |
| `mcp_oauth_manager.py` | MCP OAuth 管理器 |
| `mixture_of_agents_tool.py` | MoA（众模型混合）工具 |
| `discord_tool.py` | Discord 消息发送 |
| `homeassistant_tool.py` | Home Assistant 集成 |
| `feishu_doc_tool.py` | 飞书文档操作 |
| `feishu_drive_tool.py` | 飞书云盘操作 |
| `tool_search.py` | 工具搜索（动态查找可用工具） |
| `voice_mode.py` | 语音模式 |
| `approval.py` | **操作审批框架**：危险命令、文件写入等需确认 |
| `write_approval.py` | 文件写入审批 |
| `slash_confirm.py` | Slash 命令确认 |
| `path_security.py` | **路径安全**：防路径穿越、沙箱验证 |
| `url_safety.py` | URL 安全校验 |
| `website_policy.py` | 网站访问策略 |
| `tirith_security.py` | Tirith 安全策略引擎 |
| `osv_check.py` | OSV 漏洞检查 |
| `threat_patterns.py` | **威胁模式检测**：提示注入、越狱攻击模式库 |
| `tool_output_limits.py` | 工具输出截断（防 LLM 上下文溢出） |
| `tool_result_storage.py` | 工具结果存储 |
| `binary_extensions.py` | 二进制文件扩展名过滤 |
| `ansi_strip.py` | ANSI 转义码剥离 |
| `lazy_deps.py` | **懒加载依赖**：按需安装可选后端（Anthropic/Firecrawl 等） |
| `budget_config.py` | 预算配置 |
| `checkpoint_manager.py` | **检查点管理**：会话快照与回滚 |
| `credential_files.py` | 凭证文件管理 |
| `env_probe.py` | **Python 环境探测**：python/pip/uv/PEP-668 状态检测 |
| `env_passthrough.py` | 环境变量穿透（传给子进程） |
| `fal_common.py` | FAL 图像生成共享工具 |
| `neutts_synth.py` | NeuroTTS 合成 |
| `managed_tool_gateway.py` | 托管的工具网关（Nous Portal 资源路由） |
| `openrouter_client.py` | OpenRouter API 客户端 |
| `patch_parser.py` | Patch/diff 解析 |
| `process_registry.py` | 进程注册表（PID 追踪） |
| `schema_sanitizer.py` | **Schema 清洗器**：为 xAI/Gemini 等严格 API 清洗 tool schema |
| `thread_context.py` | 线程上下文管理 |
| `tool_backend_helpers.py` | 终端后端辅助函数 |
| `fuzzy_match.py` | 模糊匹配 |
| `microsoft_graph_auth.py` | Microsoft Graph 认证 |
| `microsoft_graph_client.py` | Microsoft Graph 客户端 |
| `xai_http.py` | xAI HTTP 客户端 |
| `yuanbao_tools.py` | 元宝（腾讯）工具集成 |

**tools/ 子目录：**

| 子目录 | 说明 |
|--------|------|
| `tools/computer_use/` (6 文件) | **桌面控制后端**：cua-driver 集成、schema、视觉路由 |
| `tools/environments/` (11 文件) | **终端后端**：local, docker, ssh, modal, singularity, daytona 六种后端实现 |
| `tools/neutts_samples/` | TTS 音频样本 |

---

### 1.3 skills/ — 内置技能库（34 文件，10K 行）

Agent 的知识模块。技能通过 Markdown + YAML frontmatter 定义，按分类组织。核心代码路径具备扫描和加载技能的机制（`agent/prompt_builder.py`）。

| 目录 | 说明 |
|------|------|
| `skills/apple/` | Apple 生态：Swift/Xcode/Shortcuts |
| `skills/autonomous-ai-agents/` | 自主 AI Agent 技能 |
| `skills/creative/` | 创意：信息图、ASCII 视频、P5.js、Manim 动画 |
| `skills/data-science/` | 数据科学 |
| `skills/devops/` | DevOps：CI/CD、K8s、Kanban |
| `skills/dogfood/` | 内部 Dogfood 测试 |
| `skills/email/` | 邮件处理 |
| `skills/github/` | GitHub：PR review、Issues |
| `skills/index-cache/` | 技能索引缓存 |
| `skills/media/` | 媒体：YouTube、Baoyu 信息图 |
| `skills/mlops/` | MLOps：模型训练/部署 |
| `skills/note-taking/` | 笔记：Obsidian |
| `skills/productivity/` | 生产力工具 |
| `skills/research/` | 学术研究：论文写作、LaTeX |
| `skills/smart-home/` | 智能家居 |
| `skills/social-media/` | 社交媒体 |
| `skills/software-development/` | **软件开发**：代码审查、TDD、文档、Refactoring 等 |
| `skills/yuanbao/` | 元宝集成 |

---

### 1.4 providers/ — 提供商抽象（2 文件，0.4K 行）

LLM 提供商的最简抽象接口。

| 文件 | 说明 |
|------|------|
| `__init__.py` | 包初始化 |
| `base.py` | 提供商基类 |
| `README.md` | 提供商开发指南 |

---

### 1.5 顶级核心脚本

这些文件与 `agent/` 紧密耦合，是 Agent 运行时的直接组成部分。

| 文件 | 行数 | 说明 |
|------|------|------|
| `run_agent.py` | 5,361 | **AIAgent 主类**：所有权转发方法、会话创建、编译/运行入口 |
| `hermes_state.py` | 4,600 | **状态管理**：会话数据库（SQLite）、配置持久化、会话恢复 |
| `hermes_constants.py` | - | 全局常量：路径、默认值、版本 |
| `hermes_logging.py` | - | 日志系统 |
| `hermes_time.py` | - | 时间工具（天级精度） |
| `mcp_serve.py` | 897 | **MCP 服务入口**：将 Hermes Agent 暴露为 MCP 服务器 |
| `utils.py` | 414 | 通用工具函数 |
| `model_tools.py` | - | 模型管理工具（切换/探测/列表） |
| `toolsets.py` | - | 工具集定义 |
| `toolset_distributions.py` | - | 工具集分发配置 |
| `hermes_bootstrap.py` | - | 启动引导 |

---

### 1.6 tests/ — 测试套件

核心代码测试放在 `tests/agent/`、`tests/tools/`、`tests/run_agent/` 中，其他测试目录对应各自的非核心模块。

| 目录 | 说明 |
|------|------|
| `tests/agent/` | Agent 核心测试：system_prompt, prompt_builder, turn_context 等 |
| `tests/tools/` | 工具测试 |
| `tests/run_agent/` | run_agent 主类测试 |
| `tests/hermes_state/` | 状态管理测试 |
| `tests/providers/` | 提供商适配测试 |
| `tests/skills/` | 技能系统测试 |
| `tests/fakes/` | 假对象/Mock 工具 |
| `tests/fixtures/` | 测试固定数据 |
| `tests/conftest.py` | Pytest 全局配置 |

---

## 分类二：非核心代码逻辑（Non-Core — 约 29 万行 / 65%）

这些是分发层、用户界面、扩展插件和平台适配器。本地开发/运行 Agent 时可能间接使用（如 CLI），但核心引擎不依赖它们。

### 2.1 hermes_cli/ — CLI 与安装工具（170 文件，130K 行）

最大的非核心模块。处理所有面向用户的操作——安装、配置、模型管理、TUI 交互。

| 文件 | 说明 |
|------|------|
| `main.py` | **CLI 入口**：`hermes` 命令分发 |
| `_parser.py` | 命令行参数解析 |
| `commands.py` | 所有 CLI 命令定义和路由 |
| `setup.py` | **安装向导**：交互式配置 API key、模型、工具 |
| `models.py` | 模型目录（200+ 模型列表） |
| `model_switch.py` | `/model` 切换逻辑 |
| `model_setup_flows.py` | 模型设置流程 |
| `model_normalize.py` | 模型名规范化 |
| `model_catalog.py` | **模型目录构建**：模型信息聚合 |
| `model_cost_guard.py` | 模型成本守卫 |
| `config.py` | **配置管理**：读取/写入 config.yaml |
| `providers.py` | 提供商选择与管理 |
| `tools_config.py` | 工具配置 UI |
| `skills_config.py` | 技能配置 |
| `skills_hub.py` | Skills Hub CLI 界面 |
| `plugins.py` | 插件发现与生命周期 |
| `plugins_cmd.py` | 插件 CLI 命令 |
| `auth.py` | 身份验证（OAuth 通用） |
| `auth_commands.py` | 认证命令 |
| `nous_account.py` | Nous Portal 账户管理 |
| `nous_subscription.py` | Nous Portal 订阅管理 |
| `portal_cli.py` | Nous Portal CLI |
| `secrets_cli.py` | 密钥管理 CLI |
| `secret_prompt.py` | 密钥输入提示 |
| `curator.py` | Curator 模式 CLI |
| `doctor.py` | **诊断工具**：`hermes doctor` |
| `gateway.py` | 网关管理 CLI |
| `gateway_windows.py` | Windows 网关管理 |
| `kanban.py` | 看板 CLI |
| `kanban_db.py` | 看板数据库 |
| `kanban_decompose.py` | 看板任务分解 |
| `kanban_specify.py` | 看板任务规格化 |
| `kanban_swarm.py` | 看板 Swarm 模式 |
| `kanban_diagnostics.py` | 看板诊断 |
| `cron.py` | Cron 作业 CLI |
| `logs.py` | 日志查看 |
| `oneshot.py` | 单次执行模式 |
| `partial_compress.py` | 部分压缩 CLI |
| `session_recap.py` | 会话回顾 |
| `send_cmd.py` | 发送消息命令 |
| `prompt_size.py` | **Prompt 大小估算**：`hermes prompt-size` |
| `uninstall.py` | 卸载器 |
| `inventory.py` | 版本清单 |
| `banner.py` | Banner 显示 |
| `callbacks.py` | CLI 回调（stream delta, status 等） |
| `cli_agent_setup_mixin.py` | Agent 设置 Mixin |
| `cli_commands_mixin.py` | CLI 命令 Mixin |
| `cli_output.py` | CLI 输出格式化 |
| `clipboard.py` | 剪贴板操作 |
| `colors.py` | 颜色主题 |
| `completion.py` | Shell 自动补全 |
| `curses_ui.py` | Curses 终端 UI |
| `debug.py` | 调试工具 |
| `dump.py` | 转储工具（调试用） |
| `env_loader.py` | **环境变量加载**：.env 文件与系统环境合并 |
| `fallback_cmd.py` | Fallback 命令 |
| `fallback_config.py` | Fallback 配置 |
| `goals.py` | 目标追踪 |
| `gui_uninstall.py` | GUI 卸载 |
| `hooks.py` | CLI 钩子系统 |
| `middleware.py` | **中间件系统**：LLM 请求/响应拦截 |
| `migrate.py` | 数据迁移 |
| `mcp_catalog.py` | MCP 目录 |
| `mcp_config.py` | MCP 配置 |
| `mcp_picker.py` | MCP 选择器 |
| `mcp_startup.py` | MCP 启动管理 |
| `memory_setup.py` | 记忆设置 |
| `pairing.py` | 设备配对 |
| `platforms.py` | 平台 CLI |
| `profile_describer.py` | Profile 描述生成 |
| `profile_distribution.py` | Profile 分发 |
| `profiles.py` | Profile 管理 |
| `psutil_android.py` | Android psutil 兼容 |
| `pt_input_extras.py` | Prompt Toolkit 扩展 |
| `pty_bridge.py` | PTY 桥接 (Unix) |
| `win_pty_bridge.py` | PTY 桥接 (Windows) |
| `relaunch.py` | 重启动 |
| `runtime_provider.py` | 运行时提供商 |
| `service_manager.py` | 服务管理器 |
| `skin_engine.py` | 皮肤引擎 |
| `slack_cli.py` | Slack CLI |
| `status.py` | 状态管理 |
| `stdio.py` | 标准 IO 管理 |
| `telegram_managed_bot.py` | Telegram 机器人管理 |
| `timeouts.py` | 超时配置 |
| `tips.py` | 使用提示 |
| `voice.py` | 语音设置 |
| `web_server.py` | Web Dashboard 服务器 |
| `webhook.py` | Webhook 管理 |
| `xai_retirement.py` | xAI 旧版迁移 |
| `active_sessions.py` | 活跃会话查看 |
| `azure_detect.py` | Azure 环境检测 |
| `backup.py` | 备份管理 |
| `browser_connect.py` | 浏览器连接 |
| `build_info.py` | 构建信息 |
| `bundles.py` | Bundle 管理 |
| `checkpoints.py` | **检查点 CLI**：保存/恢复/列出 |
| `claw.py` | OpenClaw 迁移 |
| `codex_models.py` | Codex 模型支持 |
| `codex_runtime_plugin_migration.py` | Codex 运行时插件迁移 |
| `codex_runtime_switch.py` | Codex 运行时切换 |
| `container_boot.py` | 容器启动 |
| `copilot_auth.py` | GitHub Copilot 认证 |
| `dashboard_auth/` | Dashboard 认证模块 |
| `dashboard_register.py` | Dashboard 注册 |
| `dashb` | 密钥管理 CLI |
| `dep_ensure.py` | 依赖确保 |
| `dingtalk_auth.py` | 钉钉认证 |
| `managed_uv.py` | 托管 uv（包管理器自升级） |
| `security_advisories.py` | 安全公告 |
| `security_audit.py` | 安全审计 |
| `subcommands/` | 子命令模块 |
| `proxy/` | 代理设置 |

---

### 2.2 gateway/ — 消息网关（65 文件，86K 行）

将 Agent 暴露到 20+ 消息平台的网关层。核心 Agent 完全不依赖此模块。

| 文件 | 说明 |
|------|------|
| `run.py` | **网关主循环**：多线程调度 |
| `session.py` | 会话管理（消息分发到多 platform） |
| `session_context.py` | 会话上下文 |
| `delivery.py` | 消息投递引擎 |
| `dispatch.py` (via stream_dispatch) | 流式消息分发 |
| `hocks.py` | 网关钩子系统 |
| `slash_commands.py` | Slash 命令处理 |
| `slash_access.py` | Slash 命令权限 |
| `config.py` | 网关配置 |
| `platform_registry.py` | 平台注册中心 |
| `channel_directory.py` | Channel 目录 |
| `mirror.py` | 会话镜像（多平台同步） |
| `pairing.py` | 设备配对 |
| `memory_monitor.py` | 记忆监控 |
| `kanban_watchers.py` | 看板观察者 |
| `authz_mixin.py` | 授权 Mixin |
| `display_config.py` | 显示配置 |
| `restart.py` | 网关重启动 |
| `runtime_footer.py` | 运行时 Footer |
| `shutdown_forensics.py` | 关闭诊断 |
| `status.py` | 状态报告 |
| `sticker_cache.py` | 表情包缓存 |
| `whatsapp_identity.py` | WhatsApp 身份管理 |
| `stream_consumer.py` | 流式消费者 |
| `stream_dispatch.py` | 流式分发 |
| `stream_events.py` | 流式事件 |
| `builtin_hooks/` | 内置钩子 |

**gateway/platforms/ — 平台适配器（27 个平台）：**

| 适配器 | 说明 |
|--------|------|
| `telegram.py` | Telegram 机器人（消息、命令、Markdown、音频） |
| `discord.py` （via） | Discord 机器人 |
| `slack.py` | Slack Workspace 集成 |
| `whatsapp.py` | WhatsApp 桥接（多媒体支持） |
| `signal.py` | Signal 消息 |
| `email.py` | Email 通道 |
| `sms.py` | SMS 通道 |
| `matrix.py` | Matrix 协议 |
| `wechat.py` | 微信（公众号） |
| `weixin.py` | 微信 |
| `wecom.py` | 企业微信 |
| `wecom_callback.py` | 企业微信回调模式 |
| `wecom_crypto.py` | 企业微信加密 |
| `dingtalk.py` | 钉钉 |
| `feishu.py` | 飞书 |
| `feishu_comment.py` | 飞书评论 |
| `feishu_comment_rules.py` | 飞书评论规则 |
| `feishu_meeting_invite.py` | 飞书会议邀请 |
| `bluebubbles.py` | BlueBubbles (iMessage) |
| `webhook.py` | 通用 Webhook |
| `api_server.py` | REST API 服务 |
| `yuanbao.py` | 元宝（腾讯）平台 |
| `yuanbao_media.py` | 元宝媒体处理 |
| `yuanbao_proto.py` | 元宝 Proto 协议 |
| `yuanbao_sticker.py` | 元宝表情 |
| `qqbot/` | QQ 机器人 |
| `msgraph_webhook.py` | Microsoft Graph Webhook |

---

### 2.3 plugins/ — 插件系统（137 文件，58K 行）

可选的功能扩展。每个插件通过 `plugin.yaml` 声明式注册。

| 插件目录 | 说明 |
|----------|------|
| `plugins/memory/honcho/` | **Honcho** — 辩证式用户建模（用户个性的慢变化模型） |
| `plugins/memory/hindsight/` | **Hindsight** — 时序记忆后端 |
| `plugins/memory/holographic/` | **Holographic** — 全息记忆存储 |
| `plugins/memory/mem0/` | **Mem0** — 云端记忆服务 |
| `plugins/memory/openviking/` | **OpenViking** — 开源记忆后端 |
| `plugins/memory/retaindb/` | **RetainDB** — 数据库记忆后端 |
| `plugins/memory/bytorover/` | **ByteRover** — 记忆后端 |
| `plugins/memory/supermemory/` | **SuperMemory** — 记忆后端 |
| `plugins/observability/langfuse/` | **Langfuse** — LLM 观测与追踪 |
| `plugins/observability/nemo_relay/` | **Nemo Relay** — 观测转发 |
| `plugins/browser/browserbase/` | **Browserbase** — 云端浏览器 |
| `plugins/browser/browser_use/` | **Browser Use** — 浏览器自动化 |
| `plugins/browser/firecrawl/` | **Firecrawl** — Web 爬虫 |
| `plugins/image_gen/fal/` | **FAL** — 图像生成 |
| `plugins/image_gen/krea/` | **KREA** — 图像生成 |
| `plugins/image_gen/openai/` | **OpenAI** — 图像生成 |
| `plugins/image_gen/openai-codex/` | **OpenAI Codex** — 图像生成 |
| `plugins/image_gen/xai/` | **xAI** — 图像生成 |
| `plugins/video_gen/` | 视频生成插件 |
| `plugins/security-guidance/` | **安全指导** — 代码安全审计 |
| `plugins/hermes-achievements/` | **成就系统** — 游戏化元素 |
| `plugins/spotify/` | **Spotify** — 音乐控制 |
| `plugins/google_meet/` | **Google Meet** — 会议集成 |
| `plugins/disk-cleanup/` | **磁盘清理** — 自动空间回收 |
| `plugins/context_engine/` | **上下文引擎** — 上下文增强 |
| `plugins/kanban/` | **看板插件** — 看板扩展 |
| `plugins/dashboard_auth/` | Dashboard 认证插件 |
| `plugins/teams_pipeline/` | Teams 管道 |
| `plugins/platforms/` | 平台扩展 |
| `plugins/model-providers/` | 模型提供商扩展 |

---

### 2.4 cron/ — 定时任务（3 文件，3.6K 行）

| 文件 | 说明 |
|------|------|
| `__init__.py` | 包初始化 |
| `scheduler.py` | **Cron 调度器**：解析 cron 表达式、管理任务队列 |
| `jobs.py` | 作业定义与执行 |

---

### 2.5 acp_adapter/ — ACP 协议适配（11 文件，5K 行）

Agent Client Protocol 实现：允许外部客户端（VSCode 扩展等）以标准协议与 Agent 交互。

| 文件 | 说明 |
|------|------|
| `entry.py` | ACP 入口（`hermes-acp` 命令） |
| `server.py` | ACP 服务器 |
| `session.py` | ACP 会话管理 |
| `tools.py` | 工具桥接 |
| `auth.py` | 认证 |
| `events.py` | 事件系统 |
| `permissions.py` | 权限管理 |
| `provenance.py` | 来源追踪 |
| `edit_approval.py` | 编辑审批 |

---

### 2.6 tui_gateway/ — TUI 网关（8 文件，11K 行）

终端 UI（Textual 框架）与 Agent 核心之间的桥接层。

| 文件 | 说明 |
|------|------|
| `entry.py` | TUI 入口 |
| `server.py` | WebSocket 服务器 |
| `ws.py` | WebSocket 协议 |
| `event_publisher.py` | 事件发布 |
| `transport.py` | 传输层 |
| `render.py` | 渲染 |
| `slash_worker.py` | Slash 命令 Worker |

---

### 2.7 web/ — Web Dashboard

基于 Vite + React + TypeScript 的 SPA。

| 文件 | 说明 |
|------|------|
| `src/` | React SPA 源码 |
| `package.json` | NPM 依赖 |
| `vite.config.ts` | Vite 构建配置 |
| `index.html` | 入口 HTML |

---

### 2.8 apps/ — 桌面应用

基于 Tauri 的跨平台桌面应用。

---

### 2.9 optional-skills/ — 可选技能

额外技能包，不随默认安装提供。

| 目录 | 说明 |
|------|------|
| `autonomous-ai-agents/` | 自主 AI Agent 扩展 |
| `blockchain/` | 区块链开发 |
| `communication/` | 通信工具 |
| `creative/` | 创意（更多生成/艺术） |
| `devops/` | DevOps 扩展 |
| `dogfood/` | 内部测试 |
| `email/` | Email 扩展 |
| `finance/` | 金融分析 |
| `gaming/` | 游戏开发 |
| `health/` | 健康应用 |
| `mcp/` | MCP 扩展 |
| `migration/` | 迁移工具 |
| `mlops/` | MLOps 扩展 |
| `productivity/` | 生产力扩展 |
| `research/` | 科研扩展 |
| `security/` | 安全审计扩展 |
| `software-development/` | 软件开发扩展 |
| `web-development/` | Web 开发扩展 |

---

### 2.10 optional-mcps/ — 可选 MCP 服务

预置的 MCP 服务清单，用户可通过 `hermes mcp` 命令安装。

| 目录 | 说明 |
|------|------|
| `linear/` | Linear 项目管理 MCP |
| `n8n/` | n8n 自动化 MCP |

---

### 2.11 scripts/ — 构建与工具脚本

| 文件 | 说明 |
|------|------|
| `install.sh` / `install.ps1` / `install.cmd` | 三平台安装脚本 |
| `run_tests.sh` / `run_tests_parallel.py` | 测试运行器 |
| `release.py` | 发布脚本 |
| `build_model_catalog.py` | 模型目录构建 |
| `build_skills_index.py` | 技能索引构建 |
| `lint_diff.py` | Lint 差异检查 |
| `sample_and_compress.py` | 采样与压缩 |
| `contributor_audit.py` | 贡献者审计 |
| `check-windows-footguns.py` | Windows 问题检查 |
| `analyze_livetest.py` | 实时测试分析 |
| `discord-voice-doctor.py` | Discord 语音诊断 |
| `tool_search_livetest.py` | 工具搜索实时测试 |
| `hermes-gateway` / `whatsapp-bridge` | 守护进程脚本 |
| `setup-hermes.sh` | 开发环境快速搭建 |

---

### 2.12 docker/ — Docker 部署

| 文件 | 说明 |
|------|------|
| `entrypoint.sh` | Docker 入口点 |
| `main-wrapper.sh` | 主进程包装 |
| `hermes-exec-shim.sh` | Hermes 执行 shim |
| `stage2-hook.sh` | Stage2 钩子 |
| `cont-init.d/` | 容器初始化脚本 |
| `s6-rc.d/` | s6 进程监督 |

---

### 2.13 nix/ — Nix 包管理

| 文件 | 说明 |
|------|------|
| `hermes-agent.nix` | Nix 包定义 |
| `desktop.nix` | 桌面环境 |
| `devShell.nix` | 开发环境 |
| `tui.nix` | TUI 环境 |
| `web.nix` | Web 环境 |
| `python.nix` | Python 依赖 |

---

### 2.14 packaging/ — 打包分发

| 目录 | 说明 |
|------|------|
| `packaging/homebrew/` | Homebrew formula |

---

### 2.15 cli.py — CLI 入口巨型文件（639KB / ~13.7K 行）

项目最大的单文件。虽然它是 `hermes` 命令的直接入口，包含大量 TUI 交互逻辑、彩色渲染、会话持久化显示等，但它和 `agent/` 之间通过 `hermes_cli/main.py` 桥接。核心 Agent 的 `run_conversation()` 不依赖此文件。

---

### 2.16 其他顶级辅助脚本

| 文件 | 说明 |
|------|------|
| `batch_runner.py` | **批量轨迹生成**：用于训练数据制备 |
| `trajectory_compressor.py` | **轨迹压缩**：将会话轨迹压缩为更紧凑的训练格式 |
| `mini_swe_runner.py` | **Mini SWE Agent**：轻量软件工程师代理 |
| `datagen-config-examples/` | 数据生成配置示例 |

---

## 分类三：配置文件与文档

### 3.1 项目配置

| 文件 | 说明 |
|------|------|
| `pyproject.toml` | **Python 项目配置**：依赖、构建系统、工具链设置、Python 版本约束（3.11-3.14） |
| `uv.lock` | **锁定依赖**（622K）：精确锁定的所有依赖版本，防止供应链攻击 |
| `setup.py` | 传统 setuptools 入口（指向 pyproject.toml） |
| `MANIFEST.in` | sdist 打包清单 |
| `package.json` | NPM 依赖（web/apps 的 TypeScript 构建工具） |
| `package-lock.json` | NPM 锁文件 |
| `flake.nix` | Nix Flake 定义 |
| `flake.lock` | Nix Flake 锁文件 |
| `constraints-termux.txt` | Termux（Android）平台依赖约束 |
| `.hadolint.yaml` | Dockerfile lint 配置 |

### 3.2 环境与运行时配置

| 文件 | 说明 |
|------|------|
| `.env.example` | **环境变量模板**（23.7KB）：所有可配置的环境变量及详细注释 |
| `.envrc` | direnv 配置文件 |
| `cli-config.yaml.example` | **CLI 配置模板**（64KB）：config.yaml 完整示例 |
| `docker-compose.yml` | Docker Compose 编排 |
| `docker-compose.windows.yml` | Windows Docker Compose 编排 |
| `Dockerfile` | Docker 镜像构建 |

### 3.3 Git 与 CI

| 文件/目录 | 说明 |
|-----------|------|
| `.gitignore` | Git 忽略规则 |
| `.gitattributes` | Git 属性 |
| `.mailmap` | 提交者名映射 |
| `.github/workflows/` | **CI/CD 工作流**（GitHub Actions） |
| `.github/actions/` | 自定义复合 Actions |
| `.github/ISSUE_TEMPLATE/` | Issue 模板 |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR 模板 |
| `.github/dependabot.yml` | Dependabot 配置 |

### 3.4 文档

| 文件 | 说明 |
|------|------|
| `README.md` | 英文主 README |
| `README.zh-CN.md` | 中文 README |
| `README.ur-pk.md` | 乌尔都语 README |
| `AGENTS.md` | **开发者指南**（71KB）：给 AI 辅助开发和贡献者的完整指导 |
| `CONTRIBUTING.md` | 贡献指南 |
| `SECURITY.md` | 安全策略 |
| `LICENSE` | MIT 许可证 |
| `hermes-already-has-routines.md` | 内部设计备忘录 |
| `docs/` | 项目文档目录 |
| `docs/system-prompt-design.md` | 系统提示词设计文档 |
| `docs/session-message-assembly.md` | 会话消息拼接文档 |
| `docs/middleware/` | 中间件文档 |
| `docs/observability/` | 可观测性文档 |
| `docs/security/` | 安全文档 |
| `docs/kanban/` | 看板文档 |
| `docs/hermes-kanban-v1-spec.pdf` | 看板 v1 规范 |

### 3.5 国际化

| 目录 | 说明 |
|------|------|
| `locales/` (17 文件) | **多语言翻译**：en, zh, zh-hant, ja, ko, de, fr, es, pt, it, ru, uk, tr, hu, ga, af 等 |

### 3.6 设计文档与规划

| 元数据 | 说明 |
|--------|------|
| `.plans/` | 功能设计规划文档 |
| `plans/` | 规划设计文档 |
| `infographic/` | 信息图资源 |

### 3.7 静态资源

| 目录 | 说明 |
|------|------|
| `assets/` | 品牌资源（banner.png 等） |
| `docker/SOUL.md` | Docker 容器的默认 SOUL.md |
| `acp_registry/` | ACP 注册信息（agent.json, icon.svg） |

---

## 总结

```
hermes-agent/
├── [核心 — 35%] agent/ tools/ skills/ providers/
│    run_agent.py hermes_state.py hermes_constants.py
│    model_tools.py toolsets.py mcp_serve.py utils.py
│    hermes_logging.py hermes_time.py hermes_bootstrap.py
│
├── [非核心 — 65%] hermes_cli/ gateway/ plugins/ cron/
│    acp_adapter/ tui_gateway/ web/ apps/ optional-skills/
│    optional-mcps/ scripts/ docker/ nix/ packaging/
│    cli.py batch_runner.py trajectory_compressor.py
│    mini_swe_runner.py toolset_distributions.py
│
└── [配置] pyproject.toml uv.lock Dockerfile
     .env.example cli-config.yaml.example
     README* AGENTS.md CONTRIBUTING.md SECURITY.md
     docs/ locales/ .github/
```

**核心=引擎，非核心=车身。** `agent/` 是那个最窄的腰，包含了所有的思考、记忆、规划和执行。换一个 CLI、换一个网关、换一个 TUI，Agent 的核心行为完全不变。
