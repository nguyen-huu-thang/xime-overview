# Kho thông tin tổng hợp — Xime Platform

> **Điểm vào duy nhất.** Toàn bộ thông tin của các dự án (đọc từ `.claude/`, `CLAUDE.md`, `README` của mọi repo) đã được tổng hợp và **khử trùng lặp** vào đây. Mỗi chủ đề chỉ viết một lần.
>
> Tổng hợp ngày **2026-06-12** từ: `Base Platform/{Trust, identity, user, data, notification, payment}`, `xime framework`, `D:\code\PYTHON\xime\shop`.

## Cách dùng
Thao tác với dữ liệu tại **một nơi** thay vì tìm rải rác từng repo. Bắt đầu từ [Tổng quan](docs/tong-quan-he-thong.md), rồi đi sâu vào service/chủ đề cần thiết. Khi nội dung trong các repo con thay đổi, cập nhật lại file tương ứng ở đây.

## Mục lục

### Tổng quan & danh sách
- [docs/tong-quan-he-thong.md](docs/tong-quan-he-thong.md) — Platform: 2 tầng, chiến lược ngôn ngữ, ID/KSUID, sharding, multi-tenant, routing, resilience
- [docs/danh-sach-repository.md](docs/danh-sach-repository.md) — Bảng repo (local + GitHub + PyPI) & ghi chú
- [docs/services.md](docs/services.md) — Bảng so sánh nhanh tất cả service (một trang)

### Service (chi tiết) — [docs/services/](docs/services/)
- [trust-service.md](docs/services/trust-service.md) — CA, mTLS, JWT signing key; key chain time-based; cert lifecycle; DB schema; API gRPC
- [identity-service.md](docs/services/identity-service.md) — Auth, JWT (access/refresh), token version, authorization, trạng thái
- [user-service.md](docs/services/user-service.md) — Human Identity Domain, credential, account state, normalization, CRUD status
- [data-service.md](docs/services/data-service.md) — Subject model, authorization nhiều lớp, storage, versioning, DB 9 bảng, lý do chọn Python
- [notification-service.md](docs/services/notification-service.md) — Gửi email stateless qua gRPC, domain/port/infra, proto
- [search-service.md](docs/services/search-service.md) — Shard routing, bloom filter, login flow (dự kiến)
- [payment-service.md](docs/services/payment-service.md) — Chưa thiết kế (scaffold)
- [application-service.md](docs/services/application-service.md) - Registry + lifecycle Subject APPLICATION; không JWT, định danh qua cert (chốt thiết kế nền 2026-06, chưa code)
- [agent-service.md](docs/services/agent-service.md) - Non-Human Identity Domain: BOT, AI_AGENT; credential owner, JWT qua Identity (chốt thiết kế nền 2026-06, chưa code)

### Cross-cutting — [docs/cross-cutting/](docs/cross-cutting/)
- [xac-thuc-va-tich-hop.md](docs/cross-cutting/xac-thuc-va-tich-hop.md) — Login flow, JWT verify cục bộ, mTLS, bootstrap Trust, token hierarchy, cache sync
- [mtls-grpc-troubleshooting.md](docs/cross-cutting/mtls-grpc-troubleshooting.md) — Postmortem Spring gRPC 1.0.2 (vì sao `@Bean ServerCredentials` bị bỏ qua) + cách fix & verify
- [mo-hinh-subject-va-dinh-danh.md](docs/cross-cutting/mo-hinh-subject-va-dinh-danh.md) - Mô hình Subject "hồn - xác": SubjectType chuẩn, app identity trong cert, luật resolve subject, bootstrap (chốt 2026-06)
- [quy-uoc-ma-loi-va-exception.md](docs/cross-cutting/quy-uoc-ma-loi-va-exception.md) - Chuẩn mã lỗi & exception toàn platform: bản đồ dải mã, body `{errorKey, code, message}`, mô hình exception theo type, phụ lục triển khai Java & Python

### Xime Framework — [docs/xime-framework.md](docs/xime-framework.md)
- Tham chiếu đầy đủ: triết lý, startup pipeline, core/adapters/starters, DI, transaction, config 2 tầng
- [framework/code-first-grpc.md](docs/framework/code-first-grpc.md) — Sinh `.proto` từ DTO Python; field-number stability; CLI
- [framework/socket-adapter.md](docs/framework/socket-adapter.md) — Unix Domain Socket IPC; protocol; session; bảo mật Linux

### Quy tắc, roadmap, ứng dụng
- [docs/rules/kien-truc-va-quy-uoc.md](docs/rules/kien-truc-va-quy-uoc.md) — Hexagonal, vị trí port (khác nhau giữa service), naming, DI Xime, domain patterns
- [docs/roadmap.md](docs/roadmap.md) — 17 hạng mục platform-wide + theo service (chưa áp dụng sớm)
- [docs/apps/xime-shop-example.md](docs/apps/xime-shop-example.md) — Ứng dụng ví dụ (kiến trúc đa lớp, migrate PHP→Python)

## Bản đồ trùng lặp đã hợp nhất

| Nội dung | Xuất hiện ở (trước khi gộp) | Giờ nằm ở |
|---|---|---|
| Kiến trúc hệ thống / platform overview | Trust, Identity, User, Data, Notification (5 bản) | `docs/tong-quan-he-thong.md` |
| Trust Service (thiết kế/context) | Trust, Identity, User, Data, Notification (5 bản) | `docs/services/trust-service.md` |
| Identity Service (thiết kế/context) | Trust, Identity, User, Data, Notification | `docs/services/identity-service.md` |
| User Service (thiết kế/context) | Identity, User | `docs/services/user-service.md` |
| Luồng xác thực / auth flow | Data, Notification + rải ở Trust/Identity | `docs/cross-cutting/xac-thuc-va-tich-hop.md` |
| mTLS gRPC troubleshooting | Identity, User, Trust, top-level guide | `docs/cross-cutting/mtls-grpc-troubleshooting.md` |
| Trust integration (bootstrap key/cert) | User, Data, Notification | `docs/cross-cutting/xac-thuc-va-tich-hop.md` |
| Future considerations / roadmap | Trust, Identity, User, Data, Notification | `docs/roadmap.md` |
| Quy tắc kiến trúc / DI / domain patterns | mọi repo (rules/) | `docs/rules/kien-truc-va-quy-uoc.md` |
| Xime Framework (context) | Notification + framework | `docs/xime-framework.md` + `docs/framework/` |

> Các tài liệu **chỉ có ở một repo** (vd DB schema riêng, directory structure, implementation plan/phase, mapping PHP→Python) đã được tóm tắt vào file service tương ứng; nếu cần chi tiết từng dòng, xem `.claude/` của chính repo đó.
