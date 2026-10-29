# eInvoice — hoá đơn điện tử Việt Nam

## 1. Mục đích

Cho app phát hành hoá đơn điện tử theo NĐ 254/2026/NĐ-CP và TT 91/2026/TT-BTC (thay NĐ 123/2020, NĐ 70/2025, TT 32/2025
từ 01/07/2026) qua nhà cung cấp HĐĐT (Viettel S-Invoice, MISA meInvoice…): hồ sơ phát hành (issuer), phát hành có
mã/không mã/máy tính tiền, điều chỉnh, thay thế, thông báo sai sót Mẫu 04/SS-HĐĐT, lưu XML 10 năm. Dùng độc lập (bán lẻ,
POS) hoặc qua cầu nối của billing. Module không sinh XML gửi CQT và không làm T-VAN — nhà cung cấp sinh, ký, truyền;
module giữ dữ liệu chuẩn hoá, trạng thái, quan hệ điều chỉnh/thay thế và XML trả về. Thiết kế: `docs/design/billing.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-einvoice` | platform-java 0.16.1 |
| Package web | `@vyforge/einvoice` 0.1.0 | chỉ subpath `/admin` (không có README package) |
| Zone admin | chưa có `apps/einvoice-admin` trong platform-admin (cần xác nhận: gộp vào `billing-admin`?) | |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling`, `SecretCipher` (khoá nhà cung cấp) | tuỳ chọn: files, platform-report |

## 3. Tính năng chính

- **Issuer** (hồ sơ phát hành): MST, mẫu số, ký hiệu (vd `C26TAA`), cách đánh số, loại `with_code`/không mã/máy tính
  tiền, nhà cung cấp + môi trường, khoá truy cập mã hoá bằng `SecretCipher` (không trả lại), `:test` kết nối.
- **Phát hành** (`EInvoices.issueNow`, hoặc `POST /documents?issue=true`): việc nền phát hành, truy vấn mã CQT, tải XML.
- **Vòng đời**: `draft → issuing → issued / pending_code → accepted`; lỗi `failed` (nhà cung cấp từ chối),
  `rejected` (CQT từ chối cấp mã), `unknown` (timeout — truy vấn theo `provider_ref` trước, chỉ gửi lại khi nhà cung cấp
  chưa nhận). Gốc sang `adjusted`/`replaced` khi hoá đơn điều chỉnh/thay thế đã có số.
- **Điều chỉnh** (`:adjust` `increase`/`decrease`; `info` chỉ lập thông báo 04/SS), **thay thế** (`:replace`); người mua
  là tổ chức cần `agreementRef` (văn bản thoả thuận) trừ khi `agreement-exempt`; hoá đơn máy tính tiền chỉ được thay thế.
- **Thông báo sai sót** Mẫu 04/SS-HĐĐT (`/error-notices`).
- **Bản thể hiện PDF**: của nhà cung cấp (khi có `DOWNLOAD_PDF`) hoặc dựng qua platform-report (template
  `einvoice.document`, mặc định tiếng Việt).
- **Tra cứu MST** (`/lookup/tax-code/{mst}`).
- **Lưu trữ** XML/PDF 10 năm (Luật Kế toán): bucket `einvoice-documents` của files, hoặc bảng `einv_blob` (gzip,
  SHA-256). Không có API xoá.
- **SPI**: `EInvoiceProvider` (+ `ProviderCapability`, `ProviderDescriptor`) cho nhà cung cấp mới.
- **REST** (`/api/v1/einvoice`): `/providers`, `/issuers` (`:test`), `/documents` (`:issue`, `:sync`, `:adjust`,
  `:replace`, `:resend`, `/xml`, `/pdf`, `/history`), `/error-notices`, `/lookup/tax-code/{mst}`, `GET /admin/info`;
  callback công khai `POST /callback/{provider}/{key}`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-einvoice</artifactId>
</dependency>
```

```yaml
vyforge:
  einvoice:
    providers:
      sandbox: { enabled: false }   # bắt buộc tắt ở production
```

```java
issuers.create(new CreateIssuer("HĐ GTGT có mã", "0109999999", "Công ty A", "Hà Nội", Map.of(), "viettel",
        "sandbox", "1", "1", "C26TAA", "auto_year", "with_code", Map.of("templateCode", "1/001"),
        Map.of("username", "...", "password", "..."), true));
EInvoiceView doc = einvoices.issueNow(new EInvoiceDraft(null, SourceRef.of("order", orderId), today,
        Buyer.anonymousConsumer(), "VND", null, "TM", lines, null, Map.of(), "order:" + orderId));
// doc.status() = issuing; việc nền phát hành, truy vấn mã CQT, tải XML
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.einvoice.enabled` / `api-enabled` | `true` / `true` | |
| `vyforge.einvoice.mode` | `embedded` | |
| `vyforge.einvoice.zone` | `Asia/Ho_Chi_Minh` | |
| `vyforge.einvoice.agreement-exempt` | `false` | Tenant bán qua nền tảng số/sàn TMĐT: không bắt văn bản thoả thuận khi điều chỉnh (TT 91) |
| `vyforge.einvoice.providers.<id>.enabled` / `base-url` | `true` / theo môi trường issuer | Chỉ ở cấu hình platform (chống SSRF); **tắt `sandbox` ở production** |
| `vyforge.einvoice.query-schedule` | (cần xác nhận) | Lịch truy vấn lại hoá đơn `unknown`/`pending_code` |
| `vyforge.einvoice.representation` | dựng khi nhà cung cấp không có PDF | `provider` để chỉ dùng PDF của nhà cung cấp |

## 6. Tích hợp với module khác

| Module | Tích hợp |
|---|---|
| billing | `EInvoiceSink` (ở billing): hoá đơn → HĐĐT, credit note → HĐĐT điều chỉnh giảm/thay thế; thời điểm lập theo NĐ 254 Đ.9 |
| files | Lưu XML/PDF ở bucket `einvoice-documents` (vắng thì `einv_blob`) |
| platform-report | `RepresentationRenderer`: template `einvoice.document` (`html-pdf` cần `platform-report-openhtmltopdf`); tenant sửa qua export |
| esign | `PdfSigner` "esign" có thể đóng dấu PDF báo cáo (theo README esign) |
| payment | Theo thiết kế nghe `PaymentSucceeded` — hiện đi qua billing (cần xác nhận nối trực tiếp) |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded` (không có client `remote`).
- **IAM**: `einvoice:issuer:List|Get|Create|Update`, `einvoice:document:List|Get|Create|Issue|Adjust|Replace`,
  `einvoice:notice:List|Create`.
- **Sự kiện**: `EInvoiceIssued`, `EInvoiceAccepted`, `EInvoiceRejected`, `EInvoiceFailed`, `EInvoiceAdjusted`,
  `EInvoiceReplaced`.
- **Giấy phép**: module `licensed` (`einvoice`, từ 0.14.0).

## 8. Trạng thái & còn mở

| Nhà cung cấp | Trạng thái |
|---|---|
| `sandbox` | Đủ vòng đời, giả lập trong bộ nhớ (`behavior`: accept, pending, cqt_reject, reject, timeout…) |
| `viettel` | Viết theo tài liệu công khai, **chưa chạy với sandbox thật** |
| `misa` | Viết theo tài liệu công khai, **chưa chạy với sandbox thật** |
| `vnpt`, `bkav`, `fpt` | Chưa làm (khi có hợp đồng + sandbox) |

- Chờ đối tác: kiểm Viettel S-Invoice (xác thực Basic hay `/auth/login`, trường điều chỉnh/thay thế, `templateCode`, mã
  lỗi thật) và MISA meInvoice (token, `SignType`, `ReferenceType`, ý nghĩa `SendTaxStatus`) với sandbox thật trước pilot.
  Mã lỗi nhà cung cấp đang map theo từ khoá (đoán).
- Chưa có zone admin riêng; package web chỉ có `/admin`.
