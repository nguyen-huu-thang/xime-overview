# Xime Shop Example (ứng dụng ví dụ — Python + Xime)

> Hợp nhất từ: `tong-quan-du-an.md`, `kien-truc-da-lop.md`, `framework-issues/README.md`, CLAUDE.md.
> Repo: https://github.com/nguyen-huu-thang/xime-shop-example · Local: `D:\code\PYTHON\xime\shop`
> **Không thuộc Base Platform** — là bản tham chiếu/kiểm thử cho Xime Framework.

## Mục tiêu
Migrate shop backend từ **PHP/Symfony** (`D:\code\PHP\shop-backend`) sang **Python** dùng Xime. Mục đích: **chứng minh framework chạy được ứng dụng thật** + làm tài liệu mẫu cho cộng đồng. KHÔNG phải sản phẩm thương mại, không migrate dữ liệu cũ — được tự do chọn giải pháp sạch; khi code PHP & schema mâu thuẫn thì chốt phương án hợp lý và ghi vào `quyet-dinh-thiet-ke.md`.

## Điểm đặc biệt: kiến trúc ĐA LỚP (không Hexagonal)

Giữ nguyên kiến trúc N-tier của bản gốc — **KHÔNG** dùng Hexagonal mà Xime khuyến nghị (Xime *khuyến khích* chứ không *bắt buộc* cấu trúc thư mục).

```text
Controller → Service → Repository → Entity
```

Lý do: trung thành với bản gốc (map 1-1 dễ đối chiếu); shop CRUD-nặng, ít domain logic phức tạp; service phụ thuộc **trực tiếp class Repository cụ thể** (giống PHP), không cần lớp Protocol trung gian → **không cần `dependency.bind()`** cho repository.

| Hexagonal (Xime khuyến nghị) | Đa lớp (shop) |
|---|---|
| `api/rest/external/` | `controller/` |
| `application/usecase/` + `service/` | `service/` (gộp) |
| `port/inbound`, `port/outbound` (Protocol) | **không có** — phụ thuộc class cụ thể |
| `infrastructure/persistence/repository/` | `repository/` |
| `domain/` | `entity/` |
| `application/mapper/` | DTO tự map |

> Vẫn tuân thủ DI của Xime: không annotation, constructor injection, type-hint driven, directory-driven (`controller/`/`service/`/`repository/` khai trong `config/dependency.py`), transaction tường minh, fail fast. **Khi nào vẫn dùng Protocol+bind?** Chỉ khi cần thay impl theo môi trường (vd FileStorage local vs S3).

## Stack PHP → Python
Symfony→Xime · Doctrine→SQLAlchemy async (`xime.starters.sqlalchemy`) · Lcobucci JWT→PyJWT (`xime.starters.jwt`) · Symfony Validator→Pydantic · Symfony DI→Xime DI · `EntityManagerInterface`→`AsyncSession` + `TransactionManager` · EventListener→Web middleware · `nelmio_api_doc`→OpenAPI/Swagger (FastAPI).

## Luồng request điển hình
```text
HTTP → [Security Middleware: decode JWT, set current user vào context]
     → Controller (kiểm tra phân quyền AuthorizationService, parse DTO Pydantic)
     → Service (async with transaction(): business logic)
     → Repository (SQLAlchemy) → Entity (ORM)
     → Controller map entity → response DTO (Pydantic) → JSON
```

## Phạm vi tính năng (theo 18 controller PHP)
Auth (login/logout/refresh/đổi mật khẩu), User CRUD, Group + member + permission, Permission (hệ thống + cá nhân), Category (phân cấp cha-con), Product (+ thuộc tính/tùy chọn), Cart, Order (+ detail), Coupon, Review, Wishlist, File upload, Notification, Search sản phẩm. Nguồn: 25 Doctrine entity, `giải thích cơ sở dữ liệu.txt`.

## Trạng thái
**Giai đoạn lập kế hoạch** — kế hoạch 10 phase (`.claude/docs/ke-hoach/`, phase 0–9), chưa code triển khai. Bước tiếp: Phase 0 (scaffold).

## Framework issues (phản hồi cho tác giả Xime — `.claude/framework-issues/`)
| # | Mức độ | Vấn đề |
|---|---|---|
| 1 | Thấp (cosmetic) | `TestApplication` gây cảnh báo pytest collection (Phase 0) |
| 2 | Trung bình (có workaround) | Thiếu hook public cho exception handler / middleware (Phase 1) |
