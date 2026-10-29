# platform-starter-webhooks — webhook gửi ra (+ webhooks-inbound: nhận vào)

## 1. Mục đích

Đối tác và hệ thống của khách hàng cần được báo khi có sự kiện nghiệp vụ (đơn đã thanh toán, vận đơn đổi trạng thái).
**Webhooks** gửi ra theo chuẩn Standard Webhooks, bền vững (ghi cùng transaction nghiệp vụ), có retry, thứ tự theo key,
chống SSRF, để tenant admin tự đăng ký endpoint, xem nhật ký, gửi lại, xoay secret mà không cần đội vận hành.
**Webhooks-inbound** là chiều ngược lại: kiểm chữ ký và chống replay cho webhook nhận vào (IPN cổng thanh toán, callback
nhà mạng SMS, Zalo OA, đối tác dùng Standard Webhooks).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter gửi ra | `platform-starter-webhooks` | Từ 0.13.0; có phí (`licensed: true`) |
| Starter nhận vào | `platform-starter-webhooks-inbound` | Từ 0.13.0; không thu phí; `apiPrefix` `/webhooks/in`. Gói `…webhooks.signature` là Java thuần, dùng được không cần Spring |
| Package web | `@vyforge/webhooks` 0.1.0 | Client API, Server Action; `/admin`: endpoint, nhật ký gửi, gửi thử, replay, secret, chứng chỉ, cấu hình |
| Zone admin | `/m/webhooks` | README nhắc `platform-admin/apps/webhooks-admin` nhưng chưa có trên `origin/main` (cần xác nhận) |
| Service | `platform-services` | Khuyến nghị chạy ở server khi có từ hai app (dispatcher dùng chung) |

Thiết kế: [docs/design/webhooks.md](../design/webhooks.md); quyết định: ADR-0020 mục "webhooks".

## 3. Tính năng chính

**Gửi ra**
- Định dạng Standard Webhooks: header `webhook-id`, `webhook-timestamp`, `webhook-signature`; payload
  `{"id","type","version","timestamp","data"}`; ký HMAC-SHA256 hoặc ed25519.
- **Khai báo event type** bằng `WebhookEventCatalog` (version, mô tả vi/en, nhóm, JSON Schema, ví dụ, mức dữ liệu cá nhân);
  `PayloadRenderer` dựng payload version cũ cho endpoint đã ghim.
- **Publish** `webhooks.publish(WebhookMessage...)` (idempotency key, ordering key, channel, subject) hoặc
  `@WebhookEvent` trên event domain; ghi delivery cùng transaction (bảng `webhook_delivery` là outbox), rollback thì không
  gửi.
- Bộ gửi nền lease theo endpoint: rate limit, số request đồng thời, thứ tự theo key chính xác giữa nhiều replica mà không
  cần Redis.
- Retry theo lịch Standard Webhooks (~3 ngày, jitter ±20 %), `Retry-After`, circuit breaker, giảm tốc khi 429/5xx, 410 tắt
  endpoint, tự tắt sau 5 ngày lỗi liên tục, dead-letter, retry tay, replay theo khoảng thời gian (giữ nguyên `webhook-id`).
- **Chống SSRF**: chỉ `https`, cổng được phép, chặn dải nội bộ (IANA), kiểm DNS và kết nối cùng một kết quả (chống DNS
  rebinding), không theo redirect; ngoại lệ dải nội bộ do người vận hành cấu hình; proxy egress tuỳ chọn. mTLS theo endpoint.
- Event type `SENSITIVE` chỉ tới endpoint ghi đúng tên type, và chỉ khi tenant bật `allowSensitiveEvents`.
- Xoay secret có thời gian chồng (`secret-overlap`), cấu hình theo tenant (`PUT /settings`), xoá theo chủ thể
  (`/messages/purge-subject`). Giao **ít nhất một lần**.

**Nhận vào**
- `@InboundWebhook("<nguồn>")` + tham số `VerifiedWebhook`: đọc body thô, kiểm chữ ký trên đúng các byte đó trước khi
  handler chạy; `webhook.read(Type.class)`, `json()`, `parameter(name)`, `tenant()`.
- Nguồn khai bằng cấu hình: `standard-webhooks`, `hmac-header` (header chữ ký, timestamp, mẫu chuỗi ký), `api-key`; hoặc
  bean `InboundWebhookSource` cho kiểu ký trên tham số (VNPay, MoMo, ZaloPay, payOS; payment đã có sẵn).
- Lỗi: 401 `webhooks.inbound.invalid_signature`/`expired`, 403 `ip_denied`, 413 `too_large`.
- Chống trùng `webhook.firstDelivery()`: bảng `processed_messages` (có messaging), Redis, hoặc bean `ReplayGuard`.
- `SecretResolver` theo tenant (secret do tenant dán, mã hoá bằng `SecretCipher`).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-webhooks</artifactId>
</dependency>
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-webhooks-inbound</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/webhooks
```

```yaml
vyforge:
  crypto: { keys: { k2026a: "${CRYPTO_KEY_2026A}" }, active-key: k2026a }
  webhooks.inbound.sources:
    partner-x: { type: standard-webhooks, secrets: ["${PARTNER_X_WHSEC}"] }
  security.public-paths: /webhooks/in/**
```

```java
@Transactional
public Order markPaid(UUID orderId) {
    Order order = orders.markPaid(orderId);
    webhooks.publish(WebhookMessage.builder("order.paid", OrderPaidPayload.from(order))
            .idempotencyKey("order.paid:" + orderId)
            .orderingKey(orderId.toString())
            .build());
    return order;
}

@PostMapping("/webhooks/in/partner-x")
@InboundWebhook("partner-x")
ResponseEntity<Void> partner(VerifiedWebhook webhook) {
    if (webhook.firstDelivery()) { handle(webhook.read(PartnerEvent.class)); }
    return ResponseEntity.ok().build();
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.webhooks.enabled` | `true` | |
| `vyforge.webhooks.delivery.enabled` | `true` | `false`: replica này không gửi (vẫn publish) |
| `vyforge.webhooks.delivery.poll-interval` / `max-leased-endpoints` | `500ms` / `64` | |
| `vyforge.webhooks.defaults.timeout` | `15s` | |
| `vyforge.webhooks.defaults.rate-limit-per-second` / `max-in-flight` | `20` / `4` | Mặc định mỗi endpoint |
| `vyforge.webhooks.defaults.disable-after` | `5d` | Tự tắt endpoint lỗi liên tục |
| `vyforge.webhooks.defaults.max-endpoints-per-tenant` | `20` | |
| `vyforge.webhooks.limits.max-payload` | `256KB` | |
| `vyforge.webhooks.retention.attempt-days` / `message-days` | `30` / `30` | |
| `vyforge.webhooks.egress.allow-http` | `false` | Chỉ dev |
| `vyforge.webhooks.egress.allowed-ports` | `443, 8443` | |
| `vyforge.webhooks.egress.tenant-allow-cidrs.<tenant>` | — | Dải nội bộ được phép (khách on-prem qua VPN) |
| `vyforge.webhooks.egress.proxy` | — | Proxy egress (smokescreen) cho IP nguồn cố định |
| `vyforge.webhooks.api.publish-enabled` / `reveal-secret` | `false` / `true` | |
| `vyforge.webhooks.inbound.max-body` | `256KB` | |
| `vyforge.webhooks.inbound.sources.<nguồn>.*` | — | `type`, `secrets`, `allowed-ips`, `signature-header`, `tolerance` (`5m`)… |

## 6. Tích hợp với module khác

- **notification**: nghe `WebhookEndpointDisabled`, `WebhookDeliveryDead`, `WebhookClientCertificateExpiring`… để báo tenant
  admin; webhooks đăng ký `NotificationProvider` cho kênh `webhook` của notification.
- **payment**: dùng webhooks-inbound cho IPN (nguồn VNPay, MoMo, ZaloPay, payOS, SePay có sẵn trong payment).
- **organization**: nhận event `auth.tenant.member_removed`/`auth.account.disabled` của auth-server qua webhooks-inbound.
- **messaging** (tuỳ chọn cho inbound): bảng `processed_messages` chống trùng.
- **files, collaboration, api-access…**: khai catalog event để phát ra ngoài.
- SPI: `WebhookInterceptor`, `PayloadRenderer`, `WebhookDestination` (Kafka, SQS…), `EgressPolicy`; inbound:
  `InboundWebhookSource`, `SecretResolver`, `ReplayGuard`.

## 7. Chế độ triển khai, quyền, sự kiện

- Webhooks: **embedded** hoặc trong `platform-services` (khoá `vyforge.webhooks.api.mode`: `embedded`/`services`, chỉ để
  báo ở `/admin/info`). Chưa có client `remote` trên `origin/main`. Inbound luôn nhúng ở app nhận webhook.
- **Quyền IAM** (webhooks): `webhooks:endpoint:List|Get|Create|Update|Delete|Pause|Resume|Enable|Disable|RevealSecret|RotateSecret|Test|Replay`,
  `webhooks:event-type:List|Get|Subscribe`, `webhooks:delivery:List|Get|Retry|Cancel`,
  `webhooks:message:List|Get|GetPayload|Publish|PurgeSubject`, `webhooks:settings:Get|Update`,
  `webhooks:certificate:List|Create|Delete`. REST chỉ bật khi có platform-starter-authz. Inbound không có action IAM
  (chữ ký thay xác thực).
- **Sự kiện**: `WebhookEndpointChanged`, `WebhookEndpointDisabled`, `WebhookEndpointRecovered`, `WebhookDeliveryDead`,
  `WebhookSecretRotated`, `WebhookClientCertificateExpiring`.
- Metric `vyforge.webhooks.*` (gồm `egress.blocked`), `vyforge.webhooks.inbound.verifications{source,outcome}`; job
  `webhooks-partitions`, `webhooks-retention`, `webhooks-auto-disable`, `webhooks-replays`, `webhooks-reencrypt`,
  `webhooks-certificates`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.13.0 cả hai starter; tương thích chữ ký kiểm bằng thư viện chính thức `standardwebhooks` 1.2.0 (scope test).
- **Còn mở**: zone admin riêng chưa có trên `origin/main`; thiết kế ghi "chờ duyệt" (26/10/2026), cần xác nhận đã duyệt.
- **Khuyến nghị vận hành**: production thêm `NetworkPolicy` egress (chỉ mở 443 ra ngoài) hoặc proxy smokescreen
  `enforce`. Đối tác phải khử trùng theo `webhook-id` (giao ít nhất một lần).
