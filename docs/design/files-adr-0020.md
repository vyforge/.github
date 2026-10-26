# Mục "files" cho ADR-0020 (module P1)

> Bản nháp để phiên điều phối gộp vào ADR-0020. Thiết kế đầy đủ: [files.md](files.md).

## files: lưu trữ file (`platform-starter-files`, `@vyforge/files`)

- **Thiết kế:** [files.md](../design/files.md). **Phát hành:** `platform-starter-files` (platform-java 0.13.0),
  `@vyforge/files` 0.1.0 (platform-web).
- **Trạng thái:** Chấp nhận, 27/10/2026.

### Quyết định

1. **Byte không đi qua app.** Trình duyệt gửi thẳng lên storage bằng presigned PUT (file nhỏ) hoặc S3 multipart có URL
   ký cho từng part (file ≥ 64 MB, resume được kể cả sau khi tải lại trang). App ký URL, rồi lúc complete kiểm lại kích
   thước (`HEAD`) và MIME thật (Tika, 64 KB đầu). Không dùng tus làm giao thức chính vì byte sẽ phải đi qua BFF và
   backend (ADR-0006); không dùng presigned POST vì AWS SDK Java v2 không hỗ trợ (issue #1493).
2. **Bucket logic có chính sách** (kích thước, loại file, quét, public/private, dữ liệu cá nhân, chia sẻ, TTL, retention,
   Object Lock, quyền theo bản ghi cha) khai báo trong cấu hình app. Tenant chỉ thu hẹp được phần an toàn và chỉnh dung
   lượng trong trần của platform.
3. **Mô hình dữ liệu:** file logic (`file_object`) – version (`file_version`) – nội dung vật lý (`file_blob`, dedup theo
   SHA-256 trong cùng tenant và storage, chỉ sau upload). Key bất biến `t/<tenant>/<bucket>/<yyyy>/<MM>/<blobId>`,
   không chứa tên file. Attachment (`file_attachment`) gắn file vào bản ghi của module khác, không có FK chéo module.
4. **Việc nền trên `DurableTaskQueue`** (queue `files`, kind `scan`, `transform`, `purge`), enqueue trong transaction
   complete. File chỉ tải được khi `AVAILABLE`; máy quét lỗi thì file đứng ở `PROCESSING` (fail-closed).
5. **Quét mã độc bằng ClamAV** (`clamd` INSTREAM, client tự viết vì client Java trên Maven Central đã bỏ từ 2022). Không
   có máy quét thì bucket `scan: required` từ chối upload.
6. **Ảnh:** imgproxy (URL ký HMAC, chỉ preset, có `expires`) hoặc biến thể sinh sẵn trong JVM (Thumbnailator); bỏ
   EXIF/GPS bằng bản `_clean`. Preview trang đầu PDF bằng PDFBox.
7. **Storage qua SPI `StorageAdapterFactory`:** `s3` (AWS SDK v2; AWS S3, SeaweedFS, RustFS, Ceph, R2, VNG vStorage),
   `azure` (SAS, Put Block/Put Block List), `gcs` (XML API tương thích S3 với HMAC key, không kéo `google-cloud-storage`),
   `local` (URL ký HMAC tới endpoint của app). Tenant tự mang storage, secret mã hoá bằng `SecretCipher`. Với endpoint
   không phải AWS, checksum mặc định của SDK ≥ 2.30 tắt về `WHEN_REQUIRED`.
8. **Quyền:** action `files:*`; bucket `access: entity` lấy quyền từ bản ghi cha (cấu hình hoặc bean
   `AttachmentAccessPolicy`). Không có quyền xem thì trả 404. Code chạy nền không có người dùng được coi là gọi hệ thống;
   `FilesSystemAccess` cho trường hợp tường minh.
9. **Retention, legal hold, xoá theo chủ thể dữ liệu** (Luật 91/2025/QH15): file của chủ thể bị xoá thật (object, biến
   thể, tên, metadata); file luật buộc giữ (vd. chứng từ theo NĐ 174/2016/NĐ-CP) được liệt kê kèm căn cứ trong
   `ErasureReport`. Legal hold có thể đặt thêm ở storage (Object Lock/immutability).
10. **Schema riêng** qua `StarterSchemaMigrator` (`flyway_schema_history_files`), SQL gộp cho DBA.
11. **Đóng gói độc lập:** không phụ thuộc module P1 nào. Notification đọc file qua `FileContentReader` (reflection,
    `@ConditionalOnBean`), export qua `FileService.store`, realtime/webhooks nghe event `com.vyforge.platform.files.event`,
    features cấp `QuotaProvider`. Admin: component ở `@vyforge/files/admin`, endpoint `GET /api/v1/files/admin/info`;
    zone `files-admin` (`/m/files`) dựng bằng `pnpm create-admin-zone` khi forge-37 xong.

### Thư viện và license

| Thư viện | Version | License | Phạm vi |
|---|---|---|---|
| AWS SDK v2 (`s3`, `url-connection-client`, `cloudfront` tuỳ chọn) | 2.57.3 | Apache-2.0 | bắt buộc |
| Apache Tika (`tika-core`) | 4.1.0 | Apache-2.0 | bắt buộc |
| Apache PDFBox | 3.0.8 | Apache-2.0 | tuỳ chọn |
| Thumbnailator / TwelveMonkeys ImageIO WebP | 0.4.21 / 3.15.3 | MIT / BSD-3 | tuỳ chọn |
| Azure Storage Blob | 12.36.0 | MIT | tuỳ chọn |
| ClamAV, imgproxy, SeaweedFS | 1.5.4 / v4.0.19 / 4.51 | GPL-2.0 (daemon riêng) / Apache-2.0 / Apache-2.0 | hạ tầng, không nhúng |

### Việc còn mở

- Chế độ `remote` của starter và API nội bộ dạng stream cho `store`/`open` (platform-services §3.5, §6.8).
- Sidecar `tusd`, quét qua ICAP, preview Office (Gotenberg), thumbnail video, nhập file từ URL.
- Job quét object mồ côi trên storage (hiện dọn theo phiên upload hết hạn và lifecycle của bucket).
- Kiểm với GCS thật (adapter `gcs` chưa chạy trên GCS, chỉ trên SeaweedFS qua cùng mã S3); Viettel IDC, FPT Cloud chưa
  kiểm tài liệu API S3.
- Thời hạn xử lý yêu cầu chủ thể dữ liệu (NĐ 356/2025) lấy từ nguồn thứ cấp: pháp chế đối chiếu toàn văn.
