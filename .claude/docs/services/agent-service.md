# Agent Service (Java)

> Thiết kế nền đã chốt **2026-06-12**. Repo tạo **2026-06-15**.
> GitHub: <https://github.com/nguyen-huu-thang/agent-service>
> Local: `D:\code\xime\Base Platform\agent`
> Tài liệu nền bắt buộc đọc kèm: [cross-cutting/mo-hinh-subject-va-dinh-danh.md](../cross-cutting/mo-hinh-subject-va-dinh-danh.md).

## Tên gọi

Tên **"Agent"** (tác nhân) được chốt thay cho "Bot Service" ban đầu: bao trùm software agent (bot), AI agent, và cả thực thể phần cứng + phần mềm (robot). Các comment/tài liệu cũ trong identity-service còn ghi `bot-service` - refactor dần thành `agent-service`.

## Vai trò - "Non-Human Identity Domain Service"

Quản lý các Subject **không phải con người**: `BOT` và `AI_AGENT`. Là phiên bản tương đương user-service (Human Identity Domain) cho thế giới phi-con-người: credential owner, subject state, metadata, lifecycle.

Ví dụ:

```text
BOT:       Moderation Bot · Notification Bot · Backup Bot · robot (phần cứng)
AI_AGENT:  AI Assistant · Research Agent · Data Analyst Agent
```

**Robot phần cứng xếp cùng loại**: robot luôn có phần mềm và phần mềm đó hoạt động như bot/AI bình thường - không tạo subject type riêng.

## LÀM / KHÔNG LÀM (mirror user-service)

| Agent Service LÀM | Agent Service KHÔNG LÀM |
|---|---|
| Verify credential của agent (Identity gọi qua gRPC + mTLS) | Issue JWT, refresh/revoke token |
| Lưu credential: API key hash (Argon2id/BCrypt, không lưu raw) | Quản lý platform authorization |
| Subject state + `token_version` (revoke hàng loạt khi lộ key) | Lưu session |
| Metadata + lifecycle của BOT / AI_AGENT | Chứa logic nghiệp vụ mà agent thực hiện |
| Sync subject info cho resource service | |

**Điểm khác biệt then chốt với application-service**: agent **CÓ credential và CÓ JWT**, nằm trong login flow của Identity giống user-service. Lý do: agent không phải lúc nào cũng sống trong mesh có cert (robot, bot chạy ngoài), số lượng có thể lớn và động (user tạo bot). Xem [mo-hinh-subject-va-dinh-danh.md](../cross-cutting/mo-hinh-subject-va-dinh-danh.md).

## Mô hình dữ liệu sơ bộ

Bảng `agents`:

```text
identity_id        BYTEA(24)  PK   - KSUID 24 byte
subject_type       BOT | AI_AGENT
name               vd "Moderation Bot"
description
status             vd ACTIVE / DISABLED (chi tiết chưa chốt)
owner_identity_id  ai sở hữu/quản lý agent này (human hoặc application)
tenant_id          default null
token_version      tăng → toàn bộ refresh token cũ invalid
created_at / updated_at
```

Kèm bảng `agent_credentials` riêng (pattern `user_credentials` của user-service): một agent nhiều loại credential, trước mắt `API_KEY`.

## Authentication flow (giống user-service)

```text
Agent → Identity Service (login bằng API key)
      → Identity resolve shard → gọi Agent Service verify credential (gRPC + mTLS)
      → Agent Service check credential hash + subject state
      → trả VerifiedIdentity { identity_id, subject_type: BOT|AI_AGENT, tenant_id }
      → Identity issue JWT (multi-audience, như mọi subject)
```

Resource service verify JWT cục bộ như với HUMAN - không có gì khác biệt ở phía consumer.

## Sharding

Như user-service: immutable placement — shard do placement-service cấp lúc tạo theo dung lượng & tải (KHÔNG băm `identity_id`), lưu cố định và route qua `shard_id` lưu sẵn / mang theo trong token. Tách riêng khỏi application-service vì scale khác hẳn: app chỉ vài chục/trăm bản ghi, agent có thể hàng triệu nếu mở cho user tạo bot.

## Kiến trúc dự kiến

Java 25 / Spring Boot 4, Hexagonal mirror user-service: `api` / `application` (port ở `application/port/out/`) / `domain` / `infrastructure` / `config`. Schema bằng Flyway. Credential API chỉ expose qua gRPC + mTLS nội bộ. Port HTTP/gRPC: chưa cấp.

## Điểm còn mở

- **Delegation** (agent hành động thay user - "AI Assistant của user X"): hướng là claim `act` (RFC 8693) trên JWT, chưa thiết kế.
- Provisioning device certificate cho robot phần cứng (nếu cần) qua Trust - để mở.
- Chi tiết state machine lifecycle.
