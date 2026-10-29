# platform-starter-export — xuất, nhập dữ liệu và báo cáo

## 1. Mục đích

Màn hình danh sách nào cũng có nút "Xuất Excel", nhiều nghiệp vụ cần "Nhập từ file" và in báo cáo PDF. Export gom ba việc
đó vào một starter vì chúng dùng chung job engine, kho file kết quả, mô hình quyền và trang "Việc của tôi": xuất CSV/XLSX/
PDF/JSONL dạng stream (RAM không phụ thuộc số dòng), nhập CSV/XLSX/XLS có ghép cột, kiểm từng dòng, xem trước, hoàn tác,
và báo cáo từ template (HTML → PDF, Excel), render đơn lẻ, hàng loạt hoặc theo lịch.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-export` | Từ 0.13.0; có phí (`licensed: true`), `apiPrefix` `/api/v1/reports` |
| Thư viện báo cáo | `platform-report`, `platform-report-openhtmltopdf` | Từ 0.15.0 engine báo cáo tách ra; OpenHTMLtoPDF (LGPL-2.1+) chỉ nằm trong artifact riêng, exclude được |
| Package web | `@vyforge/export` 0.1.0 | Client, route handler, Server Action, nút xuất, trang việc, wizard nhập, khu quản trị |
| Zone admin | `/m/export` | Chưa có app `export-admin` trong `platform-admin` trên `origin/main` (cần xác nhận) |

Thiết kế: [docs/design/export.md](../design/export.md). Module chưa có README trên `origin/main`.

## 3. Tính năng chính

- **Xuất**: khai báo `ExportDefinition` (cột, định dạng, quyền, filter là record + Bean Validation, nguồn dữ liệu).
  Nguồn `KeysetSource` (resumable theo checkpoint), `StreamSource`; ghi `csv`, `xlsx`, `pdf` (bảng), `jsonl`. File nhỏ
  trả đồng bộ (≤ `sync-max-rows`), lớn chạy job. Định dạng theo locale/múi giờ, cột PII có audit, `FormulaGuard` chống
  CSV injection.
- **Nhập**: `ImportDefinition` (field có alias để tự ghép cột, kiểu, ràng buộc), đọc CSV/XLSX/XLS, ghép cột có lưu mapping,
  kiểm từng dòng, xem trước, ba chế độ commit (`CommitMode`), upsert, file báo lỗi, hoàn tác có điều kiện
  (`UndoSupport`), chống nhập trùng trong `duplicate-window`.
- **Báo cáo**: template theo tenant có version (Thymeleaf HTML → PDF qua `platform-report`, Excel), font tiếng Việt,
  header/footer, watermark; in đơn lẻ, in hàng loạt, ký số PDF (PAdES) qua SPI `DocumentSigner`; Gotenberg tuỳ chọn.
- **Lịch**: báo cáo/xuất định kỳ, người nhận trong tenant, tự tắt khi lỗi lặp lại.
- **Job engine**: chạy trên `DurableTaskQueue` (queue `export`); bảng `data_job` giữ trạng thái nghiệp vụ (tiến độ,
  checkpoint, file, bước chờ người dùng); huỷ, chạy lại, hết hạn file kết quả.
- REST (base `vyforge.export.api.base-path`, mặc định `/api/v1`): `/exports`, `/imports`, `/reports`, `/data-jobs`
  (`/{id}/files/{role}`), `/export-schedules`, `/export/admin`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-export</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/export
```

Cần `platform-starter-authz`, `-scheduling`, `-messaging`, `-jpa` (PostgreSQL).

```java
@Bean
ExportDefinition<CustomerRow, CustomerFilter> customerExport(CustomerQueries queries) {
    return ExportDefinition.<CustomerRow, CustomerFilter>builder("crm.customers", CustomerFilter.class)
            .title("export.crm.customers")
            .action(CrmActions.CUSTOMER_EXPORT)
            .column(Column.text("code", "customer.code", CustomerRow::code))
            .column(Column.text("phone", "customer.phone", CustomerRow::phone))
            .column(Column.money("debt", "customer.debt", CustomerRow::debt))
            .formats("xlsx", "csv", "pdf")
            .source(KeysetSource.of(
                    (filter, after, limit) -> queries.page(filter, after, limit),
                    row -> Checkpoint.of("id", row.id()),
                    filter -> queries.count(filter)))
            .build();
}
```

```tsx
<ExportButton definition="crm.customers" filter={filter} formats={["xlsx", "csv"]} />
```

(Chữ ký `PageQuery` của `KeysetSource` và props của `ExportButton` cần đối chiếu khi dùng thật.)

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.export.enabled` | `true` | |
| `vyforge.export.api.base-path` | `/api/v1` | Gốc REST |
| `vyforge.export.worker.chunk-size` / `import-chunk-size` | `1000` / `500` | Dòng mỗi trang |
| `vyforge.export.worker.max-export-duration` | `2h` | |
| `vyforge.export.store.type` | `local` | Kho file kết quả (`local` hoặc bean `ResultStore` tự cài) |
| `vyforge.export.store.download-link-ttl` | `5m` | |
| `vyforge.export.defaults.max-export-rows` / `max-import-rows` | `2000000` / `200000` | Mặc định cấp platform cho cấu hình tenant |
| `vyforge.export.defaults.max-upload-size` | `50MB` | |
| `vyforge.export.defaults.max-running-jobs-per-user` | `2` | |
| `vyforge.export.defaults.sync-max-rows` | `5000` | Ngưỡng xuất đồng bộ |
| `vyforge.export.defaults.result-ttl` / `job-ttl` | `7d` / `90d` | |
| `vyforge.export.defaults.locale` / `zone` | `vi` / `Asia/Ho_Chi_Minh` | |
| `vyforge.export.defaults.csv-bom` | `true` | BOM cho Excel đọc UTF-8 |
| `vyforge.export.defaults.pdf-max-rows` | `20000` | |
| `vyforge.export.report.sync-max-pages` / `sync-timeout` | `50` / `20s` | |
| `vyforge.task.queues.export.*` | — | Đồng thời, retry của queue `export` |

## 6. Tích hợp với module khác

- **platform-report**: engine báo cáo dùng chung (einvoice, billing, notification, cms-content cũng dùng). Các kiểu cũ
  `spi.ReportEngine`, `ReportTemplate`, `DocumentSigner`… còn là alias `@Deprecated`, sẽ bị xoá ở bản sau.
- **audit** (tuỳ chọn): ghi audit cột PII khi xuất, lượt tải.
- **organization, settings, custom-fields…**: module khác khai `ExportDefinition`/`ImportDefinition` của mình (vd.
  organization xuất/nhập đơn vị, phân công).
- **files**: thiết kế cho `store.type=files`; trên `origin/main` chỉ thấy `LocalResultStore` (cần xác nhận).
- **realtime / notification**: nghe `DataJobProgressed`, `DataJobCompleted` (có `templateKey="export.job.completed"`) để đẩy
  tiến độ và gửi thông báo.
- SPI: `ExportSource`, `RowWriterFactory`, `RowReaderFactory`, `ImportTarget`, `ExportInterceptor`, `ExportAuthorizer`,
  `ExportAuditor`, `ResultStore`, `ReportDataProvider`, `DocumentSigner`.

## 7. Chế độ triển khai, quyền, sự kiện

- **Luôn nhúng** vào app (D10 của thiết kế platform-services: nguồn dữ liệu là của app). Không có chế độ remote.
- **Quyền IAM**: `export:data:Export|Import|UndoImport`, `export:report:Print|PrintBatch`, `export:job:ListAll|ManageAll|DownloadAll`,
  `export:schedule:Manage|ListAll`, `export:template:Manage|Publish`, `export:config:Get|Update`. Mỗi định nghĩa còn yêu cầu
  action nghiệp vụ riêng của app (vd. `crm:customer:Export`).
- **Sự kiện** (`…export.events`, `TenantAwareEvent`): `DataJobSubmitted`, `DataJobProgressed` (không bền), `DataJobStateChanged`,
  `DataJobCompleted`, `DataJobFailed`, `ImportCommitted`, `ExportFileDownloaded`, `DataJobScheduleDisabled`. Link tải trong
  event là đường dẫn API (`/api/v1/data-jobs/{id}/files/RESULT`), không phải URL ký sẵn.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.13.0 bản đầu; 0.15.0 tách engine báo cáo sang `platform-report`, sửa race header khi trả 202.
- **Còn mở**: thiết kế ghi "chờ duyệt" (26/10/2026), cần xác nhận đã duyệt; chưa có README, chưa có zone admin riêng; kho
  kết quả qua files (thiết kế) chưa thấy trong code.
- **Ngoài phạm vi**: BI/pivot, trình thiết kế report kéo thả, ETL nhiều bước, nhập nhiều entity có quan hệ trong một file.
- **Giấy phép thư viện**: OpenHTMLtoPDF là LGPL-2.1+; dịch vụ không muốn thành phần LGPL thì exclude
  `platform-report-openhtmltopdf` và cung cấp bean `HtmlPdfConverter` khác.
