# ADR-0020: Module ứng dụng dùng chung

- **Trạng thái:** Chấp nhận dần theo từng module (mỗi module một mục). Thuộc kế hoạch trong
  [MODULE_PLAN.md](../MODULE_PLAN.md) mục 4 và 8.
- **Nền chung:** module-conventions của đợt P1; khối dùng chung `SecretCipher`, `DurableTaskQueue`,
  `StarterSchemaMigrator` (platform-java 0.12.1); đóng gói độc lập và admin microfrontend theo
  [platform-services](../design/platform-services.md).

Mỗi module ghi: quyết định, phương án đã loại, hệ quả, chỗ bản triển khai lệch thiết kế, việc còn mở. Thiết kế chi tiết
nằm ở `docs/design/<module>.md`. Module lớn có ADR riêng: workflow ([ADR-0024](0024-workflow-flowable.md)), ai
([ADR-0025](0025-ai-gateway.md)), api-access ([ADR-0026](0026-api-access.md)), organization
([ADR-0022](0022-shared-organization-structure.md)).

Quy ước chung mà mọi module dưới đây đã theo, nên các mục không nhắc lại:

- Lưu trữ bằng `JdbcClient` (không entity JPA) để không lẫn vào `EntityManager` của app; mọi phương thức JDBC nhận
  `tenantId` (ArchUnit `TENANT_ISOLATION`), bảng dùng chung đánh dấu `@TenantExempt`.
- Schema riêng qua `StarterSchemaMigrator` (`flyway_schema_history_<module>`). Tên StarterSchema chỉ nhận
  `[a-z][a-z0-9]*`, nên module có gạch ngang dùng tên viết liền (`customfields`, `tenantlifecycle`, `apiaccess`); cờ tắt
  migrate vẫn theo tên có gạch ngang.
- Module khác chỉ nối qua SPI + `ObjectProvider`/`@ConditionalOnBean`, kèm bản thay thế tối thiểu; không chép code.
- Mã lỗi `<module>.<tên>` chữ thường, message vi/en.
- Job định kỳ không dùng `@ClusterScheduled(perTenant = true)` khi không bắt buộc app có `TenantDirectory`: job toàn cục
  tìm tenant từ chính dữ liệu của module.
- Admin MFE dựng bằng `pnpm create-admin-zone <module>`; thao tác nhạy cảm gọi `requireStepUp()`. Component nhận quyền
  dạng map hoặc mảng action, vì Server Component không truyền được hàm `can` sang Client Component.

## Mục lục

[jpa-extras](#jpa-extras) · [admin-kit](#admin-kit) · [platform-expression](#platform-expression) ·
[forms](#forms) · [custom-fields](#custom-fields) · [sequence](#sequence) · [master-data](#master-data) ·
[calendar](#calendar) · [settings](#settings) · [licensing](#licensing) · [privacy-dsr](#privacy-dsr) ·
[collaboration](#collaboration) · [search](#search) · [ekyc](#ekyc) · [esign](#esign) ·
[report-engine](#report-engine) · [ops-console](#ops-console) · [cms-content](#cms-content) ·
[tenant-lifecycle](#tenant-lifecycle)

## jpa-extras

- **Thiết kế:** [jpa-extras.md](../design/jpa-extras.md). **Phát hành:** `platform-core` (`VietnameseText`,
  `EntityRef`), `platform-starter-jpa`, hai job ở `platform-starter-scheduling`, `Resources.Ref` deprecated ở
  `-authz` (platform-java 0.14.0); `@vyforge/trash` 0.1.0, `@vyforge/i18n` 0.2.0, `@vyforge/api-client` 0.6.0,
  `@vyforge/ui` 0.5.0; zone `/m/trash`.

### Quyết định

1. **Không có starter mới.** `LocalizedText`, xoá mềm, tìm tiếng Việt, mã hoá cột vào `-jpa`; `VietnameseText` và
   `EntityRef` vào `platform-core` vì search, collaboration, files, workflow dùng mà không phải lúc nào cũng có `-jpa`.
2. `LocalizedText` là value object lưu một cột `jsonb`; cùng một quy tắc fallback ở Java, SQL (`vyforge.localized`) và
   TypeScript (`pickLocalized`), có bộ mẫu chung (72 ca) giữa Java và TS.
3. Xoá mềm bằng Hibernate filter (`autoEnabled`, `applyToLoadByKey`) và service `SoftDeletes`; `remove()` trên entity xoá
   mềm bị chặn trừ trong `SoftDeletes.purge…`.
4. Hàm SQL nền trong schema cố định `vyforge` (`immutable_unaccent`, `fold`, `localized`, FTS `vyforge.vi`), cài bằng
   kết nối của Flyway trước migration của app; cần PostgreSQL 15+.
5. `VietnameseText.fold` đọc chính `unaccent.rules` của PostgreSQL 18 (nhúng vào jar); test hợp đồng Java = SQL trên
   toàn bộ 2.661 ký tự nguồn, 0 lệch.
6. `@Encrypted` mã hoá bằng `SecretCipher` với AAD theo purpose và tenant; tra cứu qua `@BlindIndex` HMAC, không mã hoá
   deterministic.
7. Thùng rác dùng chung: REST `/api/v1/trash`, `@vyforge/trash`, zone `trash-admin`; xoá vĩnh viễn phải gõ lại nhãn.

### Phương án đã loại

- `@SoftDelete` của Hibernate: còn `@Incubating`, không ghi người xoá/lý do, không đọc lại được dòng đã xoá.
- Cột theo ngôn ngữ (django-modeltranslation) hoặc bảng dịch (ABP, django-parler): đổi schema khi thêm ngôn ngữ hoặc
  join mỗi truy vấn.
- Mã hoá deterministic để tìm theo giá trị: không xoay khoá được.
- Chép tay quy tắc bỏ dấu: sẽ lệch với SQL.

### Hệ quả

- Migration của app và module khác viết tên hàm đầy đủ `vyforge.*`.
- Event xoá mềm mang `tenantId` nhưng chưa implement `TenantAwareEvent` (nằm ở `-messaging`).
- Join bị Hibernate tối ưu bỏ thì filter không áp (ghi trong README).

### Lệch so với thiết kế

- Bỏ thuộc tính `vyforge.jpa.text.schema` (schema cố định), thêm `vyforge.jpa.text.enabled`.
- Mã lỗi theo quy ước repo (`jpa.restore_conflict`…), không phải `JPA_RESTORE_CONFLICT`.
- Job hai lượt: bean theo tenant chỉ tạo khi có `TenantDirectory`; bảng không có `tenant_id` chạy ở bean `…-shared`.
  Purge dừng khi một lô không purge được dòng nào (tránh lặp vô hạn); xoay khoá không có `max-per-run`.
- `Resources.Ref` là `@Deprecated` + chuyển đổi sang `EntityRef` (record không kế thừa record).
- Bộ lọc thùng rác chỉ có `q`, `deletedBy` (chưa có khoảng ngày).
- Sửa lỗi thật: bean `EncryptionRotation` tách sang auto-config chạy sau Hibernate.

### Còn mở

- Purger bỏ qua id đã veto (cursor `(deleted_at, id)`); gauge `trash.size`; khoảng ngày cho thùng rác.
- Nhiều app có thùng rác cần shell hỗ trợ nhiều mục registry cho cùng một zone.
- Chuyển `TenantAwareEvent` vào `platform-core`.

## admin-kit

- **Thiết kế:** [admin-kit.md](../design/admin-kit.md). **Phát hành:** phần backend trong `-web`, `-audit`,
  `-scheduling` (platform-java 0.14.0); `@vyforge/admin-kit` 0.1.0; sub-template Copier `resource` (template-spring-boot).

### Quyết định

1. Một package web, không starter mới, không MFE riêng. Backend nhỏ vào starter sẵn có: hợp đồng danh sách, cursor,
   bulk, `allowedActions` (`-web`); `changes` và endpoint audit theo module (`-audit`); endpoint việc nền dùng chung
   (`-scheduling`, chung hợp đồng với ops-console).
2. Dữ liệu một chiều URL → Server Component → API → props (nuqs, `createLoader`); một định nghĩa cho danh sách, lọc, xuất.
3. Hai kiểu phân trang: offset và `CursorPage`. Tham số lọc phẳng có kiểu + `SortWhitelist`.
4. Bulk `POST …/bulk/{op}` nhận `ids` hoặc `filter` + `expectedCount`; từ 500 dòng hoặc theo bộ lọc thì thành việc nền
   (`bulk.<resource>.<op>`), đếm lại với dung sai 1%. Audit cả lô `<service>:<resource>:Bulk<Verb>`.
5. `-web` không gọi thẳng `Authz` (authz phụ thuộc web): port `ActionChecker`, `BulkTaskSubmitter`, `BulkAuditSink` ở
   `-web`, cài ở `-authz`, `-scheduling`, `-audit`.
6. `TaskAdminEndpoint`, `ModuleAuditEndpoint` đăng ký method vào `RequestMappingHandlerMapping` theo `basePath`, để
   Problem Details hoạt động như controller thường và hai module chung một server có đường dẫn riêng.

### Phương án đã loại

- react-admin, Refine: data provider chạy ở trình duyệt gọi API trực tiếp, trái mô hình BFF và token chỉ ở server
  (ADR-0006).
- Ngôn ngữ lọc chung kiểu RSQL/JHipster Criteria: sinh truy vấn không index, lộ cột không được phép.
- Engine form thứ hai: `ResourceForm` chế độ schema bọc `SchemaForm` của `@vyforge/forms`.

### Hệ quả

- `PlatformTaskProperties.progressMinInterval` (mặc định 1 giây), `TaskInfo` thêm `progress`.
- Nhãn kit qua `adminKitMessages` + prop `labels`, không bắt buộc next-intl; định dạng bằng `Intl`.

### Lệch so với thiết kế

- `BulkOperation` khai báo `queue()` (thiết kế để queue là tham số của `execute`) để đăng ký handler lúc khởi động.
- Sub-template bỏ qua (kèm cảnh báo) field `localized`, `masterdata`, `attachments`, `extensible`, `number`,
  `softDelete`; enum lưu varchar.
- Năm component chung của admin-shell (`TaskQueueTable`, `AuditTrail`, `SecretField`, `JsonSchemaForm`) chuyển sang
  admin-kit; `TenantPicker` ở lại admin-shell.

### Còn mở

- Phần frontend của sub-template (template-nextjs); chuyển trang users/audit của platform-admin sang admin-kit.
- Reveal dữ liệu cá nhân có audit (`RevealPersonalData`); Storybook.

## platform-expression

- **Thiết kế:** mục biểu thức trong [automation.md](../design/automation.md). **Phát hành:** `platform-expression` và
  `TaskContextPropagator` (`-scheduling`) (platform-java 0.14.0).

### Quyết định

1. Một ngôn ngữ biểu thức cho workflow, automation, forms phía server: **CEL** (`dev.cel`, Apache-2.0) có sandbox,
   biên dịch trước, giới hạn kích thước kết quả.
2. `Conditions` biên dịch điều kiện dạng dữ liệu (như forms) sang CEL, dùng `dyn(field)` + `has()`.
3. Module thêm nhóm hàm qua SPI `ExpressionFunctions`.
4. `TaskContextPropagator` truyền ngữ cảnh (chuỗi, vd. `automation.chain`) qua việc nền; cấu hình MDC tách riêng
   (`vyforge.task.context`).

### Phương án đã loại

- JUEL/SpEL cho người dùng cuối: gọi được bean Spring.
- Caffeine làm cache: lệch version `error_prone_annotations` với Guava của cel-java, vỡ `dependencyConvergence`; dùng Guava.

### Hệ quả

- `null` trong Map coi như thiếu trường (`has()` là false), vì cel-java trả `CelUnknownSet` thay vì lỗi.
- So sánh `int`/`double` được, phép số học trộn kiểu phải viết `double(x)`.

### Lệch so với thiết kế

- `format(value, pattern)` tách thành `formatMoney`, `formatNumber`, `formatDate`.
- `changed(field)` do automation đăng ký, `businessDaysBetween` thuộc nhóm `calendar`; không nằm trong `core`.
- Thêm `maxCollectionSize` (10.000) và `timeout` (2 giây) vào `ExpressionLimits`.
- Giá trị propagator là `String`, không phải JSON tuỳ ý.
- Chưa có luật ArchUnit "chỉ `..expression..` import `dev.cel..`"; SPI vẫn lộ `CelFunctionDecl`.
- Bỏ dấu qua interface `TextFolder` (chưa dùng `VietnameseText` lúc làm).

### Còn mở

- Phía TS của forms vẫn tự đánh giá điều kiện; nên dùng chung bộ ca `conditions.cases.json`.
- Propagator trace W3C khi observability chốt API tracing.

## forms

- **Thiết kế:** [forms.md](../design/forms.md). **Phát hành:** `platform-forms-core`, `platform-starter-forms`
  (platform-java 0.14.0); `@vyforge/forms` 0.1.0; zone `/m/forms`.

### Quyết định

1. Tự xây trên JSON Schema: kiểm server bằng networknt json-schema-validator, renderer rjsf 6 (validator không `eval`).
   `FieldSpec`/`FieldType` của `platform-forms-core` là mô hình kiểu chung cho forms, custom-fields, admin-kit.
2. `fieldKey` bất biến qua mọi version; cài đặt nhận bài là cột có kiểu.
3. Điều kiện là dữ liệu có cấu trúc (tập toán tử kiểu SurveyJS), không phải chuỗi biểu thức.
4. Đồng ý không tick sẵn ở hai lớp: compiler bỏ `default`, linter báo lỗi; renderer tắt `constAsDefaults`.
5. Renderer import thẳng `@rjsf/core/lib/components/Form.js` để không kéo AJV (cần `unsafe-eval`, vỡ CSP).
6. Điểm nối qua SPI: `FormUploads` (files), `FormsNotifier`, `FormsWebhookPublisher`, `ConsentRecorder` (privacy-dsr),
   `SubmissionQuota` (features), `ReferenceNumberGenerator` (sequence), `HumanVerification` (antibot), `ChoiceSource`
   (master-data, organization). Thiếu module thì có số tham chiếu `short-code`/`form-counter`, consent trong
   `meta.consent`, xuất CSV.

### Phương án đã loại

- Formbricks/OpnForm tự host: AGPL + EE, ánh xạ user/tenant riêng, thêm một ứng dụng phải vận hành.
- SurveyJS: Creator thương mại; phải viết lại bộ đánh giá biểu thức bằng Java.
- JSON Forms 3: so sánh ở thiết kế §2.4, chọn rjsf.

### Lệch so với thiết kế

- Điều kiện chỉ tham chiếu field đứng trước (chặt hơn "trước hoặc cùng trang"), để không có vòng.
- Builder thêm field bằng palette (bấm), kéo để đổi thứ tự (dnd-kit, có bàn phím); không kéo từ palette vào canvas.
- Mã `forms.field.*`, `forms.spec.*` khai báo thành `ErrorCode`.

### Còn mở

- `mode: remote`, tra cứu biên nhận kèm OTP, bulk, báo cáo xuyên tenant, JSON Schema thô, CORS theo
  `allowedEmbedOrigins`, widget ALTCHA (chờ antibot, đã hoãn), widget địa chỉ VN, `EXTENDABLE`, test tải.
- cms-content gửi `publicId` thay `formId`; các module kia cài cổng tương ứng.

## custom-fields

- **Thiết kế:** [custom-fields.md](../design/custom-fields.md). **Phát hành:** `platform-starter-custom-fields`, hợp
  đồng `Extensible` ở `platform-core`/`-jpa` (platform-java 0.14.0); `@vyforge/custom-fields` 0.1.0 và zone
  `/m/custom-fields` xong trên nhánh, chờ merge.

### Quyết định

1. Giá trị ở `attributes -> 'cf'` (object phẳng), không thêm cột, không EAV; một GIN `jsonb_path_ops` cho mọi trường,
   chỉ mục biểu thức partial theo tenant cho trường cần sắp xếp/lọc khoảng.
2. `platform-forms-core` là phụ thuộc cứng (một mô hình kiểu, cùng mã lỗi `forms.field.*`).
3. Kiểm "khi chạm": bản ghi cũ chỉ lỗi ở trường bị thay đổi.
4. Quyền theo trường bằng action IAM; trường `SENSITIVE` bắt buộc có action riêng.
5. Đổi kiểu, điền mặc định, dọn trường là migration trên `DurableTaskQueue` (keyset, dry run, hoàn tác 30 ngày); xoá
   trường là xoá mềm 15 ngày; không đổi tên khoá.
6. Luôn nhúng trong app sở hữu bảng, không có `mode: remote`.

### Phương án đã loại

- EAV: mọi truy vấn phải join/pivot.
- Thêm cột thật (DDL lúc chạy): cần quyền ALTER, khoá bảng.
- Chặn mọi lần lưu khi thêm trường bắt buộc (kiểu Salesforce).

### Lệch so với thiết kế

- `options.source` theo dạng chuỗi của forms (`lookup:bank`).
- Trường ẩn theo điều kiện giữ nguyên giá trị khi lưu.
- Bật/tắt mã hoá và `KEEP_AS_LEGACY` bị từ chối (bản sau).
- Trường `unique` ACTIVE ngay, chưa chờ chỉ mục READY.
- Lọc ngày `To` không gồm ngày đó (khớp admin-kit).
- Cấu hình theo tenant qua SPI `CustomFieldsSettings`; cache schema bằng `ConcurrentHashMap` + kiểm revision.

### Còn mở

- ENCRYPT/DECRYPT/REKEY, COPY, `conditionAttributes`, invalidation qua Valkey, header `Vyforge-Warnings`,
  `@ExtensibleView`, chỉ mục trên bảng phân vùng, bộ nối search/export/privacy-dsr/master-data/settings.

## sequence

- **Thiết kế:** [sequence.md](../design/sequence.md). **Phát hành:** `platform-starter-sequence` (platform-java 0.14.0);
  `@vyforge/sequence` 0.1.0; zone `/m/sequence`.

### Quyết định

1. Bộ đếm là dòng `seq_counter` khoá `(tenant, sequence, scope, period)`, tạo lười bằng một câu upsert.
2. Hai chế độ: `FAST` (transaction riêng, cấp theo khối, cho nhảy số) và `GAPLESS` (trong transaction nghiệp vụ).
3. Sổ cái `seq_issued`: số đã cấp không tái sử dụng; huỷ giữ số, ghi lý do. Đặt trước `reserve` → `confirm`/`void`.
4. Mẫu số là chuỗi token có version theo ngày hiệu lực; kỳ theo ngày chứng từ và múi giờ tenant. Phạm vi bộ đếm tách
   khỏi giá trị chỉ để hiển thị (số văn bản theo NĐ 30/2020).
5. Khai báo bằng catalog YAML; tenant ghi đè trong giới hạn catalog. Đặt lại chỉ qua `adjust` có lý do, MFA, tuỳ chọn
   bốn mắt, không thấp hơn số lớn nhất đã cấp.
6. `remote`: `GAPLESS` chuyển thành ngữ nghĩa `ACCOUNTED` (đặt trước + xác nhận trước commit + huỷ khi rollback).

### Phương án đã loại

- `SEQUENCE` của PostgreSQL: DDL lúc chạy cho hàng nghìn bộ đếm, không theo tenant, khó khôi phục từng tenant.
- Tái sử dụng số bị bỏ (D365 continuous): số đã in, đã gửi khách không được cấp lại.

### Lệch so với thiết kế

- Định nghĩa và mẫu dùng `JdbcClient` (thiết kế nói JPA); trường catalog khoá gộp lúc chạy.
- Thêm `DocumentNumbers.nextAll` (khoá nhiều bộ đếm theo thứ tự cố định), `find(id)`, `continuityCsv`; internal API
  nhận tenant qua `X-Tenant-Id`; catalog app remote lưu `seq_remote_catalog`.
- Token `affectsCounter` chỉ vào khoá bộ đếm khi mẫu dùng token đó.
- Settings chỉ đọc (PUT chờ module settings).

### Còn mở

- `@Externalized` Kafka, rate limit REST, `PersonalDataProvider`, `OrganizationScopeValueResolver`, detach partition
  rỗng; nút DISABLED/ARCHIVED trong UI.

## master-data

- **Thiết kế:** [master-data.md](../design/master-data.md). **Phát hành:** `platform-starter-master-data`,
  `tools/masterdata-packs` (platform-java 0.14.0); `@vyforge/master-data` 0.1.0; zone `/m/master-data`.

### Quyết định

1. Một lõi danh mục tổng quát (`list` → `item`) với facade có kiểu (`AdministrativeUnits`, `Banks`, `Countries`,
   `Currencies`, `PublicHolidays`, `ExchangeRates`…).
2. Hai trục thời gian: hiệu lực nghiệp vụ (`valid_*`) và bản phát hành (`rev_*`); đồ thị kế thừa (gộp, tách, đổi tên,
   đổi loại) cho ĐVHC 63→34 tỉnh, ngân hàng đổi tên, tiền tệ bị thay.
3. Thay đổi qua bản phát hành: nháp → validate → diff → (bốn mắt) → công bố, checksum JCS, event, audit, revert.
4. Dữ liệu hệ thống phát hành dạng data pack ký Ed25519 + JCS, cập nhật không cần deploy; seed đóng sẵn 9 pack (ĐVHC
   hiện hành và trước 01/07/2025 với 10.690 cạnh kế thừa, 76 ngân hàng, ISO quốc gia/tiền tệ/ngôn ngữ, lịch nghỉ).
5. Đọc từ snapshot bất biến trong bộ nhớ, ETag/304, đồng bộ Valkey + poll; `mode: remote`.
6. Địa chỉ VN là value object (`VnAddress`, `AddressColumns`), app tự lưu; geocoding/autocomplete là SPI; tỷ giá là
   chuỗi thời gian riêng có chính sách theo mục đích.

### Phương án đã loại

- Bảng địa chỉ trong module: thành kho PII thứ hai.
- Chờ release platform mỗi lần ĐVHC đổi.
- Index `pg_trgm` cho tìm danh mục: tìm trên snapshot.

### Lệch so với thiết kế

- Pack đóng trong starter chưa ký (tin theo classpath); pack upload/URL bắt buộc ký.
- Ràng buộc `btree_gist` tự bật/tắt trong V1 thay cho property `overlap-constraint`.
- `AddressColumns` lưu mã cũ ở ba cột `addr_legacy_*` thay vì JSONB.
- Export và nhập pack chạy đồng bộ; import CSV/JSON chạy nền. Seed danh mục rỗng áp thẳng vào `md_item`.
- Cho phép chữ hoa sau ký tự đầu trong mã danh mục (`holiday-day:VN-GOV`).

### Còn mở

- Adapter geo `vietmap`, `goong`, `google`; XLSX; danh mục VSIC, nghề nghiệp, dân tộc, đơn vị tính; tỷ giá `sbv`.
- Ký pack seed bằng khoá thật trong CI phát hành; pháp chế xác nhận dữ liệu phụ lục văn bản và BIN/SWIFT (Q4).

## calendar

- **Thiết kế:** [calendar.md](../design/calendar.md). **Phát hành:** `platform-starter-calendar` (platform-java 0.14.0);
  `@vyforge/calendar` 0.1.0 và zone `/m/calendar` xong trên nhánh, chờ merge.

### Quyết định

1. Lịch làm việc ba lớp: ngoại lệ của tenant → lịch quốc gia (kèm phương án nghỉ tenant chọn) → mẫu tuần.
2. Dữ liệu lịch nghỉ ở master-data, tính toán ở calendar (SPI `NationalHolidaySource`); không có master-data thì đọc
   pack VN/VN-GOV 2024–2027 cùng định dạng, mọi ngày có `legalRef`.
3. Âm lịch VN tự cài (Meeus), 1900–2199.
4. Lõi là hàm thuần trên lịch đã biên dịch, O(log n), không I/O; kết quả mang `CalendarVersion`.
5. Quy tắc đếm có tên: `LEGAL` (BLDS 2015 Điều 147–148), `SAME_TIME`, `CUTOFF`. Năm chưa có văn bản: dự đoán + cờ
   `provisional`, tuỳ chọn `CONSERVATIVE`.
6. Nhắc việc trên `DurableTaskQueue`; tick `BusinessDayStarted`… theo múi giờ và lịch của từng tenant; RRULE/ICS bằng
   ical4j 4.4.0 (BSD-3) ở server; sổ lịch gom nguồn qua `CalendarFeedContributor`.

### Phương án đã loại

- `cn.6tail:lunar` ở runtime: lịch Trung Quốc UTC+8, sai Tết 2030 và nhiều ngày Giỗ Tổ; chỉ dùng trong test đối chiếu.
- Spring Modulith Moments: chạy ở mọi instance, một múi giờ, không lưu mốc, không bù khi app tắt.
- Mở rộng RRULE ở frontend (`rrule` npm không cập nhật từ 11/2023).

### Lệch so với thiết kế

- Âm lịch theo múi giờ lịch sử: UTC+8 tới 07/08/1967, UTC+7 từ 08/08/1967 (thiết kế ghi UTC+7 cho mọi năm).
- Không dùng FullCalendar: 7.1.3 chưa có plugin daygrid/timegrid ổn định; tự dựng lưới tháng, xuất adapter
  `toExternalEvents`/`toBusinessHours`.
- Tick quét tenant có đăng ký thay vì `perTenant`; `created_at` theo `Clock` thay cho `CalendarTimeTravel`.
- Chưa có Valkey invalidation: revision nằm trong khoá cache, định nghĩa TTL 60 giây.
- `SAME_TIME` khi ngày đích khác giờ làm: kẹp vào giờ làm của ngày đó.

### Còn mở

- `mode: remote`; adapter master-data, organization, notification (`NotificationReminderChannel`), privacy-dsr; lặp
  âm lịch cho sổ lịch.
- Pháp chế xác nhận cách hiểu "N ngày làm việc" và giờ chốt nhận hồ sơ (Q5).

## settings

- **Thiết kế:** [settings.md](../design/settings.md). **Phát hành:** `core.settings` ở `platform-core`,
  `platform-starter-settings` (platform-java 0.14.0); `@vyforge/settings` và zone `/m/settings` xong trên nhánh, chờ
  merge.

### Quyết định

1. Hai lớp: API đọc/khai báo trong `platform-core` (Java thuần, không thêm dependency), starter thêm lưu DB theo tầng,
   ghi, audit, lịch sử, cache, REST, `remote`.
2. Định nghĩa bằng code (`SettingDefinitionProvider`); thứ tự đọc USER → TENANT → PLATFORM → `application.yml` → mặc
   định trong code.
3. Khoá và ràng buộc theo tầng cha (`locked`, `narrowOnly`, `atMost`, `oneOf`); ba chiến lược gộp OVERRIDE, COLLECT,
   DEEP_MERGE.
4. Một bảng giá trị + bảng revision chỉ thêm; khôi phục là ghi revision mới.
5. Giá trị nhạy cảm mã hoá `SecretCipher`, API không trả bản rõ hay bản mã; mức lộ PUBLIC/MEMBER/ADMIN.
6. Event `SettingChanged` không mang giá trị; ghi theo lịch (`effectiveAt`); danh mục `CoreSettings` dùng chung.

### Phương án đã loại

- Mỗi module tự giữ múi giờ, định dạng, thương hiệu.
- Một cờ `IsVisibleToClients` kiểu ABP: không đủ cho trang công khai của tenant.
- victools cho JSON Schema: `platform-core` không thêm dependency; schema dạng Map, record map bằng reflection.

### Lệch so với thiết kế

- `jpa.trash.retention.*` mặc định P30D với trần tuyệt đối P365D thay cho `atMost(cha)`.
- Path khoá/khôi phục `/values/{scope}/{key}/lock`, `/revisions/{id}/restore` thay cho `:lock`, `:restore`.
- `remote` đọc snapshot USER bằng client credentials (không token exchange); interceptor chỉ chạy ở app sở hữu định nghĩa.
- Giới hạn ghi USER in-memory theo node.

### Còn mở

- Resolver ORG_UNIT thật, auth-server đọc branding, duyệt bốn mắt, nối jpa-extras/privacy-dsr/admin-kit.

## licensing

- **Thiết kế:** [licensing.md](../design/licensing.md). **Phát hành:** `core.license`/`core.signing` ở `platform-core`,
  `platform-starter-licensing`, `tools/license-cli` (platform-java 0.14.0); `@vyforge/licensing` 0.1.0; zone
  `/m/licensing`.

### Quyết định

1. Tự xây lớp mỏng: file giấy phép JSON đọc được, ký Ed25519 trên JCS; hai tầng khoá (root nhúng, khoá phát hành theo
   năm trong OpenBao Transit, chứng chỉ do root ký đi kèm file).
2. Phần kiểm trong `platform-core`, không DB, không mạng; `@ConditionalOnLicensedModule` trên auto-config của starter
   tính phí.
3. Ba chế độ `LICENSED`/`DEV`/`UNLICENSED`; hết hạn không tắt dịch vụ đang chạy, hành vi sau hạn là chính sách trong
   file; chặn nâng cấp theo ngày phát hành dòng minor.
4. Đọc lại file mỗi 60 giây (gia hạn không cần khởi động lại); trần cho features qua `EntitlementCustomizer`.
5. Shell `platform-admin` tự kiểm file bằng `@vyforge/licensing/server`; CLI `vyforge-license` ký qua OpenBao, sổ giấy
   phép là repo `license-ledger` duyệt bốn mắt. Không phone-home.

### Phương án đã loại

- Keygen (FCL-1.0, cần server), license3j (định dạng nhị phân riêng).
- License server online: điểm chết chung, không chạy được air-gapped.

### Hệ quả

- Chưa nhúng khoá root thật (`roots.json`, `EMBEDDED_ROOTS` rỗng): tới buổi tạo khoá (Shamir 3/5) bản build không tin
  giấy phép nào ngoài test; chế độ dev vẫn chạy.
- `vyforge.release.line-released-on` trong `platform-core/pom.xml` phải chốt đúng ngày mỗi khi cắt dòng minor.

### Lệch so với thiết kế

- `restrict-admin-writes` và chặn module bị rút làm bằng servlet filter (order −90), không phải interceptor `-web`;
  module nhận diện qua `apiPrefix` trong `META-INF/vyforge/modules/<key>.json`.
- Thêm mã lý do `too_large`, `malformed`, `unsupported_format`, `unsupported_algorithm`, `bad_certificate`,
  `outside_signing_window`.
- Root test chỉ nhận từ `TrustAnchorProvider` trong `com.vyforge.platform.test`; nhiều `roots.json` thì lấy giao.
- TS không kiểm khoá trùng và số nguyên > 2^53 (lệch chỉ dẫn tới `bad_signature` thay vì `malformed`).
- Event chưa externalize ra Kafka.

### Còn mở

- Gắn `@ConditionalOnLicensedModule` + `META-INF/vyforge/modules/<key>.json` cho các starter tính phí (miễn phí:
  licensing, ops-console, settings, admin-kit).
- Buổi tạo khoá root; iam-server `LicenseUsageProvider`; trường `license` trong manifest admin-shell; `license-ledger`;
  hướng dẫn cấp/cài.

## privacy-dsr

- **Thiết kế:** [privacy-dsr.md](../design/privacy-dsr.md). **Phát hành:** SPI trong `platform-starter-privacy`,
  engine `platform-starter-privacy-dsr` (platform-java 0.14.0); `@vyforge/privacy` 0.1.0; zone `/m/privacy`.

### Quyết định

1. Hai artifact: SPI nhẹ (`PersonalDataContributor`) trong `-privacy`, engine có dữ liệu trong `-privacy-dsr`. Module tự
   khai báo và tự xử lý dữ liệu của mình; mỗi module chạy `ContributorTestKit.verify(...)`.
2. Taxonomy phân cấp khai báo trong code, ánh xạ NĐ 356/2025; consent record bất biến gắn version thông báo, receipt có
   chuỗi băm (băm trên JSON chuẩn hoá, dùng `subject_hash` để vẫn kiểm được sau ẩn danh).
3. Căn cứ pháp lý theo purpose; erasure theo loại dữ liệu (DELETE/ANONYMIZE/PSEUDONYMIZE/RETAIN).
4. State machine cố định có hook và event cho DSR; hạn tính qua cổng `WorkingDays`.
5. Q1/Q2/Q3/Q6/Q8 là cấu hình với mặc định của thiết kế; tenant chỉ siết chặt hơn hạn luật định.

### Phương án đã loại

- Fides hoặc connector đọc thẳng DB của module: phá ranh giới module (B6), Fides đã archive repo chính.
- SaaS consent nước ngoài làm mặc định (chuyển dữ liệu xuyên biên giới); có SPI `ConsentMirror` cho khách đã dùng.
- Tự dò PII trong DB; BPMN cho DSR; IAB TCF ở bản đầu.

### Hệ quả

- Mặc định `WeekendOnlyWorkingDays` chỉ trừ thứ Bảy, Chủ nhật; dashboard cảnh báo khi chưa nối lịch nghỉ lễ.
- Checklist vận hành cho dự án: DPO, lịch nghỉ lễ, backup và xoá, mẫu thư trả lời vi/en.

### Lệch so với thiết kế

- Thêm bảng `privacy_counter`, `privacy_consent_idempotency`, `privacy_request_blob`, `privacy_retention_review`,
  `privacy_remote_contributor` và cột `subject_hash`.
- Tên package `@vyforge/privacy` và zone `privacy` (`/m/privacy`) theo thiết kế, không theo brief.

### Còn mở

- Contributor của các module P1; adapter notification/files/calendar; màn DPIA, RoPA, so sánh version thông báo,
  ghi consent thay chủ thể, quyết định review retention; e2e.
- Câu hỏi pháp chế Q1, Q2, Q3, Q6, Q8 ([danh sách](../legal-open-questions.md)).

## collaboration

- **Thiết kế:** [collaboration.md](../design/collaboration.md). **Phát hành:** `platform-starter-collaboration`
  (platform-java 0.14.0); `@vyforge/collaboration` 0.1.0; zone `/m/collaboration` xong trên nhánh, chờ merge.

### Quyết định

1. Một module, năm khả năng gắn vào bản ghi bất kỳ (`SubjectRef`): trao đổi, dòng thời gian, việc cần làm, tag/follow/
   đã đọc, kiểm duyệt. Quyền đi theo bản ghi cha.
2. Hai lớp hiển thị `INTERNAL`/`PUBLIC`; thread hai tầng; nội dung là cây JSON schema hẹp (`vf-doc/1`), không lưu HTML;
   sửa có lịch sử, xoá là tombstone.
3. Thông báo qua notification (chỉ tính người nhận); realtime "báo rồi tải lại".
4. Kiểm duyệt OFF/POST/PRE, bộ lọc từ khoá tiếng Việt chống lách; hồ sơ nội dung công khai theo NĐ 147/2024 (tài khoản
   đã xác thực, hạn gỡ 24 giờ / 6 giờ / 3 giờ).
5. Mặc định nhúng trong app; `platform-services` dùng access callback hỏi quyền theo lô.

### Phương án đã loại

- Thread lồng sâu: khó đọc trên mobile, khó phân trang.
- Lưu HTML: XSS và khó kiểm schema.
- Tự gửi email: trùng việc với notification (digest).

### Lệch so với thiết kế

- Văn bản tạm ở `internal.text` cùng chữ ký `VietnameseText` (đổi sang core sau jpa-extras).
- `matchForm` thay ký tự lách theo từng phần, word list STRICT + kiểm dấu để tránh dương tính giả.
- Yêu cầu tài khoản xác thực chỉ áp cho bài PUBLIC; moderation OFF bỏ qua link/từ khoá/phân loại.
- Composer dùng Markdown; editor Tiptap để sau.

### Còn mở

- `mode: remote` và Kafka ingest; adapter thật cho files/notification/realtime; `PersonalDataContributor`; catalog
  search/webhooks; editor Tiptap; e2e.

## search

- **Thiết kế:** [search.md](../design/search.md). **Phát hành:** `platform-starter-search`, `platform-search-opensearch`,
  `platform-search-meilisearch` (platform-java 0.14.0); `@vyforge/search` 0.1.0 và zone `/m/search` xong trên nhánh,
  chờ merge.

### Quyết định

1. Có module search riêng (lệch MODULE_PLAN §4 "không cần module riêng"; K16 yêu cầu ADR cho engine riêng, mục này là
   ADR đó). PostgreSQL vẫn là engine mặc định; OpenSearch, Meilisearch là adapter tuỳ chọn.
2. Index là bản chiếu dựng lại được từ `SearchDocumentSource`; đánh chỉ mục bất đồng bộ qua `search_change` +
   `DurableTaskQueue` (postgres có `SYNC`); reindex blue/green theo generation.
3. Chuẩn hoá tiếng Việt ở Java, giống nhau cho mọi engine; không tách từ mặc định (SPI `WordSegmenter`).
4. Lọc quyền trong truy vấn: `TENANT`, `ACL`, `POLICY`; field restricted lưu lexeme có tiền tố `<field>~`.
5. Vector qua SPI `EmbeddingProvider` (ai cài); hybrid gộp ở Java bằng RRF k=60. Highlight trả vị trí, không HTML.
6. Adapter dùng Spring `RestClient` + Jackson tree, không dùng SDK.

### Phương án đã loại

- Elasticsearch, Typesense, ParadeDB, Manticore làm adapter của Vyforge: AGPL/SSPL/ELv2/GPL-3.0.
- Tách từ VnCoreNLP (GPL-3.0), CocCoc (LGPL-3.0, C++), underthesea (Python).
- SDK của engine: xung đột Jackson 3, thêm phụ thuộc.

### Lệch so với thiết kế

- Hai StarterSchema `search` (điều khiển) và `searchpg` (bảng engine postgres) thay cho migration V100 có điều kiện.
- Engine postgres không cần hàm SQL của core; phát hiện `pg_trgm` lúc chạy, pgvector chỉ dùng khi có.
- Đổi ACL thì upsert lại cả document; log truy vấn che PII qua `QueryLogSanitizer`; REPAIR tự dựng lại generation tối đa
  một lần mỗi phút.

### Còn mở

- `mode: remote`; index cho `sortIndexed`, BM25 trên postgres (đang dùng `ts_rank_cd`), k-NN OpenSearch,
  `RankingFunction`; nạp `VietnameseText` trực tiếp thay MethodHandle; adapter DSR.

## ekyc

- **Thiết kế:** [ekyc.md](../design/ekyc.md). **Phát hành:** `platform-starter-ekyc` (platform-java 0.14.0);
  `@vyforge/ekyc` 0.1.0 và zone `/m/ekyc` xong trên nhánh, chờ merge.

### Quyết định

1. Module (không chỉ adapter): flow có tên và version, kết quả theo capability đồng dạng `PASS/WARN/FAIL/NOT_EXECUTED`
   + mã lý do, nộp lại hay từ chối hẳn, hàng đợi duyệt tay trong starter.
2. Passive/active authentication của chip CCCD tự viết trên Bouncy Castle 1.86.
3. Kho bằng chứng qua SPI `EvidenceStore` (`jdbc` cho dev/test, `files` cho production); nội dung mã hoá envelope (DEK
   riêng mỗi object), xoá bằng crypto-shredding rồi purge.
4. Token chụp `<tenant>.<ngẫu nhiên>`, DB chỉ lưu hash; mỗi lời gọi provider mang consent và request id.
5. Java API là API tin cậy; quyền kiểm ở REST; ghi đè quyết định cần `ekyc:review:Override`.

### Phương án đã loại

- Tự chạy mô hình nhận dạng khuôn mặt, liveness: không có mô hình dùng thương mại có chứng nhận PAD (TT 77/2025).
- Tự kết nối RAR/C06: đặc tả không công khai.
- Nhúng Ballerine (ELv2, service Node riêng); JMRTD ở server (LGPL).

### Lệch so với thiết kế

- Bucket `ekyc-evidence` đặt `scan: none` (thiết kế: `required`): nội dung đã mã hoá nên AV không đọc được.
- Nộp lại chỉ chạy lại bước được yêu cầu, bước khác giữ kết quả (`carryOver`), vì challenge AA dùng một lần.
- Idempotency key của retry admin kèm số lần thử; bundle lỗi riêng `i18n/ekyc-errors`.
- Provider test bằng `MockRestServiceServer` (không WireMock); contract test adapter chưa vào platform-test.

### Còn mở

- Adapter workflow cho hàng đợi duyệt; adapter VNPT eKYC, Viettel; `@vyforge/ekyc-native`.
- Pháp chế dự án duyệt flow mẫu (`sbv-payment-account`…) trước khi dùng thật.

## esign

- **Thiết kế:** [esign.md](../design/esign.md). **Phát hành:** `platform-starter-esign`, `platform-starter-esign-dss`
  xong trên nhánh, chờ tích hợp sau report-engine; `@vyforge/esign` và zone `/m/esign` chờ merge.

### Quyết định

1. Mô hình Envelope → Document → Recipient → Field (thêm vai trò INITIALER cho ký nháy); ký từ xa hai pha
   prepare/complete ghim thời điểm ký và version tài liệu.
2. **Cô lập LGPL:** engine PAdES mặc định tự làm trên PDFBox + Bouncy Castle sau SPI `PadesEngine` (B-B, B-T, `/DSS`
   B-LT, dấu thời gian tài liệu B-LTA); bộ kiểm mặc định không LGPL. EU DSS chỉ ở `platform-starter-esign-dss`, app chủ
   động thêm.
3. Không tự dựng PDF từ template: template HTML/DOCX và biên bản ký dựng bằng ReportEngine dùng chung
   (`@ConditionalOnBean`). Không có ReportEngine thì chỉ có `pdf-form` và không có biên bản PDF.
4. Mỗi người nhận một chữ ký số trên mỗi tài liệu, trường ký nháy vẽ trong bản cập nhật trước chữ ký; một lần xác nhận ký
   lô mọi tài liệu. SES niêm phong bằng con dấu tenant.
5. Provider gọi trong việc nền; lấy chứng thư và kiểm OCSP/CRL trước khi ký (NĐ 23/2025 Điều 15) chạy đồng bộ.
6. Token link mã hoá `SecretCipher`; chữ ký thô của CA lưu mã hoá; bảng sự kiện append-only có cửa bảo trì.

### Phương án đã loại

- Copy code Documenso, DocuSeal, OpenSign (AGPL-3.0); iText 9 (AGPL hoặc thương mại).
- EU DSS làm phụ thuộc mặc định (LGPL-2.1, chờ pháp chế).
- Làm CA, TSA, giữ khoá riêng của người dùng; tự làm trình soạn văn bản.

### Lệch so với thiết kế

- Thiết kế chọn EU DSS cho mọi thao tác chuẩn; bản triển khai tự làm PAdES trên PDFBox + Bouncy Castle và để DSS thành
  module tuỳ chọn.
- `ProviderContext` thêm `userRef` (MySign cần khi hỏi trạng thái); chữ ký CA không mang mã riêng ghép theo thứ tự `#n`.
- Unique của chữ ký `(provider, provider_tx_id, document_id)`; thêm `esign_recipient.ordinal`,
  `esign_trust_anchor.subject`.
- Link ký giữ hiệu lực chỉ đọc sau khi ký xong.

### Còn mở

- Chuyển sang `com.vyforge.platform.report.ReportEngine`/`PdfSigner` khi ghép platform-report.
- Sandbox thật VNPT SmartCA, Viettel MySign; nạp gốc NEAC qua API sau khi xác minh vân tay.
- `remote`, gửi hàng loạt, gia hạn dấu thời gian lưu trữ, `pkcs11`, xác thực EKYC, `EsignErasure` đầy đủ.
- Pháp chế duyệt LGPL của EU DSS ([danh sách](../legal-open-questions.md)).

## report-engine

- **Thiết kế:** quyết định của phiên điều phối khi tách khỏi export (không có tài liệu thiết kế riêng). **Phát hành:**
  `platform-report`, `platform-report-openhtmltopdf` đang tích hợp vào platform-java (sau 0.14.0).

### Quyết định

1. Facade `ReportEngine` dùng chung: export đã chuyển sang; esign, cms-content, billing/einvoice, notification sẽ dùng
   thay vì tự dựng PDF. Engine theo định dạng đổi tên thành
   `TemplateRenderer` để không đụng `export.spi.ReportEngine` (giữ làm alias deprecated).
2. **Tách LGPL ra module riêng:** lõi `platform-report` chỉ Apache/MIT; OpenHTMLtoPDF (LGPL-2.1+) nằm ở
   `platform-report-openhtmltopdf`. Export vẫn kéo adapter để giữ hành vi cũ.
3. Thư viện không kiểm quyền, không audit; job nền bền vững (checkpoint, `DurableTaskQueue`) ở lại export, chỉ dùng
   chung `ReportBatches`.
4. Mẫu DOCX lưu ở asset `template.docx`.

### Phương án đã loại

- OpenHTMLtoPDF là optional dependency của lõi: vẫn lẫn LGPL vào artifact lõi.
- Mỗi module tự dựng PDF (esign từng có kế hoạch này).

### Còn mở

- Adapter `DocumentConverter` (Gotenberg), adapter EU DSS; upload mẫu DOCX qua admin API của export.
- Pháp chế duyệt LGPL của OpenHTMLtoPDF ([danh sách](../legal-open-questions.md)).

## ops-console

- **Thiết kế:** [ops-console.md](../design/ops-console.md). **Phát hành:** `platform-starter-ops`, phần ops của
  `-scheduling`, `-messaging`, `platform-core` (`core.ops`) đang tích hợp vào platform-java; `@vyforge/ops-console` 0.1.0
  và zone `/m/ops-console` chờ merge.

### Quyết định

1. Ops API ở từng backend (`/api/v1/ops/admin/**`), không tập trung; MFE gộp nhiều target theo `OPS_TARGETS`.
2. Ops API không đi qua gateway: BFF gọi URL nội bộ bằng token đổi qua RFC 8693, NetworkPolicy chỉ cho
   `platform-admin`.
3. Chỉ người vận hành: tenant vận hành + role `platform-admin` + `amr` có `mfa` + action `ops:*`; step-up cho thao tác
   nguy hiểm; mọi thao tác ghi có lý do và audit.
4. Cùng hợp đồng với admin-kit (`TaskInfo`, `CursorPage`, `BulkRequest`/`BulkResult`).
5. Retry hàng loạt giãn nhịp (`spreadOver`); tạm dừng queue theo queue hoặc tenant; lịch sử và điều khiển job; outbox đọc
   `event_publication`, cất (`park`) event hỏng; DLT Kafka nạp vào bảng, replay nhắm đúng consumer group.
6. Payload không hiện mặc định: cần `ReadPayload` + step-up, che PII, audit cả khi đọc.

### Phương án đã loại

- Một ops service trung tâm đọc DB của mọi app: phá ranh giới dữ liệu.
- Lọc DLT trực tiếp trên Kafka: không lọc theo tenant, không có trạng thái xử lý, retention 7 ngày.
- Retry ngay cùng lúc (Sidekiq/JobRunr): vượt hạn mức provider rồi lại `DEAD`.

### Lệch so với thiết kế

- Package `@vyforge/ops-console` và zone `/m/ops-console` (thiết kế: `@vyforge/ops`, `/m/ops`); action và API giữ `ops:*`,
  `/api/v1/ops/admin/**`.
- Migration việc nền tách file riêng (V3, khi ghép với admin-kit thành V4) thay vì gộp V2.
- `CursorSlice`, `BulkOutcome` ở `platform-core` (`core.ops`); khi ghép đã đổi `OpsPage`/`OpsBulkResult` sang
  `CursorPage`/`BulkResult` của admin-kit.
- Tạm dừng queue đọc bảng điều khiển vào bộ nhớ (cache 5 giây), fail-open.
- `-messaging` không phụ thuộc `-jpa`/`-scheduling`: StarterSchema đăng ký theo tên lớp, bảo trì DLT/outbox là
  `SmartLifecycle`.
- Resubmit outbox chạy trên virtual thread riêng để listener không chạy trong tenant vận hành.

### Còn mở

- PrometheusRule `ops-rules.yaml` và dashboard; gateway chặn `/api/v*/ops/**`, `OPS_TARGETS`, NetworkPolicy, policy IAM
  mẫu; runbook.
- Auth-server giữ `amr`, `auth_time` khi token exchange; `OpsQueueSource` cho webhooks; file
  `META-INF/vyforge/ops/<module>.yml` của các module.

## cms-content

- **Thiết kế:** [cms-content.md](../design/cms-content.md). **Phát hành:** `platform-starter-cms-content` đang tích hợp
  vào platform-java; `@vyforge/cms-content` 0.1.0 và zone `/m/cms-content` chờ merge.

### Quyết định

1. **Tự xây trên nền platform, không nhúng Payload CMS.** Lõi Java trên PostgreSQL: bản ghi JSONB theo schema có version
   bất biến + upcaster, bảng index có kiểu, bản chiếu đọc `cms_published` cập nhật cùng transaction với publish; không
   DDL theo content type. Thay dòng "Payload CMS khi dự án cần" của MODULE_PLAN mục 5.
2. Rich text Tiptap 3 (MIT), lưu ProseMirror JSON, server kiểm allowlist; renderer React không dùng HTML.
3. Publish theo từng locale; gate `none`/`simple-review` có sẵn, workflow cắm qua `PublicationGate`.
4. Delivery là public path; tenant lấy từ thông tin xác thực đã kiểm (khoá đọc, token preview, JWT, domain đã xác minh),
   không từ header tự khai (ADR-0015).
5. Dùng SQL nền của jpa-extras (`vyforge.fold`, FTS `vyforge.vi`); media qua module files.

### Phương án đã loại

| Phương án | Lý do loại |
|---|---|
| Payload là service Node riêng | Schema là code: tenant không tự thêm content type lúc chạy (K9); phải viết cầu nối OIDC, tenant, IAM; admin là app Next.js riêng, không ghép vào shell; thêm runtime và DB; 3.x sắp đổi lớn sang 4.0 |
| Nhúng Payload vào app Next.js từng dự án | Như trên, nhân theo số dự án; không chạy độc lập với app |
| Strapi, Directus | Strapi chỉ đổi schema ở dev mode; Directus đổi license, cần license key |

Payload vẫn dùng được cho website đơn lẻ không cần tenant/IAM (vẫn đăng nhập qua platform-auth-server).

### Hệ quả

- Công sức ban đầu lớn hơn (editor, schema builder); chia 0.13 (lõi, delivery, admin cơ bản) và 0.14 (release, gate
  workflow, live preview iframe, nhập WXR).
- Một nơi quyết định quyền (`cms:*`), một outbox, một hàng đợi việc nền (queue `cms`).

### Lệch so với thiết kế

- Action IAM dùng kebab-case (`cms:content-type:*`, `cms:delivery-key:Manage`); event webhook `cms.entry.taken_down`.
- DDL thêm `cms_entry.type_key`, `cms_published.title/seo`, bảng `cms_unpublished`, `cms_site_domain.verify_token`,
  `cms_tenant_settings.cache_generation`; index title GIN trgm không kèm `tenant_id`.
- Cache delivery theo generation của tenant (tăng cùng transaction) thay vì xoá theo tag; Valkey nếu có, không thì
  Caffeine.
- Delivery key qua header `X-Cms-Key` hoặc `?key=` (không dùng `Authorization: Bearer`); rate limit theo khoá in-memory.
- Content type bằng code khai báo bằng bean `ContentTypeDefinition` (chưa hỗ trợ YAML).

### Còn mở

- Release + perspective `release`, gate `workflow`, live preview iframe (nới `frame-src` của admin-shell), nhập WXR.
- Mẫu `cms_template*`, `remote` của `ContentDelivery`, block `rawHtml` + sanitizer, `cms-typegen`, Pact, e2e, câu hỏi
  `use_cms` của template-nextjs, màn `/ops`, tích hợp search/realtime/notification, entitlement ở delivery, field địa
  chỉ VN.

## tenant-lifecycle

- **Thiết kế:** [tenant-lifecycle.md](../design/tenant-lifecycle.md). **Phát hành:** `platform-starter-tenant-lifecycle`
  đang tích hợp vào platform-java; scope `auth.tenant.provision` và `createdBy` ở `platform-auth-server` (main);
  `@vyforge/tenant-lifecycle` 0.1.0 và zone `/m/tenant-lifecycle` chờ merge.

### Quyết định

1. Đơn đăng ký tách khỏi tenant; tenant chỉ tạo sau khi đơn đủ điều kiện (trả trước mặc định `PAY_FIRST`), nên đơn bỏ
   ngang không để lại tenant rác.
2. Onboarding/offboarding là run có từng bước (`tl_run_step`) qua SPI `TenantProvisioner` của từng module; bước chạy
   tuần tự theo `seq`.
3. Slug bất biến, slug đã xoá giữ tombstone; tắt khác xoá (đọc-chỉ qua features, khoá đăng nhập qua auth-server).
4. Event và webhook không chứa bí mật; owner đặt mật khẩu ở auth-server.
5. Thời hạn cooling-off 14 ngày, retention 90 ngày, tombstone 180 ngày, biên bản 10 năm là cấu hình, ghi đè theo hợp
   đồng.
6. Biên bản xoá là JSON JCS + SHA-256 + audit; PDF chỉ khi có `CertificateRenderer`.

### Phương án đã loại

- Tạo tenant ngay khi bắt đầu đăng ký (MakerKit): để lại tenant rác.
- Gửi mật khẩu trong event (ABP `TenantCreatedEto`).
- Ký số biên bản ở bản đầu: chờ esign.

### Lệch so với thiết kế

- `suspend`/`resume` tuần tự (thiết kế cho song song).
- `ErasureItem` khai trong `spi` của module (privacy chưa có kiểu chung).
- Subscription đi `ACTIVE → PAST_DUE → SUSPENDED` vì features không cho chuyển thẳng; mở lại bằng `activate`.
- Participant features do bridge cài (xoá bảng `ff_*`/`ent_*`).
- Phần xuất của app ở xa trả inline base64 thay vì `uploadUrl` của files.
- Package `@vyforge/tenant-lifecycle` (thiết kế: `@vyforge/onboarding`).
- OTP tắt mặc định, bật theo loại tổ chức; link email là token HMAC có hạn, chưa dùng một lần.

### Còn mở

- Adapter thật billing (`InvoicePaid` → `markPaid`), privacy-dsr, licensing, settings, organization; auth-server mời owner
  khi email chưa có tài khoản.
- `platform-services`: `SERVICES_MODULES`, route gateway, rate limit `/public/slugs` và OTP.
- Wizard đăng ký nhiều bước, Playwright, PrometheusRule, `TemplateSource` từ CMS, ký số biên bản, verified domain.
- Runbook: khôi phục backup phải chạy lại ERASE cho tenant đã `DELETED` sau thời điểm backup.
- Pháp chế xác nhận các thời hạn retention ([danh sách](../legal-open-questions.md)).
