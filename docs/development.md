# DocVault 开发环境搭建指南

本文档描述从零搭建 DocVault 完整开发环境的步骤。适用于 M0 初始化及后续日常开发。

## 本机环境 (当前状态)

| 项目       | 详情                                          |
| ---------- | --------------------------------------------- |
| OS         | Ubuntu 24.04.4 LTS (Noble Numbat), x86_64     |
| Python     | 3.12.13 (conda env `docvault`)                |
| Node.js    | v25.9.0 (nvm)                                 |
| npm        | 11.12.1                                       |
| Rust       | 1.95.0 stable (rustup)                        |
| Cargo      | 1.95.0                                        |
| make       | GNU Make 4.3                                  |
| conda      | 26.1.1, env `docvault` 已创建                 |

### 已安装的系统库 (Tauri 依赖)

`build-essential`, `libwebkit2gtk-4.1-dev`, `libappindicator3-dev`,
`librsvg2-dev`, `patchelf`, `libssl-dev`, `libgtk-3-dev` 均已安装。

### 尚需安装

| 依赖             | 说明                                     |
| ---------------- | ---------------------------------------- |
| pnpm             | `npm install -g pnpm` 或启用 corepack    |
| Docker + Compose | PostgreSQL / Caddy / LibreOffice 容器    |
| libayatana-appindicator3-dev | Tauri 系统托盘 (Ubuntu 24.04)  |

## 前置依赖 (完整清单)

| 依赖             | 最低版本     | 用途                               |
| ---------------- | ------------ | ---------------------------------- |
| Node.js          | LTS (20+)    | 桌面 UI 构建、TypeScript 检查      |
| pnpm             | 9+           | 前端包管理 (workspace)             |
| Rust             | stable 1.75+ | Tauri 2 原生层                     |
| Python           | **3.12+**    | Agent + 服务端                     |
| Docker + Compose | 24+          | PostgreSQL、Caddy、LibreOffice     |
| LibreOffice      | —            | Office 预览 (开发时可用 Docker 版) |
| make             | —            | 统一构建入口                       |

## 系统特定依赖

### Linux (Ubuntu/Debian)

```bash
# 基础构建 + Tauri 必需
sudo apt install build-essential libwebkit2gtk-4.1-dev \
  libappindicator3-dev librsvg2-dev patchelf \
  libssl-dev libgtk-3-dev

# Ubuntu 24.04 额外需要 (系统托盘)
sudo apt install libayatana-appindicator3-dev
```

### macOS

```bash
xcode-select --install
```

### Windows

- Microsoft Visual Studio C++ Build Tools
- WebView2 (Windows 11 已内置)

## 首次初始化 (M0)

```bash
# 0. 激活 conda 环境
conda activate docvault

# 1. 安装 pnpm (如尚未安装)
npm install -g pnpm

# 2. 进入项目根目录
cd docvault

# 3. 安装 Python 开发依赖 (已完成：mypy, pytest, ruff)
pip install ruff mypy pytest

# 4. [待实现] 安装 Agent 依赖
# cd apps/client/agent && pip install -e ".[dev]"

# 5. [待实现] 安装 Server 依赖
# cd apps/server && pip install -e ".[dev]"

# 6. [待实现] 安装前端依赖
# pnpm install

# 7. 准备服务端环境变量
cp .env.example .env
# 编辑 .env，设置 DOCVAULT_TOKEN_SECRET 和其他必填项

# 8. [待实现] 生成自签名证书 (局域网 HTTPS)
# bash infra/scripts/init-certs.sh
```

## 日常开发命令

通过根目录 `Makefile` 统一调用：

```bash
# 启动服务端 (PostgreSQL + API + Worker + Caddy)
make dev-server

# 单独启动 Python Agent
make dev-agent

# 启动桌面 UI (Tauri + React)
make dev-desktop

# 代码检查
make lint
make typecheck

# 运行测试
make test

# 构建
make build
```

### 分项目命令

**Agent**

```bash
cd apps/client/agent
pytest                          # 测试
ruff check .                    # lint
mypy src/                       # 类型检查
python -m docvault_agent        # 启动 (需先配置 TOML)
```

**Server**

```bash
cd apps/server
pytest                          # 测试
ruff check .                    # lint
mypy src/                       # 类型检查
alembic upgrade head            # 数据库迁移
python -m docvault_server       # 启动 API (需先启动 Postgres)
```

**Desktop**

```bash
cd apps/client/desktop
pnpm lint                       # ESLint + Prettier
pnpm typecheck                  # tsc --noEmit
pnpm test                       # Vitest
pnpm tauri dev                  # 启动开发模式
```

## 生成 api-client

当服务端 API Schema 变更后：

```bash
# 1. 确保服务端正在运行
make dev-server

# 2. 生成 TypeScript 客户端代码
bash infra/scripts/generate-api-client.sh

# 3. 检查生成代码是否最新 (CI 也会做)
make typecheck
```

生成脚本从 `http://localhost:8000/openapi.json` 拉取 OpenAPI spec，
输出到 `packages/api-client/src/generated/`。

## shared-schemas 校验

`packages/shared-schemas/` 中的 JSON 文件是枚举值的 canonical source。

Agent 和服务端各自在其测试套件中校验本地枚举值与 JSON 一致。
添加新枚举值时的流程：

1. 编辑 `packages/shared-schemas/enums/<name>.json`
2. 更新 Agent 中对应的 Python 枚举，运行 `pytest apps/client/agent/tests/`
3. 更新 Server 中对应的 Python 枚举，运行 `pytest apps/server/tests/`
4. CI 中 `ci-schemas.yml` 会做额外的格式一致性检查

## 目录结构速查

```
数据目录 (Git 忽略)：
  data/              ← DOCVAULT_DATA_DIR，服务端 Blob + 预览存储
  temp/              ← DOCVAULT_TEMP_DIR，上传/转换临时文件

开发文件 (版本控制)：
  .env.example       ← 环境变量模板，不含密钥
  .env               ← 本地环境变量，Git 忽略
```

## 常见问题

### 端口冲突

- 服务端 API：`8000`
- PostgreSQL：`5432`
- Caddy HTTPS：`443` (局域网内需 sudo，开发可用 `:8443`)
- Agent 本地 API：随机端口 (`127.0.0.1`)

修改端口：编辑 `.env` 和 `infra/docker-compose.yml`。

### LibreOffice 转换超时

默认 120 秒超时，通过 `DOCVAULT_LIBREOFFICE_TIMEOUT_SECONDS` 调整。
大型 Office 文件可能超过此限制——超时不会影响备份状态（不变量 2.5）。

### Agent 配置 TOML 示例

详见 README.md 客户端 Agent 配置示例一节。

### 数据库重置

```bash
# 删除并重建 PostgreSQL 数据卷
docker compose -f infra/docker-compose.yml down -v
docker compose -f infra/docker-compose.yml up -d postgres

# 重新运行迁移
cd apps/server && alembic upgrade head
```
