# Mục "export" cho ADR-0020 (module P1)

> Bản nháp để phiên điều phối gộp vào ADR-0020. Thiết kế đầy đủ: [export.md](export.md).

## export: xuất, nhập dữ liệu và báo cáo (`platform-starter-export`, `@vyforge/export`)

- **Trạng thái:** Chấp nhận, 27/10/2026. Phát hành lần đầu trong platform-java 0.13.0.

### Bối cảnh

Gần như dự án nào ở Việt Nam cũng có "Xuất Excel", "Nhập từ Excel" và in chứng từ PDF tiếng Việt. Không nền tảng tham
chiếu nào có sẵn khối này. Mỗi dự án tự viết thì lặp lại cùng một nhóm lỗi:

- tải cả bảng vào RAM;
- chạy đồng bộ quá timeout của gateway;
- link tải không hết hạn;
- nhập lỗi giữa chừng mà không có file báo lỗi, nhập lại thì sinh bản ghi trùng;
- PDF mất dấu;
- CSV injection.

### Quyết định

1. **Một starter cho ba việc** (xuất, nhập, báo cáo), dùng chung một lớp job (`data_job`), một kho file kết quả, một mô hình
   quyền và một trang "Việc của tôi". Starter nhúng vào app, không phải service: nguồn dữ liệu và đích nhập là code của
   chính app, cần chung transaction và chung quyền.
2. **Thực thi trên `DurableTaskQueue`** (queue `export`, platform-java 0.12.1).
   - Module không có hàng đợi riêng. `data_job` chỉ giữ trạng thái nghiệp vụ người dùng thấy.
   - Việc chia được (nhập, báo cáo hàng loạt) chạy theo lát có checkpoint.
   - `export.run` là một task dài, gia hạn khoá bằng `extendLock`.
   - Một job đối soát chuyển task `DEAD` thành job `FAILED`.
   - Không dùng Spring Batch: 9 bảng metadata không có tenant mà vẫn phải có bảng job riêng.
   - Không dùng JobRunr: LGPL-3.0, và các tính năng cần dùng nằm ở bản Pro.
3. **Streaming từ đầu tới cuối.** Nguồn trả dữ liệu theo trang (keyset, resumable). Writer ghi ra file tạm. Heap tỉ lệ với
   chunk, không tỉ lệ với số dòng. Định dạng:
   - XLSX ghi bằng FastExcel, đọc bằng fastexcel-reader (Apache-2.0).
   - CSV bằng FastCSV (MIT).
   - `.xls` và template Excel (JXLS) dùng POI 6, là phụ thuộc **tuỳ chọn**.
4. **Nhập qua bảng staging** (`data_import_row`):
   - Bốn bước: upload → ghép cột → xem trước → xác nhận.
   - Ba chế độ commit: `ATOMIC`, `STRICT` (mặc định), `PARTIAL`.
   - Idempotency ba tầng: job, chunk (trạng thái dòng ghi cùng transaction với dữ liệu), khoá tự nhiên + upsert.
   - Có file báo lỗi và hoàn tác có điều kiện.
5. **Báo cáo** dùng engine dạng adapter:
   - `html-pdf` (Thymeleaf + OpenHTMLtoPDF, nhúng font Be Vietnam Pro/Noto, OFL);
   - `xlsx-template` (JXLS, tuỳ chọn);
   - `gotenberg` (service tuỳ chọn).
   - Ký số qua SPI `DocumentSigner`. Có sẵn PKCS#12 PAdES (PDFBox + Bouncy Castle).
6. **Không phụ thuộc code của `-files`/`-notification`/`-realtime`.**
   - Kho file là port `ResultStore` của export. Bản đầu có adapter `local`; adapter sang files thêm sau khi files merge.
   - Thông báo và tiến độ là event công khai: `DataJobCompleted`/`DataJobFailed` qua outbox, `DataJobProgressed` không qua
     outbox.
7. **Dùng khối chung:**
   - `SecretCipher` mã hoá secret của tenant (context `export:config:<tenant>:<khoá>`);
   - `StarterSchemaMigrator` với bảng lịch sử `flyway_schema_history_export`;
   - `Authz` với action catalog `export:*`, cộng với action nghiệp vụ bắt buộc cho từng definition;
   - `AuditLog` cho việc tải file và cột dữ liệu cá nhân.
8. **Mặc định:**
   - file kết quả sống 7 ngày, link tải ký sống 5 phút;
   - chế độ nhập `STRICT` khi definition không chỉ định;
   - báo cáo định kỳ chỉ gửi cho user/group trong tenant;
   - POI là phụ thuộc tuỳ chọn.

### License cần pháp chế xác nhận

**OpenHTMLtoPDF** (`io.github.openhtmltopdf` 1.1.100) dùng license **LGPL-2.1-or-later** và là phụ thuộc compile của
starter (engine `html-pdf`).

- Platform dùng thư viện này đúng nghĩa thư viện: không sửa mã, liên kết động qua classpath, jar giữ nguyên để người dùng
  thay được.
- Việc phân phối kèm sản phẩm on-prem cho khách hàng **phải được pháp chế xác nhận** trước khi đưa lên production.
- Nếu pháp chế không chấp nhận: chuyển `html-pdf` thành phụ thuộc tuỳ chọn và dùng engine `gotenberg` (MIT) làm mặc định.
  Lõi không phải sửa.

Các phụ thuộc còn lại:

| License | Thư viện | Ghi chú |
|---|---|---|
| Apache-2.0 | FastExcel, POI, JXLS, PDFBox, Thymeleaf | |
| MIT | FastCSV, Bouncy Castle | |
| SIL OFL 1.1 | Font Be Vietnam Pro, Noto | Kèm file `OFL.txt` |

Không dùng JasperReports (LGPL-3.0). Carbone (Carbone Community License) chỉ là adapter tuỳ chọn ở bản sau, cũng cần
pháp chế duyệt.

### Hệ quả

- App chỉ khai báo `ExportDefinition`/`ImportDefinition`/`ReportDataProvider` là có REST, job, file báo lỗi, quyền, audit và
  giao diện (`ExportButton`, `ImportWizard`, `JobsTable`).
- Pod chạy worker cần đĩa tạm (`emptyDir` hoặc PVC) cỡ 2 × file lớn nhất × số việc đồng thời.
- Trước khi `-files` merge, store mặc định là `local` (thư mục dùng chung). Chạy nhiều replica cần volume RWX hoặc chờ
  adapter files.
- Để bản sau:
  - adapter ký từ xa (VNPT SmartCA, Viettel MySign), PAdES-LTA (EU DSS);
  - sửa dữ liệu trên lưới xem trước;
  - nhập nhiều sheet có quan hệ;
  - mã hoá file bằng mật khẩu;
  - engine Carbone;
  - partition bảng staging.
