# Suna 项目技术与业务全景指南（中文精要版）

> 面向：首次接手本仓库的全栈工程师——希望在最短时间内建立对架构、核心业务流程、关键代码模块、开发/调试/部署路径的系统性认知。本文件比英文主 README 更偏工程实操与源码导向，可作为日常开发速查入口。

---
## 目录
1. 项目定位与总体架构概览  
2. 架构分层与模块职责  
3. 核心业务主线（从用户输入到智能执行）  
4. 关键后端代码解析速览  
5. 数据与状态流转（DB / Redis / Sandbox / 流式响应）  
6. 开发环境准备与运行方式（Wizard / 手动 / Docker / 本地调试）  
7. 常见工程任务操作手册（调试、日志、特性开关、依赖、模型切换）  
8. Agent 与工具体系（MCP / AgentPress / Sandbox 执行能力）  
9. 知识库（Knowledge Base）文件/仓库处理机制  
10. 订阅与计费/模型权限校验机制概述  
11. 目录结构导航建议  
12. 测试与验证策略（当前缺口与补全建议）  
13. 性能与扩展关注点  
14. 安全要点与潜在风险面  
15. 下一步可迭代方向建议  

---
## 1. 项目定位与总体架构概览
Suna 是一个“通用 AI 代理执行平台”，目标：通过自然语言对话驱动浏览器自动化、代码执行、文件处理、网络检索、外部 API、知识库检索等复合任务。

核心组成：
- 前端：`frontend`（Next.js + React），提供聊天 UI / 项目与 Agent 管理界面 / 流式输出渲染。
- 后端 API：`backend`（FastAPI），统一对外 REST 接口，聚合用户鉴权、线程 / 消息 / Agent 运行调度、计费校验、知识库、特性开关、MCP、邮箱、转录等。
- Agent 运行与编排：通过 `agent` 包 + `run_agent_background`（Dramatiq 任务）+ Redis 协调流式结果。
- Sandbox（Daytona）：隔离执行环境（含浏览器 / Headless / 代码解释执行 / VNC 预览），保证工具执行与文件操作安全边界。
- 数据层：Supabase（Postgres + Auth + Storage + Realtime）+ Redis（临时状态 / 流式消息 / Feature Flags）。
- LLM 访问：`services/llm.py` 基于 LiteLLM 封装多模型（OpenAI / Anthropic / OpenRouter / Bedrock 等），统一重试/工具调用/推理增强参数。
- 知识库：`knowledge_base` 支持上传文件 / ZIP / Git 仓库，解析抽取文本入库并可被 Agent 读取。
- 触发器与 OAuth：`triggers` 目录整合外部 OAuth / 统一触发型扩展能力。
- 特性开关：`backend/flags` 基于 Redis 管理可灰度发布或隐藏的功能（如 `custom_agents`）。

高层数据流：用户请求 → API 验证 & 线程/项目上下文 → 校验计费/模型权限 → 创建或复用 Sandbox → 创建 Agent Run → 后台任务拉起 LLM & 工具执行 → Redis 流式推送 → 前端 EventSource 渲染 → Run 结束持久化最终状态。

---
## 2. 架构分层与模块职责
| 层级 | 主要代码路径 | 作用 | 关键点 |
|------|--------------|------|--------|
| 接入层 | `backend/api.py` | 聚合路由、CORS、生命周期、健康检查 | 在 lifespan 中初始化 DB / Redis / 子模块 |
| 领域逻辑：Agent | `backend/agent/api.py` | 会话线程、Agent Run 管理、版本化、流式输出、文件初始上传 | 使用 Supabase 表：threads / messages / agent_runs / agents / agent_versions |
| LLM 调度 | `backend/services/llm.py` | 参数预处理、跨模型统一、重试/限速、思维链增强(`enable_thinking`) | Anthropic 特殊 header、Bedrock ARN、OpenRouter Headers |
| 执行环境 | `backend/sandbox/sandbox.py` | Daytona Sandbox 创建/启动/重启/删除，VNC 链接 | 自动 supervisord session 启动 |
| 知识库 | `backend/knowledge_base/file_processor.py` | 文件/ZIP/Git 仓库解析、OCR、结构化抽取 | 统一 entry 表记录来源与元数据 |
| 配置管理 | `backend/utils/config.py` | 环境变量集中加载 & 校验 | 缺失必需项抛错，含多 Stripe Plan ID |
| 计费/模型权限 | `backend/services/billing.py`（未展示但在引用） | 检查订阅与可用模型 | `can_use_model` / `check_billing_status` |
| 特性开关 | `backend/flags/*` | Redis Flag + CLI + API 暴露 | 宕机降级：Redis 不可用时默认 False |
| 触发器/OAuth | `backend/triggers/*` | 外部事件与统一 OAuth | `unified_oauth_api` 注入路由 |
| 邮件/转录等 | `backend/services/email*.py` / `transcription.py` | 辅助型服务 | 分路由独立注入 |
| 前端 | `frontend/` | Chat、项目/Agent 管理、文件上传、流式消费 | 通过 `/api/*` + SSE 订阅 |

---
## 3. 核心业务主线（从用户输入到响应）
以“用户通过界面输入一个新指令（含文件）”为例：
1. 前端表单 → POST `/api/agent/initiate`（`agent/api.py::initiate_agent_with_files`）。
2. API：创建 `project`（占位名）→ 创建 Daytona Sandbox（浏览器 + 文件系统）→ 创建 `thread` → 处理文件上传写入 Sandbox → 首条 `user` message 持久化。
3. 创建 `agent_runs` 记录，状态 `running`；写入 Redis 标记 `active_run:{instance}:{agent_run_id}`。
4. 通过 Dramatiq（`run_agent_background.send(...)`）后台执行：
	- 读取线程消息、Agent 配置（system prompt / MCP / 工具）
	- 调用 LLM（`make_llm_api_call`）流式/块式返回
	- 调用工具：浏览器操作 / 知识库检索 / 文件处理（视未展示的工具注册实现）
	- 每步产出写入 Redis List：`agent_run:{id}:responses` 并发布 `agent_run:{id}:new_response` channel。
5. 前端通过 SSE: GET `/api/agent-run/{id}/stream`：
	- 启动时先拉取 Redis List 已有内容
	- 订阅 Pub/Sub 等待新分片
	- 接收 `status: completed/failed/stopped` 收尾。
6. Run 结束：DB 更新 `agent_runs`（完成时间、错误、聚合响应）、Redis 关键 key TTL 过期或清理。

中途强制停止：POST `/api/agent-run/{id}/stop` → 发布 STOP → 后台感知控制通道终止。

Agent Builder 特殊模式：线程 metadata 标记 `is_agent_builder` + `target_agent_id`，用于迭代某个 Agent 的系统提示或工具配置。

---
## 4. 关键后端代码解析速览
### 4.1 `backend/api.py`
- 项目入口，生命周期 `lifespan` 初始化 DB / Redis / agent/sandbox/triggers 等。
- 汇总路由：agent、sandbox、billing、feature_flags、mcp、transcription、email、knowledge_base、triggers、oauth。
- CORS 根据 `ENV_MODE` 动态允许。  

### 4.2 `backend/agent/api.py`
核心点：
- Agent Run 生命周期：创建 / 流式 / 停止 / 状态查询。
- Agent 版本管理：`agents` + `agent_versions` + `agent_version_history`。
- 流式实现：Redis List（历史可补）+ Pub/Sub（增量事件 + 控制信号 STOP/END_STREAM/ERROR）。
- 并发控制：每个项目只允许一个活跃 Run（启动前检查并停止旧 Run）。
- 文件初始上传：写入 Sandbox `/workspace` 并在首条用户消息内容中附 “Uploaded File” 标记。

### 4.3 `services/llm.py`
- `prepare_params`：统一模型参数映射（max_tokens 兼容 o1 / claude / bedrock）、工具调用结构、OpenRouter Headers、Anthropic prompt caching 与 reasoning_effort。
- 重试：`MAX_RETRIES=2`，RateLimit 用 30s 退避，其它快速重试。
- 思考增强：`enable_thinking + reasoning_effort` 针对 Anthropic 模型追加参数。

### 4.4 `utils/config.py`
- 启动即加载 `.env` 并校验必需变量，不合法直接抛错 —— 早失败策略。
- 聚合所有第三方 Key / Stripe Plan / Sandbox 镜像 / 默认模型。

### 4.5 `knowledge_base/file_processor.py`
- 支持：纯文本 / PDF / DOCX / XLSX / 图片 OCR / JSON / YAML / XML / CSV / ZIP / Git 仓库。
- 安全与限制：大小、ZIP 内文件数、最大内容截断、编码识别、清洗不可打印字符。
- Git 仓库：浅克隆、include/exclude pattern 过滤，再逐文件建 entry。

### 4.6 `sandbox/sandbox.py`
- Daytona API：按镜像（`SANDBOX_IMAGE_NAME`）创建，配置 VNC/Chrome 变量。
- 自动启动 `supervisord`，支持重启已归档/停止的 Sandbox。

---
## 5. 数据与状态流转
| 类型 | 存储 | 说明 |
|------|------|------|
| 永久项目/线程/消息 | Supabase 表（Postgres） | 历史可审计回放 |
| Agent 定义与版本 | `agents`, `agent_versions`, `agent_version_history` | 支持版本回退与审计 |
| Agent Run 过程增量响应 | Redis List `agent_run:{id}:responses` | 保留至 TTL；SSE 启动补发 |
| Run 控制信号 | Redis Pub/Sub `agent_run:{id}:control*` | STOP / END_STREAM / ERROR |
| 运行中 Run 标记 | `active_run:{instance}:{agent_run_id}` | 实例关停清理 |
| 特性开关 | Redis Key | 异常回退默认 False |
| Sandbox 文件 | Daytona 隔离 FS `/workspace` | 上传 & 工具执行产出 |

---
## 6. 开发环境准备与运行方式
### 6.1 快速向导（推荐）
```bash
git clone https://github.com/kortix-ai/suna.git
cd suna
python setup.py   # 交互式 14 步，生成 .env
python start.py   # 启动/停止容器统一脚本
```

### 6.2 手动后端本地调试（仅 Redis/RabbitMQ 用容器）
```bash
cd backend
docker compose up redis rabbitmq
uv run api.py  # 启动 FastAPI
uv run dramatiq --processes 4 --threads 4 run_agent_background  # 任务消费者
```
注意：本地运行 API 时 `.env` 内 `REDIS_HOST=localhost` `RABBITMQ_HOST=localhost`。

### 6.3 全部容器方式
```bash
cd backend
docker compose down && docker compose up --build
```
生产：
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### 6.4 前端开发
（查看 `frontend/README.md`）典型：
```bash
cd frontend
pnpm install # 或 npm/yarn
pnpm dev
```

### 6.5 关键环境变量（节选）
参考 `backend/README.md`：Supabase / Redis / RabbitMQ / LLM Keys / Daytona / QStash / MCP / Search(Firecrawl/Tavily) 等。最少需：数据库、Redis、至少一个 LLM Key、Daytona（或临时禁用相关路径）、模型名 `MODEL_TO_USE`。

### 6.6 调试技巧
| 场景 | 方法 |
|------|------|
| 查看流式断点 | 监听 SSE `/api/agent-run/{id}/stream`，或本地用 curl / browser EventSource |
| 排查未停止 Run | Redis 查询 `active_run:*`，调用 STOP 接口 |
| 文件上传失败 | 查看 `agent/api.py` 上传日志（成功后会列出 workspace 目录文件） |
| Sandbox 无法启动 | 检查 Daytona Key / 配额，查看 `sandbox/sandbox.py` 日志 |
| LLM 请求失败 | 开启 `services/llm.py` 中 logger.debug，核对 API Key 与模型名映射 |
| Agent 版本未更新 | 确认请求 body 字段差异触发新版本；查看 `agent_versions` 表 |

---
## 7. 常见工程任务操作手册
### 7.1 切换默认模型
修改 `.env` 中 `MODEL_TO_USE`（支持别名在 `utils/constants.py` 中映射）→ 重启后端。

### 7.2 启用自定义 Agent 功能
Redis 中开启 Feature Flag：
```bash
cd backend/flags
python setup.py enable custom_agents "Enable custom agent CRUD"
```

### 7.3 添加新工具 / MCP
1. 新增工具描述（若基于 MCP：在 `mcp_service` 中注册或模板 API）。  
2. Agent 创建/更新时在 `configured_mcps` / `agentpress_tools` 中写入结构。  
3. 后台运行流程读取 agent_config 并注入 LLM 消息上下文。  

### 7.4 新增知识库文件解析器
在 `file_processor.py`：
- 增加扩展名到集合
- 编写 `_extract_xxx_content` 并在 `_extract_file_content` 分支调度
- 记录 extraction_method 便于调试。

### 7.5 排查计费或模型权限阻塞
在 `/agent/start` 或 `/agent/initiate` 报 402/403：
- 查看 `services/billing.py` 的 `check_billing_status` 与 `can_use_model`
- 校验账号订阅级别和允许模型列表。

---
## 8. Agent 与工具体系
概念关系：
- Agent（逻辑人格/策略） = system_prompt + MCP 工具列表 + 自定义工具配置 + 版本化。
- Thread：对话容器（不再绑定唯一 Agent，可运行不同 Agent Run）。
- Agent Run：一次具体执行实例（绑定模型、是否思维模式、流式任务 ID）。
- MCP（Model Context Protocol）：标准化外部工具/数据源能力，通过 `mcp_service` 统一管理。
- AgentPress Tools：内部定义工具集合（见 `agentpress` 包）。

版本机制：当关键字段（system_prompt / 工具集合等）变更时创建新版本并更新 `current_version_id`，保留历史审计。

---
## 9. 知识库文件/仓库处理机制
上传或导入 → `FileProcessor.process_file_upload` / `process_git_repository`：
1. 验证大小 / 数量限制。
2. 识别类型并调用相应解析（文本直接 decode，PDF→PyPDF2，图片→OCR，JSON/YAML/XML/CSV→结构化再序列化）。
3. 清洗不可见字符、规范换行、截断超长内容。
4. 写入表 `agent_knowledge_base_entries`（含原文件元数据）。
5. ZIP/Git：先写容器 entry，再逐文件建立子 entry。

Agent 运行时（未展示代码）可查询该表提升上下文回答质量。

---
## 10. 订阅与计费/模型权限
触发点：启动 Agent Run 前调用：
- `can_use_model`：判断账号可用模型列表（避免超出订阅）。
- `check_billing_status`：额度 / 订阅有效性。
失败返回 HTTP 402/403，前端提示升级或切换模型。

---
## 11. 目录结构导航建议（精简要点）
```
backend/
  api.py                # 统一路由与生命周期
  agent/                # Agent Run / 版本 / 流式接口
  services/             # 第三方服务 + LLM + Redis + 邮件 + 计费
  sandbox/              # Daytona 集成
  knowledge_base/       # 文件与仓库解析
  flags/                # Feature Flag 系统
  triggers/             # OAuth & 外部触发
  mcp_service/          # MCP 路由与安全模板
  utils/                # 配置 / 日志 / 常量 / 重试
frontend/               # Next.js 前端
docs/                   # 架构图等
```

---
## 12. 测试与验证策略
当前仓库未见明显自动化测试目录，建议补齐：
| 领域 | 建议测试 | 说明 |
|------|----------|------|
| Agent API | 启动/停止/流式完整 Happy Path | 使用 TestClient + Redis mock |
| LLM Wrapper | 参数映射与回退 | Mock litellm，验证 Anthropic & Bedrock 特殊分支 |
| 知识库解析 | 各文件类型抽取 | 基于临时文件；断言内容截断与元数据 |
| Feature Flags | 启用/禁用/Redis 失效 | Redis 故障默认 False 行为 |
| Config | 缺失必需 ENV 报错 | 保护启动前失败 |

快速人工验证脚本示例：
```bash
# 健康检查
curl http://localhost:8000/api/health

# 创建对话 + 运行（简化，无文件）
curl -H "Authorization: Bearer <token>" -F prompt='Hello' http://localhost:8000/api/agent/initiate

# 流式
curl http://localhost:8000/api/agent-run/<agent_run_id>/stream?token=<token>
```

---
## 13. 性能与扩展关注点
| 方向 | 现状 | 关注点 |
|------|------|--------|
| 流式与并发 | Redis List + Pub/Sub | 大量并发 Run 时 Redis 频道数量增长；可转 Channel 复用或引入消费者聚合 |
| Sandbox 成本 | 每 Project 创建 | 长时间未用可自动归档/回收（已设 auto_stop & auto_archive）|
| 模型调用重试 | 固定 2 次 | 可引入熔断与指标上报（Prometheus + Grafana）|
| 日志 | 结构化（structlog） | 需集中收集（ELK / Loki）以支持审计与问题回溯 |
| 知识库存储 | 全量文本存 Postgres | 未来可引入向量检索（pgvector / 外部 RAG 服务）|

---
## 14. 安全要点与潜在风险
| 区域 | 风险 | 建议 |
|------|------|------|
| Sandbox 文件执行 | 潜在任意代码/浏览器操控 | 最小权限镜像、网络出站控制、命令审计 |
| API 鉴权 | SSE token 校验依赖 `get_user_id_from_stream_auth` | 增加过期校验与最小权限 token |
| 环境变量 | `config` 启动即校验 | CI/CD 加密，避免在日志输出敏感 Key |
| 文件解析 | OCR / PDF / ZIP | 限制大小 & 类型（已做），继续增加沙箱化解析（外部进程）|
| Feature Flag | Redis 故障降级 | 对关键安全开关使用 DB 兜底 |

---
## 15. 下一步可迭代方向建议
1. 引入统一事件总线（如 NATS / Kafka）替代 Redis Pub/Sub，提升水平扩展性与持久化。  
2. 加入向量检索 + 语义分段，提高知识库召回精准度。  
3. 完善 CI：lint（ruff / black）+ mypy + pytest + 覆盖率。  
4. 观察指标：LLM 调用耗时、失败率、Sandbox 启动耗时、流式首 token 延迟。  
5. Agent 运行的工具调用链做可视化（Trace + 时序图）。  
6. 安全：对外部 URL 抓取增加域名白名单 / 超时 / 速率限制。  
7. 增加多租户资源配额：同时运行 Run 数量、每日 Token、Sandbox 数量等。  
8. 导出/导入 Agent 配置（YAML/JSON），支持 Marketplace 分发。  
9. SSE -> WebSocket 可选通道（提升前端兼容与控制能力）。  
10. 对大模型调用添加缓存层（语义 hash + 结果缓存）降低成本。

---
## 快速溯源索引
| 需求 | 位置 |
|------|------|
| 创建 Agent Run 入口 | `backend/agent/api.py::initiate_agent_with_files` / `start_agent` |
| 流式输出逻辑 | `backend/agent/api.py::stream_agent_run` |
| LLM 参数封装 | `backend/services/llm.py::prepare_params` |
| Sandbox 创建 | `backend/sandbox/sandbox.py::create_sandbox` |
| 知识库文件解析 | `backend/knowledge_base/file_processor.py` |
| 配置加载 | `backend/utils/config.py` |
| Feature Flag | `backend/flags/` |

---
## 附：最小本地启动脚本思路
```bash
# 1. 复制示例 .env（需自行准备 Supabase / Redis / LLM Key / Daytona）
cp .env.example .env

# 2. 启动基础服务
cd backend && docker compose up redis rabbitmq -d

# 3. 启动后端 + Worker
uv run api.py &
uv run dramatiq --processes 2 --threads 4 run_agent_background &

# 4. 启动前端
cd ../frontend && pnpm dev
```

---
**欢迎你接手继续演进 Suna，如需更详细的自托管与安装细节，请阅读 `docs/SELF-HOSTING.md` 与根目录英文 `README.md`。**  
（本文件可根据迭代持续更新，建议在 PR 模板中加入“是否需要同步更新中文 readme_min.md”提示。）

