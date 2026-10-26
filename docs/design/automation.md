# Thiết kế module automation: `platform-starter-automation` + `@vyforge/automation` + `@vyforge/automation-admin`

> **Trạng thái:** bản thiết kế chờ duyệt (27/10/2026), phiên forge-73. Chưa có code.
>
> **Phạm vi:** luật tự động "khi X thì Y" do tenant admin tự cấu hình, không cần dev:
> - trigger: event công khai của các module (`TenantAwareEvent`), thay đổi bản ghi, lịch chạy theo ngày làm việc
>   (`calendar`), mốc tương đối theo trường ngày của bản ghi, nút bấm thủ công, webhook gọi vào;
> - điều kiện viết bằng CEL có sandbox (không script tuỳ ý), soạn bằng trình dựng trực quan;
> - hành động qua SPI `AutomationAction` do module cài: gửi thông báo, phát webhook, tạo việc, giao việc, đặt giá trị
>   field, khởi động workflow, chạy export…;
> - chống vòng lặp (độ sâu chuỗi, tần suất theo luật và theo tenant, tự tạm dừng), log từng lần chạy, chạy thử, bật/tắt,
>   version luật;
> - admin cho tenant admin: trình dựng luật trên `admin-kit`, lịch sử chạy, mẫu luật.
>
> **Nền:** platform-java 0.12.1 (`SecretCipher`, `DurableTaskQueue`, `StarterSchemaMigrator`), platform-web 0.9.0,
> module-conventions mục 1–6, `platform-services.md` (forge-37: Next.js Multi-Zones, `/m/<module>`,
> `@vyforge/admin-shell`, manifest `contract: 1`, `GET /api/v1/<module>/admin/info`), `admin-kit.md` (forge-49), gap
> analysis 10/2026 mục 2.11 và 6.12. Bản phát hành đích: `platform-java` 0.13.0 (`platform-starter-automation`,
> `platform-expression`), `platform-web` (`@vyforge/automation` 0.1.0), app `platform-admin/apps/automation-admin`.
>
> **Module liên quan (chỉ dùng qua interface, không phụ thuộc cứng):** `workflow` (forge-39), `calendar` (forge-55),
> `forms` (forge-56), `collaboration` (forge-40), `notification`, `webhooks`, `features`, `export`, `privacy-dsr`,
> `licensing`, `custom-fields` (đang thiết kế song song, đợt C). Ranh giới với `workflow` chốt ở mục 1.2.
>
> **Ghi chú độ tin cậy.** Version thư viện lấy từ `maven-metadata.xml` của Maven Central và GitHub API, kiểm ngày
> 27/10/2026. Hành vi của hệ thống tham khảo lấy từ tài liệu chính thức và **mã nguồn** (Odoo 19.0, Directus `main`,
> Frappe `develop`). Chỗ chưa đối chiếu được ghi **(chưa kiểm chứng)**.

## Mục lục

0. [Tóm tắt quyết định](#0-tóm-tắt-quyết-định)
1. [Bài toán và phạm vi](#1-bài-toán-và-phạm-vi)
2. [Nghiên cứu](#2-nghiên-cứu)
3. [Kiến trúc](#3-kiến-trúc)
4. [Mô hình dữ liệu](#4-mô-hình-dữ-liệu)
5. [API](#5-api)
6. [Mở rộng](#6-mở-rộng)
7. [Vận hành](#7-vận-hành)
8. [Bảo mật và tuân thủ](#8-bảo-mật-và-tuân-thủ)
9. [Kế hoạch triển khai](#9-kế-hoạch-triển-khai)
10. [Đóng gói và triển khai độc lập](#10-đóng-gói-và-triển-khai-độc-lập)
11. [Lệch quy ước và quyết định](#11-lệch-quy-ước-và-quyết-định)

---

## 0. Tóm tắt quyết định

**Quyết định về module:**
- **Thêm** starter `platform-starter-automation` (platform-java), package `@vyforge/automation` (platform-web), admin MFE
  `automation-admin` (platform-admin).
- **Thêm** thư viện nhỏ `platform-expression` (platform-java, không phải starter): môi trường CEL dùng chung, giới hạn chi
  phí, hàm chuẩn, SPI `ExpressionFunctions`. Workflow (W6) và automation cùng dùng; forms dùng khi có công thức (bản sau).
  Không tách thì có hai bản cấu hình sandbox CEL lệch nhau (Q1).
- **Không thêm** server riêng: chạy nhúng hoặc trong `platform-services`.
- **Không gộp** vào workflow: gap analysis 2.11 đã chốt tách; ranh giới ở mục 1.2.

| # | Quyết định | Lý do ngắn |
|---|---|---|
| A1 | **Mô hình ECA, chạy xong ngay.** Luật = một trigger + một điều kiện + danh sách hành động tuần tự (tối đa 10). Không có bước chờ người, không có bước "chờ X ngày" giữa hai hành động. Hẹn giờ thể hiện bằng trigger (lịch, mốc tương đối) hoặc tham số của hành động (hạn của việc, `scheduledAt` của thông báo) | Ranh giới với workflow (gap analysis 2.11). Bước chờ kéo theo trạng thái chạy dài, tức là một engine quy trình thứ hai |
| A2 | **Luật chạy sau commit, bất đồng bộ, qua `DurableTaskQueue`** (queue `automation`). Bộ so khớp trigger chạy trong transaction nghiệp vụ (listener `BEFORE_COMMIT`), chỉ enqueue khi có luật đang bật khớp loại trigger. Không có luật "chặn" giao dịch kiểu filter hook của Directus | Luật lỗi không làm hỏng nghiệp vụ; việc chỉ tồn tại khi transaction commit; tenant không viết được logic chặn giao dịch (chỗ đó là hook trong code) |
| A3 | **Điều kiện viết bằng CEL** (`dev.cel:cel` 0.14.0, Apache-2.0) qua `platform-expression`: biến có kiểu (sinh từ JSON Schema của trigger), kiểm kiểu khi publish, giới hạn kích thước biểu thức, độ sâu, số vòng lặp comprehension, số bước đánh giá. **Không dùng JSONLogic** làm ngôn ngữ thứ hai: trình dựng lưu cây điều kiện dạng JSON rồi biên dịch sang CEL | Cùng ngôn ngữ với workflow (W6). `json-logic-java` 1.1.0 (05/2025) không kiểm kiểu, cập nhật cuối 09/2025 |
| A4 | **Tham số của hành động là dữ liệu có JSON Schema**; giá trị động viết bằng biểu thức CEL (`{"$expr": "record.total * 0.1"}`) hoặc chuỗi có chỗ trống `{{ record.code }}` (mỗi chỗ trống là một biểu thức CEL). Không có ngôn ngữ template thứ hai | Một bộ kiểm, một sandbox. Nội dung thông báo dài vẫn soạn bằng template Liquid của `notification` |
| A5 | **Snapshot lúc trigger.** Run giữ bản chụp dữ liệu trigger (event payload, hoặc `new`/`old` của bản ghi) và đánh giá điều kiện trên bản chụp đó. Trigger theo mốc thời gian đọc lại bản ghi lúc đến hạn qua `RecordAdapter` | Điều kiện phản ánh đúng thời điểm sự kiện (giống Odoo đánh giá domain lúc ghi); mốc thời gian phải xét trạng thái hiện tại ("chưa thanh toán sau 7 ngày") |
| A6 | **Chống vòng lặp ba lớp:** (1) chuỗi nhân quả `AutomationChain` (id chuỗi, độ sâu, tập `ruleId:subject` đã chạy) đi theo mọi event và thay đổi bản ghi mà hành động gây ra; mặc định độ sâu tối đa 5; (2) một luật chạy tối đa một lần trên một bản ghi trong một chuỗi (học `__action_done` của Odoo); (3) giới hạn tần suất theo luật và theo tenant; luật bị chặn vì vòng lặp 3 lần trong 10 phút thì tự chuyển `SUSPENDED` và báo tenant admin | Odoo chặn trong một transaction; Jira chặn bằng ngưỡng 10 lần kích hoạt liên tiếp (`LOOP`). Ta cần cả hai vì luật chạy bất đồng bộ |
| A7 | **Version luật bất biến.** Sửa là tạo bản nháp; publish kiểm và biên dịch, tạo version mới; run ghi version đã dùng. Bật/tắt tách khỏi version. Khôi phục = publish lại bản cũ thành version mới | Run cũ giải thích được; giống Zapier (run hiện version) và VyFlow của workflow |
| A8 | **Hành động chạy với danh tính dịch vụ của luật, quyền theo người publish.** Khi publish, người publish phải có quyền IAM mà từng hành động yêu cầu (`requiredPermissions(params)`). Mỗi lần chạy kiểm lại tài khoản người publish còn hoạt động; mất quyền thì luật `SUSPENDED`. Audit ghi actor `automation:<ruleKey>` thay mặt `<sub>` | Tenant admin không dùng luật để vượt quyền của chính mình (Directus chạy flow theo `accountability`); không cần lưu token người dùng |
| A9 | **Trigger lấy từ catalog có khai báo.** Event phải có `@AutomationEvent` hoặc được `AutomationTriggerCatalog` khai báo (key, nhãn vi/en, JSON Schema, mức dữ liệu cá nhân, ví dụ). Event không khai báo không dùng được | Tenant chỉ thấy event ổn định, có schema; đổi tên event nội bộ không làm vỡ luật. Giống `WebhookEventCatalog` |
| A10 | **Thay đổi bản ghi qua `RecordAdapter` + `@AutomationRecord`.** App khai báo loại bản ghi tự động hoá được (trường, nhãn, kiểu, trường ghi được). Starter có listener Hibernate tính trường thay đổi cho entity có `@AutomationRecord`; app không dùng JPA thì gọi `RecordChanges.publish(...)` | Học `trigger_field_ids` của Odoo (chỉ chạy khi trường được theo dõi đổi). Không cho luật đọc/ghi bảng tuỳ ý |
| A11 | **Lịch theo ngày làm việc dùng `calendar`** (`CalendarSchedules`, `ScheduleFired`). Không có calendar thì dùng cron 5 trường (`CronExpression` của Spring) theo múi giờ tenant, chạy bằng `@ClusterScheduled` có watermark | "Ngày làm việc đầu tiên của tháng, Tết thì dời" là nhu cầu thật (calendar K9). Không tự viết lịch thứ hai |
| A12 | **Chạy thử không có tác dụng phụ.** `simulate` đánh giá điều kiện, dựng tham số và gọi `AutomationAction.preview` (mặc định chỉ mô tả). Chạy thử được trên payload mẫu, trên 20 event/bản ghi gần nhất (khi có log) hoặc trên một bản ghi chọn tay | Tenant thấy luật sẽ làm gì trước khi bật; n8n và Zapier đều có chế độ thử |
| A13 | **Log mọi lần chạy khớp điều kiện; lần không khớp chỉ đếm.** Bảng `aut_run` partition theo tháng; lần điều kiện sai chỉ tăng bộ đếm ngày, trừ khi bật "ghi chi tiết" cho luật trong tối đa 24 giờ. Bản chụp dữ liệu bị xoá sau 7 ngày, run giữ 30 ngày (lỗi 90 ngày) | Luật "khi bản ghi được sửa" có thể khớp trigger hàng triệu lần/ngày nhưng khớp điều kiện rất ít. Giảm lưu PII |
| A14 | **Giới hạn thương mại qua `features`:** `module.automation`, `admin.automation`, giới hạn `automation.rules` (số luật đang bật) và hạn mức `automation.runs` (lần chạy/tháng, `consume`). Vượt hạn mức thì run `THROTTLED`, không mất event (chờ chu kỳ sau hoặc admin chạy lại) | Bán theo gói như Zapier (task) và Jira (lần chạy theo gói); `licensing` kẹp trần (licensing mục 3.10) |
| A15 | **Mặc định nhúng** (`mode: embedded`), vì `record.update` và trigger thay đổi bản ghi cần chạy trong app. Chạy được ở `platform-services`: event đến qua Kafka, ghi bản ghi qua endpoint nội bộ có ký của app | Giống collaboration C14 |
| A16 | **Hành động ra ngoài chỉ qua module có kiểm soát.** Không có hành động "gọi URL tuỳ ý": gửi ra ngoài đi qua `webhook.emit` (module webhooks: ký, retry, chống SSRF) hoặc hành động do app khai báo endpoint cố định | SSRF và rò dữ liệu là rủi ro lớn nhất của automation do người dùng cấu hình |

---

## 1. Bài toán và phạm vi

### 1.1 Bài toán

Khách hàng doanh nghiệp liên tục xin những thay đổi nhỏ kiểu "khi … thì …":

- "đơn hàng đã giao và trên 10 triệu thì gửi ZNS cảm ơn, tạo việc gọi lại sau 3 ngày làm việc";
- "hồ sơ ứng tuyển nộp thì báo phòng nhân sự, chia đều cho ba chuyên viên";
- "hợp đồng còn 30 ngày hết hạn thì nhắc người phụ trách, còn 7 ngày thì nhắc cả trưởng phòng";
- "ngày làm việc đầu tiên của tháng lúc 8h30 gửi báo cáo doanh thu tháng trước cho ban giám đốc".

Mỗi yêu cầu là một lần dev sửa code, review, deploy. Odoo (Automated Actions), Frappe (Notification, Assignment Rule),
Directus (Flows), Jira (Automation) cho người quản trị tự làm. Vyforge chưa có (gap analysis bảng so sánh: ✗).

Module automation giải bài toán bằng một lớp mỏng: đọc event và thay đổi bản ghi đã có, đánh giá điều kiện có sandbox,
gọi hành động do các module khác cung cấp. Automation **không tự gửi tin, không tự lưu việc, không tự gọi HTTP**: nó điều
phối `notification`, `collaboration`, `webhooks`, `workflow`, `export` và các hành động của app.

### 1.2 Phạm vi và ranh giới

| Trong phạm vi | Ngoài phạm vi (thuộc module khác hoặc không làm) |
|---|---|
| Trigger: event công khai, bản ghi tạo/sửa/xoá/đổi trường, lịch (ngày làm việc), mốc tương đối theo trường ngày, thủ công, webhook gọi vào | Quy trình nhiều bước có người duyệt, trạng thái chờ, SLA, escalation (**workflow**) |
| Điều kiện CEL, trình dựng điều kiện trực quan | Script tuỳ ý (Python, JS, Groovy) do tenant viết |
| Hành động qua SPI; bộ hành động có sẵn khi module tương ứng được cài | Logic gửi tin, template, kênh (**notification**); ký, retry webhook (**webhooks**); lưu việc (**collaboration**) |
| Giao việc tự động theo luật (vòng tròn, ít việc nhất, theo trường): luật nằm ở đây, việc nằm ở collaboration | Hộp việc, nhắc việc (**collaboration**, **calendar**) |
| Chống vòng lặp, giới hạn tần suất, hạn mức theo gói | Rate limit hạ tầng chung (**-ratelimit** được dùng lại) |
| Version, bật/tắt, chạy thử, log chạy, chạy lại | Kiểm tra dữ liệu đồng bộ chặn giao dịch (hook trong code, `SubmissionInterceptor` của forms) |
| Mẫu luật do app giao sẵn, tenant cài và sửa | Connector tới hàng trăm SaaS kiểu Zapier/n8n (đi qua `webhooks` + `api-access`) |
| Admin MFE cho tenant admin và người vận hành | Chợ hành động của bên thứ ba |

**Ranh giới với workflow, kiểm lại theo thiết kế workflow đã merge:**

| Câu hỏi | Automation | Workflow |
|---|---|---|
| Có người ra quyết định giữa chừng? | Không | Có (bước duyệt) |
| Có trạng thái chờ (ngày, event, người)? | Không. Luật chạy xong trong một lần | Có (`wait`, SLA, yêu cầu bổ sung) |
| Lặp lại theo thời gian? | Trigger lịch, mỗi lần là một run độc lập | Nhắc việc/escalation trong một hồ sơ |
| Ai nối ai? | Automation gọi `Workflow.start` (hành động `workflow.start`); nghe `WorkflowCompleted` làm trigger | Workflow không phụ thuộc automation (workflow mục 6.3) |
| Ngôn ngữ điều kiện | CEL qua `platform-expression` | CEL (W6), cùng thư viện |

Khi một luật cần "chờ 3 ngày rồi kiểm lại", có hai cách đúng: (a) trigger mốc tương đối (`createdAt + 3 ngày làm việc`,
điều kiện xét lúc đến hạn); (b) hành động tạo việc có hạn. Nếu cần chuỗi chờ nhiều bước, đó là workflow.

### 1.3 Kịch bản nghiệp vụ phải đáp ứng

| # | Kịch bản | Đòi hỏi | Quyết định liên quan |
|---|---|---|---|
| K1 | **TMĐT:** đơn chuyển sang "Đã giao" và tổng tiền > 10.000.000 ₫ → gửi ZNS cảm ơn, tạo việc "Gọi chăm sóc" cho nhân viên phụ trách, hạn 3 ngày làm việc | Trigger đổi trường `status` với điều kiện trên `old`/`new`; hai hành động; hạn theo ngày làm việc | A1, A10, `todo.create` + calendar |
| K2 | **Tuyển dụng:** form ứng tuyển được nộp → email cho phòng nhân sự; giao việc sàng lọc vòng tròn cho 3 chuyên viên | Trigger `forms.submission.created` lọc theo `formKey`; giao việc vòng tròn có trạng thái | A9, `todo.create` (`ROUND_ROBIN`) |
| K3 | **Hợp đồng:** còn 30 ngày đến `endDate` → nhắc người phụ trách; còn 7 ngày → nhắc cả trưởng phòng; hợp đồng đã gia hạn thì không nhắc | Mốc tương đối theo trường ngày; điều kiện xét lại lúc đến hạn; đổi `endDate` thì dời mốc | A5, trigger `RECORD_DATE` |
| K4 | **Báo cáo:** ngày làm việc đầu tiên của tháng 08:30 (Tết thì ngày làm việc kế tiếp) → chạy export doanh thu tháng trước, gửi link cho ban giám đốc | Lịch theo `calendar`; hành động `export.start` + thông báo khi xong | A11 |
| K5 | **Công nợ:** hoá đơn quá hạn 7 ngày chưa thanh toán → ZNS nhắc khách, gắn nhãn "Nợ quá hạn", báo kế toán. Gắn nhãn không làm luật tự kích hoạt lại | Mốc tương đối + `record.update`; chống vòng lặp | A5, A6 |
| K6 | **CSKH:** ticket ưu tiên cao chưa phản hồi sau 4 giờ làm việc → chuyển cho trưởng nhóm, báo qua kênh chat | Mốc theo giờ làm việc; `record.assign` | A5, calendar `addBusinessDuration` |
| K7 | **Bán hàng:** lead mới từ website → giao theo tỉnh/thành (Hà Nội → nhóm miền Bắc, TP.HCM → nhóm miền Nam), trong nhóm chia cho người ít việc nhất | Điều kiện theo trường; chiến lược `LEAST_OPEN` của collaboration | `record.assign` |
| K8 | **Tài chính:** đề nghị thanh toán > 50 triệu được tạo → khởi động quy trình duyệt cấp giám đốc; ≤ 50 triệu → tự đánh dấu "Chờ kế toán" | Hai luật hoặc một luật có hai nhánh qua điều kiện; `workflow.start` khi có workflow | A1, mục 6.3 |
| K9 | **Đồng bộ ERP:** `WorkflowCompleted` với `APPROVED` → cập nhật trạng thái phiếu, phát webhook `automation.rule` tới endpoint MISA/ERP của khách | Trigger từ event module khác; `webhook.emit` có ký | A16 |
| K10 | **Sinh nhật khách hàng:** 08:00 đúng ngày sinh nhật (bỏ năm) → gửi ZNS voucher, chỉ khách đã đồng ý nhận tin quảng cáo | Mốc "kỷ niệm" theo trường ngày; hàm `hasConsent` (privacy-dsr); quảng cáo tuân theo giờ gửi của notification | §8.2 |
| K11 | **Kho:** tồn kho dưới ngưỡng tối thiểu → báo thủ kho, phát webhook cho nhà cung cấp; không báo lại trong 24 giờ cho cùng mặt hàng | Chặn lặp theo khoá (`cooldown` theo `subject`) | A6, `cooldown` |
| K12 | **Thanh toán:** `PaymentSucceeded` (IPN VNPay/MoMo) → đổi đơn sang "Đã thanh toán", gửi email biên nhận | Event của payment; idempotent khi IPN lặp | A2, idempotency |
| K13 | **eKYC:** xác minh thất bại lần thứ 3 trong ngày → báo bộ phận rủi ro, gắn cờ hồ sơ | Điều kiện trên trường đếm của event; hành động thông báo nhóm | A9 |
| K14 | **Duyệt bài CMS:** bài viết chuyển sang "Chờ duyệt" → báo biên tập viên trực ca (theo lịch trực) | Trigger bản ghi của cms; người nhận theo nhóm | — |
| K15 | **Tenant sửa luật mùa cao điểm:** trước khi bật, chạy thử trên 20 đơn gần nhất để xem bao nhiêu đơn khớp và tin nhắn sẽ ra sao | Chạy thử trên dữ liệu thật, không gửi | A12 |
| K16 | **Vòng lặp do cấu hình sai:** luật A "đơn sửa → đặt `updatedNote`", luật B "`updatedNote` đổi → sửa đơn" → bị chặn ở độ sâu 5, luật bị tạm dừng, tenant admin được báo, có log chuỗi | A6 |
| K17 | **Người publish nghỉ việc:** tài khoản bị khoá → luật của người đó tự `SUSPENDED`, tenant admin nhận việc "chuyển chủ luật" | A8 |
| K18 | **Gói dịch vụ:** gói Cơ bản 10 luật, 5.000 lần chạy/tháng; vượt thì run `THROTTLED`, tenant thấy cảnh báo và nút nâng gói | A14 |
| K19 | **Nhắc ra ngoài hệ thống:** hệ thống chấm công bên thứ ba gọi webhook vào (`POST /hooks/automation/<token>`) khi nhân viên đi muộn → gửi thông báo cho quản lý trực tiếp | Trigger `INBOUND` có HMAC, rate limit, schema payload do tenant khai | §3.3, §8 |
| K20 | **Người vận hành:** provider SMS phục hồi (`TaskQueueResumed`) → luật cấp vận hành gửi báo cáo tới kênh trực | Luật ở tenant vận hành; event của ops-console | ops-console mục 7 |

### 1.4 Để bản sau (mục 9.4)

- Nhánh rẽ trong một luật (`if/else` giữa các hành động). Bản đầu: mỗi hành động có `when` (CEL) riêng, đủ cho K8.
- Vòng lặp `for each` trên danh sách (vd. mỗi dòng của đơn hàng).
- Hành động AI (tóm tắt, phân loại) qua `ai` use case.
- Trigger theo kết quả truy vấn định kỳ ("mỗi sáng, mọi hoá đơn quá hạn"): bản đầu dùng mốc tương đối theo bản ghi.
- Kho mẫu luật dùng chung giữa các tenant (marketplace).

---

## 2. Nghiên cứu

### 2.1 Hệ thống tham khảo

| Hệ thống | Phiên bản, license | Mô hình dữ liệu | Trigger | Điều kiện | Hành động | Chống vòng lặp, giới hạn | Điểm mạnh | Điểm yếu |
|---|---|---|---|---|---|---|---|---|
| **Odoo Automation Rules** (`base_automation`) ([tài liệu 19.0](https://www.odoo.com/documentation/19.0/applications/studio/automated_actions.html), [mã nguồn](https://github.com/odoo/odoo/blob/19.0/addons/base_automation/models/base_automation.py)) | 19.0, LGPL-3.0 (Community) | `base.automation`: `model_id`, `trigger`, `trigger_field_ids` (trường theo dõi), `filter_pre_domain` (trước khi sửa), `filter_domain` (sau khi sửa), `trg_date_id` + `trg_date_range` + `trg_date_range_type` + `trg_date_calendar_id` (mốc theo trường ngày, có lịch làm việc), `last_run`, `action_server_ids` | `on_create`, `on_create_or_write`, `on_stage_set`, `on_state_set`, `on_tag_set`, `on_user_set`, `on_priority_set`, `on_archive`, `on_unarchive`, `on_unlink`, `on_change` (UI), `on_time`/`on_time_created`/`on_time_updated`, `on_message_received/sent`, `on_webhook` (đọc mã nguồn 19.0) | Domain Odoo (`safe_eval`) trước và sau khi sửa | Server action: cập nhật bản ghi, tạo/nhân bản bản ghi, tạo activity, gửi email/SMS/WhatsApp, thêm/bớt follower, chạy code Python, gửi webhook, nhiều hành động | Ngữ cảnh `__action_done`: một luật chạy tối đa một lần trên một bản ghi trong một lượt xử lý (`_process`). Trigger theo thời gian chạy bằng cron mặc định 4 giờ/lần; tài liệu cảnh báo luật mới có thể áp dụng ngược cho bản ghi cũ | Trigger theo trường cụ thể; domain trước/sau sửa; mốc thời gian theo lịch làm việc; nhiều loại hành động | Chạy đồng bộ trong transaction: lỗi hành động làm hỏng thao tác của người dùng; "Execute Code" là Python (rủi ro); không có log chạy, không chạy thử, không version |
| **Directus Flows** ([tài liệu](https://docs.directus.io/app/flows/triggers), [mã nguồn `flows.ts`](https://github.com/directus/directus/blob/main/api/src/flows.ts)) | v12.5.1 (23/10/2026). License: **Monospace Sustainable Core License 1.0** (MSCL-1.0-GPL, đọc file `license` ở `main`), không phải OSI | `directus_flows` (trigger, `options`, `accountability`), `directus_operations` (đồ thị: `resolve`/`reject` trỏ tới operation kế tiếp), log ở `directus_activity` + `directus_revisions` khi `accountability = 'all'` | `event` (loại `filter` chặn giao dịch hoặc `action` sau giao dịch), `schedule` (cron), `operation` (flow khác gọi), `webhook`, `manual` (nút trên bản ghi) | Operation "Condition" dùng filter rules của Directus; nhánh `resolve`/`reject` | Đọc/tạo/sửa/xoá item, gửi email, thông báo, gọi webhook/HTTP, chạy script (isolated sandbox: `FLOWS_RUN_SCRIPT_MAX_MEMORY` 32 MB, `FLOWS_RUN_SCRIPT_TIMEOUT` 10 s), gọi flow khác, transform | Thao tác item có tuỳ chọn `emitEvents` (tắt thì không kích hoạt flow khác); biến môi trường chỉ lộ theo `FLOWS_ENV_ALLOW_LIST` | Đồ thị trực quan; data chain (`$trigger`, `$last`, kết quả từng bước); flow thủ công có kiểm quyền; filter hook chặn được giao dịch | Filter hook do người dùng cấu hình chặn được giao dịch (dễ gây lỗi khó hiểu); HTTP tuỳ ý (SSRF); chạy script JS; license không phải OSI |
| **Frappe Notification + Assignment Rule** ([Notification](https://docs.frappe.io/erpnext/user/manual/en/notifications), [Assignment Rule](https://docs.frappe.io/erpnext/user/manual/en/assignment-rule), [mã nguồn](https://github.com/frappe/frappe/blob/develop/frappe/email/doctype/notification/notification.py)) | Frappe v16.53.0 (22/10/2026), MIT | Notification: `document_type`, `event`, `condition`, `value_changed`, `days_in_advance`, `date_changed`, kênh, người nhận, `set_property_after_alert`. Assignment Rule: `document_type`, `priority`, điều kiện gán/đóng/bỏ gán, `rule` (Round Robin, Load Balancing, Based on Field), `users`, ngày hạn theo trường | New, Save, Submit, Cancel, Days Before/After (trường ngày), Value Change, Method | Biểu thức Python qua `frappe.safe_eval` (mã nguồn dòng 126, 315) | Gửi qua email, Slack, thông báo hệ thống, SMS, WhatsApp; đặt giá trị trường sau khi gửi; giao việc | `set_property_after_alert` là cách người dùng tự chặn gửi lặp (đặt cờ rồi đưa vào điều kiện) | Assignment Rule rất sát nhu cầu VN (vòng tròn, ít việc nhất); nhắc theo ngày trước/sau | Hai cơ chế tách rời; điều kiện là Python; chống lặp phải tự làm bằng cờ |
| **Atlassian Automation** (Jira Cloud) ([giới hạn dịch vụ](https://support.atlassian.com/cloud-automation/docs/automation-service-limits/)) | SaaS, thương mại | Rule = trigger + điều kiện + hành động + nhánh; audit log từng lần chạy | Event Jira, lịch, thủ công, webhook gọi vào | Điều kiện so sánh trường, JQL, smart values | Rất nhiều | **Loop detection 10**: flow kích hoạt chính nó hoặc flow khác liên tiếp quá 10 lần thì dừng, trạng thái `LOOP`; đồng thời 5–30 theo gói; 65 thành phần/flow; 60 phút xử lý/12 giờ; 50.000 việc trong hàng đợi toàn cục; vượt thì `THROTTLED` trong audit log; có trigger "Service limit breached" | Giới hạn rõ, công khai, có trạng thái riêng cho vòng lặp và bị bóp | Giới hạn theo gói phức tạp; không tự host |
| **Zapier** ([Zap history](https://help.zapier.com/hc/en-us/articles/8496291148685)) | SaaS, thương mại | Zap = trigger + các bước; mỗi run gắn version | Hàng nghìn app | Bước Filter, Paths | Hàng nghìn app | Tính phí theo task; replay trong 60 ngày; Filter/Paths không replay | Lịch sử run có Data In/Out từng bước, version của run, replay | Không tự host; dữ liệu khách đi qua bên thứ ba |
| **n8n** ([biến cấu hình execution](https://docs.n8n.io/hosting/configuration/environment-variables/executions/)) | 2.44.7 (26/10/2026), **Sustainable Use License** (+ phần `.ee` thương mại), không phải OSI | Workflow là đồ thị node; execution lưu dữ liệu từng node | Webhook, cron, event app, thủ công | Node IF/Switch, biểu thức JS | Hàng trăm node | `EXECUTIONS_TIMEOUT` (mặc định tắt), `EXECUTIONS_TIMEOUT_MAX` 3.600 s; lưu execution theo trạng thái (`EXECUTIONS_DATA_SAVE_ON_*`), xoá sau 336 giờ hoặc 10.000 bản (`EXECUTIONS_DATA_PRUNE*`) | Chính sách lưu log theo trạng thái, xoá tự động; chế độ chạy thử từng node | License không cho nhúng bán lại; biểu thức JS |
| **Google CEL** ([cel-java](https://github.com/google/cel-java), [Maven](https://repo1.maven.org/maven2/dev/cel/cel/)) | `dev.cel:cel` 0.14.0 (17/08/2026), Apache-2.0, Java 8+ | Biểu thức biên dịch thành AST đã kiểm kiểu, chạy bằng `CelRuntime` | — | Không có tác dụng phụ, luôn dừng; `CelOptions`: `maxExpressionCodePointSize`, `maxParseRecursionDepth`, `maxParseExpressionNodeCount`, `comprehensionMaxIterations`, `maxRegexProgramSize`; `CelEvaluationListener` để đếm bước; kiểu struct tuỳ biến qua `StructType.create(name, fields, FieldResolver)` (đọc mã nguồn `main`) | — | — | Có kiểm kiểu lúc biên dịch, dùng ở Kubernetes, Envoy, Firebase; workflow đã chọn | Người dùng nghiệp vụ không viết CEL được: cần trình dựng |
| **JSONLogic** ([json-logic-java](https://github.com/jamsesso/json-logic-java)) | 1.1.0 (02/05/2025), MIT, đẩy code lần cuối 07/09/2025 | Cây JSON | — | Toán tử `==`, `and`, `in`, `var`… | — | Không kiểm kiểu; không giới hạn chi phí sẵn | Dễ sinh từ UI | Ngôn ngữ thứ hai bên cạnh CEL; ít bảo trì |

### 2.2 Ghi chú từ mã nguồn

- **Odoo chống đệ quy theo bản ghi, không theo độ sâu.** `_process` lấy `__action_done` trong context, bỏ các bản ghi luật
  đã xử lý, đánh dấu bản ghi còn lại trước khi chạy hành động. Hành động sửa bản ghi đó không làm chính luật chạy lại,
  nhưng luật B vẫn chạy được; vòng A→B→A trên cùng bản ghi dừng vì A đã đánh dấu. Ta học cách đánh dấu `ruleId:subject`
  trong chuỗi (A6.2), nhưng phải mang qua ranh giới bất đồng bộ.
- **Odoo đánh giá điều kiện trước và sau khi sửa** (`filter_pre_domain`, `filter_domain`). "Chuyển sang Đã giao" khác
  "đang ở Đã giao": ta có biến `old` và `new` và hàm `changed('status')`.
- **Directus tách `filter` (chặn) và `action` (sau giao dịch).** Flow filter kết thúc ở nhánh reject thì huỷ giao dịch.
  Ta không làm loại chặn (A2).
- **Directus ghi log theo `accountability`** (`null` = không ghi, `activity`, `all` = kèm revision). Ta học: log chi tiết
  bật/tắt được theo luật (A13).
- **Frappe `set_property_after_alert`** là dấu hiệu người dùng cần chống gửi lặp. Ta có sẵn `cooldown` theo luật + bản ghi
  (K11), không bắt tenant thêm trường cờ.
- **Jira** gắn trạng thái riêng `LOOP`, `THROTTLED` vào audit log và có trigger khi chạm giới hạn: ta có trạng thái run
  `LOOP`, `THROTTLED` và event `AutomationRuleSuspended`.

### 2.3 Học theo và không làm

**Học theo:**

1. Trigger theo **trường cụ thể** và điều kiện trên giá trị **trước/sau** (Odoo).
2. Mốc thời gian theo trường ngày **có lịch làm việc** (Odoo `trg_date_calendar_id`), nhưng không quét bằng cron 4 giờ/lần:
   ta đặt hẹn giờ chính xác khi bản ghi đổi (§3.6).
3. Giao việc vòng tròn, ít việc nhất, theo trường (Frappe Assignment Rule).
4. Trạng thái `LOOP`/`THROTTLED`, giới hạn công khai theo gói (Jira).
5. Log từng run có dữ liệu vào/ra từng bước, version của run, chạy lại (Zapier).
6. Lưu log theo trạng thái và tự xoá (n8n).
7. Trigger thủ công trên bản ghi, có kiểm quyền (Directus `manual`).

**Không làm:**

| Không làm | Lý do |
|---|---|
| Script tuỳ ý (Odoo Execute Code, Directus Run Script, n8n Code, Frappe Server Script) | Thực thi mã do tenant viết trong JVM dùng chung; sandbox JS/Python khó đảm bảo. CEL đủ cho điều kiện và tính giá trị |
| Luật chặn giao dịch (Directus filter, Odoo `on_change`) | Lỗi cấu hình của tenant làm hỏng nghiệp vụ; logic chặn thuộc code của app |
| Gọi HTTP tới URL tuỳ ý | SSRF, rò dữ liệu; đi qua `webhooks` (A16) |
| Đồ thị tuỳ ý nhiều nhánh, vòng lặp (Directus, n8n) | Thành engine quy trình thứ hai; workflow đã có |
| Nhúng n8n hoặc Directus | License SUL/MSCL không phải OSI, không theo tenant/IAM của vyforge (gap analysis mục 4) |
| Quét bản ghi định kỳ để tìm bản ghi đến hạn (cron Odoo) | Chậm (4 giờ/lần), tốn tải; dùng hẹn giờ theo từng bản ghi |
| JSONLogic song song với CEL | Hai ngôn ngữ, hai sandbox (A3) |

---

## 3. Kiến trúc

### 3.1 Sơ đồ thành phần

```mermaid
flowchart LR
  subgraph APP["App nghiệp vụ (hoặc platform-services)"]
    BIZ["Service nghiệp vụ<br/>@Transactional"]
    ENT["Entity @AutomationRecord"]
    EVT["Event record<br/>@AutomationEvent · TenantAwareEvent"]
  end

  subgraph AUT["platform-starter-automation"]
    direction TB
    SRC["Nguồn trigger<br/>EventTriggerListener (BEFORE_COMMIT)<br/>RecordChangeListener (Hibernate)<br/>ScheduleTriggerSource · DateTriggerScheduler<br/>ManualTrigger · InboundWebhookController"]
    IDX["RuleIndex<br/>(tenant, triggerType, key) → luật đang bật<br/>cache, xoá khi publish"]
    MATCH["TriggerMatcher<br/>lọc trường theo dõi · chain guard<br/>tạo aut_run + enqueue"]
    Q[("DurableTaskQueue<br/>queue automation")]
    RUN["RuleRunner<br/>kiểm hạn mức · CEL điều kiện<br/>dựng tham số · chạy hành động"]
    ACT["ActionRegistry<br/>AutomationAction (SPI)"]
    DEF["RuleService<br/>bản nháp · publish · compile · simulate"]
    CAT["TriggerCatalog<br/>event + loại bản ghi + JSON Schema"]
    GUARD["LoopGuard · RateGuard · QuotaGuard"]
    LOG[("aut_run · aut_run_step<br/>aut_rule_stat")]
  end

  EXP["platform-expression<br/>CEL env, giới hạn, hàm"]

  subgraph MODS["Module cung cấp hành động (tuỳ chọn)"]
    NOTI["notification"]
    WH["webhooks"]
    COL["collaboration"]
    WF["workflow"]
    EXPO["export"]
    APPACT["Hành động của app"]
  end

  CALM["calendar (tuỳ chọn)"]
  FEAT["features / licensing (tuỳ chọn)"]

  BIZ --> EVT --> SRC
  BIZ --> ENT --> SRC
  CALM -- "ScheduleFired" --> SRC
  SRC --> IDX --> MATCH --> Q --> RUN
  RUN --> GUARD
  RUN --> EXP
  DEF --> EXP
  DEF --> CAT
  RUN --> ACT
  ACT --> NOTI & WH & COL & WF & EXPO & APPACT
  RUN --> LOG
  GUARD --> FEAT
```

### 3.2 Gói và lớp

```
com.vyforge.platform.automation             API công khai: AutomationRules, AutomationRuns, AutomationTriggers,
                                            RecordChanges, AutomationChain, record, event, mã lỗi AutomationErrors,
                                            AutomationActions (action catalog IAM)
com.vyforge.platform.automation.spi         AutomationAction, ActionContext, ActionResult, RecordAdapter,
                                            AutomationTriggerCatalog, AssigneeStrategy, AutomationInterceptor
com.vyforge.platform.automation.annotation  @AutomationEvent, @AutomationRecord, @AutomationField
com.vyforge.platform.automation.port        AutomationScheduler (calendar), AutomationEntitlements (features),
                                            PrincipalDirectory (IAM)
com.vyforge.platform.automation.internal    RuleIndex, TriggerMatcher, RuleRunner, RuleCompiler, LoopGuard,
                                            RateGuard, QuotaGuard, RunRecorder, DateTriggerScheduler, …
com.vyforge.platform.automation.integration.<module>   Hành động và adapter cho notification, webhooks,
                                            collaboration, workflow, export, calendar, features, privacy, custom-fields
com.vyforge.platform.automation.web         REST: /api/v1/automation/** (admin, manual, inbound, internal)

com.vyforge.platform.expression (platform-expression)
                                            Expressions (biên dịch/đánh giá), ExpressionEnvironment, ExpressionLimits,
                                            ExpressionFunctions (SPI), SchemaTypes (JSON Schema → kiểu CEL), Template
                                            (chuỗi có {{ }}), CompiledExpression
```

ArchUnit: chỉ `..expression..` được import `dev.cel..`; `..integration.<module>..` là nơi duy nhất import package của
module tương ứng; mọi lớp `integration` có `@ConditionalOnClass` + `@ConditionalOnBean`.

### 3.3 Khái niệm

| Khái niệm | Nghĩa |
|---|---|
| **Luật** (`aut_rule`) | Định danh ổn định (`key`, duy nhất theo tenant), trạng thái, version đang chạy, người publish |
| **Version luật** (`aut_rule_version`) | Định nghĩa bất biến sau publish: trigger, điều kiện, hành động, tuỳ chọn |
| **Trigger** | Một trong 6 loại (bảng dưới). Mỗi loại có `key` để tra chỉ mục |
| **Điều kiện** | Biểu thức CEL trả `bool`; rỗng = luôn đúng |
| **Hành động** | Một mục trong danh sách: `type` (của `AutomationAction`), `params`, `when` (CEL tuỳ chọn), `onError` (`STOP` mặc định, `CONTINUE`) |
| **Run** (`aut_run`) | Một lần luật được kích hoạt và điều kiện đúng (hoặc bị chặn: `LOOP`, `THROTTLED`) |
| **Chuỗi** (`AutomationChain`) | Nhân quả giữa các run: run do hành động của run khác gây ra thì cùng chuỗi, độ sâu + 1 |
| **Subject** | Bản ghi chính của run, dạng `EntityRef` (`crm:order/01J…`), dùng cho chống lặp, cooldown, log theo bản ghi |

**Loại trigger:**

| Loại | `key` | Nguồn | Dữ liệu cho điều kiện |
|---|---|---|---|
| `EVENT` | Tên event trong catalog: `forms.submission.created`, `workflow.completed`, `payment.succeeded` | Event Spring có `@AutomationEvent` (cùng app) hoặc Kafka (ở `platform-services`) | `event` (kiểu theo schema), `trigger`, `actor` |
| `RECORD` | Loại bản ghi + thao tác: `crm:order` + `CREATED`/`UPDATED`/`DELETED`/`CREATED_OR_UPDATED`; tuỳ chọn `fields[]` (chỉ kích hoạt khi một trong các trường đổi) | `@AutomationRecord` (Hibernate) hoặc `RecordChanges.publish` | `record` (sau), `old` (trước, null khi tạo), `changed` (danh sách trường), `actor` |
| `SCHEDULE` | `ScheduleSpec` của calendar (`nthBusinessDay`, `everyBusinessDay`, `rrule`) hoặc cron 5 trường + múi giờ | calendar `ScheduleFired` (owner `automation`) hoặc bộ tick dự phòng | `trigger.scheduledAt`, `now` |
| `RECORD_DATE` | Loại bản ghi + trường ngày + độ lệch: `crm:contract.endDate`, `-30d` / `-7bd` / `+4bh` / `anniversary 08:00` | `DateTriggerScheduler` đặt hẹn giờ khi bản ghi tạo/đổi trường | `record` đọc lại lúc đến hạn qua `RecordAdapter.load` |
| `MANUAL` | Loại bản ghi (hoặc không) | Nút "Chạy luật" (`POST /rules/{key}/execute`) | `record`, `input` (form nhập do luật khai, JSON Schema) |
| `INBOUND` | Token endpoint | `POST /api/v1/automation/hooks/{token}` có HMAC | `payload` (kiểu theo JSON Schema tenant khai), `headers` (danh sách cho phép) |

### 3.4 Định nghĩa luật (DSL)

Luật lưu dạng JSON có JSON Schema (`META-INF/vyforge/automation/schema/rule-v1.json`), soạn bằng trình dựng hoặc
YAML (nhập/xuất). Ví dụ K1:

```yaml
# Mẫu do app giao: META-INF/vyforge/automation/templates/order-delivered-thanks.yml
schemaVersion: 1
key: order-delivered-thanks
name: { vi: "Cảm ơn khách sau khi giao đơn lớn", en: "Thank customer after large delivery" }
trigger:
  type: RECORD
  recordType: crm:order
  on: UPDATED
  fields: [status]
condition: >-
  changed('status') && record.status == 'DELIVERED' && old.status != 'DELIVERED'
  && record.total.amount > 10000000
options:
  cooldown: { per: subject, duration: P1D }        # tuỳ chọn; bỏ qua nếu cùng đơn đã chạy trong 1 ngày
  maxRunsPerMinute: 60                              # ≤ trần của tenant
actions:
  - id: thank
    type: notification.send
    params:
      type: crm.order.thank-you                     # type do app khai và cho phép automation dùng
      to: [{ kind: field, path: record.customer }]  # người nhận lấy từ trường bản ghi
      data:
        orderCode: "{{ record.code }}"
        total: { $expr: "record.total" }
  - id: callback
    type: todo.create
    params:
      subject: { $expr: "record.ref" }
      todoType: call
      title: "Gọi chăm sóc khách {{ record.customerName }} (đơn {{ record.code }})"
      assignee: { strategy: FIELD, path: record.ownerSub }
      due: { businessDays: 3 }
    onError: CONTINUE
```

**Biến trong điều kiện và tham số:**

| Biến | Kiểu | Có ở trigger |
|---|---|---|
| `event` | struct theo JSON Schema của event | `EVENT` |
| `record`, `old` | struct theo khai báo `@AutomationRecord` / `RecordTypeDefinition`; `old` có thể `null` | `RECORD`, `RECORD_DATE` (`old` không có), `MANUAL` |
| `changed` | `list(string)` | `RECORD` |
| `payload`, `headers` | struct theo schema tenant khai; `map(string,string)` | `INBOUND` |
| `input` | struct theo form của luật | `MANUAL` |
| `trigger` | `{type, key, occurredAt, scheduledAt?}` | mọi loại |
| `actor` | `{sub, groups, isSystem}`; `isSystem` = do automation/hệ thống gây ra | `EVENT`, `RECORD`, `MANUAL` |
| `tenant` | `{id, timezone, locale}` | mọi loại |
| `steps` | kết quả các hành động trước (`steps.thank.notificationId`) | chỉ trong `params`/`when` của hành động |
| `now` | `timestamp` | mọi loại |

**Hàm chuẩn** (`platform-expression`, mục 6.1): `changed(field)`, `money(amount, 'VND')`, `fold(text)` (bỏ dấu tiếng
Việt, theo `VietnameseText`), `daysBetween(a, b)`, `businessDaysBetween(a, b)` (khi có calendar), `inGroup(sub, group)`,
`hasConsent(subjectRef, purpose)` (khi có privacy-dsr), `date(ts, tz)`, `format(value, pattern)` và extension chuẩn của
cel-java (`strings`, `math`, `lists`). Không có hàm truy cập mạng, file, đồng hồ hệ thống ngoài `now` cố định của run.

### 3.5 Luồng chính: event → run

```mermaid
sequenceDiagram
  autonumber
  participant S as Service nghiệp vụ (tx)
  participant L as EventTriggerListener (BEFORE_COMMIT)
  participant I as RuleIndex
  participant M as TriggerMatcher
  participant Q as DurableTaskQueue
  participant R as RuleRunner (worker)
  participant G as Guards
  participant E as CEL (platform-expression)
  participant A as AutomationAction
  participant DB as aut_run / aut_run_step

  S->>S: publishEvent(PaymentSucceeded) [@AutomationEvent("payment.succeeded")]
  S->>L: trước commit
  L->>I: rules(tenant, EVENT, "payment.succeeded")
  I-->>L: [rule-12 v3] (cache theo tenant)
  L->>M: match(rule, payload, chain hiện tại)
  M->>M: chain guard: depth+1 ≤ max? ruleId:subject chưa có trong chuỗi?
  M->>DB: insert aut_run(QUEUED, snapshot đã che, chainId, depth)
  M->>Q: enqueue(automation/run, {runId}, idempotency = rule:eventId)
  S->>S: commit (run và task cùng commit với nghiệp vụ)
  Q->>R: handle(runId) trong TenantContext.run
  R->>G: RateGuard (luật, tenant) · QuotaGuard (automation.runs)
  G-->>R: OK
  R->>E: evaluate(condition, snapshot) ≤ 10.000 bước
  E-->>R: true
  loop từng hành động
    R->>E: when? dựng params ({{ }}, $expr)
    R->>A: execute(ctx) với AutomationChain(depth+1) gắn vào ngữ cảnh
    A-->>R: Completed(outputs) / Failed(code, retryable)
    R->>DB: aut_run_step
  end
  R->>DB: aut_run SUCCEEDED | PARTIAL | FAILED; aut_rule_stat++
```

Điều kiện sai: run chuyển `SKIPPED` và **bị xoá ngay trong cùng transaction** (chỉ tăng `aut_rule_stat.skipped`), trừ khi
luật đang bật "ghi chi tiết" (A13). Vì vậy điều kiện được đánh giá ở worker chứ không ở listener: listener không chạy
CEL trong transaction nghiệp vụ, nên thao tác của người dùng không chậm thêm và không lỗi vì luật.

**Tối ưu khi khớp nhiều:** luật có trigger `RECORD` trên bảng ghi nhiều có thể tạo nhiều run rồi bị bỏ. Có hai lớp lọc
rẻ trước khi enqueue: (1) `fields[]` (chỉ khi trường theo dõi đổi), (2) `prefilter`: phần điều kiện mà trình biên dịch tách
được thành so sánh bằng trên trường trạng thái (`record.status == 'DELIVERED'`) được đánh giá ngay ở listener bằng so sánh
giá trị, không chạy CEL. Phần còn lại đánh giá ở worker.

### 3.6 Trigger theo mốc thời gian của bản ghi (`RECORD_DATE`)

```mermaid
sequenceDiagram
  autonumber
  participant S as Service (tx)
  participant RC as RecordChangeListener
  participant D as DateTriggerScheduler
  participant C as calendar (BusinessCalendar, tuỳ chọn)
  participant Q as DurableTaskQueue
  participant R as RuleRunner
  participant RA as RecordAdapter

  S->>RC: hợp đồng HD-001 lưu (endDate = 2026-12-31)
  RC->>D: onChange(crm:contract/HD-001, changed=[endDate])
  D->>D: luật RECORD_DATE trên crm:contract.endDate: rule-7 (-30d), rule-8 (-7bd)
  D->>C: addBusinessDays(2026-12-31, -7) [rule-8]
  D->>Q: enqueue(automation/date, runAt=2026-12-01 08:00, key=rule-7:HD-001:2026-12-31)
  D->>Q: enqueue(automation/date, runAt=2026-12-22 08:00, key=rule-8:HD-001:2026-12-31)
  D->>D: upsert aut_timer(rule, subject, fireAt, taskId); huỷ task cũ nếu endDate đổi
  Note over Q: … đến hạn …
  Q->>R: fire(rule-7, HD-001, anchor=2026-12-31)
  R->>RA: load(crm:contract/HD-001)
  RA-->>R: record hiện tại (đã gia hạn? endDate khác anchor?)
  R->>R: anchor lệch → bỏ (đã có hẹn mới); điều kiện CEL trên record hiện tại
```

- Giờ chạy mặc định 08:00 theo múi giờ tenant (đổi được: `at`), mốc theo ngày làm việc dùng `calendar`; không có
  calendar thì `bd` = ngày lịch và `admin/info` cảnh báo.
- Bản ghi xoá → `aut_timer` của bản ghi bị huỷ. Luật tắt hoặc version mới đổi trường/độ lệch → job nền tính lại hẹn giờ.
- **Bản ghi có sẵn khi luật mới bật:** không tự áp dụng ngược (học cảnh báo của Odoo). Trình dựng có tuỳ chọn "Lên lịch cho
  bản ghi hiện có" khi `RecordAdapter` hỗ trợ `scan(type, dateField, from, to)`; chạy bằng việc nền theo lô 1.000, có
  xem trước số lượng.
- `anniversary`: hẹn giờ cho lần kỷ niệm kế tiếp; chạy xong tự hẹn năm sau (29/02 → 28/02 năm không nhuận).

### 3.7 Chống vòng lặp

```mermaid
sequenceDiagram
  autonumber
  participant R1 as Run #1 (rule A, depth 1)
  participant APP as RecordAdapter.update
  participant RC as RecordChangeListener
  participant M as TriggerMatcher
  participant LG as LoopGuard

  R1->>APP: record.update(order-9, note="…") trong ScopedValue AutomationChain{c1, depth 1, {A:order-9}}
  APP->>RC: thay đổi bản ghi (cùng thread, cùng tx)
  RC->>M: rules(UPDATED crm:order) = [A, B]
  M->>LG: A trên order-9 đã có trong chuỗi c1 → bỏ (không ghi run)
  M->>LG: B: depth 2 ≤ 5 → run #2 {c1, depth 2, {A:order-9, B:order-9}}
  Note over M: … B sửa order-9 → A? đã có → bỏ. Vòng A↔B dừng ở lượt thứ hai
  Note over LG: Chuỗi dài A→B→C→D→E→F trên các bản ghi khác nhau: depth 6 > 5 → run LOOP,<br/>tăng aut_rule_stat.loop; 3 lần LOOP/10 phút → luật SUSPENDED, AutomationRuleSuspended
```

- `AutomationChain` mang qua: (1) cùng thread: `ScopedValue` (Java 25) đặt trong lúc hành động chạy; (2) qua
  `DurableTaskQueue`: `EnqueueOptions` hiện chỉ có `idempotencyKey`, `runAt`, `delay`, `priority`, `maxAttempts` (đọc code
  0.12.1), nên đề xuất thêm SPI nhỏ `TaskContextPropagator` ở `-scheduling` (chụp ngữ cảnh lúc `enqueue`, lưu vào cột
  `context jsonb` của `vyforge_task`, khôi phục quanh handler, giống cách `TenantContext.run` bọc việc). Automation cài
  propagator cho `AutomationChain`, nên việc nền của notification, webhooks, collaboration do hành động tạo ra vẫn mang
  chuỗi (Q3). Chưa có SPI này thì chuỗi đứt ở ranh giới task và lớp tần suất là chốt chặn; (3) qua Kafka: header
  `vyforge-automation-chain` (`<chainId>;<depth>;<ruleIds>`); (4) qua HTTP nội bộ: header cùng tên.
- Event do hệ thống gây ra mà không mang chuỗi (vd. notification gửi xong rồi phát `NotificationDelivered`) tạo chuỗi mới:
  lớp (3) của A6 (tần suất) chặn phần còn lại.
- **Giới hạn tần suất** dùng `-ratelimit` (bucket Redis hoặc bộ nhớ): mặc định 60 run/phút/luật, 600 run/phút/tenant, 20
  run/phút/luật/subject. Vượt → run `THROTTLED` (có ghi), task không retry; tenant thấy trong lịch sử. Người vận hành
  đổi mặc định; tenant chỉ giảm.
- **Cooldown** theo luật + subject (K11): khoá Redis/DB `aut_cooldown` hết hạn sau `duration`; run khớp trong cooldown
  bị bỏ và đếm `skipped_cooldown`.
- **Tự tạm dừng:** `LOOP` ≥ 3 hoặc `FAILED` liên tiếp ≥ 20 (cấu hình) trong 10 phút → `SUSPENDED`, phát
  `AutomationRuleSuspended`, gửi thông báo `automation.rule.suspended` cho tenant admin (khi có notification).

### 3.8 State machine

**Luật:**

```mermaid
stateDiagram-v2
  [*] --> DRAFT: tạo
  DRAFT --> ACTIVE: publish (biên dịch OK, quyền OK, trong giới hạn automation.rules)
  ACTIVE --> PAUSED: tắt
  PAUSED --> ACTIVE: bật (kiểm lại quyền, giới hạn)
  ACTIVE --> SUSPENDED: hệ thống (LOOP, lỗi liên tiếp, người publish mất quyền/bị khoá, hành động không còn)
  SUSPENDED --> ACTIVE: tenant admin bật lại (sau khi sửa hoặc nhận quyền sở hữu)
  ACTIVE --> ACTIVE: publish version mới
  PAUSED --> ARCHIVED: lưu trữ
  DRAFT --> ARCHIVED
  SUSPENDED --> ARCHIVED
  ARCHIVED --> [*]
```

Bản nháp tồn tại song song với version đang chạy (`draft_version` khác `published_version`).

**Run:**

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> RUNNING
  QUEUED --> LOOP: (ghi ngay ở listener)
  RUNNING --> SKIPPED: điều kiện sai (chỉ lưu khi ghi chi tiết)
  RUNNING --> THROTTLED: tần suất / hạn mức
  RUNNING --> SUCCEEDED: mọi hành động xong
  RUNNING --> PARTIAL: có hành động lỗi với onError=CONTINUE
  RUNNING --> RETRYING: hành động lỗi retryable
  RETRYING --> RUNNING: backoff (DurableTaskQueue)
  RETRYING --> FAILED: hết lượt
  RUNNING --> FAILED: lỗi không retry
  QUEUED --> CANCELLED: admin huỷ / luật bị lưu trữ
  FAILED --> QUEUED: admin chạy lại (cùng version hoặc version hiện tại)
  THROTTLED --> QUEUED: admin chạy lại
  SUCCEEDED --> [*]
  PARTIAL --> [*]
  SKIPPED --> [*]
  CANCELLED --> [*]
  LOOP --> [*]
```

**Retry an toàn:** hành động đã `COMPLETED` trong run không chạy lại khi run retry (tiếp tục từ bước lỗi). Mỗi hành động
nhận `idempotencyKey = automation:<runId>:<actionId>` và chuyển cho module (notification, webhooks, collaboration đều nhận
idempotency key), nên crash giữa chừng không gửi trùng.

### 3.9 Luồng lỗi

| Lỗi | Xử lý |
|---|---|
| Biểu thức sai kiểu, trường không tồn tại | Bắt lúc publish (`automation.condition_invalid` kèm vị trí). Không vào được trạng thái chạy |
| Schema event/bản ghi đổi sau khi publish (trường bị bỏ) | Lúc khởi động và khi catalog đổi: biên dịch lại mọi version đang chạy; lỗi → luật `SUSPENDED` (lý do `schema_incompatible`) và báo admin |
| CEL vượt giới hạn bước, lỗi chia 0, `null` | Run `FAILED`, `automation.expression_runtime_error`, không retry |
| Hành động lỗi tạm (provider, mạng, khoá lạc quan) | `Failed(retryable=true)` → retry theo backoff queue (mặc định 5 lượt, 30 s → 30 phút) |
| Hành động lỗi vĩnh viễn (403, 404 bản ghi, tham số sai) | `FAILED`; `onError: CONTINUE` thì `PARTIAL` |
| Module cung cấp hành động bị gỡ | Luật dùng hành động đó `SUSPENDED` lúc khởi động (`action_unavailable`) |
| Người publish mất quyền hoặc bị khoá | Kiểm lúc chạy (cache 5 phút); `SUSPENDED` + việc "chuyển chủ luật" cho tenant admin |
| Hết hạn mức tháng | `THROTTLED`; một thông báo/ngày cho tenant admin |
| Worker chết giữa run | `DurableTaskQueue` trả việc về khi `locked_until` hết; run tiếp tục từ bước chưa `COMPLETED` |
| `RecordAdapter.load` không thấy bản ghi (mốc thời gian) | Run `SKIPPED` lý do `subject_gone`, xoá `aut_timer` |
| Inbound sai chữ ký, quá tần suất, payload sai schema | 401/429/422; không tạo run; đếm metric |

### 3.10 Chạy thử (`simulate`)

`POST /admin/rules/{key}/simulate` với `version` (mặc định bản nháp) và một trong: `sample` (ví dụ của catalog hoặc JSON
tự nhập), `recentEvents: 20` (từ `aut_trigger_sample`, mục 4.3), `subject` (bản ghi chọn tay, đọc qua `RecordAdapter`).
Kết quả cho mỗi mẫu: điều kiện đúng/sai và giá trị từng vế (`CelEvaluationListener`), tham số đã dựng (đã che PII theo
quyền người xem), `preview` của từng hành động ("Gửi ZNS `crm.order.thank-you` tới 0912***678"; "Tạo việc 'Gọi chăm sóc'
cho Nguyễn Văn A, hạn 30/10/2026"). Không có tác dụng phụ, không tăng hạn mức. `aut_trigger_sample` giữ 20 payload gần
nhất cho mỗi `(tenant, triggerType, key)` đã che PII, hết hạn sau 7 ngày, chỉ ghi khi có ít nhất một luật (kể cả bản nháp)
dùng trigger đó.

Có thêm "Chạy thật một lần" (`execute` với `dryRun=false`) trên một bản ghi, cần quyền `automation:rule:Execute`.

---

## 4. Mô hình dữ liệu

### 4.1 ERD

```mermaid
erDiagram
  aut_rule ||--o{ aut_rule_version : "có"
  aut_rule ||--o{ aut_run : "sinh"
  aut_rule ||--o{ aut_timer : "hẹn giờ"
  aut_rule ||--o| aut_inbound_endpoint : "INBOUND"
  aut_rule ||--o{ aut_rule_stat : "đếm theo ngày"
  aut_rule ||--o{ aut_assign_cursor : "vòng tròn"
  aut_run ||--o{ aut_run_step : "gồm"
  aut_settings ||--|| aut_rule : "giới hạn tenant"

  aut_rule {
    uuid id PK
    varchar tenant_id
    varchar key "duy nhất theo tenant"
    jsonb name "LocalizedText"
    varchar status "DRAFT|ACTIVE|PAUSED|SUSPENDED|ARCHIVED"
    varchar trigger_type
    varchar trigger_key "chỉ mục so khớp"
    int published_version
    int draft_version
    varchar owner_sub "người publish (A8)"
    varchar source "TENANT|TEMPLATE"
    varchar template_key
    jsonb attributes
    bigint version "optimistic lock"
  }
  aut_rule_version {
    uuid id PK
    uuid rule_id FK
    int version_no
    jsonb definition "DSL, schema_version"
    jsonb compiled "kiểu, prefilter, quyền cần"
    varchar state "DRAFT|PUBLISHED|SUPERSEDED"
    varchar definition_hash
  }
  aut_run {
    uuid id PK
    varchar tenant_id
    uuid rule_id
    int rule_version
    varchar status
    varchar subject_type
    varchar subject_id
    uuid chain_id
    smallint depth
    jsonb input "snapshot đã che, xoá sau 7 ngày"
    timestamptz created_at "khoá partition"
  }
  aut_run_step {
    uuid run_id FK
    varchar action_id
    varchar action_type
    varchar status
    jsonb params "đã che"
    jsonb output
  }
  aut_timer {
    uuid rule_id
    varchar subject_id
    timestamptz fire_at
    uuid task_id
  }
```

### 4.2 Nguyên tắc

- Mọi bảng có `tenant_id`; bảng cấu hình (`aut_rule`, `aut_rule_version`, `aut_settings`, `aut_inbound_endpoint`) là
  `TenantEntity`; bảng ghi nhiều (`aut_run`, `aut_run_step`, `aut_rule_stat`, `aut_timer`, `aut_trigger_sample`) dùng
  JDBC với `tenant_id` tường minh (giống workflow L3) vì partition và ghi lô.
- Không có enum trong DB: `varchar` + kiểm ở code. Loại trigger, loại hành động là chuỗi mở.
- `definition` có `schemaVersion`; bộ nâng cấp (`RuleDefinitionUpgrader`) đọc version cũ, ghi version mới khi publish.
- `attributes` jsonb có kiểm (≤ 8 KB, khoá theo `^[a-z][a-zA-Z0-9_.-]{0,63}$`) cho app gắn nhãn riêng.

### 4.3 DDL dự kiến (`META-INF/vyforge/db/automation/V1__automation_core.sql`)

```sql
create table aut_rule (
  id                 uuid         primary key,
  tenant_id          varchar(64)  not null,
  key                varchar(100) not null,
  name               jsonb        not null,                -- {"vi": "...", "en": "..."}
  description        jsonb,
  status             varchar(16)  not null,                -- DRAFT | ACTIVE | PAUSED | SUSPENDED | ARCHIVED
  status_reason      varchar(64),                          -- loop_detected | owner_inactive | action_unavailable | …
  trigger_type       varchar(16)  not null,                -- EVENT | RECORD | SCHEDULE | RECORD_DATE | MANUAL | INBOUND
  trigger_key        varchar(200) not null,                -- 'payment.succeeded' | 'crm:order#UPDATED' | 'crm:contract.endDate'
  published_version  int,
  draft_version      int,
  owner_sub          varchar(100),                         -- người publish gần nhất, danh tính quyền (A8)
  source             varchar(16)  not null default 'TENANT',
  template_key       varchar(100),
  template_version   int,
  tags               varchar(50)[] not null default '{}',
  debug_until        timestamptz,                          -- ghi chi tiết cả run SKIPPED tới thời điểm này (≤ 24 giờ)
  attributes         jsonb        not null default '{}',
  created_by         varchar(100) not null,
  created_at         timestamptz  not null,
  updated_by         varchar(100) not null,
  updated_at         timestamptz  not null,
  version            bigint       not null default 0,
  constraint uq_aut_rule_key unique (tenant_id, key)
);
-- RuleIndex nạp theo tenant: chỉ luật đang chạy
create index ix_aut_rule_trigger on aut_rule (tenant_id, trigger_type, trigger_key) where status = 'ACTIVE';

create table aut_rule_version (
  id               uuid        primary key,
  tenant_id        varchar(64) not null,
  rule_id          uuid        not null references aut_rule (id) on delete cascade,
  version_no       int         not null,
  state            varchar(16) not null,                   -- DRAFT | PUBLISHED | SUPERSEDED
  schema_version   int         not null,
  definition       jsonb       not null,                   -- DSL (mục 3.4), có condition_ui (cây của trình dựng)
  compiled         jsonb,                                  -- kiểu biến, prefilter, required_permissions, actions[]
  definition_hash  varchar(64) not null,                   -- sha256 của definition chuẩn hoá (so sánh, nhập/xuất)
  comment          varchar(500),
  published_by     varchar(100),
  published_at     timestamptz,
  created_by       varchar(100) not null,
  created_at       timestamptz not null,
  constraint uq_aut_rule_version unique (rule_id, version_no)
);

create table aut_run (
  id              uuid         not null,
  tenant_id       varchar(64)  not null,
  rule_id         uuid         not null,
  rule_version    int          not null,
  trigger_type    varchar(16)  not null,
  trigger_key     varchar(200) not null,
  event_id        varchar(100),                            -- id event / id thay đổi / mốc lịch
  subject_type    varchar(100),                            -- EntityRef (jpa-extras D14)
  subject_id      varchar(200),
  chain_id        uuid         not null,
  depth           smallint     not null,
  parent_run_id   uuid,
  status          varchar(16)  not null,                   -- QUEUED | RUNNING | RETRYING | SUCCEEDED | PARTIAL | FAILED
                                                           -- | SKIPPED | THROTTLED | LOOP | CANCELLED
  status_reason   varchar(64),
  dry_run         boolean      not null default false,
  manual_by       varchar(100),                            -- MANUAL / chạy lại
  input           jsonb,                                   -- snapshot đã che theo @PersonalData; null sau 7 ngày
  input_purged_at timestamptz,
  error_code      varchar(100),
  error_message   varchar(2000),                           -- LogSafeJson, không chứa PII
  task_id         uuid,
  idempotency_key varchar(255) not null,                   -- rule:<id>:<event_id> | timer:<…> | manual:<uuid>
  attempts        smallint     not null default 0,
  created_at      timestamptz  not null,
  started_at      timestamptz,
  finished_at     timestamptz,
  primary key (id, created_at)
) partition by range (created_at);
create unique index uq_aut_run_idem on aut_run (tenant_id, idempotency_key, created_at);
create index ix_aut_run_rule on aut_run (tenant_id, rule_id, created_at desc);
create index ix_aut_run_subject on aut_run (tenant_id, subject_type, subject_id, created_at desc);
create index ix_aut_run_status on aut_run (tenant_id, status, created_at desc)
  where status in ('FAILED', 'RETRYING', 'THROTTLED', 'LOOP', 'PARTIAL');
create index ix_aut_run_chain on aut_run (chain_id);

create table aut_run_step (
  run_id        uuid         not null,
  created_at    timestamptz  not null,                     -- = aut_run.created_at, cùng partition
  tenant_id     varchar(64)  not null,
  seq           smallint     not null,
  action_id     varchar(50)  not null,
  action_type   varchar(100) not null,
  status        varchar(16)  not null,                     -- PENDING | SKIPPED_WHEN | COMPLETED | FAILED | RETRYING
  params        jsonb,                                     -- đã dựng, đã che
  output        jsonb,                                     -- id do module trả (notificationId, todoId, messageId…)
  error_code    varchar(100),
  error_message varchar(2000),
  attempts      smallint     not null default 0,
  started_at    timestamptz,
  finished_at   timestamptz,
  primary key (run_id, created_at, seq)
) partition by range (created_at);

create table aut_rule_stat (                               -- đếm theo ngày, cả lần điều kiện sai
  tenant_id   varchar(64) not null,
  rule_id     uuid        not null,
  day         date        not null,
  matched     bigint      not null default 0,              -- trigger khớp
  skipped     bigint      not null default 0,              -- điều kiện sai
  succeeded   bigint      not null default 0,
  failed      bigint      not null default 0,
  throttled   bigint      not null default 0,
  loop        bigint      not null default 0,
  cooldown    bigint      not null default 0,
  p95_ms      int,
  primary key (tenant_id, rule_id, day)
);

create table aut_timer (                                   -- RECORD_DATE: một hẹn giờ cho mỗi (luật, bản ghi)
  tenant_id     varchar(64)  not null,
  rule_id       uuid         not null,
  subject_type  varchar(100) not null,
  subject_id    varchar(200) not null,
  anchor        timestamptz  not null,                     -- giá trị trường ngày lúc đặt hẹn
  fire_at       timestamptz  not null,
  task_id       uuid         not null,
  primary key (tenant_id, rule_id, subject_type, subject_id)
);
create index ix_aut_timer_subject on aut_timer (tenant_id, subject_type, subject_id);

create table aut_cooldown (
  tenant_id   varchar(64)  not null,
  rule_id     uuid         not null,
  scope_key   varchar(300) not null,                       -- subject hoặc biểu thức `cooldown.key`
  until       timestamptz  not null,
  primary key (tenant_id, rule_id, scope_key)
);

create table aut_assign_cursor (                           -- ROUND_ROBIN: người được giao gần nhất
  tenant_id   varchar(64)  not null,
  rule_id     uuid         not null,
  action_id   varchar(50)  not null,
  pool_hash   varchar(64)  not null,                       -- đổi danh sách người → cursor mới
  last_index  int          not null,
  updated_at  timestamptz  not null,
  primary key (tenant_id, rule_id, action_id)
);

create table aut_inbound_endpoint (
  id            uuid         primary key,
  tenant_id     varchar(64)  not null,
  rule_id       uuid         not null references aut_rule (id) on delete cascade,
  token_hash    varchar(64)  not null unique,              -- sha256(token); token chỉ hiện một lần
  secret        text         not null,                     -- SecretCipher, context automation:inbound:<tenant>:<id>
  payload_schema jsonb       not null,                     -- JSON Schema tenant khai (≤ 32 KB)
  allowed_headers varchar(100)[] not null default '{}',
  status        varchar(16)  not null,                     -- ACTIVE | DISABLED
  last_used_at  timestamptz,
  created_at    timestamptz  not null,
  version       bigint       not null default 0
);

create table aut_trigger_sample (                          -- 20 payload gần nhất cho chạy thử (A12)
  tenant_id     varchar(64)  not null,
  trigger_type  varchar(16)  not null,
  trigger_key   varchar(200) not null,
  captured_at   timestamptz  not null,
  subject_type  varchar(100),
  subject_id    varchar(200),
  payload       jsonb        not null,                     -- đã che
  primary key (tenant_id, trigger_type, trigger_key, captured_at)
);

create table aut_settings (
  tenant_id            varchar(64) primary key,
  max_depth            smallint,                           -- ≤ trần platform
  max_runs_per_minute  int,
  rule_runs_per_minute int,
  retention_days       int,
  failed_retention_days int,
  notify_on_failure    varchar(16) not null default 'DAILY', -- OFF | IMMEDIATE | DAILY
  notify_group         varchar(100),                       -- nhóm IAM nhận báo lỗi/tạm dừng
  attributes           jsonb not null default '{}',
  updated_by           varchar(100) not null,
  updated_at           timestamptz not null,
  version              bigint not null default 0
);
```

### 4.4 Retention, partition, dung lượng

| Bảng | Partition | Giữ mặc định | Dọn |
|---|---|---|---|
| `aut_run`, `aut_run_step` | Theo tháng (`created_at`), tạo trước 2 tháng bằng job `@ClusterScheduled` | Thành công 30 ngày, lỗi/`LOOP`/`THROTTLED` 90 ngày (tenant chỉnh trong `[7, 365]`) | Partition cũ hơn giới hạn lớn nhất của mọi tenant thì `DROP`; trong hạn thì xoá theo lô 5.000 dòng theo tenant |
| `aut_run.input` | — | 7 ngày | Job đặt `input = null`, `input_purged_at = now()` |
| `aut_rule_stat` | Không | 400 ngày | Xoá theo ngày |
| `aut_trigger_sample` | Không | 20 bản/khoá, 7 ngày | Ghi đè vòng; job xoá hết hạn |
| `aut_rule_version` | Không | Mọi version còn tham chiếu bởi run trong hạn + 20 version gần nhất | Job gộp |

Ước lượng: tenant vừa có 50 luật, 20.000 run khớp/ngày, mỗi run ~3 KB (kèm step) → ~60 MB/ngày, ~1,8 GB cho 30 ngày.
Không lưu run `SKIPPED` là chỗ tiết kiệm chính (thường gấp 10–100 lần số run khớp).

### 4.5 Tiến hoá schema

- DSL có `schemaVersion`; mỗi version có JSON Schema trong jar; `RuleDefinitionUpgrader` chuyển tiếp từng bước (1→2→3).
- Event trong catalog có `version`; luật ghi version event đã biên dịch. Event lên version mới mà không tương thích thì
  catalog giữ bản cũ ở chế độ `deprecated` cho tới khi không còn luật dùng (giống webhooks).
- Migration có version ở `META-INF/vyforge/db/automation`, bảng lịch sử `flyway_schema_history_automation`, tắt bằng
  `vyforge.automation.schema.migrate=false`; file gộp `META-INF/vyforge/sql/automation.sql` cho DBA.

---

## 5. API

### 5.1 Java

```java
package com.vyforge.platform.automation;

/** Quản lý luật. Mọi method chạy trong TenantContext hiện tại; kiểm quyền ở lớp REST (admin) hoặc caller. */
public interface AutomationRules {
    RuleView create(CreateRule command);                                   // DRAFT, version 1 nháp
    RuleView saveDraft(String key, RuleDefinition definition, long expectedVersion);
    ValidationReport validate(String key, @Nullable Integer versionNo);   // biên dịch, kiểm quyền, kiểm giới hạn
    RuleView publish(String key, PublishOptions options);                  // comment, activate=true
    RuleView enable(String key);
    RuleView disable(String key, String reason);
    RuleView archive(String key);
    RuleView transferOwnership(String key, String newOwnerSub);           // K17; kiểm quyền của chủ mới
    SimulationResult simulate(String key, SimulationRequest request);     // A12
    RunView execute(String key, ExecuteRequest request);                  // MANUAL hoặc "chạy thật một lần"
    RuleView installTemplate(String templateKey, InstallOptions options); // tạo bản sao TEMPLATE → TENANT
    Optional<RuleView> find(String key);
    Page<RuleSummary> list(RuleQuery query, Pageable page);
    RuleExport export(String key);                                         // YAML + hash
    RuleView importRule(RuleExport export, ImportOptions options);
}

/** Lịch sử chạy. */
public interface AutomationRuns {
    Page<RunSummary> list(RunQuery query, CursorRequest cursor);           // cursor (admin-kit D4)
    Optional<RunDetail> find(UUID runId);                                  // kèm step, chuỗi
    List<RunSummary> chain(UUID chainId);
    RunView retry(UUID runId, RetryMode mode);                             // SAME_VERSION | CURRENT_VERSION
    void cancel(UUID runId, String reason);
    RuleStats stats(String key, LocalDate from, LocalDate to);
}

/** Catalog trigger (đọc), dùng cho trình dựng. */
public interface AutomationTriggers {
    List<TriggerDescriptor> events();                                      // đã lọc theo quyền đọc dữ liệu
    List<RecordTypeDescriptor> recordTypes();
    List<ActionDescriptor> actions();                                      // type, nhãn, paramsSchema, quyền cần
    ExpressionHelp expressionHelp(TriggerRef trigger);                     // biến, kiểu, hàm (gợi ý cho editor)
}

/** App không dùng JPA (hoặc thay đổi đến từ hệ thống ngoài) tự báo thay đổi bản ghi. Gọi trong transaction. */
public interface RecordChanges {
    void publish(RecordChange change);
}
public record RecordChange(String recordType, String recordId, ChangeKind kind,        // CREATED | UPDATED | DELETED
                           Map<String, Object> after, @Nullable Map<String, Object> before,
                           Set<String> changedFields, @Nullable String actorSub, Instant occurredAt) {}

/** Chuỗi nhân quả hiện tại (A6). Module khác chỉ cần đọc khi tự mang qua ranh giới bất đồng bộ. */
public final class AutomationChain {
    public static Optional<AutomationChain> current();
    public UUID id(); public int depth(); public Set<String> visited();
    public String toHeader();                                              // vyforge-automation-chain
    public static Optional<AutomationChain> fromHeader(@Nullable String value);
}
```

Annotation cho app:

```java
/** Event dùng được làm trigger. Record phải implement TenantAwareEvent. Schema sinh từ record component. */
@Target(TYPE) @Retention(RUNTIME)
public @interface AutomationEvent {
    String value();                         // "payment.succeeded" — ổn định, không đổi khi đổi tên class
    int version() default 1;
    String label() default "";              // khoá message: automation.event.payment.succeeded
    String subject() default "";            // CEL trên event ra EntityRef chính: "'payment:transaction/' + id"
    PersonalDataLevel personalData() default PersonalDataLevel.NONE;
}

/** Entity tự động hoá được. Hibernate listener phát RecordChange cho entity này (A10). */
@Target(TYPE) @Retention(RUNTIME)
public @interface AutomationRecord {
    String type();                          // "crm:order" (EntityRef type)
    String label() default "";
    String[] dateFields() default {};       // trường dùng được cho RECORD_DATE
}

/** Trường lộ ra cho luật. Không có annotation thì trường không hiện, trừ khi @AutomationRecord(exposeAll = true). */
@Target(FIELD) @Retention(RUNTIME)
public @interface AutomationField {
    String label() default "";
    boolean writable() default false;       // record.update được ghi trường này
    String format() default "";             // money:VND, user, enum
}
```

Ví dụ:

```java
@AutomationEvent(value = "payment.succeeded", subject = "'payment:transaction/' + transactionId")
public record PaymentSucceeded(String tenantId, UUID transactionId, String orderRef, Money amount,
                               String provider, Instant paidAt) implements TenantAwareEvent {}

@Entity @AutomationRecord(type = "crm:order", dateFields = {"deliveredAt", "dueDate"})
public class Order extends TenantEntity {
    @AutomationField(label = "order.code") String code;
    @AutomationField(label = "order.status", writable = true) @Enumerated(STRING) OrderStatus status;
    @AutomationField(label = "order.total", format = "money:VND") BigDecimal total;
    @AutomationField(label = "order.owner", format = "user", writable = true) String ownerSub;
    @PersonalData @AutomationField(label = "order.customerPhone") String customerPhone;   // che khi log
}
```

### 5.2 REST (OpenAPI phác thảo)

Route gateway `/api/v1/automation/**` (platform-services §5.1). Problem Details có `code` ổn định.

```yaml
openapi: 3.1.0
info: { title: vyforge automation, version: 0.13.0 }
paths:
  # --- Quản trị (tenant admin, người vận hành đang xem tenant) ---
  /admin/info:                         { get: { summary: 'Capability: calendar, notification, webhooks, workflow…, giới hạn, hạn mức đã dùng' } }
  /admin/catalog/events:               { get: { summary: 'Event dùng được làm trigger (schema, ví dụ, mức PII)' } }
  /admin/catalog/record-types:         { get: { summary: 'Loại bản ghi, trường, kiểu, trường ghi được, trường ngày' } }
  /admin/catalog/actions:              { get: { summary: 'Hành động khả dụng: paramsSchema, outputsSchema, quyền cần' } }
  /admin/catalog/functions:            { get: { summary: 'Hàm CEL và biến theo trigger (gợi ý cho editor)' } }
  /admin/expressions:check:            { post: { summary: 'Biên dịch biểu thức với ngữ cảnh trigger; trả lỗi có vị trí' } }
  /admin/rules:
    get:  { summary: 'Danh sách (lọc status, trigger, tag, owner, q)' }
    post: { summary: 'Tạo luật (nháp)' }
  /admin/rules/{key}:
    get:    { summary: 'Chi tiết: bản đang chạy, bản nháp, thống kê 7 ngày' }
    put:    { summary: 'Lưu bản nháp (If-Match: version)' }
    delete: { summary: 'Lưu trữ' }
  /admin/rules/{key}/validate:         { post: {} }
  /admin/rules/{key}/simulate:         { post: { summary: 'Chạy thử: sample | recentEvents | subject' } }
  /admin/rules/{key}/publish:          { post: { summary: 'Publish bản nháp; body: comment, activate' } }
  /admin/rules/{key}/enable:           { post: {} }
  /admin/rules/{key}/disable:          { post: { summary: 'body: reason' } }
  /admin/rules/{key}/owner:            { put: { summary: 'Chuyển chủ luật' } }
  /admin/rules/{key}/debug:            { put: { summary: 'Ghi chi tiết tới thời điểm (≤ 24 giờ)' } }
  /admin/rules/{key}/versions:         { get: {} }
  /admin/rules/{key}/versions/{n}:     { get: { summary: 'Định nghĩa của version; ?format=yaml' } }
  /admin/rules/{key}/versions/{n}/restore: { post: { summary: 'Chép version cũ thành bản nháp' } }
  /admin/rules/{key}/stats:            { get: { summary: 'Theo ngày: matched, skipped, succeeded, failed, throttled, loop' } }
  /admin/rules/{key}/backfill:         { post: { summary: 'RECORD_DATE: lên lịch cho bản ghi hiện có (xem trước: ?dryRun=true)' } }
  /admin/rules/{key}/inbound:
    post:   { summary: 'Tạo endpoint INBOUND; trả token + secret một lần' }
    delete: { summary: 'Thu hồi' }
  /admin/rules/{key}/inbound/rotate:   { post: { summary: 'Xoay secret (chồng lấn 24 giờ)' } }
  /admin/rules:import:                 { post: { summary: 'Nhập YAML/JSON (kiểm hash, đổi key khi trùng)' } }
  /admin/rules/{key}/export:           { get: {} }
  /admin/templates:                    { get: { summary: 'Mẫu luật do app giao (đã lọc theo capability)' } }
  /admin/templates/{key}/install:      { post: {} }
  /admin/runs:                         { get: { summary: 'Cursor; lọc rule, status, subject, khoảng thời gian, chainId' } }
  /admin/runs/{id}:                    { get: { summary: 'Chi tiết + step + chuỗi' } }
  /admin/runs/{id}/retry:              { post: { summary: 'mode: SAME_VERSION | CURRENT_VERSION' } }
  /admin/runs/{id}/cancel:             { post: {} }
  /admin/runs/bulk/{op}:               { post: { summary: 'retry|cancel theo ids hoặc filter (admin-kit D8)' } }
  /admin/settings:                     { get: {}, put: {} }
  /admin/usage:                        { get: { summary: 'Luật đang bật / giới hạn, lần chạy tháng này / hạn mức' } }
  /admin/tasks/**:                     { summary: 'TaskAdminEndpoint của -scheduling cho queue automation (admin-kit D10)' }
  /admin/audit:                        { get: { summary: 'ModuleAuditEndpoint (admin-kit D9)' } }

  # --- Người dùng cuối ---
  /rules/manual:                       { get: { summary: '?recordType=crm:order — luật MANUAL tôi được chạy trên loại bản ghi này' } }
  /rules/{key}/execute:                { post: { summary: 'Chạy MANUAL trên subject; body: subject, input' } }
  /runs/by-subject:                    { get: { summary: '?subject=crm:order/01J… — run gần đây trên bản ghi tôi xem được' } }

  # --- Gọi vào (không cần token người dùng) ---
  /hooks/{token}:
    post:
      summary: 'INBOUND. Header X-Vyforge-Signature: t=<unix>,v1=<hmac-sha256(secret, t + "." + body)>; lệch ≤ 5 phút'
      responses: { '202': {}, '401': {}, '404': {}, '413': {}, '422': {}, '429': {} }

  # --- Nội bộ (platform-services, client credentials + X-Tenant-Id) ---
  /internal/record-changes:            { post: { summary: 'App ở mode remote đẩy RecordChange theo lô (dự phòng khi không có Kafka)' } }
  /internal/catalogs/{service}:        { put: { summary: 'App đăng ký event/record type/action lúc khởi động' } }
```

Mã lỗi chính (`platform-errors-automation*.properties`, vi/en): `automation.rule_not_found`, `automation.rule_key_taken`,
`automation.condition_invalid`, `automation.params_invalid`, `automation.action_unavailable`,
`automation.trigger_unavailable`, `automation.permission_missing` (người publish thiếu quyền hành động),
`automation.rules_limit_reached` (features), `automation.runs_quota_exceeded`, `automation.rule_not_manual`,
`automation.subject_not_found`, `automation.expression_runtime_error`, `automation.inbound_signature_invalid`,
`automation.version_conflict`.

### 5.3 Event công khai

Mọi event là `record` implement `TenantAwareEvent`, có `schemaVersion`, phát qua Modulith sau commit. Bật
`vyforge.automation.events.externalize=true` thì ra Kafka `automation-events` (key `ruleId`).

| Event | Khi | Trường chính | Ai nghe |
|---|---|---|---|
| `AutomationRulePublished` | Publish version | `ruleId`, `key`, `versionNo`, `publishedBy` | Audit, RuleIndex các instance khác (xoá cache) |
| `AutomationRuleStatusChanged` | Bật, tắt, lưu trữ | `ruleId`, `key`, `from`, `to`, `reason` | RuleIndex, admin |
| `AutomationRuleSuspended` | Hệ thống tạm dừng | `ruleId`, `key`, `reason` (`loop_detected`, `owner_inactive`, `action_unavailable`, `schema_incompatible`, `failure_streak`) | notification (báo tenant admin), ops |
| `AutomationRunFailed` | Run `FAILED` | `runId`, `ruleId`, `key`, `errorCode`, `subjectRef` | Báo cáo, ops-console |
| `AutomationQuotaReached` | Đạt 80 % và 100 % hạn mức tháng | `used`, `limit`, `level` | notification, billing (gợi ý nâng gói) |

Event của automation **không có trong catalog trigger** (không làm trigger cho luật khác), để tránh luật phản ứng với
chính hoạt động của automation. Run thành công không phát event (khối lượng lớn); ai cần thì đọc API run.

Automation **nghe**: mọi event có `@AutomationEvent`; `ScheduleFired` (calendar, `owner = automation`);
`BusinessCalendarChanged` (calendar, tính lại `aut_timer` theo ngày làm việc); `TenantDeleted`/offboarding (dọn dữ liệu);
`UserDeactivated` (IAM, kiểm luật của người đó, K17); `FeatureEntitlementChanged` (features, kiểm lại giới hạn).

### 5.4 Package frontend `@vyforge/automation`

Server-only client qua `@vyforge/api-client`, component dùng `@vyforge/ui` và `@vyforge/admin-kit`, i18n vi/en.

| Export | Nội dung |
|---|---|
| `@vyforge/automation/server` | `createAutomationClient({ baseUrl, getAccessToken })`; Server Action dựng sẵn: `executeRule`, `listManualRules`, `listRunsBySubject` |
| `@vyforge/automation` | `RunRuleMenu` (nút "Tự động hoá ▾" trên trang bản ghi: liệt kê luật `MANUAL` của loại bản ghi, form `input` theo JSON Schema, xác nhận), `SubjectRunList` (run gần đây trên bản ghi, cho người có quyền) |
| `@vyforge/automation/admin` | Component quản trị (mục 10.3) |
| `@vyforge/automation/expression` | `CelEditor` (CodeMirror 6: `codemirror` 6.0.2 + `@codemirror/view` 6.43.15, MIT, kiểm npm 27/10/2026; nạp lazy; gợi ý biến/hàm từ `/admin/catalog/functions`, kiểm qua `/admin/expressions:check` có debounce), `ConditionBuilder` (cây AND/OR ↔ CEL), `TemplateInput` (chuỗi có `{{ }}`, chèn biến bằng menu) — dùng lại được cho workflow |
| `@vyforge/automation/messages` | `vi.json`, `en.json` |

Không đánh giá CEL ở trình duyệt (không cần thư viện CEL JS): mọi kiểm và chạy thử gọi server.
