# ADR-0020: Module ứng dụng dùng chung

- **Trạng thái:** Chấp nhận dần theo từng module (mỗi module một mục). Thuộc kế hoạch trong
  [MODULE_PLAN.md](../MODULE_PLAN.md) mục 4 và 8.
- **Nền chung:** module-conventions của đợt P1; khối dùng chung `SecretCipher`, `DurableTaskQueue`,
  `StarterSchemaMigrator` (platform-java 0.12.1); đóng gói độc lập và admin microfrontend theo
  [platform-services](../design/platform-services.md).

Mỗi module ghi: quyết định, thư viện và license, điểm nối với module khác, việc còn mở. Thiết kế chi tiết nằm ở
`docs/design/<module>.md`.

## webhooks

- **Thiết kế:** [webhooks.md](../design/webhooks.md). **Phát hành:** `platform-starter-webhooks-inbound`,
  `platform-starter-webhooks` (platform-java 0.13.0), `@vyforge/webhooks` 0.1.0 (platform-web).
- **Pilot:** không cần (người dùng quyết định 26/10/2026).

### Quyết định

1. **Engine gửi nhúng trong app trên PostgreSQL, không chạy Svix/Convoy/Outpost.** Chạy được ba cách: nhúng, trong
   `platform-services`, server riêng (cùng image). Port `WebhookPublisher` chừa chỗ cho adapter Svix khi vượt khoảng 5
   triệu delivery/ngày.
2. **Định dạng Standard Webhooks** (`webhook-id`, `webhook-timestamp`, `webhook-signature`; `v1` HMAC-SHA256, `v1a`
   ed25519). Ký và kiểm tự viết trên JCA; thư viện chính thức `com.standardwebhooks:standardwebhooks` 1.2.0 chỉ dùng làm
   oracle trong test (nó chỉ có `v1`, nhận `String`, tolerance cố định).
3. **Hàng đợi riêng `webhook_delivery` + lease theo endpoint (L5), không dùng `DurableTaskQueue`.** Rate limit, số
   request đồng thời và thứ tự theo key chỉ chính xác khi một endpoint tại một thời điểm do một replica gửi;
   `DurableTaskQueue` chỉ giới hạn theo queue và tenant, sẽ cần thêm slot trong DB và Valkey, và gấp đôi số lần ghi. Hàm
   claim `vyforge_webhooks_claim` (`security definer`, `SKIP LOCKED`) là truy vấn xuyên tenant duy nhất (L2).
4. **Chống SSRF ở bước kết nối:** `DnsResolver` của Apache HttpClient 5 trả đúng IP đã kiểm (chống DNS rebinding),
   chặn dải IANA đặc biệt, IPv4 nhúng (NAT64, 6to4, IPv4-mapped), dạng IP thập phân/hex; không redirect; ngoại lệ dải
   nội bộ chỉ do người vận hành cấu hình theo platform hoặc theo tenant; proxy egress tuỳ chọn.
5. **Secret, header tuỳ chỉnh, khoá mTLS mã hoá bằng `SecretCipher`** (context có tenant và id). Xoay secret có thời gian
   chồng tối đa 7 ngày, ký bằng mọi secret còn hiệu lực.
6. **Schema riêng qua `StarterSchemaMigrator`** (`flyway_schema_history_webhooks`, L1); `webhook_message`,
   `webhook_delivery`, `webhook_attempt` partition theo tháng, drop theo retention lớn nhất; retention theo tenant xoá
   theo lô.
7. **Event chứa dữ liệu cá nhân nhạy cảm** chỉ đến endpoint ghi đúng tên type và khi tenant bật `allowSensitiveEvents`
   (Luật 91/2025/QH15, NĐ 356/2025/NĐ-CP); có `allowedDomains` theo tenant và xoá theo chủ thể dữ liệu.
8. **Webhook nhận vào là artifact riêng** `platform-starter-webhooks-inbound` (không bảng, không job): `@InboundWebhook`
   kiểm chữ ký trên body thô trước handler; khối dựng cho VNPay/payOS/MoMo/ZaloPay/SePay; chống trùng qua
   `processed_messages` (L6).
9. **Không phụ thuộc cứng module khác:** `-notification` nghe event công khai; kênh webhook của `-notification` gọi
   `WebhookPublisher` khi có bean; `-payment` dùng artifact inbound (khuyến nghị optional).
10. **Admin MFE:** component ở `@vyforge/webhooks/admin` (nhận `can`), manifest `contract: 1`, endpoint
    `GET /admin/info` và `GET /admin/egress` (người vận hành); zone `webhooks-admin` (`/m/webhooks`) dựng bằng
    `pnpm create-admin-zone` khi forge-37 xong.

### Thư viện

| Thư viện | Phiên bản | License | Dùng cho |
|---|---|---|---|
| Apache HttpClient 5 | 5.6.4 (Spring Boot 4.1.1) | Apache-2.0 | Gửi webhook, `DnsResolver`, mTLS |
| `com.standardwebhooks:standardwebhooks` | 1.2.0 | Apache-2.0 | Chỉ test (oracle tương thích) |
| Bouncy Castle `bcpkix-jdk18on` | 1.86 | MIT | Chỉ test (sinh CA, chứng chỉ mTLS) |

### Còn mở

- Chế độ `remote` (D8): topic `webhooks-messages`, `PUT /internal/catalogs/{service}`; app zone `webhooks-admin`.
- Health indicator `webhooks`; che PII trong body phản hồi đã lưu; adapter đích Kafka/SQS; OAuth2 client credentials.
- `ReplayGuard` của inbound đang dùng UUID name-based (MD5, v3) chứ không phải v5 như thiết kế; chỉ là khoá khử trùng.
