# Tổng quan hệ thống — Xime Platform

> Nguồn tổng hợp (khử trùng lặp) từ `platform-overview` / `system-architecture` / `kien-truc-he-thong` / `thiet-ke-he-thong` của Trust, Identity, User, Data, Notification.
> 4 bản này gần như giống hệt nhau — tài liệu này là bản hợp nhất duy nhất.

## Mục tiêu

Một **Backend Platform dạng Microservice** dùng chung cho nhiều ứng dụng:

- Nền tảng backend tái sử dụng, tránh lặp lại các chức năng phổ biến
- Horizontal scaling, kiến trúc phân tán quy mô lớn
- Multi-tenant / multi-application; tách dữ liệu & hạ tầng cho từng tenant khi cần

Định hướng xuyên suốt: `distributed-first` · `shared-nothing` · `horizontally scalable` · `service-oriented` · `tenant-aware` · `failure-tolerant` · `loosely coupled`.

Mục tiêu dài hạn: phục vụ social systems, realtime systems, ecommerce, SaaS platforms, AI infrastructure, large-scale distributed applications.

---

## Hai tầng kiến trúc

### 1. Base Platform — service lõi dùng chung (độc lập nghiệp vụ)

> ⚠ **Bảng dưới cập nhật 2026-07-31.** Bản trước ghi payment "chưa thiết kế", application và agent "chưa code", Application Layer "chưa bắt đầu code" - tất cả đã lỗi thời từ lâu. Nguyên tắc rút ra và **áp cho mọi bảng trạng thái trong repo**: đối chiếu thư mục `src/` trước khi tin.

| Service | Vai trò | Ngôn ngữ | Trạng thái |
|---|---|---|---|
| `trust-service` | Trust/cryptographic infrastructure — CA, mTLS, JWT signing key | Java | **Chạy thật.** Port 8081/9090 |
| `identity-service` | Authentication infrastructure — JWT, refresh token | Java | **Chạy thật.** Passkey + OTP email E2E thật. Port 8082/9091 |
| `user-service` | Human Identity Domain Service — credential, profile, account state | Java | **Chạy thật.** Port 8084/9093 |
| `data-service` | Distributed data infrastructure — object storage, capability permission | Python | **Chạy thật.** Service Python hoàn chỉnh nhất, là **bản tham chiếu** cho mọi app Python. Port 8086/9095 |
| `notification-service` | Gửi thông báo ra ngoài (email trước, SMS sau) — stateless | Python | **Chạy thật**, đã gửi email thật. Port 9097 |
| `payment-service` | Thanh toán: thuê bao B2B, ledger double-entry, định tuyến callback theo cert | Java | **Chạy thật v0.** Vòng tiền khép kín E2E 2026-07-07; định tuyến callback theo app LIVE 2026-07-29. Port 8087/9096 |
| `application-service` | Registry + lifecycle cho Subject APPLICATION - định danh qua cert, không JWT | Java | **Chạy thật.** 140 file, mô hình "hồn - xác" LIVE 2026-07-26. Port 8083/9092 |
| `agent-service` | Non-Human Identity Domain - BOT, AI_AGENT (credential owner) | Java | **Đã code đầy đủ** (203 file, test pass), **chưa app nào gọi tới**. Port 8085/9094 |
| `organization-service` | Định danh công ty/doanh nghiệp dùng chung: pháp nhân, thành viên tầng platform, danh bạ workspace | Java | **Chạy thật từ 2026-07-29.** 55 test + E2E mTLS 10/10. Port 8094/9103 |
| `search-service` / `user-locator` | Distributed lookup, shard routing | (dự kiến Python) | Hoãn - 1 shard thì vị trí là hằng số |
| `placement-service` | Chọn shard cho bản ghi mới | (dự kiến) | Hoãn - 1 shard thì luôn chọn shard đó |
| `analytics-service` | Phân tích dữ liệu | (dự kiến Python) | Dự kiến |

> Chi tiết từng service: xem [services/](services/). Tên cũ của `trust-service` là `key-service` (vẫn còn trong code cũ, refactor dần).
>
> **Bảng port đầy đủ và là nguồn sự thật duy nhất:** [`đăng kí mạng.md`](../../../đăng%20kí%20mạng.md) ở root workspace.

### 2. Application Layer — logic nghiệp vụ từng ứng dụng (dùng lại Base Platform)

> ⚠ **Cập nhật 2026-07-31.** Bản trước ghi "chưa bắt đầu code" và liệt kê định hướng social/ecommerce/AI. Thực tế đã đi hướng khác hẳn và đã có code chạy thật.

**Hiện trạng:** 13 app ở `Application Layer/`, trong đó **8 app đã code + E2E thật** (nha-tro, spa, gara-oto, gym, admin, dai-ly-phan-phoi, sua-chua-dien-may, trung-tam-day-hoc), 6 app **đã dựng khung hạ tầng** chờ nghiệp vụ, cộng `saas-foundation/template` và `portal`.

**Hướng sản phẩm thật:** SaaS dọc theo ngành bán cho hộ kinh doanh, chọn theo hai tiêu chí **né pháp lý + dung lượng thấp**. Social network, sàn thương mại điện tử và AI **không nằm trong giai đoạn này** - lý do đầy đủ ở [`ranh-gioi-phap-ly-cua-san-pham.md`](../../../thiết%20kế%20chi%20tiết%20trong%20giai%20đoạn%20khởi%20nghiệp/ranh-gioi-phap-ly-cua-san-pham.md) và [`.claude/tham-khao/04`](../../../.claude/tham-khao/04-tam-nhin-dai-han-mang-xa-hoi-san.md).

### 3. Service ngang — tầng thứ ba (mới 2026-07-30)

Tính năng chung của mọi ngành (chấm công, sổ thu chi, giao việc, CRM, kho): viết **một lần**, backend riêng, frontend nhúng vào app dọc qua package npm, **cách ly theo `org_id` chứ không theo `workspace_id`**. Năm repo ở `Service ngang/`, khung hạ tầng đã chạy thật, chưa có nghiệp vụ.

Vì sao không đặt ở Base (Base phải trung lập nghiệp vụ) và không đặt ở Application Layer (không bán riêng theo nghề): [`service-ngang.md`](../../../thiết%20kế%20chi%20tiết%20trong%20giai%20đoạn%20khởi%20nghiệp/service-ngang.md).

---

## Chiến lược ngôn ngữ

Mỗi service viết bằng ngôn ngữ phù hợp nhất với workload (xem lý do chi tiết trong [services/data-service.md](services/data-service.md) — phần "Tại sao Python thay vì Go"):

| Ngôn ngữ | Dùng cho | Service |
|---|---|---|
| **Java 25 / Spring Boot 4** | Transaction-heavy, security-critical, consistency-sensitive | trust, identity, user, payment |
| **Python / FastAPI + Xime Framework** | IO-bound, async, AI/data | data, notification, search, analytics |
| **Golang** (tương lai) | Realtime, streaming, video/media (phát trực tiếp, video call) | — |

Triết lý: *"Ngôn ngữ là công cụ; kiến trúc hệ thống mới quyết định tuổi thọ và khả năng mở rộng."* Khả năng scale đến từ **sharding/routing**, không phải từ việc đổi ngôn ngữ. Phần CPU-nặng (mã hóa file) sẽ tách sang tiến trình C/C++ (`xime-cryptod`) qua Unix Domain Socket khi cần, thay vì viết lại bằng Go.

---

## Thiết kế ID

### Binary ID (dùng trong DB) — KSUID

| Dùng cho | Cấu trúc | Kích thước |
|---|---|---|
| Trust Service | 4 bytes timestamp + 16 bytes random | **20 bytes** |
| Các service khác | 4 bytes timestamp + 20 bytes random | **24 bytes** |

- **KSUID epoch**: `1_400_000_000` (~ tháng 5/2014).
- Tính chất: **sortable** (mới hơn = lớn hơn khi so sánh bytes nhờ timestamp prefix), **unique** (random đủ dài), **opaque** (không encode thông tin nhạy cảm).
- Trong code dùng `bytes`, không dùng `str`/`UUID`.

### String ID (con người thao tác trực tiếp)

- Encoding **Base62**, độ dài 1–20 ký tự.
- **Service ID**: chữ thường + số — `user-service`, `identity-service`.
- **Tenant Service ID**: `user-service-001`, `user-service-companyabc`.
- **Shard ID**: chữ IN HOA + số — `VN01`, `EU02`, `A0B1C2`, `I00000`.

---

## Nguyên tắc Sharding

| Nguyên tắc | Mô tả |
|---|---|
| **Shared-nothing** | Mỗi shard có database riêng, không chia sẻ dữ liệu trực tiếp |
| **Immutable placement** | Dữ liệu gán cố định vào shard từ đầu, không migrate |
| **Direct routing** | Route trực tiếp từ `service_id + shard_id + network_location`, không global lookup |
| **Shard growth** | `data growth → shard growth` |
| **Đơn vị đặt shard là TỔ CHỨC** | Toàn bộ dữ liệu của một `org_id` nằm cùng một shard, vì **giữa các shard không JOIN được** |
| **Mọi bảng nghiệp vụ tự khai shard** | `shard_id` + `org_id` khai **tường minh** trên từng entity, không mixin kế thừa |
| **Cấm suy shard bằng `hash(id)`** | Mô hình cũ `identity_id → hash → partition → shard` đã gỡ khỏi tài liệu (2026-06-21) **và khỏi code** (2026-07-29). Có test canh không cho thêm lại |

> **Luật đầy đủ, bắt buộc mọi repo kể cả repo chưa code:** [`.claude/rules/01-song-song-hoa-va-shard.md`](../../../.claude/rules/01-song-song-hoa-va-shard.md). Tám chỗ dễ vi phạm + bốn test canh. Áp cho 19 codebase ngày 2026-07-31, kết quả ở [`.claude/docs/ket-qua-dong-bo-luat-shard-2026-07-31.md`](../../../.claude/docs/ket-qua-dong-bo-luat-shard-2026-07-31.md).
>
> ⚠ `hash` **vẫn đúng** ở một chỗ duy nhất: tầng tìm kiếm tự phân mảnh index của nó (mục "Search / Discovery" dưới). Đó không phải hàm placement.

Mỗi shard có 2 ngưỡng:
- **Soft Limit** → khuyến nghị tạo shard mới
- **Hard Limit** → shard chuyển read-only, không ghi thêm

Mỗi service instance ≈ một **shard**. HA: mỗi shard có thể chạy cluster nhỏ (read workers + failover nội bộ), backup bắt buộc lưu ngoài cluster chính.

---

## Định vị & truy xuất dữ liệu

### Direct Routing (đã biết vị trí)
Mỗi bản ghi lưu `service_id`, `shard_id`, `network_location` → route trực tiếp, latency thấp.

### Search / Discovery (chưa biết vị trí)
Dùng **Search Service** với deterministic routing:

```text
identifier → normalize → hash → partition → search shard → resolve target shard
```

Search Service **chỉ là locator/index, KHÔNG phải source of truth** — vị trí thật của dữ liệu là bất biến. Search index có thể stale (eventual consistency chấp nhận được); nếu lỗi có thể rebuild. Chi tiết: [services/search-service.md](services/search-service.md).

---

## Multi-tenant

```text
tenant = null    → service mặc định của platform
tenant = <id>    → service riêng của tenant (vẫn shard được)
```

Tenant **không phải Subject**, không sở hữu dữ liệu — chỉ là **isolation boundary**.

---

## Mô hình resilience (rất quan trọng)

**Trust Service là control-plane, KHÔNG nằm trong request flow runtime.**

- Một instance logic chính, không yêu cầu realtime dependency từ service khác.
- Nếu Trust Service down **1–2 tháng**, các service khác vẫn hoạt động bình thường nhờ **cached key/cert**:
  - JWT verify vẫn chạy (cached public key)
  - Service-to-service mTLS vẫn chạy (cert đã có)
  - Identity vẫn sign JWT bằng cached signing key
- Chỉ bị ảnh hưởng khi: cần **rotate key**, **issue cert mới**, hoặc **rotate cert**.

→ Giảm coupling, tăng resilience, tránh single-point runtime dependency. Đây là nguyên tắc kiến trúc cốt lõi của toàn platform. Chi tiết tích hợp: [cross-cutting/xac-thuc-va-tich-hop.md](cross-cutting/xac-thuc-va-tich-hop.md).
