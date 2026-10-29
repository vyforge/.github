# platform-starter-licensing — giấy phép module cho bản cài on-prem

## 1. Mục đích

Bản cài on-prem và air-gapped cần một "trần theo hợp đồng" cho cả bản triển khai: được chạy module có phí nào, bao nhiêu
tenant, bao nhiêu người dùng, tới khi nào, được nâng cấp lên bản nào. Licensing kiểm một file giấy phép ký số **hoàn toàn
offline** (không phone-home, không khoá phần cứng, không tắt dịch vụ từ xa). Nó không thay `features` (gói, entitlement
theo tenant) hay `billing` (giá, hoá đơn): hai module đó làm việc bên dưới trần của giấy phép.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Phần kiểm (Java) | `platform-core` (`com.vyforge.platform.core.license`, `core.signing`) | Chạy trong **mọi** tiến trình Java, không cần DB hay mạng |
| Starter quản trị | `platform-starter-licensing` | Từ 0.13.0; module không thu phí (`licensed: false`). Nhúng ở **một** nơi có trang quản trị, mặc định `platform-services` |
| Package web | `@vyforge/licensing` 0.1.0 | Kiểm file giấy phép offline (Ed25519 + JCS) cho shell quản trị, client và component quản trị |
| Zone admin | `apps/licensing-admin` 0.1.0 (`/m/licensing`) | |
| Công cụ phía Vyforge | `tools/license-cli` (`vyforge-license`) | Không phát hành ra Maven; sổ giấy phép `license-ledger` |
| Test | `platform-test`: `@WithLicense`, `LicenseTestSupport`, `DevLicenseModeForTests` | |

Thiết kế: [docs/design/licensing.md](../design/licensing.md).

## 3. Tính năng chính

- **File giấy phép `vyforge-license/1`**: JSON đọc được, ký Ed25519 trên JCS (RFC 8785), kèm chứng chỉ khoá phát hành do
  root nhúng sẵn ký. Đọc lại định kỳ (`reload-interval`): gia hạn không cần khởi động lại.
- **Starter có phí cần giấy phép** (từ 0.16.0, L6): auto-config phần embedded gắn `@ConditionalOnLicensedModule("<key>")`
  và khai báo `META-INF/vyforge/modules/<key>.json` (`key`, `title`, `licensed`, `limits`, `caps`, `apiPrefix`). Module
  không có trong giấy phép thì không tạo bean nào (không migration, job, API); app cần bean đó thì `LicenseFailureAnalyzer`
  in rõ lý do. Client `remote` không gắn điều kiện này.
- **Kiểm lúc chạy** bằng `LicenseGuard`: `requireWithinLimit("tenants", n)` (403 `license.limit_reached`),
  `actionActive("...")`. Số hiện có của giới hạn lấy từ bean `LicenseUsageProvider`.
- **Hết hạn không bao giờ tắt dịch vụ**: chỉ áp các hành động có tên `banner`, `block-upgrade`, `block-new-tenants`,
  `block-create-over-limit`, `restrict-admin-writes`.
- `LicensedModuleFilter`: module bị rút khỏi giấy phép lúc chạy → API module trả `404 license.module_not_licensed`;
  `restrict-admin-writes` → ghi dưới `<apiPrefix>/admin/**` trả `403 license.admin_restricted`.
- `LicenseEntitlementCeiling` (khi có features): chạy cuối chuỗi `EntitlementCustomizer`, tắt `module.<m>`/`admin.<m>` của
  module ngoài giấy phép, kẹp hạn mức về trần `modules[].caps`.
- **Mức dùng**: chụp hằng ngày (`@ClusterScheduled`), báo cáo sử dụng xuất được, lịch sử cài đặt.
- Kiểm URL bản triển khai (`vyforge.license.deployment-url`): lệch với `deployments[].urls` thì cảnh báo
  `DEPLOYMENT_URL_MISMATCH` (không chặn).
- `LicensingNotice`: sự kiện ghi audit, phát đúng một lần cho cả bản triển khai (kích hoạt, từ chối, đổi trạng thái, chạm
  giới hạn, sắp hết hạn).
- REST `/api/v1/licensing/admin/**`: `info`, `license` (GET, PUT với `If-Match: "<serial>"`), `license/validate`,
  `modules`, `usage`, `usage/report`, `deployment-request`, `history`, `ceiling`.
- Công cụ `vyforge-license`: `keys init-root | generate | certify-issuing`, `issue`, `renew`, `reissue`, `verify`,
  `inspect`, `check`, `catalog`; ký bằng OpenBao Transit hoặc file khoá (chỉ dev).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-licensing</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/licensing
```

Cần `DataSource`, `platform-starter-authz`, `platform-starter-audit`, `platform-starter-scheduling`. Mọi dịch vụ dùng
starter có phí (kể cả không cài starter licensing) cần giấy phép ở staging/production:

```yaml
# Secret vyforge-license mount thành thư mục (không subPath)
vyforge.license:
  path: /etc/vyforge/license/license.lic   # hoặc VYFORGE_LICENSE_TEXT
  deployment-id: ${VYFORGE_DEPLOYMENT_ID}
  deployment-url: ${VYFORGE_DEPLOYMENT_URL}
```

Đánh dấu một starter có phí và kiểm giới hạn:

```java
@AutoConfiguration
@ConditionalOnLicensedModule("payment")
public class PaymentEmbeddedAutoConfiguration { … }

@Bean
LicenseUsageProvider merchants(MerchantRepository repo) {
    return LicenseUsageProvider.of("payment.merchants", repo::count);
}

licenseGuard.requireWithinLimit("tenants", tenants.countActive() + 1);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.license.text` (`VYFORGE_LICENSE_TEXT`) | — | Nội dung giấy phép, ưu tiên hơn file |
| `vyforge.license.path` | `/etc/vyforge/license/license.lic` | File giấy phép |
| `vyforge.license.deployment-id` (`VYFORGE_DEPLOYMENT_ID`) | — | Phải có trong `deployments[]` của giấy phép |
| `vyforge.license.deployment-url` (`VYFORGE_DEPLOYMENT_URL`) | — | Lệch thì chỉ cảnh báo |
| `vyforge.license.mode` | `auto` | `auto`: không có file + profile `dev`/`local`/`test` → `DEV`, còn lại `UNLICENSED`; `enforce`; `dev` |
| `vyforge.license.reload-interval` | `60s` | Chu kỳ đọc lại file |
| `vyforge.license.dev.max-tenants` / `max-users` | `5` / `50` | Giới hạn chế độ dev |
| `vyforge.licensing.operator-tenant` | `default` | Chỉ token của tenant này dùng được API |
| `vyforge.licensing.authz.operator-role` | `platform-admin` | Role nhận policy dựng sẵn `licensing:*` |
| `vyforge.licensing.install.writable` | `auto` | Secret chỉ đọc thì PUT trả `501 license.store_read_only` kèm lệnh cập nhật |
| `vyforge.licensing.install.require-mfa` | `true` | Cài giấy phép cần `amr` có `mfa` |
| `vyforge.licensing.usage.collect-cron` | `0 30 1 * * *` | Chụp mức dùng hằng ngày |
| `vyforge.licensing.usage.retention` | `P3Y` | Giữ snapshot mức dùng |
| `vyforge.licensing.events.externalize` | `false` | `true`: `LicensingNotice` ra topic `vyforge.licensing` |
| `vyforge.licensing.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/licensing.sql` |

## 6. Tích hợp với module khác

- **Mọi starter có phí**: `@ConditionalOnLicensedModule` + manifest `META-INF/vyforge/modules/<key>.json`. Danh sách
  `licensed: true` (0.16.0): ai, api-access, automation, billing, calendar, cms-content, collaboration, custom-fields,
  einvoice, ekyc, esign, export, features, files, forms, master-data, notification, organization, payment, privacy
  (privacy-dsr), realtime, search, sequence, tenant-lifecycle, webhooks, workflow. Không cần: licensing, ops-console,
  settings, webhooks-inbound và các starter nền.
- **features** (tuỳ chọn): `LicenseEntitlementCeiling` kẹp entitlement về trần giấy phép.
- **tenant-lifecycle**: giới hạn `tenants`, hành động `block-new-tenants`.
- **notification**: ví dụ `licenseGuard.actionActive("notification.block-marketing")`.
- **Shell `platform-admin`**: `@vyforge/licensing` kiểm giấy phép offline để ẩn menu module ngoài giấy phép.
- Metric `vyforge.license.*` (platform-core), health `license` (luôn `UP`, không thuộc readiness/liveness).

## 7. Chế độ triển khai, quyền, sự kiện

- **Triển khai**: phần kiểm luôn embedded trong mọi tiến trình Java. Starter quản trị nhúng ở một nơi (mặc định
  `platform-services`). Không có chế độ `remote`.
- **Bảng** cấp bản triển khai (không có `tenant_id`): `license_installation`, `license_usage_snapshot`, `license_state`.
- **Quyền IAM**: `licensing:license:Get`, `licensing:license:Install`, `licensing:usage:Get`, `licensing:usage:Export`,
  `licensing:history:List`. API chỉ nhận token của tenant vận hành.
- **Sự kiện**: `LicensingNotice` (audit; ra Kafka topic `vyforge.licensing` khi bật `events.externalize`).

## 8. Trạng thái & còn mở

- **Đã làm**: kiểm offline trong platform-core (0.13.0), starter quản trị, admin zone, CLI phát hành; cưỡng chế starter có
  phí (0.16.0), kiểm URL bản triển khai, `LicensingNotice`.
- **Chờ buổi tạo khoá**: khoá root thật chưa có trong `META-INF/vyforge/trust/license/roots.json`. Tới lúc đó bản build
  không tin giấy phép nào ngoài test.
- **Chờ chủ sở hữu/pháp chế** (thiết kế mục 10.3): Q1 hành vi mặc định khi hết hạn (đang dùng phương án A), Q3 đếm người
  dùng theo `users` hay `active_users`, Q5 danh sách module có phí (quyết định kinh doanh), Q6 điều khoản EULA/hợp đồng
  mẫu (Luật SHTT Điều 28, 198), Q7 người giữ mảnh khoá root (Shamir 3/5) và OpenBao giữ khoá phát hành, Q8 cổng nâng cấp
  theo dòng hay theo bản. Trạng thái duyệt của từng câu (cần xác nhận).
- **Giới hạn đã biết**: Secret chỉ đọc thì không cài qua API được (trả lệnh cập nhật để chạy tay). Cảnh báo URL chỉ so
  host.
