# platform-java 0.16.0

Bản này gộp mốc follow-up của các module dùng chung: tích hợp chéo giữa module, chế độ remote, và các adapter còn thiếu
ở 0.15. Có thêm cưỡng chế giấy phép cho starter trả phí, và một đợt "mặc định an toàn" cho những chỗ còn chờ ý kiến
pháp chế.

**Có thay đổi phá vỡ và thay đổi hành vi cần làm khi nâng cấp.** Xem mục [Nâng cấp từ 0.15](#nâng-cấp-từ-015). Quan trọng
nhất: trên môi trường không có giấy phép, starter trả phí không còn tự bật.

| | |
|---|---|
| Java | 25 |
| Spring Boot | 4.1.1 (không đổi so với 0.15) |
| Nâng cấp | đổi `platform-bom`/`platform-parent` sang `0.16.0`, rồi làm các bước ở mục Nâng cấp |
| Migration DB | chỉ có migration mới. Không sửa migration đã phát hành. `files V2` **dừng lại** nếu dữ liệu cũ không hợp lệ (xem mục 2) |
| Bảo mật | Guava 33.7.2-jre (CVE-2026-102554) |

## Module mới

Không có artifact mới. `platform-bom` có thêm `platform-starter-ekyc` dạng `test-jar`
(`EkycProviderContractTests`, `ChipFixture`), dùng khi viết adapter nhà cung cấp eKYC ở module khác.

## Tính năng mới trong module sẵn có

- **licensing:**
  - Starter trả phí cần giấy phép (mục 1 của phần Nâng cấp).
  - Kiểm URL bản triển khai: `vyforge.license.deployment-url` (biến `VYFORGE_DEPLOYMENT_URL`) khác mọi
    `deployments[].urls` của giấy phép thì cảnh báo `DEPLOYMENT_URL_MISMATCH`. Chỉ cảnh báo, không chặn. So theo host,
    không phân biệt hoa thường.
  - `LicensingNotice`: sự kiện đã ghi audit, phát đúng một lần cho cả bản triển khai, gồm kích hoạt, từ chối, đổi trạng
    thái, chạm giới hạn, sắp hết hạn. Đặt `vyforge.licensing.events.externalize=true` thì sự kiện ra topic
    `vyforge.licensing`.
- **files:**
  - Attachment dùng `EntityRef` của platform-core (mục 2 của phần Nâng cấp).
  - File đi theo xoá mềm của jpa-extras:
    - xoá mềm bản ghi cha thì các file chỉ gắn vào bản ghi đó vào thùng rác cùng nó;
    - khôi phục bản ghi thì khôi phục đúng các file đó;
    - purge bản ghi thì xử lý file theo `on-entity-deleted`;
    - bản ghi còn file đang legal hold thì không bị purge (`PurgeVeto`).
  - Sửa race khi dedup.
- **jpa (thùng rác):**
  - Lọc theo ngày `deletedFrom`/`deletedTo`.
  - Mỗi app một API thùng rác: `vyforge.jpa.trash.api.path`, `vyforge.jpa.trash.api.app`. `GET …/admin/info` trả `app`
    và `apiBase` để khu quản trị gom nhiều nguồn.
  - Retention của thùng rác theo settings.
- **custom-fields:**
  - Migration bật/tắt mã hoá cho trường đã có dữ liệu (`ENCRYPT`/`DECRYPT`), và `rekey` sau khi xoay khoá.
  - `COPY` sang trường mới khi cần đổi kiểu ngoài ma trận chuyển đổi.
  - `KEEP_AS_LEGACY` cho giá trị không chuyển được.
  - Điều kiện `visibleWhen`/`requiredWhen` tham chiếu thuộc tính gốc của bản ghi (`conditionAttributes`). Có thay đổi
    trong platform-core, xem mục 3 của phần Nâng cấp.
  - `lookup-failure: ACCEPT_UNVERIFIED`: khi nguồn lựa chọn không trả lời, vẫn nhận giá trị và để job
    `custom-fields-lookup-verification` kiểm lại sau.
  - Invalidation cache schema qua Valkey (kênh `vyforge:customfields:changed`).
  - Header `Vyforge-Warnings` báo cảnh báo của lần lưu.
  - Chỉ mục cho bảng phân vùng.
- **privacy-dsr:**
  - Mặc định an toàn chờ pháp chế (mục 4 của phần Nâng cấp).
  - Adapter: notification (thông báo cho chủ thể dữ liệu, đồng ý nhận quảng cáo), files (gói xuất ở bucket
    `privacy-packages`), calendar (ngày làm việc theo lịch của tenant).
  - Checklist vận hành `OPERATIONS.md`.
- **tenant-lifecycle:**
  - Retention cần được duyệt trước khi tự xoá (mục 5 của phần Nâng cấp).
  - Bridge thật sang billing (checkout, hoá đơn đầu), payment (QR/link thanh toán), privacy-dsr (`TENANT_EXPORT`,
    `TENANT_ERASE` có duyệt 4 mắt) và giấy phép (giới hạn `tenants`, `block-new-tenants`).
  - Rate limit cho slug, signup, OTP.
- **api-access:**
  - K9: kiểm tra installation, initiate login URI (migration `apiaccess V4`), thu hồi installation.
  - Bridge sang notification.
  - `ApplicationAttributeSchema`.
  - Entitlement `api.max-plan`, `api.keys.max`.
  - Gauge `keys.active`/`keys.expiring`.
  - Health của provisioner.
  - Giới hạn số request đồng thời theo từng application ở gateway.
  - Hợp đồng Pact gateway → auth-server và gateway → module.
- **ai:** OCR sidecar, rerank, đồng bộ với files, ngân sách lưu ở Valkey, MCP client, chế độ remote (token exchange
  RFC 8693), gauge. Reset cooldown trong test giờ ổn định.
- **esign:**
  - Xác thực người ký bằng eKYC.
  - Workflow gate cho phong bì.
  - Đi kèm chứng thư gốc NEAC.
  - Chính sách bucket của files.
  - Render template và seal báo cáo qua `platform-report` (template DOCX đã nối vào).
  - **Sửa lỗi:** chứng thư CA của NEAC phát hành từ 2025 (ví dụ VNPT SmartCA RS 2026) đặt CRL Distribution Points là
    critical. PKIX của JDK từ chối cả đường chứng thư. Nay đã xử lý (`DistributionPointsChecker`); thu hồi vẫn được kiểm
    riêng.
- **workflow:** adapter sang các module, lịch theo từng định nghĩa quy trình, privacy, retention, health.
- **automation:** adapter sang các module, audit trail, contributor cho privacy-dsr, trạng thái tenant, rate limit dùng
  chung.
- **forms:**
  - Chế độ remote.
  - Tra cứu biên nhận (OTP), CORS cho form nhúng.
  - Thao tác hàng loạt, thống kê usage, raw schema, `EXTENDABLE`, `vn-address`.
  - Migration `forms V2`–`V5`.
- **cms-content:** `rawHtml`, template (`cms V2`), delivery qua remote, release (`cms V3`), workflow gate.
- **collaboration:**
  - Dùng text và ref của platform-core.
  - Adapter sang các module, catalog webhook.
  - Chế độ remote, ingest qua Kafka.
- **search:** chế độ remote, contributor cho privacy-dsr, dùng `VietnameseText` của platform-core.
- **settings:**
  - Duyệt 4 mắt (`settings V2`).
  - Lớp `ORG_UNIT`.
  - Contributor cho privacy-dsr.
  - Retention của thùng rác jpa, chuỗi locale.
  - Sửa list có chỉ mục.
- **sequence:**
  - Phát event qua Kafka, rate limit cho REST.
  - privacy-dsr, phạm vi theo organization, settings.
  - Detach partition.
- **organization:**
  - `vyforge.organization.integrity=application`: chạy migration không cần extension `btree_gist`
    (`organization-application V1`).
    - Mặc định vẫn là `database`.
    - Phải chọn trước lần migrate đầu tiên.
  - `OrgAdmin` remote, tìm người, benchmark JMH.
- **master-data:** adapter địa lý, nguồn tỷ giá, XLSX, compaction, giới hạn theo entitlement, ký gói dữ liệu.
- **calendar:** tích hợp, lặp theo âm lịch, đồng bộ qua Valkey, chế độ remote.
- **payment:** trường merchant `lookup:bank` lấy từ danh sách ngân hàng của master-data.
- **ekyc:** duyệt tay qua workflow; test-jar hợp đồng cho adapter nhà cung cấp.
- **scheduling:** **sửa lỗi**: điều kiện tạm dừng queue được kiểm ngay trong câu lệnh claim.
  - Trước đây worker có thể nhận task của queue vừa bị tạm dừng: worker đọc cache "không dừng" ngay trước lệnh dừng,
    hoặc lệnh dừng được ghi từ instance khác trong lúc cache chưa hết hạn.

## Nâng cấp từ 0.15

### 1. Starter trả phí cần giấy phép (licensing L6)

**Trước:** mọi starter tự bật khi có trên classpath.

**Nay:** phần `embedded` của starter trả phí có `@ConditionalOnLicensedModule("<key>")`. Module không có trong giấy phép
thì không có bean nào của module được tạo: không migration, không job, không API. App cần bean đó thì khởi động hỏng, và
`LicenseFailureAnalyzer` in lý do.
- Client `remote` không gắn điều kiện này, vì module được kiểm ở nơi nó chạy.
- Module bị rút khỏi giấy phép lúc đang chạy thì API của module trả `404 license.module_not_licensed`.

| Cần giấy phép (`licensed: true`) | Không cần |
|---|---|
| `ai`, `api-access`, `automation`, `billing`, `calendar`, `cms-content`, `collaboration`, `custom-fields`, `einvoice`, `ekyc`, `esign`, `export`, `features`, `files`, `forms`, `master-data`, `notification`, `organization`, `payment`, `privacy` (privacy-dsr), `realtime`, `search`, `sequence`, `tenant-lifecycle`, `webhooks`, `workflow` | `licensing`, `ops-console`, `settings`, `webhooks-inbound`; các starter nền (core, web, security, jpa, messaging, audit, scheduling, ratelimit, privacy cơ bản) |

Danh sách lấy từ `META-INF/vyforge/modules/<key>.json` của từng starter (trường `licensed`).

Chế độ theo `vyforge.license.mode` (mặc định `auto`):
- **`auto`:** không có file giấy phép mà profile là `dev`/`local`/`test` thì chạy `DEV` (mọi module bật, tối đa 5 tenant
  và 50 người dùng). Còn lại là `UNLICENSED`: starter trả phí tắt.
- **`enforce`:** luôn đòi giấy phép.
- **`dev`:** luôn chạy chế độ dev.

**Test chạy ở chế độ dev license.** `platform-test` đăng ký `DevLicenseModeForTests` (một `LauncherSessionListener` của
JUnit). Listener này đặt `vyforge.license.mode=dev` khi phiên JUnit bắt đầu, nên starter trả phí bật trong cả
`@SpringBootTest` lẫn `ApplicationContextRunner` mà không cần profile `test`.
- Listener không ghi đè giá trị đã đặt sẵn qua system property hay biến `VYFORGE_LICENSE_MODE`.
- Test cần giấy phép cụ thể dùng `@WithLicense`. Khi có giấy phép, chế độ là `LICENSED`.

**Cần làm:**
1. Môi trường staging và production: cài giấy phép trước khi nâng cấp, ở `/etc/vyforge/license/license.lic` hoặc biến
   `VYFORGE_LICENSE_TEXT`, và đặt `VYFORGE_DEPLOYMENT_ID`. Giấy phép phải liệt kê mọi module có phí mà dịch vụ dùng.
   Nếu không, dịch vụ dùng starter trả phí sẽ không khởi động được.
2. Nên đặt thêm `VYFORGE_DEPLOYMENT_URL` để được cảnh báo khi giấy phép dùng sai bản triển khai.
3. Máy dev chạy profile `dev` hoặc `local` thì không phải làm gì. Môi trường chạy không có profile mà cũng không có giấy
   phép (ví dụ demo nội bộ) thì đặt `vyforge.license.mode=dev`.
4. Test có `platform-test` trên classpath thì không phải làm gì. Test không dùng `platform-test` thì tự đặt
   `vyforge.license.mode=dev` (ví dụ qua `systemPropertyVariables` của surefire).

### 2. files dùng `EntityRef` của platform-core (BREAKING)

**Trước:** `com.vyforge.platform.files.EntityRef`. Type khớp `[a-z0-9-]{1,40}:[a-zA-Z0-9-]{1,58}`, id là chuỗi bất kỳ
dài 1–255 ký tự.

**Nay:** lớp đó đã bị xoá. API của files (`Attachments`, `AttachTarget`, `AttachmentView`, event `FileAttached`,
`FileDetached`, `FileUploaded`) dùng `com.vyforge.platform.core.ref.EntityRef`. Kiểu này dùng chung với collaboration,
workflow, search, esign, cms:
- `type`: `<service>:<resource>`, toàn chữ thường (`[a-z][a-z0-9-]{0,39}:[a-z][a-z0-9-]{0,58}`);
- `id`: `[A-Za-z0-9._~%-]{1,200}`, không có `/` hay `:`. Khoá có ký tự khác thì tạo bằng `EntityRef.ofKey(type, rawKey)`
  (percent-encode);
- dạng chuỗi chuẩn: `crm:order/01J9…`; JSON: `{"type": "crm:order", "id": "01J9…"}`;
- REST nhận ref sai định dạng thì trả `400 platform.invalid_entity_ref`.

**Cần làm:**
1. Đổi import `com.vyforge.platform.files.EntityRef` thành `com.vyforge.platform.core.ref.EntityRef`.
   `EntityRef.of(type, id)` vẫn còn; có thêm `of(service, resource, id)`, `ofKey`, `parse`, `tryParse`.
2. Type có chữ hoa ở phần resource (`crm:salesOrder`) phải đổi sang chữ thường (`crm:sales-order`). Kiểm lại cả cấu hình
   `entity-types` của bucket và policy authz.
3. **Migration `files V2` dừng (raise exception)** nếu `file_attachment` còn dòng sai một trong các điều kiện sau:
   - id dài hơn 200 ký tự;
   - id có ký tự ngoài `[A-Za-z0-9._~%-]`;
   - type có chữ hoa.

   Platform cố ý không tự cắt hay sửa dữ liệu. Trước khi nâng cấp, chạy câu kiểm sau, rồi cập nhật các dòng này về dạng
   `EntityRef.ofKey` (và sửa phía app cho khớp):
   ```sql
   select entity_type, entity_id from file_attachment
    where length(entity_id) > 200
       or entity_id !~ '^[A-Za-z0-9._~%-]+$'
       or entity_type !~ '^[a-z][a-z0-9-]{0,39}:[a-z][a-z0-9-]{0,58}$';
   ```
4. `files V3` thêm cột `file_object.trashed_with`, dùng cho thùng rác đi cùng bản ghi cha.

### 3. custom-fields mở rộng `SaveContext` và `ExtensionGuard` của platform-core

- `SaveContext` có thêm thành phần `attributes`: thuộc tính gốc của bản ghi mà điều kiện tham chiếu.
  - Constructor 5 tham số cũ, `user(…)`, `withMode`, `withRecordVrn` vẫn còn.
  - Code tạo `SaveContext` không phải sửa.
  - Phải sửa nếu code destructure record (`case SaveContext(var a, var b, var c, var d, var e)`) hoặc so sánh hai
    `SaveContext` bằng `equals`, vì record giờ có 6 thành phần.
- `ExtensionGuard` có thêm method `default Set<String> conditionAttributes(String entityType)`. Implementation tự viết
  không phải sửa.
- App không dùng JPA mà muốn dùng `visibleWhen`/`requiredWhen` theo thuộc tính gốc thì tự đưa giá trị vào:
  `SaveContext.user(vrn, isNew).withAttributes(Map.of("segment", "B2B"))`. Listener JPA tự làm việc này.
- **Đổi mặc định:** `vyforge.custom-fields.index.max-per-table` hạ từ `200` xuống `50`. Benchmark cho thấy 50 chỉ mục
  `cfx_` đã làm ghi chậm 25–60% cho mọi tenant của bảng. Bảng đã có hơn 50 chỉ mục vẫn giữ nguyên các chỉ mục đó, nhưng
  không tạo thêm được. Cần nhiều hơn thì đặt lại giá trị này (khi bảng ít ghi) hoặc dùng search.
- Migration `customfields V2` thêm bảng `cf_pending_verification`.

### 4. privacy-dsr: mặc định an toàn chờ pháp chế (Q1, Q2, Q3, Q6, Q8)

Các câu hỏi Q1, Q2, Q3, Q6, Q8 trong
[legal-open-questions.md](../legal-open-questions.md) chưa có ý kiến pháp chế. Engine giữ phương án an toàn (hạn sớm hơn,
giữ bằng chứng lâu hơn, miễn trừ hẹp hơn, residency chỉ ghi nhận) và **chặn mọi cấu hình lỏng hơn**:

| Câu hỏi | Khoá | Mặc định (an toàn) | Lỏng hơn (cần `legal.<q>` = `CONFIRMED`) |
|---|---|---|---|
| Q1 | `sla.execute-day-kind` | `CALENDAR` | `WORKING`; dời hạn rơi vào ngày nghỉ (`integration.calendar.roll-forward-calendar-days`) |
| Q2 | `sla.clock-start` | `RECEIVED` | `VERIFIED` (ở platform hoặc settings của tenant) |
| Q3 | `consent.evidence-retention` | `1826d` (5 năm) | ngắn hơn (ở platform hoặc `consent.evidenceRetentionYears` của tenant) |
| Q6 | `exemption.large-scale-threshold` | `100000` | cao hơn (thấp hơn là chặt hơn, được phép) |
| Q8 | quy tắc residency `default-vn` | `MONITOR` | quy tắc platform (`'*'`) ở `ENFORCE` |

Tất cả khoá trên nằm dưới `vyforge.privacy.dsr.`. Giá trị mặc định không đổi so với 0.15, nhưng nay có cơ chế chặn:
- cấu hình platform lỏng hơn mặc định thì app **không khởi động**, thông báo ghi rõ khoá và câu hỏi;
- settings của tenant, hoặc quy tắc residency của platform, lỏng hơn thì bị từ chối `422 privacy.legal_pending`;
- lúc khởi động có log WARN liệt kê các câu còn chờ; `GET /admin/dashboard` trả `legalReview`.

**Cần làm:**
1. Dịch vụ nào đã đặt một trong các khoá trên lỏng hơn mặc định thì sẽ không khởi động. Đưa về mặc định, hoặc khi pháp
   chế đã trả lời thì đặt `vyforge.privacy.dsr.legal.<q>.status=CONFIRMED` kèm `reference=<số văn bản>` (bắt buộc).
2. Có platform-starter-calendar thì hạn SLA tính theo lịch nghỉ lễ của tenant (`CalendarWorkingDays`). Năm chưa có dữ
   liệu thì tính kiểu an toàn; calendar lỗi thì chỉ trừ thứ Bảy và Chủ nhật. Nạp dữ liệu nghỉ lễ cho lịch mặc định của
   tenant.
3. Có platform-starter-files và `packaging.store=auto` (mặc định) thì gói xuất DSR lưu ở bucket `privacy-packages` của
   files (`FilesPackageStore`), không còn lưu trong bảng JDBC. Muốn giữ cách cũ thì đặt
   `vyforge.privacy.dsr.packaging.store=jdbc`.
4. Có platform-starter-notification thì privacy-dsr gửi thông báo qua notification (loại thông báo ở
   `META-INF/vyforge/notification/types/privacy.yml`). Tắt bằng `vyforge.privacy.dsr.integration.notification.enabled=false`.
5. Đi qua checklist `platform-starter-privacy-dsr/OPERATIONS.md` trước khi bật cho khách hàng.

### 5. tenant-lifecycle: hết hạn giữ dữ liệu không còn tự xoá

**Trước:** offboarding tự xoá dữ liệu tenant khi hết `retention_days`.

**Nay:** `vyforge.tenant-lifecycle.retention-policy.status` mặc định là `PENDING_LEGAL`.
- Các mốc khác vẫn chạy: đọc-chỉ, khoá đăng nhập, huỷ gói, hết hạn link xuất.
- Khi hết hạn giữ dữ liệu:
  - offboarding **dừng ở `RETAINED`**;
  - dòng thời gian của tenant có `erase_awaiting_confirmation`;
  - phát event `TenantErasureAwaitingConfirmation`;
  - người vận hành (`operators`) nhận thông báo `tenant.offboarding.erase-confirmation`.
- Người vận hành xoá tay bằng `POST /admin/tenants/{slug}/offboarding:erase-now`. Khi bật
  `erase-now-requires-approval`, cần hai người khác nhau.
- Khi pháp chế duyệt chính sách: đặt `retention-policy.status: APPROVED` kèm `version` (bắt buộc) và `reference`. Từ đó
  hết hạn thì tự xoá như trước, và biên bản xoá ghi `retentionPolicy {status, version, reference}`.
- Còn chờ quyết định (Q3): các con số `cooling-off`, `default-retention`, `tombstone`, `certificate-retention`,
  `signup.purge-after`, và có bật `auto-offboard-after` hay không.

**Cần làm:** theo dõi các tenant ở `RETAINED` (`/admin/info` trả `retentionPolicy`; MFE nhắc việc), và xoá tay khi đủ
điều kiện.

### 6. SBOM: `makeBom` thay cho `makeAggregateBom`

`platform-parent` tắt execution `default` (`makeAggregateBom`, kế thừa từ `spring-boot-starter-parent`) và chạy
`makeBom` ở execution `sbom`. Lý do: `makeAggregateBom` là aggregator mojo. Khi build song song (`-T`), Maven 3.9 giữ khoá
cho nó, nên mỗi module phải chờ mojo của module khác chạy xong. Lỗi này đã làm build của platform chậm khoảng 20 phút.

- Vị trí không đổi: `META-INF/sbom/application.cdx.json` trong jar, Actuator `/actuator/sbom` đọc được như cũ.
- Định dạng chỉ còn JSON.
- **Khác:** mỗi jar chỉ chứa SBOM của chính module đó cùng dependency của nó.
  - Dịch vụ một module: nội dung như trước.
  - Dịch vụ nhiều module trước đây có một SBOM gộp cả reactor. Nếu pipeline cần SBOM gộp (ví dụ để quét lỗ hổng), chạy
    riêng `cyclonedx:makeAggregateBom` ở bước release, hoặc ghép từ SBOM của từng jar.
- Dịch vụ tự cấu hình lại execution `default` của `cyclonedx-maven-plugin`: kiểm lại, vì execution đó giờ là
  `<phase>none</phase>`.

### 7. Guava 33.7.2-jre (CVE-2026-102554)

CEL 0.14.0 (qua `platform-expression`) kéo Guava 33.6.0-jre, bản bị CVE-2026-102554. `platform-bom` ghim
`com.google.guava:guava` **33.7.2-jre**, và nâng `error_prone_annotations` lên 2.50.0 cho khớp
(`dependencyConvergence`). Các module bị ảnh hưởng: platform-expression, workflow, organization, automation,
platform-services.

Dịch vụ import `platform-bom` thì tự nhận bản vá. Dịch vụ tự ghim Guava thấp hơn thì bỏ ghim đó. Dùng
`./mvnw dependency:tree -Dincludes=com.google.guava:guava` để kiểm.

### Khác

- **Migration mới** (Flyway của từng starter tự chạy, hoặc file gộp `META-INF/vyforge/sql/*.sql` cho DBA khi
  `schema.migrate=false`):
  - `apiaccess V4` (initiate login URI);
  - `cms V2` (template), `V3` (release);
  - `customfields V2` (`cf_pending_verification`);
  - `files V2` (EntityRef, **có kiểm dữ liệu**), `V3` (`trashed_with`);
  - `forms V2` (độ dài `compiler_version`), `V3` (OTP biên nhận), `V4` (job hàng loạt), `V5` (khai báo remote);
  - `settings V2` (duyệt 4 mắt);
  - `organization-application V1`: thư mục riêng, chỉ dùng khi `integrity=application`.
- **Độ tin cậy của bộ test** (chỉ ảnh hưởng người build platform):
  - **Container dùng chung:** `SharedPostgres`, `SharedRedis`, `SharedKafka` của platform-test chạy một container cho cả
    JVM test, mỗi context một database (Kafka: topic riêng).
  - **Kafka trong test:** `PlatformKafkaContainer` sửa lỗi `Text file busy` (mã 126) lúc khởi động. Không còn dùng image
    `kafka-native`, bản hay chết với mã 139. Broker chết thì báo ngay kèm mã thoát và log.
  - **Context test được cache tạm dừng ngay khi không còn lớp nào dùng** (`spring.test.context.cache.pause=ALWAYS`, chỉ
    trong POM gốc của repo platform). Worker và listener của context cũ không còn nhận việc của lớp sau.
  - Sửa các test hay hỏng: ai cooldown, ratelimit, ai prompt, webhooks dead-letter (giờ in chi tiết từng lần gửi khi hỏng).
