# Data Service (Python + Xime)

> Hợp nhất từ: `data-service.md`, `thiet-ke-data-service.md`, `thiet-ke-database.md`, `luong-xac-thuc.md`, `lua-chon-python.md`, `lua-chon-cong-nghe.md`, `domain-coding-patterns.md`, CLAUDE.md.
> Repo: https://github.com/nguyen-huu-thang/data-service · Local: `D:\code\xime\Base Platform\data`
> **Là service Python hoàn chỉnh nhất — dùng làm tham chiếu pattern cho mọi service Python khác.**

## Vai trò

**Distributed Data Infrastructure** của toàn platform: object storage, object permission, distributed data routing, blob storage abstraction, ownership & lifecycle management.

KHÔNG chứa business domain — không biết "đây là ảnh đại diện / ảnh sản phẩm / ảnh bài viết". Đó là việc của Application Service. Chỉ quan tâm: Ownership, Permission, Routing, Storage, Lifecycle, Audit.

## DataObject — thực thể trung tâm

Mọi dữ liệu (image, video, document, dataset, AI artifact, attachment, backup...) đều là `DataObject`. Trường quan trọng: `object_id`, `owner_identity_id`, `owner_subject_type`, `tenant_id`, `shard_id` (immutable), `object_type`, `visibility`, `status`, `current_version_id`, `storage_pointer` (KHÔNG lưu binary trong DB).

## Subject Model

Mọi dữ liệu thuộc về một **Subject** = thực thể có thể sở hữu dữ liệu / được cấp quyền:

```text
HUMAN · BOT · AI_AGENT · APPLICATION       → mỗi Subject có: identity_id, subject_type, name
```

**Application là Subject hợp lệ** (vd `Xime Social`, `Xime Chat`) — có thể sở hữu dữ liệu & được cấp quyền hệ thống. Phân biệt rõ:

| Subject (thực thể kinh doanh) | Service (thực thể runtime) |
|---|---|
| HUMAN/BOT/AI_AGENT/APPLICATION | post-service, feed-service, identity-service... |
| có `identity_id` | có `service_id`, `certificate`, `shard_id` (Trust quản lý) |
| **sở hữu dữ liệu** | **không** sở hữu — chỉ đại diện tiến trình, hành động thay mặt Subject |

## Authorization — nhiều lớp, capability-based

```text
Request → Subject Resolution → System Permission → Object Permission → Ownership → Visibility → ALLOW/DENY
```

1. **System Permission** (cấp cho Subject) - đúng 5 mã theo khuôn `SERVICE_RESOURCE_ACTION_SCOPE` (chốt 2026-07-22): `DATA_OBJECT_READ_ANY`, `DATA_OBJECT_WRITE_ANY`, `DATA_OBJECT_DELETE_ANY`, `DATA_OBJECT_SHARE_ANY`, `DATA_PERMISSION_GRANT_ANY` - vd Moderation System có `DATA_OBJECT_DELETE_ANY` xóa được mà không cần là Owner. Danh sách này phải bằng đúng `ObjectCapability` của data-service và `PermissionCode` của application-service.
2. **Object Permission** (ACL từng object): capability `READ/WRITE/DELETE/SHARE/DOWNLOAD`; role `OWNER/EDITOR/VIEWER`.
3. **Ownership Rule**: Owner mặc định có toàn quyền trên dữ liệu của mình.
4. **Visibility**: `PRIVATE` (Owner/được cấp quyền) · `INTERNAL` (theo policy app) · `PUBLIC` (không cần authz).

## Routing & Sharding (immutable)

```text
object mới → placement (dung lượng & tải) → shard_id → lưu cùng bản ghi   (cố định mãi mãi)
đọc/route → shard_id lấy từ bản ghi / tham chiếu (địa chỉ đi kèm dữ liệu, không băm từ identity_id)
```
Mọi dữ liệu của một identity nằm cùng một data shard. Shared-nothing: mỗi shard có **PostgreSQL riêng + Blob Storage riêng + metadata riêng**.

## Storage — tách Metadata và Blob

- **Metadata** → PostgreSQL (object, permission, version, audit).
- **Blob** → Local disk / storage adapter (filesystem, MinIO, S3, Ceph) — DB chỉ lưu `storage_pointer`, serve binary qua FastAPI.

## Object Versioning (immutable)
Object có `current_version` + `historical_versions`. Mỗi version: `content_hash`, `content_size`, `mime_type`, `created_by`, `created_at`. Version **không sửa, chỉ tạo mới**.

## Object Reference Model
Business Service **không lưu Blob**, chỉ lưu `object_id` (vd Post → `image_object_id`). Data Service là chủ sở hữu thật của dữ liệu. Bảng `object_reference` theo dõi Application nào đang dùng object.

## Tích hợp & sync (cache, không gọi realtime)
- **JWT verify**: startup `GetVerificationKeys` (Trust) → `VerificationKeyCache`; runtime verify theo `kid`; cache miss → background sync + retry. Verify claims `exp/nbf/iss/aud` (vd `aud = data-service`).
- **Subject cache** (`subject_cache`): cache `identity_id/subject_type/name` cho audit/logging — source of truth ở Subject Service; sync qua event hoặc periodic.
- **Permission cache** (`subject_permission`): cache system permission — source of truth ở application/subject service; cập nhật khi permission đổi.
- **mTLS**: bootstrap nhận cert từ Trust → cache → dùng cho outbound gRPC tới Identity/User/Application/Search.

Token hierarchy: `Trust (key pair) → Identity (sign JWT) → Subject (identity_id, permissions) → Data (verify, resolve, evaluate, execute)`.

## Database (MVP — 9 bảng, binary(24) ID)
`data_object` (trung tâm), `object_version` (immutable), `object_permission` (ACL), `subject_cache`, `subject_permission`, `object_reference`, `object_tag`, `object_share`, `object_audit`. Tương lai: `object_governance` (legal_hold, retention, moderation_lock), `object_replica` (PRIMARY/SECONDARY/ARCHIVE).
Index quan trọng: `data_object(owner_identity_id[, status|object_type])`, `(shard_id)`, `(status)`; `object_permission(subject_identity_id, object_id)`.

## Data Lifecycle
`ACTIVE → ARCHIVED → SOFT_DELETED → PURGED`. Mọi hành động (READ/DOWNLOAD/UPDATE/DELETE/SHARE/RESTORE) đều audit được (`actor_identity_id`, `actor_subject_type`, `action`, `object_id`, `timestamp`).

## Tại sao Python (không Go)?
Workload Data Service là **I/O-bound** (DB/disk/network/permission eval), không CPU-bound — chênh lệch latency Python vs Go không đáng kể (vd 125ms vs 124ms khi PostgreSQL chiếm 120ms). Python thắng bằng **developer productivity**; Xime Framework xóa boilerplate wiring. Scale đến từ **sharding** chứ không từ ngôn ngữ.

**Hai giai đoạn công nghệ**:
- **GĐ1** — Python thuần + Xime: hoàn thiện kiến trúc, làm tài liệu tham chiếu cộng đồng, không mã hóa file, không tối ưu CPU.
- **GĐ2** — thêm `xime-cryptod` (C/C++) qua Unix Domain Socket cho encrypt/decrypt/stream file (tận dụng OpenSSL AES-NI/AVX, worker pool, tách process tăng ổn định). Data Service vẫn lo routing/permission/metadata.

## Kiến trúc code & quy tắc (Xime, Hexagonal)

- Tầng: `api/` · `application/` · `domain/` · `infrastructure/` · `integration/`. **Port interface ở `application/port/outbound/`** (Protocol), implement ở `infrastructure/persistence/repository/`, bind tường minh trong `config/dependency.py`. Tránh "God Repository" → tách port theo usecase (`LoadObjectPort`, `SaveObjectPort`, `BlobStoragePort`...).
- Domain pattern (xem [rules/kien-truc-va-quy-uoc.md](../rules/kien-truc-va-quy-uoc.md)): `@dataclass(frozen=True)` immutable, state change `replace(self, ...)`, KSUID 24 bytes (`bytes`), normalize identifier, tách Entity (SQLAlchemy) ↔ Domain model (pure) qua Mapper, domain method nhận `datetime` làm tham số (testable), không import gì từ `infrastructure/persistence/sqlalchemy`.

## Chạy
```bash
pip install -e "D:\code\xime\xime framework"
pip install -e .
python app/main.py
```
Migration dùng Alembic (`alembic.ini`, `migrations/`).
