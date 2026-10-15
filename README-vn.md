<div align="center">

<a href="https://xime.vn">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/xime-wordmark-white.png">
    <img src="assets/xime-wordmark.png" alt="Xime" width="240">
  </picture>
</a>

# Xime Platform

**Ứng dụng theo ngành và công cụ quản lý trên cùng một nền tảng.**

[English](README.md) · **Tiếng Việt**

![Trạng thái](https://img.shields.io/badge/platform-in%20development-0284c7?style=flat-square)
![Công nghệ](https://img.shields.io/badge/stack-Java%20%7C%20Python%20%7C%20TypeScript-334155?style=flat-square)
[![PyPI](https://img.shields.io/pypi/v/xime?style=flat-square&label=Xime%20Framework&color=7c3aed)](https://pypi.org/project/xime/)

[**Khám phá Xime**](https://xime.vn) · [**Mở ERP**](https://erp.xime.vn) · [**Xem framework**](https://github.com/nguyen-huu-thang/xime-framework)

</div>

---

Xime kết nối các ứng dụng quản lý theo ngành, những module nghiệp vụ dùng chung và một nền tảng backend thống nhất. Các sản phẩm cùng sử dụng hạ tầng đăng nhập, tài khoản, tổ chức, gói sử dụng, lưu trữ và thông báo.

Repository này là trang giới thiệu công khai: nơi khám phá các ứng dụng, tìm hiểu kiến trúc và truy cập những phần mã nguồn được công bố.

## Khám phá Xime

| Website | Nội dung |
| :--- | :--- |
| **[Xime Portal · xime.vn](https://xime.vn)** | Website chính, thông tin sản phẩm và nơi quản lý tài khoản Xime, tổ chức, gói sử dụng |
| **[Xime ERP · erp.xime.vn](https://erp.xime.vn)** | Không gian làm việc chung cho Nhân sự & chấm công, Khách hàng, Kho, Sổ thu chi và Giao việc |

### Ứng dụng theo ngành

| Ứng dụng quản lý | Công việc chính | Website mẫu |
| :--- | :--- | :--- |
| **[Xime Nha khoa](https://nhakhoa.xime.vn)** | Bệnh nhân, lịch hẹn và quy trình phòng khám | [Xem mẫu](https://mau-nhakhoa.xime.vn) |
| **[Xime Nhà hàng](https://nhahang.xime.vn)** | Bàn, đơn món, bếp và thu ngân | [Xem mẫu](https://mau-nhahang.xime.vn) |
| **[Xime Nhà trọ](https://nhatro.xime.vn)** | Nhà, phòng, hợp đồng và các khoản thu | — |
| **[Xime Shop Hoa & Quà tặng](https://shophoa.xime.vn)** | Sản phẩm, đơn đặt theo yêu cầu và giao hàng | [Xem mẫu](https://mau-shophoa.xime.vn) |

> **Đang thử nghiệm.** Trang quản lý dùng tài khoản Xime. Website mẫu minh họa giao diện dành cho khách hàng của cơ sở; tính năng sẽ tiếp tục được cập nhật.

## Một nền tảng, ba tầng

| Tầng | Vai trò |
| :--- | :--- |
| **Ứng dụng** | Nghiệp vụ theo ngành, website công khai, giao diện quản lý và Portal |
| **Module nghiệp vụ dùng chung** | Nhân sự & chấm công, Khách hàng, Kho, Sổ thu chi và Giao việc; tái sử dụng trong ứng dụng theo ngành và kết hợp trong Xime ERP |
| **Base Platform** | Các dịch vụ độc lập nghiệp vụ: định danh, tổ chức, gói sử dụng, thanh toán, lưu trữ, thông báo và hạ tầng phân tán |

Tổ chức là phạm vi dùng chung giữa các sản phẩm. Mỗi ứng dụng giữ quy trình riêng của ngành, đồng thời tái sử dụng những năng lực chung của nền tảng.

### Hạ tầng dùng chung

| Nhóm | Dịch vụ và trách nhiệm |
| :--- | :--- |
| **Tin cậy và đăng nhập** | Trust quản lý hạ tầng tin cậy mật mã; Identity xử lý xác thực; User quản lý tài khoản và hồ sơ |
| **Định danh ứng dụng** | Application và Agent quản lý định danh ứng dụng, bot và AI agent |
| **Tổ chức** | Hồ sơ tổ chức, thành viên, lời mời và vòng đời tổ chức |
| **Gói sử dụng và thanh toán** | Billing quản lý gói, thuê bao và quyền sử dụng; Payment xử lý thanh toán và thông báo kết quả về bên gọi |
| **Lưu trữ và thông báo** | Data cung cấp lưu trữ object; Notification phụ trách gửi thông báo và hộp thông báo cá nhân được lưu lâu dài |
| **Hạ tầng phân tán** | Placement cung cấp thông tin sức khỏe shard; User Locator định vị shard người dùng từ thông tin đăng nhập |

Kiến trúc hướng tới nhiều tenant, các dịch vụ độc lập và khả năng mở rộng theo chiều ngang. Chi tiết triển khai hiện tại và tài liệu vận hành được duy trì nội bộ.

## Xime Framework

**Một đồ thị phụ thuộc, nhiều cửa vào.**

[Xime Framework](https://github.com/nguyen-huu-thang/xime-framework) là framework Python mã nguồn mở được dùng cho các dịch vụ Python của nền tảng. Framework xây trên FastAPI, SQLAlchemy và gRPC, bổ sung dependency injection qua constructor, kiểm tra phụ thuộc lúc khởi động, transaction tường minh và quản lý vòng đời ứng dụng.

Framework hỗ trợ HTTP, WebSocket, gRPC, Unix socket, MQTT, Modbus TCP và OPC UA, cùng cơ chế chạy đa tiến trình.

```bash
pip install xime
```

[Mã nguồn](https://github.com/nguyen-huu-thang/xime-framework) · [Gói PyPI](https://pypi.org/project/xime/) · [Bắt đầu sử dụng](https://github.com/nguyen-huu-thang/xime-framework/blob/main/docs/vn/getting-started.md)

### Công nghệ

| Công nghệ | Vai trò |
| :--- | :--- |
| **Java · Spring Boot** | Dịch vụ lõi, xác thực và xử lý giao dịch |
| **Python · Xime Framework** | Dịch vụ hạ tầng, module nghiệp vụ dùng chung và backend theo ngành |
| **TypeScript · Next.js · React** | Portal, ERP, giao diện quản lý và website công khai |
| **PostgreSQL** | Dữ liệu lưu bền do từng dịch vụ quản lý |
| **gRPC · mTLS** | Giao tiếp có xác thực giữa các dịch vụ backend |

## Mã nguồn công khai và các repository lịch sử

**Xime Framework tiếp tục được phát triển công khai.** Các dịch vụ nền tảng và ứng dụng hiện hành được phát triển nội bộ; mã nguồn của chúng không được công bố trong repository này.

Những repository Base Platform từng công khai được giữ lại để tham khảo quá trình phát triển trước đây. Chúng đã được lưu trữ từ **20/08/2026**, ở trạng thái chỉ đọc và không đại diện cho phiên bản nền tảng hiện tại.

<details>
<summary><strong>Xem bảy repository Base Platform đã lưu trữ</strong></summary>

| Repository lịch sử | Phạm vi |
| :--- | :--- |
| [Trust Service](https://github.com/nguyen-huu-thang/trust-service) | Tin cậy mật mã và chứng chỉ |
| [Identity Service](https://github.com/nguyen-huu-thang/identity-service) | Xác thực và token |
| [User Service](https://github.com/nguyen-huu-thang/user-service) | Tài khoản và hồ sơ |
| [Application Service](https://github.com/nguyen-huu-thang/application-service) | Định danh ứng dụng |
| [Agent Service](https://github.com/nguyen-huu-thang/agent-service) | Định danh bot và AI agent |
| [Data Service](https://github.com/nguyen-huu-thang/data-service) | Lưu trữ object và phân quyền |
| [Notification Service](https://github.com/nguyen-huu-thang/notification-service) | Phần gửi thông báo ở phiên bản trước |

</details>

---

<div align="center">

Phát triển bởi [**Nguyễn Hữu Thắng**](https://github.com/nguyen-huu-thang)

[Xime Portal](https://xime.vn) · [Liên hệ](https://xime.vn/lien-he) · [English](README.md)

</div>
