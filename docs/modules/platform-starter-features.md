# platform-starter-features — feature flag và entitlement

## 1. Mục đích

Một module, hai nửa dùng chung hạ tầng. **Flag** trả lời "tính năng này có bật cho request này không" (phát hành dần, kill
switch, thử nghiệm) theo chuẩn OpenFeature. **Entitlement** trả lời "tenant này đã mua quyền dùng chưa, còn bao nhiêu"
(gói dịch vụ, giới hạn, quota theo kỳ, đo mức dùng, trial, grace). Ứng dụng nghiệp vụ không phải tự viết bảng gói/hạn mức
hay tự đánh giá flag.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-features` | Từ 0.13.0; có phí (`licensed: true`, key `features`). Bộ đánh giá `flagd-core` nhúng trong app |
| Package web | `@vyforge/features` 0.1.0 | Client server-only (OFREP), gate, hook, component quản trị `@vyforge/features/admin` |
| Zone admin | `/m/features` | Chưa có app `features-admin` trong `platform-admin` trên `origin/main`; component quản trị có sẵn trong package (cần xác nhận lịch làm zone) |
| Service | `platform-services` | Bật module `features`; app khác dùng chế độ `remote` |

Thiết kế: [docs/design/features.md](../design/features.md) (đã duyệt 26/10/2026).

## 3. Tính năng chính

**Flag**
- Khai báo bằng code: `FlagKey.bool("checkout.new-ui", false)` + bean `FlagCatalog` (kind, owner, `expiresAt`,
  `clientVisible`). Flag chưa có trong DB được tạo với default của code.
- Đọc: `featureFlags.isEnabled(key)`, hoặc gọi thẳng `OpenFeatureAPI` (cùng provider, cùng ngữ cảnh). `@RequiresFlag("…")`
  → tắt thì `404 feature.not_available`.
- Ngữ cảnh tự dựng: `targetingKey` = `sub`, `tenantId`, `user.*`, `tenant.plan|subscriptionStatus|trial`, thuộc tính từ
  `TenantAttributesProvider`, `locale`, `app.version` (`X-App-Version`), bean `EvaluationContextContributor`.
- Thứ tự: break-glass `vyforge.features.overrides.<flag>` → flag tắt → prerequisite → override theo tenant → rule → rollout
  (`fractional` của flagd, trọng số phần vạn). Toán tử rule: so sánh, `in`, `semver_*`, thời gian, `segment`, `jsonLogic`.
- Revision, rollback, lịch thay đổi, báo cáo flag cũ (`/reports/stale-flags`), simulate.
- OFREP 0.4 cho frontend (`POST /api/v1/features/ofrep/v1/evaluate/flags`): chỉ flag `client_visible`, có `ETag`/304.

**Entitlement**
- Khai báo bằng `FeatureCatalog`: `metered` (reset theo kỳ, 00:00 giờ VN), `limit`, `bool`, `config`.
- `entitlements.require(…)` (403 `entitlement.not_entitled`), `checkLimit(…)` (403 `entitlement.limit_reached`),
  `consume(feature, n, idempotencyKey)` (403 `entitlement.quota_exceeded`), annotation `@RequiresEntitlement`.
- `consume` nguyên tử trong transaction của người gọi; cùng idempotency key chỉ tính một lần. Enforcement `HARD`, `SOFT`
  (+ overage), `NONE`; ngưỡng cảnh báo (80, 100 %).
- Grant (khuyến mãi, mua thêm), add-on, override theo tenant, gói có gói cha, version gói nháp → phát hành (bất biến).
- Đếm nhanh trên Valkey cho feature tần suất cao (`usage.fast.features`), đẩy về PostgreSQL định kỳ.
- Không đọc được entitlement quá `stale-ttl` → 503 `entitlement.unavailable` (không mở quyền).
- **Subscription**: `startTrial`, `activate` (idempotent theo tham chiếu thanh toán), `changePlan`, `cancel`. Vòng đời
  `TRIALING → ACTIVE | EXPIRED`, `ACTIVE → PAST_DUE → SUSPENDED → ACTIVE`.

**Quản trị**: duyệt 4 mắt tuỳ chọn (`FourEyesApprovalPolicy`, trả 202 kèm change request), policy dựng sẵn cho người vận
hành và nhân viên kinh doanh.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-features</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/features
```

Cần `DataSource`, `platform-starter-authz`, `platform-starter-audit`, `platform-starter-scheduling` (thiếu thì module không
bật, app vẫn chạy). Valkey tuỳ chọn để thay đổi tới mọi replica ngay.

```java
public final class CheckoutFlags {
    public static final FlagKey<Boolean> NEW_UI = FlagKey.bool("checkout.new-ui", false);
}

@Bean
FeatureCatalog billingFeatures() {
    return FeatureCatalog.builder()
            .metered("einvoice.issued", m -> m.unit("invoice"))
            .limit("users", l -> l.unit("user"))
            .build();
}

@RequiresEntitlement(feature = "einvoice.issued", consume = 1, idempotencyKey = "#request.number()")
@Transactional
public Invoice issue(IssueRequest request) {
    if (featureFlags.isEnabled(CheckoutFlags.NEW_UI)) { … }
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.features.mode` | `embedded` | `remote`: gọi module ở `platform-services` |
| `vyforge.features.remote.url`, `.registration-id` | — | URL server, client registration (`client_credentials`, scope `features.internal`) |
| `vyforge.features.sync.poll-interval` | `15s` | Poll thay đổi khi không có Valkey |
| `vyforge.features.sync.max-staleness` | `10m` | Quá ngưỡng thì health `featuresSnapshot` = `OUT_OF_SERVICE` |
| `vyforge.features.sync.flagd-export` | `false` | Xuất `GET /flagd/flags.json` cho flagd daemon |
| `vyforge.features.zone` | `Asia/Ho_Chi_Minh` | Múi giờ cho `time_between`, reset kỳ |
| `vyforge.features.overrides.<flag>` | — | Break-glass: ép variant |
| `vyforge.features.admin.operator-tenant` | `default` | Tenant duy nhất được gọi API quản trị |
| `vyforge.features.entitlements.thresholds` | `80, 100` | Ngưỡng phát `QuotaThresholdReached` |
| `vyforge.features.usage.fast.features` | — | Feature đếm trên Valkey (vd. `ai.tokens`) |
| `vyforge.features.usage.record-retention-months` | `13` | Giữ `ent_usage_record` |
| `vyforge.features.usage.record-actor` | `true` | Lưu `sub` vào bản ghi mức dùng |
| `vyforge.features.approval.enabled` | `false` | Duyệt 4 mắt |
| `vyforge.features.internal-api.scopes` | `features.internal`, `services.internal` | Scope được gọi `/internal/**` |
| `vyforge.features.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/features.sql` |

## 6. Tích hợp với module khác

- **licensing**: `LicenseEntitlementCeiling` (một `EntitlementCustomizer`) kẹp hạn mức về trần giấy phép, tắt
  `module.<m>`/`admin.<m>` của module ngoài giấy phép.
- **platform-services**: `ModuleEntitlementInterceptor` chặn API module tenant chưa mua (`404 module.not_enabled`).
- **payment / billing**: features không phụ thuộc payment; app hoặc billing gọi `SubscriptionService` khi nhận IPN và nghe
  `UsagePeriodClosed` để tính overage.
- **tenant-lifecycle**: bridge chọn gói, trial, tạm ngừng.
- **api-access, master-data**: dùng entitlement (`api.max-plan`, `api.keys.max`, giới hạn danh mục).
- SPI: `FlagCatalog`, `FeatureCatalog`, `TenantAttributesProvider`, `EvaluationContextContributor`,
  `EntitlementCustomizer`, `ChangeApprovalPolicy`; hook OpenFeature dạng bean được gắn tự động.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded**: đầy đủ (migration, API quản trị, job).
- **remote**: app giữ nguyên `FeatureFlags`, `Entitlements`, annotation, OFREP. Flag đánh giá tại chỗ trên snapshot
  (`/internal/snapshot`); entitlement đọc qua `/internal/tenants/{id}/entitlements` (có cache); mức dùng ghi đồng bộ
  `POST /internal/tenants/{id}/usage`. Không có migration, API quản trị hay việc nền ở app. Gateway chặn `/internal/**`.
- **Quyền IAM**: catalog `features:*`. Policy dựng sẵn ở tenant vận hành: role `FEATURES_ADMIN` →
  `vyforge:FeaturesOperator` (`features:*`); role `PLAN_SALES` → `vyforge:FeaturesPlanSales` (`subscription:Get/Change`,
  `grant:Create`, đọc gói, danh mục, mức dùng).
- **Sự kiện** (outbox Spring Modulith, package `…features.event`): `FlagChanged`, `TenantFlagOverrideChanged`,
  `FlagScheduledChangeFailed`, `ChangeRequestChanged`, `SubscriptionChanged`, `TrialEnding`, `QuotaThresholdReached`,
  `QuotaExceeded`, `UsagePeriodClosed`.
- Metric `vyforge.features.*`; job `features-usage-partitions`, `features-cleanup`, `features-reconcile`,
  `features-usage-fast-flush`.

## 8. Trạng thái & còn mở

- **Đã làm**: flag + entitlement đầy đủ, subscription, duyệt 4 mắt, chế độ remote, OFREP (0.13.0–0.16.x). Sửa lỗi 0.15.0:
  subscription không hết trial trên Linux (độ chính xác `Instant`).
- **Còn mở**: zone admin `/m/features` chưa có app riêng trong `platform-admin` (cần xác nhận). Thiết kế không còn câu hỏi
  chặn.
- **Giới hạn đã biết**: đếm nhanh trên Valkey có thể thiếu tối đa một chu kỳ flush khi Valkey mất dữ liệu, không dùng cho
  số liệu có hậu quả pháp lý. Ngữ cảnh đánh giá không được lưu.
