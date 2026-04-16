# OpenHarness Enterprise 后端技术选型研究报告

> 基于 PRD.yaml (OHE-001) 和 require.md 中的技术选型决策，为各微服务的落地实现提供具体方案。

---

## 1. FastAPI + SQLAlchemy 2.0 Async 架构

### 1.1 推荐项目结构

```
services/
└── {service_name}/
    ├── pyproject.toml
    ├── alembic.ini                  # 仅需要 DB 的服务
    └── src/{service_name}/
        ├── __init__.py
        ├── __main__.py              # python -m {service_name}
        ├── main.py                  # FastAPI app 入口 + lifespan
        ├── config.py                # Pydantic Settings 配置
        ├── models/                  # SQLAlchemy ORM 模型
        │   ├── __init__.py
        │   └── base.py              # DeclarativeBase 定义
        ├── schemas/                 # Pydantic v2 请求/响应模型
        │   ├── __init__.py
        │   └── ...
        ├── api/                     # 路由
        │   ├── __init__.py
        │   ├── router.py            # 总路由汇总
        │   └── v1/
        │       ├── __init__.py
        │       └── ...
        ├── services/                # 业务逻辑层
        │   ├── __init__.py
        │   └── ...
        ├── db.py                    # async engine + session 工厂
        └── deps.py                  # FastAPI Depends 依赖注入
```

### 1.2 SQLAlchemy 2.0 Async Engine + Session 使用模式

**依赖安装**：`pip install sqlalchemy[asyncio] asyncpg greenlet`

> asyncpg 是 PostgreSQL 的高性能异步驱动；greenlet 是 SQLAlchemy asyncio 扩展的必需依赖。

**db.py — 引擎与 Session 工厂**：

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from {service_name}.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,  # postgresql+asyncpg://user:pass@host:5432/dbname
    echo=settings.SQL_ECHO,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600,
)

async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # asyncio 环境必须设为 False
)
```

**deps.py — FastAPI 依赖注入**：

```python
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from {service_name}.db import async_session_factory

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# 类型别名，简化使用
DbSession = AsyncSession
```

**models/base.py — ORM Base 定义**：

```python
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import func, DateTime
from datetime import datetime

class Base(AsyncAttrs, DeclarativeBase):
    """所有 ORM 模型的基类。
    
    AsyncAttrs 提供 awaitable_attrs 用于懒加载关系属性。
    """
    pass

class TimestampMixin:
    """通用时间戳混入。"""
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )
```

**关键注意事项**：
- `expire_on_commit=False` 是 asyncio 环境的**强制要求**，否则 commit 后访问属性会触发隐式 IO 抛异常。
- `AsyncSession` 不是线程/协程安全的，每个并发任务需要独立 Session（FastAPI Depends 天然保证每请求一个 Session）。
- 关系属性使用 `selectinload()` 预加载，或设 `lazy="raise"` 避免隐式 IO。

### 1.3 Alembic 数据库迁移最佳实践

**依赖**：`pip install alembic`

**初始化**（在需要 DB 的服务目录下执行）：
```bash
cd services/memory
alembic init alembic
```

**alembic/env.py 核心配置**：

```python
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# 导入所有 ORM 模型以确保 metadata 包含所有表
from {service_name}.models.base import Base
from {service_name}.models import *  # noqa: F401, F403

target_metadata = Base.metadata

def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

**alembic.ini**：
```ini
[alembic]
script_location = alembic
sqlalchemy.url = postgresql+asyncpg://user:pass@localhost:5432/oh_enterprise
```

> **最佳实践**：`sqlalchemy.url` 从环境变量读取，不在配置文件中硬编码密码。可在 env.py 中用 `os.getenv("DATABASE_URL")` 覆盖。

**常用命令**：
```bash
alembic revision --autogenerate -m "create users table"
alembic upgrade head
alembic downgrade -1
alembic history
```

### 1.4 Pydantic v2 与 SQLAlchemy 模型映射

**推荐模式**：Pydantic schema 和 SQLAlchemy model 分离，通过 `model_validate` / `model_dump` 手动转换。

**schemas/user.py**：

```python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, EmailStr

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str

class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None

class UserPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # Pydantic v2 替代 orm_mode
    
    id: int
    name: str
    email: EmailStr
    role: str
    org_id: int
    created_at: datetime
```

**API 路由中使用**：

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()

@router.post("/users/", response_model=UserPublic)
async def create_user(
    data: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    # Pydantic → SQLAlchemy
    user = User(**data.model_dump(exclude={"password"}))
    user.hashed_password = hash_password(data.password)
    db.add(user)
    await db.flush()
    await db.refresh(user)
    return user  # FastAPI 自动用 UserPublic (from_attributes=True) 序列化
```

> **ConfigDict(from_attributes=True)** 是 Pydantic v2 的写法，替代 v1 的 `class Config: orm_mode = True`。

---

## 2. JWT 认证方案

### 2.1 库选型：PyJWT

**选型结论**：使用 **PyJWT**（而非 python-jose）。

| 对比项 | PyJWT | python-jose |
|--------|-------|-------------|
| 维护状态 | 活跃维护 | 维护不活跃，issues 堆积 |
| 安装大小 | 轻量 | 较重（依赖 ecdsa 等） |
| API 简洁性 | 简洁直观 | 接口较复杂 |
| RS256 支持 | `pip install PyJWT[crypto]` | 内置 |

**安装**：`pip install PyJWT[crypto]`

> `[crypto]` extra 安装 `cryptography`，支持 RS256/RS384/RS512/ES256 等非对称算法。

### 2.2 JWT 签发、验证、刷新流程

**shared/ohent_shared/security.py**：

```python
import jwt
from datetime import datetime, timedelta, timezone
from typing import Any

SECRET_KEY = "your-secret-key"  # 从环境变量读取
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 24  # 24 小时
REFRESH_TOKEN_EXPIRE_DAYS = 30

def create_access_token(
    data: dict[str, Any],
    expires_delta: timedelta | None = None,
) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(
    data: dict[str, Any],
    expires_delta: timedelta | None = None,
) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    )
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict[str, Any]:
    """验证并解码 JWT。失败时抛 jwt.PyJWTError。"""
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

**认证流程**：
1. **登录**：客户端发送 API Key → auth 服务验证 → 签发 access_token + refresh_token
2. **请求认证**：Gateway 中间件从 `Authorization: Bearer <token>` 提取 JWT → 验证签名和过期 → 注入 user 信息
3. **Token 刷新**：客户端用 refresh_token 换新的 access_token
4. **登出**：将 token 加入 Redis 黑名单

### 2.3 Token 黑名单（Redis）实现

**shared/ohent_shared/token_blacklist.py**：

```python
import redis.asyncio as redis
from {service_name}.config import settings

redis_client = redis.from_url(settings.REDIS_URL)

BLACKLIST_PREFIX = "token:blacklist:"

async def blacklist_token(token: str, jti: str, exp: int) -> None:
    """将 token 加入黑名单，TTL 设为 token 剩余过期时间。"""
    ttl = exp - int(datetime.now(timezone.utc).timestamp())
    if ttl > 0:
        await redis_client.set(f"{BLACKLIST_PREFIX}{jti}", "1", ex=ttl)

async def is_blacklisted(jti: str) -> bool:
    """检查 token 是否在黑名单中。"""
    result = await redis_client.get(f"{BLACKLIST_PREFIX}{jti}")
    return result is not None
```

**JWT Payload 中必须包含 `jti`（JWT ID）字段用于黑名单标识**：

```python
import uuid

def create_access_token(data: dict[str, Any], ...) -> str:
    to_encode = data.copy()
    to_encode.update({
        "exp": expire,
        "type": "access",
        "jti": str(uuid.uuid4()),  # 唯一标识，用于黑名单
    })
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

---

## 3. Redis 使用场景

### 3.1 连接方式（redis-py async）

**安装**：`pip install redis[hiredis]`

> `[hiredis]` 安装 C 解析器，显著提升解析性能。

**shared/ohent_shared/redis.py**：

```python
import redis.asyncio as aioredis
from {service_name}.config import settings

redis_client: aioredis.Redis = aioredis.from_url(
    settings.REDIS_URL,
    encoding="utf-8",
    decode_responses=True,
    max_connections=20,
)
```

**在 FastAPI lifespan 中管理连接**：

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from {service_name}.redis import redis_client

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown
    await redis_client.aclose()

app = FastAPI(lifespan=lifespan)
```

### 3.2 服务注册心跳

**scheduler 侧 — 服务注册表**：

```python
import json
import time
from {service_name}.redis import redis_client

SERVICE_REGISTRY_KEY = "service:registry"
HEARTBEAT_INTERVAL = 10  # 秒
HEARTBEAT_TTL = 30        # 超过此时间未心跳则视为离线

async def register_service(
    service_name: str,
    host: str,
    port: int,
) -> None:
    """服务启动时注册，并定期心跳续期。"""
    key = f"service:{service_name}:{host}:{port}"
    info = json.dumps({
        "name": service_name,
        "host": host,
        "port": port,
        "last_heartbeat": time.time(),
    })
    await redis_client.hset(SERVICE_REGISTRY_KEY, key, info)

async def heartbeat(service_name: str, host: str, port: int) -> None:
    """定期调用，续期 TTL。"""
    key = f"service:{service_name}:{host}:{port}"
    info = json.dumps({
        "name": service_name,
        "host": host,
        "port": port,
        "last_heartbeat": time.time(),
    })
    await redis_client.hset(SERVICE_REGISTRY_KEY, key, info)
    # 单独设 TTL（Hash field 级别用 Sorted Set 或单独 key 更好）
    await redis_client.set(f"service:heartbeat:{key}", "1", ex=HEARTBEAT_TTL)

async def get_healthy_services() -> list[dict]:
    """获取所有健康服务（心跳未过期）。"""
    all_services = await redis_client.hgetall(SERVICE_REGISTRY_KEY)
    healthy = []
    for key, info in all_services.items():
        heartbeat_key = f"service:heartbeat:{key}"
        if await redis_client.exists(heartbeat_key):
            healthy.append(json.loads(info))
    return healthy
```

> **生产建议**：更健壮的方案是使用 Redis **Sorted Set**（score 为时间戳），定期清理过期条目。当前方案足以满足骨架阶段需求。

### 3.3 Celery Broker 配置（使用 Redis）

**scheduler/pyproject.toml 依赖**：`celery[redis]`

**scheduler/celery_app.py**：

```python
from celery import Celery

celery_app = Celery(
    "ohent_scheduler",
    broker="redis://localhost:6379/1",       # 使用 DB 1 隔离
    backend="redis://localhost:6379/2",       # 结果存储使用 DB 2
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="Asia/Shanghai",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,            # 任务执行完才确认
    worker_prefetch_multiplier=1,   # 每次只预取 1 个任务
    result_expires=3600,            # 结果 1 小时过期
)

# 自动发现任务
celery_app.autodiscover_tasks(["scheduler.tasks"])
```

**启动 Celery Worker**：
```bash
celery -A scheduler.celery_app worker --loglevel=info --concurrency=4
```

**启动 Celery Beat（定时任务）**：
```bash
celery -A scheduler.celery_app beat --loglevel=info
```

**定义异步任务**：

```python
# scheduler/tasks/agent_tasks.py
from scheduler.celery_app import celery_app

@celery_app.task(bind=True, max_retries=3)
def create_agent_instance(self, agent_config: dict) -> dict:
    """异步创建 Agent 实例（通过 Host 服务 API）。"""
    import httpx
    try:
        resp = httpx.post(
            "http://localhost:8002/internal/agents",
            json=agent_config,
            timeout=30.0,
        )
        resp.raise_for_status()
        return resp.json()
    except Exception as exc:
        raise self.retry(exc=exc, countdown=5)
```

---

## 4. PostgreSQL 本地部署（Podman）

### 4.1 运行 PostgreSQL 容器

```powershell
# 创建数据持久化卷
podman volume create pgdata

# 运行 PostgreSQL 容器
podman run -d `
    --name oh-enterprise-db `
    -p 5432:5432 `
    -e POSTGRES_USER=ohent `
    -e POSTGRES_PASSWORD=ohent_secret `
    -e POSTGRES_DB=oh_enterprise `
    -v pgdata:/var/lib/postgresql/data `
    docker.io/postgres:16-alpine

# 验证
podman exec -it oh-enterprise-db psql -U ohent -d oh_enterprise -c "SELECT version();"
```

### 4.2 启用 ltree 扩展

```sql
-- 在 oh_enterprise 数据库中执行
CREATE EXTENSION IF NOT EXISTS ltree;

-- 验证
SELECT extname, extversion FROM pg_extension WHERE extname = 'ltree';
```

**创建初始化脚本** `scripts/init_db.sql`：

```sql
-- 启用扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS ltree;

-- 创建初始数据库（如需多库）
-- CREATE DATABASE oh_enterprise_memory;
-- CREATE DATABASE oh_enterprise_market;
```

**一键初始化**：
```powershell
podman exec -i oh-enterprise-db psql -U ohent -d oh_enterprise < scripts/init_db.sql
```

### 4.3 DATABASE_URL 格式

各服务连接字符串（用于 SQLAlchemy async engine）：

```
# 异步连接（SQLAlchemy async engine）
postgresql+asyncpg://ohent:ohent_secret@localhost:5432/oh_enterprise

# 同步连接（Alembic 迁移、Celery 等同步工具）
postgresql://ohent:ohent_secret@localhost:5432/oh_enterprise
```

> **注意**：Alembic env.py 中使用同步连接字符串即可，asyncpg 也支持同步方式通过 async_engine_from_config 运行迁移。

---

## 5. Supervisor 配置

### 5.1 安装

```powershell
# Windows 无原生 supervisor，替代方案：
# 方案 A: 使用 NSSM (Non-Sucking Service Manager)
# 方案 B: 使用 PowerShell 脚本管理进程
# 方案 C: WSL + supervisor（推荐开发阶段）
```

> **注意**：Supervisor 原生仅支持 Linux。Windows 开发环境建议使用 **NSSM** 或 **PowerShell 启动脚本**。以下配置为 Linux/WSL 环境下的标准 Supervisor 配置。

### 5.2 Supervisor 配置模板

**deploy/supervisor/ohent.conf**：

```ini
[program:ohent-gateway]
command=/path/to/.venv/bin/uvicorn gateway.main:app --host 0.0.0.0 --port 8000 --workers 2
directory=/path/to/OpenHarnessEnterprise/services/gateway
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/gateway-stdout.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=5
stderr_logfile=/var/log/ohent/gateway-stderr.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=5
environment=DATABASE_URL="postgresql+asyncpg://...",REDIS_URL="redis://localhost:6379/0"

[program:ohent-auth]
command=/path/to/.venv/bin/uvicorn auth.main:app --host 0.0.0.0 --port 8001 --workers 2
directory=/path/to/OpenHarnessEnterprise/services/auth
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/auth-stdout.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=5
stderr_logfile=/var/log/ohent/auth-stderr.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=5
environment=DATABASE_URL="postgresql+asyncpg://...",REDIS_URL="redis://localhost:6379/0",JWT_SECRET_KEY="..."

[program:ohent-scheduler]
command=/path/to/.venv/bin/uvicorn scheduler.main:app --host 0.0.0.0 --port 8003 --workers 1
directory=/path/to/OpenHarnessEnterprise/services/scheduler
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/scheduler-stdout.log
stderr_logfile=/var/log/ohent/scheduler-stderr.log

[program:ohent-host]
command=/path/to/.venv/bin/uvicorn host.main:app --host 0.0.0.0 --port 8002 --workers 1
directory=/path/to/OpenHarnessEnterprise/services/host
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/host-stdout.log
stderr_logfile=/var/log/ohent/host-stderr.log

[program:ohent-memory]
command=/path/to/.venv/bin/uvicorn memory.main:app --host 0.0.0.0 --port 8004 --workers 2
directory=/path/to/OpenHarnessEnterprise/services/memory
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/memory-stdout.log
stderr_logfile=/var/log/ohent/memory-stderr.log

[program:ohent-market]
command=/path/to/.venv/bin/uvicorn market.main:app --host 0.0.0.0 --port 8005 --workers 2
directory=/path/to/OpenHarnessEnterprise/services/market
autostart=true
autorestart=true
startsecs=5
stopwaitsecs=10
stdout_logfile=/var/log/ohent/market-stdout.log
stderr_logfile=/var/log/ohent/market-stderr.log

; Celery Worker
[program:ohent-celery-worker]
command=/path/to/.venv/bin/celery -A scheduler.celery_app worker --loglevel=info --concurrency=4
directory=/path/to/OpenHarnessEnterprise/services/scheduler
autostart=true
autorestart=true
startsecs=10
stopwaitsecs=30
stdout_logfile=/var/log/ohent/celery-worker-stdout.log
stderr_logfile=/var/log/ohent/celery-worker-stderr.log

; Celery Beat (定时任务)
[program:ohent-celery-beat]
command=/path/to/.venv/bin/celery -A scheduler.celery_app beat --loglevel=info
directory=/path/to/OpenHarnessEnterprise/services/scheduler
autostart=true
autorestart=true
startsecs=10
stdout_logfile=/var/log/ohent/celery-beat-stdout.log
stderr_logfile=/var/log/ohent/celery-beat-stderr.log

[group:ohent]
programs=ohent-gateway,ohent-auth,ohent-scheduler,ohent-host,ohent-memory,ohent-market,ohent-celery-worker,ohent-celery-beat
```

### 5.3 服务端口分配

| 服务 | 端口 | 说明 |
|------|------|------|
| Gateway | 8000 | 对外唯一入口 |
| Auth | 8001 | 权限管理 |
| Host | 8002 | Agent 宿主 |
| Scheduler | 8003 | 中央调度 |
| Memory | 8004 | 记忆服务 |
| Market | 8005 | 市场服务 |

### 5.4 Windows 替代方案 — PowerShell 启动脚本

**deploy/start_all.ps1**：

```powershell
$services = @(
    @{ Name = "gateway";   Port = 8000; Workers = 2 },
    @{ Name = "auth";      Port = 8001; Workers = 2 },
    @{ Name = "host";      Port = 8002; Workers = 1 },
    @{ Name = "scheduler"; Port = 8003; Workers = 1 },
    @{ Name = "memory";    Port = 8004; Workers = 2 },
    @{ Name = "market";    Port = 8005; Workers = 2 },
)

$basePath = "H:\PyProjects\Harness Agent\OpenHarnessEnterprise\services"
$venvPython = "H:\PyProjects\Harness Agent\OpenHarnessEnterprise\.venv\Scripts\python.exe"

foreach ($svc in $services) {
    $dir = Join-Path $basePath $svc.Name
    Write-Host "Starting $($svc.Name) on port $($svc.Port)..."
    Start-Process -FilePath $venvPython -ArgumentList "-m", "uvicorn", "$($svc.Name).main:app", "--host", "0.0.0.0", "--port", $svc.Port, "--workers", $svc.Workers -WorkingDirectory $dir -WindowStyle Minimized
}

Write-Host "All services started."
```

---

## 6. 包结构设计 — pyproject.toml 依赖清单

所有服务统一使用 `hatchling` 构建后端（与 SDK 和 OpenHarness 核心一致），Python >= 3.11。

### 6.1 shared (ohent_shared)

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-shared"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.7.0",
    "pydantic-settings>=2.0.0",
    "sqlalchemy>=2.0.30",
    "PyJWT>=2.8.0",
    "redis>=5.0.0",
    "httpx>=0.27.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/ohent_shared"]
```

### 6.2 gateway

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-gateway"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "httpx>=0.27.0",
    "PyJWT>=2.8.0",
    "redis>=5.0.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0", "httpx"]

[tool.hatch.build.targets.wheel]
packages = ["src/gateway"]
```

### 6.3 host

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-host"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "openharness-sdk",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "httpx>=0.27.0",
    "redis>=5.0.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/host"]
```

### 6.4 auth

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-auth"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "PyJWT>=2.8.0",
    "redis>=5.0.0",
    "passlib[bcrypt]>=1.7.4",
    "alembic>=1.13.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/auth"]
```

### 6.5 scheduler

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-scheduler"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "celery[redis]>=5.4.0",
    "redis>=5.0.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/scheduler"]
```

### 6.6 memory

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-memory"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/memory"]
```

### 6.7 market

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "ohent-market"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "ohent-shared",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "pytest-asyncio>=0.23.0"]

[tool.hatch.build.targets.wheel]
packages = ["src/market"]
```

---

## 7. 依赖版本汇总

| 包 | 推荐版本 | 用途 |
|----|---------|------|
| Python | >=3.11 | 运行时 |
| FastAPI | >=0.115.0 | Web 框架 |
| Uvicorn | >=0.30.0 | ASGI 服务器 |
| SQLAlchemy | >=2.0.30 | ORM |
| asyncpg | >=0.29.0 | PostgreSQL 异步驱动 |
| Alembic | >=1.13.0 | 数据库迁移 |
| Pydantic | >=2.7.0 | 数据验证 |
| pydantic-settings | >=2.0.0 | 配置管理 |
| PyJWT | >=2.8.0 | JWT 认证 |
| redis | >=5.0.0 | Redis 客户端 |
| Celery | >=5.4.0 | 异步任务队列 |
| passlib | >=1.7.4 | 密码哈希 |
| httpx | >=0.27.0 | 异步 HTTP 客户端 |
| openharness-sdk | >=0.1.0 | Agent 编排 SDK |

---

## 信息来源

- [SQLAlchemy 2.0 Asyncio 文档](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html) — 2026-04-14 抓取
- [FastAPI SQL Databases 教程](https://fastapi.tiangolo.com/tutorial/sql-databases/) — 2026-04-14 抓取
- [OpenHarness Enterprise require.md](OpenHarnessEnterprise/docs/require.md) — 项目内部文档
- [OpenHarness Enterprise api-protocol.md](OpenHarnessEnterprise/docs/api-protocol.md) — 项目内部文档
- [OpenHarnessEnterprise PRD.yaml](docs/PRD.yaml) — 项目需求文档
