# Mô hình Subject & định danh "hồn - xác"

> Nguồn: chuỗi trao đổi thiết kế với chủ dự án **2026-06-12** (chốt nền tảng cho Application Service và Agent Service).
> Đây là tài liệu nền cho [services/application-service.md](../services/application-service.md) và [services/agent-service.md](../services/agent-service.md). Liên quan chặt với [xac-thuc-va-tich-hop.md](xac-thuc-va-tich-hop.md).

## 1. Ba khái niệm phải phân biệt (dễ lẫn)

| Khái niệm | Là gì | Định danh | Ví dụ |
|---|---|---|---|
| **SERVICE** | Workload runtime, hạ tầng | cert mTLS (`service_id`, `shard_id`) do Trust cấp | `social-post-service`, `data-service` |
| **APPLICATION** | Sản phẩm logic, một Subject hạng nhất | `identity_id` 24 byte do application-service cấp | Xime Social, Xime Chat, app nha khoa |
| **BOT / AI_AGENT** | Tác nhân tự hành hoạt động trong domain ứng dụng như một chủ thể | `identity_id` 24 byte do agent-service cấp | Moderation Bot, AI Assistant, robot |

Lưu ý: "Notification Bot" (BOT - chủ thể) khác hoàn toàn `notification-service` (SERVICE - hạ tầng).

## 2. SubjectType chuẩn toàn platform

Theo chuẩn của data-service (`SubjectType` enum):

```text
HUMAN · BOT · AI_AGENT · APPLICATION
```

**Luật bất biến: SERVICE không bao giờ là Subject** - không bao giờ xuất hiện trong `sub` của JWT, không sở hữu dữ liệu, chỉ hành động thay mặt Subject. Danh tính service chỉ sống ở tầng cert mTLS.

> Việc cần dọn: comment cũ trong identity-service còn ghi `HUMAN / BOT / SERVICE / SYSTEM` (vd `VerifiedIdentity.java`, `authentication_common.proto`) - refactor dần về danh sách chuẩn trên.

## 3. Bản đồ Subject → owner service → cơ chế xác thực

```text
subject_type      Owner service             Cơ chế xác thực                    JWT?
HUMAN          →  user-service              credential qua Identity            Có
BOT, AI_AGENT  →  agent-service             credential qua Identity            Có
APPLICATION    →  application-service       cert (Trust) + cache sync          KHÔNG
(SERVICE       →  trust-service             cert mTLS                          không bao giờ là subject)
```

## 4. Mô hình "hồn - xác" cho APPLICATION

Một Application có hai mặt định danh, **không gộp làm một, không tách rời, mà nối bằng cert**:

- **Phần hồn** (Subject identity): `identity_id` 24 byte, do application-service cấp khi đăng ký app. Sống hàng năm, bất biến (sharding/ownership phụ thuộc nó). Dùng để sở hữu dữ liệu, cấp quyền, audit "nhân danh ai".
- **Phần xác** (Runtime identity): cert mTLS của từng service con, do Trust cấp. Sống theo deploy, scale ngang. Dùng cho mTLS, routing, audit "tiến trình nào".
- **Quan hệ 1:N**: một app gồm nhiều service con. Không cần M:N - service nào dùng chung cho nhiều app thì theo định nghĩa thuộc Base Platform (vd payment).

### Cách nối: app identity nằm trong cert

Cert của service con (Application Layer) mang thêm trường `owner_app_identity_id`:

- Trust registry (bảng `services`) thêm cột `owner_app_identity_id` - **nullable**: service thuộc Base Platform không thuộc app nào, cert không có trường này.
- Trong cert: một entry SAN thứ hai (cạnh entry `service_id` hiện có), giá trị 24 byte encode hex/Base62. Với Trust giá trị này là **opaque** - chỉ khắc đúng cái được đăng ký, không validate runtime.
- Service con **đọc cert của chính nó** để biết mình thuộc app nào - không cần API bootstrap riêng, không hardcode id vào config.

### Nguyên tắc phân kênh: cert chỉ chứa thứ bất biến

Cert rotation ~100 ngày, không CRL/OCSP → cert **không được chứa trạng thái**:

```text
Cert (Trust):                       binding "service này thuộc app X" - bất biến
application-service → resource svc: trạng thái app (ACTIVE/DISABLED) + System Permission
                                    qua subject cache + permission cache, trễ tính bằng phút
```

Data-service đã chừa sẵn chỗ (subject cache, bảng cache quyền cho app). Khi quyền/trạng thái đổi, application-service gửi tín hiệu cho resource service. **Cơ chế tín hiệu chi tiết chưa chốt** - không chặn thiết kế.

## 5. Luật resolve Subject tại resource service

```text
Request có JWT                  → subject = JWT.sub            (HUMAN / BOT / AI_AGENT)
Không JWT, cert có app id       → subject = cert.owner_app_id  (APPLICATION)
Không JWT, cert không có app id → request KHÔNG có subject     (chỉ endpoint hạ tầng:
                                                                health, sync, bootstrap)
```

Quy tắc đi kèm:

- **JWT thắng khi cả hai cùng có mặt**: service xử lý request của user thì subject là user; muốn hành động nhân danh app (job nền, dữ liệu app sở hữu) thì gọi **không kèm JWT** - tường minh theo cấu trúc.
- **Audit luôn ghi cặp**: subject (hồn) + actor (`service_id` từ cert - xác). Quyền đánh giá theo subject, truy vết theo actor.
- **Edge tự an toàn**: REST public không có client cert → subject APPLICATION không thể đến từ ngoài; adapter external chỉ chấp nhận JWT.
- Cả gRPC lẫn REST nội bộ đều resolve về cùng một model ở tầng application: `AuthenticatedSubject { identity_id, subject_type, actor_service_id?, tenant_id }` - adapter khác nhau, application layer thấy một thứ duy nhất.

## 6. Thứ tự bootstrap (con người quản lý)

```text
1. Đăng ký app ở application-service        → sinh identity_id 24 byte
2. Đăng ký service ở Trust, kèm owner_app_identity_id
3. Trust cấp cert                            → app id nằm trong cert, không gì can thiệp được
4. Service đọc cert của chính nó             → tự biết mình thuộc app nào
```

Chống dual-write: danh sách "app X gồm những service nào" do **Trust** nắm (cột `owner_app_identity_id`); application-service muốn hiển thị thì đọc từ Trust, không giữ bản sao.

## 7. Lý do thiết kế (tóm tắt các quyết định)

- **Vì sao APPLICATION không dùng JWT**: mọi lời gọi nhân danh app đều nội bộ, cert luôn có sẵn và là proof-of-possession (mạnh hơn bearer token); không phụ thuộc Identity lúc runtime (đúng nguyên tắc control-plane); không tồn tại token nào để lọt ra web. mTLS verify vốn chạy ở mọi hop - đọc thêm một trường cert gần như miễn phí, JWT mới là thứ "thêm vào".
- **Vì sao không cấp identity 24 byte cho từng service con**: quyền sẽ vỡ vụn theo N service thay vì cấp một lần cho app; audit theo service đã có cert lo.
- **Vì sao không dùng `service_id` (string) làm owner dữ liệu**: sharding/ownership cần hash trên identity 24 byte; service ephemeral/đổi tên được, app trường tồn.
- **Vì sao BOT/AI_AGENT vẫn dùng JWT**: agent không phải lúc nào cũng sống trong mesh có cert (robot, bot ngoài), số lượng lớn và động (user tạo bot) → đi đường credential + login flow Identity như HUMAN.

## 8. Điểm còn mở

- Binding có gắn shard/tenant không - chưa quyết. Hiện tại **toàn bộ tenant = null** (tenant chỉ dùng khi cho bên ngoài thuê hạ tầng tách biệt hoàn toàn); để sẵn cột tenant default null, không migrate sau.
- Cơ chế tín hiệu sync (event push / periodic) giữa application-service và resource service - chưa chốt chi tiết.
- Delegation cho agent ("AI Assistant hành động thay user X") - claim `act` (RFC 8693), chưa thiết kế.
