# platform-iam-server

## 1. Mục đích

Dịch vụ IAM tập trung cho chế độ `remote` (ADR-0010). Khi nhiều backend dùng chung người dùng và group, policy nằm ở
đây. Mỗi service nạp quyền của người dùng, cache lại và tự đánh giá trong app. Admin quản trị policy ở một chỗ duy nhất.
Tenant admin tự quản quyền trong tenant của mình bằng policy dựng sẵn.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | module **app** `platform-iam-server` trong reactor platform-java. Không nằm trong `platform-bom`, không deploy lên Maven repository; phát hành **image** `platform-iam-server` (cùng version với platform-java, hiện 0.16.1) |
| Package web | `@vyforge/iam` 0.5.0 |
| Zone admin | trang `/iam/*` của `apps/shell` (platform-admin) |
| App liên quan | `platform-auth-server` (issuer, role `TENANT_ADMIN`), `platform-services` và mọi service ở chế độ `remote`, `platform-gateway` (route `/api/v1/iam/**`) |

## 3. Tính năng chính

- **Toàn bộ nghiệp vụ nằm trong `platform-starter-authz`** (chế độ `local` + `internal-api.enabled=true`). Module chỉ
  cấu hình, đóng gói và thêm policy cho tenant admin. Trước đây là repo riêng, đã chuyển vào platform-java và archive.
- **API**:
  - `/api/v1/iam/**` cho admin (quyền `iam:*`): policy, group, thành viên, gắn policy, Policy Simulator, audit log,
    `me/evaluate` cho action `iam:*`.
  - `POST /api/v1/iam/internal/resolve`: service hỏi group và policy của một người dùng (scope `iam.internal`).
  - `PUT /api/v1/iam/internal/catalogs/{service}`: service đăng ký danh mục action lúc khởi động. `service` phải trùng
    `client_id`/`azp`, trừ scope `iam.internal.admin`. Có ghi audit.
- **Xoá cache**: mỗi thay đổi phát `tenantId` lên kênh Redis `vyforge:iam:changed` sau khi commit.
- **Tenant admin**: role `TENANT_ADMIN` (gán ở auth-server) nhận policy `vyforge:TenantAdministrator` trong tenant của
  token.
  - Có mọi `iam:*` trừ inline policy.
  - Luật chống leo thang vẫn áp dụng.
  - Không áp dụng ở tenant `default`.
  - Server không khởi động nếu `admin-roles` chứa role tenant admin.
- **Schema**: Flyway location `META-INF/vyforge/db/iam` của starter. Module không có migration riêng.

## 4. Cách dùng nhanh

```bash
./mvnw install -DskipTests
./mvnw -pl platform-iam-server spring-boot:run -Dspring-boot.run.profiles=local   # PostgreSQL :5434, Redis :6379
docker build -f platform-iam-server/Dockerfile -t platform-iam-server .          # ở gốc repo platform-java
```

Phía service:

```yaml
vyforge.authz:
  mode: remote
  remote: { url: http://platform-iam-server:8085, service: crm }
# client registration vyforge-iam: client_credentials, scope iam.internal
```

## 5. Cấu hình quan trọng

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `DATABASE_URL` / `_USERNAME` / `_PASSWORD` | `jdbc:postgresql://localhost:5434/iam` / `iam` / `iam` | Database riêng |
| `REDIS_URL` | `redis://localhost:6379` | Dùng chung với các service ở chế độ remote |
| `OIDC_ISSUER_URI` / `OIDC_AUDIENCE` | `http://localhost:9000` / `platform-iam-server` | |
| `IAM_ADMIN_ROLES` | `IAM_ADMIN` | Role toàn quyền `iam:*` (dùng khi khởi tạo) |
| `IAM_TENANT_ADMIN_ROLE` | `TENANT_ADMIN` | Role nhận `vyforge:TenantAdministrator` |
| `IAM_TENANT_REQUIRED` | `false` | Bật khi auth-server đã phát `tenant_id` |
| `PORT` / `MANAGEMENT_PORT` | `8085` / `8086` | |
| `vyforge.authz.internal-api.clients.<client_id>.tenants` | mọi tenant | Giới hạn tenant cho từng service |

## 6. Tích hợp với module khác

- Service ở chế độ remote dùng `platform-starter-authz`.
  - IAM server lỗi thì service dùng bản cache cũ tối đa `stale-ttl` (10 phút); quá hạn thì 503 `iam.unavailable`,
    không bao giờ mở quyền.
- `platform-services` đăng ký catalog cho từng module đang chạy, nên client của nó cần scope `iam.internal.admin`.
- Gateway route `/api/v1/iam/**` tới đây, giữ nguyên path (`strip-prefix: false`). `/api/v1/<service>/iam/**` khác
  `me` thì gateway trả 404.

## 7. Chế độ triển khai, quyền, sự kiện

- Deployment riêng: image `platform-iam-server`, chạy non-root uid 10001, chart `app`.
- Quyền: `iam:*`, `iam:audit:List`. Scope nội bộ: `iam.internal`, `iam.internal.admin`.
- Sự kiện: kênh Redis `vyforge:iam:changed`.
- Health: `db`, `redis`, `iamInvalidation`. Metric `vyforge.authz.*`.

## 8. Trạng thái & còn mở

- Ổn định.
- Migration `V1__iam.sql` của starter trùng từng byte với bản chép tay cũ, nên database đang chạy không phải làm gì
  (`IamMigrationTests`).
- Còn mở:
  - Thuộc tính `resource.*`/`principal.*` riêng của service không có ở IAM server. Vì vậy `can()` cho action nghiệp vụ
    phải gọi `/me/evaluate` của chính service.
  - Policy dựng sẵn cho tenant admin nằm trong code, không sửa được qua API.
