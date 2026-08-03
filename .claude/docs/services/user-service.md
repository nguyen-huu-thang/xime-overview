# User Service (Java)

> Hợp nhất từ: `user-service-design.md`, `architecture-rules.md`, `trust-integration.md`, `crud-plan.md`, CLAUDE.md.
> Repo: https://github.com/nguyen-huu-thang/user-service · Local: `D:\code\xime\Base Platform\user`

## Vai trò — "Human Identity Domain Service"

Credential owner, account state, user profile. **KHÔNG** phải authentication gateway (JWT do Identity Service issue).

| User Service LÀM | User Service KHÔNG LÀM |
|---|---|
| verify credential | issue JWT |
| resolve account state | refresh/revoke token |
| cung cấp human identity data | quản lý platform authorization |
| lưu password hash, oauth, passkey, MFA | lưu session |
| | chứa social graph, post, order, cart |

## Credential ownership

Source of truth cho: password (**BCrypt/Argon2id** — KHÔNG MD5/SHA1/SHA256 trực tiếp), google/github oauth, passkey, MFA.

- Thiết kế bảng `user_credentials` (không phải `users.password` cứng) — một user có nhiều loại credential (PASSWORD, OAUTH, PASSKEY, MFA, API_KEY).
- **Password hash chưa nằm trong `User` aggregate** — thiết kế có chủ đích, sẽ tách `Credential` aggregate sau.

## Account State Model
`ACTIVE`, `LOCKED`, `DISABLED`, `BANNED`, `PENDING_VERIFICATION`. Identity gọi User để verify account state trước khi cho login.

## Token Version
`token_version` trên user record: JWT chứa nó; `token_version += 1` → toàn bộ refresh token cũ invalid (revoke hàng loạt, force logout, lock account).

## Identifier normalization (BẮT BUỘC)
`IdentifierNormalizer` phải gọi trước mọi thao tác lưu/tìm kiếm. Không bao giờ compare raw identifier.
```text
Email: TEST@MAIL.COM → test@mail.com   ·   Phone: +84 123 456 789 → 84123456789
Username: lowercase + trim + unicode normalization
```
Không normalize → routing sai, duplicate, shard mismatch, security bypass.

## Sharding & routing
- User gán shard **cố định** (immutable placement); User Service chỉ quản lý dữ liệu shard local (không phải global directory).
- Tìm user: `identifier → normalize → Search Service → resolve shard → direct routing → User shard` (không broadcast cluster).
- Eventual consistency với search index — User Service là source of truth, search index rebuild được nếu lỗi.

## Security model
- Credential API **internal only** — chỉ gRPC + mTLS, không expose public, không API key/shared secret.
- Verify caller (Identity) qua certificate, không tin client trực tiếp.
- **JWT verify locally** bằng public key cache từ Trust (không gọi Identity/Trust mỗi request).
- Tách **Auth data** (password, oauth, MFA, account state — nhạy cảm) khỏi **Profile data** (display name, avatar, bio, link, timezone). Credential + account state + auth metadata nên **cùng shard** → tăng locality.

## Tích hợp Trust Service (bootstrap)
```text
startup → GetPublicKeys (Trust) → cache local
        → RotateCertificate (nếu chưa có cert) → lưu cert + private key encrypted
        → sẵn sàng nhận request
```
- Cache nhiều public key (backward-compatible khi rotate); không fetch theo từng request; Trust down → vẫn verify bằng cached key.
- `infrastructure/integration/trust/VerificationKeyResolver.java` là nơi kết nối Trust. Cert + private key lưu ở `src/main/resources/security/trust/` và `runtime/security/`. Private key phải encrypt khi lưu, không hardcode path.

## Kiến trúc code

```text
api/            ← gRPC (nội bộ) + REST (ngoài)
application/    ← Use cases, DTOs, Input/Output ports (repository interface ở port/out/)
domain/         ← Business logic thuần (không phụ thuộc framework)
infrastructure/ ← JPA, BCrypt, JWT, gRPC client, Redis
config/         ← @Configuration beans
```

**Dependency rule**: `domain` không phụ thuộc ai → `application` chỉ phụ thuộc `domain` → `infrastructure`/`api` implement port. Vi phạm là lỗi kiến trúc nghiêm trọng.

**Luồng thêm tính năng**: (1) định nghĩa port `port/in|out/` → (2) use case `application/usecase/` → (3) implement output port `infrastructure/` → (4) expose `api/grpc|rest/` → (5) đăng ký bean `config/usecase/UseCaseConfig.java` (use case dùng `@Component` thì không cần đăng ký thủ công).

## Tech & Build
**Spring Boot 4.0.3 · Java 25 · PostgreSQL · Redis · gRPC · Flyway.** REST `8083`, gRPC `9092`.
```bash
./mvnw clean package
./mvnw spring-boot:run
./mvnw test -Dtest=TenTestClass
./mvnw generate-sources        # compile proto (src/main/proto/ → target/generated-sources/protobuf/)
```
Môi trường: PostgreSQL `localhost:5432` db `user_service` (user `thang`/`123456`), Redis, Trust Service (gRPC cấp key/cert khi bootstrap).

## Trạng thái CRUD
Domain + infrastructure (port out, JPA repo, adapter) **đã xong** mọi nhóm. Đang bổ sung application/REST cho các nhóm profile-phụ. Tham chiếu pattern: nhóm `profile` (đã hoàn chỉnh).

| Nhóm | Trạng thái | REST endpoint chính |
|---|---|---|
| Contact | ✅ | `GET/POST /api/v1/me/contacts`, `DELETE/{id}`, `PATCH/{id}/primary` |
| Address | ✅ | `GET/POST /api/v1/me/addresses`, `PATCH/DELETE /{id}` |
| Link | ✅ | `GET/POST /api/v1/me/links`, `PATCH/DELETE /{id}` |
| Interest | ✅ | `GET /api/v1/interests` (public), `GET/POST/DELETE /api/v1/me/interests` |
| UsernameHistory | ✅ (read-only) | `GET /api/v1/me/username-history` |
