# Xime Framework — tham chiếu đầy đủ

> Hợp nhất từ: README.md, CLAUDE.md, `tai-lieu-thiet-ke.md`, `gioi-thieu-framework.md`, `coding.md`, `transaction.md`, `interface-binding.md`, `config-discovery.md` (framework) + bản context ở data/notification.
> Repo: https://github.com/nguyen-huu-thang/xime-framework · PyPI: https://pypi.org/project/xime/ · Local: `D:\code\xime\xime framework` · License MIT · hoàn thiện ~95%.

## Là gì

Python backend framework mang trải nghiệm kiểu **Spring Boot** nhưng tôn trọng triết lý Python. **Không** thay thế FastAPI/SQLAlchemy/gRPC — nằm **bên trên**, cung cấp convention engine + dependency injection tự động + guardrail kiến trúc. Hỗ trợ Clean Architecture / DDD / Modular Monolith / Microservice (nhưng **không bắt buộc** cấu trúc thư mục — xem [apps/xime-shop-example.md](apps/xime-shop-example.md) dùng kiến trúc đa lớp).

```python
# Không cần wire thủ công — chỉ viết class, khai báo dependency qua type hint constructor:
class UserService:
    def __init__(self, repository: UserRepository, transaction: TransactionManager):
        self.repository = repository
        self.transaction = transaction
```

## Triết lý thiết kế

Convention Over Configuration · Constructor Injection · Type Hint Driven · Directory Driven · Fail Fast · Minimal Boilerplate · Explicit Architecture. *"Dependency nên được ẩn bởi framework, nhưng Transaction nên thể hiện rõ trong code nghiệp vụ."*

## Trình tự khởi động (startup pipeline)

```text
1. Load Framework Configuration (config/dependency.py)
2. Load Runtime Configuration (resources/application.yml)
3. Scan Packages
4. Resolve Type Hints
5. Build Dependency Graph
6. VALIDATE Graph — phát hiện cycle, missing implementation, ambiguous binding, missing type hint
7. Create Singletons
8. Start Adapters (FastAPI, gRPC, Socket...)
```
Sai là **fail ngay lúc startup** với thông báo rõ ràng, không phải lúc runtime khi user hit endpoint.

## Thành phần

### Core (`core/`) — không phụ thuộc adapter
`bootstrap/` (khởi động & điều phối) · `container/` (**DI tự viết**: quét package, phân giải type hint, dựng/validate graph, topological sort, detect cycle — chỉ dùng lib `dependency-injector` ở `registry.py` làm lớp lưu singleton, **không** dùng wiring `@inject`/`Provide` của lib) · `config/` (2 tầng YAML + env) · `context/` (request data qua `ContextVar`) · `contract/` (`@command`/`@stream`, `UploadStream`/`DownloadStream`, `ControllerScanner` — **dùng chung Socket + Code-First gRPC**) · `lifecycle/` (`PostConstruct`/`PreDestroy`) · `event/` (event bus, fire-and-forget async) · `security/` (`SecurityContext`, `AuthenticationManager`, `AuthorizationManager`) · `transaction/` (`TransactionManager`, `TransactionContext`) · `metadata/` · `exception/`.

### Adapters (`adapters/`) — mỗi adapter set up request `Context`
- `web/` — FastAPI; routing decorators `@get/@post/@ws`; middleware **pure-ASGI**; OpenAPI/Swagger; WebSocket (partial); streaming file (`files/`: Range download, chunked upload).
- `grpc/` — `grpc.aio`; **proto-first** (`configure_grpc_services`) **và code-first** cùng tồn tại; client SDK; TLS/mTLS động; interceptors. Xem [framework/code-first-grpc.md](framework/code-first-grpc.md).
- `socket/` — Unix Domain Socket RPC (Linux); frame protocol; peer auth (`SO_PEERCRED`). Xem [framework/socket-adapter.md](framework/socket-adapter.md).
- `mqtt/` — **MQTT** pub/sub (`@subscribe`) + RPC over MQTT v5 (`@rpc`); `MqttPublisher`; auto-reconnect; extra `xime[mqtt]` (aiomqtt).

### Starters (`starters/`) — tùy chọn, kiểu `spring-boot-starter-*`
`sqlalchemy` ✅ (AsyncSession, `SqlAlchemyTransactionManager`) · `jwt` ✅ (HS/RSA/EC/EdDSA, middleware, ép `audience`/`issuer`) · `scheduler` ✅ (APScheduler v4, cron/interval) · `cache` ✅ (Protocol `CacheService`) · `redis` ✅ (backend cache) · `storage` ✅ (`StorageService`) · `localfs` ✅ (backend filesystem) · `s3` ✅ (backend S3/MinIO, extra `xime[s3]`).

### Testing (`testing/`) · CLI (`cli/`)

## Dependency Injection — quy tắc

- **Constructor injection only** — không `@inject`/`@autowired`/`@service`/`@repository`/field injection.
- **Vị trí thư mục quyết định vai trò** component (vd `application/usecase/`, `infrastructure/repository/`).
- **Điều kiện một class được vào DI khi scan**: không phải `ABC`/`Protocol`; **mọi** tham số constructor có type hint; không thuộc package excluded.
- **Thiếu type hint = class thường, KHÔNG phải lỗi** (thiết kế có chủ đích): `scan()` SKIP class nếu thiếu hint ở bất kỳ param nào; type hint là tín hiệu opt-in cho DI.
- **Packages excluded** khỏi DI scan: `domain`, `dto`, `entity`, `vo`, `constant`, `exception` (và `port`, `mapper`).
- **`__init__.py`**: không có `__all__` → scan toàn bộ; có `__all__` → chỉ scan class export.

### Đăng ký thủ công (cho class trong package excluded, tương đương `@Bean`)
- `dependency.register(IdFactory, IdService, ...)` — framework tự inject (mọi param phải có type hint).
- `dependency.configure(DomainConfig)` — config class có factory method: mỗi public method có return type → một singleton; param method được inject; config class **stateless** (không có constructor param).

### Interface Binding
Interface dùng **`Protocol`** (không `ABC`, không `@abstractmethod`). Implementation là class thường, không bắt buộc kế thừa. Bind tường minh:
```python
dependency.bind({ UserRepository: JpaUserRepository, CacheService: RedisCacheService })
```
Startup validate implementation thỏa Protocol (thiếu method → fail). Nhiều candidate không binding → **fail**.

### Scope
`Singleton` (mặc định) · `Factory` (instance mới mỗi lần) · tương lai `Request`/`Session` (qua ContextVar).

### Fail Fast — ví dụ lỗi startup
`Missing Type Hint (Class/Parameter)` · `No Implementation Found (Interface)` · `Multiple Implementations Found (Candidates...)` · `Circular dependency detected: A → B → C → A`.

## Transaction — tường minh, không AOP

Không `@transactional`/proxy ẩn. Dùng async context manager:
```python
async def create_user(self):
    async with self.transaction():          # BEGIN
        await self.repository.save_user()
        await self.repository.save_profile()
    # COMMIT (hoặc ROLLBACK nếu Exception)
```
`TransactionManager` là abstract của framework; `SqlAlchemyTransactionManager` (starter) là impl. Ưu điểm: không magic, dễ đọc/debug (stack trace đúng luồng), tương thích async. Tương lai: `transaction.read_only()`, `transaction(isolation="SERIALIZABLE")`.

## Cấu hình 2 tầng
- **Framework config** (Developer, Python): `config/dependency.py`, `config/routing.py`, `config/security.py`, `config/grpc.py`, `config/socket.py` — DI scanning, binding, routing, lifecycle.
- **Runtime config** (Operator, YAML): `resources/application.yml`, `application-{dev,prod,test}.yml` — host, port, secrets, database, redis.

## Quick start
```python
# app/main.py
from xime import Application
from xime.adapters.web import WebAdapter
from xime.adapters.grpc import GrpcAdapter

app = Application()
app.use(WebAdapter())                              # server_id="default", port từ application.yml
app.use(WebAdapter("admin", "127.0.0.1", 8081))   # multi-server: host/port riêng
app.use(GrpcAdapter())                             # REST + gRPC đồng thời
app.run()
```
```python
# app/api/rest/user_controller.py — class-based controller (DI singleton, method map route)
class UserController:
    prefix = "/users"
    def __init__(self, use_case: GetUserUseCase) -> None: self._use_case = use_case
    @get("/{user_id}", response_model=UserResponse)
    async def get_user(self, user_id: int) -> UserResponse: return await self._use_case.execute(user_id)
```
```bash
pip install xime    # hoặc: pip install -e "D:\code\xime\xime framework"
python app/main.py
```

## Tính năng nổi bật
Constructor/Directory-driven DI · Interface binding validate startup · **Multi-server** (nhiều WebAdapter/GrpcAdapter/SocketAdapter/MqttAdapter mỗi process, mỗi cái `server_id`/`client_id` riêng) · Event bus · Request/Security context · Two-layer config · Class-based controllers · **Code-First gRPC** (sinh `.proto` từ DTO Python, ổn định field-number qua lock file) · **Socket Adapter** (UDS IPC) · **MQTT Adapter** (pub/sub + RPC over MQTT v5) · **File Storage** (`StorageService` local/S3-MinIO + streaming HTTP Range/chunked) · Initialization order (`dependency.order([A,B,C])`). Đã phát hành **0.5.0** (1050+ test).

## Đóng góp / tài liệu
Solo project, cần cộng đồng: hoàn thiện WebSocket, CLI scaffolding, testing utilities. Docs: `docs/en/{getting-started,architecture,core-concepts,configuration,routing,transaction,grpc-codefirst,socket-adapter,starters,testing,contributing}.md`.
