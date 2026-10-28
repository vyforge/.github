# platform-java 0.15.0

Bản này gộp mốc tích hợp 2 của các module dùng chung, cùng một đợt sửa độ tin cậy. Đợt sửa này làm cho kết quả nghiệp vụ và
bộ test không còn phụ thuộc vào máy chạy (múi giờ, độ chính xác đồng hồ, thứ tự chạy test).

**Có hai thay đổi hành vi cần làm khi nâng cấp.** Xem mục [Nâng cấp từ 0.14](#nâng-cấp-từ-014).

| | |
|---|---|
| Java | 25 |
| Spring Boot | theo `platform-bom` (không đổi so với 0.14) |
| Nâng cấp | đổi `platform-bom`/`platform-parent` sang `0.15.0`, rồi làm 2 bước ở mục Nâng cấp |
| Migration DB | chỉ có migration mới. Không sửa migration đã phát hành |

## Module mới

| Artifact | Dùng để làm gì |
|---|---|
| `platform-starter-automation` | Luật tự động "khi X thì Y" do tenant admin tự cấu hình. Trigger từ event công khai của module, thay đổi bản ghi, lịch theo ngày làm việc, mốc tương đối, nút bấm, webhook gọi vào. Điều kiện viết bằng CEL có sandbox, hành động đi qua SPI `AutomationAction`. Có chống vòng lặp, log từng lần chạy và chạy thử. |
| `platform-starter-esign` | Sinh tài liệu từ template (PDF form, DOCX, HTML), phong bì ký nhiều bên (tuần tự hoặc song song). Hỗ trợ ba mức chữ ký, ký số từ xa qua adapter CA Việt Nam (VNPT SmartCA, Viettel MySign, chuẩn CSC), PAdES có TSA/LTV, kiểm tra chữ ký (đối chiếu DSS) và biên bản ký. Không giữ khoá bí mật của người dùng. |
| `platform-starter-billing` | Gói giá, subscription, đo mức dùng, hoá đơn bán hàng, thuế, coupon, dunning. |
| `platform-starter-einvoice` | Hoá đơn điện tử Việt Nam qua adapter nhà cung cấp (Viettel, MISA). Hiện adapter mới chỉ được kiểm bằng mock, chưa thử với môi trường thật của nhà cung cấp. |
| `platform-starter-ops` | API cho bảng điều khiển vận hành (ops-console): việc nền của mọi tenant, outbox, dead-letter Kafka, job định kỳ, cache theo tenant. |
| `platform-report`, `platform-report-openhtmltopdf` | Bộ dựng tài liệu dùng chung (`ReportEngine`): template theo tenant có version, html-pdf (Thymeleaf, font Be Vietnam Pro), docx, docx-pdf qua SPI `DocumentConverter`, in hàng loạt ra ZIP hoặc một PDF gộp, ký PAdES. OpenHTMLtoPDF (LGPL-2.1+) nằm riêng trong `platform-report-openhtmltopdf`, sau SPI `HtmlPdfConverter`, và đang chờ pháp chế xác nhận. |
| `platform-starter-cms-content` | Headless CMS: content type định nghĩa động, editor theo block, đa ngôn ngữ, draft → review → publish, lên lịch, version, preview, delivery API có cache/ETag, SEO, sitemap, redirect. |
| `platform-starter-tenant-lifecycle` | Tự đăng ký, khởi tạo tenant theo saga, chọn gói, tạm ngừng, offboarding. Có protocol để các module tham gia (participant) và bridge sang features, billing, privacy-dsr. |
| `platform-services` | Backend dùng chung nhúng các starter, bật/tắt từng module, gắn entitlement và nhóm OpenAPI theo module. |

## Tính năng mới trong module sẵn có

- **authz: Permission boundary.** `PermissionBoundary` là một tập policy giới hạn quyền tối đa của một principal, tương
  tự permission boundary của AWS IAM. SPI là `PermissionBoundaryProvider`. Request chỉ được phép khi policy thường cho
  phép **và** mọi boundary đều cho phép; `Deny` tường minh ở bất kỳ đâu thì thắng.
  - Áp dụng cho `check`/`allowed`, `@RequiresPermission`, `hasPermission`, `holds` và Policy Simulator. Simulator nhận
    claim của token và boundary chưa lưu.
  - Có metric `vyforge.authz.boundary.denied`. Ở remote mode, boundary của provider cục bộ được áp như local mode.
  - Dùng trong test: `TestAuthz.Builder.boundaries`.
- **api-access: `ScopeBoundaryProvider`.** Biến scope của token, của installation và của key thành boundary cho app
  token. Key bị thu hồi hoặc hết hạn, installation không hoạt động, consent hết hạn, app bị đình chỉ: mọi hành động đều bị
  chặn. Bật/tắt bằng `vyforge.api-access.authz.permission-boundary`.
- **scheduling:**
  - `TaskOps` thao tác việc nền của mọi tenant: nhóm lỗi theo cột mới `last_error_type`, retry hàng loạt có rải
    `next_run_at`, chỉ cancel/purge hàng DEAD/CANCELLED.
  - `QueueControl` tạm dừng một queue, một tenant hoặc một kind đến một thời điểm.
  - `keep-dead` (90 ngày) dọn task DEAD bị bỏ quên.
  - Lịch sử chạy của `@ClusterScheduled` lưu ở `vyforge_job_run` (schema starter `job`). Người vận hành có thể tạm dừng
    job, chạy ngay cho các tenant chọn trước, nhả khoá shedlock bị kẹt.
- **messaging:** `OutboxOps` đọc, gửi lại, park và khôi phục event của outbox. `DeadLetterIngester` nạp `<topic>-dlt`
  vào `vyforge_dead_letter`; replay có đích (`ReplayTargetInterceptor` nằm trong `RecordInterceptor` mặc định).
- **core:** `TenantCacheAdmin` xoá cache của một tenant (Caffeine; Valkey/Redis dùng SCAN + UNLINK, đồng bộ giữa các
  replica qua kênh `vyforge:cache:evict`). Thêm kiểu dùng chung `CursorSlice` và `BulkOutcome` cho các service vận hành.
- **einvoice, billing, notification, cms-content:** dựng PDF qua `ReportEngine` dùng chung.
- **api-access, organization, tenant-lifecycle, ekyc:** gọi các endpoint mới của auth-server.

## Sửa lỗi quan trọng

**Header và thứ tự**
- **security: race của header.** Security header được ghi ngay trong mọi filter chain. Trước đây, với request async
  (export trả 202, SSE), header có thể bị bỏ sót hoặc lỗi "response committed".
- **export:** hết race của header khi trả 202.
- **realtime:** thứ tự fan-out qua Valkey đúng.

**Việc nền**
- **scheduling: `QueuePauses` mất lệnh tạm dừng.** Worker đang nạp lại bảng tạm dừng có thể ghi đè `loadedAt` sau
  `invalidate()` của người vận hành. Kết quả là worker giữ cache cũ suốt `control-cache` và vẫn nhận task của queue đã
  tạm dừng.
- **search: generation dựng lại bị FAILED khi tài liệu lỗi được sửa trong lúc kiểm.** Bước kiểm số tài liệu trừ đi con
  số tài liệu bị từ chối *lúc quét*. Nếu tài liệu đó được sửa và retry trong lúc đang kiểm, generation lệch 1 và bị
  đánh FAILED, index không còn generation active. Giờ bước kiểm chấp nhận cả khi khớp với số thay đổi DEAD hiện tại.

**Thời gian**
- **features: subscription không bao giờ hết trial, quá hạn hay bị treo trên Linux.** `Instant` trên Linux có nano
  giây, còn Postgres lưu micro giây, nên mốc milestone trong task không bao giờ `equals` với hàng đọc lại từ DB. Giờ
  hai mốc được so trong phạm vi 1 µs. Lỗi này chỉ lộ trên Linux; macOS chỉ có micro giây.
- **Partition theo tháng tính theo múi giờ của máy.** PgJDBC đặt `TimeZone` của session bằng múi giờ JVM, nên literal
  ngày không có offset bị Postgres hiểu theo giờ máy.
  - ai, notification, search và ekyc giờ dùng mốc UTC tường minh.
  - privacy-dsr có migration `V2__privacy_partitions_utc.sql` thay hàm `privacy_ensure_partitions`. Tháng nào chồng lên
    partition cũ (tạo bởi session ở múi giờ âm) thì được bỏ qua, phần hở nằm trong partition default.
  - Partition đã tạo giữ nguyên, không mất dữ liệu.
- **Ngày "hôm nay" theo múi giờ của máy** trong sequence (ví dụ pattern, fallback ngày chứng từ remote), master-data
  (FX, geo), ekyc (thế kỷ năm sinh của MRZ), calendar (năm mặc định của lịch quốc gia), forms (tên file CSV), search
  (cửa sổ truy vấn phổ biến). Tất cả giờ dùng múi giờ của tenant/module (mặc định `Asia/Ho_Chi_Minh`) và `Clock` inject
  được.

**Độ tin cậy của bộ test** (chỉ ảnh hưởng người build platform)
- Các lớp test để lại trạng thái trong DB dùng chung cho lớp chạy sau, nên chỉ hỏng khi thứ tự lớp khác
  thứ tự chữ cái (Linux):
  - privacy-dsr: contributor `http://hrm` không tồn tại.
  - export: worker của context `@SpringBootTest` được cache nhận task của lớp khác.
  - collaboration: ngưỡng báo cáo bị nâng.
- Test so mốc thời gian đọc từ DB giờ dùng độ chính xác của DB.

## Nâng cấp từ 0.14

### 1. Cron mặc định chạy theo giờ Việt Nam (`vyforge.scheduling.zone`)

**Trước:** `@ClusterScheduled(cron = …)` không khai báo `zone` thì chạy theo múi giờ của JVM. Máy dev ở giờ Việt Nam và
máy chủ/Kubernetes ở UTC chạy cùng một cron vào hai giờ khác nhau.

**Nay:** `vyforge.scheduling.zone` mặc định là `Asia/Ho_Chi_Minh`. `ZoneId.systemDefault()` không còn được dùng.
- Job có `zone = "…"` riêng: không đổi.
- Job `fixedDelay`: không đổi.

**Ảnh hưởng:** trên máy chủ UTC, mọi cron không khai báo zone giờ chạy **sớm hơn 7 giờ theo UTC**, đúng giờ Việt Nam ghi
trong biểu thức. Ví dụ `0 0 7 * * *` trước chạy 07:00 UTC (14:00 giờ VN), nay chạy 07:00 giờ VN. Job của chính platform
bị ảnh hưởng (đều là việc bảo trì, chạy sớm vài giờ không sai nghiệp vụ):

| Nhóm | Job (cron mặc định) |
|---|---|
| Tạo partition | `ai-partitions`, `billing-usage-partitions`, `ekyc-event-partitions`, `features-usage-partitions`, `notification-partitions`, `privacy-partitions`, `sequence-partitions`, `workflow-partitions` |
| Retention, dọn dẹp | `ai-retention`, `billing-usage-dedupe-cleanup`, `calendar-reminder-purge`, `cms-revision-pruner`, `features-cleanup`, `files-cleanup`, `files-retention`, `masterdata-geocode-cache-purge`, `notification-retention`, `privacy-housekeeping`, `sequence-retention` |
| Xoay khoá | `export-rotate-secrets`, `files-rotate-secrets`, `notification-rekey`, `payment-rekey` |
| Khác | `calendar-national-notices` (07:00, nay đúng 07:00 giờ VN), `files-reconcile-quota`, `payment-sweep`, `sequence-reconcile` |

**Cần làm:**
1. Rà các `@ClusterScheduled(cron = …)` của dịch vụ mà không có `zone`. Biểu thức đang viết theo giờ VN thì không cần
   làm gì (trước đây trên máy chủ UTC chúng chạy lệch, nay chạy đúng).
2. Muốn giữ đúng hành vi cũ trên máy chủ UTC thì đặt:
   ```yaml
   vyforge:
     scheduling:
       zone: UTC   # hoặc múi giờ JVM đang dùng
   ```
   Hoặc khai báo `zone` riêng cho từng job.
3. Kiểm lại trên ops-console: `JobInfo.zone` và `nextRunAt` của từng job (`GET /api/v1/ops/admin/jobs`).

### 2. Luật kiến trúc `NO_SYSTEM_TIME_ZONE` nằm trong `PlatformArchitectureRules.ALL`

**Nay:** `ALL` (platform-test) cấm các lời gọi dùng múi giờ mặc định của JVM:
- `now()` không tham số của `LocalDate`, `LocalDateTime`, `LocalTime`, `ZonedDateTime`, `OffsetDateTime`, `OffsetTime`,
  `YearMonth`, `Year`, `MonthDay`;
- `ZoneId.systemDefault()`, `Clock.systemDefaultZone()`, `TimeZone.getDefault()`.

Dịch vụ có test kiến trúc dùng `ALL` (template `template-spring-boot` sinh sẵn) sẽ **lỗi build** nếu code chính còn các
lời gọi này. Lỗi ghi rõ method và dòng, ví dụ:
`InvoiceService.dueDate() calls LocalDate.now() with the JVM default time zone (InvoiceService.java:42)`.
`platform-auth-server` và template hiện không có vi phạm.

**Sửa theo hướng:** ngày và kỳ nghiệp vụ tính theo múi giờ tường minh, với `Clock` inject được.

```java
// trước
LocalDate today = LocalDate.now();

// sau: Clock là bean (test thay bằng Clock.fixed), zone của tenant hoặc cấu hình module
LocalDate today = LocalDate.now(clock.withZone(zone));   // zone mặc định ZoneId.of("Asia/Ho_Chi_Minh")
Instant now = clock.instant();                            // mốc thời gian tuyệt đối: dùng Instant
```

Chỗ cố ý dùng giờ máy (log, tên file tạm cục bộ, trang chẩn đoán) thì đánh dấu kèm lý do. Annotation áp được cho method,
constructor hoặc class; lambda được tính theo method chứa nó:

```java
@SystemTimeZoneExempt("tên file dump cục bộ theo giờ máy")   // com.vyforge.platform.core
String dumpName() { return "dump-" + LocalDateTime.now() + ".hprof"; }
```

**Chuyển dần** (dịch vụ có nhiều chỗ cần sửa): tạm thay `ALL` bằng các luật thành phần, và freeze riêng luật này để chỉ
chặn vi phạm mới:

```java
@ArchTest
static final ArchRule platform = CompositeArchRule.of(PlatformArchitectureRules.NO_FIELD_INJECTION)
        .and(PlatformArchitectureRules.NO_REST_TEMPLATE)
        .and(PlatformArchitectureRules.NO_COMMON_OR_UTIL_PACKAGES)
        .and(PlatformArchitectureRules.CONTROLLERS_IN_WEB_PACKAGES)
        .and(PlatformArchitectureRules.CONTROLLERS_DO_NOT_USE_REPOSITORIES)
        .and(PlatformArchitectureRules.DOMAIN_IS_FRAMEWORK_FREE)
        .and(GeneralCodingRules.NO_CLASSES_SHOULD_USE_JAVA_UTIL_LOGGING)
        .and(GeneralCodingRules.NO_CLASSES_SHOULD_ACCESS_STANDARD_STREAMS)
        .and(GeneralCodingRules.NO_CLASSES_SHOULD_USE_JODATIME);

@ArchTest
static final ArchRule timeZone = FreezingArchRule.freeze(PlatformArchitectureRules.NO_SYSTEM_TIME_ZONE);
```

Sửa xong thì quay lại dùng `ALL`.

### Khác

- **Migration mới** (Flyway của từng starter tự chạy, hoặc file gộp `META-INF/vyforge/sql/*.sql` cho DBA khi
  `schema.migrate=false`):
  - `billing V1`, `einvoice V1`, `cms V1`, `tenant-lifecycle V1`;
  - `messaging V1` (ops);
  - `job V1`, `task V4` (cột `last_error_type`, bảng tạm dừng);
  - `privacy V2` (partition UTC).
- **export:** các kiểu public cũ vẫn còn dưới dạng alias `@Deprecated` (`spi.ReportEngine`, `ReportTemplate`,
  `DocumentSigner`, `report.HtmlPdfRenderer`, `VnFormat`, `ThymeleafPdfEngine`, `Pkcs12PadesSigner`). Hãy chuyển sang
  `com.vyforge.platform.report.*`; các alias sẽ bị xoá ở một bản sau.
- **OpenHTMLtoPDF (LGPL-2.1+):** chỉ còn nằm trong `platform-report-openhtmltopdf`. Starter export kéo artifact này vào
  sẵn. Dịch vụ không muốn có thành phần LGPL thì exclude nó và cung cấp bean `HtmlPdfConverter` khác.
- **Kiểm múi giờ cho người build platform:** profile Maven `timezones` chạy lại test ở `UTC`, `Asia/Ho_Chi_Minh`,
  `Pacific/Kiritimati` (UTC+14) và `Pacific/Pago_Pago` (UTC-11):
  ```sh
  ./mvnw verify -Ptimezones -pl <module> -Dtest='<lớp>' -Dsurefire.failIfNoSpecifiedTests=false
  ```
  `RepositoryTimeZoneTests` áp luật `NO_SYSTEM_TIME_ZONE` cho toàn bộ mã nguồn (main và test) của repo.
