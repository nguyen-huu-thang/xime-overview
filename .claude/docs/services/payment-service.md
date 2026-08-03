# Payment Service (Java — chưa thiết kế)

> Local: `D:\code\xime\Base Platform\payment`. Chưa có repo GitHub trong danh sách.

## Trạng thái

**Mới là scaffold Spring Boot** — chủ dự án **chưa kịp viết** thiết kế. Thư mục chỉ có `pom.xml`, `src/`, `HELP.md`, `mvnw` (không có `.claude/`, `CLAUDE.md`, `README`).

## Định vị trong platform (theo tài liệu tổng quan)

- Thuộc **Base Platform**, viết bằng **Java** (nhóm transaction-heavy, security-critical, consistency-sensitive — cùng nhóm với trust/identity/user).
- Vai trò: **thanh toán**. Là ứng viên cho distributed workflow / Saga trong tương lai (flow `payment → order → inventory → notification`) — xem [roadmap.md](../roadmap.md).

> Tài liệu này sẽ được cập nhật khi service được thiết kế. Hiện chưa có domain model, DB schema, API hay cấu hình cụ thể.
