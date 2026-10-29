# platform-starter-ratelimit

## 1. Mục đích

Giới hạn số request bằng token bucket. Khi có Valkey/Redis, mọi replica dùng chung bucket, nên N replica không thành N
lần giới hạn như các filter cũ tự viết. Dùng cho endpoint nhạy cảm (đăng nhập, OTP), endpoint tốn tài nguyên (export,
search) và rate limit chung ở `platform-gateway` và `platform-auth-server`.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-ratelimit` (kéo `-web`; `spring-boot-starter-data-redis` tuỳ chọn) |
| Package web | không có. Client đọc 429 `ratelimit.exceeded` qua `@vyforge/api-client` |
| Zone admin | không có |
| App liên quan | `platform-gateway`, `platform-auth-server`, các starter files, notification, forms, search, sequence, collaboration, ekyc, privacy-dsr, api-access (phụ thuộc tuỳ chọn) |

## 3. Tính năng chính

- **`@RateLimited(name, key, capacity, refill, refillTokens, sensitive, failureMode)`** đặt trên method của bất kỳ bean
  nào.
  - `key` là SpEL, có sẵn `#sub`, `#ip`, `#tenant`. Mặc định khoá theo `sub`, chưa đăng nhập thì theo IP.
  - Cấu hình sai thì app không khởi động.
- **Rule theo path** `vyforge.ratelimit.rules.<tên>` (`paths`, `methods`, `capacity`, `refill-period`, `key:
  default|ip|tenant`): không cần sửa code. Request khớp nhiều rule thì bị trừ ở từng rule.
- **Bị từ chối**: 429 Problem Details `ratelimit.exceeded` (`params.retryAfter`), kèm header `Retry-After`,
  `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`.
- **Khi Valkey mất**: rule thường cho qua (fail-open). Rule `sensitive: true` từ chối 503 `ratelimit.unavailable`
  (fail-closed). Ghi đè được bằng `failure-mode`. Sau một lần lỗi, starter bỏ qua Redis trong khoảng `backoff`.
- **Khoá bucket**: `t:<tenant>:ratelimit:<rule>:<giới hạn>:<hash>`. Đối tượng (sub, IP, email) được băm SHA-256.
- **Kiểm trong code**: bean `RateLimiter.consume(RateLimitRule, key)`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-ratelimit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
vyforge.ratelimit.rules:
  search:
    paths: [/api/v1/search/**]
    methods: [GET]
    capacity: 30
    refill-period: 1m
```

```java
@PostMapping("/otp")
@RateLimited(name = "otp", key = "#body.phone", capacity = 3, refill = "10m", sensitive = true)
void sendOtp(@RequestBody OtpRequest body) { ... }
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.ratelimit.enabled` | `true` | Tắt cả annotation lẫn rule theo path |
| `vyforge.ratelimit.backend` | `auto` | `auto`: Valkey khi có `LettuceConnectionFactory`, không có thì bộ nhớ; `redis`/`local` để ép |
| `vyforge.ratelimit.redis.timeout` / `.backoff` | `250ms` / `1s` | |
| `vyforge.ratelimit.local.max-buckets` | `100000` | Số bucket tối đa trong bộ nhớ |
| `vyforge.ratelimit.filter-order` | `-98` | Sau filter tenant, trước Idempotency-Key. auth-server đặt `-105` để chạy trước Spring Security |
| `vyforge.ratelimit.rules.<tên>.*` | `refill-period` `1m`, `refill-tokens` = `capacity`, `key` `default` | Rule theo path |

## 6. Tích hợp với module khác

- Đặt sau filter tenant của `-security` nên khoá được theo `sub` và tenant.
- `platform-gateway`:
  - rule `gateway` (`/**`);
  - rule riêng cho IPN thanh toán, link chia sẻ file, OFREP, mở luồng realtime;
  - `platform-api-access-gateway` dùng starter này cho rate theo app và theo key.
- `platform-auth-server`: rule `sign-in`, `email` (sensitive) và `token` (fail-open), khoá theo IP.
- Các starter nghiệp vụ đặt `@RateLimited` trên REST công khai (OTP, nhận form, slug của tenant-lifecycle…).

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Không có action IAM, không phát sự kiện.
- Metric `vyforge.ratelimit.requests{rule,outcome=allowed|rejected|unavailable_allowed|unavailable_rejected}`.

## 8. Trạng thái & còn mở

- Ổn định từ 0.11. Starter thay `RateLimitFilter` tự viết ở gateway và auth-server; mã lỗi đổi từ `too_many_requests`
  sang `ratelimit.exceeded`.
- Không có Valkey thì bucket nằm trong bộ nhớ từng instance, chỉ đúng khi chạy một replica. Production của auth-server
  bắt buộc có Valkey.
- Rate limit danh bạ của auth-server (`directory.requests-per-minute`) vẫn đếm theo từng replica.
