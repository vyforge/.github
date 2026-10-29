# platform-report (+ platform-report-openhtmltopdf)

## 1. Mục đích

Thư viện dựng PDF và tài liệu dùng chung, để các module không phải tự copy code dựng PDF:

- export: báo cáo, in hàng loạt;
- esign: tài liệu cần ký;
- notification: file đính kèm;
- cms-content: bản in;
- billing/einvoice: bản thể hiện hoá đơn.

Template theo tenant có phiên bản; bản trong classpath làm mặc định. Có font tiếng Việt nhúng, mẫu DOCX, và ký PAdES qua
SPI. Bộ chuyển HTML → PDF có giấy phép LGPL được cô lập trong module riêng.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-report` (**thư viện**, không phải starter, không phụ thuộc `platform-core`); `com.vyforge.platform:platform-report-openhtmltopdf` (bộ chuyển HTML → PDF, LGPL-2.1+) |
| Package web | không có riêng. Màn quản lý template nằm ở `@vyforge/export` 0.1.0 (`…/export/admin/templates`) |
| Zone admin | không có riêng. Template được quản lý trong khu export của app |
| App liên quan | `platform-starter-export` (kéo cả hai module), esign, notification, cms-content, billing, einvoice, workflow (phụ thuộc tuỳ chọn) |

## 3. Tính năng chính

- **`ReportEngine`**:
  - `render(RenderRequest)`, `render(ReportTemplate, RenderRequest)`.
  - `renderBatch(requests, ZIP | MERGED, out)`: in hàng loạt, ghi từng PDF ra file tạm rồi gộp; tài liệu lỗi ghi vào
    `BatchResult.failures()`.
  - `check(template)` trước khi lưu hoặc phát hành.
  - Kết quả `RenderedDocument` gồm nội dung, số trang, đã ký hay chưa, phiên bản template.
- **Engine** (`spi.TemplateRenderer`):
  - `html-pdf`: Thymeleaf → HTML → `HtmlPdfConverter`. CSS paged media, font Be Vietnam Pro nhúng, tuỳ chọn PDF/A-3b.
    Có biến `vn` (`amountInWords`, `money`, `longDate`…).
  - `docx`: mẫu Word có chỗ trống `{{ … }}`, bộ lọc `money|words|date`, dòng bảng lặp `{{#each}}`.
  - `docx-pdf`: khi có `DocumentConverter`, vd. Gotenberg/LibreOffice.
- **Template theo tenant** (`spi.ReportTemplateSource`):
  - `ClasspathReportTemplates` (`META-INF/vyforge/reports/<key>/…`), `InMemoryReportTemplates`, `firstOf(...)`.
  - export cài bản lưu trong DB.
  - `RenderRequest.preview(version)`: bản nháp, không ký, có chữ chìm `PREVIEW`.
- **Ký PAdES**: `spi.PdfSigner`, `SigningProfiles`, `SecretLookup`. Có sẵn `Pkcs12PadesSigner` (PDFBox + Bouncy Castle,
  khoá PKCS#12 lấy từ secret của tenant).
- **An toàn**:
  - Template của tenant không chạy được code: từ chối `T(...)`, `new`, `@bean`, tiền xử lý `__…__`.
  - Chỉ nạp tài nguyên `asset:<tên>`, chặn http và file (chống SSRF).
  - Khoá ký không bao giờ mượn của tenant khác.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-report</artifactId>
</dependency>
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-report-openhtmltopdf</artifactId>   <!-- engine html-pdf -->
</dependency>
<!-- tuỳ chọn: org.apache.poi:poi-ooxml (docx), org.bouncycastle:bcpkix-jdk18on (ký pkcs12) -->
```

```java
RenderedDocument pdf = reports.render(RenderRequest.of(tenantId, "hr.payslip", Map.of("data", payslip))
        .withLocale(Locale.forLanguageTag("vi-VN"))
        .withFileName("phieu-luong-" + payslip.code()));
```

Template mặc định của app đặt ở `META-INF/vyforge/reports/<key>/template.html` (hoặc `.docx`), kèm `settings.json` và
`assets/*`.

## 5. Cấu hình quan trọng

Thư viện không có khoá `vyforge.*` riêng. Cấu hình nằm ở `settings.json` của template (`title`,
`engine`, `pdfA`, `watermark`, `signingProfile`) và ở module gọi (vd. hồ sơ ký `signingProfiles` của export).

## 6. Tích hợp với module khác

| SPI | Dùng để |
|---|---|
| `TemplateRenderer` | Thêm engine mới |
| `ReportTemplateSource` | Nguồn template theo tenant |
| `HtmlPdfConverter` | Thay OpenHTMLtoPDF; API chỉ dùng `org.w3c.dom.Document` |
| `PdfSigner`, `SigningProfiles`, `SecretLookup` | Ký PDF |
| `DocumentConverter` | Chuyển tài liệu sang PDF ngoài tiến trình |

- Job in hàng loạt bền vững (checkpoint, chạy lại) nằm ở export (`report.render` trên `DurableTaskQueue`).
- Thư viện không kiểm quyền và không audit: module gọi tự làm theo resource của mình.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Không có bảng, REST, action IAM hay sự kiện.

## 8. Trạng thái & còn mở

- Tách từ export ở 0.15. Ở 0.16, esign render template và seal tài liệu qua thư viện này (có cả template DOCX).
- **Giấy phép đang chờ pháp chế**:
  - OpenHTMLtoPDF là LGPL-2.1+, cô lập trong `platform-report-openhtmltopdf` (jar riêng, không sửa, không shade). App
    bỏ được module này và khai bean `HtmlPdfConverter` khác.
  - EU DSS (PAdES-B-LT/LTA) cũng là LGPL-2.1. Hiện chỉ có ở `platform-starter-esign-dss`, không nằm trong
    platform-report.
- Giới hạn: CSS 2.1, không có flexbox/grid; phải dàn trang bằng bảng.
