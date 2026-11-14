# CalendarDIFF

一个监视学生课程日历订阅和邮箱的工具：把截止日期的变化整理成可审阅的 diff，并发出提醒。

[English](README.md)

## 状态

不再维护。

CalendarDIFF 写于 2026 年初，目的是把一整套系统从头到尾接通：数据源接入、解析、canonical 事件库、审核队列、Web 界面和通知。动手之前没有先确认学生是否需要它。目前没有人在用。

代码可以在本地跑起来（步骤见下），大部分测试能通过，但没有线上实例，也没有继续开发的计划。

## 怎么跑

以下步骤在一个全新 clone 上启动后端和前端。验证时用的是 Python 3.12；后端镜像用 Python 3.11（`Dockerfile`），前端镜像用 Node 20（`frontend/Dockerfile`）。

### 1. PostgreSQL 和 Redis

Docker 在运行时：

```bash
docker compose up -d postgres redis
```

这会在 `5432` 启动 PostgreSQL 16（数据库 `deadline_diff`，用户名和密码都是 `postgres`），在 `6379` 启动 Redis 7，和 `.env.example` 一致。

不用 Docker 的话，自己起 PostgreSQL 和 Redis，为应用建一个数据库，再把 `.env` 里的 `DATABASE_URL` 和 `REDIS_URL` 指过去。

### 2. 后端

```bash
cp .env.example .env
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

set -a; source .env; set +a
python -m alembic upgrade head
SERVICE_NAME=backend PORT=8200 ./scripts/start_service.sh
```

另开一个终端检查：

```bash
curl http://127.0.0.1:8200/health
```

给 `./scripts/start_service.sh` 加上 `RUN_MIGRATIONS=true` 时，它会自己跑迁移。

### 3. 前端

```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev -- --hostname 127.0.0.1 --port 3000
```

打开 `http://127.0.0.1:3000`。`frontend/.env.local` 里的 `BACKEND_API_KEY` 必须和 `.env` 里的 `APP_API_KEY` 一致（两个示例文件用的是同一个占位值）。

`.env` 已存在、`frontend/node_modules` 已安装时，`./scripts/dev_stack.sh up` 可以一条命令完成第 1 到 3 步。它需要 Docker。

### 4. 测试

测试需要 PostgreSQL 和 Redis，会自己创建和删除测试库。

```bash
TEST_DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/deadline_diff_test \
REDIS_URL=redis://127.0.0.1:6379/15 \
python -m pytest -q
```

全新 clone 上会失败或跳过的测试见“限制”。

## 设计

- **后端：** 一个 FastAPI 进程（`services/app_api/main.py`），底层是 PostgreSQL（SQLAlchemy、Alembic 迁移）和 Redis（任务队列和 LLM 请求限流）。模块边界在 `app/modules/` 下。
- **前端：** `frontend/` 下的 Next.js 应用，通过 `BACKEND_BASE_URL` 访问后端。
- **接入：** Canvas ICS 订阅和 Gmail（OAuth，只读）作为数据源轮询。新增或变化的条目经过解析，再和 canonical 事件基线比对。
- **审核：** 首次导入形成基线，用户审核一次。之后的变化进入同一个审核队列（`Changes`），每条变更被批准、修改或拒绝之后才会写入基线。开启时通过 SMTP 发通知。
- **Agent 和 MCP：** 内部 gateway（`app/modules/agents/gateway.py`）和 MCP server（`services/mcp_server/main.py`）提供只读查询、proposal 和 approval ticket。

更详细的文档：

- [PURPOSE.md](PURPOSE.md)：系统要检测什么、忽略什么
- [docs/architecture.md](docs/architecture.md)：运行时模型和模块边界
- [docs/project_structure.md](docs/project_structure.md)：仓库结构
- [docs/api_surface_current.md](docs/api_surface_current.md)：HTTP 路由
- [docs/frontend_backend_contracts.md](docs/frontend_backend_contracts.md)：前端依赖的接口约定
- [docs/README.md](docs/README.md)：其余文档索引

OpenAPI 快照在 `contracts/openapi/public-service.json`，用 `python scripts/update_openapi_snapshots.py` 刷新。

## 限制

- **解析需要 LLM。** ICS 和 Gmail 条目都通过 `app/modules/llm_gateway` 解析，需要在 `.env` 里配置 `LLM_API_KEY`、`LLM_MODEL` 和 `LLM_BASE_URL`。不配置时后端仍能启动，但解析会报 `LLM_API_KEY is not configured`。用 LLM 改写 agent proposal 文案（`AGENT_GENERATION_MODE=llm_assisted`）和 `training/` 下的 BERT Gmail 过滤器是可选的，默认关闭。
- **Gmail 需要你自己的 Google OAuth client。** 把 `GMAIL_OAUTH_CLIENT_SECRETS_FILE` 指向一个 client secrets 文件。
- **依赖生成 fixture 的测试在全新 clone 上会跳过。** 有三个测试读取 `tests/fixtures/private/email_pool/`，这个目录被 gitignore，缺失时跳过。`python -m tools.datasets.export_year_timeline_fixtures` 会根据合成的学年时间线生成这些 fixture；`tests/test_year_timeline_fixture_export.py` 运行时也会顺带生成，所以在同一个 clone 里第二次跑测试时它们不再跳过。第二次运行时 `test_fake_source_provider_can_overlay_full_sim_email_bucket` 会失败，因为 `scripts/fake_source_provider.py` 从一个写死的绝对路径读取 email pool（见下）。
- **训练相关测试会跳过**，除非装了 `evaluate`、`numpy`、`torch`、`datasets` 和 `transformers`。它们不在 `requirements.txt` 里。
- **全新 clone 上有三个测试失败：**
  - `tests/test_full_repo_validation_runner.py` 里的两个 preflight 测试，因为 `run_preflight` 还会检查磁盘上的 `frontend/node_modules`、`data/` 下的合成 manifest 和生成的 email pool；
  - `tests/test_oauth_runtime_config.py::test_frontend_app_base_url_falls_back_to_first_public_origin`，测试期望 `PUBLIC_WEB_ORIGINS` 的第二项，而代码返回第一项。
- **写死的本地路径。** 下面这些文件里有原作者机器上的绝对路径，换一台机器不改就用不了：
  - `scripts/compare_gmail_cache_modes.py`
  - `scripts/fake_source_provider.py`
  - `scripts/probe_gmail_live_accuracy.py`
  - `scripts/probe_real_llm_usage.py`
  - `specs/frontend/2026-03-23-agent-entry-surfaces/GUIDANCE_PROMPT.md`
  - `specs/frontend/2026-03-23-agent-entry-surfaces/OUTPUT.md`
  - `specs/frontend/2026-03-23-perceived-performance-tuning/GUIDANCE_PROMPT.md`
  - `specs/frontend/2026-03-23-perceived-performance-tuning/OUTPUT.md`
- **`scripts/run_full_repo_validation.py`** 依赖原作者的本地环境，在全新 clone 上跑不通。
- **依赖上限：** `fastapi` 限制在 `0.130` 以下，`starlette` 限制在 `1.0` 以下。用更新的版本（例如 FastAPI 0.162 搭配 Starlette 1.10）时，应用启动会报 `OAuth callback route is not registered in input service app`。

## 谁写了什么

代码几乎全部由 coding agent 编写。方向和需求由我定：监听哪些来源，截止时间的变化怎样变成可审阅的 diff，什么时候提醒。

## 许可

No license has been chosen.（尚未选择许可证。）
