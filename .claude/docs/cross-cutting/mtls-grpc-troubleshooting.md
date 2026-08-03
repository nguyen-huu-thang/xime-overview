# mTLS gRPC — Postmortem & Troubleshooting (Spring gRPC 1.0.2)

> Hợp nhất từ: `grpc-mtls-troubleshooting.md` (identity), `mtls-grpc-troubleshooting.md` (user), `mtls-grpc-external-server.md` (trust), `architecture-decisions.md` (gRPC TLS strategy), top-level `grpc-mtls-guide.md`.
> Phát hiện 2026-06-05 · Trạng thái: **RESOLVED** (đã fix ở Trust Service). Áp dụng cho mọi service Java dùng gRPC.

## Vấn đề cốt lõi

**Spring gRPC 1.0.2 KHÔNG sử dụng `@Bean ServerCredentials` do ứng dụng khai báo.**

Dù bean `TlsServerCredentials` tạo thành công và log hiện `BUILDING TLS SERVER CREDENTIALS`, server runtime vẫn chạy **plaintext**. Dump `ShadedNettyGrpcServerFactory`: `keyManager=null`, `trustManager=null`, `clientAuth=NONE`.

Nguyên nhân — `DefaultGrpcServerFactory.credentials()`:
```java
if (keyManager == null) return InsecureServerCredentials.create();  // plaintext!
```
Spring gRPC chỉ lấy TLS từ pipeline: `spring.grpc.server.ssl → SslBundle → KeyManagerFactory/TrustManagerFactory → DefaultGrpcServerFactory → NettyServerBuilder`. Bean của ứng dụng bị bỏ qua.

## Triệu chứng

| Phía | Lỗi |
|---|---|
| Server | `HTTP/2 client preface string missing or corrupt` + hex `16 03 01 ...` |
| Client | `NotSslRecordException` + hex `00 00 12 04 ...` |

`16 03 01` = TLS ClientHello → client gửi TLS nhưng server chờ plaintext HTTP/2.

## Quy trình debug (thứ tự bắt buộc)

1. Client đổi sang `.usePlaintext()` → restart → nếu thành công: server đang plaintext.
2. Inspect `GrpcServerFactory` — dump `keyManager`, `trustManager`, `clientAuth`.
3. **Chỉ khi `keyManager != null`** mới debug certificate (CA, SAN, hostname verification).

> Đừng mất thời gian debug Root CA / SAN khi TLS handshake còn chưa bắt đầu.

## Giải pháp theo loại service

### Service thông thường (Identity, User, Notification, Data...) — dùng `spring.grpc.server.ssl` + SslBundle (PEM file)
```yaml
spring:
  ssl:
    bundle:
      pem:
        identity-service:
          keystore: { certificate: classpath:certs/server.crt, private-key: classpath:certs/server.key }
          truststore: { certificate: classpath:certs/ca.crt }
  grpc:
    server:
      ssl: { enabled: true, bundle: identity-service, client-auth: REQUIRE }
```

### Trust Service (cert lưu trong DB, cần rotation runtime) — tự build `NettyServerBuilder`
SSL Bundle thiết kế cho PEM file → không hợp với kiến trúc cert-from-DB. Giải pháp: **bỏ `spring-grpc-server-spring-boot-starter`** khỏi `pom.xml` (tránh conflict port), tự build:
```java
NettyServerBuilder.forPort(9090, provider.buildServerCredentials()).addService(...).build();
```
Pipeline: `Database → TrustSelfCertificateLoader → TrustSelfCertificateResolver → GrpcExternalServerCredentialsProvider → TlsServerCredentials → NettyServerBuilder`.

#### Các file Trust tạo để bật mTLS port 9090 (5 file, package `vn.xime.trust.infrastructure.ssl` + `config.grpc`)
- `SelfCertificate` (record: publicCertificatePem, privateKeyPem)
- `TrustSelfCertificateResolver` (RAM cache, `AtomicReference`)
- `TrustSelfCertificateLoader` (`@PostConstruct`: gọi `GetCertificatesUseCase.getActiveCertificate("trust-service")` → wrap PEM → resolver.update; fail-fast nếu không có cert)
- `GrpcExternalServerCredentialsProvider` (build `TlsServerCredentials`: keyManager từ selfCert, trustManager từ `GetRootCertificateUseCase`, `clientAuth REQUIRE`)
- `ExternalGrpcServerMtlsConfig` (`@Bean @DependsOn("trustSelfCertificateLoader") ServerCredentials`)

Nguyên tắc layering: loader/provider **không** gọi repository/domain trực tiếp — luôn qua `application.usecase`. Internal server (9091, `InternalGrpcServerConfig`) **không đổi**. Format: `ServiceCertDto.publicCert` = Base64 DER → wrap `BEGIN CERTIFICATE`; `privateKey` = Base64 PKCS8 đã decrypt → wrap `BEGIN PRIVATE KEY`; root CA đã là PEM đầy đủ.

## Xác minh mTLS hoàn chỉnh (2 test là đủ)

| Test | Kết quả mong đợi | Chứng minh |
|---|---|---|
| Bỏ `trustManager` phía client | `PKIX path building failed` | Client verify server cert |
| Bỏ `keyManager` phía client | `TLSV1_ALERT_CERTIFICATE_REQUIRED` | Server yêu cầu client cert |

## Bài học
- Không kết luận TLS đang chạy chỉ vì thấy bean `TlsServerCredentials` hay log `BUILDING TLS...`.
- Custom property (`trust.self.mtls.enabled=true`) **không** ảnh hưởng Spring gRPC — Spring chỉ đọc `spring.grpc.server.ssl`.
- Luôn xác minh runtime: `keyManager/trustManager/clientAuth` trong `GrpcServerFactory`.
