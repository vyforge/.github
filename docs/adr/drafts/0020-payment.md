# ADR-0020, mục `payment`: thanh toán nội địa và quốc tế

> Bản nháp của mục `payment` trong ADR-0020 (các module P1). Phiên điều phối ghép vào ADR-0020 chung.
> Thiết kế: [docs/design/payment.md](../../design/payment.md).

## Bối cảnh

Dự án thương mại ở VN phải tích hợp VNPay, MoMo, ZaloPay hoặc chuyển khoản VietQR. Mỗi dự án tự làm lại thì hay mắc cùng
một nhóm lỗi: tin Return URL, không đối chiếu số tiền, xử lý IPN hai lần, mất IPN mà không truy vấn lại, để lộ secret của
merchant, không đối soát được.

## Quyết định

1. **Starter `platform-starter-payment`.** Starter nhúng vào app, chạy chung transaction với nghiệp vụ. Nó cũng chạy
   được trong `platform-services` (D7 của platform-services) mà không đổi API.
2. **Mô hình Stripe/Hyperswitch:** `PaymentIntent` (thu bao nhiêu, cho việc gì) tách khỏi `PaymentAttempt` (thu qua cổng
   nào). Cả hai có state machine khai báo trong code. Hoàn tiền là trục riêng (`refundState`), không phải trạng thái của
   intent.
3. **Bằng chứng mới đổi trạng thái.**
   - Chỉ IPN/webhook đã kiểm chữ ký, hoặc kết quả truy vấn do server gọi, mới đổi trạng thái tiền.
   - Return URL chỉ kích hoạt truy vấn rồi chuyển hướng.
   - Lệch số tiền thì không ghi nhận và intent cần xem xét.
   - Tiền về sau khi huỷ/hết hạn, hoặc trả trùng, thì không bỏ qua: intent sang `requires_review`.
4. **Tenant ở đường vào không có token** lấy từ `notify_key` trong URL (256 bit ngẫu nhiên) của merchant account. Việc
   này chỉ xảy ra **sau khi** kiểm chữ ký bằng secret của chính merchant đó và kiểm mã merchant trong payload. Đây là
   ngoại lệ có chủ đích của ADR-0015 ("tenant chỉ từ claim"), giới hạn ở các truy vấn `@TenantExempt` của đường IPN,
   Return URL và trang trạng thái công khai (có client secret).
5. **Adapter sau SPI `PaymentProvider`**, khai báo capability. Lõi không so tên cổng. Bản 0.13 có VNPay, MoMo, ZaloPay,
   payOS, SePay, Stripe và `manual`. Adapter gọi REST bằng `RestClient` của platform (không dùng SDK). Lệnh ghi không
   tự gửi lại: lỗi thì thành `unknown`, rồi janitor truy vấn.
6. **Khử trùng ba lớp:**
   - `Idempotency-Key` khi tạo intent và hoàn tiền;
   - `payment_provider_event` unique theo (merchant, khoá của cổng);
   - khoá idempotent của bút toán sổ cái.
   Mọi thay đổi tiền khoá dòng intent rồi attempt theo thứ tự cố định.
7. **Sổ cái bút toán kép trong PostgreSQL.**
   - Constraint trigger deferred từ chối giao dịch lệch nợ/có lúc COMMIT; trigger khác chặn UPDATE/DELETE.
   - Tài khoản nóng không giữ số dư (số dư = snapshot + bút toán), nên không tranh khoá.
   - Ví trả trước và công nợ chỉ có mẫu hạch toán, không có API (chờ pháp chế).
8. **Khối dùng chung của platform:**
   - `SecretCipher` (credentials, token mandate, client secret);
   - `StarterSchemaMigrator` (`flyway_schema_history_payment`);
   - `DurableTaskQueue` queue `payment` (janitor, đối soát, lịch thu, tự hoàn);
   - `@ClusterScheduled` cho các job quét.
9. **Tuân thủ:**
   - Chỉ redirect, QR hoặc deeplink, không nhận số thẻ: đủ điều kiện SAQ A của PCI DSS v4.0.1. Có luật ArchUnit cấm field
     dữ liệu thẻ.
   - Payload gốc của cổng giữ 10 năm làm chứng từ (Luật Kế toán, NĐ 174/2016), PII được che sau 2 năm (Luật 91/2025).
   - App là đơn vị chấp nhận thanh toán, không cung ứng trung gian thanh toán (TT 40/2024, sửa bởi TT 41/2025 và
     TT 22/2026; cần pháp chế đọc toàn văn TT 22/2026).
10. **Không phụ thuộc module P1 nào.** Điểm nối là event qua outbox: realtime, notification, webhooks, einvoice. Quản trị
    là zone `/m/payment` theo Multi-Zones (platform-services D1–D10); component ở `@vyforge/payment/admin`; có
    `GET /api/v1/payments/admin/info`.

## Hệ quả

- Thêm cổng mới chỉ cần thêm adapter và test vector, không phải sửa lõi hay migration.
- Chi tiết giao thức của MoMo và ZaloPay lấy từ mẫu code chính thức và đoạn trích tài liệu, vì trang tài liệu chặn truy cập
  tự động. Phải đối chiếu toàn văn trước khi lên production; test vector đã có để kiểm nhanh.
- Chưa có chế độ `remote` (app gọi payment ở `platform-services` qua interface Java). Khi chạy ở server, app nhận kết quả
  qua topic `payment-events`.
