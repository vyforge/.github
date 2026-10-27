# ADR-0024: Workflow phê duyệt nhúng Flowable 8 sau lớp nghiệp vụ của Vyforge

- **Trạng thái:** Chấp nhận — 27/10/2026. Thiết kế chi tiết: [workflow.md](../design/workflow.md). Liên quan
  [ADR-0009](0009-iam-authorization.md), [ADR-0015](0015-tenant-data-isolation-platform-core.md),
  [ADR-0020](0020-shared-application-modules.md) (mục platform-expression, calendar, sequence, forms),
  [ADR-0022](0022-shared-organization-structure.md).
- **Phát hành:** `platform-starter-workflow` (platform-java 0.14.0); `@vyforge/workflow` 0.1.0 (platform-web) và admin
  MFE `/m/workflow` (platform-admin) đã xong trên nhánh, chờ merge.

## Bối cảnh

App nghiệp vụ cần quy trình phê duyệt kiểu Việt Nam: trình duyệt nhiều cấp theo hạn mức số tiền, yêu cầu bổ sung, thêm
người duyệt, uỷ quyền khi vắng, hạn xử lý tính theo ngày làm việc, nhật ký quyết định có giá trị chứng từ. MODULE_PLAN
mục 5 cấm tự viết workflow engine và loại Camunda 8; nguyên tắc 4 yêu cầu lớp mỏng trên OSS. Người dùng nghiệp vụ không
vẽ BPMN, còn hộp việc phải lọc theo dữ liệu nghiệp vụ (số tiền, loại hồ sơ, hạn) mà biến của engine (EAV) không lọc
nhanh được.

## Quyết định

1. **Nhúng Flowable 8.0, chỉ process engine BPMN**, sau port nội bộ `ProcessRuntime`; chỉ package `internal.flowable`
   được import `org.flowable..` (ArchUnit). Dùng `flowable-spring` và tự dựng engine, không dùng
   `flowable-spring-boot-starter-process` (tránh auto-config IDM, REST và property riêng của Flowable).
2. **Hai cách soạn:** VyFlow (DSL dạng khối: tuần tự, rẽ nhánh, song song, bước duyệt, bước hệ thống; JSON/YAML có JSON
   Schema; trình dựng dọc) biên dịch sang BPMN; BPMN 2.0 thô chỉ cho người vận hành/dev và tắt mặc định.
3. **Bảng `wf_*` của Vyforge là nguồn sự thật nghiệp vụ** (hồ sơ, bước, việc, quyết định, uỷ quyền); Flowable chỉ giữ
   token thực thi. Hộp việc đọc `wf_task`, không đọc `ACT_RU_TASK`. Mọi bảng `wf_*` truy cập bằng `JdbcClient`.
4. **Thao tác duyệt ở lớp Vyforge:** yêu cầu bổ sung, thêm người duyệt trước/sau/song song, tham vấn là việc con
   (`parent_task_id`); trả lại bước trước dùng `ChangeActivityStateBuilder`. Từ chối kết thúc bằng gateway + terminate
   end event trong BPMN sinh ra; `vy_outcome` do lớp Vyforge đặt.
5. **Người duyệt chụp lại khi bước bắt đầu**; chỉ "nhóm xử lý chung" giữ ứng viên động và phải nhận việc.
6. **Điều kiện viết bằng CEL** qua `platform-expression` (`Expressions`, `ExpressionFunctions`), biên dịch khi publish.
   Tenant không bao giờ viết JUEL (JUEL gọi được bean Spring).
7. **SLA, nhắc việc, escalation chạy ở lớp Vyforge** trên `DurableTaskQueue` và port lịch làm việc; BPMN do VyFlow sinh
   không có timer. Thời gian tạm dừng (chờ bổ sung) không tính vào SLA.
8. **Ma trận thẩm quyền là dữ liệu** tenant admin sửa được, không phải publish lại quy trình.
9. **Nhật ký quyết định chỉ ghi thêm, có chuỗi băm** (`wf_action.hash`), trigger chặn `UPDATE`/`DELETE`.
10. **Bản ghi nghiệp vụ gắn qua `SubjectRef` + revision** và SPI `SubjectAdapter`; bản ghi đổi revision sau lúc trình thì
    không duyệt được (`workflow.subject_changed`).
11. **Quyền:** danh tính quyết định ai được duyệt; policy IAM chỉ thu hẹp. `workflow:task:Act` và
    `workflow:instance:Start` chỉ chặn khi có `Deny` tường minh. API quản trị chỉ có khi có `-authz`.
12. **Không phụ thuộc cứng module khác:** organization (`OrgDirectory`), calendar, sequence, forms, files, export, esign,
    ai nối qua port của workflow, adapter bật bằng `@ConditionalOnBean`.

## Phương án đã loại

| Phương án | Lý do loại |
|---|---|
| Tự viết bộ thông dịch cây khối (khoảng 10 bảng) | Trái MODULE_PLAN mục 5 và nguyên tắc 4; phải tự làm song song, bù, timer, di chuyển version |
| Camunda 8 | Camunda License v1, production self-managed phải mua Enterprise; cụm Zeebe chạy riêng, không nhúng được |
| Camunda 7 CE | Đã End of Life, repo archive |
| Operaton 2.2 (fork Camunda 7, Apache-2.0) | Dùng được; giữ làm phương án B nếu Flowable 8 không chạy trên Boot 4.1/Java 25. Spike Flowable đạt nên không dùng |
| Temporal | Mô hình workflow-as-code và cụm server riêng; không có task người duyệt, không có BPMN cho người vận hành |
| Kogito / jBPM | Repo archive, hướng codegen cloud-native, không hợp nhúng |
| Biểu thức JUEL của Flowable cho tenant | Gọi được bean Spring: rủi ro thực thi mã |
| Timer và async executor của Flowable cho SLA | Không tính được ngày làm việc VN và tạm dừng; thêm một hệ việc nền thứ hai |

## Hệ quả

- Thêm khoảng 32 bảng engine (MyBatis) trong schema riêng của module. Script V1 của Flowable **sinh lúc build bằng
  antrun** từ jar Flowable, không chép SQL vào repo, có test kiểm khớp.
- Đổi engine về sau chỉ đụng `internal.flowable` (port `ProcessRuntime`).
- Workflow phụ thuộc `platform-expression`, nên nhánh expression phải merge trước (đã ở 0.14.0).
- Một hộp việc chung cho nhiều app chỉ có khi chạy ở `platform-services`; `mode: remote` chưa làm ở 0.14.

## Lệch so với thiết kế

- V1 là SQL sinh lúc build thay cho Flowable Java migration `V1__flowable_8_0_0.java`, vì `StarterSchemaMigrator` chỉ
  quét `classpath:META-INF/vyforge/db/workflow`.
- Bảng cấu hình cũng dùng JDBC (thiết kế cho phép JPA).
- `notify` là một StepAction (receiveTask) có retry, thay vì "không có nút BPMN".
- Thêm cột `wf_task.sla_breached_at`, `wf_instance.returned_steps/incident/preselected/category`, `wf_wait.payload`,
  `wf_action.step_id`; `sla_used_seconds` lưu thời lượng làm việc còn lại lúc tạm dừng; `client_ip` là varchar.
- KEEP_LAST dự đoán người duyệt sau theo đường đi hiện tại, chưa áp tự duyệt/uỷ quyền trên dự đoán.
- Khi bật static org, `ParticipantDirectory` tĩnh thắng adapter IAM.
- Trang BPMN của zone chỉ hướng dẫn nhập qua API; FlowDesigner chỉ lưu VyFlow.

## Còn mở

- `mode: remote` (client HTTP, `/internal/**`, Kafka `workflow-events`, `WorkflowContractTests`), di chuyển hồ sơ sang
  version mới, job xoá theo retention, `PersonalDataEraser`, bước `http.call`, lịch theo từng định nghĩa, gauge
  `instances.active`/`tasks.open`, health indicator, e2e cùng shell.
- Adapter thật cho organization, calendar, sequence (GAPLESS), forms, files, export, esign (`EnvelopeGate`), ai.
- Thời hạn lưu mặc định 10 năm của nhóm `finance` cần pháp chế xác nhận (Q4, xem
  [câu hỏi pháp chế](../legal-open-questions.md)).
