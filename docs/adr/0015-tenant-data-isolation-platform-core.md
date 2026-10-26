# ADR-0015: Tenant ở tầng dữ liệu, `platform-core` và mặc định cho lời gọi ra ngoài

- **Trạng thái:** Chấp nhận — 26/10/2026. Bổ sung cho ADR-0011 (tenant và claim `tenant_id`), [ADR-0009](0009-iam-authorization.md)
  và [ADR-0010](0010-api-gateway-central-iam.md). Thuộc kế hoạch trong [MODULE_PLAN.md](../MODULE_PLAN.md).

## Bối cảnh

ADR-0011 đưa tenant vào token qua claim `tenant_id`, nhưng mới giải quyết ở tầng xác thực. Ở tầng dữ liệu hiện có ba
vấn đề:

- Khái niệm tenant (`TenantResolver`) chỉ nằm trong `platform-starter-authz`. Module nào cần tenant thì phải phụ thuộc
  IAM, hoặc tự đọc claim theo cách riêng.
- Chưa có quy ước chung cho bảng, cache, file, event và job. Các starter sắp làm (files, notification, export, features,
  realtime) đều lưu dữ liệu theo tenant. Nếu mỗi module tự cách ly theo một kiểu, sớm muộn sẽ rò dữ liệu giữa các tenant.
- Ngữ cảnh (tenant, request id) không tự đi theo khi code chuyển sang thread khác: `@Async`, virtual thread, job nền,
  consumer Kafka.

Ngoài ra, `RestClient` do `platform-starter-web` cấp không có timeout mặc định. Chế độ remote của authz từng treo request
người dùng theo IAM server vì thiếu timeout. Lỗi này sẽ lặp lại ở mọi service gọi service khác.

## Quyết định

### 1. Thư viện nền `platform-core`

Thêm module `platform-core` vào `platform-java`. Module này nhỏ và không phụ thuộc Spring Security hay JPA. Mọi starter
khác phụ thuộc vào nó.

- `TenantContext`: tenant hiện tại, chỉ đọc đối với code nghiệp vụ.
- `RequestContext`: request id và `sub` hiện tại.
- `TenantResolver` chuyển từ `-authz` về đây. `-authz` giữ một alias deprecated trong một minor, rồi gỡ ở 1.0.
- Request id chuyển từ `-web` về đây. `RequestIdFilter` và `RequestIdInterceptor` của `-web` dùng `RequestContext`.
- **Truyền ngữ cảnh** dùng thư viện `io.micrometer:context-propagation`, thư viện Spring Boot đã dùng sẵn. Không tự viết
  cơ chế riêng:
  - `TenantContext` và `RequestContext` được đăng ký là `ThreadLocalAccessor`.
  - `TaskDecorator` mặc định cho executor của `@Async`, scheduler và executor của virtual thread.
- **Quy ước cache** (tuỳ chọn, tự bật khi có `spring-boot-starter-cache`):
  - Có sẵn `CacheManager`: Caffeine khi chạy một instance; Valkey/Redis khi có `spring.data.redis`.
  - Dùng `KeyGenerator` mặc định; mọi key tự có tiền tố `t:<tenant>:`.
  - Mỗi cache bắt buộc khai báo TTL qua `vyforge.cache.caches.<tên>.ttl`. Cache nào thiếu TTL thì app không khởi động.
  - Có metric hit/miss cho từng cache.

### 2. Nguồn của tenant

- Tenant **chỉ** lấy từ claim `tenant_id` của access token đã kiểm chữ ký. `-security` điền `TenantContext` ngay sau khi
  xác thực.
- **Không bao giờ** nhận tenant từ header, query hay body. Gateway và `-web` chỉ truyền request id qua header.
- Gọi giữa các service: tenant đi theo token. Với token exchange (ADR-0010), token mới giữ nguyên `tenant_id`.
- Thuộc tính `vyforge.tenant.required` (mặc định `false`) thay cho `vyforge.authz.tenant.required`. Key cũ vẫn được đọc
  trong một minor. Khi bật, request có token mà thiếu `tenant_id` bị từ chối với 403.
- Thiếu claim và `required=false` thì dùng tenant `default`, như ADR-0011.
- Job và consumer không có request thì đặt tenant tường minh: `TenantContext.run(tenant, ...)`. Không có tenant ngầm định.

### 3. PostgreSQL

- Mọi bảng thuộc tenant có cột `tenant_id varchar(63) not null`, và có index bắt đầu bằng `tenant_id`.
- Entity JPA dùng `@TenantId` có sẵn của Hibernate (multitenancy kiểu cột phân biệt). `-jpa` cung cấp
  `CurrentTenantIdentifierResolver` đọc từ `TenantContext`. Hibernate tự thêm điều kiện lọc khi đọc và tự gán giá trị
  khi ghi, nên code nghiệp vụ không viết `where tenant_id = ?`.
- `JdbcClient`, `JdbcTemplate` và native query **không** qua lớp lọc của Hibernate. Các truy vấn này phải có tham số
  tenant tường minh. `platform-test` có luật ArchUnit kiểm việc này, và cần chú thích để miễn từng chỗ khi có lý do.
- Tier 2 bật thêm Row-Level Security của PostgreSQL làm lớp phòng thủ thứ hai. Cách làm:
  - Khi mượn connection, `-jpa` chạy `set_config('app.tenant_id', ..., true)`.
  - Policy RLS so `tenant_id` với giá trị này.
  - Bật bằng `vyforge.jpa.tenant.row-level-security=true` và migration mẫu trong template.
- Bảng dùng chung cho mọi tenant (danh mục hệ thống) không có `tenant_id` và không có `@TenantId`.

### 4. Các kênh dữ liệu khác

| Kênh | Quy ước |
|---|---|
| Cache | Key có tiền tố `t:<tenant>:` (mục 1) |
| Redis/Valkey dùng trực tiếp | Key có tiền tố `t:<tenant>:`, trừ key hạ tầng của platform đã có tên riêng (ví dụ kênh `vyforge:iam:changed`) |
| Kafka | Producer của `-messaging` tự gắn header `tenant_id`. Consumer khôi phục `TenantContext` từ header. Listener thuộc tenant từ chối event thiếu header và đưa event vào DLT |
| File (S3) | Object key có prefix `<tenant>/`. Presigned URL chỉ ký trong prefix của tenant hiện tại (áp dụng khi có `-files`) |
| Job nền | Mỗi lượt chạy gắn với một tenant. Job xử lý nhiều tenant thì lặp tường minh qua danh sách tenant |
| Feature flag | `tenant_id` là thuộc tính mặc định của evaluation context (áp dụng khi có `-features`) |
| Log, trace | `tenant_id` và `request_id` có trong MDC và trong thuộc tính span |

### 5. Mặc định cho lời gọi ra ngoài trong `-web`

- `RestClient.Builder` và client `@HttpExchange` do platform cấp có sẵn connect timeout 2 giây và read timeout 10 giây.
- Ghi đè theo từng client bằng `vyforge.http.clients.<tên>.connect-timeout` và `read-timeout`.
- Retry dùng `@Retryable` của Spring Framework 7. Mặc định chỉ retry method idempotent (GET, HEAD, PUT, DELETE) khi gặp
  lỗi kết nối hoặc 502/503/504; tối đa 2 lần, có backoff.
- Circuit breaker tự bật khi có Resilience4j trên classpath; không bắt buộc.
- Request id tự truyền sang service khác. Token đi theo cơ chế sẵn có của `-security` (relay hoặc token exchange).

### 6. i18n cho backend

Giữ như hiện tại trong `-web`: `MessageSource` vi/en, đọc `Accept-Language`, Problem Details có `code` ổn định và message
theo ngôn ngữ. Service chỉ thêm `messages_vi.properties` và `messages_en.properties` cho message nghiệp vụ.

### 7. Kiểm tra bắt buộc

`platform-test` cung cấp:
- Luật ArchUnit: entity thuộc tenant phải có `@TenantId`. Truy vấn `JdbcClient` phải nhận tham số tenant.
- Helper `TestTenants.withTenant("a", ...)`.
- Test mẫu "tenant A không đọc được dữ liệu tenant B". Template sinh sẵn test này cho entity mẫu, và mọi starter có lưu
  dữ liệu đều phải có test tương tự.
- Test truyền ngữ cảnh qua `@Async`, virtual thread, scheduler và Kafka.

### 8. Phạm vi áp dụng

- `platform-java` phát hành các thay đổi này ở **0.10.0**. Không có breaking change: key và class cũ được giữ dưới dạng
  deprecated tới 1.0.
- template-spring-boot dùng `@TenantId` cho entity mẫu và bỏ cấu hình cache tự viết.
- `platform-iam-server` và `platform-gateway` nâng lên 0.10.0. Gateway chỉ dùng phần request id.
- `platform-auth-server` nâng parent nhưng giữ module tenant riêng. Auth server là nơi sinh ra tenant, nên không đọc
  tenant từ token như các service khác.

## Hệ quả

- Service nghiệp vụ không tự xử lý tenant, cache, timeout hay i18n. Service chỉ đánh dấu `@TenantId`, khai báo TTL cache
  và message, rồi platform lo phần còn lại.
- Thêm một artifact (`platform-core`), nhưng nó release cùng các module khác trong `platform-java`.
- Truy vấn SQL viết tay vẫn là chỗ có thể rò dữ liệu. ArchUnit và RLS giảm rủi ro này nhưng không thay được việc review
  code.
- Dữ liệu cũ được gán vào tenant `default` bằng migration khi bật tenant cho từng service.
- Mặc định timeout có thể làm lộ ra các lời gọi chậm vốn đang chạy được. Service cần tăng timeout theo từng client nếu
  có lý do.
- Chưa làm: chia schema hoặc database theo tenant (chỉ xét khi có khách hàng yêu cầu cách ly vật lý), và di chuyển dữ
  liệu của một tenant giữa các cụm.
