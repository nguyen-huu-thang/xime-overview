# Trust Service (Java)

> Hợp nhất từ: `trust-service.md` (×5 repo), `database.md`, `key-rotation.md`, `certificate-mtls.md`, `api-grpc.md`, `mtls-grpc-external-server.md`, `kien-truc.md`.
> Repo: https://github.com/nguyen-huu-thang/trust-service · Local: `D:\code\xime\Base Platform\Trust`

## Vai trò

**Cryptographic infrastructure & trust authority** của toàn platform. Là **control-plane**, KHÔNG nằm trong request flow runtime.

Chịu trách nhiệm:
- Generate & quản lý lifecycle JWT signing key
- Cấp & quản lý X.509 certificate cho service (mTLS)
- Làm **Root of Trust** nội bộ (internal CA)
- Cung cấp public key để các service verify JWT

KHÔNG: authenticate user · nằm trong request flow thông thường · tham gia business logic.

## Hai hệ thống độc lập

| | A. JWT Key System | B. Service Trust System (mTLS) |
|---|---|---|
| Mục đích | Authentication JWT của user | Service-to-service communication |
| Cơ chế | Identity sign bằng private key; service khác verify bằng public key (local) | Certificate-based identity, internal CA |
| Rotation | Time-based, không cần event trigger | Chủ động qua refresh token one-time + mTLS |
| Đặc điểm | Backward-compatible (key cũ vẫn verify token cũ) | — |

Hai hệ thống **hoàn toàn độc lập**: key rotation ≠ cert rotation (logic khác nhau).

## Nguyên tắc thiết kế cốt lõi

1. Tách biệt JWT key system và mTLS cert system
2. Không revoke JWT tức thì → dùng short TTL thay thế
3. Cert rotation phải an toàn: yêu cầu **cả** mTLS hiện tại **và** refresh token one-time
4. Refresh token **one-time** + **bound với cert hiện tại**
5. Trust Service không tham gia business logic, không nằm trong runtime request flow

## Key System — time-based, không có status

Key **không** có label CURRENT/NEXT/OLD. **Thời gian quyết định hành vi:**

```text
current key   = max(activate_at) với activate_at <= now
next key      = min(activate_at) với activate_at > now
next-next key = key sau next key
```

- **Rotation = mở rộng timeline** — thêm key mới với `activate_at` tương lai (KHÔNG update/activate/switch key).
- Tham số: `rotation_interval_seconds` (khoảng giữa 2 lần SIGN), `preload_seconds` (window an toàn — trong đó phải có sẵn next-next key).
- Identity tự tính `currentKey = max(activate_at <= now)` và tự switch theo thời gian — không gọi API rotate.
- Verify service load mọi key chưa hết hạn, verify theo `kid`.

### Domain invariants (key chain) — phải nắm để không phá hệ thống

1. **Append-only**: `new.activate_at` strictly after `last.activate_at`
2. **No duplicate**: không 2 key cùng `activate_at` trong 1 cặp (signer, verifier)
3. **Rotation interval**: `expiresAt - activateAt >= policy.rotationIntervalSeconds`
4. **No past activation**: `activateAt` không ở quá khứ (cho phép clock skew 5 giây)

### Reset flow (khi compromise)
markDeleted toàn bộ keys → admin tạo K1 mới → scheduler tự tạo K2 & đảm bảo continuity.

## Certificate / mTLS (X.509)

- Tách **logic tạo cert** khỏi **cơ chế lưu khóa** (file/HSM/KMS). Abstraction bắt buộc: `CertificateAuthoritySigner` cung cấp CA cert + `sign(byte[])`, **không expose** `PrivateKey`.
- Flow tạo cert: **Build** `TBSCertificate` (issuer, subject, serial, validity, extensions) → **Sign** `DER(tbsCertificate)` qua signer → **Assemble** thành X509Certificate. (Build và Sign là 2 bước tách biệt.)
- Extensions cho mTLS: **SAN = `service_id`** (extract identity từ SAN, không dùng CN), KeyUsage `digitalSignature`/`keyEncipherment`, ExtendedKeyUsage `clientAuth`/`serverAuth`, BasicConstraints `CA=false`.
- DEV: load key từ `dev-keys/` sign trong JVM. PROD: gửi data tới HSM/KMS (key không rời thiết bị). Application không phân biệt — chỉ thay `CaSignerConfig`.

### Cert rotation flow
Service (dùng mTLS hiện tại) gửi refresh token → Trust verify (token tồn tại, `used_at IS NULL`, chưa hết hạn, `bound_kid == current cert`) → issue cert mới + refresh token mới → mark token used.

### Cert lifecycle
| Sự kiện | Timing |
|---|---|
| Rotation | ~100 ngày sau phát hành |
| Hết hạn (lifetime) | ~1 năm |
| Hard delete | giữ 5 năm rồi xóa (audit trail) |

## Scheduler (polling, không per-key)

Nguyên tắc: `scheduler_interval << preload_seconds`.

| Job | Trigger | Chức năng |
|---|---|---|
| `KeyContinuityJob` | ~1h | `EnsureKeyContinuityUseCase` — đảm bảo timeline không thiếu key |
| `KeyCleanupJob` | ~6h | Hard-delete keys hết hạn |
| `CertificateLifecycleJob` | ~1h | Kiểm tra & rotate cert |
| `CertificateCleanupJob` | ~6h | Hard-delete certs hết hạn |

## API gRPC

Hai nhóm: **External** (runtime service-to-service) và **Internal** (admin). Proto ở `src/main/proto/{external,internal}/`.

| Service (external) | RPC | Ghi chú |
|---|---|---|
| `KeyDistributionService` | `GetPublicKeys` | trả **tất cả** key chưa hết hạn để verify (cache, theo kid) |
| | `GetSigningKey` | trả key `activate_at <= now` gần nhất — **chỉ cho Identity Service** (kèm `private_key`) |
| `CertificateService` | `RotateCertificate` | gửi refresh token → cert mới + next refresh token |
| `TrustService` | `GetRootCertificate` | lấy Root CA (PEM) để bootstrap trust store; cache dài |
| `HealthService` | `CheckLiveness/Readiness`, `GetServiceInfo` | |

Java package: `vn.xime.trust.grpc.external.*`. **Không** expose public key ra Internet (internal gRPC only; có thể thêm hybrid JWKS sau).

**Khi nào gọi Trust**: bootstrap (cert, root CA), rotate cert khi gần hết hạn, periodic key refresh. **KHÔNG** gọi trong request flow thông thường.

### mTLS authentication 2 lớp (chống spoofing)
mTLS verify trust chain (Root CA) + extract `service_id` từ SAN → so sánh `cert.service_id == request.service_id` và `cert.shard_id == request.shard_id`.

## Database (PostgreSQL, BYTEA = KSUID 20 bytes, **không Flyway** — JPA `ddl-auto: update`)

Bảng: `services`, `shards`, `key_policies` (UNIQUE signer+verifier, CHECK signer≠verifier), `keys` (UNIQUE signer+verifier+activate_at, CHECK expires>activate, soft-delete), `key_access_logs`, `key_events` (audit, JSONB), `certificates` (UNIQUE service+issued_at), `cert_refresh_tokens` (token_hash, is_bootstrap, one-time), `cert_events`.

## Kiến trúc code (Hexagonal/DDD, 4 lớp)

```text
api/             ← Input adapters (gRPC external/internal, REST)
application/     ← Use cases, ports, DTOs, mappers
domain/          ← Models, factories, repositories (interfaces), domain services
infrastructure/  ← JPA, crypto, scheduler, security, JWK
```

**Khác biệt với Identity/User**: Trust đặt **repository interface ở `domain/repository/`** (DDD tactical truyền thống), vì domain giàu business logic. Identity/User đặt ở `application/port/out/`. Xem [rules/kien-truc-va-quy-uoc.md](../rules/kien-truc-va-quy-uoc.md).

- **Domain service** (`domain/service/`) là **Plain Java POJO** (không `@Service`/`@Component`): `KeyValidationDomainService`, `CertificateLifecycleService`, `KeyPolicyDomainService`.
- **MapStruct** cho mọi mapper (entity↔domain, domain↔DTO, DTO↔protobuf).

## Build & cấu hình

```bash
mvn clean package           # build + test
mvn spring-boot:run         # HTTP 8081, gRPC 9090
mvn test -Dtest=KeyValidationDomainServiceTest#testValidateNewKeyAppendOnlyRule
```

Cấu hình (`application.yml`): `spring.datasource.url` (PostgreSQL `key_service`), `key.encryption.secret` (Base64 AES mã hóa private key at rest), `trust.key.rotation.interval-ms`, `trust.cert.lifecycle.interval-ms`. Yêu cầu **Java 25**.

## mTLS cho external gRPC server (port 9090)
Chi tiết pitfall Spring gRPC 1.0.2 (vì sao Trust phải tự build `NettyServerBuilder` thay vì `@Bean ServerCredentials`): xem [cross-cutting/mtls-grpc-troubleshooting.md](../cross-cutting/mtls-grpc-troubleshooting.md). Trust load cert của chính nó từ DB qua `GetCertificatesUseCase` → wrap PEM → `TlsServerCredentials(ClientAuth.REQUIRE)`. Internal server (9091) giữ nguyên.
