# ADR-0010: API Gateway, dịch vụ IAM tập trung và token exchange khi có nhiều service

- **Trạng thái:** Chấp nhận — 26/10/2026. Mở rộng [ADR-0009](0009-iam-authorization.md) (IAM) và
  [ADR-0008](0008-platform-auth-server.md) (auth server); mô hình BFF của [ADR-0006](0006-frontend-auth-bff.md) giữ nguyên.

## Bối cảnh

ADR-0009 đặt policy IAM trong database của từng app. Khi hệ thống có nhiều backend dùng chung người dùng và group,
cách này buộc admin cấu hình quyền ở nhiều nơi, và thay đổi quyền chỉ có hiệu lực ngay trên replica nhận thay đổi
(replica khác chậm tới TTL cache). Các service cũng cần gọi nhau **thay mặt người dùng** mà không dùng lại nguyên token
của client gọi (token đó có `aud` là service đầu tiên), và cần một cửa vào chung cho BFF, mobile, đối tác.

## Quyết định

### 1. API Gateway (`platform-gateway`) — xác thực và định tuyến, không phân quyền

- Spring Cloud Gateway (biến thể Server **WebMVC**, Spring Cloud 2026.0 — dùng bản RC đến khi GA, Renovate tự nâng),
  repo riêng, image riêng, dùng `platform-parent` và các starter như mọi service.
- Gateway làm: kiểm JWT (chữ ký qua JWKS của auth server, hạn, `aud` có `platform-gateway`) cho route cần đăng nhập;
  định tuyến theo path (`/api/v1/<service>/**` hoặc cấu hình); rate limit theo `sub` (có token) hoặc IP; giới hạn kích
  thước body; gắn `X-Request-Id`/trace; trả lỗi dạng Problem Details; CORS cho client công khai.
- Gateway **chuyển nguyên header `Authorization`** cho service. Không thay token bằng header kiểu `X-User-Id`.
- Gateway **không đánh giá IAM/ABAC**. Mọi service vẫn tự kiểm JWT và quyền (zero trust: gọi nội bộ, job, consumer không
  đi qua gateway).
- BFF đổi `API_BASE_URL` sang gateway khi có từ hai backend trở lên; token của BFF phải có `aud` gồm `platform-gateway`
  và service đích.
- Cổng local: gateway `8000` (management `8001`).

### 2. Dịch vụ IAM tập trung (`platform-iam-server`) — lưu tập trung, đánh giá tại chỗ

- `platform-starter-authz` có hai chế độ `vyforge.authz.mode`:
  - `local` (mặc định, như ADR-0009): bảng `iam_*` trong database của app, API quản trị trong app.
  - `remote`: app **không** có bảng IAM; nạp group + policy của người dùng từ `platform-iam-server`, cache cục bộ,
    **đánh giá trong bộ nhớ** bằng cùng engine. Thuộc tính `principal.*` (app tự cung cấp) và `resource.*`
    (`ResourceAttributesResolver`) vẫn ở app. `/api/v1/iam/me/evaluate` vẫn chạy trong app (để `can()` của BFF dùng);
    API quản trị `/api/v1/iam/**` khác chỉ có ở IAM server.
- `platform-iam-server`: app Spring Boot dùng starter ở chế độ `local` + bật API nội bộ
  (`vyforge.authz.internal-api.enabled=true`). Database PostgreSQL riêng. Cổng local `8085` (management `8086`).
- **API nội bộ** (chỉ client có scope `iam.internal`, lấy token bằng `client_credentials`, `aud` gồm
  `platform-iam-server`):

  | Endpoint | Nội dung |
  |---|---|
  | `POST /api/v1/iam/internal/resolve` | Body `{"tenantId","principalId","roles":[...]}` → `{"groups":[{"id","name"}],"policies":[{"id","name","document"}]}` (document là JSON policy dạng chuỗi) |
  | `PUT /api/v1/iam/internal/catalogs/{service}` | Body `{"actions":[{"action","description"}]}` — service đăng ký danh mục action lúc khởi động (idempotent, thay toàn bộ danh mục của service đó) |

  IAM server kiểm tra policy theo **hợp danh mục** của mọi service đã đăng ký cùng action của chính nó. Service ở chế độ
  `remote` cấu hình `vyforge.authz.remote.url` và client registration Spring Security `vyforge-iam`.
- Lỗi khi gọi IAM server: dùng bản cache cũ nếu còn (tối đa `stale-ttl`, mặc định 10 phút), không có thì **từ chối**
  (không bao giờ mở quyền khi lỗi).

### 3. Sự kiện xoá cache

- Mỗi thay đổi policy, group, thành viên, gắn policy, inline policy phát sự kiện lên Redis pub/sub kênh
  **`vyforge:iam:changed`**, nội dung là `tenantId` (chuỗi). Mọi replica (cả app `local` lẫn `remote`) đăng ký kênh và
  xoá cache của tenant đó.
- Bật tự động khi có Redis (`spring-data-redis` + `RedisConnectionFactory`); tắt bằng
  `vyforge.authz.invalidation.redis.enabled=false`. Không có Redis thì giữ hành vi cũ (TTL).
- Mất kết nối Redis không làm hỏng request; TTL (`cache-ttl`, mặc định 60 giây) vẫn là giới hạn trên.

### 4. Token exchange (RFC 8693) ở `platform-auth-server`

- Grant `urn:ietf:params:oauth:grant-type:token-exchange`, chỉ cho **confidential client** được bật grant này.
- Request: `subject_token` = access token do auth server cấp, `subject_token_type` =
  `urn:ietf:params:oauth:token-type:access_token`, `audience` = service đích (bắt buộc), `scope` tuỳ chọn.
- Quy tắc:
  - `subject_token` còn hạn, tài khoản còn hoạt động, và `aud` của nó **chứa client_id của client đang đổi** (service
    chỉ đổi được token gửi tới chính nó).
  - `audience` phải nằm trong `audiences` đăng ký của client đang đổi.
  - Scope kết quả ⊆ scope của `subject_token`.
- Token kết quả: `sub`, `roles`, `amr` (và `tenant_id` nếu có) giữ nguyên của người dùng; `aud` = `[audience]`; thêm
  claim `act` = `{"sub": "<client_id đang đổi>"}` (lồng `act` cũ nếu subject token đã có); không cấp refresh token; hạn
  không vượt hạn còn lại của `subject_token`.
- `platform-starter-security`: `TokenExchangeInterceptor` cho `RestClient` — đổi token của request hiện tại sang
  audience của service đích, cache theo `(token, audience)` đến trước khi hết hạn; dùng client registration
  `vyforge-token-exchange`.

## Hệ quả

- Admin quản lý quyền ở một nơi; thu hồi quyền có hiệu lực gần như ngay trên mọi replica (khi có Redis).
- Thêm hai thành phần phải vận hành (gateway, IAM server) và một phụ thuộc lúc chạy: service `remote` cần IAM server
  để nạp quyền cho người dùng mới (đã có cache và stale cache giảm ảnh hưởng).
- Gateway chạy Spring Cloud bản RC cho đến khi 2026.0 GA.
- Dự án một backend tiếp tục dùng chế độ `local` và gọi thẳng backend; không bắt buộc gateway.
- Chưa làm: tenant trong token (`tenant_id` do auth server phát), danh bạ người dùng cho app, lọc danh sách theo policy
  ở truy vấn — các ADR sau.
