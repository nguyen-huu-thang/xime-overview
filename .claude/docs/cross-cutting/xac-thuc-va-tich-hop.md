# Xác thực & Tích hợp giữa các service

> Hợp nhất từ: `luong-xac-thuc.md` (data, notification), `trust-integration.md` (user), `trust-service.md` (mô hình tương tác), `identity-service-design.md` (auth flow), `data-service` token hierarchy.
> Đây là bản duy nhất mô tả cách các service phối hợp — các bản gốc nằm rải ở nhiều repo.

## Chuỗi tin cậy phân lớp

```text
Trust Service          Identity Service          Các service khác (User, Data, App...)
     │ provide signing key →│                          │
     │                      │── sign JWT ─────────────→│ (client nhận token)
     │ provide cert (mTLS) →│                          │
     │                      │←── verify JWT (LOCAL) ───│ (bằng cached public key)
```

- **Trust** = source of truth duy nhất cho key/cert lifecycle. Identity **không** tự generate signing key.
- **Identity** = nơi duy nhất issue/revoke JWT.
- Các service khác **verify JWT cục bộ** bằng public key đã cache — không gọi Identity/Trust mỗi request.

## 1. Login Flow (end-to-end)

```text
Client  POST /api/v1/auth/login
        { identifier, identifierType, credential, credentialType, userShardId? }
   │
   ▼ Identity Service
   1. Normalize identifier
   2. Resolve shard (dùng userShardId nếu có, hoặc gọi Search Service)
   3. Gọi Subject Owner Service (user-service) verify credential
   │
   ▼ User Service shard
   4. Verify credential (BCrypt/Argon2id) + check account state
   5. Trả VerifiedIdentity { identity_id, subject_type, tenant_id }
   │
   ▼ Identity Service
   6. Generate access token (ES256, 5–15') + refresh token (family_id)
   7. Sign JWT bằng signing key lấy từ Trust (cached)
   │
   ▼ Client  nhận Access + Refresh Token
```
Identity không tin tuyệt đối `userShardId` từ client — validate & fallback Search. Chi tiết routing: [services/search-service.md](../services/search-service.md).

## 2. Request flow vào một resource service (vd Data Service)

Mỗi request mang **2 lớp danh tính**: `Authorization: Bearer JWT` (danh tính Subject) **và** mTLS service certificate (danh tính Service runtime).

```text
Bước 1 — Verify Service Identity (mTLS): verify cert (Trust ký) + service_id/shard_id khớp request metadata
Bước 2 — Verify JWT: extract kid → public key từ cache → verify signature → validate claims (exp, nbf, iss, aud)
Bước 3 — Resolve Subject: lấy sub (identity_id) → subject_cache → subject_type, name → "Authenticated Subject"
Bước 4 — Authorize: System Permission → Object Permission → Ownership → Visibility → ALLOW/DENY
```

Token hierarchy tổng quát:
```text
Trust (key pair) → Identity (sign JWT) → Subject (identity_id, subject_type, permissions) → Resource Service (verify → resolve → evaluate → execute)
```

## 3. JWT Public Key Sync (không gọi realtime)

```text
Startup:  service → Trust.GetPublicKeys(verifier_service_id) → VerificationKeyCache (cache nhiều key)
Runtime:  JWT → kid → VerificationKeyCache → public key → verify
Cache miss (unknown kid): background sync → Trust → retry
```
Cache nhiều key cùng lúc → backward-compatible khi Trust rotate key. Trust down → vẫn verify được bằng cached key.

## 4. mTLS — service-to-service

- Credential/internal API chỉ expose qua **gRPC + mTLS** (không public REST, không API key/shared secret).
- Cert chứa `service_id`, `shard_id`; verify trust chain qua Root CA; extract `service_id` từ **SAN** (không dùng CN).
- **So khớp 2 lớp** (chống spoofing): `cert.service_id == request.service_id` và `cert.shard_id == request.shard_id`.

## 5. Bootstrap & tích hợp Trust (mọi service)

```text
Service startup
  → Trust.GetRootCertificate          → build trust store (cache dài)
  → Trust.GetPublicKeys               → VerificationKeyCache
  → Trust.RotateCertificate (nếu chưa có cert)  → lưu cert + private key (ENCRYPTED) + next refresh token
  → sẵn sàng nhận request
Runtime: scheduler rotate cert khi gần hết hạn (RotateCertificate + refresh token one-time, bound với cert hiện tại)
```

**Nguyên tắc bắt buộc:**
- Trust là **control-plane** — không nằm trong flow verify từng request; service chạy được khi Trust down 1–2 tháng (cached key/cert).
- Không tự generate/ký cert → phải lấy từ Trust. Private key phải **encrypt** khi lưu, không hardcode path.
- Giữ boundary: mỗi service là một credential/data domain; Trust là cryptographic trust.

## 6. Subject & Permission cache sync (cho resource service như Data)
- **Subject cache** (`identity_id/subject_type/name`): source of truth ở Subject Service (user/application-service); sync qua **event** hoặc **periodic**; dùng cho audit/logging/authz.
- **Permission cache** (system permission của Subject): source of truth ở application/subject-service; cập nhật khi permission đổi.
- Eventual consistency chấp nhận được — không tight-couple giữa DB nguồn và cache.
