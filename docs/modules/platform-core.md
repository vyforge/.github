# platform-core

## 1. Mục đích

Nền chung cho mọi starter của platform-java (ADR-0015). Module giữ ngữ cảnh tenant và request, các kiểu giá trị dùng
chung (tiền, tham chiếu bản ghi, chuẩn hoá tiếng Việt) và các khối hạ tầng nhỏ như cache theo tenant và mã hoá secret.
Module không phụ thuộc Spring Security hay JPA, nên domain của app và các thư viện thuần (`platform-expression`,
`platform-forms-core`) dùng được.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-core` (có sẵn qua `platform-starter-web` và gần như mọi starter) |
| Package web | không có package riêng |
| Zone admin | không có |
| App liên quan | mọi backend Vyforge |

## 3. Tính năng chính

- **Ngữ cảnh tenant/request**: `TenantContext.require()`, `current()`, `run(tenant, …)`, `call`, `supply`;
  `RequestContext.currentRequestId()`, `sub`. Code nghiệp vụ chỉ đọc được ngữ cảnh. Tenant chỉ lấy từ claim `tenant_id`
  của token đã kiểm chữ ký, không bao giờ lấy từ header, query hay body. `TenantResolver`/`ClaimTenantResolver` đọc
  claim này.
- **Truyền ngữ cảnh sang thread khác** (Micrometer Context Propagation): `@Async`, `applicationTaskExecutor`, scheduler
  và virtual thread của Boot. Platform bật sẵn `spring.task.execution.propagate-context=true`. MDC có `tenant_id` và
  `request_id`.
- **Annotation miễn luật kiến trúc**: `@TenantExempt("lý do")` cho truy vấn bảng dùng chung, `@SchedulingExempt` cho
  `@Scheduled` mà mọi replica đều phải chạy, `@SystemTimeZoneExempt` cho code cố ý dùng múi giờ của JVM. Luật tương ứng
  nằm trong `platform-test`.
- **Cache theo tenant** (`core.cache`): tự bật `@EnableCaching` và cấp `CacheManager` (Caffeine hoặc Valkey/Redis). Key
  luôn có dạng `t:<tenant>:…`. Cache nào chưa khai báo TTL thì app không khởi động. Metric `cache.gets|puts|evictions`.
  `TenantCacheAdmin` cho ops-console xoá cache theo tenant.
- **`SecretCipher`** (`core.crypto`): AES-256-GCM, bản mã dạng `v1:<kid>:<base64url>`, `context` làm AAD (chép bản mã
  sang tenant hay dòng khác thì không giải mã được). Có keyring nhiều khoá, `needsRotation` và `reencrypt` để xoay khoá.
- **`Money`** (`core.money`): số tiền tính theo minor unit kèm mã ISO 4217. payment, billing và expression dùng chung.
- **`EntityRef`** (`core.ref`): tham chiếu bản ghi dạng `crm:order/01J9…`. Có `vrn()` cho IAM, `ofKey` cho khoá có ký tự
  đặc biệt (percent-encode), `parse`/`tryParse`. Từ 0.16, files, collaboration, workflow, search, esign và cms đều dùng
  kiểu này.
- **`VietnameseText`** (`core.text`): `normalize`, `fold` (bỏ dấu, khớp đúng hàm SQL `vyforge.fold`), `tokens` và
  `matchForm` (dạng so khớp cho bộ lọc kiểm duyệt). Quy tắc lấy từ `unaccent.rules` của PostgreSQL 18.6.
- **Các SPI nền mà starter khác cài**:
  - `core.extensible`: `Extensible`, `SaveContext`, `ExtensionGuard`, dùng cho custom-fields.
  - `core.settings`: `Settings`, `SettingDefinition` và chế độ chỉ có giá trị mặc định khi không cài settings.
  - `core.license`: `LicenseGuard`, `@ConditionalOnLicensedModule`, `ModuleDescriptor`, chế độ `auto`/`dev`/`licensed`.
  - `core.signing`: `EnvelopeSigner`/`Verifier` Ed25519 + JCS, dùng cho giấy phép.
  - `core.ops`: `BulkOutcome`, `CursorSlice`.

## 4. Cách dùng nhanh

Không cần khai báo riêng: thêm `platform-starter-web` là có. Khi app dùng `SecretCipher`:

```yaml
vyforge.crypto:
  keys:
    k2026a: ${CRYPTO_KEY_2026A}   # openssl rand -base64 32
  active-key: k2026a
vyforge.cache.caches:
  products: { ttl: 10m, max-size: 5000 }
```

```java
String tenant = TenantContext.require();
for (String t : tenants) {
    TenantContext.run(t, () -> reportService.generate());   // job nền đặt tenant tường minh
}
String stored = cipher.encrypt(apiKey, "payment:vnpay:" + tenant + ":" + configId);
EntityRef ref = EntityRef.of("crm", "order", id);              // "crm:order/<id>", ref.vrn() cho authz
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.tenant.claim` / `.default-tenant` | `tenant_id` / `default` | Claim chứa tenant; token không có claim thì dùng tenant mặc định |
| `vyforge.tenant.required` | `false` | `true`: token thiếu claim tenant → 403 `tenant_required` |
| `vyforge.cache.enabled` / `.type` | `true` / `auto` | `auto`: dùng Valkey/Redis khi có `RedisConnectionFactory`, không có thì Caffeine |
| `vyforge.cache.caches.<tên>.ttl` / `.max-size` | (bắt buộc) / `10000` | Cache thiếu TTL thì app không khởi động |
| `vyforge.crypto.keys.<kid>` / `.active-key` | rỗng | Keyring của `SecretCipher`. Có bean cần cipher mà keyring sai thì app không khởi động |
| `spring.task.execution.propagate-context` | `true` | Truyền ngữ cảnh sang thread khác |

## 6. Tích hợp với module khác

- `platform-starter-security` điền `TenantContext` ngay sau khi xác thực. `platform-starter-jpa` dùng ngữ cảnh này để
  lọc tenant bằng Hibernate `@TenantId`. `platform-starter-messaging` mang tenant qua header Kafka `tenant_id`.
- `platform-starter-authz` dùng `EntityRef.vrn()` (`Resources.of(ref)`).
- Các SPI `license`, `settings`, `extensible` do `platform-starter-licensing`, `platform-starter-settings` và
  `platform-starter-custom-fields` cài.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng, không có chế độ remote.
- Không có action IAM riêng.
- Sự kiện: package `core.license.event` (đổi trạng thái giấy phép) cho `platform-starter-licensing` dùng.

## 8. Trạng thái & còn mở

- Đã ổn định từ 0.10–0.12. Phiên bản hiện tại 0.16.1. Từ 0.16, `SaveContext` có thêm `attributes`; code destructure
  record này phải sửa.
- Giới hạn đã biết:
  - Executor tự tạo, hoặc `CompletableFuture.supplyAsync(...)` không truyền executor, không mang ngữ cảnh. Bọc bằng
    `ContextExecutorService.wrap(...)`.
  - Khi nâng PostgreSQL bản major phải kiểm lại `VietnameseText` bằng
    `JpaExtrasAssertions.assertVietnameseTextContract` của `platform-test`.
