# Notification Service (Python + Xime)

> Hợp nhất từ: `thiet-ke-notification-service.md`, `framework-first.md`, CLAUDE.md.
> Repo: https://github.com/nguyen-huu-thang/notification-service · Local: `D:\code\xime\Base Platform\notification`

## Vai trò & boundary

**Infrastructure service nhẹ, stateless (không database)** — gửi thông tin ra ngoài qua email (trước) và số điện thoại/SMS (sau).

| LÀM | KHÔNG LÀM |
|---|---|
| Nhận request gửi thông báo từ service khác **qua gRPC** | Tạo/lưu/xác minh OTP (đó là việc của caller) |
| Gửi email transactional (OTP, xác nhận, thông báo hệ thống) | Chứa business domain ứng dụng |
| Render template email | Biết ngữ nghĩa thông báo ("xác nhận đơn hàng"...) |
| Gửi SMS *(về sau)* | Quản lý notification preferences, issue JWT, có DB |

**Giao thức duy nhất: gRPC + mTLS** — không có REST API. Với OTP, caller (Identity/User) tự tạo & quản lý mã, chỉ truyền mã trong template data.

## Caller
- **Base Platform**: Identity (MFA login, verify email, reset password), User (xác nhận email/SĐT mới, thông báo đổi mật khẩu).
- **Application Layer**: xác nhận đơn hàng, nhắc lịch hẹn, thông báo giao dịch...

## Kiến trúc (Xime, Hexagonal)

```text
app/
  domain/email/EmailNotification.py    # @dataclass(frozen=True): notification_id(KSUID 24B), recipient, subject, body, channel, status, created_at, sent_at
  application/
    usecase/SendEmailUseCase.py        # normalize email → render template → EmailSenderPort.send
    port/outbound/                     # Protocol: EmailSenderPort.send(to,subject,body); TemplatePort.render(name,context)
    dto/                               # Pydantic: SendEmailCommand(to,subject,template_name,template_data), SendEmailResult(notification_id)
  infrastructure/
    smtp/SmtpEmailAdapter.py           # implements EmailSenderPort (aiosmtplib); config host/port/user/pass
    template/JinjaTemplateAdapter.py   # implements TemplatePort (Jinja2); templates/*.html.j2
  api/grpc/external/NotificationGrpcHandler.py   # nhận gRPC; chỉ map message→Command→UseCase, không business logic
  api/grpc/mapper/NotificationGrpcMapper.py
  config/dependency.py                 # bind EmailSenderPort→Smtp, TemplatePort→Jinja
  common/{exception,constants,util}/   # NotificationChannel, NotificationStatus, IdGenerator(KSUID)
  main.py
```

**Không** có: persistence layer, OTP record/port, database. Handler map exception → gRPC StatusCode (vd `InvalidRecipientError → INVALID_ARGUMENT`), không leak domain exception.

### Proto
```protobuf
service NotificationService { rpc SendEmail (SendEmailRequest) returns (SendEmailResponse); }
message SendEmailRequest { string to=1; string subject=2; oneof content { string body=3; TemplateContent tmpl=4; } }
message TemplateContent { string template_name=1; map<string,string> context=2; }
message SendEmailResponse { bytes notification_id=1; }
```

### Templates
`otp-email.html.j2` (nhận biến `otp_code` từ caller — service không biết mã là gì), `login-alert`, `password-changed`, `generic-email`.

## Nguyên tắc Framework-First
Ưu tiên tối đa tính năng có sẵn của Xime trước khi tự viết. Service tham chiếu thực tế: **data-service** (`app/config/dependency.py`, `app/main.py`, `app/api/grpc/`, `app/application/usecase/`, `app/infrastructure/persistence/repository/`).

## Chạy
```bash
pip install -e "D:\code\xime\xime framework"
pip install -e .
python app/main.py
```
