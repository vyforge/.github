# platform-starter-settings — cài đặt phân tầng platform → tenant → người dùng

## 1. Mục đích

Mỗi ứng dụng nghiệp vụ đều cần chỗ lưu cấu hình mà người vận hành, tenant admin hay người dùng tự sửa được: múi giờ,
ngôn ngữ, logo, giờ yên lặng, mật độ bảng, view đã lưu. Settings cho giá trị **có kiểu**, **khai báo bằng code**, ghi đè
theo tầng platform → tenant → (đơn vị) → người dùng, đọc nhiều và rẻ, sửa qua API có kiểm quyền, audit, lịch sử, khôi
phục, hẹn giờ và gói xuất/nhập. Không thay feature flag (`features`), secret hạ tầng hay cấu hình triển khai.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| API khai báo và đọc | `platform-core` (`com.vyforge.platform.core.settings`) | Module nào cũng khai báo/đọc được; không có starter thì `DefaultsOnlySettings` |
| Starter | `platform-starter-settings` | Từ 0.14.0; không thu phí (`licensed: false`) |
| Package web | `@vyforge/settings` 0.2.0 | Client server-only, provider + hook cho trình duyệt, component quản trị, adapter view đã lưu |
| Zone admin | `apps/settings-admin` 0.1.0 (`/m/settings`) | |
| Service | `platform-services` | Bật module `settings`; app khác dùng chế độ `remote` |

Thiết kế: [docs/design/settings.md](../design/settings.md).

## 3. Tính năng chính

- **Khai báo**: `SettingDefinition.of(key, SettingTypes.X)` với default, tầng cho phép, `Exposure`, nhóm, nhãn vi/en, ràng
  buộc (`Constraints.atMost()`, `narrowOnly()`), JSON Schema; đăng ký qua bean `SettingDefinitionProvider`.
- **Đọc**: `settings.get(def)` (tenant + người dùng hiện tại), `forUser(tenantId, sub)`, `forTenant(tenantId).find(…)`,
  `settings.secret(def).reveal()` (giá trị nhạy cảm, giải mã lúc dùng), `withUnits(...)`.
- Thứ tự đọc: mặc định → `application.yml` → PLATFORM → TENANT → ORG_UNIT → USER. Giá trị **khoá** chặn tầng dưới; ràng buộc
  so với tầng cha được kiểm khi ghi và kẹp khi đọc. Chiến lược gộp `OVERRIDE`, `COLLECT`, `DEEP_MERGE` (RFC 7396).
- `CoreSettings` dùng chung: `core.locale`, `core.timezone`, `core.currency`, `core.date-format`, `core.number-format`,
  `core.week-start`, `core.fiscal-year-start`, `branding.*`, `ui.*`.
- Đường đọc không ném lỗi vì hạ tầng: dùng snapshot cũ (tối đa `cache.stale-max`), rồi `application.yml` và mặc định.
- **Ghi** (`SettingsAdmin`): `set`, `setAll` (một transaction), hẹn giờ (`effectiveAt`), `restore(revisionId)`, áp gói
  preset (`applyBundle`), khoá lạc quan (`expectedVersion`). Mỗi lần ghi: kiểm kiểu, schema, tầng, khoá cha, ràng buộc;
  chạy `SettingChangeInterceptor` (có thể veto); lưu giá trị + revision + audit + event trong một transaction; xoá cache mọi
  replica sau commit.
- **Duyệt 4 mắt** (từ 0.16.0): khoá đánh dấu `approval=required` hoặc khớp `approval.keys` → API trả `202` kèm
  `approvalId`; người khác duyệt/từ chối; hết hạn sau 72 giờ.
- Tầng USER không audit, không phát event (mặc định), tối đa 60 lần ghi/phút/người.
- REST `/api/v1/settings`: `/me/effective` (ETag), `/me/definitions`, `/me/values/{key}`; `/public?tenant=`;
  `/admin/info|groups|values|explain|overrides|revisions|schedules|bundles|presets|health|approvals`;
  `/internal/snapshot|catalogs/{service}|values`.
- UI tự sinh từ JSON Schema: bản người vận hành, tenant admin, trang "Tuỳ chọn của tôi".

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-settings</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/settings
```

Cần PostgreSQL, `platform-starter-authz`, `platform-starter-audit` (thiếu thì app vẫn chạy với `DefaultsOnlySettings`).
Có setting nhạy cảm thì phải cấu hình `vyforge.crypto.keys.*`.

```java
public static final SettingDefinition<TimeRange> QUIET_HOURS = SettingDefinition
        .of("notification.quiet-hours", SettingTypes.TIME_RANGE)
        .defaultValue(TimeRange.parse("22:00-07:00"))
        .scopes(SettingScope.PLATFORM, SettingScope.TENANT, SettingScope.USER)
        .label("Giờ yên lặng", "Quiet hours")
        .build();

@Bean
SettingDefinitionProvider notificationSettings() {
    return r -> r.group("notification.delivery", LocalizedLabel.of("Gửi thông báo", "Delivery"), 200)
            .add(QUIET_HOURS);
}

ZoneId zone = settings.get(CoreSettings.TIMEZONE);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.settings.mode` | `embedded` | `remote`: đọc/ghi qua `platform-services` |
| `vyforge.settings.remote.url` | — | URL server (client registration `vyforge-settings`, scope `settings.internal`) |
| `vyforge.settings.defaults.<key>` | — | Giá trị mặc định ở tầng `application.yml` |
| `vyforge.settings.cache.near-ttl` | `5m` | Không có Valkey thì replica khác thấy giá trị mới sau tối đa chừng này |
| `vyforge.settings.cache.stale-max` | (cần xác nhận) | Dùng snapshot cũ tối đa bao lâu khi DB/server lỗi |
| `vyforge.settings.admin.operator-tenant` | `default` | Tầng PLATFORM chỉ ghi từ tenant này |
| `vyforge.settings.approval.enabled` | `true` | `false` tắt hẳn duyệt 4 mắt |
| `vyforge.settings.approval.keys` | — | Mẫu khoá cần duyệt (`*` ở cuối) |
| `vyforge.settings.approval.expires-after` | `72h` | Yêu cầu duyệt hết hạn |
| `vyforge.settings.events.user-scope` | `false` | Phát event cho tầng USER |
| `vyforge.settings.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/settings.sql` |
| `vyforge.security.public-paths` | — | Thêm `/api/v1/settings/public` nếu cần trang công khai (logo, tên) |

## 6. Tích hợp với module khác

- **platform-starter-jpa**: `SettingsLocaleChain` (`LocaleChain` `@Primary`: `ui.locale` → `core.locale` → vi, en);
  `SettingsRetentionPolicyResolver` cho thùng rác (`jpa.trash.retention.<type>`, trần 365 ngày).
- **admin-kit**: view đã lưu ở khoá `admin-kit.views.<module>.<resource>` (`COLLECT`, TENANT + USER, tối đa 50 view/tầng).
- **organization**: tầng ORG_UNIT theo đơn vị chính của người dùng và các đơn vị cha.
- **privacy-dsr**: `UserSettingsContributor` xuất/xoá tuỳ chọn cá nhân theo `sub`.
- **files**: bean `FileRefUrlResolver` để `/public` trả URL logo.
- Module khác (notification, export, sequence, calendar, privacy-dsr…) đọc `CoreSettings` thay vì tự giữ múi giờ, ngôn ngữ.
- Valkey (kênh `vyforge:settings:changed`), `spring-boot-starter-cache` (lớp cache thứ hai) đều tuỳ chọn.
- SPI: `SettingDefinitionProvider`, `SettingChangeInterceptor`, preset bundle.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote** (`vyforge.settings.mode`): app remote đọc snapshot qua server (cache cục bộ), gửi định
  nghĩa của mình lên server lúc khởi động, job ghi qua `/internal/values`. App remote không đọc được giá trị nhạy cảm và
  interceptor của app không chạy ở server.
- **Quyền IAM** (`SettingsActions`): `settings:value:Get|Update|Lock`, `settings:secret:Update`,
  `settings:revision:List|Restore`, `settings:bundle:Export|Apply` (áp gói cần MFA), `settings:override:List`,
  `settings:approval:Decide`. Định nghĩa có thể đổi action theo tầng (`WriteActions`).
- **Sự kiện**: `SettingChanged` (không mang giá trị).
- Health `settings`; metric `vyforge.settings.*`; job `settings-schedule` (mỗi phút), `settings-maintenance` (mỗi giờ).
  README có sẵn gợi ý PrometheusRule.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm duyệt 4 mắt (`settings V2`), tầng `ORG_UNIT`, contributor privacy-dsr, retention
  thùng rác jpa, chuỗi locale.
- **Còn chờ xác nhận** (thiết kế mục 10.3): đặt API settings trong `platform-core` hay artifact riêng (đã làm theo
  `platform-core`); có bắt buộc các module chuyển sang `core.*` ngay không; số ADR.
- **Giới hạn đã biết**: không có Valkey thì replica khác thấy giá trị mới chậm tới `near-ttl`; app remote không đọc được
  secret.
