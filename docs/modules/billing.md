# Billing — thu phí theo gói (subscription, hoá đơn bán hàng)

## 1. Mục đích

Cho app SaaS (hoặc app bán dịch vụ định kỳ) lớp thương mại: catalog gói/giá có version bất biến, subscription (trial,
nâng/hạ gói có proration, số ghế, add-on, giá riêng, tạm dừng, huỷ), đo mức dùng, hoá đơn bán hàng số liên tục bất biến
sau khi chốt, credit note, thuế GTGT Việt Nam, coupon, nhắc nợ. Quyền dùng thuộc `features`, thu tiền thuộc `payment`,
hoá đơn điện tử thuộc `einvoice` — billing nối với chúng bằng cầu nối tuỳ chọn và vẫn chạy khi thiếu. Thiết kế:
`docs/design/billing.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-billing` | platform-java 0.16.1; bảng `bill_*` |
| Package web | `@vyforge/billing` 0.1.0 | subpath `/ui`, `/admin` (không có README package — cần xác nhận danh sách component) |
| Zone admin | `apps/billing-admin` (`/m/billing`) | |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling` (queue `billing`), `-authz`, `-audit`, `SecretCipher` | |

## 3. Tính năng chính

- **Catalog**: product (mã thuế, thuộc tính thuế), metric, plan + version (`draft` → `:publish` bất biến → `:retire`;
  đổi giá = version mới + `:migrate`); gắn `entitlementPlan` của features.
- **Customer**: tổ chức/cá nhân, MST, địa chỉ, số dư (`:adjust`), discount; ẩn danh hoá (`billing:customer:Anonymize`).
- **Subscription**: `:change-plan` (nâng giữa kỳ = credit note phần chưa dùng + hoá đơn proration; hạ gói/giảm ghế cuối
  kỳ), `:change-quantity`, `:set-addon`, `:override-price`, `:pause`/`:resume`, `:cancel`, `:extend-trial`,
  `:preview`, `/upcoming-invoice`; trạng thái `incomplete` tới khi trả kỳ đầu.
- **Mức dùng**: `usage.ingest(UsageEvent.of(...))` khử trùng theo `transactionId`, lô ≤ 1.000; metric nguồn features
  chờ `featuresWait` để đóng kỳ.
- **Hoá đơn**: phí trả sau/mức dùng kỳ cũ + phí trả trước kỳ mới; số liên tục không lỗ gán lúc chốt
  (`INV-{yyyy}-{seq:6}`), `:finalize`, ghi nhận thanh toán tay, `:void`, `:mark-uncollectible`, `:retry-collection`,
  `/pdf`. Không ghi dòng âm (HĐĐT VN phải điều chỉnh hoá đơn gốc) — dùng credit note (`CN-{yyyy}-{seq:6}`).
- **Thuế** (`VietnamVatCalculator`): `VAT10/8/5/0` tra `bill_tax_rate` theo ngày lập, `KCT`, `KKKNT`, `KHAC:<x>`; làm
  tròn theo nhóm thuế suất; giảm 10% → 8% (NQ 204/2025, tới 31/12/2026) cho sản phẩm `vatReductionEligible`; tỷ giá
  (`/exchange-rates`).
- **Dunning** `standard-vn`: bước 0, 3, 7, 10 ngày (hạn chế → `suspended`), 30 ngày (huỷ); giờ yên lặng 22:00–07:00.
- **Coupon**, promotion code (`/promotion-codes:check`).
- **Cổng khách** `/me/*` (`billing:self:Get|Update|Pay`).
- **Việc nền** (queue `billing`, khoá idempotent): `subscription.cycle`, `subscription.milestone`, `invoice.finalize`,
  `invoice.collect`, `invoice.overdue`, `invoice.render`, `invoice.einvoice`, `invoice.payment`, `dunning.step`,
  `entitlement.sync`; job `billing-cycle-sweep` (lưới an toàn).
- **REST** (`/api/v1/billing`): `/products`, `/metrics`, `/plans`, `/customers`, `/subscriptions`, `/usage/events`,
  `/usage`, `/invoices`, `/credit-notes`, `/coupons`, `/dunning/policies|runs`, `/settings`, `/tax-rates`,
  `/exchange-rates`, `/me/*`, `GET /admin/info`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-billing</artifactId>
</dependency>
```

```java
// SaaS bán gói Pro 499.000 đ/tháng
catalog.upsertProduct(UpsertProduct.of("pro-fee", "Phí thuê bao gói Pro", "recurring")
        .withTax("VAT10", Map.of("vatReductionEligible", true)));
catalog.createPlan(CreatePlan.of("pro", "Gói Pro").withEntitlementPlan("pro"));
catalog.draftVersion("pro", PlanVersionSpec.monthly(new BigDecimal("499000")).withFeeProduct("pro-fee"));
catalog.publish("pro", 1, null);

CustomerView c = customers.upsert(UpsertCustomer.organization("school-1", "Trường Hoa Sen", "0101234567")
        .withAddress("12 Láng Hạ, Hà Nội"));
SubscriptionResult r = subscriptions.create(CreateSubscription.of(c.id(), "pro").withMandate(mandateId));
```

Cấu hình tối thiểu: `vyforge.crypto.*` (khoá `SecretCipher`). DBA tự chạy schema thì
`vyforge.billing.schema.migrate=false` + `META-INF/vyforge/sql/billing.sql`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.billing.enabled` / `api-enabled` | `true` / `true` | |
| `vyforge.billing.mode` | `embedded` | `services` khi chạy trong platform-services (báo qua `admin/info`) |
| `vyforge.billing.zone` | `Asia/Ho_Chi_Minh` | Múi giờ khách và kỳ |
| `vyforge.billing.defaults.number-format` / `credit-note-number-format` / `sequence-reset` | `INV-{yyyy}-{seq:6}` / `CN-{yyyy}-{seq:6}` / `yearly` | |
| `vyforge.billing.defaults.default-tax-code` / `tax-behavior` / `tax-rounding` | `VAT10` / `exclusive` / `per_group` | |
| `vyforge.billing.defaults.grace-period` / `payment-terms-days` | `0s` / 0 | |
| `vyforge.billing.defaults.proration-unit` | `day` | |
| `vyforge.billing.defaults.incomplete-expiry` / `trial-ending-notice` | `23h` / `3d` | |
| `vyforge.billing.defaults.dunning-policy` | `standard-vn` | |
| `vyforge.billing.defaults.features-wait` / `max-event-age` | `6h` / `35d` | Mức dùng |
| `vyforge.billing.defaults.einvoice-partial-payment` | `per_payment` | Xem mục 8 (mâu thuẫn với README) |
| `vyforge.billing.bridges.<payment|features|einvoice|report|files>` | bật khi có module | `bridges.report=false` để tắt |
| `vyforge.billing.schema.migrate` | `true` | |

Phần lớn mặc định trên là mặc định của `BillingSettings` — tenant ghi đè qua `/settings`.

## 6. Tích hợp với module khác

| Module | Có module | Không có |
|---|---|---|
| payment | `PaymentCollector`: intent `Reference("billing_invoice", id)`, trừ bằng mandate; nghe `PaymentSucceeded`/`PaymentAttemptFailed`/`RefundSucceeded` | Chỉ ghi nhận tay (`POST /invoices/{id}/payments`) |
| features | `EntitlementSync` (trial, assign, renew, past_due, suspend, activate, change_plan, cancel, addon); `UsagePeriodClosed` → usage | App tự nghe event subscription |
| einvoice | `EInvoiceSink`: hoá đơn → HĐĐT, credit note → HĐĐT điều chỉnh/thay thế; thời điểm lập theo NĐ 254 Đ.9 (`EInvoiceTrigger`) | `einvoice_status = not_required` |
| platform-report | `InvoiceRenderer`: template `billing.invoice` (mặc định tiếng Việt, `html-pdf`, cần `platform-report-openhtmltopdf`); tenant sửa qua export | Bản thể hiện HTML |
| files | PDF lưu ở bucket `billing-documents` | Render khi cần |
| sequence | App khai bean `InvoiceNumbering` gọi `DocumentNumbers.next(key, GAPLESS)` | Bảng `bill_sequence` |
| tenant-lifecycle | Bridge checkout và hoá đơn đầu khi tạo tenant (0.16.0) | |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded`; chạy được trong platform-services (`mode: services`). Chưa có client `remote`.
- **IAM**: `billing:catalog:List|Get|Update`, `billing:plan:List|Get|Create|Update|Publish|Migrate`,
  `billing:customer:List|Get|GetPersonal|Create|Update|Anonymize`, `billing:balance:Get|Adjust`,
  `billing:subscription:List|Get|Create|Update|OverridePrice`, `billing:usage:Get|Ingest`,
  `billing:invoice:List|Get|Create|Finalize|Collect|RecordPayment|Void|MarkUncollectible`,
  `billing:credit-note:List|Create|Void`, `billing:coupon:List|Create|Apply`, `billing:dunning:List|Update`,
  `billing:settings:Get|Update`, `billing:self:Get|Update|Pay`.
- **Sự kiện**: `SubscriptionCreated`, `SubscriptionActivated`, `SubscriptionRenewed`, `SubscriptionPlanChanged`,
  `SubscriptionQuantityChanged`, `SubscriptionScheduledChangeApplied`, `SubscriptionTrialEnding`, `SubscriptionPastDue`,
  `SubscriptionSuspended`, `SubscriptionPaused`, `SubscriptionResumed`, `SubscriptionCanceled`, `InvoiceCreated`,
  `InvoiceFinalized`, `InvoicePaid`, `InvoicePartiallyPaid`, `InvoicePaymentFailed`, `InvoicePaymentReminder`,
  `InvoiceOverdue`, `InvoiceVoided`, `InvoiceMarkedUncollectible`, `InvoiceStatusChanged`, `CreditNoteIssued`,
  `CreditNoteVoided`, `CustomerBalanceChanged`.
- **Giấy phép**: module `licensed` (`billing`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: catalog version, subscription đầy đủ vòng đời, proration, mức dùng, hoá đơn + credit note, thuế GTGT VN, dunning,
  coupon, cổng khách, 5 cầu nối, zone admin.
- README (0.13) ghi chưa làm: HĐĐT theo từng lần trả một phần — 0.13 lập một HĐĐT khi trả đủ; trong khi mặc định code
  `einvoice-partial-payment = per_payment` (cần xác nhận hành vi hiện tại). Hoá đơn gộp theo khách (`per_customer`),
  customer hierarchy, ngưỡng mức dùng giữa kỳ; chế độ `remote`.
- Pháp lý (`docs/legal-open-questions.md`): SaaS có thuộc "dịch vụ phần mềm" không chịu GTGT (KCT) không — không gán sẵn,
  tenant chọn theo kế toán; số định danh cá nhân trên hồ sơ khách là dữ liệu nhạy cảm hay cơ bản; hoá đơn ngoại tệ có bắt
  buộc in tổng quy đổi VND không.
