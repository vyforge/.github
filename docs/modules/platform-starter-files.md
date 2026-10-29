# platform-starter-files — lưu trữ file dùng chung

## 1. Mục đích

Hầu hết ứng dụng nghiệp vụ phải nhận file (ảnh đại diện, hồ sơ KYC, chứng từ đính kèm, file xuất/nhập) và lo cùng một bộ
việc: upload lớn có resume, kiểm MIME thật, quét virus, biến thể ảnh, preview, gắn file vào bản ghi, quyền tải, link chia
sẻ, quota, retention, legal hold, xoá theo yêu cầu chủ thể dữ liệu. Files làm các việc đó một lần trên nhiều loại storage
(S3-compatible, Azure Blob, GCS, local) để app chỉ khai báo bucket logic và gọi API.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-files` | Từ 0.13.0; có phí (`licensed: true`) |
| Package web | `@vyforge/files` 0.1.0 | Client server-only, route handler BFF, engine upload resume được, component upload/danh sách/preview/picker, component quản trị |
| Zone admin | `/m/files` | Chưa có app `files-admin` trong `platform-admin` trên `origin/main`; component có trong `@vyforge/files/admin` (cần xác nhận) |
| Hạ tầng | S3-compatible (SeaweedFS, RustFS, Ceph RGW, AWS S3, R2…), ClamAV (clamd), imgproxy (tuỳ chọn) | |
| Service | `platform-services` | `vyforge.files.deployment=services` khi chạy ở đó |

Thiết kế: [docs/design/files.md](../design/files.md) (đã duyệt 26/10/2026). Module chưa có README trên `origin/main`.

## 3. Tính năng chính

- **Upload**: presigned PUT cho file nhỏ, S3 multipart có presigned từng part cho file lớn (resume, ≤ 100 part mỗi lần
  ký), upload phía server dạng stream (`FileService.store`), idempotent.
- **Dữ liệu**: metadata, bucket logic, version (khôi phục version cũ = version mới trỏ blob cũ), dedup SHA-256 trong
  tenant.
- **Xử lý bất đồng bộ** (queue `files` của `DurableTaskQueue`): kiểm MIME thật (Tika), quét ClamAV, metadata ảnh/PDF, biến
  thể ảnh (imgproxy hoặc trong JVM), preview trang đầu PDF (PDFBox).
- **Gắn vào bản ghi** (`Attachments`): `attach`, `detach`, `list`, `reorder`, `detachAll`, `copy` theo `EntityRef` của
  platform-core; quyền suy ra từ bản ghi cha (`access: entity`).
- **Truy cập**: `downloadLink` (URL ký qua storage hoặc CloudFront, TTL ngắn), public/private, link chia sẻ có hạn
  (`ShareLinks`), audit khi tải (bucket nhạy cảm).
- **Vòng đời**: thùng rác, khôi phục, purge; retention, legal hold (`FileRetention`); quota theo tenant và bucket; dọn
  upload dở và object mồ côi. Từ 0.16.0 đi theo xoá mềm của jpa-extras (xoá/khôi phục/purge bản ghi cha kéo theo file;
  file có legal hold chặn purge bằng `PurgeVeto`).
- **Dữ liệu cá nhân** (`FileErasure`): tìm và xoá file theo chủ thể (`findForSubject`, `eraseForSubject` có dry-run).
- **Storage riêng của tenant** (BYO storage), cấu hình theo tenant (`TenantFileSettings`).
- REST `/api/v1/files` (+ `/uploads`, `/attachments`, `/admin`).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-files</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/files
```

```yaml
vyforge:
  files:
    default-storage: main
    storages:
      main:
        type: s3
        endpoint: http://seaweedfs:8333
        public-endpoint: https://files.example.vn
        bucket: vyforge-files
        path-style: true
        access-key: ${S3_ACCESS_KEY}
        secret-key: ${S3_SECRET_KEY}
    buckets:
      kyc:
        visibility: private
        max-size: 10MB
        allowed-types: [image/jpeg, image/png, application/pdf]
        scan: required
        personal-data: sensitive
        sharing: disabled
        audit-downloads: true
```

```java
FileView stored = files.store(StoreFile.of("kyc", "cccd.pdf", "application/pdf"), ContentSource.ofPath(tmp));
attachments.attach(stored.id(), AttachTarget.of(EntityRef.of("crm:customer", customerId)), Map.of());
DownloadLink link = files.downloadLink(stored.id(), DownloadOptions.ATTACHMENT);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.files.enabled` | `true` | Tắt hẳn module |
| `vyforge.files.api.enabled` | `true` | Tắt REST, chỉ dùng Java API |
| `vyforge.files.deployment` | `embedded` | `services` khi chạy trong `platform-services` |
| `vyforge.files.default-storage` | `main` | Storage của bucket không khai `storage` |
| `vyforge.files.storages.<tên>.*` | — | `type` (`s3`, `azure`, `gcs`, `local`), endpoint, khoá |
| `vyforge.files.buckets.<bucket>.*` | — | `visibility`, `max-size`, `allowed-types`, `scan`, `image`, `personal-data`, `sharing`, `audit-downloads`, `dedup`, `download-ttl`, `on-entity-deleted` |
| `vyforge.files.limits.max-file-size` | `5GB` | Trần platform |
| `vyforge.files.upload.multipart-threshold` | `64MB` | |
| `vyforge.files.delivery.download-ttl` | `5m` | |
| `vyforge.files.trash.retention` | `30d` | |
| `vyforge.files.share.max-ttl` | `30d` | |
| `vyforge.files.scan.clamav.host` / `port` | `localhost` / `3310` | |
| `vyforge.files.scan.max-size` | `100MB` | ≤ `StreamMaxLength` của clamd |
| `vyforge.files.events.externalize` | `false` | Event ra Kafka topic `vyforge.files` |
| `vyforge.files.schema.migrate` | `true` | |

## 6. Tích hợp với module khác

- **jpa (jpa-extras)**: file đi theo xoá mềm, khôi phục, purge của bản ghi cha; `PurgeVeto` khi có legal hold.
- **privacy-dsr**: gói xuất DSR ở bucket `privacy-packages`; `FileErasure` cho yêu cầu xoá.
- **tenant-lifecycle**: gói xuất ZIP và PDF biên bản ở bucket `tenant-lifecycle`.
- **settings**: `FileRefUrlResolver` trả URL logo cho trang công khai.
- **ai, esign**: đồng bộ với files, chính sách bucket (0.16.0).
- **realtime / notification / webhooks**: nghe event `FileAvailable`, `FileQuarantined` để đẩy tiến độ, báo admin, chuyển
  tiếp ra ngoài.
- **ratelimit** (tuỳ chọn).
- Adapter có sẵn: S3, Azure Blob, GCS, local, ClamAV, CloudFront, imgproxy, JVM image, PDFBox, Tika. SPI trong
  `com.vyforge.platform.files.spi` (storage, scanner, transformer, preview, validator).

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc trong **`platform-services`** (`deployment: services`). Chế độ `remote` cho app khác (cần xác nhận:
  không thấy auto-config remote trên `origin/main`).
- **Quyền IAM** (`FilesActions`): `files:file:Upload|List|Get|Download|Update|Delete|Purge|PutLegalHold`,
  `files:share:Create|List|Revoke`, `files:attachment:Manage`, `files:settings:Get|Update`, `files:storage:Manage`,
  `files:task:Manage`, `files:subject:Get`. Thuộc tính tài nguyên: `resource.bucket`, `resource.ownerSub`,
  `resource.subjectRef`, `resource.personalData`, `resource.status`, `resource.entityType`.
- **Sự kiện** (outbox, không chứa tên file hay PII): `FileUploaded`, `FileAvailable`, `FileQuarantined`, `FileRejected`,
  `FileTrashed`, `FileRestored`, `FilePurged`, `FileAttached`, `FileDetached`, `FileLegalHoldChanged`,
  `FileRetentionExpired`, `FileTaskFailed`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.13.0 bản đầu; 0.16.0 dùng `EntityRef` của platform-core (**BREAKING**, migration `files V2` dừng nếu dữ liệu
  cũ không hợp lệ), đi theo xoá mềm, sửa race khi dedup.
- **Để bản sau** (thiết kế 1.2): sidecar `tusd`, nhập file từ URL, thư mục ảo, tìm toàn văn nội dung, preview Office qua
  Gotenberg, thumbnail video, quét ICAP, watermark/DRM, chuyển tầng hot/cold, trình chỉnh ảnh.
- **Còn thiếu tài liệu**: README module; zone admin riêng.
