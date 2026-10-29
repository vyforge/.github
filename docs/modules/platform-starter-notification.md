# platform-starter-notification — thông báo đa kênh

## 1. Mục đích

Ứng dụng nghiệp vụ nào cũng phải báo cho người dùng qua email, SMS brandname, Zalo ZNS, push, inbox trong app, kênh chat
hay webhook. Notification cho phép code nghiệp vụ chỉ gọi `Notifications.send(...)`; module chọn kênh và template, tôn
trọng tuỳ chọn và giờ yên lặng của người nhận, áp luật chống tin nhắn rác (NĐ 91/2020), gửi lại khi lỗi, chuyển kênh dự
phòng, ghi trạng thái và nhận callback của nhà cung cấp.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-notification` | Từ 0.13.0; có phí (`licensed: true`), `apiPrefix` `/api/v1/notifications` |
| Package web | `@vyforge/notifications` 0.1.0 | Inbox in-app, tuỳ chọn nhận tin, Web Push, component quản trị |
| Zone admin | `/m/notification` | Chưa có app `notification-admin` trong `platform-admin` trên `origin/main` (cần xác nhận) |
| Service | `platform-services` | Khuyến nghị chạy ở server khi có từ hai app (provider, inbox dùng chung) |

Thiết kế: [docs/design/notification.md](../design/notification.md).

## 3. Tính năng chính

- **Gửi**: `Notifications.send(NotificationRequest.of(type).to(...).data(...).idempotencyKey(...).build())`; ghi nhận trong
  transaction của người gọi (rollback thì không có tin), gửi nền trên `DurableTaskQueue` (queue `notification`).
- **Người nhận**: `Recipient.user(sub)` (contact đã đăng ký + `RecipientResolver` của app), `Recipient.email/phone(...)`
  (ad-hoc), `Recipient.object(key)` (kênh chat/webhook của đối tượng).
- **Loại thông báo** khai bằng YAML `META-INF/vyforge/notification/types/*.yml`: category (`SECURITY`, `TRANSACTIONAL`,
  `OPERATIONAL`, `PROMOTIONAL`), `dataSchema`, routing (kênh, `fallback`, `escalateAfter`, `delay`, `stopWhen`), template
  theo kênh và ngôn ngữ (Liquid-like, filter `vnd`…). Template, layout sửa được ở màn quản trị (nháp, lint, publish, xem
  trước, gửi thử).
- **Kênh và provider có sẵn**: `email` (`smtp`, có `List-Unsubscribe` RFC 8058), `sms` (`esms`), `zalo_zns` (`zalo-oa`,
  gọi thẳng OpenAPI Zalo), `push` (`fcm`, `webpush` VAPID/RFC 8291), `chat` (`slack`, `teams`, `telegram`), `webhook`
  (`http`, Standard Webhooks), `in_app` (inbox). Thêm provider bằng bean `NotificationProvider`.
- **Tuỳ chọn và luật**: tuỳ chọn `GLOBAL`/`CATEGORY`/`TYPE` × kênh, tenant đặt mặc định và ép; `SECURITY`/`CRITICAL` luôn
  gửi; `PROMOTIONAL` cần opt-in có bằng chứng, nhãn `[QC]`, khung 7h–22h, tối đa 3 tin/24 giờ/kênh/địa chỉ; ZNS không phải
  OTP chỉ 6h–22h; giờ yên lặng theo múi giờ người nhận; suppression list (chỉ lưu hash).
- **Inbox**: `/me/inbox`, số đếm (ETag), đọc/lưu trữ, event `InboxChanged`, `InboxRealtimeBridge`.
- **Đính kèm**: file (qua files) và PDF dựng từ template báo cáo lúc gửi (qua `platform-report`).
- Không đo mở thư/click (tắt theo quyết định của người dùng, cần pháp chế trước khi bật).
- REST `/api/v1/notifications`: `/me/**`, `/admin/**`, `/callbacks/{type}/{tenant}/{ref}`, `/unsubscribe?t=`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-notification</artifactId>
</dependency>
<dependency>                               <!-- tuỳ chọn: kênh email SMTP -->
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/notifications
```

```yaml
vyforge:
  crypto: { keys: { k1: "${CRYPTO_KEY_K1}" }, active-key: k1 }   # bắt buộc
  privacy.hash-key: ${PRIVACY_HASH_KEY}                           # cố định giữa các lần chạy
  notification:
    links:
      base-url: https://app.example.vn
      signing-key: ${NOTIFICATION_LINK_KEY}                       # không đặt thì email không có link huỷ nhận
    providers:
      email:
        channel: email
        type: smtp
        settings: { host: smtp.example.vn, port: 587, username: no-reply, from: "Vyforge <no-reply@example.vn>" }
        secrets: { password: "${SMTP_PASSWORD}" }
```

```java
notifications.send(NotificationRequest.of("order.shipped")
        .to(Recipient.user(order.customerSub()))
        .data("orderCode", order.code())
        .data("total", order.total())
        .idempotencyKey("order-shipped:" + order.id())
        .correlationKey("vrn:shop:order/" + order.id())
        .build());
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.notification.mode` | `embedded` | `services` khi chạy trong `platform-services` (trả ở `/admin/info`); không phải client remote |
| `vyforge.notification.links.base-url` | — | Callback của provider, link huỷ nhận |
| `vyforge.notification.links.signing-key` | — | ≥ 32 ký tự; thiếu thì email không có link huỷ nhận |
| `vyforge.notification.providers.<id>.*` | — | Provider cấp platform: `channel`, `type`, `settings`, `secrets`, `rate-limit` |
| `vyforge.notification.compliance.vn.*` | 3 tin/24 giờ, 7h–22h | Luật tin quảng cáo (cần pháp chế xác nhận) |
| `vyforge.notification.compliance.zalo-start` / `zalo-end` | `06:00` / `22:00` | Khung gửi ZNS không phải OTP |
| `vyforge.task.queues.notification.concurrency`, `max-running-per-tenant` | — | Đồng thời của queue |
| `vyforge.crypto.keys.*`, `active-key` | — | Bắt buộc: mã hoá secret provider và địa chỉ contact |
| `vyforge.privacy.hash-key` | — | Bắt buộc cố định: suppression list và khử trùng contact dựa vào hash |

Retention mặc định: notification/delivery 180 ngày, dữ liệu biến 30 ngày, callback 90 ngày, inbox đã đọc 90 ngày.

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: security, authz, audit, ratelimit, platform-report.

- **files**: có `FileContentReader` → `FilesAttachmentResolver` đính kèm email.
- **platform-report**: có `ReportEngine` → `ReportAttachmentResolver` dựng PDF từ template của tenant lúc gửi (ký số nếu
  template có `signingProfile`).
- **realtime**: `InboxRealtimeBridge` → topic `user:inbox`.
- **webhooks**: đăng ký `NotificationProvider` cho kênh `webhook`.
- **privacy-dsr**: `PromotionalConsent` (tin quảng cáo chỉ gửi khi purpose `GRANTED`); privacy-dsr gửi thông báo qua đây.
- **calendar, tenant-lifecycle, api-access, licensing**: gửi nhắc việc, OTP, thông báo vòng đời qua `Notifications.send`.
- **authz / audit / ratelimit**: API admin, audit thao tác quản trị, throttle provider chung mọi replica.
- SPI: `NotificationProvider` (+ `callbackHandler`), `RecipientResolver`, `PromotionalConsent`, `InboxRealtimeBridge`.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc chạy trong `platform-services` (khuyến nghị khi nhiều app). Chưa có client `remote` trên `origin/main`:
  app khác gửi qua REST/Kafka của server hoặc nhúng starter (cần xác nhận hướng chốt).
- **Quyền IAM**: `notification:message:Send|List|Get|GetContent|Retry|Cancel`, `notification:type:List|Get|Update|Reset`,
  `notification:template:List|Get|Update|Publish|Preview|TestSend`, `notification:provider:List|Get|Create|Update|Delete|Test`,
  `notification:suppression:List|Create|Delete`, `notification:preference:Get|Update`, `notification:stats:Get`.
- **Sự kiện**: `NotificationAccepted`, `NotificationStatusChanged`, `NotificationEngaged`, `InboxChanged`,
  `RecipientUnsubscribed`, `AddressSuppressed`.
- Job: `notification-partitions`, `notification-retention`, `notification-reconcile`, `notification-rekey`,
  `notification-zalo-token`. Metric `vyforge.notification.*`; health `notification`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.13.0 bản đầu với đủ kênh trên; 0.15.0 dựng PDF đính kèm qua `ReportEngine` dùng chung.
- **Chờ pháp chế**: cách hiểu NĐ 91/2020 (3 tin/24 giờ, 7h–22h, nhãn `[QC]`); bật đo mở/click cho tin quảng cáo.
- **Chờ đối tác/kiểm chứng**: định dạng callback eSMS chưa kiểm với callback thật; ZNS cần OA đã xác thực của khách.
- **Chờ chủ sở hữu** (thiết kế 10.2): dự án pilot, provider SMS đầu tiên (đang chọn eSMS), ZNS gọi thẳng hay qua reseller.
- **Còn thiếu**: zone admin riêng trong `platform-admin`.
