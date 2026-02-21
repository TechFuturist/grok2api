# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Grok2API 是一个将 Grok Web 接口转换为 OpenAI 兼容 API 的代理服务。项目有两个独立的运行时实现：
- **FastAPI (Python)**: 本地开发/Docker 部署，位于 `app/` 和 `main.py`
- **Cloudflare Workers (TypeScript)**: 边缘部署，位于 `src/`

两个实现共享相同的管理 UI（`app/static/`）和 API 语义。

## Development Commands

### Python (FastAPI)

```bash
# 依赖安装与本地运行
uv sync
uv run main.py

# 运行测试
uv run pytest -q

# 启动后自检
python scripts/smoke_test.py --base-url http://127.0.0.1:8000

# 模型目录同步检查（确保 Python 与 TypeScript 模型定义一致）
python scripts/check_model_catalog_sync.py
```

### TypeScript (Cloudflare Workers)

```bash
# 类型检查
npm run typecheck

# 本地开发
npm run dev

# 部署前预检
npx wrangler deploy --dry-run --config wrangler.toml

# D1 数据库迁移
npm run db:migrate
```

### Docker

```bash
# 直接拉取镜像启动
docker compose up -d

# 从源码构建
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build

# 验证配置
docker compose -f docker-compose.yml config
```

## Architecture

### Python FastAPI (`app/`)

```
app/
├── api/v1/           # API 路由
│   ├── admin.py      # 管理后台 API（Token/Key/Config）
│   ├── chat.py       # /v1/chat/completions
│   ├── image.py      # /v1/images/generations, /v1/images/edits
│   └── models.py     # /v1/models
├── core/
│   ├── config.py     # 配置管理（TOML 加载/合并/迁移）
│   ├── storage.py    # 存储抽象（local/redis/mysql/pgsql）
│   └── auth.py       # API Key 验证
├── services/
│   ├── grok/         # Grok 核心逻辑
│   │   ├── chat.py       # 对话处理
│   │   ├── model.py      # 模型定义（与 src/grok/models.ts 保持同步）
│   │   ├── processor.py  # 响应流处理
│   │   └── imagine_*.py  # 图像生成
│   ├── token/        # Token 池管理
│   │   ├── pool.py       # Token 选择与轮换
│   │   ├── scheduler.py  # 定时刷新
│   │   └── service.py    # Token CRUD
│   └── register/     # 自动注册服务
│       ├── manager.py    # 注册流程编排
│       └── solver.py     # Turnstile 验证码解决器
└── static/           # 管理 UI（共享给 Workers）
```

### TypeScript Workers (`src/`)

```
src/
├── index.ts          # Hono 应用入口
├── routes/
│   ├── openai.ts     # /v1/* OpenAI 兼容 API
│   ├── admin.ts      # /api/v1/admin/* 管理 API
│   └── media.ts      # /images/* 缓存代理
├── grok/
│   ├── models.ts     # 模型定义（与 app/services/grok/model.py 保持同步）
│   ├── conversation.ts
│   └── processor.ts
├── repo/             # D1 数据库访问层
│   ├── tokens.ts
│   ├── apiKeys.ts
│   └── cache.ts
└── kv/               # KV 缓存管理
```

## Key Patterns

### 模型定义同步
`app/services/grok/model.py` 和 `src/grok/models.ts` 必须保持一致。修改模型后运行：
```bash
python scripts/check_model_catalog_sync.py
```

### Token 池策略
- `ssoBasic`: Basic 账号，80次/20h
- `ssoSuper`: Super 账号（仅用于 `grok-4-heavy`）
- Token 选择逻辑在 `app/services/token/pool.py` 和 `src/repo/tokens.ts`

### 配置层级
1. `config.defaults.toml` - 默认值基线
2. `data/config.toml` - 运行时配置
3. 旧版 `data/setting.toml` 会自动迁移

### 存储后端
通过 `SERVER_STORAGE_TYPE` 切换：`local`（默认）/ `redis` / `mysql` / `pgsql`

## Important Conventions

- 两套代码（Python/TypeScript）的 API 语义必须一致
- 管理 UI 是共享的，修改 `app/static/` 会同时影响两个运行时
- Token 归一化：所有 Token 操作需经过 `sso=` 前缀处理
- 图像生成有两种模式：`legacy`（旧方法）和 `imagine_ws_experimental`（新 WebSocket 方法）
