# Thiết kế: `platform-starter-calendar` và `@vyforge/calendar`

> Trạng thái: **thiết kế, chờ duyệt** (forge-55, 27/10/2026). Nhánh `docs/design-calendar`.
> Theo `module-conventions.md` (mục 1–6, gồm khối dùng chung của platform-java 0.12.1) và kiến trúc admin microfrontend
> đã chốt trong `docs/design/platform-services.md` (D1–D10, mục 4.11–4.12, 5.1–5.6). Phạm vi theo
> `docs/research/2026-10-gap-analysis.md` mục 2.7 và 6.7. Nối với các thiết kế đã merge: master-data (mục 3.9, lịch nghỉ
> quốc gia), privacy-dsr (port `WorkingDays`), workflow (port `BusinessCalendar`, đơn vị `bd`/`bh`), collaboration
> (`TodoDueCalculator`, `ReminderScheduler`), forms (`SubmissionInterceptor` kiểm ngày làm việc), cms-content (lịch xuất
> bản dùng component lịch).

Lịch làm việc là khối nền mà workflow, privacy-dsr, collaboration, helpdesk, booking, automation đều cần, nhưng không
module nào nên tự giữ. Module này trả lời ba câu hỏi theo **lịch của đúng tenant, đúng chi nhánh, đúng ca**: *lúc này có
phải giờ làm việc không*, *hạn N ngày (giờ) làm việc tính từ lúc này là khi nào*, *giữa hai mốc có bao nhiêu thời gian
làm việc*. Ngoài ra module lo phần **âm lịch Việt Nam** (UTC+7), **nhắc việc** theo thời gian thật hoặc theo giờ làm
việc, **sự kiện thời gian** theo múi giờ của tenant, **lịch lặp RRULE**, **xuất/nhập iCalendar**, và một **sổ lịch
(agenda)** gọn cho lịch công tác, lịch họp, lịch hiển thị chung từ nhiều module.

## Mục lục

- [0. Tóm tắt quyết định](#0-tóm-tắt-quyết-định)
- [1. Bài toán và phạm vi](#1-bài-toán-và-phạm-vi)
- [2. Nghiên cứu](#2-nghiên-cứu)
- [3. Kiến trúc](#3-kiến-trúc)
- [4. Mô hình dữ liệu](#4-mô-hình-dữ-liệu)
- [5. API](#5-api)
- [6. Mở rộng](#6-mở-rộng)
- [7. Vận hành](#7-vận-hành)
- [8. Bảo mật và tuân thủ](#8-bảo-mật-và-tuân-thủ)
- [9. Kế hoạch triển khai](#9-kế-hoạch-triển-khai)
- [9a. Đóng gói và triển khai độc lập](#9a-đóng-gói-và-triển-khai-độc-lập)
- [10. Lệch quy ước và quyết định](#10-lệch-quy-ước-và-quyết-định)
- [Phụ lục A. Ranh giới với các module khác](#phụ-lục-a-ranh-giới-với-các-module-khác)
- [Phụ lục B. Dữ liệu lịch nghỉ 2026–2027 đã kiểm](#phụ-lục-b-dữ-liệu-lịch-nghỉ-20262027-đã-kiểm)
- [Phụ lục C. Nguồn đã kiểm](#phụ-lục-c-nguồn-đã-kiểm)

---

## 0. Tóm tắt quyết định

| # | Quyết định | Lý do ngắn |
|---|---|---|
| C1 | **Lịch làm việc = ba lớp chồng nhau**, ưu tiên từ trên xuống: (1) *ngoại lệ* của tenant (nghỉ riêng, làm thêm, đổi giờ), (2) *lịch quốc gia* (ngày lễ, nghỉ bù, hoán đổi, làm bù) kèm lựa chọn phương án của tenant, (3) *mẫu tuần* (giờ làm theo thứ, chu kỳ nhiều tuần, có hiệu lực theo ngày). Một tenant có nhiều lịch (`default`, `cn-hcm`, `cskh-24x7`, `ca-dem`…), lịch con kế thừa lịch cha | Cách của Odoo (`resource.calendar` + `leave_ids` + `global_leave_ids`) và Jira SLA calendar (giờ theo ngày + ngày nghỉ), thêm lớp quốc gia có version mà hai hệ đó để người dùng nhập tay |
| C2 | **Dữ liệu lịch nghỉ quốc gia ở master-data, tính toán ở calendar** (xác nhận ranh giới master-data mục 3.9, Q1). Calendar khai báo SPI `NationalHolidaySource`; adapter master-data bật bằng `@ConditionalOnBean(PublicHolidays.class)`; không có master-data thì đọc **pack cùng định dạng** đóng sẵn trong starter, người vận hành nhập pack mới không cần deploy | Một nơi công bố, nhiều nơi dùng; vẫn cài calendar độc lập được (quy ước 6) |
| C3 | **Mô hình "phương án nghỉ" (holiday option)** cho khu vực doanh nghiệp: văn bản hằng năm thường cho người sử dụng lao động *chọn* (Quốc khánh 2026: 2/9 + chọn 1/9 hoặc 3/9; Tết 2027: chọn 1 trong 3 phương án). Dữ liệu quốc gia mang nhóm phương án; lịch tenant ghi phương án đã chọn, chưa chọn thì dùng phương án mặc định của nhóm | Đúng văn bản 2026–2027 (Phụ lục B). Không hệ tham khảo nào mô hình hoá điều này |
| C4 | **Âm lịch VN tự cài theo UTC+7**, không dùng `cn.6tail:lunar` ở runtime. Đã chạy thử: thư viện này (lịch Trung Quốc, UTC+8) cho **Tết 2030 là 03/02** trong khi lịch VN là **02/02**; lệch tương tự ở Tết 2007, 2053 và Giỗ Tổ 2037, 2038, 2085, 2086, 2095, 2100 (mục 2.4) | Giỗ Tổ và Tết là ngày nghỉ luật định; sai một ngày là sai SLA. Thuật toán thiên văn (Meeus) công khai, ~250 dòng, test bằng bảng ngày Tết đã công bố |
| C5 | **Lõi tính toán là hàm thuần trên "lịch đã biên dịch"** (`CompiledCalendar`): mảng khoảng làm việc theo năm + tổng tích luỹ, nên `addBusinessDays`, `addBusinessDuration`, `businessDurationBetween` là O(log n), không I/O. Cache theo `(tenant, key, revision, năm)` | SLA của workflow/helpdesk được gọi rất nhiều; chạy ở chế độ `remote` chỉ cần tải snapshot |
| C6 | **Quy tắc đếm ngày có tên, chọn được theo lời gọi** (`DayCountRule`): `LEGAL` (không tính ngày bắt đầu, hạn là cuối ngày làm việc thứ N, theo BLDS 2015 Điều 147–148), `SAME_TIME` (giữ giờ trong ngày, kiểu "2bd" của workflow), `CUTOFF(15:00)` (nhận sau giờ chốt thì tính từ ngày làm việc kế tiếp) | Mỗi nghiệp vụ hiểu "N ngày làm việc" khác nhau; không ép một nghĩa |
| C7 | **Năm chưa có dữ liệu chính thức**: tính bằng *dự đoán theo quy tắc* (âm lịch + Điều 112) và gắn cờ `provisional`; lời gọi có thể chọn `UnknownYearPolicy.CONSERVATIVE` (chỉ trừ cuối tuần: hạn sớm hơn, an toàn cho hạn luật như privacy-dsr) | Đúng yêu cầu của privacy-dsr: "sai số nghiêng về an toàn" |
| C8 | **Kết quả mang `CalendarVersion`** (`default@r12+VN@2026.10.09`). Hạn đã lưu không tự đổi khi lịch đổi; module dùng nghe `BusinessCalendarChanged{key, affectedFrom, affectedTo}` để tự quyết tính lại | privacy-dsr lưu `calendar_version`; workflow tăng `generation` khi tính lại |
| C9 | **Nhắc việc (`Reminders`) chạy trên `DurableTaskQueue`** (queue `calendar`, `runAt`, idempotency key), mốc theo thời gian thật hoặc theo giờ làm việc (`-4bh` trước hạn), lặp theo RRULE, kênh qua notification nếu có, luôn phát event `ReminderDue` | Một hệ việc nền (quy ước 5.2); collaboration và privacy-dsr chuyển sang dùng mà không đổi hợp đồng |
| C10 | **Không dùng Spring Modulith Moments cho sự kiện nghiệp vụ.** Đọc mã nguồn 2.1.1: Moments chạy `@Scheduled("@hourly"/"@daily")` ở **mọi** instance, một `zone-id` cho cả app, không lưu mốc đã phát, không bù khi app tắt. Calendar tự phát `BusinessDayStarted`, `BusinessPeriodStarted`… theo múi giờ và lịch của từng tenant, bằng `@ClusterScheduled` + watermark + idempotency | Nhiều replica sẽ phát trùng; tenant ở múi giờ khác nhau; mất sự kiện khi deploy là không chấp nhận được với automation |
| C11 | **Lịch lặp và iCalendar dùng ical4j 4.4.0 (BSD-3-Clause)** ở server. Frontend không tự mở rộng RRULE; xem trước lần lặp gọi API | Một bộ máy RRULE duy nhất, tránh lệch giữa JS và Java; `rrule` (npm) 2.8.1 không cập nhật từ 11/2023 |
| C12 | **Sổ lịch (agenda) gọn + nguồn hợp nhất.** Calendar lưu sự kiện của chính nó (lịch công tác tuần, lịch họp, sự kiện công ty) và gom sự kiện của module khác qua SPI `CalendarFeedContributor` (to-do, hạn hồ sơ, lịch hẹn booking, lịch xuất bản CMS) mà **không sao chép** dữ liệu | Lịch công tác tuần của lãnh đạo là nhu cầu rất phổ biến ở cơ quan VN; một màn hình lịch cho mọi thứ có hạn |
| C13 | **Gán lịch cho đối tượng** (`cal_assignment`: người dùng, đơn vị tổ chức, tài nguyên) có hiệu lực theo ngày; `CalendarResolver` tìm lịch theo thứ tự người → đơn vị (qua organization nếu có) → mặc định tenant | Workflow cần "hạn theo lịch của chi nhánh người xử lý"; booking cần lịch của bác sĩ/quầy |
| C14 | **Web dùng FullCalendar 7.1.3 (MIT), chỉ plugin chuẩn** (`daygrid`, `timegrid`, `list`, `interaction`). Không dùng plugin premium (`resource-timeline`… có license thương mại riêng) | License sạch cho khách on-prem; lịch tài nguyên dạng timeline để booking tự làm khi cần |

---

## 1. Bài toán và phạm vi

### 1.1 Use case

| Nhóm | Hiện tại | Tương lai |
|---|---|---|
| Lịch làm việc | Tuần làm việc, giờ làm theo ca, nghỉ trưa, thứ Bảy cách tuần, ca đêm qua nửa đêm, lịch theo chi nhánh/phòng/ca, kế thừa lịch cha, đổi giờ có hiệu lực từ ngày | Lịch theo cá nhân kết hợp nghỉ phép (đọc từ HRM của dự án qua SPI) |
| Ngày nghỉ | Lịch quốc gia VN (người lao động `VN`, khu vực công `VN-GOV`), nghỉ bù, hoán đổi, làm bù, phương án do doanh nghiệp chọn; ngày nghỉ riêng của tenant; lịch nước khác qua pack | Ngày nghỉ theo địa phương (lễ hội tỉnh) nếu có văn bản |
| Tính toán | `isWorkingTime`, `addBusinessDays`, `addBusinessDuration`, `businessDurationBetween`, ngày làm việc kế tiếp/trước, điều chỉnh ngày rơi vào ngày nghỉ, khoảng làm việc trong một khoảng thời gian | SLA nhiều mức có tạm dừng (`SlaTimer`) dùng chung cho helpdesk |
| Âm lịch | Đổi âm ↔ dương theo UTC+7, tháng nhuận, can chi, hiển thị ngày âm trên lịch, sinh nháp ngày lễ năm mới | Ngày giỗ, rằm/mùng một cho app tôn giáo, bán lẻ |
| Nhắc việc | Nhắc tại thời điểm, trước/sau mốc theo giờ thật hoặc giờ làm việc, lặp RRULE, kênh qua notification, event cho app | Nhắc thông minh theo lịch rảnh của người nhận |
| Sự kiện thời gian | `BusinessDayStarted/Ended`, `BusinessPeriodStarted` (tuần/tháng/quý/năm theo ngày làm việc), lịch chạy `ScheduleSpec` ("ngày làm việc đầu tiên của tháng 08:30") cho automation, export định kỳ | Trigger theo âm lịch |
| Sổ lịch | Lịch công tác tuần, lịch họp, sự kiện công ty, lặp RRULE, ngoại lệ một lần, người tham dự (user nội bộ), mức hiển thị, xuất/nhập ICS, link đăng ký (webcal) | Đồng bộ hai chiều CalDAV/Google/Microsoft 365 qua adapter |
| Giao diện | Lịch tháng/tuần/ngày/danh sách, chọn ngày làm việc, xem trước hạn, trình soạn mẫu tuần, trình soạn RRULE | Lịch tài nguyên dạng timeline |

### 1.2 Kịch bản nghiệp vụ phải đáp ứng

| # | Kịch bản | Đặc thù | Cần gì | Khi nào |
|---|---|---|---|---|
| K1 | **DSR theo NĐ 356/2025**: yêu cầu nhận 16:40 thứ Sáu 20/11/2026; hạn phản hồi 2 ngày làm việc. Thứ Ba 24/11 là Ngày Văn hoá Việt Nam (NQ 28/2026/QH16) | VN, luật | `addBusinessDays(t, 2, LEGAL)` = hết ngày thứ Tư 25/11 (bỏ qua T7, CN và 24/11); trả `CalendarVersion` để privacy lưu | Hiện tại |
| K2 | **Một cửa ở UBND phường**: hồ sơ nộp sau giờ chốt chiều được tính từ ngày làm việc kế tiếp; hạn giải quyết 10 ngày làm việc; trả kết quả trước 16:00 ngày hạn | VN, khu vực công, lịch `VN-GOV` có hoán đổi (thứ Hai 31/8/2026 nghỉ, thứ Bảy 22/8/2026 làm bù) | `DayCountRule.cutoff(15:00)`, lịch kế thừa `VN-GOV`, làm bù dùng giờ của ngày được hoán đổi | Hiện tại |
| K3 | **Doanh nghiệp chọn phương án Quốc khánh 2026**: nghỉ 2/9 và chọn 3/9 (thay vì 1/9) | VN, BLLĐ 2019 Điều 112; thông báo 2026 | Nhóm phương án trong dữ liệu quốc gia; tenant chọn trong admin; hạn đang mở được báo `BusinessCalendarChanged` | Hiện tại |
| K4 | **Tết 2027 (TB 11483/TB-BNV)**: khu vực công nghỉ 04–10/02/2027; doanh nghiệp chọn 1 trong 3 phương án, phải báo người lao động trước 30 ngày | VN | `VN-GOV` có 5 ngày Tết + 2 nghỉ bù; `VN` có nhóm phương án 3 lựa chọn; admin nhắc tenant chọn trước 30 ngày | Hiện tại |
| K5 | **Helpdesk CSKH 2 khung**: kênh thường 08:00–17:30 T2–T6 + 08:00–12:00 T7; kênh VIP 24x7 trừ mùng 1–3 Tết. Hạn phản hồi 4 giờ làm việc, tạm dừng khi chờ khách | Nhiều lịch trong một tenant | Hai lịch, lịch VIP `nationalCode = none` + ngoại lệ lặp theo âm lịch; `SlaTimer` (chạy/dừng) | Đợt C (helpdesk) |
| K6 | **Nhà máy 3 ca**: ca đêm 22:00–06:00 qua nửa đêm, thứ Bảy làm cách tuần (chu kỳ 2 tuần từ mốc 05/01/2026) | Ca, chu kỳ | Khoảng qua nửa đêm thuộc ngày bắt đầu; `cycleWeeks = 2`, `anchorDate` | Hiện tại |
| K7 | **Chuỗi bán lẻ có chi nhánh ở Singapore**: chi nhánh SG dùng lịch nghỉ `SG`, múi giờ `Asia/Singapore`; workflow phê duyệt liên chi nhánh tính hạn theo lịch của người xử lý | Đa quốc gia | Pack `holiday-day:SG`; `CalendarResolver.forSubject(user)` theo đơn vị | Tương lai gần |
| K8 | **Nhắc duyệt đơn nghỉ phép** (collaboration K3): hạn 2 ngày làm việc, nhắc trước 4 giờ làm việc, quá hạn báo trưởng phòng | — | `Reminders.schedule(anchor=due, offset=-4bh)`; event `ReminderDue`; tính lại khi lịch đổi | Hiện tại |
| K9 | **Báo cáo tự động ngày làm việc đầu tiên của tháng lúc 08:30** (automation, export); nếu ngày 1 rơi vào Tết thì chạy ngày làm việc kế tiếp | VN | `ScheduleSpec.nthBusinessDay(1, MONTH, 08:30)`; sự kiện không trùng giữa replica, bù khi app tắt | Đợt C (automation) |
| K10 | **Lịch công tác tuần của Giám đốc Sở** công bố công khai trên cổng thông tin, cán bộ đăng ký theo dõi trên điện thoại | VN, khu vực công | Sổ lịch `agenda` của đơn vị, mức `PUBLIC`, xuất ICS, link webcal có token thu hồi được; trang công khai đọc qua BFF | Hiện tại |
| K11 | **Phòng khám đặt lịch**: slot khám chỉ trong giờ làm việc của bác sĩ trừ ngày nghỉ và giờ bận | Booking | `workingIntervals(range)` theo lịch gán cho bác sĩ, trừ khoảng bận từ `CalendarFeedContributor` của booking | Đợt C (booking) |
| K12 | **Form đặt phòng**: "ngày nhận phòng phải sau hôm nay ít nhất 1 ngày làm việc"; date picker ẩn ngày nghỉ, hiện ngày âm và tên ngày lễ | forms, web | `SubmissionInterceptor` mẫu của calendar; `<BusinessDatePicker>` | Hiện tại |
| K13 | **Cuối năm chưa có thông báo nghỉ Tết năm sau** (thường có vào tháng 10–12): hạn SLA tháng 2 năm sau được tính theo dự đoán, gắn `provisional`, cảnh báo người vận hành; khi văn bản ra, công bố dữ liệu, tenant được báo hạn đang mở bị ảnh hưởng | VN | Sinh nháp theo âm lịch (`HolidayDraftGenerator`), health `calendar.national-data`, event `BusinessCalendarChanged` | Hiện tại |
| K14 | **Khách on-prem không ra Internet** nhập pack lịch nghỉ bằng file đã ký; không cài master-data | VN, on-prem | Pack định dạng master-data 4.5, kiểm chữ ký Ed25519, lưu trong `cal_holiday_pack` | Hiện tại |
| K15 | **Người dùng nhập lịch họp từ Outlook (.ics)** và đăng ký lịch nghỉ của công ty vào Google Calendar cá nhân | Interop | Nhập/xuất RFC 5545 (VEVENT, RRULE, EXDATE, VTIMEZONE) | Hiện tại |
| K16 | **Hạn nộp hồ sơ theo BLDS 2015 Điều 148**: ngày cuối của thời hạn (tính theo ngày lịch) rơi vào ngày nghỉ thì lùi sang hết ngày làm việc kế tiếp | VN, luật | `adjust(date, NEXT_WORKING_DAY)`; `addCalendarDays(..., rollForward=true)` | Hiện tại |

### 1.3 Không thuộc phạm vi

- **Nghỉ phép, chấm công, bảng lương của nhân viên.** Thuộc HRM của dự án; calendar chỉ có SPI `PersonalAbsenceSource`
  để trừ ngày vắng khi tính lịch rảnh (bản sau).
- **Đặt lịch hẹn, slot, sức chứa, giữ chỗ.** Thuộc `booking`; calendar cho giờ làm việc và khoảng bận.
- **Workflow, SLA nhiều mức, escalation.** Thuộc workflow/helpdesk; calendar cho phép tính và `SlaTimer`.
- **Lưu dữ liệu lịch nghỉ quốc gia khi có master-data.** master-data giữ; calendar chỉ đọc.
- **Đồng bộ hai chiều với Google/Microsoft/CalDAV.** Bản sau, qua SPI `ExternalCalendarSync`.
- **Phòng họp, tài nguyên, check-in.** booking.

---

## 2. Nghiên cứu

### 2.1 Văn bản pháp lý VN về ngày nghỉ và cách tính thời hạn

| Văn bản | Nội dung dùng cho thiết kế | Kiểm |
|---|---|---|
| **Bộ luật Lao động 2019** (45/2019/QH14), Điều 112 | Ngày nghỉ lễ, tết hưởng nguyên lương: Tết Dương lịch 1 ngày (1/1); Tết Âm lịch 5 ngày; Chiến thắng 30/4 1 ngày; Quốc tế lao động 1/5 1 ngày; Quốc khánh 2 ngày (2/9 và 1 ngày liền kề trước hoặc sau); Giỗ Tổ Hùng Vương 1 ngày (10/3 âm lịch). Trùng ngày nghỉ hằng tuần thì nghỉ bù ngày kế tiếp. Thủ tướng quyết định cụ thể ngày nghỉ Tết Âm lịch và Quốc khánh | Nguồn thứ cấp ([vneconomy](https://vneconomy.vn/nguoi-lao-dong-co-22-ngay-nghi-le-tet-trong-nam-2025.htm), [thuvienphapluat](https://thuvienphapluat.vn/hoi-dap-phap-luat/tu-khi-nao-le-quoc-khanh-duoc-nghi-2-ngay-138014384.html)); trang văn bản gốc trên thuvienphapluat trả 403 khi kiểm. Nội dung trùng với mã nguồn `python-holidays` (dẫn BLLĐ) |
| **Nghị quyết 28/2026/QH16** về phát triển văn hoá Việt Nam, Điều 2 | Ngày 24/11 hằng năm là **Ngày Văn hoá Việt Nam**, người lao động được nghỉ, hưởng nguyên lương. Thông qua 24/04/2026, hiệu lực 01/07/2026. Tổng ngày nghỉ lễ, tết thành 12 | [LuatVietnam](https://luatvietnam.vn/tin-van-ban-moi/tu-hom-nay-01-7-2026-nguoi-lao-dong-chinh-thuc-co-them-1-ngay-nghi-huong-nguyen-luong-186-109961-article.html), [Xây dựng chính sách – Chính phủ](https://xaydungchinhsach.chinhphu.vn/ngay-24-11-hang-nam-la-ngay-van-hoa-viet-nam-nguoi-lao-dong-duoc-nghi-huong-nguyen-luong-119260113152642414.htm). **Điều 112 BLLĐ chưa được sửa** theo: dự thảo Luật sửa đổi BLLĐ 2026 đang lấy ý kiến bổ sung 24/11 vào Điều 112 ([thuvienphapluat](https://thuvienphapluat.vn/banan/tin-tuc/toan-van-du-thao-luat-sua-doi-bo-luat-lao-dong-moi-nhat-2026-bo-sung-ngay-nghi-le-tet-trong-nam-chi-50507.html)). Hệ quả: quy tắc nghỉ bù khi 24/11 trùng cuối tuần chưa rõ → dữ liệu theo thông báo hằng năm, không suy ra |
| **Ngày Văn hoá Việt Nam 2026** | Nghỉ thứ Ba 24/11/2026, **không hoán đổi** (Phó Thủ tướng đồng ý phương án của Bộ Nội vụ, tin 02/10/2026). Phương án 4 ngày (đổi 23/11 sang 28/11) bị bỏ | [Tiền Phong](https://tienphong.vn/cong-chuc-nguoi-lao-dong-duoc-nghi-ngay-2411-khong-hoan-doi-ngay-lam-viec-post1881722.tpo), [Dân trí](https://dantri.com.vn/noi-vu/chinh-phu-chot-nghi-1-ngay-ngay-van-hoa-viet-nam-2026-20260919231056506.htm), [Thanh Niên](https://thanhnien.vn/bo-noi-vu-thay-doi-de-xuat-ngay-van-hoa-viet-nam-nghi-1-ngay-khong-hoan-doi-185260920073806526.htm) |
| **Thông báo 9441/TB-BNV** (Quốc khánh 2026) | Khu vực công nghỉ 29/8–02/9/2026 (5 ngày): hoán đổi thứ Hai 31/8 sang **thứ Bảy 22/8 làm bù**. Doanh nghiệp: nghỉ 2/9 và **chọn** 1/9 hoặc 3/9 | [Báo Lào Cai](https://baolaocai.vn/thu-bay-ngay-228-nhung-ai-phai-di-lam-bu-post906228.html), [Tạp chí Công Thương](https://tapchicongthuong.vn/chinh-thuc--cong-chuc--nguoi-lao-dong-duoc-nghi-5-ngay-dip-le-quoc-khanh-2026-538937.htm), [Hà Tĩnh](https://hatinh.gov.vn/vi/bai-viet/-ha-tinh-trien-khai-lich-nghi-le-quoc-khanh-29-nam-2026) |
| **Thông báo 11483/TB-BNV** (09/10/2026, Tết Âm lịch và Quốc khánh 2027) | Khu vực công nghỉ Tết **04/02–10/02/2027** (5 ngày Tết + 2 ngày nghỉ bù), Quốc khánh 4 ngày 02/9–05/9/2027. Doanh nghiệp chọn 1 trong 3 phương án Tết (1+4, 2+3, 3+2 ngày trước/sau Tết), báo người lao động trước ít nhất 30 ngày | [LuatVietnam](https://luatvietnam.vn/linh-vuc-khac/lich-nghi-tet-nguyen-dan-2027-duoc-chot-the-nao-tai-thong-bao-11483-tb-bnv-883-113717-article.html), [Báo Chính phủ](https://baochinhphu.vn/nghi-tet-am-lich-2027-keo-dai-7-ngay-nghi-quoc-khanh-4-ngay-102261010095648278.htm), [Tuổi Trẻ](https://tuoitre.vn/bo-noi-vu-thong-bao-chi-tiet-lich-nghi-tet-am-lich-quoc-khanh-2027-100261010082304185.htm), [VietnamPlus](https://www.vietnamplus.vn/bo-noi-vu-huong-dan-doanh-nghiep-3-phuong-an-nghi-tet-am-lich-nam-2027-post1141406.vnp) |
| **Bộ luật Dân sự 2015**, Điều 147–148 | Thời hạn tính bằng ngày: ngày đầu tiên không tính; kết thúc vào thời điểm kết thúc ngày cuối cùng; ngày cuối cùng là ngày nghỉ cuối tuần hoặc ngày lễ thì kết thúc vào thời điểm kết thúc **ngày làm việc tiếp theo** | Nguồn thứ cấp ([Tạp chí Tòa án](https://tapchitoaan.vn/khong-co-vuong-mac-ve-tinh-thoi-han-trong-to-tung-dan-su), [Luật Minh Khuê](https://luatminhkhue.vn/thoi-han-la-gi-cach-tinh-thoi-han-theo-quy-dinh-cua-phap-luat.aspx)). Là cơ sở của `DayCountRule.LEGAL` và `adjust(NEXT_WORKING_DAY)` |
| **NĐ 356/2025/NĐ-CP** | Bên kiểm soát dữ liệu phản hồi trong 02 ngày làm việc (bảng SLA trong thiết kế privacy-dsr mục 2.1) | Đã kiểm ở thiết kế privacy-dsr |
| Giờ chốt nhận hồ sơ một cửa | Nhiều cơ quan quy định giờ chốt nhận hồ sơ trong ngày (nhận sau giờ chốt tính sang ngày làm việc kế tiếp). **Chưa tìm được điều khoản gốc** trong NĐ 118/2025/NĐ-CP | Thiết kế để giờ chốt là **cấu hình theo lời gọi/quy trình**, không ghi cứng |

**Kết luận cho mô hình dữ liệu:**

1. Ngày nghỉ cụ thể từng năm *không suy ra được hoàn toàn* từ luật: Tết do Thủ tướng chọn (ngày trước/sau giao thừa),
   Quốc khánh có ngày liền kề do người sử dụng lao động chọn, hoán đổi và làm bù khác giữa khu vực công và doanh nghiệp,
   24/11 chưa có quy tắc nghỉ bù. → Dữ liệu năm là **dữ liệu công bố** có nguồn văn bản; quy tắc chỉ dùng để *sinh nháp*
   và *dự đoán*.
2. Có ít nhất hai lịch quốc gia khác nhau trong cùng năm: `VN` (người lao động nói chung, có nhóm phương án) và `VN-GOV`
   (cán bộ, công chức, viên chức; có hoán đổi và làm bù). Khớp với master-data mục 3.9.
3. Có **ngày làm bù vào thứ Bảy** (22/8/2026) → lịch phải biết *ngày đó làm theo giờ của ngày nào* (giờ của thứ Hai
   31/8 được hoán đổi).
4. Văn bản ra **muộn** (Tết 2027 công bố 09/10/2026) và có thể đổi sau khi đã đề xuất (24/11/2026 đổi từ 4 ngày sang
   1 ngày) → cần version, diff, event và chính sách cho năm chưa công bố.

### 2.2 So sánh hệ thống tham khảo

| Hệ thống | Phiên bản, license | Mô hình dữ liệu | API | Điểm mạnh | Điểm yếu |
|---|---|---|---|---|---|
| **Odoo `resource.calendar`** ([mã nguồn 19.0](https://github.com/odoo/odoo/blob/19.0/addons/resource/models/resource_calendar.py)) | Odoo 19 (09/2025), LGPL-3.0 | `resource.calendar` (`tz`, `hours_per_day`, `two_weeks_calendar`, `schedule_type` fixed/flexible, `full_time_required_hours`) → `attendance_ids` (`dayofweek`, `hour_from`, `hour_to`, `day_period` sáng/chiều/nghỉ trưa, `week_type` cho tuần chẵn/lẻ) và `leave_ids`/`global_leave_ids` (`resource.calendar.leaves` có `date_from`, `date_to`, `resource_id` rỗng = ngày nghỉ chung) | Python: `plan_days(days, day_dt, compute_leaves)`, `plan_hours(hours, day_dt)`, `get_work_hours_count(start, end)`, `_work_intervals_batch(start, end, resources)` = attendance trừ leave | Mô hình gọn, tính bằng *khoảng* (intervals) — cùng ý tưởng `CompiledCalendar`; nghỉ chung và nghỉ theo người tách bạch; lịch hai tuần | Chỉ có chu kỳ 2 tuần; ngày lễ nhập tay theo công ty, không có lớp quốc gia/version; không có ngày làm bù; giờ là float (`8.5`) dễ lỗi làm tròn |
| **Frappe/ERPNext Holiday List** ([docs](https://docs.frappe.io/erpnext/user/manual/en/holiday-list)) | Frappe HR v15/v16, GPL-3.0 | `Holiday List` (from/to date, bảng `holidays`), "Add Weekly Holidays" sinh cả ngày nghỉ tuần thành dòng; nhập ngày lễ theo quốc gia bằng gói `holidays` (python); gán mặc định ở Company, có Holiday List Assignment, Workstation | DocType REST chung | Đơn giản, người dùng hiểu ngay; nhập ngày lễ theo quốc gia | Ngày nghỉ tuần là dòng dữ liệu (phải sinh lại mỗi năm); không có giờ làm việc trong Holiday List (nằm ở Shift Type); không có làm bù |
| **Jira Service Management SLA calendars** ([docs](https://support.atlassian.com/jira-service-management-cloud/docs/set-up-sla-calendars/)) | Cloud (SaaS) | Calendar: tên, múi giờ, giờ làm theo từng thứ (nhiều khoảng một ngày, mặc định 09:00–17:00), danh sách ngày nghỉ; SLA chọn calendar ở từng goal | UI, REST nội bộ | Nhiều khoảng một ngày (nghỉ trưa), calendar gắn theo goal (mỗi loại yêu cầu một lịch) | Ngày lễ nhập tay; không kế thừa; không có lịch quốc gia |
| **Zendesk Schedules** ([API](https://developer.zendesk.com/api-reference/ticketing/ticket-management/schedules/)) | SaaS API v2 | `schedule {name, time_zone, intervals[{start_time, end_time}]}` — phút tính từ 0h Chủ nhật, không vượt nửa đêm; `holidays {name, start_date, end_date}` nhiều ngày | REST `/business_hours/schedules/{id}/holidays?start_date&end_date` | Biểu diễn khoảng bằng số phút gọn, dễ so sánh; ngày nghỉ nhiều ngày một dòng | Ca qua nửa đêm phải tách thủ công; không làm bù; không phương án |
| **Google Calendar API v3 Events** ([reference](https://developers.google.com/workspace/calendar/api/v3/reference/events)) | SaaS | `start`/`end` (`date` cho cả ngày hoặc `dateTime` + `timeZone` IANA), `recurrence[]` (RRULE/EXRULE/RDATE/EXDATE, không có DTSTART), instance có `recurringEventId` + `originalStartTime`, `iCalUID`, `sequence`, `eventType` (`default`, `outOfOffice`, `focusTime`, `workingLocation`…), `reminders.overrides` (≤ 5, `email`/`popup`, 0–40320 phút), `transparency` | REST, `instances`, đồng bộ tăng dần bằng `syncToken` | Chuẩn de facto cho sự kiện lặp và ngoại lệ; tách cả ngày/giờ; `transparency` cho free/busy | Không có khái niệm ngày làm việc/SLA; reminder không theo giờ làm việc |
| **python-holidays** ([vietnam.py](https://github.com/vacanza/holidays/blob/main/holidays/countries/vietnam.py)) | 0.107 (PyPI), MIT | Lớp quốc gia sinh ngày lễ theo quy tắc (lịch Trung Quốc cho Tết), `special_public_holidays` theo năm dạng *(ngày nghỉ, ngày làm bù)* (vd. 2026: 31/8 ↔ 22/8) | Thư viện | Đã có 24/11, có làm bù theo năm kèm nguồn | Tết lấy từ lịch Trung Quốc; chỉ một lịch (không tách khu vực công/doanh nghiệp); không có phương án; Python |
| **Jollyday** ([repo](https://github.com/focus-shift/jollyday), [Holidays_vn.xml](https://github.com/focus-shift/jollyday/blob/main/jollyday-core/src/main/resources/holidays/Holidays_vn.xml)) | v2.22.1 (15/10/2026), Apache-2.0, Java | XML theo quốc gia, quy tắc cố định/di động | Java `HolidayManager` | Java, nhiều nước | File VN **chỉ có 5 ngày dương lịch cố định** (1/1, 30/4, 1/5, 2/9, 24/11), **không có Tết, Giỗ Tổ**, không nghỉ bù → không dùng được cho VN |
| **ical4j** ([repo](https://github.com/ical4j/ical4j)) | 4.4.0 (Maven Central, 02/10/2026), BSD-3-Clause | Mô hình RFC 5545 đầy đủ (VEVENT, VTODO, VTIMEZONE, `Recur`), dùng `java.time` từ 4.x | Java | Parse/sinh ICS chuẩn, mở rộng RRULE kèm múi giờ | API khá nặng; chỉ dùng ở biên (adapter), không lộ ra API công khai |
| **FullCalendar** ([repo](https://github.com/fullcalendar/fullcalendar)) | 7.1.3 (npm 16/10/2026), MIT; plugin premium có license riêng (`@fullcalendar/resource-timeline` "SEE LICENSE IN LICENSE.md") | Event object `{start, end, allDay, rrule?}`; `businessHours`; `eventSources` (hàm lấy theo khoảng) | React `@fullcalendar/react` 7.1.3 | Hiển thị tháng/tuần/ngày/danh sách, kéo thả, `businessHours` tô vùng giờ làm | Không tính ngày làm việc; plugin timeline tài nguyên không MIT |

### 2.3 Spring Modulith Moments (C10)

[Tài liệu 2.1.1](https://docs.spring.io/spring-modulith/reference/moments.html) và
[mã nguồn `Moments` tag 2.1.1](https://github.com/spring-projects/spring-modulith/blob/2.1.1/spring-modulith-moments/src/main/java/org/springframework/modulith/moments/support/Moments.java):

- Phát `HourHasPassed`, `DayHasPassed`, `WeekHasPassed`, `MonthHasPassed`, `QuarterHasPassed`, `YearHasPassed`; cấu hình
  `granularity`, `zone-id` (mặc định UTC), `locale`, `quarter-start-month`; `TimeMachine` dịch thời gian cho test.
- Cơ chế: `@Scheduled(cron = "@hourly")` và `@Scheduled(cron = "@daily")` trên bean `Moments`; **không lưu mốc đã phát,
  không bù khi app tắt**; trạng thái duy nhất là `shift` trong bộ nhớ. Mỗi replica tự phát → N replica phát N lần.
- platform-java đang dùng Spring Modulith 2.1.1 (`platform-bom`); bản mới nhất trên Maven Central là 2.2.0-RC1.

**Kết luận:** Không dùng Moments làm trigger nghiệp vụ. Học theo: tên event dạng "đã qua", `TimeMachine` cho test →
calendar dùng `java.time.Clock` bean và cung cấp `CalendarTimeTravel` trong `platform-test` để dịch thời gian và chạy
bù mọi tick trung gian (giống `shiftBy`). App vẫn dùng Moments cho việc nội bộ không cần chính xác được, calendar không
chặn.

### 2.4 Âm lịch Việt Nam: kiểm `cn.6tail:lunar` (C4)

`cn.6tail:lunar` 1.7.7 (Maven Central, cập nhật 05/11/2025, MIT, Java 1.5+, không phụ thuộc) là thư viện âm lịch Trung
Quốc. Lịch VN dùng cùng quy tắc (tháng bắt đầu ở ngày có trăng mới, tháng 11 chứa Đông chí, tháng nhuận là tháng đầu tiên
không có trung khí) nhưng **tính theo giờ Hà Nội UTC+7** từ 1968, Trung Quốc theo UTC+8
([Nhân Dân](https://nhandan.vn/chung-quanh-thong-tin-ve-ngay-am-lich-khac-nhau-tren-mot-so-loai-lich-post489744.html),
[RFA](https://www.rfa.org/vietnamese/in_depth/WhyVietnameseLunarCalendarDiffersFromChina_GMinh-20070207.html),
[quy tắc tính của Hồ Ngọc Đức](https://www.informatik.uni-leipzig.de/~duc/amlich/calrules_en.html)).

Kiểm thực tế (27/10/2026): viết thuật toán trăng mới/kinh độ mặt trời theo Meeus với tham số múi giờ, so với
`Lunar.fromYmd(y, m, d).getSolar()` của 1.7.7 cho mồng 1 tháng Giêng và 10/3 âm lịch, năm 1900–2100. Với tham số UTC+8,
thuật toán trùng thư viện ở mọi năm có Tết; với UTC+7 thì lệch:

| Năm | Tết VN (UTC+7) | `cn.6tail:lunar` | Giỗ Tổ VN (UTC+7) | `cn.6tail:lunar` |
|---|---|---|---|---|
| 1985 | 21/01/1985 | 20/02/1985 (lệch **một tháng**, đúng sự kiện lịch sử) | — | — |
| 2007 | 17/02/2007 | 18/02/2007 | — | — |
| 2009 | — | — | 04/04/2009 | 05/04/2009 |
| 2030 | **02/02/2030** | 03/02/2030 | — | — |
| 2037 | — | — | 24/04/2037 | 25/04/2037 |
| 2038 | — | — | 13/04/2038 | 14/04/2038 |
| 2053 | 18/02/2053 | 19/02/2053 | — | — |
| 2085, 2086, 2095, 2100 | — | — | lệch 1 ngày | |

Đối chiếu năm hiện hành: thuật toán UTC+7 cho Tết 2026 = 17/02/2026, Giỗ Tổ 2026 = 26/04/2026 (khớp thông báo nghỉ
bù 27/4/2026), Tết 2027 = 06/02/2027 (khớp TB 11483: 04/02 là 28 tháng Chạp... 10/02 là mùng 5), Giỗ Tổ 2027 =
16/04/2027.

**Quyết định:** cài `VietnameseLunarCalendar` riêng (thuật toán thiên văn công khai, viết lại từ công thức của Meeus,
không chép mã có điều khoản "phi thương mại"), tham số múi giờ mặc định 7. `cn.6tail:lunar` chỉ là **test dependency**
để đối chiếu tự động: hai bên phải trùng khi đặt múi giờ 8, và danh sách năm lệch khi đặt múi giờ 7 phải đúng bảng trên.
Phạm vi hỗ trợ 1900–2199; ngoài phạm vi ném `LunarOutOfRangeException`.

### 2.5 Học theo và không làm

| Học theo | Từ |
|---|---|
| Tính bằng *khoảng làm việc* (attendance − leave), có hàm batch theo khoảng thời gian | Odoo `_work_intervals_batch` |
| Nhiều khoảng mỗi ngày (nghỉ trưa), nhiều lịch, mỗi SLA chọn lịch | Jira SLA calendars |
| Ngày nghỉ nhiều ngày là một dòng `start_date..end_date` | Zendesk holidays |
| Làm bù theo năm dạng *(ngày nghỉ ↔ ngày làm)* kèm nguồn văn bản | python-holidays `special_public_holidays` |
| `start.date` vs `start.dateTime` + `timeZone`; `recurringEventId`/`originalStartTime` cho ngoại lệ; `iCalUID` + `sequence`; `transparency` | Google Calendar API |
| `TimeMachine` dịch thời gian trong test | Spring Modulith Moments |
| `businessHours` để tô vùng làm việc trên lịch | FullCalendar |

| Không làm | Lý do |
|---|---|
| Sinh ngày nghỉ tuần thành dòng dữ liệu (Frappe) | Phình dữ liệu, phải sinh lại hằng năm; mẫu tuần + biên dịch là đủ |
| Giờ dạng số thực (Odoo `8.5`) | Lỗi làm tròn; dùng `LocalTime` và phút |
| Lấy Tết/Giỗ Tổ từ thư viện lịch Trung Quốc (python-holidays, `cn.6tail:lunar`) | Sai ở 2030, 2053, Giỗ Tổ 2037/2038 (mục 2.4) |
| Moments làm trigger nghiệp vụ | Không cluster-safe, không bù, một múi giờ (mục 2.3) |
| Server CalDAV đầy đủ ở bản đầu | Lớn, ít dự án cần; xuất ICS + webcal đủ cho K10, K15 |
| Mở rộng RRULE ở trình duyệt | Hai bộ máy cho ra hai kết quả khác nhau ở biên múi giờ |
| Plugin premium của FullCalendar | License thương mại riêng |

---
