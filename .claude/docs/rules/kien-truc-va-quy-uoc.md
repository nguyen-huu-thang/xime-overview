# Quy tắc kiến trúc & convention (toàn hệ thống)

> Hợp nhất từ: `kien-truc.md` (trust), `architecture-decisions.md` (identity), `architecture-rules.md` (user), `hexagonal-ports.md` + `domain-coding-patterns.md` + `service-design.md` (data), `framework-first.md` (notification), `coding.md` + `transaction.md` + `interface-binding.md` (framework).

## 1. Hexagonal / DDD — chuẩn chung

Tầng (tên có thể khác nhẹ giữa Java/Python): **`api`** (adapter vào: gRPC/REST) · **`application`** (use case, port, DTO, mapper) · **`domain`** (logic thuần, không phụ thuộc framework) · **`infrastructure`** (adapter ra: JPA/SQLAlchemy, crypto, redis, gRPC client) · **`config`** (wiring).

**Dependency rule (bất biến):** `domain` ← không phụ thuộc ai · `application` ← chỉ phụ thuộc `domain` · `infrastructure`/`api` ← implement port của `application` · `config` ← wire tất cả. Vi phạm = lỗi kiến trúc nghiêm trọng.

**Luồng thêm tính năng:** định nghĩa port (`port/in|out`) → use case → implement output port (`infrastructure`) → expose adapter (`api/grpc|rest`) → đăng ký bean (`config`).

## 2. Đặt repository/port interface ở đâu? (có khác biệt giữa các service!)

| Service | Vị trí | Lý do |
|---|---|---|
| **Trust** (Java) | `domain/repository/` | DDD tactical truyền thống — domain giàu business logic, repository là collection abstraction của aggregate |
| **Identity, User** (Java) | `application/port/out/` | Hexagonal hiện đại — domain pure hơn, tránh "God Repository", dễ CQRS-lite |
| **Data, Notification** (Python) | `application/port/outbound/` | Như trên, dùng `Protocol` |

> Cả hai đều hợp lệ. Quy ước Hexagonal hiện đại (port ở application) được ưu tiên cho service mới. **Tránh "God Repository"** 100 method → tách port theo usecase: `LoadUserForLoginPort` ≠ `LoadUserProfilePort`, `LoadObjectPort`/`SaveObjectPort`/`BlobStoragePort`.

## 3. Naming convention (Java)

| Loại | Pattern | Ví dụ |
|---|---|---|
| JPA Entity | `*Entity` | `KeyEntity` |
| Repository impl | `*RepositoryImpl` / `Jpa*RepositoryAdapter` | `KeyRepositoryImpl` |
| JPA Repository | `Jpa*Repository` | `JpaKeyRepository` |
| Use Case | `*UseCase` | `InitKeyUseCase` |
| Domain Service | `*DomainService` | `KeyValidationDomainService` |
| gRPC Service | `*GrpcService` | `KeyAdminGrpcService` |

## 4. Quy tắc DI Xime (Python) — xem chi tiết [../xime-framework.md](../xime-framework.md)

- **Constructor injection only**, **không annotation** (`@service`/`@repository`/`@inject`) — vị trí thư mục quyết định vai trò.
- Interface dùng **`Protocol`** (không `ABC`), bind tường minh trong `config/dependency.py`. Nhiều candidate không binding → **startup fail**.
- Packages excluded khỏi scan: `domain`, `dto`, `entity`, `vo`, `port`, `mapper`, `constant(s)`, `exception`.
- **Thiếu type hint = class thường, không phải lỗi** (opt-in DI). Đăng ký thủ công class trong package excluded: `register()` (tự inject) / `configure()` (factory method).
- **Transaction tường minh**: `async with self.transaction():` — không `@transactional`/AOP.
- **Fail fast**: circular dep / missing impl / ambiguous binding / missing type hint → fail ngay startup.
- **Framework-first**: ưu tiên tối đa tính năng có sẵn của Xime trước khi tự viết.

## 5. Domain coding patterns (rút từ Java, áp dụng cho Python)

- **Immutable domain model**: `@dataclass(frozen=True)`; state change → `return replace(self, field=new)`. Java tương đương: return instance mới.
- **KSUID**: ID là `bytes` (Trust 20B, còn lại 24B = 4B timestamp + random), epoch `1_400_000_000`. Sortable, unique, opaque. Không dùng `str`/`UUID`.
- **Identifier normalization**: bắt buộc trước mọi routing/lookup (NFKC + lowercase email/username; bỏ ký tự không phải số ở phone). Deterministic: `normalize(normalize(x)) == normalize(x)`.
- **Tách Entity ↔ Domain model**: Entity (ORM, biết DB) ≠ Domain (pure, không import `infrastructure/persistence/sqlalchemy`); Mapper chuyển đổi (mapper **excluded** khỏi DI, gọi trực tiếp). Java dùng **MapStruct**.
- **Domain method nhận `datetime` làm tham số** (không gọi `datetime.now()` bên trong) → testable. Timestamp dùng `timezone.utc`.
- **gRPC handler**: chỉ map DTO→Command/Query rồi gọi UseCase; map exception→gRPC StatusCode tại handler (không leak domain exception); constructor injection, không decorator/global.

## 6. Bảo mật & tích hợp (nhắc lại — chi tiết [../cross-cutting/xac-thuc-va-tich-hop.md](../cross-cutting/xac-thuc-va-tich-hop.md))
- Internal/credential API chỉ qua **gRPC + mTLS**, không public REST/API key.
- **JWT verify cục bộ** bằng public key cache từ Trust; không gọi Trust/Identity mỗi request.
- **Trust là control-plane** — không tạo realtime dependency tới Trust.
- Schema migration: Java dùng **Flyway** (trừ Trust dùng JPA `ddl-auto: update`); Python dùng **Alembic**.
- **Không hardcode** shard count / routing logic / network address — thiết kế shard-aware & configurable.
