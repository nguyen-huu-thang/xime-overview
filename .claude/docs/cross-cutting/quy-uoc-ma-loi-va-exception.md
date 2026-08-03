# Quy ước Mã lỗi & Exception toàn platform

> Nguồn gốc: `D:\tài liệu\xime\quy hoạch mã lỗi.docx` (quy hoạch dải mã ban đầu).
> Đây là **chuẩn chung trung lập ngôn ngữ** cho mọi service Java và Python. Mỗi service tự triển khai theo phụ lục ngôn ngữ tương ứng, nhưng phải tuân thủ cùng một bản đồ dải mã, cùng một định dạng body, cùng một mô hình phân loại lỗi.

## 1. Mục đích & hiện trạng (vì sao cần chuẩn này)

Trước chuẩn này, mỗi service xử lý lỗi một kiểu, không nhất quán và còn lỗ hổng:

- **Java (user, identity, trust):** chưa có domain exception, không có `@RestControllerAdvice` - lỗi nghiệp vụ (không tìm thấy, trùng lặp) trả về HTTP 500 + stack trace thay vì status tường minh. gRPC dùng `IllegalArgumentException` chung + tự map thủ công, làm lộ `exception.getMessage()` ở nhánh INTERNAL.
- **Python (data, notification):** mỗi exception là một class rời, không base chung, không mang mã lỗi, hardcode `try/except` trong từng controller/handler. Chưa dùng hai hook có sẵn của Xime Framework.

Mục tiêu: một nguồn sự thật duy nhất cho mã lỗi, để lỗi trả ra **tường minh, nhất quán, và không bao giờ lộ chi tiết nội bộ ra ngoài phạm vi được phép**.

## 2. Nguyên tắc quy hoạch

1. Hệ thống chia **hai tầng**: Base Platform (số thấp) và Application Layer (số cao hơn). Nhìn độ lớn con số là biết tầng.
2. Mỗi service / app con có một **block mã riêng biệt**, không trùng. Block chừa rộng để mở rộng.
3. Trong mỗi block, mã chia **ba nhóm theo mức độ phơi bày** (visibility), giảm dần mức bảo mật: Private -> System -> Public.
4. Catalog tập trung trong mỗi service; phần dải chung (common) định nghĩa tại chuẩn này.
5. CI kiểm tra: không trùng mã, mã nằm đúng block của service, và **visibility của mã khớp với vùng số** (xem mục 12).

## 3. Phân tầng số

| Tầng | Phạm vi số | Độ rộng block | errorKey (zero-pad) |
|---|---|---|---|
| **Base Platform** | 0 - 999,999 | 10,000 / service | `E` + 6 chữ số (vd `E040000`) |
| **Application Layer** | 1,000,000 - 9,999,999 | 100,000 / app | `E` + 7 chữ số (vd `E1000000`) |

Base Platform reserve tới 999,999 (đủ ~99 service), Application Layer reserve tới 9,999,999 (đủ ~90 app). App có block lớn hơn (100,000) vì một ứng dụng thường gồm nhiều domain (product, order, cart...).

> Quy tắc phân tầng: service dùng chung cho nhiều ứng dụng thuộc Base Platform (số thấp). Logic riêng của một ứng dụng nằm ở Application Layer (số cao). Khi app gọi một base service và nhận lỗi, lỗi đó mang **mã của base service** (dải thấp); chỉ lỗi sinh ra trong chính app mới mang mã dải app.

## 4. Bản đồ dải Base Platform (block 10,000)

Thứ tự phản ánh chuỗi tin cậy: trust -> identity -> application -> user -> agent -> data -> payment -> ...

| # | Service | Block |
|---|---|---|
| 00 | **Lỗi chung (common)** | 000000 - 009999 |
| 01 | Trust | 010000 - 019999 |
| 02 | Identity | 020000 - 029999 |
| 03 | Application | 030000 - 039999 |
| 04 | User | 040000 - 049999 |
| 05 | Agent | 050000 - 059999 |
| 06 | Data | 060000 - 069999 |
| 07 | Payment | 070000 - 079999 |
| 08 | Notification | 080000 - 089999 |
| 09 | Search | 090000 - 099999 |
| 10 | Analytics | 100000 - 109999 |
| 11+ | Dự phòng | 110000 - 999999 |

## 5. Bản đồ dải Application Layer (block 100,000)

> Cập nhật 2026-07-03: quy hoạch lại theo độ quan trọng chiến lược. **Nguồn sự thật cấp block: [../../../../DANG-KY-MA-LOI.md](../../../../DANG-KY-MA-LOI.md)** (root workspace).

| # | Ứng dụng | Block |
|---|---|---|
| 1 | admin (Platform Admin) | 1000000 - 1099999 |
| 2 | xime chat | 1100000 - 1199999 |
| 3 | marketplace (sàn TMĐT) | 1200000 - 1299999 |
| 4 | dental | 1300000 - 1399999 |
| 5 | nha-tro | 1400000 - 1499999 |
| 6 | spa | 1500000 - 1599999 |
| 7 | gara-oto | 1600000 - 1699999 |
| 8 | gym | 1700000 - 1799999 |
| 9+ | Dự phòng | 1800000 - 9999999 |

## 6. Ba nhóm lỗi theo mức phơi bày (cấu trúc trong mỗi block)

Mỗi block chia ba vùng theo **chữ số hàng nghìn** của phần offset - nhìn mã biết ngay loại:

**Service Base Platform (block 10,000):**

| Vùng offset | Hàng nghìn | Nhóm | Mức bảo mật |
|---|---|---|---|
| 0000 - 3999 | 0,1,2,3 | **Private** (nội bộ chính service) | cao nhất |
| 4000 - 6999 | 4,5,6 | **System** (service đọc lẫn nhau) | trung bình |
| 7000 - 9999 | 7,8,9 | **Public** (browser xem được) | thấp nhất |

**App Application Layer (block 100,000):**

| Vùng offset | Nhóm |
|---|---|
| 00000 - 39999 | Private |
| 40000 - 69999 | System |
| 70000 - 99999 | Public |

Ví dụ **User** (040000 - 049999): `E040000-E043999` Private · `E044000-E046999` System · `E047000-E049999` Public.

## 7. Ba base exception class + cơ chế che lỗi theo kênh

Mỗi mã thuộc một trong ba nhóm visibility, hiện thực bằng **ba base exception class**:

```
AppException (abstract, mang ErrorCode; ErrorCode mang sẵn visibility suy từ vùng số)
 ├─ PrivateError   visibility = PRIVATE   - lỗi chỉ tồn tại trong chính service
 ├─ SystemError    visibility = SYSTEM    - service khác (qua gRPC mTLS) được đọc
 └─ PublicError    visibility = PUBLIC    - browser / REST external được đọc
```

Adapter **tự che (redact)** lỗi tùy kênh đi ra, dựa trên `visibility`:

| Class | Log nội bộ | gRPC nội bộ (service <-> service) | REST external (browser) |
|---|---|---|---|
| `PrivateError` | đầy đủ | **che** -> mã common generic | **che** -> mã common generic |
| `SystemError` | đầy đủ | errorKey + message thật | **che** -> mã common generic |
| `PublicError` | đầy đủ | errorKey + message thật | errorKey + message thật |

**Quy tắc che:** khi che, adapter giữ nguyên `httpStatus`/`grpcStatus` (để client vẫn biết loại lỗi), nhưng thay `errorKey`/`message` bằng **mã common cùng họ trạng thái** (mục 9). `PrivateError` bị che luôn quy về `E000000` (lỗi nội bộ, 500/INTERNAL). Chi tiết thật chỉ nằm trong log kèm trace id.

Nhờ vậy, mức bảo mật được thực thi **tự động ở tầng adapter**, không phụ thuộc lập trình viên nhớ che thủ công. Ba class chỉ để người viết code khai báo đúng ý định; nguồn sự thật về phơi bày là `visibility` trong catalog (CI kiểm tra class khớp vùng số).

## 8. Cấu trúc một mã lỗi & định dạng trả ra

Mỗi mã trong catalog mang: `errorKey` (vd `"E040000"`) · `code` (int, vd `40000`) · `message` (tiếng Việt) · `httpStatus` · `grpcStatus` · `visibility` (PRIVATE/SYSTEM/PUBLIC). `errorKey = "E" + code zero-pad` theo độ rộng tầng.

**Body lỗi REST (chung cả Java + Python):**

```json
{ "errorKey": "E047005", "code": 47005, "message": "Người dùng không tồn tại" }
```

HTTP status của response = `httpStatus` của mã (sau khi áp quy tắc che nếu cần).

**Lỗi gRPC:** status code = `grpcStatus`; message = `message`; trailing metadata `xime-error: <errorKey>` + `xime-error-code: <code>` (sau khi áp quy tắc che). Nhánh không xác định -> `E000000`, log đầy đủ, không lộ stack trace.

## 9. Bảng mã common (000000 - 009999)

Dùng chung mọi service; cũng là mã generic mà adapter dùng khi che lỗi (mục 7). Phân theo ba vùng visibility:

**Private (000000 - 003999) - lỗi hạ tầng, không bao giờ ra ngoài:**

| errorKey | code | message | http | grpc |
|---|---|---|---|---|
| E000000 | 0 | Lỗi không xác định | 500 | INTERNAL |
| E000001 | 1 | Lỗi nội bộ hệ thống | 500 | INTERNAL |
| E000002 | 2 | Lỗi cơ sở dữ liệu | 500 | INTERNAL |
| E000003 | 3 | Lỗi cấu hình | 500 | INTERNAL |

**System (004000 - 006999) - lỗi liên service, chỉ service nội bộ đọc:**

| errorKey | code | message | http | grpc |
|---|---|---|---|---|
| E004000 | 4000 | Lỗi gọi service nội bộ | 502 | INTERNAL |
| E004001 | 4001 | Service phụ thuộc không khả dụng | 503 | UNAVAILABLE |
| E004002 | 4002 | Hết thời gian chờ service nội bộ | 504 | DEADLINE_EXCEEDED |
| E004003 | 4003 | Xác thực liên service thất bại | 401 | UNAUTHENTICATED |

**Public (007000 - 009999) - lỗi an toàn cho client, browser xem được:**

| errorKey | code | message | http | grpc |
|---|---|---|---|---|
| E007000 | 7000 | Yêu cầu không hợp lệ | 400 | INVALID_ARGUMENT |
| E007001 | 7001 | Dữ liệu đầu vào không hợp lệ | 400 | INVALID_ARGUMENT |
| E007002 | 7002 | Chưa xác thực | 401 | UNAUTHENTICATED |
| E007003 | 7003 | Phiên làm việc đã hết hạn | 401 | UNAUTHENTICATED |
| E007004 | 7004 | Không có quyền truy cập | 403 | PERMISSION_DENIED |
| E007005 | 7005 | Không tìm thấy tài nguyên | 404 | NOT_FOUND |
| E007006 | 7006 | Tài nguyên đã tồn tại | 409 | ALREADY_EXISTS |
| E007007 | 7007 | Vi phạm ràng buộc nghiệp vụ | 422 | FAILED_PRECONDITION |
| E007008 | 7008 | Quá nhiều yêu cầu | 429 | RESOURCE_EXHAUSTED |

> Quy tắc che ở mục 7 ánh xạ về các mã Public trên theo cùng họ trạng thái (vd che một `SystemError` NOT_FOUND ra REST -> `E007005`), riêng `PrivateError` quy về `E000000`.

### Ví dụ mã theo service (mỗi service tự quản đầy đủ trong repo)

| errorKey | service / nhóm | message | http | grpc |
|---|---|---|---|---|
| E040005 | User / Private | Lỗi đồng bộ shard nội bộ | 500 | INTERNAL |
| E044000 | User / System | Không resolve được user theo identity | 404 | NOT_FOUND |
| E047005 | User / Public | Người dùng không tồn tại | 404 | NOT_FOUND |
| E047006 | User / Public | Tài khoản đã tồn tại | 409 | ALREADY_EXISTS |
| E027002 | Identity / Public | Token không hợp lệ | 401 | UNAUTHENTICATED |
| E1070200 | Web bán hàng / Public | Sản phẩm không tồn tại | 404 | NOT_FOUND |

## 10. Mã thành công (1 - 999)

Mã thành công luôn Public, ít hơn 4 chữ số, errorKey `"S" + số`:

| Dải | Phạm vi | Ví dụ |
|---|---|---|
| 1-99 | Microservice cha | `S1` Thành công, `S2` Lưu thành công, `S3` Xóa thành công, `S4` Xác thực thành công |
| 100-199 | Web bán hàng | `S100` Đăng ký thành công, `S101` Đặt hàng thành công |
| 200-299 | Web lưu trữ | `S200` Tải lên thành công |
| 300-399 | Mạng xã hội | `S300` Tạo bài viết thành công |

Mã thành công là tùy chọn: REST mặc định dùng HTTP 2xx + body dữ liệu. Chỉ đính khi nghiệp vụ cần phân biệt loại thành công cho client.

## 11. Phụ lục A - Triển khai Java (Spring Boot)

### A.1 Visibility + Catalog enum

> **Giữ domain pure:** `ErrorCode` nằm ở tầng domain nên KHÔNG dùng Spring `HttpStatus` hay `io.grpc.Status.Code`. Dùng `int httpStatus` + một enum `GrpcCode` thuần (mirror tên `io.grpc.Status.Code`). Adapter chuyển đổi: REST `ResponseEntity.status(int)`, gRPC `Status.Code.valueOf(ec.getGrpcCode().name())`.

> **Chuẩn bố trí package (chốt 2026-06):** tách 2 nơi để giữ domain framework-neutral và không tạo phụ thuộc vòng:
> - **`domain/error/`** = object thuần: `Visibility`, `GrpcCode`, `Channel`, `ErrorCode` (catalog + `genericFor`), `ErrorRedactor`.
> - **`common/exception/`** = ba lớp exception `AppException` + `PrivateError`/`SystemError`/`PublicError` (import `domain.error.ErrorCode`).
> - **`api/`** = adapter: `api/rest/error/GlobalExceptionHandler` + `ErrorResponse`, `api/grpc/error/GrpcErrorMapper`.
>
> Hướng phụ thuộc: `domain/error` (không phụ thuộc ai) ← `common/exception` ← `application` + `api`. Domain thuần vẫn ném `IllegalArgument`/`IllegalState`; adapter map về mã common - domain KHÔNG phụ thuộc `common/exception`.
>
> **Tham chiếu hiện thực đầy đủ theo layout này: trust-service** (`Base Platform/trust/src/main/java/vn/xime/trust/{domain/error,common/exception,api/grpc/error,api/rest/error}/`). Lưu ý: user-service hiện gộp tất cả tại `domain/sharedkernel/error/` (layout cũ, sẽ migrate). Tương tự cho Python: object thuần ở tầng domain, exception ở `common/exception` (data-service đang để object thuần ở `common/error/` - sẽ migrate).

```java
public enum Visibility { PRIVATE, SYSTEM, PUBLIC }

// enum thuần, mirror tên io.grpc.Status.Code
public enum GrpcCode { INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS, PERMISSION_DENIED,
    UNAUTHENTICATED, FAILED_PRECONDITION, UNAVAILABLE, DEADLINE_EXCEEDED, RESOURCE_EXHAUSTED, INTERNAL }

public enum ErrorCode {
    // common Private
    UNKNOWN("E000000", 0, 500, GrpcCode.INTERNAL, Visibility.PRIVATE, "Lỗi không xác định"),
    // common Public
    NOT_FOUND("E007005", 7005, 404, GrpcCode.NOT_FOUND, Visibility.PUBLIC, "Không tìm thấy tài nguyên"),
    // user Public
    USER_NOT_FOUND("E047005", 47005, 404, GrpcCode.NOT_FOUND, Visibility.PUBLIC, "Người dùng không tồn tại"),
    ACCOUNT_EXISTS("E047006", 47006, 409, GrpcCode.ALREADY_EXISTS, Visibility.PUBLIC, "Tài khoản đã tồn tại");

    private final String errorKey; private final int code;
    private final int httpStatus; private final GrpcCode grpcCode;
    private final Visibility visibility; private final String message;
    // constructor + getters; thêm static genericFor(ErrorCode) trả mã common cùng họ
}
```

### A.2 Ba base exception class

```java
public abstract class AppException extends RuntimeException {
    private final ErrorCode errorCode;
    protected AppException(ErrorCode ec) { super(ec.getMessage()); this.errorCode = ec; }
    public ErrorCode getErrorCode() { return errorCode; }
}
public class PrivateError extends AppException { public PrivateError(ErrorCode ec){ super(ec);} }   // PRIVATE
public class SystemError  extends AppException { public SystemError(ErrorCode ec){ super(ec);} }    // SYSTEM
public class PublicError  extends AppException { public PublicError(ErrorCode ec){ super(ec);} }    // PUBLIC
```

Ném lỗi: `throw new PublicError(ErrorCode.USER_NOT_FOUND);`

### A.3 Lớp che chung (redaction)

```java
public final class ErrorRedactor {
    // channel: REST_EXTERNAL hoặc GRPC_INTERNAL
    public static ErrorCode forChannel(ErrorCode ec, Channel channel) {
        Visibility v = ec.getVisibility();
        boolean allowed = switch (channel) {
            case GRPC_INTERNAL -> v == Visibility.SYSTEM || v == Visibility.PUBLIC;
            case REST_EXTERNAL -> v == Visibility.PUBLIC;
        };
        if (allowed) return ec;
        if (v == Visibility.PRIVATE) return ErrorCode.UNKNOWN;     // E000000
        return ErrorCode.genericFor(ec); // SYSTEM bị che ở REST -> mã common cùng họ (E007xxx)
    }
}
```

### A.4 REST advice + gRPC mapper (đều che trước khi trả)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    public record ErrorResponse(String errorKey, int code, String message) {}

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handle(AppException ex) {
        if (ex.getErrorCode().getVisibility() != Visibility.PUBLIC) log.error("Non-public error", ex);
        ErrorCode ec = ErrorRedactor.forChannel(ex.getErrorCode(), Channel.REST_EXTERNAL);
        return ResponseEntity.status(ec.getHttpStatus())
            .body(new ErrorResponse(ec.getErrorKey(), ec.getCode(), ec.getMessage()));
    }
    // @ExceptionHandler(Exception.class) -> UNKNOWN (E000000), log đầy đủ
}
```

```java
// gRPC: trong catch của mỗi API
public static StatusRuntimeException toStatus(Throwable ex) {
    ErrorCode raw = (ex instanceof AppException ae) ? ae.getErrorCode() : ErrorCode.UNKNOWN;
    if (raw.getVisibility() == Visibility.PRIVATE) log.error("Private error leaked to gRPC", ex);
    ErrorCode ec = ErrorRedactor.forChannel(raw, Channel.GRPC_INTERNAL);
    Metadata t = new Metadata();
    t.put(Metadata.Key.of("xime-error", Metadata.ASCII_STRING_MARSHALLER), ec.getErrorKey());
    return Status.fromCode(Status.Code.valueOf(ec.getGrpcCode().name()))
        .withDescription(ec.getMessage()).asRuntimeException(t);
}
```

## 12. Phụ lục B - Triển khai Python (Xime Framework)

### B.1 Catalog + visibility

```python
from dataclasses import dataclass
from enum import Enum
import grpc

class Visibility(Enum):
    PRIVATE = 1; SYSTEM = 2; PUBLIC = 3

@dataclass(frozen=True)
class ErrorDef:
    error_key: str; code: int; message: str
    http_status: int; grpc_status: grpc.StatusCode; visibility: Visibility

ERROR_CODES: dict[str, ErrorDef] = {
    "E000000": ErrorDef("E000000", 0, "Lỗi không xác định", 500, grpc.StatusCode.INTERNAL, Visibility.PRIVATE),
    "E007005": ErrorDef("E007005", 7005, "Không tìm thấy tài nguyên", 404, grpc.StatusCode.NOT_FOUND, Visibility.PUBLIC),
    "E060500": ErrorDef("E060500", 60500, "Không tìm thấy object", 404, grpc.StatusCode.NOT_FOUND, Visibility.PUBLIC),
}
def get_error(key: str) -> ErrorDef:
    return ERROR_CODES.get(key, ERROR_CODES["E000000"])
```

### B.2 Ba base exception class

```python
class AppException(Exception):
    def __init__(self, error_key: str, custom_message: str | None = None) -> None:
        d = get_error(error_key)
        self.error_key, self.code = d.error_key, d.code
        self.http_status, self.grpc_status, self.visibility = d.http_status, d.grpc_status, d.visibility
        self.message = custom_message or d.message
        super().__init__(self.message)

class PrivateError(AppException): ...   # PRIVATE
class SystemError(AppException): ...    # SYSTEM
class PublicError(AppException): ...    # PUBLIC
```

Ném: `raise PublicError("E060500")`.

### B.3 REST handler + gRPC interceptor (đều che theo visibility)

```python
# REST: đăng ký qua hook framework
async def app_exception_handler(request, exc: AppException) -> JSONResponse:
    ec = redact_for_rest(exc)   # PUBLIC giữ nguyên; còn lại -> common generic
    if exc.visibility is not Visibility.PUBLIC: _log.error("Non-public error", exc_info=exc)
    return JSONResponse(status_code=ec.http_status,
        content={"errorKey": ec.error_key, "code": ec.code, "message": ec.message})

# app/config/web.py
configure_exception_handlers({AppException: app_exception_handler, RequestValidationError: validation_handler})
```

```python
# gRPC: interceptor đọc thẳng catalog, che PRIVATE
class AppExceptionInterceptor(grpc.aio.ServerInterceptor):
    # bắt AppException -> ec = redact_for_grpc(exc) (SYSTEM/PUBLIC giữ; PRIVATE -> E000000)
    #   context.abort(ec.grpc_status, ec.message,
    #       trailing_metadata=(("xime-error", ec.error_key), ("xime-error-code", str(ec.code))))
    ...
# app/config/grpc.py
configure_grpc_interceptors([AppExceptionInterceptor()])
```

Thay cho pattern `try/except ... HTTPException`/`context.abort` hardcode trong từng controller/handler của data-service và notification-service.

## 13. Quản trị mã lỗi

- **Sở hữu dải:** mỗi service chỉ định nghĩa mã trong block của mình; common (000000-009999) định nghĩa tại đây, copy vào từng service (hoặc tách thư viện chung sau).
- **CI kiểm tra:** (1) không trùng `code`/`errorKey`; (2) mã nằm đúng block service; (3) `visibility` khớp vùng số (Private 0-3999, System 4000-6999, Public 7000-9999 trong block 10,000; tương ứng cho block app 100,000); (4) class exception ném khớp visibility của mã.
- **Bất biến ngữ nghĩa:** không tái dùng một code đã phát hành cho ý nghĩa khác; message có thể tinh chỉnh, ý nghĩa + status + visibility không đổi.

## 14. Lộ trình áp dụng dần

1. **Java user-service** (đang làm REST): thêm `Visibility` + `ErrorCode` enum + ba class `AppException` + `ErrorRedactor` + advice + gRPC mapper; thay dần exception generic.
2. identity, trust khi chạm tới.
3. **Python data, notification:** thêm `error_code.py` (kèm visibility) + ba class + handler/interceptor che, đăng ký qua hook; gỡ try/except hardcode.
4. payment, application, agent, search, analytics và các app Application Layer: áp dụng chuẩn ngay từ đầu khi code.

## 15. Liên quan

- Nguồn quy hoạch gốc: `D:\tài liệu\xime\quy hoạch mã lỗi.docx`.
- Hook framework: `xime framework/xime/adapters/web/_config.py` (`configure_exception_handlers`), `xime framework/xime/adapters/grpc/interceptors/_config.py` (`configure_grpc_error_mappings`, `configure_grpc_interceptors`). Gap ghi nhận tại `xime framework/.claude/docs/routing-layer.md`.
- Bảng so sánh service & dải: [../services.md](../services.md).
