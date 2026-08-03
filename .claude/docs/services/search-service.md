# Search Service (dự kiến — Python)

> Hợp nhất từ: `search-service.md` (identity). Service này **chưa code**, mới có thiết kế.

## Vai trò

**Distributed routing infrastructure** — resolve **vị trí (shard) của user** trong user-service và kiểm tra identifier có tồn tại không. **Đây KHÔNG phải full-text search.**

KHÔNG phải: source of truth · credential database · authentication service. Chỉ lưu: routing index, shard location metadata.

## Mục tiêu kiến trúc
Tối ưu **perceived performance** — frontend gọi Search Service ngay khi user đang nhập (frontend-assisted routing), không chờ nhấn login → warm connection sớm, giảm latency cảm nhận.

## Deterministic Routing
```text
identifier → normalize → hash → partition → search shard → resolve target shard
```
Ví dụ: `username → SHA256 → partition routing → search shard → user-service shard = VN02`.
Normalize trước hash (Email/Phone/Username — giống Identity/User).

## Search Shard Architecture
Search Service được shard. Mỗi shard quản lý một phần identifier space, **lookup hoàn toàn trên RAM**: bloom filter + hashmap + partition routing.

### Bloom Filter
```text
request → bloom filter
  → negative: reject fast (identifier không tồn tại)
  → positive: memory lookup
```
Dùng **counting bloom filter** để hỗ trợ delete/update index (đổi username/email).

## Identifier Change
Khi identifier đổi (`alice123 → alice456`): hash mới → search shard mới; index cũ remove, index mới insert. **Search index có thể di chuyển shard — bình thường.** Dữ liệu gốc ở user-service **KHÔNG đổi shard**.

## Response
Chỉ trả `user_shard_id` (+ optional routing metadata). **Không** trả IP, internal endpoint, network topology.
```json
{ "user_shard_id": "VN02" }
```

## Login Flow

**Client-First (ưu tiên):**
```text
User typing → Frontend normalize + hash local → background search lookup (debounce 100–200ms)
  → search shard → bloom filter → memory lookup → resolve user shard → frontend cache routing
  → user click login → any Identity shard → Identity gọi User shard → verify credential → issue JWT
```
Frontend cache `identifier → user_shard_id` TTL 5–15 phút.

**Fallback (Identity-First):** nếu client không gửi `user_shard_id`:
```text
Client → Identity → Search → resolve shard → User shard → issue JWT
```
Identity không tin tuyệt đối hint từ client — validate & fallback.

## Security & nguyên tắc
Public-facing → cần rate limiting, bot detection, brute-force/anomaly protection. Search chỉ resolve location (không phải source of truth); index stale chấp nhận được (eventual consistency), identity-service retry được; routing luôn **deterministic** (tránh broadcast toàn cluster).
