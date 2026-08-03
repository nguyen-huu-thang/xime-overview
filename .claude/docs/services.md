# So sánh nhanh các service

> Bảng đối chiếu một trang. Chi tiết đầy đủ từng service: thư mục [services/](services/).

| Service | Ngôn ngữ | Vai trò | Port (HTTP/gRPC) | Schema | Repo interface ở | Trạng thái |
|---|---|---|---|---|---|---|
| [Trust](services/trust-service.md) | Java 25 | CA, mTLS, JWT signing key (control-plane) | 8081 / 9090 | JPA `ddl-auto` (no Flyway) | `domain/repository/` | Đang code |
| [Identity](services/identity-service.md) | Java 25 | Authentication, issue JWT, refresh+rotation | 8082 / 9091 | Flyway | `application/port/out/` | Đang code (JWT filter chưa xong) |
| [User](services/user-service.md) | Java 25 | Human Identity Domain — credential, profile | 8083 / 9092 | Flyway | `application/port/out/` | Domain+infra xong, đang làm app/REST |
| [Data](services/data-service.md) | Python + Xime | Object storage, capability permission, sharding | (FastAPI/gRPC) | Alembic | `application/port/outbound/` | **Hoàn chỉnh nhất** (tham chiếu) |
| [Notification](services/notification-service.md) | Python + Xime | Gửi email (SMS sau), stateless | (gRPC only) | — (no DB) | `application/port/outbound/` | Đang triển khai |
| [Payment](services/payment-service.md) | Java | Thanh toán: thuê bao B2B, ledger double-entry | 8087 | 9096 | `payment_service` | ✅ **v0 chạy thật** (2026-07-07), định tuyến callback theo cert LIVE 2026-07-29 |
| Organization | Java | Định danh công ty/doanh nghiệp, danh bạ workspace | 8094 | 9103 | `organization_service` | ✅ **v1 chạy thật** (2026-07-29), 4 app đã nối |
| [Application](services/application-service.md) | Java | Registry + lifecycle Subject APPLICATION - không JWT, định danh qua cert | - | Flyway | `application/port/out/` | Repo tạo 2026-06-15, bắt đầu code |
| [Agent](services/agent-service.md) | Java | Non-Human Identity Domain - BOT, AI_AGENT (credential owner, JWT qua Identity) | - | Flyway | `application/port/out/` | Repo tạo 2026-06-15, bắt đầu code |
| [Search](services/search-service.md) | (Python dự kiến) | Shard routing, bloom filter (không full-text) | — | RAM index | — | Dự kiến (có thiết kế) |
| Analytics | (Python dự kiến) | Phân tích dữ liệu | — | — | — | Dự kiến |
| [Lockbox](services/lockbox-service.md) | (chưa chốt) | Hạ tầng khóa mã hóa - registry (sharded) + core (master key/HSM) | — | (sharded) | — | Thiết kế xong (2026-06-16), chưa code |

## Điểm chung
- Giao tiếp nội bộ: **gRPC + mTLS**; JWT verify cục bộ bằng public key cache từ Trust.
- ID: **KSUID** (Trust 20B, còn lại 24B). String ID Base62 (service id chữ thường, shard id chữ HOA).
- Sharding bất biến identity-centric; shared-nothing (mỗi shard 1 DB).
- Java: transaction/security-critical · Python (Xime): IO-bound/async · Go: realtime (tương lai).

## Liên quan
- Cách các service phối hợp: [cross-cutting/xac-thuc-va-tich-hop.md](cross-cutting/xac-thuc-va-tich-hop.md)
- Mô hình Subject & định danh "hồn - xác" (nền của Application/Agent Service): [cross-cutting/mo-hinh-subject-va-dinh-danh.md](cross-cutting/mo-hinh-subject-va-dinh-danh.md)
- Chuẩn mã lỗi & exception toàn platform (dải mã, body chuẩn, triển khai Java/Python): [cross-cutting/quy-uoc-ma-loi-va-exception.md](cross-cutting/quy-uoc-ma-loi-va-exception.md)
- Mô hình mã hóa dữ liệu riêng tư & quản lý khóa (5 mức, envelope, cụm lockbox): [cross-cutting/mo-hinh-ma-hoa-va-quan-ly-khoa.md](cross-cutting/mo-hinh-ma-hoa-va-quan-ly-khoa.md)
- mTLS gRPC troubleshooting: [cross-cutting/mtls-grpc-troubleshooting.md](cross-cutting/mtls-grpc-troubleshooting.md)
- Quy tắc kiến trúc & convention: [rules/kien-truc-va-quy-uoc.md](rules/kien-truc-va-quy-uoc.md)
- Tổng quan platform: [tong-quan-he-thong.md](tong-quan-he-thong.md)
