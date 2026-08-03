# Application Service (Java)

> Thiết kế nền đã chốt **2026-06-12**. Repo tạo **2026-06-15**.
> GitHub: <https://github.com/nguyen-huu-thang/application-service>
> Local: `D:\code\xime\Base Platform\application`
> Tài liệu nền bắt buộc đọc kèm: [cross-cutting/mo-hinh-subject-va-dinh-danh.md](../cross-cutting/mo-hinh-subject-va-dinh-danh.md).

## Vai trò

Quản lý các Subject loại **APPLICATION** - registry, metadata, lifecycle, quyền hệ thống. Application là **thực thể hạng nhất** (first-class subject) ngang hàng User, Bot, AI Agent: sở hữu dữ liệu, được cấp quyền, audit được xuyên suốt hệ thống.

Ví dụ Application:

```text
Xime Social · Xime Chat · Xime Shopping · Xime AI · app phòng khám nha khoa
```

Bối cảnh: tầng Application Layer sẽ gồm nhiều app (social, ecommerce, SaaS bán theo dịch vụ...), mỗi app gồm **nhiều service con**; tất cả service con của một app dùng chung **một** app identity khi thao tác dữ liệu (vd với data-service), còn từng service con vẫn có cert riêng để giao tiếp.

## LÀM / KHÔNG LÀM

| Application Service LÀM | Application Service KHÔNG LÀM |
|---|---|
| Registry app: cấp `identity_id` 24 byte khi đăng ký | Authentication, issue/verify JWT |
| Metadata: `application_code`, name, description | Lưu credential (app không có credential - định danh qua cert) |
| Lifecycle / status + state version | Binding registry service→app (Trust nắm, qua cột `owner_app_identity_id`) |
| Source of truth cho System Permission của subject APPLICATION | Nằm trong login flow của Identity |
| Gửi tín hiệu sync (subject info + permission) cho resource service | Là runtime dependency của bất kỳ service nào |
| API quản trị (admin, con người thao tác) | Chứa business logic của từng app |

**Điểm khác biệt then chốt với user-service / agent-service**: APPLICATION **không dùng JWT, không có credential**. Service con của app được định danh bằng cert (Trust khắc `owner_app_identity_id` vào SAN); resource service resolve subject APPLICATION trực tiếp từ cert. Identity Service hoàn toàn không biết APPLICATION lúc runtime. Chi tiết và lý do: [mo-hinh-subject-va-dinh-danh.md](../cross-cutting/mo-hinh-subject-va-dinh-danh.md).

## Mô hình dữ liệu sơ bộ

Bảng `applications`:

```text
identity_id        BYTEA(24)  PK   - KSUID 24 byte, cấp khi đăng ký
application_code   string, unique  - Base62 chữ thường, vd "xime-social"
name               vd "Xime Social"
description
status             lifecycle (đề xuất: ACTIVE / SUSPENDED / DISABLED / RETIRED - chưa chốt)
state_version      tăng khi đổi trạng thái/quyền - để resource service invalidate cache
tenant_id          default null (hiện tại toàn bộ tenant = null)
created_at / updated_at
```

Kèm bảng quyền hệ thống cấp cho app (System Permission, vd `DATA_OBJECT_READ_ANY`...) - là source of truth, resource service giữ cache (data-service đã chừa sẵn bảng cache quyền cho app).

## Tích hợp

- **Trust Service**: registry `services` của Trust thêm cột `owner_app_identity_id` (nullable); cert của service con mang app id trong SAN. Thứ tự bootstrap (con người quản): đăng ký app ở đây lấy `identity_id` → đăng ký service ở Trust kèm app id → Trust cấp cert → service đọc cert tự biết mình thuộc app nào. Danh sách "app gồm những service nào" đọc từ Trust khi cần - **không giữ bản sao** (tránh dual-write).
- **Data Service / resource service khác**: nhận sync subject info (name, status) + permission qua cache; application-service gửi tín hiệu khi thay đổi. Cơ chế tín hiệu chi tiết **chưa chốt** (event push / periodic) - không chặn thiết kế.
- **Identity Service**: không liên quan runtime. Không login flow, không token, không verify.

## Kiến trúc dự kiến

Java 25 / Spring Boot 4, Hexagonal như user-service: `api` / `application` (port ở `application/port/out/`) / `domain` / `infrastructure` / `config`. Schema bằng Flyway. ID theo chuẩn KSUID 24 byte. Port HTTP/gRPC: chưa cấp.

## Điểm còn mở

- Chi tiết state machine lifecycle (danh sách status mới là đề xuất).
- Cơ chế tín hiệu sync với resource service.
- Binding gắn shard/tenant (chủ dự án đang cân nhắc; tenant = null toàn bộ hiện tại nên chưa cần).
