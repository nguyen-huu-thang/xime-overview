# Identity Service (Java)

> Hợp nhất từ: `identity-service-design.md`, `architecture-decisions.md`, `trust-service-proto.md`, CLAUDE.md, + bản context ở các repo khác.
> Repo: https://github.com/nguyen-huu-thang/identity-service · Local: `D:\code\xime\Base Platform\identity`

## Vai trò

**Authentication infrastructure** của toàn platform: authentication orchestration · JWT issuing (access + refresh) · refresh token handling & rotation · token revocation · identity verification · platform-level authorization.

KHÔNG phải: user database · profile service · credential database · business account service.

## Identity Model

Identity là **"authenticated subject abstraction"** — không đồng nghĩa user/profile/account. Có thể là human, bot, service, internal system, AI agent.

Identity chỉ chứa: `identity_id`, authentication metadata, token metadata, platform authorization metadata. Thông tin domain thực tế nằm ở **domain owner service**:

```text
identity_id → user-service     (human)
identity_id → bot-service      (bot)
identity_id → system-service   (service/machine)
```

**Credential ownership**: credential KHÔNG lưu trong Identity. Human → user-service (password, oauth, passkey, MFA); Bot → bot-service; Service identity → trust-service (certificate).

## Authentication Flow

```text
Client → Identity Service → Search Service (resolve user shard)
       → User Service shard (verify credential) → Identity Service (issue JWT)
```

Identity không tin tuyệt đối routing hint từ client — có thể validate & fallback search. Xem flow đầy đủ: [cross-cutting/xac-thuc-va-tich-hop.md](../cross-cutting/xac-thuc-va-tich-hop.md).

## Token Architecture

### Access Token
- Stateless JWT, asymmetric signed (**ES256**), short-lived **5–15 phút**
- Verify locally bằng public key — **KHÔNG gọi realtime** về identity-service
- Không realtime revoke (dùng TTL ngắn); multi-audience (một token per consuming service)

### Refresh Token
- Signed JWT, revocable, rotatable; dùng để issue access token mới
- **Family tracking**: mỗi session có `family_id`
- **Rotation**: dùng refresh_token_A → nhận B, A bị invalidated
- Blacklist chỉ lưu `jti`/`hash(token)` (không lưu raw), chỉ trong identity infrastructure (không broadcast)

### Token Version
User có `token_version`; JWT chứa `token_version`. Tăng version → toàn bộ refresh token cũ invalid (force logout / lock account).

### JWT Claims
```text
Access header:  { kid, alg: "ES256", typ: "JWT" }
Access claims:  { jti, sub, aud, iss, auth_time, iat, exp, token_version }
Refresh claims: { jti, sub, aud: "identity-service", family_id, iss, auth_time, iat, exp, token_version }
```

## Authorization Model

- **Platform Authorization** (trong identity): quyền cấp hạ tầng — `platform_admin`, `support`, `developer`, `operator`.
- **Application Authorization** (trong app services): business permission do từng app tự quản — identity không chứa business rule.

## Distributed & Routing

- Identity shard là **stateless authentication processing node**, KHÔNG phải credential owner → request vào bất kỳ shard nào, không cần sticky session.
- **Deterministic identifier routing**: identifier phải normalize trước khi hash (Email `TEST@MAIL.COM`→`test@mail.com`; Phone `+84 123 456 789`→`84123456789`; Username lowercase+trim+unicode).
- Identity giữ `shard_id → internal endpoint` mapping trong RAM cache (vd `VN02 → grpc://user-service-vn02`).

## Service-to-Service Security
mTLS + certificate-based auth; cert chứa `service_id`, `shard_id`. Payload cũng chứa `service_id + shard_id` để so với mTLS identity (chống spoofing).

## Quyết định kiến trúc

- **Repository interface ở `application/port/out/`** (Hexagonal hiện đại) — domain pure hơn, tránh "God Repository", dễ CQRS-lite (`LoadUserForLoginPort` ≠ `LoadUserProfilePort`). Mỗi port một trách nhiệm.
- **Stateless authentication** — JWT, không centralized session store.
- **Trust Service không runtime dependency** — cache key/cert, chạy được khi Trust down 1–2 tháng.
- **gRPC TLS**: Identity dùng `spring.grpc.server.ssl` + SSL Bundle (cách Spring chính thức hỗ trợ); Trust dùng custom Netty. Lý do: xem [mtls-grpc-troubleshooting.md](../cross-cutting/mtls-grpc-troubleshooting.md).

## Kiến trúc code

```text
api/            REST (port 8082) + gRPC server (port 9091)
application/    UseCases, DTOs, Ports (in/out)
domain/         Models, Factories — không biết persistence
integration/    gRPC clients: trust, user, search, oauth
infrastructure/ Persistence, Crypto, Redis, Scheduler
```

Proto client (`src/main/proto/dependency/`): user (login/registration), trust (key_distribution, certificate). Compiled bởi `protobuf-maven-plugin`. APIs `exposed/` chưa implement.

## Tech & Build

**Spring Boot 4.0.6 · Java 25 · PostgreSQL · gRPC · Redis · Flyway · MapStruct · Nimbus JOSE JWT · OpenTelemetry.**

```powershell
.\mvnw.cmd clean install              # Windows
.\mvnw.cmd test "-Dtest=ClassName#methodName"
.\mvnw.cmd spring-boot:run            # REST 8082, gRPC 9091
```

Cấu hình: `spring.datasource.url` (`identity_service`), `key.encryption.secret`, `identity.bootstrap` (service-id `identity-service`, shard-id `I00000`, `path: ./runtime/security/bootstrap.txt`).

## Trạng thái hiện tại

- JWT filter **chưa implement** (đang dùng basic auth để test)
- mTLS infrastructure có sẵn (`TrustSslContextProvider`) nhưng chưa tích hợp đủ
- Search Service integration chưa implement; gRPC server (9091) chưa expose API
- Token policy hardcode (15' access, 30d refresh)
- Resilience4j (circuit breaker) có dependency nhưng chưa dùng

## Mục tiêu dài hạn
Reusable authentication platform · distributed authentication infrastructure · multi-application identity · machine + human authentication · SSO · zero-trust internal communication.
