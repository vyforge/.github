# platform-gateway (repo riêng)

## 1. Mục đích

API gateway của Vyforge (ADR-0010). Gateway làm ba việc: **xác thực JWT, định tuyến, rate limit**; nó **không phân
quyền**. Frontend và đối tác gọi một địa chỉ duy nhất `/api/v<n>/<service>/…`. Gateway gỡ header danh tính giả, chặn API
nội bộ và ops, và đổi API key của đối tác lấy JWT ngắn hạn. Mỗi service vẫn tự kiểm JWT và quyền.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | repo **`platform-gateway`** (app, image `platform-gateway`), version **0.2.1**; Spring Cloud Gateway Server WebMVC (Spring Cloud 2025.1), parent `platform-parent` 0.16.0; dùng `platform-api-access-gateway` |
| Package web | không có. BFF Next.js đặt `API_BASE_URL` trỏ tới gateway |
| Zone admin | không có |
| App liên quan | `platform-auth-server` (issuer, token exchange cho API key), `platform-iam-server`, `platform-services`, backend nghiệp vụ |

## 3. Tính năng chính

- **Xác thực**: JWT của auth-server, `aud` phải có `platform-gateway`. Route công khai khai ở
  `vyforge.security.public-paths`.
- **Định tuyến theo service**:
  - Gateway bỏ `<tên>` trong `/api/v<n>/<tên>/<còn lại>` rồi chuyển tới `<uri>/api/v<n>/<còn lại>`. Khai đích ở
    `vyforge.gateway.services.<tên>.uri`, hoặc biến `VYFORGE_GATEWAY_SERVICES_<TÊN>_URI`.
  - Module của platform-services giữ nguyên path. Mặc định đi tới `PLATFORM_SERVICES_URL`; tách từng module bằng biến
    riêng (`PAYMENTS_URL`, `REALTIME_URL`…).
- **Token**: chuyển nguyên `Authorization`. Gỡ `X-User-Id`, `X-Tenant-Id`, `X-Vyforge-Tenant`… do client tự gửi.
- **Chặn path**: `/api/*/*/internal/**`, `/api/*/ops/**`, `/api/*/*/ops/**` → 404, so cả path đã giải mã.
- **Rate limit** (`platform-starter-ratelimit` trên Valkey, fail-open):
  - bucket chung theo `sub` hoặc IP;
  - rule riêng cho IPN thanh toán, trạng thái thanh toán, callback notification, link chia sẻ file, OFREP, mở luồng
    realtime.
- **API key đối tác** (`platform-api-access-gateway`, tắt mặc định):
  - Kiểm checksum, rồi token exchange ở auth-server để đổi key lấy JWT 5 phút.
  - Kiểm IP allowlist, rate theo app/key, quota, giới hạn request đồng thời theo app (0.16).
  - Gỡ key trước khi chuyển tiếp, gom usage gửi về module api-access.
  - Thu hồi qua kênh Valkey `vyforge:api-access:revoked`. Chặn installation đã gỡ (K9).
- **SSE**: dùng Apache HttpClient 5 nên read-timeout tính giữa hai gói; `text/event-stream` được flush từng gói.
- **Giới hạn**: body quá `max-body-size` → 413; upstream không kết nối được → 502; quá timeout → 504. Lỗi dạng Problem
  Details.

## 4. Cách dùng nhanh

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local   # :8000, Actuator :8001
```

```yaml
vyforge:
  gateway:
    services:
      orders:
        uri: http://orders-api:8080          # /api/v1/orders/customers/1 → /api/v1/customers/1
      iam:
        uri: http://platform-iam-server:8085
        strip-prefix: false
```

Thêm service mới:

1. Khai URI của service ở gateway.
2. Thêm audience của service vào client BFF ở auth-server.
3. Frontend gọi `/api/v1/<tên>/…`; `can()` gọi `/api/v1/<tên>/iam/me/evaluate`.

## 5. Cấu hình quan trọng

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `OIDC_ISSUER_URI` / `OIDC_AUDIENCE` | `http://localhost:9000` / `platform-gateway` | |
| `PLATFORM_SERVICES_URL` | `http://localhost:8087` | Đích mặc định của route module |
| `IAM_SERVER_URL` | `http://localhost:8085` | |
| `GATEWAY_PUBLIC_PATHS` | danh sách mặc định của module | Đặt biến này là thay cả danh sách |
| `GATEWAY_RATE_LIMIT_CAPACITY` / `_REFILL` | `200` / `50` | Bucket chung: dồn tối đa / bù mỗi giây |
| `REDIS_URL` | `redis://localhost:6379` | Valkey dùng chung |
| `GATEWAY_API_KEY_ENABLED` | `false` | Bật API key; cần `GATEWAY_CLIENT_SECRET` |
| `GATEWAY_API_KEY_CACHE_TTL` / `GATEWAY_API_KEY_CONCURRENCY_LEASE` | `60s` / `5m` | |
| `GATEWAY_MAX_BODY_SIZE` | `10MB` | |
| `UPSTREAM_CONNECT_TIMEOUT` / `UPSTREAM_READ_TIMEOUT` | `2s` / `30s` | Read-timeout tính giữa hai gói |
| `GATEWAY_REALTIME_CONNECT_CAPACITY` | `10` | Số lần mở luồng realtime mỗi phút theo `sub` |
| `GATEWAY_CORS_ORIGINS` | rỗng | |

## 6. Tích hợp với module khác

- Dùng starter: `-web`, `-security`, `-ratelimit`, `-observability`, `platform-api-access-gateway`.
- api-access: đẩy usage tới `<API_ACCESS_URL>/api/v1/api-access/internal/usage:ingest`. Hợp đồng Pact gateway →
  auth-server và gateway → module có từ 0.16.
- realtime: SSE đi qua gateway; WebSocket phải dùng route ingress riêng.
- ops: không đi qua gateway.

## 7. Chế độ triển khai, quyền, sự kiện

- Deployment riêng (chart `app`, `docs/helm-values.example.yaml`). CI: `vyforge-ci java-verify` → `image` → `gitops-bump`
  vào `platform-infra`.
- Không đánh giá IAM, không có action riêng.
- Nghe kênh Valkey `vyforge:api-access:revoked` (gồm tin `installation-revoked:<id>`).

## 8. Trạng thái & còn mở

- v0.2.1 chạy trên platform 0.16.0.
- WebSocket chưa proxy được: cần Spring Cloud Gateway Server WebMVC 5.1 (Spring Cloud 2026.0).
- Giới hạn đã biết:
  - Không có Valkey thì giới hạn đồng thời theo app nhân lên theo số replica.
  - Mất pub/sub thì thu hồi key chậm tối đa bằng `GATEWAY_API_KEY_CACHE_TTL`.
- Đổi `base-path` của module (sequence, forms, ai) thì phải sửa route tương ứng.
