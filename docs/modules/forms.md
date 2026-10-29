# Forms — biểu mẫu động và khảo sát

## 1. Mục đích

Cho mọi app một bộ biểu mẫu do người dùng định nghĩa lúc chạy: form liên hệ/ứng tuyển công khai, form nội bộ, form theo
lời mời, khảo sát CSAT/NPS, hồ sơ một cửa có số tham chiếu và tra cứu biên nhận. Định nghĩa có version, biên dịch ra JSON
Schema 2020-12 + UI schema, kiểm ở cả server và client, chống spam, lưu bài nộp JSONB có retention và xoá theo yêu cầu
chủ thể. Lõi không có database (`platform-forms-core`) được custom-fields, workflow, admin-kit dùng chung. Thiết kế:
`docs/design/forms.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java lib | `com.vyforge.platform:platform-forms-core` | `FormSpec`, `FormCompiler`, `FieldType` (25 kiểu có sẵn: CCCD, MST, SĐT VN, tiền VNĐ, NPS…), `FormValidator`, `ConditionEvaluator`; không bean, không bảng |
| Java starter | `com.vyforge.platform:platform-starter-forms` | platform-java 0.16.1; bảng `form_*`, REST công khai/của tôi/quản trị/nội bộ, chế độ `remote` |
| Package web | `@vyforge/forms` 0.2.0 | `/spec`, `/renderer` (rjsf 6 + validator không `eval`), `/next` (`PublicForm`), `/admin` (`FormBuilder`…), `/messages` |
| Zone admin | `apps/forms-admin` 0.2.0 (`/m/forms`) | danh sách form, builder, bài nộp, báo cáo khảo sát, lời mời, hành động, cài đặt, usage |
| Phụ thuộc bắt buộc | PostgreSQL, `-jpa`, `-web`, `-authz`, `-scheduling`, `-messaging`, `-ratelimit` | |

## 3. Tính năng chính

- **Định nghĩa**: form có version bất biến sau khi xuất bản; nháp → xuất bản → đóng → lưu trữ; nhiều trang/section;
  logic hiện/ẩn/bắt buộc theo điều kiện; i18n vi/en. Ba cách tạo: builder kéo thả, khai trong code
  (`FormDeclaration`/`META-INF/vyforge/forms/*.form.json`), dán JSON Schema thô (`forms:form:EditRawSchema`).
- **Phân phối**: công khai (`/f/{publicId}`, nhúng, CORS theo `allowedEmbedOrigins`), đăng nhập (`/forms/me`), theo
  lời mời (link một lần), gắn ngữ cảnh (`contextType/contextId`, vd CSAT của ticket).
- **Bài nộp**: lưu nháp/tiếp tục (session), sửa sau khi nộp (tuỳ form), số tham chiếu, trạng thái
  SUBMITTED/ACCEPTED/REJECTED/RETURNED/WITHDRAWN/SPAM, field nhạy cảm mã hoá riêng, tra cứu biên nhận qua OTP, thao tác
  hàng loạt chạy nền, xuất CSV.
- **Chống lạm dụng**: captcha (`HumanVerification`, mặc định `ADAPTIVE`), honeypot, thời gian điền tối thiểu, rate limit
  theo form/IP, hạn mức bài nộp, chặn trùng, hàng đợi spam.
- **Khảo sát**: rating, NPS, CSAT, CES, Likert, ma trận; báo cáo tổng hợp, chuỗi NPS theo thời gian; ô < 5 gộp "Khác".
- **Hành động sau khi nộp** (`SubmissionAction`, chạy nền): `email.internal`, `email.confirmation` (khi có
  notification), `webhook`, `target:*` (`SubmissionTarget` đẩy thành bản ghi của app).
- **API Java**: `Forms` (find, render, validate), `Submissions`, `Invitations`, `FormDefinitions`, `SurveyReports`.
- **SPI** (`…forms.spi`): `FormDeclaration`, `FormTemplateProvider`, `SubmissionInterceptor`, `SubmissionTarget`,
  `SubmissionAction`, `HumanVerification`, `ReferenceNumberGenerator`, `FormUploads`, `FormsWebhookPublisher`,
  `ConsentRecorder`, `SubmissionQuota`, `FormsAuthorizer`, `FormsNotifier`, `CapabilityContributor`; lõi: `FieldType`,
  `ChoiceSource`.
- **REST** (gốc `/api/v1/forms`): `public/{publicId}` (render, `submissions`, `sessions`, `uploads`,
  `receipts/lookup`, `choices/{source}`); `me/forms/{key}` (nộp, nháp, bài của tôi); `admin/**` (forms CRUD, `draft`,
  `draft/raw`, `publish`, `close`, `reopen`, `archive`, `duplicate`, `versions`, `prefill`, `actions`, `field-types`,
  `templates`, `settings`, `submissions` (+ `bulk`), `usage`, `info`); `internal/**` (cho app ở chế độ remote).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-forms</artifactId>
</dependency>
<!-- chỉ cần bộ field + bộ kiểm, không bảng: platform-forms-core -->
```

```yaml
vyforge:
  forms:
    public-api:
      base-url: https://app.example.vn   # dựng link /f/{publicId}, link lời mời, QR
    signing-key: ${FORMS_SIGNING_KEY}     # khi app không có SecretCipher (base64 ≥ 32 byte)
```

```java
@Component
class LeadFromContactForm implements SubmissionInterceptor {
    // beforeValidate / validate / beforeSave / afterCommit: kiểm nghiệp vụ hoặc đẩy dữ liệu sang bản ghi của app
}
```

```tsx
// app/f/[publicId]/page.tsx (Next.js)
const form = await loadPublicForm(forms(), publicId, await searchParams);
if (!form) notFound();
return <PublicForm form={form} submit={submitForm} />;
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.forms.enabled` | `true` | Tắt cả module |
| `vyforge.forms.mode` | `embedded` | `remote`: module chạy ở platform-services, app gọi `/forms/internal/**` |
| `vyforge.forms.locales` | `vi, en` | Locale được soạn |
| `vyforge.forms.api.base-path` | `/api/v1` | Tiền tố REST |
| `vyforge.forms.public-api.base-url` | `http://localhost:3000` | Dựng link công khai |
| `vyforge.forms.public-api.max-payload` / `session-ttl` / `session-upload-quota` | 256KB / 24h / 50MB | |
| `vyforge.forms.public-api.form-rate-limit` | 200 | Bài nộp mỗi 10 phút một form |
| `vyforge.forms.validation.strict-unknown-fields` | `true` | Field lạ → 422 |
| `vyforge.forms.defaults.retention` / `captcha` / `timezone` | `P2Y` / `ADAPTIVE` / `Asia/Ho_Chi_Minh` | Mặc định cài đặt tenant |
| `vyforge.forms.events.externalize` | `false` | Phát event ra Kafka topic `vyforge.forms` |
| `vyforge.forms.jobs.retention-cron` | `0 30 2 * * *` | Xoá dữ liệu hết hạn lưu |
| `vyforge.forms.remote.registration-id` | `vyforge-forms` | Client credentials (scope `forms.internal`) ở chế độ remote |
| `vyforge.forms.internal-api.scopes` | `forms.internal, services.internal` | Scope được gọi API nội bộ |

## 6. Tích hợp với module khác

| Module | Trong code hiện tại |
|---|---|
| notification | `NotificationsFormsNotifier` (auto-config): email lời mời, xác nhận, OTP biên nhận |
| master-data | `AdminUnitChoices` (`ChoiceSource` đơn vị hành chính, kiểu `vn-address`) |
| workflow | `FormsFormRegistry` nằm ở workflow: publish quy trình kiểm `form` của bước tồn tại |
| ratelimit (bắt buộc), audit | rate limit form công khai; ghi audit thao tác quản trị (tuỳ chọn) |
| custom-fields, admin-kit | dùng `platform-forms-core` và renderer `SchemaForm` |
| files, webhooks, antibot, sequence, privacy-dsr, features | có SPI (`FormUploads`, `FormsWebhookPublisher`, `HumanVerification`, `ReferenceNumberGenerator`, `ConsentRecorder`, `SubmissionQuota`) nhưng chưa thấy adapter trong repo (cần xác nhận) |
| cms-content | block `form` gửi thẳng tới `/api/v1/forms/public/**` (theo thiết kế) |
| automation | trigger trên `FormSubmitted`, `SubmissionStatusChanged` theo thiết kế; event của forms **chưa** đánh `@AutomationEvent` nên app phải khai `AutomationTriggerCatalog` |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded` (mặc định) và `remote` (đã cài: `FormsRemoteClient`, `RemoteForms`, `RemoteSubmissions`,
  `RemoteInvitations`, đăng ký `FormDeclaration` qua `PUT /forms/internal/declarations/{service}`).
- **IAM**: `forms:form:List|Get|Create|Update|Delete|Publish|Close|Archive|ManageActions|EditRawSchema`,
  `forms:submission:List|Get|GetSensitive|Create|Edit|Review|MarkSpam|Erase|Export`,
  `forms:invitation:List|Create|Revoke`, `forms:report:Get`, `forms:template:List|Create|Update|Delete`,
  `forms:settings:Get|Update`, `forms:usage:Get`, `forms:task:Manage`.
- **Sự kiện** (`TenantAwareEvent`, không mang giá trị field): `FormPublished`, `FormStateChanged`, `FormSubmitted`,
  `SubmissionStatusChanged`, `SubmissionErased`, `InvitationExpired`.
- **Giấy phép**: module `licensed` (`forms`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: lõi + starter, form công khai/nội bộ/lời mời, khảo sát, tra cứu biên nhận OTP, CORS nhúng, thao tác hàng loạt,
  usage, schema thô, kiểu `vn-address`, chế độ remote, zone admin.
- Chưa thấy adapter cho files (kiểu `file`), webhooks, antibot (ALTCHA), sequence (số tham chiếu theo mẫu), privacy-dsr
  (contributor/consent), features (hạn mức theo gói), export XLSX — hiện xuất CSV tích hợp (cần xác nhận).
- Để bản sau theo thiết kế: index search bài nộp, action `pdf`/`esign.envelope`, kiểu field `ekyc-check`, lịch đóng theo
  ngày làm việc.
- Không có README module trong repo platform-java; tài liệu này dựa trên code và `docs/design/forms.md` (đề mục còn ghi
  "đề xuất, chờ duyệt").
