# ADR-0026: API key, ứng dụng đối tác và phantom token qua gateway

- **Trạng thái:** Chấp nhận — 27/10/2026. Thiết kế chi tiết: [api-access.md](../design/api-access.md). Liên quan
  [ADR-0008](0008-platform-auth-server.md), [ADR-0009](0009-iam-authorization.md),
  [ADR-0010](0010-api-gateway-central-iam.md), [ADR-0011](0011-tenant-and-user-directory.md),
  [ADR-0015](0015-tenant-data-isolation-platform-core.md).
- **Phát hành:** `platform-api-access-core`, `platform-starter-api-access`, `platform-api-access-gateway` (platform-java
  0.14.0); điểm nối ở `platform-auth-server` (main); `@vyforge/api-access` 0.1.0 và admin MFE `/m/api-access` xong trên
  nhánh, chờ merge; route và bảo vệ ở `platform-gateway` đang làm.

## Bối cảnh

Khách hàng và đối tác cần gọi API của app bằng máy (tích hợp ERP, đồng bộ đơn hàng, marketplace). Cần API key có scope,
xoay khoá, rate limit và quota theo gói, ứng dụng đối tác cài vào từng tenant, developer portal. Thay đổi chạm cả
auth-server và gateway, nên ghi thành ADR riêng thay vì một mục của ADR-0020. Service phía sau chỉ tin JWT (ADR-0010,
ADR-0015); gateway đang gỡ mọi header danh tính không ký.

## Quyết định

1. **Ứng dụng là danh tính trung tâm.** Key, OAuth client, rate, quota, webhook gắn vào app; một app nhiều key; app tách
   khỏi installation (đăng ký một lần, cài vào từng tenant, quyền theo từng lần cài). Policy IAM quản lý của app là inline
   policy `api-access:app/<id>` của principal `app:<id>`.
2. **Định dạng key** có tiền tố theo loại và môi trường, checksum CRC32 base62 (kiểm trước khi tra DB, quét rò rỉ được).
   Chỉ lưu hash mật mã, hiện bản rõ một lần; xoay có thời gian chồng tối đa 7 ngày.
3. **Phantom token:** gateway đổi API key lấy JWT ngắn hạn của auth-server bằng token exchange; auth-server kiểm key qua
   `/internal/keys:verify` của api-access. Service phía sau không đổi. Mã kết quả kiểm key (`VALID`, `EXPIRED`,
   `DISABLED`, `RATE_LIMITED`…) là `code` của Problem Details và nhãn metric; lỗi exchange mang `vyforge_verify_code`.
4. **Chế độ local (không có gateway):** bọc `JwtDecoder` bằng BeanPostProcessor, key kiểm trong JVM và dựng thành `Jwt`
   cùng claim với token của auth-server, nên `TenantContextFilter`, `Authz`, `CurrentUser` chạy y nguyên;
   `BearerTokenResolver` đọc `X-Api-Key`.
5. **Rate và quota theo API plan của app**, không theo key; `KeyPolicy.rate/burst` là `RateSpec` đầy đủ để gateway dựng
   rule mà không cần cấu hình đồng bộ.
6. **Chống leo quyền khi cài:** người cài phải giữ mọi action của scope được đồng ý (`Authz.holds(action, "*")`).
7. **Key `test` chỉ là nhãn**, chưa ghép tenant sandbox.
8. **auth-server:** token tự ký cho lời gọi `/internal/**` (scope `api-access.verify`); token từ API key không lưu
   `oauth2_authorization` (cache 15 giây + pub/sub ở gateway cho thu hồi); scope hẹp `auth.clients.partner`; client
   multi-tenant; scope trong JWT của client credentials theo installation là giao (xin ∩ được cấp).
9. **Authorization code cho client multi-tenant ở tenant khác (K9, bổ sung 27/10/2026)**, chi tiết ở
   [api-access.md §3.12](../design/api-access.md#312-authorization-code-cho-app-multi-tenant-ở-tenant-khác-k9):
   - Tenant chọn bằng tham số `tenant` ở `/oauth2/authorize` (mở rộng ADR-0011 §2 cho client `multiTenant`); không có
     tham số thì tenant nhà. App lấy slug từ marketplace (`initiateLoginUri`, OIDC third-party initiated login) hoặc
     webhook `ApplicationInstalled`.
   - Tenant khác tenant nhà bắt buộc installation `ACTIVE` (còn hạn đồng ý), kiểm ở authorize, đổi code và mỗi lần
     refresh; không kiểm được thì không cấp. Scope token = xin ∩ scope của client ∩ granted ∩ người dùng đồng ý. Quyền
     người dùng không giao lúc cấp mà kiểm ở mỗi request (policy IAM ∧ `PermissionBoundary` theo scope); vì vậy K9 chỉ
     bật (`vyforge.auth.api-access.user-delegation.enabled`) khi service đích có `PermissionBoundary` (đã có, quyết định 10).
   - Hai tầng consent: tenant admin đồng ý khi cài (`ConsentScreen` của api-access); người dùng đồng ý ở trang consent
     của auth-server, khoá `(client, người dùng, tenant)` trong bảng `oauth2_tenant_consent` (không dùng khoá
     `(client, người dùng)` của SAS). Hỏi lại khi lần đầu ở tenant, xin thêm scope, cài lại, hết hạn, `prompt=consent`,
     người dùng đã rút.
   - Token người dùng có `sub` người dùng, `tenant_id` tenant cài, `client_id` app, `vyf_inst`. `InstallationRevoked` →
     task gọi `POST /api/v1/admin/clients/{clientId}/tenants/{tenant}:revoke-authorizations` xoá authorization và
     consent của `(client, tenant)`; refresh trả `invalid_grant`; gateway chặn access token còn hạn theo `vyf_inst`.

10. **Ranh giới quyền (`PermissionBoundary`, bổ sung 28/10/2026)**, chi tiết ở
    [api-access.md §3.6](../design/api-access.md#36-scope-iam-và-ranh-giới-quyền):
    - `-authz`: ranh giới là **tập policy** (`PermissionBoundary`), SPI `PermissionBoundaryProvider` khai claim cần đọc.
      Allow ⇔ policy ∧ mọi ranh giới; Deny ở đâu cũng thắng; ranh giới không cấp thêm quyền. Áp cho mọi đường kiểm quyền
      (cả `holds`, việc nền, Policy Simulator), cache cùng policy.
    - api-access: `ScopeBoundaryProvider` = scope của token ∧ scope installation ∧ scope key, hai phần sau đọc lại từ kho
      (key thu hồi, installation gỡ có hiệu lực với token còn hạn).
    - Khác đề xuất ban đầu (`Optional<Predicate<String>> boundaryFor(AuthzPrincipal)`): cần Deny/Condition, giải thích
      được trong simulator, và khoá cache/lưu vào việc nền theo claim.

## Phương án đã loại

| Phương án | Lý do loại |
|---|---|
| Chạy Unkey, Kong hoặc Tyk làm thành phần bắt buộc | Thêm service phải trực; Unkey FSL (không phải OSI), Kong từ 3.10 cần Enterprise; không gắn được tenant + IAM |
| Gateway chuyển header danh tính không ký (`X-Consumer-ID`) | Trái ADR-0010/0015; service chỉ tin JWT |
| Nhận key qua query string | Lộ trong log và lịch sử trình duyệt |
| Lấy lại bản rõ của key (vault) | Mất thì xoay |
| Mở dynamic client registration ra Internet | Bề mặt tấn công (CVE-2026-59354) |
| Hash nhanh không mật mã (murmur kiểu Tyk) | Không chống được dò khi rò DB |
| Lưu phantom token vào `oauth2_authorization` | Mỗi lượt miss phải ghi DB; chọn cache ngắn và chấp nhận không introspect/revoke ở auth-server |
| K9: chọn tenant sau đăng nhập bằng màn hình ở auth-server | Màn hình chọn tenant chưa có (ADR-0011), thêm bước cho mọi lần đăng nhập; để sau, dùng chung khi có |
| K9: suy tenant từ installation | Mơ hồ khi người dùng thuộc nhiều tenant cùng cài app; app không chủ động chọn được |
| K9: consent của người dùng bằng `ConsentScreen` của api-access | Bước consent phải ở origin và phiên của auth-server giữa luồng chuyển hướng |
| Ranh giới quyền dạng `Predicate<String>` trên action | Không có Deny/Resource/Condition; simulator không giải thích được; không biết claim nào để cache và lưu vào việc nền |
| K9: giao scope với quyền người dùng lúc cấp token | Auth-server không giữ IAM; quyền đổi trong vòng đời refresh token; kiểm ở request đúng và rẻ hơn |

## Hệ quả

- Thu hồi key có hiệu lực sau tối đa khoảng 15 giây (cache auth-server) cộng pub/sub ở gateway.
- Bảng usage partition theo ngày, tạo khi nhận lô (và job tạo trước 7 ngày), không có partition default.
- Key thu hẹp scope có ranh giới quyền thật ở service có module api-access (`PermissionBoundary`, quyết định 10); service
  chỉ nhận JWT mà không nhúng module thì key thu hẹp chỉ phản ánh trong claim `scope`/`aud` (vẫn bị chặn bởi policy quản lý
  của app).

## Lệch so với thiết kế

- Tên StarterSchema là `apiaccess` (`StarterSchema` chỉ nhận `[a-z][a-z0-9]*`): lịch sử
  `flyway_schema_history_apiaccess`; cờ tắt vẫn là `vyforge.api-access.schema.migrate`.
- `key_id` của bảng usage NOT NULL, UUID 0 cho request không qua key; rollup theo ngày UTC; `last_used_at` cập nhật lúc
  nhận lô thay vì job riêng.
- App PRIVATE tạo xong là ACTIVE; PUBLIC đi DRAFT → IN_REVIEW → PUBLISHED; installation của app PRIVATE không gỡ riêng.
- Secret OAuth không trả lúc tạo app (client tạo bất đồng bộ qua task `client-sync`); lấy bằng `client-secret:rotate`.
- Portal refresh chạy đồng bộ (job và lệnh operator), không qua task.
- auth-server: client không multi-tenant gửi `tenant` thì bỏ qua (tương thích ngược); không phụ thuộc
  `platform-api-access-core`, hằng số giao thức chép vào `ApiKeyTokenPolicy`.

## Còn mở

- `AuthServerClientProvisioner` gửi `multiTenant` và chuyển sang scope `auth.clients.partner`; cấu hình
  `vyforge.auth.api-access.*` và client `platform-gateway` (grant `token_exchange`) khi deploy.
- `platform-gateway`: dependency `platform-api-access-gateway`, `vyforge.gateway.api-key.*`, route
  `/api/v1/api-access/**`, public path portal.
- `NotificationBridge`, entitlement `api.max-plan`/`api.keys.max`, gauge key, mẫu che
  key trong `-privacy`, Pact, endpoint secret scanning.
- K9 (quyết định 9) đã code (28/10/2026): auth-server (`TenantSelectionPolicy` có `multiTenant`, kiểm installation khi
  authorize, đổi code và refresh, trang consent và `oauth2_tenant_consent`, `vyf_inst`, `iss` ở phản hồi authorize, API
  thu hồi theo tenant, trang "Ứng dụng đã cấp quyền"), api-access (`installations:check` mở rộng, `initiateLoginUri`, task
  `installation-revoke`), gateway (chặn `vyf_inst`), `@vyforge/api-access` (nút "Mở ứng dụng", trường `initiateLoginUri`).
  Test chấp nhận K9.1–K9.26 ở [api-access.md §3.12.9](../design/api-access.md#3129-test-chấp-nhận-k9x). Còn lại khi
  deploy: bật `vyforge.auth.api-access.user-delegation.enabled` sau khi service đích có `PermissionBoundary`; gateway nâng
  `platform-api-access-gateway` lên bản sau 0.15.0.
