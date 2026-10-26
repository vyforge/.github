# ADR-0020: Module ứng dụng dùng chung

- **Trạng thái:** Chấp nhận dần theo từng module (mỗi module một mục). Thuộc kế hoạch trong
  [MODULE_PLAN.md](../MODULE_PLAN.md) mục 4 và 8.
- **Nền chung:** module-conventions của đợt P1; khối dùng chung `SecretCipher`, `DurableTaskQueue`,
  `StarterSchemaMigrator` (platform-java 0.12.1); đóng gói độc lập và admin microfrontend theo
  [platform-services](../design/platform-services.md).

Mỗi module ghi: quyết định, thư viện và license, điểm nối với module khác, việc còn mở. Thiết kế chi tiết nằm ở
`docs/design/<module>.md`.

## notification

- **Thiết kế:** [notification.md](../design/notification.md). **Phát hành:** `platform-starter-notification`
  (platform-java 0.13.0), `@vyforge/notifications` 0.1.0 (platform-web).
- **Pilot:** không cần (người dùng quyết định 27/10/2026: platform đi trước nhu cầu).

### Quyết định

1. **Thư viện nhúng, không tự host máy chủ thông báo (Novu…).** Chạy được ba cách: nhúng vào app, trong
   `platform-services`, hoặc server riêng (cùng image).
2. **Bảy kênh qua SPI `NotificationProvider`:** email (SMTP), SMS brandname (eSMS là provider đầu tiên; SPI đủ để thêm
   brandname Viettel, VNPT, VNPT-VinaPhone), Zalo ZNS (gọi thẳng OpenAPI Zalo bằng OA của khách; reseller là provider
   khác), push (FCM HTTP v1, Web Push VAPID), in-app inbox, chat (Slack, Teams Workflows, Telegram), webhook. Thêm
   provider hay kênh không sửa lõi.
3. **Thực thi trên `DurableTaskQueue`** (queue `notification`): bảng `notification_delivery` là nguồn sự thật nghiệp vụ,
   task chỉ là lịch chạy; handler idempotent. Không dùng outbox Modulith cho việc gửi (thiếu hẹn giờ, backoff từng tin,
   ưu tiên, huỷ); outbox chỉ cho event công khai.
4. **Lưu trữ JDBC, partition theo tháng** (`notification`, `notification_delivery`, `notification_delivery_event`), job
   tạo trước và drop theo retention; schema riêng qua `StarterSchemaMigrator` (`flyway_schema_history_notification`).
   Lệch quy ước L1 đã được duyệt.
5. **Template của tenant dùng Liquid (Liqp, MIT)** có giới hạn render và chặn `include`; Thymeleaf chỉ cho template
   trong code (tránh SSTI). Lệch quy ước L2 đã được duyệt. Biến không có trong schema bị chặn lúc publish.
6. **Tuỳ chọn người nhận theo mô hình Knock**; `SECURITY`/`CRITICAL` luôn gửi; `PROMOTIONAL` cần opt-in tường minh có
   bằng chứng hoặc `PromotionalConsent`.
7. **Secret và địa chỉ mã hoá bằng `SecretCipher`**, tra cứu bằng HMAC (`vyforge.privacy.hash-key` phải cố định).
   Suppression list chỉ giữ hash.
8. **Đo mở thư/click không làm** (người dùng quyết định: mặc định tắt). Cần ý kiến pháp chế trước khi bật.
9. **`tenants-may-override-providers` mặc định `true`.**
10. **Không phụ thuộc cứng module khác:** files qua `FileContentReader` (`@ConditionalOnBean`, gọi reflection),
    realtime qua `InboxChanged`/`InboxRealtimeBridge`, webhooks qua `NotificationProvider`, privacy qua
    `PromotionalConsent`; authz/audit/ratelimit/security là optional.
11. **Admin MFE:** component ở `@vyforge/notifications/admin` (nhận `can`), endpoint `GET /admin/info`; zone
    `notification-admin` (`/m/notification`) dựng bằng `pnpm create-admin-zone` khi forge-37 xong.

### Pháp lý: cần pháp chế xác nhận

Mọi giới hạn theo NĐ 91/2020/NĐ-CP là **cấu hình**, giá trị mặc định là cách hiểu hiện tại, chưa đối chiếu toàn văn, và
**cần pháp chế của dự án xác nhận** trước khi dùng với dữ liệu thật:

| Cấu hình | Mặc định | Cách hiểu |
|---|---|---|
| `vyforge.notification.compliance.vn.promo-window` | 07:00–22:00 (giờ Việt Nam) | Khung giờ gửi tin nhắn quảng cáo |
| `compliance.vn.promo-max-per-24h`, `frequency-channels` | 3, `sms`/`email` | Không quá 3 tin nhắn, 3 thư quảng cáo / 24 giờ / người nhận |
| `compliance.vn.promo-label`, `label-channels` | `[QC]`, `sms`/`email` | Nhãn bắt buộc của tin quảng cáo |
| Từ khoá huỷ nhận SMS | `TC`, `HUY`, `STOP` | Từ chối bằng tin nhắn (Điều 16); app tự gửi tin xác nhận |
| `compliance.zalo-start`/`zalo-end` | 06:00–22:00, trừ OTP | Quy định của Zalo (nguồn reseller) |

Bộ Công an đang lấy ý kiến dự thảo nghị định thay thế NĐ 91/2020 (tên định danh, danh sách DoNotCall); khi ban hành thì
đổi cấu hình và bổ sung nguồn DNC cho suppression list.

### Thư viện

Liqp 0.9.2.3 (MIT), json-schema-validator 3.0.8 (Apache-2.0), libphonenumber 9.0.42 (Apache-2.0), jsoup 1.23.2 (MIT),
Angus Mail qua Spring Boot (EPL-2.0/GPL-2.0+CE). FCM và Web Push cài bằng JDK (không kéo firebase-admin, BouncyCastle).

### Còn mở

- Định dạng callback eSMS và chữ ký webhook Zalo chưa kiểm chứng với hệ thống thật; mã lỗi ZNS theo SDK cộng đồng.
- Chế độ `remote` của starter, `PUT /internal/catalogs/{service}`, `RecipientResolver` đọc danh bạ auth-server khi chạy
  ở `platform-services` (platform-services mục 6.8).
- App zone `notification-admin`, `NotificationTypeForm`, `PlatformDefaultsForm`, `TaskQueueTable`.
- Provider SES (bounce qua SNS), APNs trực tiếp, ZBS gửi theo Zalo UID, Kafka ingress `notification-requests`.
