# eSign — sinh tài liệu, phong bì ký, chữ ký số

## 1. Mục đích

Cho app ký hợp đồng, biên bản, đơn từ ngay trong hệ thống: sinh tài liệu PDF từ template có version, gửi phong bì ký
nhiều bên theo thứ tự, ký điện tử đơn giản (SES, OTP) hoặc ký số PAdES qua CA Việt Nam (VNPT SmartCA, Viettel MySign,
chuẩn CSC), đóng con dấu điện tử của tổ chức, dấu thời gian RFC 3161, LTV, kiểm tra chữ ký theo danh sách tin cậy NEAC,
chuỗi bằng chứng chống sửa và biên bản ký. Thiết kế: `docs/design/esign.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-esign` | platform-java 0.16.1; engine ký PDFBox + Bouncy Castle (Apache-2.0/MIT) |
| Java adapter | `com.vyforge.platform:platform-starter-esign-dss` | EU DSS 6.5.1 (LGPL-2.1) thay bộ kiểm chữ ký mặc định bằng bộ kiểm ETSI EN 319 102-1; app chủ động thêm |
| Package web | `@vyforge/esign` 0.1.0 | client + route handler BFF (`createEsignClient`, `createEnvelopeActions`, `createSigningRouteHandlers`), `/signing` (phòng ký), `/ui`, `/admin` |
| Zone admin | `apps/esign-admin` (`/m/esign`) | template, nhà cung cấp ký số, chứng thư tin cậy, cấu hình, việc nền |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling` (queue `esign`) | production cần files (kho tài liệu) |

## 3. Tính năng chính

- **Sinh tài liệu** (`DocumentGeneration.generate`): template có version — `pdf-form`; `html` (cần
  `platform-report-openhtmltopdf`); `docx` (cần `DocumentConverter`, vd Gotenberg; neo chữ ký dạng `#sign:giam_doc#`).
- **Phong bì** (`EnvelopeService`): tạo từ template hoặc tài liệu, nhiều người nhận có thứ tự, người ký/duyệt/xem, uỷ
  quyền ký, nhắc tự động, hết hạn, huỷ, tải bản cuối và biên bản ký; duyệt nội bộ trước khi gửi (qua workflow).
- **Phòng ký** (`/sign/{token}`, công khai): xác thực người ký `EMAIL_OTP` (mặc định với người ngoài), `LINK`, `ACCOUNT`,
  `EKYC` (step-up sinh trắc qua ekyc); ký SES hoặc ký số từ xa (app CA, có callback).
- **Chính sách ký**: `vn-default`, `vn-digital`, `vn-labor-contract` hoặc policy của app (`min-level`, `seal`,
  `pades-level` B_T/B_LT/B_LTA, `timestamp`).
- **Provider**: `vnpt-smartca`, `viettel-mysign`, `csc` (CSC v2 — không cần code), `pkcs12` (con dấu), `rfc3161` (TSA),
  `mock`; provider của tenant tạo qua admin, secret mã hoá bằng `SecretCipher`, chỉ ghi.
- **Con dấu tổ chức** (`DocumentSigning.seal`, đồng bộ) và `PdfSigner` `provider() = "esign"` cho platform-report (PDF
  báo cáo của export, einvoice, billing có dấu).
- **Kiểm chữ ký** (`SignatureVerification.verify`, `POST /verify`): danh sách tin cậy = 8 gốc NEAC đóng gói (kiểm SHA-256
  lúc khởi động) + gốc nạp qua `/admin/trust-anchors`; không gọi mạng ở bản DSS để kết quả lặp lại được.
- **Chuỗi bằng chứng** chống sửa + biên bản ký PDF (khi có platform-report).
- **SPI**: `RemoteSigningProvider` (CA API riêng, schema ở `META-INF/vyforge/esign/providers/<type>.schema.json`),
  `SignatureValidator`, `SignerAuthenticator`, `EnvelopeGate`, `EsignAuthorizer` (khi không có authz).
- **REST** (`/api/v1/esign`): `/envelopes/**` (người gửi), `/sign/{token}/**` (phòng ký, public), `/callbacks/{provider}/{configId}`
  (public, chỉ là tín hiệu), `/verify`, `/admin/**`. Lỗi Problem Details mã `esign.*` (vi/en).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-esign</artifactId>
</dependency>
<!-- tuỳ chọn, chờ pháp chế duyệt LGPL: platform-starter-esign-dss -->
```

```yaml
vyforge:
  esign:
    public-base-url: https://app.example.vn      # link ký = public-base-url + signing-path + /{token}
    default-policy: vn-default
    providers:
      tsa:  { type: rfc3161, purpose: TSA, settings: { url: ${ESIGN_TSA_URL} } }
      seal: { type: pkcs12, purpose: SEAL, secrets: { keystore: ${ESIGN_SEAL_P12_BASE64}, password: ${ESIGN_SEAL_P12_PASSWORD} } }
```

```java
EnvelopeView draft = envelopes.createFromTemplate(CreateFromTemplate.of(
        "hd-mua-ban", "Hợp đồng mua bán 2026-001", data,
        Map.of("ben_a", RecipientSpec.signer("Nguyễn Văn A", "a@congty.vn", 1),
               "ben_b", RecipientSpec.signer("Trần Thị B", "b@khachhang.vn", 2))));
envelopes.send(draft.id());

@ApplicationModuleListener
void on(EnvelopeCompleted event) { ... }   // event.documents(), event.certificateRef()
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.esign.enabled` | `true` | |
| `vyforge.esign.public-base-url` / `signing-path` | `http://localhost:3000` / `/esign/sign` | Dựng link ký |
| `vyforge.esign.link-ttl` / `default-expiry` | `P30D` / `P30D` | Hạn link và phong bì |
| `vyforge.esign.default-policy` | `vn-default` | |
| `vyforge.esign.default-external-auth` | `EMAIL_OTP` | Xác thực người ký bên ngoài |
| `vyforge.esign.max-document-size` / `max-documents-per-envelope` / `verify-max-size` | 50MB / 20 / 20MB | |
| `vyforge.esign.signature-timeout` | `PT5M` | Chờ người ký xác nhận trên app CA |
| `vyforge.esign.store.type` / `bucket` / `local-directory` | `auto` / `esign` / `./data/esign` | Kho tài liệu (local chỉ cho dev) |
| `vyforge.esign.otp.length` / `ttl` / `max-attempts` | 6 / `PT5M` / 5 | OTP phòng ký |
| `vyforge.esign.reminder.first-after` / `every` | `P2D` / `P3D` | Nhắc người ký |
| `vyforge.esign.providers.*`, `policies.*` | — | Provider cấp platform, policy riêng |
| `vyforge.esign.trust.bundled-neac` | `true` | Tin gốc NEAC đóng gói |
| `vyforge.esign.integrations.workflow.default-process-key` | — | Duyệt trước khi gửi |
| `vyforge.esign.schema.migrate` | `true` | Flyway riêng `flyway_schema_history_esign` |
| `vyforge.esign.dss.enabled` | `true` | Tắt adapter DSS mà không bỏ dependency |

## 6. Tích hợp với module khác

| Module | Có | Không có |
|---|---|---|
| files | Bucket `esign` (PII `basic`, không chia sẻ, audit khi tải), URL presigned, legal hold | Thư mục local, chỉ dev |
| notification | Type `esign.*` (lời mời, OTP, nhắc) | Ghi log; app tự gửi link qua `EnvelopeService.issueLink` |
| platform-report | Template `html`/`docx`, biên bản ký PDF; `PdfSigner` "esign" cho báo cáo có dấu | Chỉ `pdf-form`, không biên bản PDF |
| workflow | Template có `approval.processKey` → phong bì `AWAITING_APPROVAL`, duyệt mới gửi | Gửi thẳng |
| ekyc | Người ký `EKYC` (step-up sinh trắc qua `StepUpAuthentication`) | Tạo người nhận `EKYC` bị từ chối |
| authz | Action `esign:*` | Cần bean `EsignAuthorizer`, không có thì từ chối |
| Redis | OTP, phiên phòng ký nhiều replica | Bộ nhớ (một replica) |
| esign-dss | Kiểm chữ ký ETSI đầy đủ | Bộ kiểm PDFBox + Bouncy Castle |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: chỉ `embedded`; `remote` (chạy trong platform-services) chưa có.
- **IAM**: `esign:envelope:List|ListAll|Get|GetHistory|Create|Update|Send|BulkSend|Void|Download|IssueLink`,
  `esign:template:List|Get|Create|Update|Publish|Retire`, `esign:document:Generate|Seal`, `esign:signature:Verify`,
  `esign:provider:List|Manage|Test`, `esign:trust:List|Manage`, `esign:settings:Get|Update`, `esign:stats:Get`,
  `esign:subject:Get`, `esign:task:Manage`.
- **Sự kiện**: `EnvelopeCreated`, `EnvelopeApprovalRequested`, `EnvelopeSent`, `RecipientNotified`, `RecipientViewed`,
  `RecipientDelegated`, `RecipientCompleted`, `RecipientDeclined`, `SigningProgressed`, `SignatureFailed`,
  `EnvelopeCompleted`, `EnvelopeDeclined`, `EnvelopeExpired`, `EnvelopeVoided`, `TemplatePublished`.
- **Giấy phép**: module `licensed` (`esign`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: template + sinh tài liệu, phong bì nhiều bên, SES/OTP, ký số từ xa (SmartCA, MySign, CSC), con dấu PKCS#12,
  TSA, LTV, kiểm chữ ký + NEAC, biên bản, duyệt qua workflow, người ký eKYC, zone admin, package web.
- Chờ đối tác: adapter VNPT SmartCA và Viettel MySign viết theo tài liệu công khai, kiểm bằng WireMock — phải chạy thử
  sandbox thật (định dạng `signature_value` của SmartCA) trước production.
- Chờ pháp lý: EU DSS (LGPL-2.1) — chỉ nằm trong `platform-starter-esign-dss` đến khi pháp chế xác nhận
  (`docs/legal-open-questions.md`).
- Chưa có: chế độ `remote`, gửi hàng loạt (action IAM `BulkSend` đã khai nhưng chưa có API), gia hạn dấu thời gian lưu trữ định
  kỳ, provider `pkcs11` (HSM/USB token), trình đặt trường trên ảnh trang.
- NEAC đổi gốc thì cập nhật `META-INF/vyforge/esign/trust/neac` và phát hành lại; trong lúc chờ nạp gốc mới qua API.
