# Payment — thanh toán, hoàn tiền, định kỳ, sổ cái, đối soát

## 1. Mục đích

Cho app thu tiền qua các cổng Việt Nam (VNPay, MoMo, ZaloPay, VietQR qua payOS/SePay) và quốc tế (Stripe) theo một mô
hình chung: ý định thanh toán (`PaymentIntent`) tách khỏi lần thử (`PaymentAttempt`), trạng thái tiền chỉ đổi khi có
bằng chứng từ server của cổng (IPN đã kiểm chữ ký hoặc truy vấn chủ động — Return URL không bao giờ đổi trạng thái), mọi
biến động ghi vào sổ cái bút toán kép append-only, ba lớp chống sai lệch (janitor, đối soát sao kê, kiểm toàn vẹn sổ cái).
App là đơn vị chấp nhận thanh toán ký hợp đồng trực tiếp với cổng, không phải trung gian thanh toán. Thiết kế:
`docs/design/payment.md` (đã duyệt 26/10/2026).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-payment` | platform-java 0.16.1 (có từ 0.13.0) |
| Package web | `@vyforge/payment` 0.2.0 | `/server` (client, Server Action), `/ui` (`PaymentMethodPicker`, `PaymentQr`, `PaymentStatus`…), `/admin` |
| Zone admin | chưa có `apps/payment-admin` trong platform-admin (component `/admin` dùng được trong app; cần xác nhận kế hoạch zone) | |
| Phụ thuộc bắt buộc | PostgreSQL, `-jpa`, `-web`, `-scheduling`, `-messaging`, `-authz`, `-audit` | tuỳ chọn: master-data, resilience4j (mạch ngắt), micrometer |

## 3. Tính năng chính

- **Intent và attempt**: `PaymentService.createIntent` (idempotent theo `idempotencyKey`, `clientSecret` trả một lần),
  `startAttempt` trả `nextAction` (`redirect`, `qr`, `deeplink`, `none`), `availableMethods`, `cancel`, `sync` (truy
  vấn cổng ngay), `resolveReview`, `recordManual` (tiền mặt, chuyển khoản đối chiếu tay), `matchUnallocated` (ghép tiền
  chuyển sai nội dung), `summaryByReference`.
- **Cổng có sẵn** (SPI `PaymentProvider` khai báo capability, lõi không `if provider == X`): `vnpay`, `momo`,
  `zalopay`, `payos`, `sepay` (VietQR chuyển khoản), `stripe`, `manual`.
- **Merchant theo tenant**: nhiều merchant account mỗi cổng (theo chi nhánh/pháp nhân), `attributes` để định tuyến,
  `environment` sandbox/production, xoay `notifyKey` của URL IPN, nút kiểm kết nối; merchant mặc định của platform; trường
  `lookup:bank` lấy danh sách ngân hàng từ master-data.
- **Hoàn tiền** (`RefundService`): một phần/toàn phần, nhiều lần qua API cổng; cổng không hỗ trợ thì ghi hoàn tay có
  chứng từ.
- **Định kỳ**: mandate (token, liên kết ví) và lịch thu (`schedules`, pause/resume/cancel) — tự trừ khi cổng hỗ trợ,
  hoặc gửi link/QR mỗi kỳ.
- **Sổ cái** bút toán kép (DB từ chối bút toán lệch), hệ tài khoản mặc định, `PostingRules` thay được; bút toán tay,
  đảo bút toán.
- **Đối soát**: janitor truy vấn lại giao dịch treo (`payment-sweep`), đối soát tự động hằng ngày với cổng có
  `STATEMENT_API` hoặc nhập sao kê CSV, xử lý từng dòng lệch; kiểm toàn vẹn sổ cái.
- **Chính sách**: thanh toán trùng (`review`/auto refund), thiếu tiền (`wait_then_review`, dung sai), tiền tố nội dung
  chuyển khoản (`VF`), cửa sổ hoàn tiền.
- **Hook**: `PaymentInterceptor` (`beforeCreateIntent`, `beforeAttempt`, `afterTransition`, `beforeRefund`),
  `PaymentRouter` (mặc định `PriorityRouter`), `PostingRules`, `DuplicatePaymentPolicy`, `UnderpaymentPolicy`,
  `TransferMemoFormat`, `PaymentReferenceDescriber`, `AttributeDefinition`.
- **REST** (`/api/v1/payments`): `intents` (CRUD, `attempts`, `methods`, `cancel`, `sync`, `review`,
  `manual-payments`, `manual-refunds`, `by-reference/{type}/{id}`), `refunds`, `mandates`, `schedules`, `providers`,
  `merchant-accounts` (+ `rotate-notify-key`, `test`), `settings`, `provider-events` (+ `match`),
  `reconciliation/runs|statements|items`, `ledger/accounts|transactions` (+ `reverse`), `admin/info`; công khai:
  `notify/**` (IPN), `return/{provider}/{notifyKey}`, `intents/{id}/status`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-payment</artifactId>
</dependency>
```

```yaml
vyforge:
  payment:
    public-base-url: https://shop.example.vn   # dựng URL IPN và Return URL; production phải https
    allowed-return-origins: [https://shop.example.vn]
```

```java
CreatedIntent intent = payments.createIntent(new CreateIntent(
        new Reference("order", order.id().toString()), Money.vnd(250_000),
        customerRef, "Đơn hàng " + order.code(), null, null, Map.of("branch", "HN"), "order-" + order.id()));
AttemptStarted started = payments.startAttempt(intent.id(),
        new StartAttempt("vnpay_qr", null, null, URI.create("https://shop.example.vn/thanh-toan/ket-qua"),
                Locale.of("vi"), clientIp, null, "web"));

@ApplicationModuleListener
void on(PaymentSucceeded event) { orders.markPaid(event.reference()); }
```

Merchant account của tenant tạo qua `POST /api/v1/payments/merchant-accounts` (form dựng từ JSON Schema của provider).
Frontend: `PaymentMethodPicker` → `PaymentNextAction`/`PaymentQr` → `PaymentStatus` (SSE hoặc poll).

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.payment.enabled` / `api-enabled` | `true` / `true` | |
| `vyforge.payment.public-base-url` | `http://localhost:8080` | Gốc URL IPN/Return; production phải https |
| `vyforge.payment.allowed-return-origins` | chỉ `public-base-url` | Origin được làm `returnUrl` |
| `vyforge.payment.public-endpoints` | `true` | Tự mở `notify/**`, `return/**`, `intents/*/status` |
| `vyforge.payment.deployment-mode` | `embedded` | `services` khi chạy trong platform-services (báo qua `admin/info`) |
| `vyforge.payment.defaults.intent-expiry` / `max-attempts` | `30m` / 10 | Mặc định tenant |
| `vyforge.payment.defaults.duplicate-policy` / `underpayment-policy` / `underpayment-tolerance` | `review` / `wait_then_review` / 0 | |
| `vyforge.payment.defaults.transfer-memo-prefix` | `VF` | Tiền tố nội dung chuyển khoản |
| `vyforge.payment.defaults.allow-platform-merchant` / `refund-window` | `false` / `0s` | |
| `vyforge.payment.attempt.default-expiry` | `15m` | Hạn attempt khi cổng không có mặc định |
| `vyforge.payment.routing.match-attributes` | rỗng | Khoá metadata so với `attributes` của merchant |
| `vyforge.payment.sweep.cron` | `0 */15 * * * *` | Janitor |
| `vyforge.payment.retention.pii` | `730d` | Sau đó payload của cổng được che PII |
| `vyforge.payment.reconciliation.cron` / `lookback` | `0 30 6 * * *` / `3d` | Đối soát tự động (`-` để tắt) |
| `vyforge.payment.platform-merchants.*` | — | Merchant mặc định của platform (secret từ env, `notify-key` ≥ 32 ký tự) |
| `vyforge.payment.providers.<id>.enabled` / `base-url` / `api-url` | `true` / theo env | Ghi đè URL cổng (test, proxy) |

## 6. Tích hợp với module khác

| Module | Tích hợp |
|---|---|
| billing | `BillingPaymentBridgeAutoConfiguration` (ở billing): thu tiền hoá đơn qua intent, nghe `PaymentSucceeded`/`RefundSucceeded` |
| master-data | Trường merchant `lookup:bank` |
| einvoice | Nghe `PaymentSucceeded`/`RefundSucceeded` để phát hành hoá đơn (theo thiết kế; nối qua billing — cần xác nhận nối trực tiếp) |
| tenant-lifecycle | Bridge thanh toán QR/link khi kích hoạt tenant (0.16.0) |
| automation | Ví dụ `@AutomationEvent("payment.succeeded")` trong tài liệu automation; event của payment hiện **chưa** đánh annotation |
| notification, webhooks, realtime | Theo thiết kế nghe event qua outbox; chưa thấy adapter trong code (cần xác nhận) |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded`; có thể bật trong `platform-services` (`deployment-mode: services`), app khác gọi REST
  `/api/v1/payments/**`. Không có client Java `remote`.
- **IAM**: `payment:intent:Create|Get|List|Pay|Cancel|Review|RecordManual|Match`, `payment:refund:Create|Get|List|RecordManual`,
  `payment:mandate:Create|List|Revoke`, `payment:schedule:Create|List|Update`,
  `payment:merchant:Create|Get|List|Update`, `payment:event:List|GetRaw|Reprocess`,
  `payment:reconciliation:Run|List|Resolve`, `payment:ledger:List|Post`, `payment:settings:Update`.
- **Sự kiện**: `PaymentIntentCreated`, `PaymentAttemptStarted`, `PaymentAttemptFailed`, `PaymentStatusChanged`,
  `PaymentSucceeded`, `PaymentRequiresReview`, `PaymentCanceled`, `PaymentExpired`, `RefundSucceeded`, `RefundFailed`,
  `MandateActivated`, `MandateRevoked`, `PaymentScheduleDue`, `PaymentSchedulePastDue`, `ReconciliationCompleted`.
- **Giấy phép**: module `licensed` (`payment`, từ 0.13.0).

## 8. Trạng thái & còn mở

- Đã có: intent/attempt, 7 adapter cổng, merchant theo tenant, hoàn tiền, mandate + lịch thu, sổ cái, janitor, đối soát
  API/CSV, xoay khoá (`payment-rekey`), package web.
- Chờ đối tác: chạy thử từng cổng trên sandbox/production thật với hợp đồng merchant của khách (cần xác nhận cổng nào đã
  kiểm thật).
- Pháp lý: Thông tư 22/2026/TT-NHNN chưa đọc được toàn văn — pháp chế phải đọc trước pilot; "ví trả trước" chỉ có mẫu
  hạch toán, không API, cần pháp chế xác nhận trước khi bật.
- Chưa có: zone admin riêng trong platform-admin; định tuyến thông minh theo tỷ lệ thành công; chống gian lận chuyên sâu
  (chỉ có hook `beforeAttempt`). Không nhận số thẻ (PCI DSS: chỉ redirect/hosted page).
- Không có README module trong repo platform-java; tài liệu này dựa trên code và thiết kế.
