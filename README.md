# DocVault

DocVault 是一款面向个人、多设备和局域网/NAS 环境的跨平台文档自动备份与版本管理桌面工具。

它会在本地监听指定目录，把文档的不可变历史版本上传到局域网服务器，并提供文件预览、版本 Diff、下载和安全恢复。DocVault 的定位是**备份**，不是网盘，也不是多设备同步工具。

> 当前状态：规格与工程骨架阶段。本文描述已确认的第一版目标；仓库中的命令和目录应在初始化后保持与本文一致。

## 核心特性

- Windows、macOS、Linux 桌面客户端。
- 多台设备备份到同一台 NAS/局域网服务器。
- 每台设备拥有完全独立的文件树和版本历史。
- 文件停止变化 10 分钟后备份，避免高频编辑产生大量版本。
- 每 30 分钟补偿扫描，修复文件监听漏报。
- 断网任务落盘，恢复连接后自动重试。
- SHA-256 内容寻址存储和重复内容去重。
- 历史版本不可变，永不自动清理。
- 本地删除只生成 tombstone，旧版本继续保留。
- Markdown、文本和常见代码格式支持预览与 Diff。
- PDF、图片以及 DOCX/XLSX/PPTX 支持预览。
- 恢复到原路径或用户选择的新路径。
- 关闭窗口后缩小到系统托盘并继续备份。
- 可为目录设置应用内界面密码锁。

## 重要边界

### 这是备份，不是同步

服务端不会把某台设备的变化自动下载到其他设备。不同设备中名称相同、相对路径相同的文件仍是两个独立文件。

```text
Windows-PC / Documents / notes.md
MacBook    / Documents / notes.md
```

它们不会合并历史，也不会产生跨设备分支。用户只有在主动执行恢复时，服务端内容才会写回本地。

### 界面密码锁不是加密

第一版的密码功能只限制通过 DocVault 界面打开受保护目录。原始文件在 NAS 上仍以明文形式存储；拥有 NAS 或数据库管理权限的人仍可能读取内容。

### 预览失败不等于备份失败

原始 Blob 上传并校验完成后，版本即视为备份成功。LibreOffice 转换、Markdown 渲染或 Diff 生成失败只会影响派生预览，可单独重试，不会删除或回滚历史版本。

## 总体架构

```text
┌──────────────────────────────────────────────┐
│                 桌面客户端                   │
│                                              │
│  Tauri 2                                     │
│  ├─ React + TypeScript 桌面界面              │
│  ├─ 系统托盘、开机启动、通知、文件选择器     │
│  └─ 启动并管理 Python Agent                  │
│                                              │
│  Python Agent                                │
│  ├─ FastAPI 本地接口（127.0.0.1 随机端口）  │
│  ├─ watchdog 文件监听                        │
│  ├─ 10 分钟防抖 / 30 分钟补偿扫描            │
│  ├─ SQLite 任务队列                          │
│  ├─ SHA-256、分块上传、断网重试              │
│  └─ 下载与安全恢复                           │
└──────────────────────┬───────────────────────┘
                       │ LAN / HTTPS
┌──────────────────────▼───────────────────────┐
│              NAS / 局域网服务端              │
│                                              │
│  FastAPI                                     │
│  ├─ 设备注册与令牌认证                       │
│  ├─ 上传、版本、历史、下载和恢复接口         │
│  ├─ 内容寻址 Blob 存储                       │
│  ├─ Diff 与预览任务                          │
│  └─ 永久删除与审计                           │
│                                              │
│  PostgreSQL：元数据                          │
│  NAS 文件系统：原始 Blob 与预览派生物        │
│  Worker + LibreOffice headless：Office 预览  │
│  Caddy：局域网 HTTPS                         │
└──────────────────────────────────────────────┘
```

## Monorepo 结构

```text
docvault/
├── AGENTS.md
├── README.md
├── apps/
│   ├── client/
│   │   ├── README.md
│   │   ├── desktop/              # Tauri 2 + React + TypeScript
│   │   │   ├── src/
│   │   │   ├── src-tauri/
│   │   │   └── package.json
│   │   └── agent/                # Python 本地备份 Agent
│   │       ├── src/
│   │       ├── tests/
│   │       └── pyproject.toml
│   └── server/                   # NAS / 局域网 FastAPI 服务
│       ├── README.md
│       ├── src/
│       ├── tests/
│       ├── migrations/
│       ├── pyproject.toml
│       └── Dockerfile
├── packages/
│   ├── api-client/               # OpenAPI 生成的 TypeScript 客户端
│   └── shared-schemas/           # 协议级 Schema 和枚举
├── infra/
│   ├── docker-compose.yml
│   ├── caddy/
│   └── libreoffice/
└── docs/
    ├── architecture.md
    ├── backup-semantics.md
    ├── security.md
    └── api.md
```

客户端与服务端是两个可独立构建、独立发布、独立运行的项目。客户端内部的桌面 UI 与 Python Agent 共同组成一个桌面产品。

## 技术栈

### 桌面客户端

- Tauri 2
- React
- TypeScript
- Vite
- TanStack Query
- Zustand
- Monaco Editor 或 CodeMirror
- `react-markdown`
- Python 3.12+
- FastAPI
- watchdog
- httpx
- SQLite
- PyInstaller

### NAS 服务端

- Python 3.12+
- FastAPI
- SQLAlchemy 2
- Alembic
- PostgreSQL
- NAS 本地文件系统
- 后台 Worker
- LibreOffice headless
- Caddy
- Docker Compose

第一版不需要 MinIO 或 Redis。文件实体直接写入 NAS 挂载目录；若后续出现多节点、对象存储兼容或任务吞吐需求，再评估引入。

## 第一版功能范围

### 1. 自动备份

用户可在每台设备配置多个备份根目录。Agent 监听创建、修改、移动、重命名和删除事件。

```text
文件事件
  → 更新 10 分钟防抖截止时间
  → 文件稳定后计算 SHA-256
  → 与最近成功版本哈希比较
  → 内容变化时创建上传任务
  → 上传并在服务端校验
  → 创建不可变 FileVersion
```

补偿机制：

- 每 30 分钟扫描一次所有启用的备份根目录。
- 扫描结果与 SQLite 中的最近状态比较。
- 发现监听漏报时补建创建、修改、移动或删除任务。
- 服务器不可用时任务保留在 SQLite，并使用有上限的指数退避重试。

默认排除规则建议包括：

```gitignore
.git/
node_modules/
.DS_Store
*.tmp
*.swp
~$*
```

用户可以为每个备份根目录配置类似 `.gitignore` 的规则。

### 2. 文件历史

每个版本至少记录：

- 设备与备份根目录。
- 相对路径和文件名。
- 文件大小、MIME 类型和扩展名。
- SHA-256 内容哈希。
- 文件修改时间、客户端上传时间和服务端接收时间。
- 版本号、来源事件和可选备注。
- Blob 引用、预览状态和删除状态。

历史版本采用追加式设计。旧版本不能被客户端覆盖。

### 3. 删除与永久保留

```text
本地删除
  → Agent 上报删除事件
  → 服务端创建 tombstone 版本
  → 文件在 UI 中显示“已删除”
  → 所有旧版本继续保留
```

系统不设置自动清理期限，也不进行后台历史压缩。用户可手动永久删除，但必须：

- 查看将删除的版本和预计释放空间。
- 输入明确确认文字。
- 重新验证界面密码锁。
- 生成审计日志。
- 只清理不再被任何版本引用的 Blob。

### 4. Diff

以下格式支持历史预览和文本 Diff：

```text
.md .txt .json .yaml .yml .csv .xml
.html .css .js .ts .py .java .go .rs
.toml .ini .log
```

第一版至少提供：

- Unified Diff。
- 左右并排 Diff。
- 忽略空白字符。
- 忽略行尾差异。
- 任意两个版本之间跳转和比较。
- 超大文件降级为下载。

Markdown 同时提供源码视图、渲染视图、标题导航和代码块高亮。

### 5. 文件预览

| 类型 | 第一版预览方式 | Diff |
|---|---|---|
| Markdown | 渲染视图 + 源码视图 | 是 |
| 文本和代码 | 行号 + 语法高亮 | 是 |
| JSON / YAML | 格式化展示 | 是 |
| CSV | 表格预览 | 是 |
| PDF | 内嵌 PDF 查看器 | 否 |
| PNG/JPG/JPEG/GIF/WebP/SVG | 图片预览 | 否 |
| DOCX | LibreOffice 转换为 PDF/HTML 派生预览 | 否 |
| XLSX | LibreOffice 转换为 PDF/HTML 派生预览 | 否 |
| PPTX | LibreOffice 转换为 PDF 派生预览 | 否 |

Office 转换是异步任务。转换失败时仍必须允许查看历史、下载原文件和重试转换。

### 6. 恢复

#### 恢复到原路径

```text
下载历史版本
  → 校验哈希
  → 检查当前文件
  → 当前内容不同则先备份
  → 写入同目录临时文件
  → 再次校验哈希
  → 原子替换目标文件
  → 记录恢复事件
```

不得直接覆盖当前文件。恢复前的内容必须形成一个新版本。

#### 恢复到可选路径

桌面端调用系统文件选择器，让用户选择：

- 新文件名。
- 新目录。
- 是否覆盖已有文件。
- 是否保留历史版本的修改时间。

整个目录按时间点恢复不在第一版范围内。

### 7. 桌面界面

第一版建议包含六个主页面：

1. **概览**：服务器状态、上次成功备份、等待任务、失败任务、存储使用量和最近文件。
2. **备份目录**：添加/移除目录、暂停、排除规则、文件大小限制和扫描频率。
3. **文件库**：按“设备 → 备份根目录 → 文件树”浏览。
4. **文件详情**：预览、元数据、历史、Diff、下载和恢复。
5. **任务中心**：上传中、等待重试、失败、恢复任务和扫描进度。
6. **设置**：NAS 地址、HTTPS 证书、设备名称、开机启动、本地缓存、日志和密码锁规则。

关闭主窗口后应用进入系统托盘。只有“退出 DocVault”才停止 Agent。

## 数据模型概览

### 客户端 SQLite

建议包含：

- `backup_roots`
- `scan_entries`
- `debounce_entries`
- `backup_tasks`
- `restore_tasks`
- `server_profiles`
- `local_settings`

客户端数据库只描述本机状态，不复制服务端业务模型。

### 服务端 PostgreSQL

建议包含：

- `devices`
- `device_tokens`
- `backup_roots`
- `file_entries`
- `file_versions`
- `blobs`
- `preview_jobs`
- `preview_artifacts`
- `password_lock_rules`
- `audit_events`

文件身份必须包含设备和备份根目录，禁止只用相对路径做全局唯一键。

## 内容寻址存储

建议按 SHA-256 保存原始 Blob：

```text
/data/
├── blobs/
│   └── ab/
│       └── cd/
│           └── abcdef...<full_sha256>
├── previews/
└── temp/
```

写入步骤：

1. 上传到独立临时文件。
2. 校验大小与 SHA-256。
3. 若目标 Blob 已存在则复用。
4. 否则原子移动到最终路径。
5. 在数据库事务中创建 Blob 引用和 FileVersion。

预览派生物必须与原始 Blob 分离，删除预览缓存不影响历史版本。

## API 契约

服务端 FastAPI 是 API Schema 的唯一来源：

```text
FastAPI OpenAPI Schema
        ↓
生成 packages/api-client
        ↓
React 桌面 UI 与其他 TypeScript 代码使用
```

Python Agent 通过独立 HTTP 客户端模块访问服务端。客户端和服务端不得共享 ORM 模型。

API 应覆盖：

- 设备注册、令牌刷新和撤销。
- 备份根目录注册。
- 上传会话、分块上传、提交和校验。
- 文件创建、修改、移动、重命名和 tombstone。
- 文件树、历史、版本详情和 Diff。
- 预览任务状态和派生物下载。
- 原始版本下载。
- 永久删除和审计查询。

## 安全模型

DocVault 默认只在局域网运行，但仍执行以下安全措施：

- Caddy 提供 HTTPS。
- 每台设备使用独立、可撤销的访问令牌。
- 本地 Agent 只监听 `127.0.0.1` 随机端口，并使用启动时临时 Token。
- 服务端不接受客户端绝对路径，只接受经过规范化的相对路径。
- 防止 `..` 路径穿越、符号链接逃逸和非法文件名。
- 上传设置文件大小和并发限制。
- LibreOffice 任务设置超时、资源限制和隔离临时目录。
- 日志不记录文件正文、密码或完整令牌。
- 恢复、永久删除、密码锁和设备令牌操作写入审计日志。

## 开发环境

### 前置依赖

开发完整系统通常需要：

- Node.js LTS 与包管理器（仓库初始化时固定 pnpm 或 npm）。
- Rust stable 与 Tauri 2 系统依赖。
- Python 3.12+。
- Docker 与 Docker Compose。
- PostgreSQL（推荐通过 Compose）。
- LibreOffice headless（推荐只安装在服务端/Worker 容器）。

PyInstaller 不支持跨平台交叉构建，因此：

- Windows 安装包在 Windows Runner 构建。
- macOS 安装包在 macOS Runner 构建。
- Linux 安装包在 Linux Runner 构建。

### 目标开发入口

仓库初始化后应统一提供以下命令：

```bash
make dev-server       # 启动 PostgreSQL、服务端 API、Worker 和 Caddy
make dev-agent        # 启动本地 Python Agent
make dev-desktop      # 启动 Tauri + React 桌面端
make lint
make typecheck
make test
make build
```

这些命令在代码尚未初始化时可能还不存在。新增子项目时应优先实现并维护统一入口，而不是让使用者记住多套零散命令。

### 服务端环境变量示例

```dotenv
DOCVAULT_ENV=development
DOCVAULT_DATABASE_URL=postgresql+psycopg://docvault:docvault@postgres:5432/docvault
DOCVAULT_DATA_DIR=/app/data
DOCVAULT_PREVIEW_DIR=/app/data/previews
DOCVAULT_TEMP_DIR=/app/data/temp
DOCVAULT_PUBLIC_BASE_URL=https://docvault.lan
DOCVAULT_TOKEN_SECRET=replace-me
DOCVAULT_MAX_UPLOAD_BYTES=104857600
DOCVAULT_LIBREOFFICE_TIMEOUT_SECONDS=120
```

### 客户端 Agent 配置示例

```toml
[device]
name = "My-Laptop"

[server]
base_url = "https://docvault.lan"

[backup]
debounce_seconds = 600
full_scan_interval_seconds = 1800

[[backup.roots]]
path = "~/Documents"
exclude = [".git/", "node_modules/", "*.tmp", "*.swp", "~$*"]
```

设备令牌应存入操作系统安全凭据存储，不应直接写在普通配置文件中。

## 建议开发里程碑

### M0：仓库与契约

- 建立 monorepo 目录。
- 初始化客户端 desktop、客户端 agent 和服务端。
- 建立 Docker Compose、PostgreSQL 和 Caddy。
- 定义状态枚举、错误码和 OpenAPI 生成流程。
- 提供统一 lint、test、typecheck、build 命令。

### M1：单文件端到端备份

- 设备注册与令牌认证。
- Agent 手动扫描一个备份目录。
- SHA-256 上传和 NAS Blob 写入。
- 创建不可变版本。
- UI 展示设备文件树和版本列表。

### M2：可靠自动备份

- watchdog 监听。
- 10 分钟持久化防抖。
- 30 分钟补偿扫描。
- SQLite 队列、断网重试和任务中心。
- 创建、修改、移动、重命名和删除事件。

### M3：预览与 Diff

- Markdown、文本、代码、JSON、YAML、CSV 预览。
- Unified 和并排 Diff。
- PDF 与图片预览。
- 大文件降级策略。

### M4：Office 预览

- LibreOffice Worker。
- DOCX、XLSX、PPTX 转换。
- 超时、失败隔离、重试和缓存。
- 预览失败不影响备份状态的集成测试。

### M5：安全恢复与桌面体验

- 恢复到原路径和可选路径。
- 恢复前自动备份、临时写入、哈希校验和原子替换。
- 系统托盘、开机启动和桌面通知。
- 应用内目录密码锁和危险操作确认。

### M6：跨平台发布

- Windows `.msi` / `.exe`。
- macOS `.dmg`。
- Linux `.AppImage` / `.deb`。
- 三平台 CI 构建和最小冒烟测试。

## 第一版非目标

- 自动跨设备同步。
- 多用户、团队共享与细粒度权限。
- 公网暴露和 SaaS 托管。
- 端到端加密或服务器不可读加密。
- 在线编辑、协作和自动合并。
- 整目录时间点恢复。
- 自动历史清理或分层保留。
- MinIO、Redis、Kubernetes 或多节点高可用。
- 音视频预览、OCR、全文搜索和内容推荐。

## 开发规则

开始修改代码前请阅读 [AGENTS.md](./AGENTS.md)。其中包含数据安全、设备隔离、历史不可变、预览隔离、恢复流程和 AI 代理工作方式等强制约束。

任何实现都应优先保证：

1. 不丢数据。
2. 不覆盖历史。
3. 不混合设备文件树。
4. 离线任务可恢复。
5. 预览失败不影响原始备份。
6. 危险操作可审计、可确认。
