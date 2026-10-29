# platform-starter-calendar — lịch làm việc, âm lịch, nhắc việc

## 1. Mục đích

Nhiều nghiệp vụ tính hạn theo ngày làm việc: SLA một cửa, hạn trả lời yêu cầu dữ liệu cá nhân (DSR), SLA của workflow,
nhắc việc trước hạn. Calendar trả lời theo lịch của đúng tenant, chi nhánh, ca: *lúc này có phải giờ làm việc không*,
*hạn N ngày (giờ) làm việc tính từ lúc này là khi nào*, *giữa hai mốc có bao nhiêu thời gian làm việc*. Kèm âm lịch Việt
Nam (UTC+7), nhắc việc, sự kiện theo ngày làm việc, sổ lịch với RRULE và iCalendar.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-calendar` | Từ 0.14.0; có phí (`licensed: true`) |
| Package web | `@vyforge/calendar` 0.1.0 | Ngày làm việc, tính hạn, âm lịch, sổ lịch, component quản trị |
| Zone admin | `apps/calendar-admin` 0.1.0 (`/m/calendar`) | |
| Công cụ | `tools/build_seed_packs.py` | Dựng pack lịch nghỉ seed |
| Service | `platform-services` | Bật module `calendar`; app khác dùng `remote` |

Thiết kế: [docs/design/calendar.md](../design/calendar.md).

## 3. Tính năng chính

- **Ba lớp lịch**: ngoại lệ của tenant (`CLOSED`/`OPEN`/`HOURS`, lặp dương hoặc âm lịch) > lịch quốc gia (`VN`, `VN-GOV`)
  với phương án nghỉ tenant chọn > mẫu tuần (chu kỳ 1–4 tuần, ca qua nửa đêm). Lịch con kế thừa lịch cha (sâu ≤ 3). Ngày
  làm bù làm theo giờ ngày được hoán đổi.
- **Tính toán** (`BusinessCalendar`): `addBusinessDays(at, n, DayCountRule, CalcOptions)` (quy tắc `LEGAL`, giờ chốt
  `cutoff(...).endingAt(...)`), `add(at, BusinessDuration.parse("4bh"))`, `businessDurationBetween`, `isWorkingTime`. Lõi
  thuần, O(log n), cache Caffeine.
- **Năm chưa công bố**: dự đoán theo âm lịch và BLLĐ Điều 112 (`provisional = true`); `CONSERVATIVE` chỉ trừ cuối tuần và
  ngày lễ dương lịch cố định.
- Mọi kết quả mang `CalendarVersion` (vd. `default@r12+VN-GOV@2026.10.09`); module dùng tự lưu và tính lại khi nhận
  `BusinessCalendarChanged`.
- **Lịch theo chủ thể**: `calendars.forSubject(SubjectRef.user(sub))` (gán trực tiếp → `CalendarResolverStep` → mặc định).
- **Nhắc việc** (`Reminders`): tương đối theo giờ làm việc (vd. `-4bh` trước hạn), tự tính lại khi lịch đổi.
- **Lịch chạy** (`CalendarSchedules`): vd. ngày làm việc đầu tiên của tháng lúc 08:30 → `ScheduleFired`. Tick đầu/cuối
  ngày làm việc.
- **Âm lịch** `VietnameseLunarCalendar.vietnam()`: thuật toán Meeus, UTC+8 tới 07/08/1967 và UTC+7 sau đó (Tết khác lịch
  Trung Quốc ở 1968, 1985, 2007, 2030…), phạm vi 1900–2199.
- **Sổ lịch (agenda)**: sự kiện với RRULE, lặp theo ngày làm việc, lặp theo âm lịch (giỗ, rằm; xử lý tháng thiếu, tháng
  nhuận), nhập/xuất iCalendar, feed `.ics` có token. Tắt được bằng `agenda.enabled=false`.
- **Dữ liệu lịch nghỉ**: pack `VN`, `VN-GOV` 2024–2027 đóng sẵn (mỗi ngày có căn cứ pháp lý), pack người vận hành nhập
  (bắt buộc chữ ký Ed25519 trên JCS).
- REST `/api/v1/calendar`: `calendars/**` (`days`, `business-hours`, `:compute`, `holidays.ics`), `national/{code}/days`,
  `lunar/convert`, `recurrence:preview`, `agendas/**`, `occurrences`, `events/**`, `feeds/**` (`/feeds/{token}.ics` không
  cần đăng nhập); quản trị `/admin/**`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-calendar</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/calendar
```

Cần `DataSource`, `platform-starter-scheduling` (queue `calendar`). Tenant chưa có lịch thì tự tạo lịch `default`
(T2–T6, `VN`).

```java
BusinessCalendar cal = calendars.defaultCalendar();

// DSR: 2 ngày làm việc, không tính ngày nhận, hết 24:00 ngày thứ 2
Deadline due = cal.addBusinessDays(receivedAt, 2, DayCountRule.LEGAL, CalcOptions.CONSERVATIVE);
store(due.at(), due.version().asString(), due.provisional());

// nhắc trước hạn 4 giờ làm việc
reminders.schedule(ReminderRequest.relative("vrn:collaboration:todo/77", "todo.due-soon", "todo77:due-soon",
        dueAt, BusinessDuration.parse("-4bh")).owner(assigneeSub).channel("event"));
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.calendar.mode` | `embedded` | `remote`: tính cục bộ trên snapshot của `platform-services` |
| `vyforge.calendar.remote.url`, `registration-id` | — | Server và client registration (scope `calendar.internal`) |
| `vyforge.calendar.remote.snapshot-poll` / `stale-after` | `60s` / `24h` | Làm mới snapshot; quá hạn thì health `DEGRADED` |
| `vyforge.calendar.defaults.time-zone` | `Asia/Ho_Chi_Minh` | |
| `vyforge.calendar.defaults.national-code` | `VN` | Lịch quốc gia mặc định |
| `vyforge.calendar.defaults.pattern` | `MON-FRI 08:00-12:00,13:00-17:00` | Mẫu tuần mặc định |
| `vyforge.calendar.defaults.option-defaults` | `FOLLOW_GOV` | Phương án nghỉ mặc định (`FOLLOW_GOV`, `FIRST`, `NONE`) |
| `vyforge.calendar.defaults.past-change-policy` | `REQUIRE_REASON` | Sửa lịch quá khứ |
| `vyforge.calendar.national.trusted-keys.<kid>` | — | Khoá kiểm pack lịch nghỉ |
| `vyforge.calendar.national.expect-next-year-by` | `11-15` | Sau ngày này năm sau chưa chính thức thì health `WARN` |
| `vyforge.calendar.max-horizon-years` | `5` | |
| `vyforge.calendar.reminders.max-attempts` / `retention` | `8` / `90d` | |
| `vyforge.calendar.ticks.enabled` / `cron` | `true` / mỗi phút | Tick ngày làm việc |
| `vyforge.calendar.agenda.enabled` | `true` | Sổ lịch |
| `vyforge.calendar.integration.<phần>.enabled` | `true` | Tắt từng tích hợp |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: master-data, organization, workflow, notification.

- **master-data**: `MasterDataNationalHolidaySource` (order 50) đọc `holiday-day:<mã>`, thắng pack; calendar cài
  `HolidayDraftGenerator` để master-data sinh nháp năm mới; nghe `MasterDataPublished`. Chỉ ở chế độ nhúng.
- **organization**: `OrgUnitStep` (order 200) tìm lịch theo đơn vị của người dùng; chạy cả ở remote.
- **notification**: kênh nhắc `notification:<type>` → `Notifications.send` (type mặc định `calendar.reminder`).
- **workflow**: SLA dùng `add(..)` với ngày làm việc; `CalendarImpactProbe` đếm việc bị ảnh hưởng (cần xác nhận phạm vi
  đã làm).
- **privacy-dsr**: hạn SLA theo lịch nghỉ của tenant (`CalendarWorkingDays`).
- SPI: `NationalHolidaySource`, `CalendarResolverStep`, `ReminderChannel`, `CalendarFeedContributor`,
  `CalendarChangeInterceptor`, `CalendarImpactProbe`.
- Valkey (kênh `vyforge:calendar:changed`): xoá cache mọi replica ngay; không có thì đúng sau `cache.definition-ttl` (60 s).

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote**: app remote tính cục bộ trên snapshot (kết quả và `CalendarVersion` trùng server);
  `Reminders`, `CalendarSchedules` gọi `/internal/**` của server; `Agendas`, `CalendarAdmin` không có ở app.
  `CalendarContractTests` chạy cùng kịch bản ở cả hai chế độ.
- **Quyền IAM**: `calendar:calendar:List|Get|Create|Update|Archive|EditPast`, `calendar:exception:Edit`,
  `calendar:assignment:Edit`, `calendar:option:Choose`, `calendar:national:Read|Import`, `calendar:reminder:List|Retry|Cancel`,
  `calendar:schedule:List`, `calendar:agenda:List|Create|Update|Delete`, `calendar:event:Read|Create|Update|Delete`,
  `calendar:feed:Create|Revoke`, `calendar:settings:Get|Update`.
- **Sự kiện** (`TenantAwareEvent`, package `…calendar.event`): `BusinessCalendarChanged`, `ReminderDue`, `ReminderFailed`,
  `BusinessDayStarted`, `BusinessDayEnded`, `BusinessPeriodStarted`, `ScheduleFired`, `TickGapDetected`,
  `CalendarEventChanged`, `NationalOptionPending`, `NationalDataStale`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm tích hợp, lặp theo âm lịch, đồng bộ Valkey, chế độ remote.
- **Còn lại** (README): adapter privacy-dsr (`PersonalDataContributor`; phần xoá đã có ở store) và interceptor mẫu của
  forms; master-data chưa có thuộc tính phương án nghỉ (`optionGroup`…, `OPTIONAL_DAY_OFF`) nên ngày đọc từ master-data
  luôn là ngày nghỉ chắc chắn; ở remote chưa có tick/`ScheduleFired` qua Kafka và `CalendarFeedContributor` của app.
- **Chờ pháp chế** (thiết kế 10.3 Q5): cách hiểu "N ngày làm việc" cho DSR và giờ chốt nhận hồ sơ một cửa. Q6: ai ký pack
  seed (cùng khoá với master-data).
- **Đã biết**: `ReminderIntegrationTests` từng lỗi do độ chính xác nano/micro giây trên Linux (forge-108 đang sửa, cần xác
  nhận đã merge).
