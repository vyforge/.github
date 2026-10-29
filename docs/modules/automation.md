# Automation — luật tự động hoá "khi X thì Y"

## 1. Mục đích

Cho tenant admin tự cấu hình luật tự động hoá mà không cần dev: "khi đơn hàng chuyển DELIVERED và giá trị lớn thì gửi
thông báo cảm ơn", "30 ngày trước hạn hợp đồng thì tạo việc nhắc", "mùng 1 hằng tháng (ngày làm việc) thì xuất báo cáo".
Mô hình ECA chạy xong ngay: một trigger, một điều kiện CEL, tối đa 10 hành động tuần tự; chạy sau commit, chống vòng
lặp, quyền của luật bằng quyền người publish. Thiết kế: `docs/design/automation.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-automation` | platform-java 0.16.1 |
| Java API (nhẹ) | `com.vyforge.platform:platform-automation-api` | chỉ annotation `@AutomationEvent`, `@AutomationRecord`, `@AutomationField` và `AutomationTriggerCatalog`; module khác thêm dạng `<optional>` mà không kéo cả starter |
| Package web | `@vyforge/automation` 0.1.0 | client server-only, Server Action, `/react` (`RunRuleMenu`, `ConditionBuilder`, `CelEditor`), `/expression`, `/admin` |
| Zone admin | `apps/automation-admin` (`/m/automation`) | luật, trình dựng, chạy thử, lịch sử chạy, mẫu, cài đặt |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling` (queue `automation`), `platform-expression` | |

## 3. Tính năng chính

- **Trigger**: `EVENT` (event có `@AutomationEvent`), `RECORD` (entity `@AutomationRecord` hoặc
  `RecordChanges.publish`), `SCHEDULE` (cron 5 trường; `calendarSpec` ngày làm việc khi có calendar), `RECORD_DATE`
  (mốc tương đối theo trường ngày, vd `-30d`, `-2bd`, `anniversary`), `MANUAL` (nút "Chạy luật" trên trang bản ghi),
  `INBOUND` (webhook gọi vào có HMAC, tắt mặc định).
- **Điều kiện/tham số**: CEL qua `platform-expression` (kiểm kiểu lúc publish), `{"$expr": "…"}`, chuỗi `{{ … }}`; hàm
  `changed('field')`, `inGroup`, `hasConsent`, `businessDaysBetween` (khi có calendar). Biến `event`, `record`/`old`,
  `payload`, `input`, `actor`, `tenant`, `steps.<id>`.
- **Hành động có sẵn**: `automation.stop`, `automation.log`, `record.update`, `record.assign` (cần bean
  `RecordUpdater`), và khi có module: `notification.send`, `webhook.emit`, `export.start`, `todo.create`,
  `comment.post`, `workflow.start`. Không có hành động gọi URL tuỳ ý.
- **An toàn**: chạy sau commit qua `DurableTaskQueue`; chống vòng lặp 3 lớp (độ sâu chuỗi, một luật một lần trên một bản
  ghi trong chuỗi, tần suất) → tự `SUSPENDED`; trường `SENSITIVE` (`@PersonalData`) không lưu trong bản chụp.
- **Vận hành**: version bất biến, bản nháp, `simulate` không tác dụng phụ, log từng run, retry/cancel (cả hàng loạt),
  thống kê ngày (`aut_rule_stat`), mẫu luật của app (`META-INF/vyforge/automation/templates/*.yml`), nhập/xuất YAML,
  `backfill` cho `RECORD_DATE`.
- **SPI**: `AutomationAction` (hành động tuỳ biến, khai `requiredPermissions`, idempotent theo `idempotencyKey`),
  `AutomationInterceptor`, `AssigneeStrategy`, `RecordAdapter`, `AutomationTriggerCatalog`.
- **REST**: `/api/v1/automation/admin/**` (`info`, `catalog/*`, `expressions/check`, `rules` CRUD + `validate`,
  `simulate`, `publish`, `enable`, `disable`, `owner`, `debug`, `execute`, `versions`, `restore`, `stats`, `backfill`,
  `export`, `inbound`; `rules/import`, `templates`, `runs`, `settings`, `usage`, `tasks/**`, `audit`); người dùng cuối:
  `GET /api/v1/automation/rules/manual`, `POST /rules/{key}/execute`, `GET /runs/by-subject`; gọi vào:
  `POST /api/v1/automation/hooks/{token}` (header `X-Vyforge-Signature: t=…,v1=<hmac-sha256>`).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-automation</artifactId>
</dependency>
```

```java
@Entity
@AutomationRecord(type = "crm:order", label = "Đơn hàng", dateFields = {"dueDate"})
public class Order extends TenantEntity {
    @AutomationField(label = "Trạng thái", writable = true) @Enumerated(STRING) OrderStatus status;
    @AutomationField(label = "Người phụ trách", format = "user", writable = true) String ownerSub;
    @AutomationField(label = "Hạn") Instant dueDate;
}

@Bean
RecordUpdater<Order> orderUpdater(OrderService orders) {   // bật record.update / record.assign
    return RecordUpdater.of(Order.class, (id, changes, options) -> orders.applyAutomation(id, changes, options));
}
```

```yaml
# luật (tenant tạo trên admin, hoặc mẫu của app)
key: order-delivered-thanks
trigger: { type: RECORD, recordType: "crm:order", on: UPDATED, fields: [status] }
condition: "changed('status') && record.status == 'DELIVERED'"
actions:
  - id: thank
    type: notification.send
    params: { type: crm.order.thank-you, to: [{ kind: user, value: { $expr: "record.ownerSub" } }] }
```

Cấu hình tối thiểu: không cần (mặc định bật). DBA tự chạy schema thì `vyforge.automation.schema.migrate=false` và chạy
`META-INF/vyforge/sql/automation.sql`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.automation.mode` | `embedded` | Chỉ `embedded` đã cài |
| `vyforge.automation.schema.migrate` | `true` | Flyway riêng (`flyway_schema_history_automation`) |
| `vyforge.automation.limits.max-depth` / `max-depth-ceiling` | 5 / 10 | Độ sâu chuỗi luật |
| `vyforge.automation.limits.{tenant,rule,subject}-runs-per-minute` | 600 / 60 / 20 | Tần suất (chung cụm khi có ratelimit + Redis) |
| `vyforge.automation.limits.loop-suspend-threshold` / `failure-suspend-threshold` / `suspend-window` | 3 / 20 / 10m | Ngưỡng tự tạm dừng |
| `vyforge.automation.retention.{succeeded,failed,input,samples}` | 30d / 90d / 7d / 7d | Thời hạn giữ log run |
| `vyforge.automation.inbound.enabled` | `false` | Bật trigger `INBOUND` |
| `vyforge.automation.notifications.suspended` / `admin-group` | `true` / `admin` | Báo khi luật bị tạm dừng |
| `vyforge.automation.queue.max-attempts` | 5 | Retry của hành động |
| `vyforge.automation.api.base-packages` | gói của app | Gói quét annotation |
| `vyforge.automation.privacy.enabled` | `true` | Contributor DSR |

## 6. Tích hợp với module khác

Automation sở hữu port; module tương ứng có trên classpath thì adapter tự bật (app khai bean cùng kiểu để thay):

| Module | Adapter | Tác dụng |
|---|---|---|
| calendar | `CalendarAutomationScheduler`, `CalendarFunctions` | `SCHEDULE` theo ngày làm việc, offset `±bd/±bh`, `businessDaysBetween` |
| collaboration | `CollaborationAutomationTodos` | `todo.create`, `comment.post`, chiến lược `LEAST_OPEN` |
| workflow | `WorkflowAutomationWorkflows` | `workflow.start` (người yêu cầu = người publish) |
| notification, webhooks, export | hành động tương ứng | `notification.send`, `webhook.emit`, `export.start` |
| privacy, privacy-dsr | `AutomationPersonalData`, `DsrConsentChecker` | DSR, `hasConsent` |
| audit | `AuditLogTrail` | `audit_log` cho luật/cài đặt/run; `GET /admin/audit` |
| tenant-lifecycle | `LifecycleTenantStatus` | run của tenant không `ACTIVE` bị huỷ |
| ratelimit | `SharedRateGuard` | giới hạn tần suất chung giữa replica |
| authz | `PrincipalDirectory` (IAM) | bắt buộc để publish luật qua admin; kiểm quyền người publish |

Module khác (payment, billing…) chỉ cần `platform-automation-api` (optional) để đánh dấu event/entity làm trigger.

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded` (bảng `aut_*` trong DB của app, `record.update` gọi service của app). Thiết kế có `remote`
  chạy trong `platform-services` (event qua Kafka `remote-topics`, ghi bản ghi qua `/internal/automation/records` có
  ký) nhưng **chưa cài** trong code.
- **IAM**: `automation:rule:List|Get|Create|Update|Delete|Publish|Enable|Disable|Suspend|Simulate|Execute|TransferOwnership`,
  `automation:run:List|Get|Retry|Cancel`, `automation:template:List|Install`, `automation:settings:Get|Update`,
  `automation:stats:Get`, `automation:inbound:Manage`, `automation:task:List|Manage`, `automation:audit:List`. Nút chạy
  `MANUAL` cần quyền xem bản ghi (`<loại>:Get`).
- **Sự kiện** (`TenantAwareEvent`, Spring event sau commit; thiết kế có Kafka `automation-events`
  qua `vyforge.automation.events.externalize` nhưng chưa thấy trong code — cần xác nhận): `AutomationRulePublished`, `AutomationRuleStatusChanged`,
  `AutomationRuleSuspended`, `AutomationRunFailed`, `AutomationQuotaReached`. Event của automation không làm trigger cho
  luật khác. Thông báo `automation.rule.suspended` khi có notification.
- **Giấy phép**: module `licensed` (`automation`, từ 0.14.0) — cần license hợp lệ của platform (module licensing).

## 8. Trạng thái & còn mở

- Đã có: đủ 6 loại trigger, hành động có sẵn + adapter module, chống vòng lặp, simulate, version, mẫu luật, inbound
  HMAC, admin MFE, package web.
- Để bản sau (thiết kế §1.4): rẽ nhánh `if/else` trong luật (hiện mỗi hành động có `when`), `for each`, hành động AI,
  trigger theo truy vấn định kỳ, kho mẫu dùng chung giữa tenant.
- Chưa có chế độ `remote` / chạy trong `platform-services`.
