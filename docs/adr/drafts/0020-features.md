<!-- Mục "features" cho ADR-0020 (module P1). Phiên điều phối gộp vào ADR-0020 chung. -->

### features: `platform-starter-features` và `@vyforge/features`

- **Trạng thái:** Chấp nhận — 26/10/2026. Thiết kế: [docs/design/features.md](../../design/features.md).
- **Bối cảnh:** App cần bật tính năng theo tenant, người dùng, phần trăm, lịch, và tắt nhanh khi sự cố (feature flag); đồng
  thời cần bán theo gói: quyền theo gói, giới hạn, quota theo kỳ, đo mức dùng, trial, grace (entitlement). MODULE_PLAN mục
  5 cấm tự host flag server (Unleash: AGPL, thêm service).
- **Quyết định:**
  1. Một starter, hai nửa: flag (API công khai là OpenFeature, `dev.openfeature:sdk` 1.23.0) và entitlement (tắt được).
  2. Engine đánh giá flag là `flagd-core` 3.0.0 nhúng trong app (Apache-2.0). Định nghĩa trong PostgreSQL được biên dịch
     sang định dạng flagd; cùng snapshot xuất được cho flagd daemon (service ngoài JVM) và OFREP (frontend). Không tự viết
     JsonLogic hay thuật toán hash; không bắt buộc daemon.
  3. Thứ tự đánh giá: break-glass → tắt → prerequisite → override theo tenant → rule → rollout mặc định. Không lưu
     override theo từng người dùng.
  4. Đánh giá trong bộ nhớ trên snapshot bất biến; đồng bộ giữa replica bằng Valkey pub/sub, dự phòng bằng poll version
     trong DB (mất Valkey chỉ chậm, không mất thay đổi).
  5. Entitlement: feature `BOOLEAN | LIMIT | METERED | CONFIG`; gói có version và kế thừa; add-on, override theo tenant,
     grant (priority, hạn dùng, rollover). Thứ tự giải: override → gói + add-on → gói theo trạng thái → mặc định của
     code. Quota trừ nguyên tử trên PostgreSQL trong transaction của người gọi, có idempotency key; Valkey chỉ tăng tốc
     cho feature chọn `FAST`. Kỳ tính theo `Asia/Ho_Chi_Minh`.
  6. Subscription có state machine `TRIALING, ACTIVE, PAST_DUE, SUSPENDED, CANCELED, EXPIRED`; mốc thời gian là task
     trên `DurableTaskQueue` của `-scheduling`. Không phụ thuộc `-payment`: payment gọi `SubscriptionService`, nghe
     `UsagePeriodClosed`.
  7. Duyệt 4 mắt là SPI `ChangeApprovalPolicy`, mặc định tắt; cài đặt có sẵn `FourEyesApprovalPolicy` (người duyệt khác
     người tạo, có audit, có hết hạn).
  8. Policy dựng sẵn `vyforge:FeaturesOperator` (`features:*`) và `vyforge:FeaturesPlanSales`
     (`subscription:Change`, `grant:Create` + quyền đọc), chỉ ở tenant vận hành, áp qua `Authz.evaluate(..., additional)`
     nên `Deny` của IAM vẫn thắng.
  9. Dùng khối chung: `StarterSchemaMigrator` (lịch sử `flyway_schema_history_features`), `DurableTaskQueue`,
     `SecretCipher` (chưa cần ở 0.13).
- **Hệ quả:** Không có service mới. Flag rác được kiểm bằng loại flag + hạn dự kiến + báo cáo stale. Bảng
  `ent_usage_record` cần partition tháng và retention. `flagd-core` còn mới (3.0.0): bọc sau provider của Vyforge, có test
  tương thích với flagd daemon. Pilot dùng module phải ghi tên ở đây trước khi bật trên production (nguyên tắc 5 của
  MODULE_PLAN).
