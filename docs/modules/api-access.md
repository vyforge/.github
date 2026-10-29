# api-access — truy cập API cho đối tác (core, gateway, starter, K9)

## 1. Mục đích

Khách hàng và đối tác cần gọi API của ứng dụng: hệ thống ERP của chính tenant (API key), đối tác vận chuyển tích hợp cho
hàng trăm shop, phần mềm kế toán dạng marketplace gọi thay người dùng (K9). Api-access lấy **ứng dụng** làm danh tính
trung tâm (như GitHub Apps), cấp API key an toàn, scope hẹp ánh xạ sang action IAM, plan giới hạn tốc độ và quota, đo mức
dùng, và developer portal. Service phía sau chỉ kiểm JWT: gateway đổi key lấy JWT ngắn hạn (phantom token).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Lõi Java thuần | `platform-api-access-core` | `ApiKeyFormat`, `HmacKeyHasher`, `CidrBlock`, hợp đồng `VerifyResult`/`KeyPolicy`/`UsageBatch`, `@PartnerApi`; test vector và pact cho gateway ngôn ngữ khác |
| Bộ lọc gateway | `platform-api-access-gateway` | Cho `platform-gateway` (Spring Cloud Gateway WebMVC); không kéo JPA/DataSource |
| Starter | `platform-starter-api-access` | Từ 0.14.0; có phí (`licensed: true`) |
| Package web | `@vyforge/api-access` 0.2.0 | Client API, Server Action, lỗi vi/en; `/ui` (hiện key một lần, bảng key, chọn scope, IP allowlist, màn hình đồng ý, marketplace, biểu đồ); `/admin`; `/portal` (Scalar) |
| Zone admin | `apps/api-access-admin` 0.1.0 (`/m/api-access`) | |
| Phụ thuộc ngoài | `platform-auth-server` (scope `auth.clients.partner`, token exchange, authorization code multi-tenant), `platform-gateway` | |
| Service | `platform-services` | Bật module `api-access` |

Thiết kế: [docs/design/api-access.md](../design/api-access.md) (K9: mục 3.12).

## 3. Tính năng chính

- **Ứng dụng** `app:<id>`: app `PRIVATE` của tenant có installation ngầm; app `PUBLIC` của đối tác được người vận hành
  duyệt rồi cài vào từng tenant, tenant đồng ý scope (có hạn). Khoá/mở, xoay client secret, thuộc tính theo
  `ApplicationAttributeSchema`.
- **API key** `vyk_live_<43 base62>_<CRC32>`: chỉ lưu `HMAC-SHA-256(pepper, key)`, bản rõ trả một lần; scope thu hẹp, hạn,
  xoay có thời gian chồng (≤ 7 ngày), IP allowlist (CIDR v4/v6), báo rò rỉ, đổi pepper có fallback (kid). Môi trường
  `live`/`test`.
- **Scope** công khai (`orders.read`) ánh xạ sang action IAM; module ghi policy quản lý cho `app:<id>` trong `-authz`.
- **Ranh giới quyền** (`PermissionBoundary` của `-authz`): quyền của token app = policy IAM của `app:<id>` ∧ scope token ∧
  scope installation ∧ scope key; key thu hồi, hết đồng ý, app bị khoá thì chặn ngay kể cả token còn hạn.
- **Phantom token** ở gateway: key ở `Authorization: Bearer` hoặc `X-Api-Key` → kiểm định dạng/checksum (sai → 401, không
  gọi mạng) → cache → token exchange ở auth-server (`vyforge_key_policy`) → hạn, IP, rate theo app và key, quota (Lua trên
  Valkey), giới hạn request đồng thời theo app → chuyển tiếp với JWT 5 phút → gom usage theo phút. Fail-closed (503
  `api_access.unavailable`). Thu hồi qua kênh Redis `vyforge:api-access:revoked`.
- Không có gateway: `verification: local`, app tự kiểm key (`JwtDecoder` nhận API key).
- **K9 — marketplace gọi thay người dùng**: app `PUBLIC` multi-tenant dùng authorization code + PKCE, người dùng của tenant
  bấm "Kết nối" và đồng ý; token `sub=<người dùng>`, `tenant_id=<tenant cài>`, `vyf_inst`. Từ 0.16.0: kiểm installation
  (`/internal/installations:check`), initiate login URI (migration `apiaccess V4`), thu hồi installation.
- **Plan** (rate theo app và key, `concurrency`), quota qua `Entitlements` của features; usage rollup giờ/ngày, partition.
- **Developer portal**: gộp `/v3/api-docs` của service (chỉ operation `@PartnerApi`), snapshot có hash, changelog, cảnh
  báo breaking, SDK, key thử.
- REST `/api/v1/api-access`: nhóm tenant (`applications`, `keys`, `network-policies`, `marketplace`, `installations`,
  `usage`, `settings`, `scopes`, `plans`), người vận hành (`operator/**`), chính app (`self/**`), nội bộ
  (`internal/keys:verify`, `installations:check`, `usage:ingest`, `scopes`), portal (`portal/**`), `admin/info`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-api-access</artifactId>
</dependency>
<!-- ở platform-gateway -->
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-api-access-gateway</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/api-access
```

```yaml
vyforge:
  crypto: { keys: { k2026a: "${CRYPTO_KEY_2026A}" }, active-key: k2026a }
  api-access:
    hash: { active-key: p2026a, keys: { p2026a: "${API_ACCESS_HASH_KEY}" } }   # khác khoá SecretCipher, ≥ 32 byte
    verification: gateway
    oauth:
      provisioner: auth-server
      auth-server-uri: http://platform-auth-server:9000
      client-registration: vyforge-api-access
      client-secret: ${API_ACCESS_CLIENT_SECRET}      # scope auth.clients.partner
    scopes:
      "[orders.read]":
        title: { vi: "Đọc đơn hàng", en: "Read orders" }
        audience: orders
        actions: [ "orders:order:Get", "orders:order:List" ]
        personal-data: BASIC
spring.security.oauth2.resourceserver.jwt.audiences: [ orders, platform-services ]
```

Đánh dấu API cho portal:

```java
@PartnerApi(scopes = "orders.read")
@GetMapping("/api/v1/orders/{id}")
OrderView get(@PathVariable UUID id) { ... }
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.api-access.verification` | `gateway` | `gateway`, `local`, `both` |
| `vyforge.api-access.hash.active-key`, `keys.<kid>` | — | Pepper HMAC của key (giữ tối đa 2 kid cũ) |
| `vyforge.api-access.oauth.provisioner` | `none` | `none`: chỉ API key; `auth-server`: tạo OAuth client cho app |
| `vyforge.api-access.scopes.<scope>.*` | — | `title`, `audience`, `actions`, `personal-data` |
| `vyforge.api-access.portal.sources.<svc>.url` | — | Nguồn `/v3/api-docs` cho portal |
| `vyforge.api-access.internal-audience` | `platform-services` | Audience token auth-server gọi `/internal/**` |
| `vyforge.api-access.authz.permission-boundary` | `true` | Tắt ranh giới quyền (không khuyến nghị) |
| `vyforge.api-access.notification.enabled` | `true` | Thông báo key sắp hết hạn, rò rỉ, quota, đồng ý hết hạn… |
| `vyforge.api-access.schema.migrate` | `true` | Schema `apiaccess` |
| `vyforge.gateway.api-key.enabled` | — | Bật bộ lọc ở gateway |
| `vyforge.gateway.api-key.cache.max-ttl` / `negative-ttl` | `60s` / `30s` | Thu hồi chậm tối đa `max-ttl` khi mất pub/sub |
| `vyforge.gateway.api-key.usage.flush-interval` | `10s` | Đẩy usage về module |
| `vyforge.gateway.api-key.trusted-proxies` | — | Lấy IP thật từ `X-Forwarded-For` |

## 6. Tích hợp với module khác

- **authz**: bắt buộc cho REST; `PermissionBoundary`, `ScopeBoundaryProvider` (0.15.0), `IamStore` ghi policy cho `app:<id>`.
  Policy dựng sẵn: `TENANT_ADMIN` → `vyforge:ApiAccessAdministrator`, `API_ACCESS_DEVELOPER` → `vyforge:ApiAccessDeveloper`
  (chỉ key `test` của mình), `API_ACCESS_OPERATOR` → `vyforge:ApiAccessOperator`.
- **features**: quota qua `Entitlements`; giới hạn `api.keys.max`, `api.max-plan`.
- **webhooks**: endpoint webhook cho đối tác qua `WebhookEndpoints`.
- **notification**: type `api-access.key.expiring`, `api-access.key.leaked`, `api-access.quota.exceeded`,
  `api-access.consent.*`, `api-access.application.*`.
- **ratelimit**: `RateLimiter` ở gateway; Valkey cho quota và đếm đồng thời.
- **auth-server**: token exchange (`urn:vyforge:params:oauth:token-type:api-key`), provisioner OAuth client `app_*`,
  authorization code multi-tenant (K9). Hợp đồng Pact gateway → auth-server và gateway → module.
- SPI: `ScopeCatalogContributor`, `ApiKeyVerificationHook`, `ApplicationLifecycleHook`, `ApplicationAttributeSchema`,
  `OAuthClientProvisioner`, `QuotaPort`, `WebhookSubscriptionPort`, `ApplicationPermissions`, `RevocationNotifier`,
  `UsageSink`, `PortalSpecSource`.

## 7. Chế độ triển khai, quyền, sự kiện

- Starter thường chạy ở `platform-services`; bộ lọc ở `platform-gateway`; auth-server gọi `/internal/**` bằng token
  `aud=platform-services`. Không có gateway thì `verification: local`.
- **Quyền IAM**: catalog `api-access:*` (tự vào danh mục IAM khi có authz).
- **Sự kiện**: `ApplicationCreated`, `ApplicationUpdated`, `ApplicationDeleted`, `ApplicationSubmitted`,
  `ApplicationReviewed`, `ApplicationSuspended`, `ApplicationResumed`, `ApplicationInstalled`, `ApplicationClientSyncFailed`,
  `InstallationRevoked`, `ConsentExpiring`, `ConsentExpired`, `ApiKeyCreated`, `ApiKeyRotated`, `ApiKeyRevoked`,
  `ApiKeyExpiring`, `ApiKeyLeaked`, `ApiQuotaExceeded`, `PortalSpecPublished`.
- Metric `vyforge.api_access.keys.active|expiring`, `vyforge.api_access.gateway.*`, `vyforge.api_access.usage.*`; health
  `apiAccessProvisioner`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu (core, gateway, starter); 0.15.0 `ScopeBoundaryProvider`, gọi endpoint mới của auth-server;
  0.16.0 K9 (kiểm installation, initiate login URI, thu hồi installation), bridge notification, `ApplicationAttributeSchema`,
  entitlement `api.max-plan`/`api.keys.max`, gauge key, health provisioner, giới hạn đồng thời ở gateway, Pact.
- **Chờ xác nhận** (thiết kế 11.3): tenant sandbox ghép đôi cần quan hệ `sandbox_of` ở auth-server (ADR-0011 chưa có; hiện
  key `test` chỉ là nhãn); ADR riêng hay mục trong ADR-0020. K9 chỉ nên bật khi service đích có `PermissionBoundary` và
  `ScopeBoundaryProvider`.
- **Giới hạn đã biết**: không có Valkey thì đếm request đồng thời theo từng replica; mất pub/sub thì thu hồi chậm tới
  `cache.max-ttl`. Kiểm `api.max-plan` chỉ khi đổi plan.
