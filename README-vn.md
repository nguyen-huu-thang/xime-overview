<div align="center">

# 🧩 Xime Platform

### Nền tảng backend microservice: điểm vào duy nhất tới mọi repository.

[English](README.md) · **Tiếng Việt**

![Status](https://img.shields.io/badge/trạng%20thái-đang%20phát%20triển-orange)
![Stack](https://img.shields.io/badge/công%20nghệ-Java%20·%20Python%20·%20Go-blue)
![Architecture](https://img.shields.io/badge/kiến%20trúc-microservices-success)
![Frameworks](https://img.shields.io/badge/frameworks-Spring%20Boot%20·%20Xime-6E56CF)

</div>

---

**Xime Platform** là nền móng backend để xây nhiều sản phẩm trên cùng một lõi dùng chung. **Base Platform** cung cấp các hạ tầng mà ứng dụng nào cũng cần - tin cậy & bảo mật, xác thực, định danh người dùng, lưu trữ dữ liệu, thông báo - để các ứng dụng (mạng xã hội, thương mại điện tử, SaaS / AI…) tái sử dụng thay vì xây lại mỗi lần. Hệ thống được thiết kế phân tán, đa tenant và mở rộng theo chiều ngang.

Dự án trải trên **nhiều repository**. Thay vì để chúng rải rác, trang này là **nơi duy nhất để khám phá toàn bộ dự án và đi tới từng repo**.

> 👇 **Cần xem code? Chọn một repository bên dưới.**

---

## 📦 Danh sách Repository

### ⭐ Bắt đầu từ đây: nền móng

Nền tảng dùng **đa ngôn ngữ**: các service Java chạy trên **Spring Boot**, còn các service Python chạy trên **xime framework**, nó là framework Python tự xây theo tinh thần Spring Boot (dependency injection tự động, cấu trúc theo quy ước, gRPC code-first). Xime là framework tự xây có repository riêng:

| Repository | Mô tả | |
| :--- | :--- | :--- |
| **[Xime Framework](https://github.com/nguyen-huu-thang/xime-framework)** | Backend framework Python tự xây: nền tảng cho các service Python của hệ thống | [![PyPI](https://img.shields.io/pypi/v/xime?label=pip%20install%20xime&color=3776AB&logo=pypi&logoColor=white)](https://pypi.org/project/xime/) |

### 🧱 Base Platform: các service hạ tầng dùng chung

Service tái sử dụng, độc lập nghiệp vụ: viết một lần, dùng lại cho mọi ứng dụng.

| Repository | Mô tả | Ngôn ngữ |
| :--- | :--- | :---: |
| **[Trust Service](https://github.com/nguyen-huu-thang/trust-service)** | Hạ tầng tin cậy: CA nội bộ, mTLS, khóa ký JWT | `Java` |
| **[Identity Service](https://github.com/nguyen-huu-thang/identity-service)** | Xác thực: cấp JWT, refresh token & xoay vòng | `Java` |
| **[User Service](https://github.com/nguyen-huu-thang/user-service)** | Human Identity Domain: credential, hồ sơ, trạng thái tài khoản | `Java` |
| **[Application Service](https://github.com/nguyen-huu-thang/application-service)** | Application Identity Domain: registry, lifecycle & quyền hệ thống cho subject APPLICATION | `Java` |
| **[Agent Service](https://github.com/nguyen-huu-thang/agent-service)** | Non-Human Identity Domain: credential, lifecycle & luồng JWT cho subject BOT và AI_AGENT | `Java` |
| **[Data Service](https://github.com/nguyen-huu-thang/data-service)** | Lưu trữ object phân tán, phân quyền theo capability | `Python` |
| **[Notification Service](https://github.com/nguyen-huu-thang/notification-service)** | Gửi thông báo ra ngoài (email trước, SMS sau): stateless | `Python` |

> 🛠️ **Đã có code, repo chưa công khai:** **Payment** (thanh toán: thuê bao, ledger ghi kép, định tuyến callback theo chứng chỉ) và **Organization** (định danh công ty/doanh nghiệp dùng chung).
>
> 🔭 Đang dự kiến: **Search** và **Analytics**.

---

## 🏗️ Cách mọi thứ ghép lại

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│  Application Layer   các ứng dụng SaaS theo ngành                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                   ▲  nhúng màn hình
┌──────────────────────────────────────────────────────────────────────────────┐
│  Service ngang   chấm công · sổ thu chi · giao việc · CRM · kho              │
└──────────────────────────────────────────────────────────────────────────────┘
                                   ▲  tái sử dụng
┌──────────────────────────────────────────────────────────────────────────────┐
│  Base Platform                                                                │
│  trust · identity · user · application · agent · organization                │
│  data · notification · payment                                                │
└──────────────────────────────────────────────────────────────────────────────┘
                                   ▲  xây dựng bằng
┌──────────────────────────────────────────────────────────────────────────────┐
│  Frameworks   Spring Boot (Java)  ·  Xime (Python)                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Một nền móng backend xây một lần, dùng lại cho nhiều sản phẩm**: ưu tiên phân tán, shared-nothing, scale ngang, multi-tenant và chịu lỗi tốt.

Mỗi service được viết bằng ngôn ngữ phù hợp nhất với đặc thù của nó:

| Ngôn ngữ | Phù hợp cho | Service |
| :--- | :--- | :--- |
| **Java** (Spring Boot) | nặng giao dịch, trọng bảo mật | trust · identity · user · application · agent · payment |
| **Python** (Xime) | IO-bound, bất đồng bộ, AI / dữ liệu | data · notification · search · analytics |
| **Go** *(dự kiến)* | realtime, streaming, video | phát trực tiếp · cuộc gọi video |

---

<div align="center">

`distributed-first` · `shared-nothing` · `scale ngang` · `multi-tenant` · `chịu lỗi`

**Thực hiện bởi [@nguyen-huu-thang](https://github.com/nguyen-huu-thang)**

</div>
