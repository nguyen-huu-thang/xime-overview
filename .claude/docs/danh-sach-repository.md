# Danh sách Repository

> Tổng hợp từ brief (`giới thiệu.txt`) + xác minh thực tế trên đĩa.

## Đường dẫn gốc

- Base Platform: `D:\code\xime\Base Platform`
- Application Layer: `D:\code\xime\Application Layer` (rỗng — chưa bắt đầu)
- Xime Framework: `D:\code\xime\xime framework`

## Repository

| # | Service | Local | GitHub | Khác |
|---|---|---|---|---|
| 1 | Trust Service | `D:\code\xime\Base Platform\Trust` | https://github.com/nguyen-huu-thang/trust-service | |
| 2 | Identity Service | `D:\code\xime\Base Platform\identity` | https://github.com/nguyen-huu-thang/identity-service | |
| 3 | User Service | `D:\code\xime\Base Platform\user` | https://github.com/nguyen-huu-thang/user-service | |
| 4 | Application Service | `D:\code\xime\Base Platform\application` | <https://github.com/nguyen-huu-thang/application-service> | Repo tạo 2026-06-15 |
| 5 | Agent Service | `D:\code\xime\Base Platform\agent` | <https://github.com/nguyen-huu-thang/agent-service> | Repo tạo 2026-06-15 |
| 6 | Data Service | `D:\code\xime\Base Platform\data` | https://github.com/nguyen-huu-thang/data-service | |
| 7 | Notification Service | `D:\code\xime\Base Platform\notification` | https://github.com/nguyen-huu-thang/notification-service | |
| 8 | Xime Framework | `D:\code\xime\xime framework` | https://github.com/nguyen-huu-thang/xime-framework | PyPI: https://pypi.org/project/xime/ |
| 9 | Xime Shop Example | `D:\code\PYTHON\xime\shop` | https://github.com/nguyen-huu-thang/xime-shop-example | App mẫu dùng framework |

## Ghi chú (đã xác nhận với chủ dự án — 2026-06-12)

1. **Identity Service** — link GitHub: https://github.com/nguyen-huu-thang/identity-service ✅
2. **Data Service** — đường dẫn đúng là `...\Base Platform\data` (brief gõ nhầm thành `identity`) ✅
3. **Payment Service** — chủ dự án **chưa kịp viết**; mới là scaffold Spring Boot, chưa có thiết kế/README. Liệt kê là "dự kiến".
4. **Shop Example** — dự án migrate shop PHP/Symfony sang Python dùng Xime, kiến trúc đa lớp (không Hexagonal). Là "ví dụ ứng dụng framework", không thuộc Base Platform.
