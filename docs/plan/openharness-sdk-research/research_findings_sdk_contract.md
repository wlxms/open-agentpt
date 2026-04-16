# OpenHarness SDK 契约与集成点研究报告

## 1. SDK 完整 API 表面 — `OrchestratorClient`

> 源文件: `OpenHarnessSDK/src/openharness_sdk/client.py`

### 构造函数

```python
class OrchestratorClient:
    def __init__(
        self,
        allowed_roots: list[str],
        default_model: str = "deepseek-chat",
        docker_image: str = "openharness:latest",
        podman_image: str = "openharness:latest",
        db_path: str | Path = ":memory:",
        use_docker: bool = True,
        runtime: str | None = None,   # "docker" | "podman" | "noop"
    ) -> None
```

内部构建三个核心对象:
- `self.store = SQLiteStore(db_path)` — 持久化层
- `self.driver` — 容器驱动 (DockerDriver / PodmanDriver / NoopDockerDriver)
- `self.instance_service = InstanceService(driver, store, allowed_roots)`
- `self.message_service = MessageService(driver, instance_service, default_model)`

### 公开方法一览

| 方法 | 签名 | 返回类型 | 说明 |
|------|------|----------|------|
| `create_instance` | `(req: InstanceCreateRequest, request_id: str \| None = None)` | `InstanceRecord` | 创建实例，幂等(request_id去重) |
| `list_instances` | `(host: str \| None = None, status: str \| None = None, include_deleted: bool = False)` | `list[InstanceRecord]` | 列表过滤 |
| `get_instance` | `(instance_id: str)` | `InstanceRecord` | 获取单实例，不存在抛异常 |
| `destroy_instance` | `(instance_id: str, remove_data: bool \| None = None)` | `DestroyResult` | 停止+删除容器 |
| `recover` | `()` | `RecoverResult` | 修正状态不一致 |
| `exec_instance` | `(instance_id: str, argv: list[str], env: dict \| None = None)` | `ExecResult` | 容器内执行命令 |
| `send_message` | `(instance_id: str, prompt: str, model: str \| None = None, max_turns: int = 1)` | `MessageResult` | 发送AI消息(通过容器内oh命令) |

---

## 2. 数据模型契约

> 源文件: `OpenHarnessSDK/src/openharness_sdk/contracts/models.py`

### `InstanceCreateRequest`
```python
class InstanceCreateRequest(BaseModel):
    name: str                # 实例名，正则 ^[a-z0-9-]{3,32}$
    host: str = "local"      # 主机标识
    template_id: str         # 模板ID
    workspace_root: str      # 工作区绝对路径
    seed: SeedConfig         # 种子配置
```

### `SeedConfig`
```python
class SeedConfig(BaseModel):
    mode: Literal["merge", "replace"] = "merge"
    template_dir: str        # 模板目录路径
    custom_dir: str | None = None
```

### `InstanceRecord`
```python
class InstanceRecord(BaseModel):
    instance_id: str
    container_id: str = ""
    name: str
    host: str
    status: InstanceStatus   # 状态枚举
    workspace_path: str
    seed_revision: str | None = None
    created_at: datetime
    updated_at: datetime
    last_error: ErrorEnvelope | None = None
    owner_id: str = "default"              # ★ 多租户字段
    image_fingerprint: str = ""
    managed_root_fingerprint: str = ""
    instance_signature: str = ""
```

### `InstanceStatus` (Literal Union)
```
"creating" | "running" | "seeding" | "ready" | "failed" | "stopping" | "stopped" | "deleting" | "deleted"
```

### `ErrorEnvelope`
```python
class ErrorEnvelope(BaseModel):
    code: str
    message: str
    retryable: bool = False
    details: dict = {}
```

### `SeedManifest`
```python
class SeedManifest(BaseModel):
    instance_id: str
    template_hash: str
    custom_hash: str | None = None
    mode: Literal["merge", "replace"]
    applied_files: list[str] = []
    skipped_files: list[str] = []
    conflicts: list[str] = []
    applied_at: datetime
```

### 结果模型
```python
class DestroyResult(BaseModel):
    deleted: bool
    data_removed: bool

class RecoverResult(BaseModel):
    reconciled: int
    stale_records: int

class ExecResult(BaseModel):
    exit_code: int
    stdout: str = ""
    stderr: str = ""

class MessageResult(BaseModel):
    instance_id: str
    request_id: str
    exit_code: int
    reply_text: str
    raw_stdout: str
    raw_stderr: str
    latency_ms: int
    model: str
    tokens: int | None = None
```

### 错误码枚举
```python
class ErrorCode(StrEnum):
    VALIDATION_ERROR = "VALIDATION_ERROR"
    DOCKER_CONNECT_ERROR = "DOCKER_CONNECT_ERROR"
    IMAGE_PULL_ERROR = "IMAGE_PULL_ERROR"
    SEED_PATH_UNSAFE = "SEED_PATH_UNSAFE"
    SEED_CONFLICT = "SEED_CONFLICT"
    INSTANCE_NOT_FOUND = "INSTANCE_NOT_FOUND"
    STATE_CONFLICT = "STATE_CONFLICT"
    SECURITY_VALIDATION_FAILED = "SECURITY_VALIDATION_FAILED"
```

---

## 3. Store Protocol 接口

> 源文件: `OpenHarnessSDK/src/openharness_sdk/store/store_base.py`

```python
class StoreBase(Protocol):
    # 生命周期
    def insert_creating(self, req: InstanceCreateRequest, request_id: str | None) -> InstanceRecord: ...
    def set_container_id(self, instance_id: str, container_id: str) -> None: ...
    def transition(self, instance_id: str, nxt: str) -> None: ...
    def mark_ready(self, instance_id: str, revision: str | None) -> None: ...
    def mark_failed(self, instance_id: str, error: ErrorEnvelope) -> None: ...
    def mark_deleted(self, instance_id: str) -> None: ...
    
    # 查询
    def get(self, instance_id: str) -> InstanceRecord | None: ...
    def list_all(self) -> list[InstanceRecord]: ...
    def exists_request(self, request_id: str) -> bool: ...
    def get_by_request(self, request_id: str) -> InstanceRecord: ...
    
    # 种子 & 安全
    def save_seed_manifest(self, manifest: SeedManifest) -> None: ...
    def save_security_evidence(self, instance_id: str, evidence: dict) -> None: ...
    def get_security_evidence(self, instance_id: str) -> dict | None: ...
```

共 **13 个方法**，全部基于 `Protocol`（structural subtyping），无需继承。

### SQLiteStore 表结构

| 表名 | 主键 | 关键字段 |
|------|------|----------|
| `instances` | `instance_id TEXT PK` | container_id, name, host, status, workspace_path, seed_revision, created_at, updated_at, last_error(TEXT/JSON), owner_id, image_fingerprint, managed_root_fingerprint, instance_signature, request_id(UNIQUE) |
| `seed_manifests` | `id INTEGER AUTOINCREMENT` | instance_id(FK), data(TEXT/JSON) |
| `security_evidence` | `instance_id TEXT PK(FK)` | data(TEXT/JSON) |

---

## 4. Driver Protocol 接口

> 源文件: `OpenHarnessSDK/src/openharness_sdk/runtime/driver_base.py`

```python
class DriverBase(Protocol):
    def create_container(self, req: InstanceCreateRequest, labels: dict[str, str]) -> str: ...
    def stop_container(self, container_id: str) -> None: ...
    def remove_container(self, container_id: str) -> None: ...
    def exec_in_container(self, container_id: str, argv: list[str], env: dict[str, str] | None = None) -> ExecResult: ...
    def list_managed_containers(self) -> list[dict]: ...
    def inspect_container(self, container_id: str) -> dict: ...
    def read_container_fingerprint(self, container_id: str) -> dict: ...
```

共 **7 个方法**，同样基于 `Protocol`。

### 三种实现

| Driver | 后端 | 特点 |
|--------|------|------|
| `DockerDriver` | docker-py SDK | 完整功能，自动pull镜像，volume挂载 |
| `PodmanDriver` | podman CLI subprocess | 完整功能，兼容远程podman |
| `NoopDockerDriver` | 内存字典 | 测试用，exec返回假数据 |

---

## 5. 状态机

> 源文件: `OpenHarnessSDK/src/openharness_sdk/core/state_machine.py`

### 状态转换规则

```
creating  ──→ [running, failed]
running   ──→ [seeding, stopping, deleting, failed]
seeding   ──→ [ready, failed]
ready     ──→ [stopping, failed, deleting]
stopping  ──→ [stopped, failed, deleting]
failed    ──→ [stopping, deleting]
stopped   ──→ [running, deleting]
deleting  ──→ [deleted, failed]
deleted   ──→ []  (终态)
```

非法转换抛出 `OrchestratorError(ErrorCode.STATE_CONFLICT)`。

### 典型创建流程
```
creating → running → seeding → ready
```

### 典型销毁流程
```
ready → stopping → stopped → deleting → deleted
```
或:
```
ready → deleting → deleted
```

---

## 6. 企业后台集成关键分析

### 6.1 多租户支持 (`owner_id`)

**是的，SDK 支持多租户。**

- `InstanceRecord.owner_id` 字段存在，默认值为 `"default"`
- SQLite 表中有 `owner_id TEXT NOT NULL DEFAULT 'default'` 列
- `create_instance` 时 `owner_id` 来自 `insert_creating`，当前硬编码为 default（从 `InstanceCreateRequest` 到 `InstanceRecord` 的映射中未传递 `owner_id`）
- **缺口**: `InstanceCreateRequest` 模型中 **没有** `owner_id` 字段，`SQLiteStore.insert_creating()` 也未写入调用者指定的 owner_id
- **结论**: 数据层已预留，但 API 层尚未暴露，企业集成需要扩展 `InstanceCreateRequest` 和 `insert_creating`

### 6.2 NoopDriver 行为

`NoopDockerDriver` 是纯内存的测试替身：
- `create_container` → 返回 `"noop-{uuid}"`, 存入内存字典
- `exec_in_container` → 对 `oh --help` 返回成功；其余命令返回 `exit_code=0, stdout="noop"`
- ** readiness probe 通过**: `run_runtime_readiness_probe` 执行 `oh --help` 恰好返回 exit_code=0
- **seed engine 正常运行**: `apply_seed` 操作的是宿主机文件系统(workspace_path)，不经过容器
- 适合: 单元测试、CI/CD 无 Docker 环境

### 6.3 SQLiteStore → PostgreSQL 可替换性

**高度可行，但需注意：**

1. **Protocol 接口**: `StoreBase` 是 `typing.Protocol`，SQLiteStore 是隐式实现（structural subtyping），任何实现 13 个方法的类都可替换
2. **SQL 差异**:
   - SQLite 用 `:memory:` 内存模式，PostgreSQL 需要连接字符串
   - SQLite 的 `AUTOINCREMENT` → PostgreSQL 的 `SERIAL`/`GENERATED ALWAYS AS IDENTITY`
   - JSON 序列化: SQLite 存 TEXT，PostgreSQL 可用原生 `JSONB`
   - 事务语义: SQLite 默认 autocommit 每条语句，PostgreSQL 需显式 commit
3. **改造要点**:
   - 实现 `StoreBase` Protocol 的 PostgreSQL 版本
   - 将 `last_error TEXT` 改为 `JSONB` 列
   - `request_id UNIQUE` 约束直接对应
   - 外键约束(`seed_manifests.instance_id`, `security_evidence.instance_id`)可直接保留
4. **OrchestratorClient 集成**: 当前 `__init__` 硬编码 `SQLiteStore`，需改为注入 store 参数

### 6.4 MessageService.send_message 工作流

```
send_message(instance_id, prompt, model, max_turns)
    │
    ├─ 1. 校验 model (必须有值)
    ├─ 2. 从环境变量读取 DS_API_KEY
    ├─ 3. 调用 instance_service.get_instance(instance_id) 获取容器ID
    ├─ 4. 构建 oh CLI 命令:
    │      oh --api-format openai \
    │         --base-url https://api.deepseek.com/v1 \
    │         --model {model} -k {api_key} -p {prompt} \
    │         --output-format text
    ├─ 5. 通过 driver.exec_in_container() 在容器内执行
    └─ 6. 返回 MessageResult (包含 reply_text, latency_ms, model 等)
```

**关键约束**:
- 硬编码 DeepSeek API (`base-url=https://api.deepseek.com/v1`)
- 依赖环境变量 `DS_API_KEY`
- `max_turns` 参数当前 **未被使用**（仅传入了 oh 命令但未体现多轮）
- 消息执行完全在容器内，企业后台若需替换为直接 HTTP 调用需修改 MessageService

### 6.5 安全策略

`core/policy.py` 的 `validate_create_request`:
- 实例名正则: `^[a-z0-9-]{3,32}$`
- workspace_root 必须为绝对路径
- seed 的 template_dir 和 custom_dir 必须在 allowed_roots 白名单内

---

## 7. 架构总览

```
OrchestratorClient
  ├── InstanceService
  │     ├── DriverBase (Docker/Podman/Noop)
  │     ├── StoreBase (SQLite)
  │     ├── core/policy.py (请求校验)
  │     └── seed/engine.py (文件种子)
  └── MessageService
        ├── DriverBase (容器内执行oh命令)
        └── InstanceService (获取容器信息)
```

## 8. Scheduler 扩展点

```python
class SchedulerBase(Protocol):
    def enqueue(self, fn, *args, **kwargs): ...
    def run_worker(self): ...
    def cancel(self, task_id: str) -> None: ...
    def get_task(self, task_id: str): ...
```

`InProcScheduler` 是同步内存实现，企业级需替换为 Redis/Celery 等异步调度器。

当前 Scheduler **未在 OrchestratorClient 中使用**（未集成），属于预留接口。
