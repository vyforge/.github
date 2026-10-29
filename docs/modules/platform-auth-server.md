# platform-auth-server (repo riêng)

## 1. Mục đích

Identity provider OIDC của Vyforge, xây trên Spring Authorization Server (ADR-0008). Server cấp token cho mọi app:

- `sub` ổn định;
- `roles` theo tenant;
- `tenant_id`;
- `aud` theo backend.

Ngoài cấp token, server lo đăng ký, MFA, đăng nhập qua IdP ngoài, quản trị tenant và danh bạ (ADR-0011, ADR-0012). App
chỉ phụ thuộc chuẩn OIDC, nên đổi qua lại với Keycloak chỉ cần đổi issuer, client và claim.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | repo **`platform-auth-server`** (app, image `platform-auth-server`), version **0.2.1**; parent `platform-parent` 0.15.0 ở tag v0.2.1 |
| Package web | `@vyforge/auth` 0.4.0 (Better Auth, provider `vyforge` cho BFF Next.js) |
| Zone admin | Admin console trong `apps/shell` của platform-admin: users, roles, clients, tenants, identity-providers, sign-ins, activity |
| App liên quan | mọi backend (resource server), `platform-gateway` (đổi API key), `platform-iam-server`, `platform-services` (tenant-lifecycle, ekyc, api-access, privacy-dsr) |

## 3. Tính năng chính

- **Giao thức**:
  - Authorization code + PKCE bắt buộc, refresh token xoay vòng, client credentials.
  - Discovery, JWKS, userinfo, RP-initiated logout, revoke, introspect.
  - Token exchange RFC 8693 (`act.sub`, không đổi sang audience của chính auth server).
- **Token**: access token JWT 10 phút. `roles` đọc lại từ database mỗi lần cấp. `tenant_id` cố định theo phiên.
  Claim thêm: `phone_number_verified`, `vyforge_identity_verified`.
- **Tài khoản**:
  - Tự đăng ký (bật/tắt) có xác thực email, quên và đổi mật khẩu, chính sách mật khẩu NIST.
  - Khoá tạm sau 5 lần sai.
  - Rate limit theo IP qua `platform-starter-ratelimit`.
- **MFA**: TOTP và 10 mã khôi phục; passkey (WebAuthn) cũng tính là MFA.
- **IdP ngoài**:
  - Google, Microsoft Entra ID, OIDC bất kỳ.
  - Theo ADR-0012, cấu hình qua API: SAML 2.0, LDAP/AD, Zalo Login. JIT tạo tài khoản, đồng bộ role theo group.
- **Tenant và danh bạ**:
  - Một tài khoản thuộc nhiều tenant, role theo từng tenant.
  - Quản trị `/api/v1/admin/tenants/**` (role `platform-admin` + MFA, hoặc scope `auth.admin`).
  - Quản trị uỷ quyền `/api/v1/tenant-admin/**` (role `TENANT_ADMIN`).
  - Danh bạ `/api/v1/directory/users`.
- **Scope hẹp cho module tự động hoá**:
  - `auth.clients.partner` (api-access);
  - `auth.identity.verify` (ekyc);
  - `auth.tenant.provision` (tenant-lifecycle).
- **API key** (bật `AUTH_API_ACCESS_ENABLED`): đổi key `vyk_…` lấy JWT 5 phút (`sub=app:<id>`); client credentials theo
  installation.
- **Contributor DSR**: `auth.verification`, `auth.account` cho privacy-dsr. Xoá chỉ ở tenant `default`, tài khoản được
  ẩn danh.
- **Sự kiện tích hợp**: `auth.tenant.member_removed`, `auth.account.disabled`, gửi theo Standard Webhooks qua outbox.
- **Vận hành**:
  - Khoá ký RSA trong DB, mã hoá AES-GCM, tự xoay 90 ngày.
  - Session JDBC, chạy nhiều replica không cần sticky session.
  - Job `@ClusterScheduled`.
  - Audit đăng nhập giữ 180 ngày.

## 4. Cách dùng nhanh

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local   # :9000, PostgreSQL :5433, Mailpit :8025
```

Khai client trong cấu hình (đồng bộ mỗi lần khởi động) hoặc qua admin console:

```yaml
vyforge.auth.clients.crm-frontend:
  secret: ${CRM_FRONTEND_SECRET}
  redirect-uris: https://crm.example.com/api/auth/callback/vyforge
  audiences: crm-backend
```

Backend đặt `OIDC_ISSUER_URI`, `OIDC_AUDIENCE=<slug>-backend`, `OIDC_ROLES_CLAIM=roles`. Frontend đặt `OIDC_ISSUER`,
`OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`.

## 5. Cấu hình quan trọng

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `AUTH_ISSUER` | — (bắt buộc) | URL công khai, khớp `OIDC_ISSUER` của app |
| `AUTH_KEY_ENCRYPTION_KEY` | — (bắt buộc, prod từ chối khoá mặc định) | Mã hoá khoá ký và secret TOTP. Mất khoá là mất MFA của mọi người |
| `DATABASE_URL` / `SMTP_*` / `AUTH_MAIL_FROM` | — | PostgreSQL riêng; SMTP gửi email |
| `AUTH_REGISTRATION_ENABLED` | `false` | Bật tự đăng ký |
| `AUTH_ACCESS_TOKEN_TTL` / `AUTH_REFRESH_TOKEN_TTL` | `10m` / `8h` | |
| `AUTH_LOCKOUT_MAX_FAILED_ATTEMPTS` / `AUTH_LOCKOUT_DURATION` | `5` / `15m` | |
| `AUTH_KEY_ROTATION_PERIOD` / `AUTH_AUDIT_RETENTION` | `90d` / `180d` | |
| `AUTH_PASSKEY_ENABLED` | `false` | |
| `AUTH_API_ACCESS_ENABLED` / `AUTH_API_ACCESS_URL` | `false` / — | Đổi API key qua module api-access |
| `AUTH_PRIVACY_CONTRIBUTOR_ENABLED` | `false` | Contributor DSR |
| `AUTH_RATE_LIMIT_BACKEND` / `REDIS_URL` | `local` / — | Production phải đặt `redis` |

## 6. Tích hợp với module khác

- Dùng starter của platform-java: `-web`, `-jpa`, `-audit`, `-scheduling`, `-ratelimit`, `-privacy` (contributor), `-observability`. Không dùng `-security`, vì server tự là issuer.
- `platform-gateway`: client `platform-gateway` có grant `client_credentials,token_exchange`, scope
  `api-access.usage.write`.
- `platform-services`: client có scope `iam.internal iam.internal.admin auth.tenant.provision`.
- `@vyforge/auth` (BFF) tự thêm tham số `tenant=<slug>` khi đăng nhập.

## 7. Chế độ triển khai, quyền, sự kiện

- Deployment riêng (chart `app`, cổng 9000/9001), database PostgreSQL riêng.
- Quyền không qua IAM policy:
  - role `admin` + MFA, hoặc scope `auth.admin` cho `/api/v1/admin/**`;
  - role `platform-admin` cho tenant;
  - role `TENANT_ADMIN` cho tenant-admin;
  - token có `act` luôn bị từ chối ở API quản trị.
- Sự kiện: webhook `auth.tenant.member_removed`, `auth.account.disabled`; event nội bộ thu hồi phiên qua outbox.

## 8. Trạng thái & còn mở

- Đủ phạm vi MVP của ADR-0008 nhưng **chưa qua review bảo mật/pentest**. Theo ADR-0008, production vẫn dùng Keycloak
  cho tới khi có kết quả review.
- Giới hạn đã biết:
  - Access token đã phát vẫn hợp lệ tới hết hạn (10 phút) sau khi thu hồi.
  - Rate limit danh bạ đếm theo từng replica.
  - `actor_token` (delegation) chưa hỗ trợ.
- Thứ tự migration: nhánh tenant (V020–V029) phải deploy trước nhánh idp (V030+). Lệch thứ tự thì chạy một lần với
  `AUTH_FLYWAY_OUT_OF_ORDER=true`.
- Tag v0.2.1 vẫn dùng parent 0.15.0. Nâng lên 0.16.x (cần xác nhận lịch).
