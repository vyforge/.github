# platform-starter-tenant-lifecycle — vòng đời tenant

## 1. Mục đích

SaaS nhiều tenant cần một "control plane" cho tenant: khách tự đăng ký, hệ thống khởi tạo tenant qua mọi module (danh
tính, gói, dữ liệu nền, quyền chủ tenant), tạm ngừng khi quá hạn, và offboarding có xuất dữ liệu, thời hạn lưu giữ, legal
hold, xoá ở mọi module kèm biên bản xoá. auth-server vẫn giữ danh tính tenant (ADR-0011); module này chỉ giữ vòng đời
(bảng `tl_*`).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-tenant-lifecycle` | Từ 0.14.0 (phát hành ở 0.15.0); có phí (`licensed: true`) |
| Package web | `@vyforge/tenant-lifecycle` 0.1.0 | Client đăng ký tổ chức, tiến độ khởi tạo, trang "Tổ chức của tôi", component quản trị |
| Zone admin | `apps/tenant-lifecycle-admin` 0.1.0 (`/m/tenant-lifecycle`) | |
| Service | `platform-services` (control plane) | App khác tham gia ở chế độ `participant` |
| Phụ thuộc ngoài | `platform-auth-server` | `/api/v1/admin/tenants`, scope `auth.tenant.provision` |

Thiết kế: [docs/design/tenant-lifecycle.md](../design/tenant-lifecycle.md).

## 3. Tính năng chính

- **Đăng ký tự phục vụ** (`SignupService`): giữ slug, OTP số điện thoại, đồng ý ToS + DPA, chấm rủi ro, duyệt
  `auto | risk-based | manual`, trả trước (`PAY_FIRST`), hết hạn đơn. Captcha qua port `HumanVerification`.
- **Quản lý tenant** (`TenantLifecycle`): tạo hộ, nhập tenant có sẵn, tạm ngừng `READ_ONLY`/`LOCKED`, mở lại, hồ sơ, dòng
  thời gian.
- **Saga khởi tạo/xoá** (`RunService`): run `PROVISION`, `SUSPEND`, `RESUME`, `EXPORT`, `ERASE`; thử lại, bỏ qua, huỷ,
  quyết định bảng không ai nhận. Mỗi bước là một task `DurableTaskQueue` (queue `tenant-lifecycle`), retry có backoff,
  việc dài trả checkpoint.
- **Participant** (`TenantProvisioner`): module/app khai `ParticipantDescriptor` (phase `IDENTITY` → `COMMERCIAL` →
  `FOUNDATION` → `MODULES` → `APPLICATION` → `ACCESS`, `dependsOn`, `tables`), cài `provision`/`deprovision` idempotent;
  `JdbcTenantPurger` xoá bảng. Bước `inventory` đầu run ERASE chặn bảng có `tenant_id` mà không ai khai.
- **Offboarding** (`OffboardingService`): yêu cầu đóng (cooling-off), huỷ, legal hold, gia hạn, xoá ngay (duyệt 4 mắt),
  link tải gói xuất ZIP, **biên bản xoá** JSON JCS + SHA-256 (PDF khi có files).
- **Template tenant** có version (`TemplateCatalog`, `META-INF/vyforge/tenant-templates/*.yml`).
- REST `/api/v1/tenant-lifecycle`: `/public/**` (không cần token), `/signups/**` (chủ đơn), `/me/tenant/**` (tenant admin,
  MFA khi đóng), `/admin/**` (người vận hành), `/internal/participants/{service}`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-tenant-lifecycle</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/tenant-lifecycle
```

Cần PostgreSQL, `platform-starter-authz`, `-audit`, `-scheduling`.

```yaml
vyforge.tenant-lifecycle:
  hmac-key: ${TL_HMAC_KEY}              # base64 ≥ 32 byte; trống thì chỉ chạy được một replica
  auth-server.url: http://platform-auth-server
  plans:
    free: { policy: FREE }
    pro: { policy: PAY_FIRST }
    trial: { policy: TRIAL, trial: 14d }
spring.security.oauth2.client.registration.tenant-lifecycle-auth:
  provider: vyforge
  client-id: platform-services
  client-secret: ${PLATFORM_SERVICES_SECRET}
  authorization-grant-type: client_credentials
  scope: auth.tenant.provision
```

```java
@Component
class CrmTenantProvisioner implements TenantProvisioner {
    public ParticipantDescriptor descriptor() {
        return ParticipantDescriptor.builder("app.crm").phase(Phase.APPLICATION)
                .capabilities(PROVISION, EXPORT, DEPROVISION).tables("crm_lead", "crm_note").build();
    }
    public ProvisionResult provision(ProvisionContext ctx) { ... idempotent theo ctx.runId() ... }
    public ErasureOutcome deprovision(LifecycleContext ctx, DeprovisionPlan plan) {
        return purger.purge(ctx, plan, List.of("crm_note", "crm_lead"));
    }
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.tenant-lifecycle.mode` | `embedded` | Control plane đầy đủ; `participant`: app ở xa chỉ tham gia saga (`participant.service`, `base-url`, `control-plane-url`) |
| `vyforge.tenant-lifecycle.operator-tenant` | `default` | Tenant vận hành |
| `vyforge.tenant-lifecycle.directory` | `auth-server` | `in-memory` cho bản một tenant, dev, test |
| `vyforge.tenant-lifecycle.hmac-key` | — | OTP, băm IP, link biên bản |
| `vyforge.tenant-lifecycle.signup.approval` | `risk-based` | `auto`, `risk-based`, `manual` |
| `vyforge.tenant-lifecycle.signup.otp.required` | `false` | Bật OTP (cần notification) |
| `vyforge.tenant-lifecycle.signup.purge-after` | `30d` | Xoá đơn không thành khách |
| `vyforge.tenant-lifecycle.provisioning.max-attempts` / `step-timeout` | `8` / `2m` | |
| `vyforge.tenant-lifecycle.offboarding.cooling-off` | `14d` | Thời gian hối hận |
| `vyforge.tenant-lifecycle.offboarding.default-retention` | `90d` | Giữ dữ liệu trước khi xoá |
| `vyforge.tenant-lifecycle.offboarding.tombstone` | `180d` | Slug đã xoá không đăng ký lại được |
| `vyforge.tenant-lifecycle.offboarding.erase-now-requires-approval` | `true` | Xoá ngay cần hai người |
| `vyforge.tenant-lifecycle.offboarding.retention-policy.status` | `PENDING_LEGAL` | `APPROVED` (kèm `version`, `reference`) mới tự xoá khi hết hạn |
| `vyforge.tenant-lifecycle.authz.operator-role` | `platform-admin` | Role nhận `tenant-lifecycle:*` |

## 6. Tích hợp với module khác

Tuỳ chọn (`@ConditionalOnBean`): features, notification, files, billing, payment, privacy-dsr.

- **features**: `FeaturesParticipant` gắn gói/trial, `SUSPENDED` → `suspended-plan`, xoá `ff_*`/`ent_*`; `auto-offboard-after`.
- **notification**: OTP (ZNS → SMS), thông báo `tenant.welcome`, `tenant.signup.*`, `tenant.provisioning.failed`,
  `tenant.offboarding.*`.
- **files**: gói xuất ZIP và PDF biên bản ở bucket `tenant-lifecycle`.
- **billing**: `BillingCheckout` tạo customer `tenant:<slug>`, subscription, hoá đơn đầu; `InvoicePaid` → `markPaid`.
- **payment** (cùng billing): QR/link thanh toán hoá đơn đầu.
- **privacy-dsr**: `PrivacyDsrTenantData` gửi `TENANT_EXPORT`/`TENANT_ERASE` (duyệt 4 mắt ở privacy-dsr).
- **licensing** (platform-core, luôn có): giới hạn `tenants`, `block-new-tenants`.
- **ratelimit**: rule cho slug (30/phút/IP), signup (10/giờ), OTP (10/15 phút, fail-closed).
- **auth-server**: tạo/khoá/xoá tenant, thêm chủ tenant với `invite: true` (gửi email mời).
- SPI: `TenantProvisioner`, `TenantDataOperations`, `HumanVerification`, `TenantDirectoryPort`, `CheckoutPort`, `LicenseLimit`.

## 7. Chế độ triển khai, quyền, sự kiện

- **Control plane** (thường ở `platform-services`) + **participant** ở app khác (endpoint
  `/internal/tenant-provisioner/v1/{key}/{provision|suspend|resume|export|deprovision}`, tự khai với control plane lúc
  khởi động).
- **Quyền IAM**: catalog `TenantLifecycleActions` (`tenant-lifecycle:<resource>:<Verb>`). Role `platform-admin` ở tenant vận
  hành nhận `tenant-lifecycle:*`; `TENANT_ADMIN` nhận `tenant-lifecycle:self:*`.
- **Sự kiện** (không PII): `SignupSubmitted`, `SignupAwaitingApproval`, `SignupExpired`, `SignupRejected`,
  `TenantRequested`, `TenantProvisioned`, `TenantProvisioningFailed`, `TenantRunProgress`, `TenantSuspended`,
  `TenantResumed`, `TenantOffboardingRequested`, `TenantOffboardingCancelled`, `TenantDataExported`, `TenantErased`,
  `TenantErasureAwaitingConfirmation`.
- Metric `vyforge.tenant_lifecycle.*`; health `tenantLifecycleDirectory`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.15.0 bản đầu (signup, saga, offboarding, participant protocol); 0.16.0 bridge thật sang billing, payment,
  privacy-dsr, giấy phép; rate limit; retention cần duyệt trước khi tự xoá.
- **Chờ pháp chế/chủ sở hữu (Q3)**: các con số `cooling-off`, `default-retention`, `tombstone`, `certificate-retention`,
  `signup.purge-after`, có bật `auto-offboard-after` không. Trong lúc chờ, offboarding dừng ở `RETAINED` và người vận hành
  xoá tay (`POST /admin/tenants/{slug}/offboarding:erase-now`).
- **Chưa làm**: ký số biên bản (Q4, hiện là JSON + SHA-256 + audit), verified domain (T14), silo/region (T15), import khi
  sáp nhập (T16), run hàng loạt, adapter antibot (mới có port `HumanVerification`).
