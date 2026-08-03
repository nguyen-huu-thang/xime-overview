# Roadmap & Các điểm cân nhắc tương lai

> Hợp nhất từ: `future-considerations.md` (identity — bản đầy đủ 17 mục), `can-nhac-tuong-lai.md` (trust), `future-considerations.md` (user), `roadmap.md` (data, notification).
> **Nguyên tắc chung:** không triển khai kiến trúc phức tạp quá sớm. Ưu tiên hiện tại: (1) core services ổn định (2) architecture boundary rõ ràng (3) maintainability (4) observability cơ bản (5) scalability hợp lý (6) development speed. Chỉ mở rộng khi có **bottleneck thực tế / dữ liệu thực tế / nhu cầu production rõ ràng**.

## Hạng mục platform-wide (cân nhắc khi quy mô tăng)

| # | Hạng mục | Khi nào cần | Công nghệ tham khảo |
|---|---|---|---|
| 1 | **Service Governance** | shard/service nhiều lên | Consul, etcd, ZooKeeper, K8s service discovery |
| 2 | **Event-Driven Architecture** | cần async (vd `IdentityCreatedEvent → user/analytics/notification`) | Kafka, Pulsar, NATS, Redis Streams |
| 3 | **Distributed Workflow / Saga** | flow nhiều bước (payment→order→inventory→notification) | Temporal, Camunda, Conductor |
| 4 | **Outbox / Inbox Pattern** | tránh mất event khi DB tx & publish không atomic | — |
| 5 | **Observability Platform** | nhiều service | OpenTelemetry (đã có ở Identity), Prometheus, Grafana, Loki, Jaeger, Tempo |
| 6 | **Advanced Search** | cần full-text/ranking/recommendation/graph | Elasticsearch/OpenSearch, graph DB, CQRS projection |
| 7 | **Advanced Shard Management** | auto provisioning, hot shard, geo routing, split/merge/migration | — |
| 8 | **Multi-Region** | phục vụ nhiều khu vực (VN/SG/EU/US) | region-aware routing, geo replication, regional failover |
| 9 | **Security Hardening** | production | HSM (đã có abstraction `CaSignerConfig`), SPIFFE/SPIRE, Vault, PKI automation |
| 10 | **Rate Limiting / Abuse Protection** | public internet quy mô lớn | bot detection, IP reputation, anti-spam, brute-force |
| 11 | **Advanced Cache Strategy** | traffic tăng | distributed cache, invalidation, warming, hot key, stale handling |
| 12 | **CQRS / Materialized View** | query lớn (feed, analytics, recommendation) | — (không áp dụng sớm) |
| 13 | **Event Sourcing** | domain đặc biệt (audit, financial ledger, security history) | — (complexity rất cao) |
| 14 | **Kubernetes / Orchestration** | autoscaling, rolling deploy, self-healing | đánh giá kỹ operational complexity & cost |
| 15 | **Data Retention / Archival** | dữ liệu lớn | cold storage, archival, backup lifecycle |
| 16 | **Platform Operation Tooling** | nhiều shard/service | shard/tenant dashboard, deployment & migration tooling |
| 17 | **Message Delivery Guarantee** | nếu dùng event-driven | at-most/at-least/exactly-once tùy service |

## Theo service

- **Trust**: HSM integration cho CA key; advanced shard management; multi-region; SPIFFE/SPIRE.
- **Data**: Roadmap 7 phase (Core Storage → Data Platform); GĐ2 thêm `xime-cryptod` (C/C++ + OpenSSL qua UDS) cho mã hóa file; bảng `object_governance`, `object_replica`.
- **Notification**: thêm kênh SMS (sau email); xem `roadmap.md` của repo.
- **Search, Analytics**: chưa code.
- **Payment**: **đã code v0 và chạy thật** (thu tiền qua VietQR/SePay, callback ký HMAC + outbox, định tuyến theo app). Còn ở roadmap: ledger double-entry, PSP VNPay/Momo, escrow/settlement - Payment vẫn là ứng viên Saga cho phần đó.
- **Application & Agent**: **đã code đầy đủ và chạy thật** (đính chính 2026-07-29; dòng cũ ghi "chưa code" lỗi thời). Thiết kế nền chốt 2026-06 (xem [services/application-service.md](services/application-service.md), [services/agent-service.md](services/agent-service.md), [cross-cutting/mo-hinh-subject-va-dinh-danh.md](cross-cutting/mo-hinh-subject-va-dinh-danh.md)). Kéo theo: Trust thêm cột `owner_app_identity_id` + SAN entry mới trong cert; identity-service dọn enum subject type cũ (`SERVICE`/`SYSTEM`) và đổi tham chiếu `bot-service` → `agent-service`.
- **Xime Framework**: hoàn thiện WebSocket; starter Redis/Cache (planned); CLI scaffolding; testing utilities; bidi gRPC streaming; SDK client đa ngôn ngữ; socket credit-based flow control.

## Python — hướng tương lai
CPython **Free Threading** (bỏ dần GIL) sẽ giúp tác vụ CPU (JWT verify, serialization, hashing, ACL eval) hưởng lợi mà không cần đổi kiến trúc. Khi Python thực sự không đáp ứng, tách thành phần đặc thù sang Go/C++ mà không phá kiến trúc tổng thể.
