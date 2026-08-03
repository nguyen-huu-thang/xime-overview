# Xime Framework — Socket Adapter (Unix Domain Socket IPC)

> Tóm tắt từ `socket-adapter.md` (framework). Transport layer cho Local IPC.

## Mục tiêu & phạm vi
Transport layer cho **Local IPC** giữa các process trên **cùng máy Linux** qua **Unix Domain Socket (UDS)**. Cho phép Xime Service (Python) gọi **Native Engine** (C++/Rust/Go) xử lý tác vụ nặng (encryption, compression, hashing, thumbnail, video) — developer chỉ làm việc với `@command`/`@stream` + DTO + `UploadStream`/`DownloadStream`; framework giấu socket/framing/serialization/session/dispatch.

**Vì sao không dùng gRPC**: `Microservice ≠ Native Engine`. Native Engine giống FFmpeg (`Input → Process → Output`). UDS: ít tầng hơn (`Python → UDS → C++` thay vì `Python → TCP → HTTP/2 → proto → C++`), ít copy, latency thấp, bảo mật bằng kernel Linux (file permission + `SO_PEERCRED`) thay vì TLS/token. gRPC vẫn là chính cho Service↔Service; Socket **bổ sung**.

**v1 có**: UDS Linux, command/upload/download, multi-server theo `server_id`, session multiplex trên 1 connection, bảo mật file-permission + SO_PEERCRED. **v1 chưa có**: TCP/Named Pipe/Windows, bidi đồng thời, credit-based flow control, reconnect/retry tự động, TLS/token.

## Vị trí & cấu trúc
Ngang hàng WebAdapter/GrpcAdapter (không phải starter). `xime/adapters/socket/`: `_adapter.py` (vòng đời, accept loop, dispatch), `_config.py`, `_protocol.py`, `_session.py`, `_peercred.py`, `_client.py` (SDK Python↔Python). `@command`/`@stream`/streams/scanner nằm ở **`core/contract/`** (dùng chung Code-First gRPC).

```python
app.use(SocketAdapter("crypto"))      # → /run/xime/crypto.sock
app.use(SocketAdapter("thumbnail"))
```

## Controller
```python
class CryptoController:
    server_id = "crypto"
    def __init__(self, crypto_service: CryptoService): ...
    @command("hash")
    async def hash(self, request: HashRequest) -> HashResponse: ...
    @stream("encrypt")
    async def encrypt(self, request: EncryptFileRequest, upload: UploadStream) -> EncryptFileResponse: ...
    @stream("download")
    async def download(self, request: DownloadFileRequest, download: DownloadStream) -> None: ...
```
Đăng ký: `configure_socket_controllers("api.socket")` (package cũng phải trong `dependency.scan`). Error mapping: `configure_socket_error_mappings({NotFoundException: "NOT_FOUND", ...})`.

## Protocol nội bộ
Frame header **16 byte cố định**: `MAGIC("XM",2B) + VERSION(1B) + MSG_TYPE(1B) + SESSION_ID(u64,8B) + PAYLOAD_LEN(u32,4B) + PAYLOAD` (`HEADER_FORMAT = ">2sBBQI"`).

MessageType: `COMMAND_REQUEST/RESPONSE`, `STREAM_START/CHUNK/END/RESPONSE`, `ERROR`, `CANCEL`. Payload: envelope MessagePack `{"endpoint","data"}`, response `model_dump()`, **chunk = raw bytes** (không msgpack, tránh overhead), error `{"code","message"}`. `msgpack` là optional dep (`xime[socket]`).

## Session Manager & dispatch
Trái tim adapter: **một connection chạy nhiều stream đồng thời**. Core: **một read loop / connection** đọc tuần tự, **demux theo `session_id`** vào `{session_id: asyncio.Queue}`; handler `await upload.read()` thực chất `await queue.get()`. `ConnectionWriter` serialize ghi frame (`asyncio.Lock` + `drain()` → backpressure). Trade-off v1: queue đầy → **head-of-line blocking** cả connection (chấp nhận vì Native Engine thường mở 1 connection/job nặng); v2 sẽ credit-based per-session.

## Bảo mật Linux (2 lớp)
1. **File permission** (sau bind): `chmod 0600`, `chown` → kernel chặn ngay ở `connect()`.
2. **SO_PEERCRED**: server đọc `(pid, uid, gid)` của client khi accept; `allowed_uids` rỗng = chấp nhận mọi UID (đã có file perm); lưu `peer_pid/peer_uid` vào request context cho audit.

Config `application.yml`: `socket.{dir:/run/xime, permission:"0600", owner, group, allowed_uids, session_timeout, max_chunk_size, recv_queue_size}`.

## Client SDK Python
```python
client = SocketClient("/run/xime/crypto.sock"); await client.connect()
resp = await client.command("hash", HashRequest(file_id="f1"))
async with client.stream("encrypt", EncryptFileRequest(...)) as up:
    while chunk := await file.read(65536): await up.write(chunk)
    response = await up.finish()
```
Native Engine C++/Rust/Go tự implement cùng protocol.

## Lộ trình
Credit-based flow control, abstraction `Transport` (TCP/Named Pipe), bidi stream, Contract Model dùng chung Code-First gRPC (một Controller → Socket + gRPC), SDK client đa ngôn ngữ.
