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

## Hệ quả

- Thu hồi key có hiệu lực sau tối đa khoảng 15 giây (cache auth-server) cộng pub/sub ở gateway.
- Bảng usage partition theo ngày, tạo khi nhận lô (và job tạo trước 7 ngày), không có partition default.
- Ranh giới quyền thật của key thu hẹp scope cần `PermissionBoundary` trong `-authz`; hiện key thu hẹp chỉ phản ánh
  trong claim `scope`/`aud`.

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
- `PermissionBoundary` (`-authz`), `NotificationBridge`, entitlement `api.max-plan`/`api.keys.max`, gauge key, mẫu che
  key trong `-privacy`, Pact, endpoint secret scanning, authorization code cho client multi-tenant ở tenant khác.
