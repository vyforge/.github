# platform-starter-web

## 1. Mục đích

Chuẩn hoá phần web của mọi backend để client (web, mobile, đối tác) gặp cùng một hợp đồng ở mọi service. Lỗi trả theo
Problem Details có mã ổn định và được dịch vi/en. Request id đi xuyên các service. Request ghi gửi lại không tạo bản
trùng. API danh sách quản trị có một dạng chung để `@vyforge/admin-kit` gọi được. Starter cũng đặt mặc định chống treo
cho `RestClient` khi gọi service khác.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-web` (kéo theo `platform-core`) |
| Package web | `@vyforge/api-client` 0.6.0 (đọc Problem Details), `@vyforge/admin-kit` 0.2.0 (bảng gọi API danh sách) |
| Zone admin | không có |
| App liên quan | mọi backend, `platform-services`, `platform-gateway` |

## 3. Tính năng chính

- **Lỗi → Problem Details (RFC 9457)**, gồm `code`, `detail`, `params`, `errors[]`, `requestId`.
  - Mã lỗi khai báo bằng enum `ErrorCode`.
  - Câu chữ nằm trong `i18n/errors*.properties` và được dịch theo `Accept-Language` (`ProblemDetailLocalizer`).
  - Exception có sẵn: `PlatformException`, `NotFoundException`, `ConflictException`, `BusinessRuleException`.
  - Mã chung của platform: `not_found`, `validation_failed`, `forbidden`, `internal_error`…
- **`X-Request-Id`**: nhận từ client hoặc tự sinh, ghi vào MDC và gửi kèm khi gọi service khác.
- **`Idempotency-Key`** cho POST/PATCH (cách của Stripe):
  - Gửi lại cùng khoá và cùng nội dung thì nhận lại response cũ kèm `Idempotent-Replayed: true`.
  - Cùng khoá, khác nội dung → 422 `idempotency_key_reused`.
  - Request trước đang chạy → 409 `idempotency_request_in_progress`.
- **API danh sách quản trị** (`web.list`):
  - Tham số và kết quả: `ListParams` (`page`, `size`, `sort`, `q`, `cursor`, `include`), `SortWhitelist`,
    `PageResponse`, `CursorPage` (cursor ký HMAC).
  - `AllowedActions`: client gửi `include=allowedActions` để nhận thao tác được phép trên từng dòng.
  - Thao tác hàng loạt `BulkOperation`/`BulkExecutor`: dưới 500 id chạy ngay; lớn hơn thì thành việc nền trên
    `DurableTaskQueue`.
  - Endpoint dựng sẵn `TaskAdminEndpoint` (`{base}/admin/tasks/**`) và `ModuleAuditEndpoint` (`{base}/admin/audit`).
  - `AuditChanges.diff(before, after)` cho audit theo từng field.
- **Gọi service khác** (`RestClient.Builder`, `@HttpExchange`):
  - Mặc định timeout connect 2 s, read 10 s.
  - Retry khi lỗi kết nối hoặc 502/503/504: chỉ với method idempotent, hoặc POST/PATCH có `Idempotency-Key`.
  - Circuit breaker khi app có Resilience4j.
  - Lỗi của service kia → `RemoteServiceException` (đọc Problem Details của nó). Không bắt thì trả 502
    `upstream_error`.
  - Timeout theo từng client: `PlatformHttpClients.builder("reports")`.
- **OpenAPI** (springdoc) có sẵn bearer JWT.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-web</artifactId>
</dependency>
```

```java
public enum CustomerErrors implements ErrorCode {
    EMAIL_TAKEN(409, "customer.email_taken");
    // constructor + code() + status()
}

throw new PlatformException(CustomerErrors.EMAIL_TAKEN, "email", Map.of("email", email));
```

```properties
# src/main/resources/i18n/errors_vi.properties
customer.email_taken=Email '{email}' đã được đăng ký
```

Test giữ danh mục mã lỗi luôn đủ:
`ErrorCatalogAssertions.assertComplete("com.vyforge.crm", Locale.ENGLISH, Locale.forLanguageTag("vi"))`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.web.request-id-header` | `X-Request-Id` | |
| `vyforge.web.idempotency.enabled` | `false` | Bật thì phải tạo bảng `idempotency_keys` (`META-INF/vyforge/sql/idempotency_keys.sql`) |
| `vyforge.web.idempotency.methods` / `.retention` | `POST,PATCH` / `24h` | |
| `vyforge.web.client.propagate-request-id` / `.problem-errors` / `.relay-token` | `true` / `true` / `false` | Hành vi của `RestClient` |
| `vyforge.http.defaults.connect-timeout` / `.read-timeout` | `2s` / `10s` | Áp cho mọi client, trừ khi đặt `spring.http.clients.*` |
| `vyforge.http.retry.enabled` / `.max-retries` | `true` / `2` | Backoff 200 ms ×2, tối đa 2 s |
| `vyforge.http.circuit-breaker.enabled` | `true` | Chỉ chạy khi có Resilience4j |
| `vyforge.web.list.default-size` / `.max-size` / `.max-depth` | `50` / `200` / `10000` | Giới hạn của `ListParams` |
| `vyforge.web.list.cursor-secret` | dẫn xuất từ `vyforge.crypto.*` | Khoá HMAC của cursor, phải giống nhau giữa các replica |
| `vyforge.web.bulk.sync-limit` / `.max-items` | `500` / `100000` | Giới hạn của `BulkExecutor` |
| `spring.web.locale` | `en` | Locale khi request không có `Accept-Language` |

## 6. Tích hợp với module khác

- Thêm `platform-starter-authz` thì có `AllowedActions` (qua port `ActionChecker`).
- Thêm `platform-starter-scheduling` thì có bulk chạy nền và `TaskAdminEndpoint`.
- Thêm `platform-starter-audit` thì có `ModuleAuditEndpoint` và một dòng audit cho mỗi thao tác hàng loạt.
- `platform-starter-security` cung cấp `relay-token` và `TokenExchangeClient` cho `RestClient`.
- Test: lớp con của `AdminListContractTests` (`platform-test`) cho mỗi API danh sách.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng.
- Không có action IAM riêng. Thao tác hàng loạt ghi audit `<service>:<resource>:Bulk<Op>`.
- Không phát sự kiện.

## 8. Trạng thái & còn mở

- Ổn định. API danh sách quản trị có từ 0.13 (admin-kit).
- App tự khai báo `ResponseEntityExceptionHandler` hay `OpenAPI` thì cấu hình của platform lùi lại.
- Giới hạn: không có ngôn ngữ lọc chung. Mỗi module tự bind bộ lọc phẳng vào record filter của mình.
