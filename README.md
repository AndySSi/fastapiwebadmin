# 项目运行指南（本机开发模式）

本文档描述在 **WSL2 / Linux** 本机直接运行本项目的完整步骤：
**MySQL、Redis 跑在 Docker 容器里，后端（FastAPI）和前端（Vite）跑在本机。**

> 生产环境可直接用根目录的 [docker-compose.yml](docker-compose.yml) 一键全容器部署，本文档面向**本地开发**。

---

## 1. 架构与端口

| 组件 | 运行方式 | 地址 / 端口 |
| --- | --- | --- |
| MySQL 8.0 | Docker 容器 | `localhost:3306` |
| Redis 6.0 | Docker 容器 | `localhost:6379` |
| 后端 API（FastAPI） | 本机 uv/venv | http://127.0.0.1:8100 |
| 前端（Vite + Vue3） | 本机 yarn | http://localhost:3000 |
| Celery worker / beat | 本机（可选） | — |

**默认登录账号：`admin` / `123456`**

---

## 2. 前置依赖

| 工具 | 版本 | 说明 |
| --- | --- | --- |
| Docker | 已安装并运行 | 跑 MySQL / Redis |
| uv | 0.11.x | Python 包/环境管理（`uv --version` 验证） |
| Python | 3.11 | 由 uv 管理，无需手动装 |
| Node.js | 18.x | 前端构建 |
| Yarn | 1.22.x | **必须用 yarn**（见下方说明） |

---

## 3. 启动 MySQL 和 Redis（Docker）

在仓库根目录执行，只启动这两个数据服务（不带后端/前端容器）：

```bash
docker compose up -d mysql redis
docker compose ps          # 确认两个容器都是 healthy
```

容器配置（来自 [docker-compose.yml](docker-compose.yml)）：
- MySQL root 密码：`root123456`，自动创建数据库 `fastapiwebadmin`
- Redis：**无密码**

---

## 4. 启动后端（FastAPI）

### 4.1 进入目录并准备环境

```bash
cd backend
uv sync                    # 按 pyproject.toml + uv.lock 创建 .venv 并安装依赖
```

### 4.2 配置 `.env`

复制示例并确认连接串与容器一致（`backend/.env`）：

```dotenv
# redis（容器无密码）
REDIS_URI=redis://localhost:6379/4

# mysql 异步（应用用）
MYSQL_DATABASE_URI=mysql+asyncmy://root:root123456@localhost:3306/fastapiwebadmin?charset=UTF8MB4

# mysql 同步（Alembic 迁移用，必填，否则启动报缺字段）
MYSQL_DATABASE_URI_SYNC=mysql+pymysql://root:root123456@localhost:3306/fastapiwebadmin?charset=UTF8MB4

# celery
CELERY_BROKER_URL=redis://localhost:6379/5
CELERY_RESULT_BACKEND=redis://localhost:6379/5
CELERY_BEAT_DB_URL=mysql+pymysql://root:root123456@localhost:3306/fastapiwebadmin?charset=UTF8MB4
```

> ⚠️ 如果你的容器密码不是 `root123456`，请把上面 4 处密码同步改掉。

### 4.3 初始化数据库（首次运行必做）

数据库是空的，需要建表 + 导入初始数据（账号、菜单、角色等）。

```bash
# 1) 建表（执行 Alembic 迁移）
.venv/bin/python -m alembic upgrade head

# 2) 导入初始数据（完整 dump：admin 用户 / 42 菜单 / 12 角色 等）
docker exec -i fast-admin-mysql mysql -uroot -proot123456 fastapiwebadmin < db_script/db_init.sql
.venv/bin/python -m alembic stamp head      # 让迁移版本与 dump 对齐
```

> 说明：`db_script/db_init.sql` 是一份完整 mysqldump（建表 + 数据）。直接整份导入比用 `cli.py seed` 可靠（seed 只挑 INSERT，列顺序会错位）。

### 4.4 启动服务

```bash
.venv/bin/python start.py          # 监听 0.0.0.0:8100，带热重载
```

验证：
- API 文档：http://127.0.0.1:8100/docs
- ReDoc：http://127.0.0.1:8100/redoc
- 健康检查：http://127.0.0.1:8100/api/health/health （返回 200）

### 4.5（可选）启动 Celery

异步任务 / 定时任务需要时，**另开终端**：

```bash
cd backend
# Worker
.venv/bin/python -m celery -A celery_worker.worker.celery worker --pool=gevent -c 10 -l INFO
# Beat（定时调度，再开一个终端）
.venv/bin/python -m celery -A celery_worker.worker.celery beat -S celery_worker.scheduler.schedulers:DatabaseScheduler -l INFO
```

---

## 5. 启动前端（Vite + Vue3）

```bash
cd frontend

# 必须用 yarn（npm 会因 pinia 插件 peer 冲突失败）
corepack enable
corepack prepare yarn@1.22.22 --activate
yarn install --frozen-lockfile

yarn dev                   # 监听 http://localhost:3000
```

前端接口地址在 [frontend/.env.development](frontend/.env.development) 配置，默认指向 `http://127.0.0.1:8100`，与后端一致。

> WSL 无图形界面时，自动打开浏览器会报错。可设 `VITE_OPEN=false yarn dev`，再手动在 Windows 浏览器打开 http://localhost:3000。

---

## 6. 访问与登录

1. 浏览器打开 **http://localhost:3000**
2. 账号 `admin` / 密码 `123456`

---

## 7. 停止服务

```bash
pkill -f start.py          # 停后端
pkill -f vite              # 停前端
pkill -f celery            # 停 Celery（如启动过）
docker compose stop mysql redis   # 停数据库容器（数据保留在卷里）
```

---

## 8. 常见问题（FAQ）

**Q1. 后端启动报 `MYSQL_DATABASE_URI_SYNC Field required`**
`.env` 缺少同步连接串。补上第 4.2 节里的 `MYSQL_DATABASE_URI_SYNC` 即可（Alembic 用同步驱动）。

**Q2. 登录报 `password cannot be longer than 72 bytes`**
`bcrypt` 版本与 `passlib 1.7.4` 不兼容（新版 bcrypt 把 72 字节超限从静默截断改成抛错）。已在 [backend/pyproject.toml](backend/pyproject.toml) 钉死 `bcrypt==4.0.1`，执行 `uv sync` 后重启后端即可。

**Q3. `/docs` 报 `FileNotFoundError: static/swagger/swagger.html`**
已修复：[backend/main.py](backend/main.py) 改用 FastAPI 内置 `get_swagger_ui_html` / `get_redoc_html` 指向本地静态资源。若仍报错，确认 `static/swagger/` 下的 js/css 资源存在。

**Q4. 前端 `npm install` 报 peer 依赖冲突（pinia）**
本项目用 yarn 管理（自带 `yarn.lock`）。按第 5 节用 yarn 安装；不要用 npm。

**Q5. VS Code 里 `import fastapi/uvicorn could not be resolved`（Pylance 报红）**
不是真缺包，是解释器没选对。已在 [.vscode/settings.json](.vscode/settings.json) 指向 `backend/.venv`。如仍报红：`Ctrl+Shift+P` → Python: Select Interpreter → 选 `./backend/.venv/bin/python`，或 Reload Window。

**Q6. MySQL 客户端里中文显示成 `??`**
那只是 `docker exec` 终端字符集问题，库里存的是 utf8mb4，数据本身正常，不影响应用。
