# Workflow — quy trình phê duyệt nhiều cấp

## 1. Mục đích

Cho mọi app nghiệp vụ một quy trình phê duyệt dùng chung (BPM nhẹ): đề nghị thanh toán, mua sắm, nghỉ phép… đi qua
nhiều cấp duyệt theo sơ đồ tổ chức, ma trận hạn mức, SLA ngày làm việc và uỷ quyền đúng luật Việt Nam. App chỉ khai báo
quy trình bằng YAML (VyFlow) và cài `SubjectAdapter` cho bản ghi của mình; hộp việc, dòng thời gian, in phiếu duyệt, lưu
trữ 5–10 năm đều do module lo. Thiết kế: `docs/design/workflow.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-workflow` | platform-java 0.16.1; engine Flowable 8.0.0 nhúng (chỉ process engine) |
| Package web | `@vyforge/workflow` 0.1.0 | hộp việc, chi tiết việc, panel duyệt; subpath `/server`, `/hooks`, `/admin`, `/messages` |
| Zone admin | `apps/workflow-admin` (platform-admin, `/m/workflow`) | định nghĩa, ma trận, hồ sơ, sự cố, uỷ quyền, cài đặt |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling` (queue `workflow`), `platform-expression` (CEL) | |

## 3. Tính năng chính

- **VyFlow** (`META-INF/vyforge/workflow/definitions/<key>.yml`): khối `approval`, `condition`, `parallel`, `task`
  (bước hệ thống qua `StepAction`, có retry), `notify`, `wait` (chờ `Workflow.signal(correlationKey)`). Bản `CODE` do
  app giao; tenant sửa thành bản `TENANT`, xoá bản tenant là về mặc định. Publish có validate + simulate.
- **Chế độ duyệt**: `kind` APPROVE/INITIAL/ACKNOWLEDGE/REVIEW; `mode` ANY/ALL/SEQUENTIAL/`COUNT:n`/`PERCENT:p`;
  `onReject` END/RETURN_TO_REQUESTER/RETURN_TO_PREVIOUS; `skipIf` (CEL), `pool`, `bulk`, `requireComment`.
- **Người duyệt** (resolver): `user`, `group`, `requester`, `field`, `chosenBy`, `expression`, `matrix` (ma trận hạn
  mức `AuthorityMatrices`, `CHAIN_UNTIL_LIMIT`), `manager`/`unitHead`/`position` (cần module organization). Chính sách
  theo thứ tự: uỷ quyền → bốn mắt → tự duyệt (`selfApproval`) → không có ai → trùng người (`duplicateApprover`).
- **SLA**: `2bd`, `4bh` theo lịch làm việc (`BusinessCalendar`), nhắc trước hạn, leo thang (`ADD_APPROVER`…), job quét
  quá hạn.
- **Uỷ quyền**: một tầng, không uỷ quyền lại, tối đa 366 ngày (BLDS 2015 Đ.140, NĐ 30/2020 Đ.13).
- **Đánh số hồ sơ** (`numbering`: FAST/GAPLESS) qua module sequence; rút/huỷ hồ sơ thì huỷ số.
- **Nhật ký quyết định có chuỗi băm** (`wf_action` không sửa được, `GET /admin/instances/{id}/verify`).
- **API Java**: `Workflow` (start, signal), `WorkflowTasks` (act), `WorkflowDefinitions`, `Delegations`,
  `AuthorityMatrices`, `WorkflowAdmin`.
- **SPI**: `SubjectAdapter` (snapshot, revision, khoá sửa bản ghi), `WorkflowHook` (`beforeStart` chặn được),
  `AssigneeResolver`, `StepAction`.
- **REST** `/api/v1/workflow`: `GET /me/tasks?view=TODO|UPCOMING|DONE|SUBMITTED|CC`, `/me/tasks/counts`,
  `/me/delegations`; `POST /instances`, `/instances/{id}/withdraw|resubmit|comments|watchers`, `/timeline`, `/print`
  (PDF); `POST /tasks/{id}/actions`, `/tasks/bulk-actions`; `/admin/**` (definitions, matrices, instances
  cancel/suspend/resume/move/retry, reassign, settings, stats); `GET /admin/info` (capabilities cho admin MFE).
- **Quan sát**: metric `vyforge.workflow.*`, health `workflow` (kiểm schema Flowable).
- **Dữ liệu cá nhân**: `WorkflowPersonalData` cho privacy-dsr (xoá bình luận, giữ quyết định trong thời hạn lưu); job
  `workflow-retention-purge` xoá hồ sơ hết hạn lưu, tôn trọng `legal_hold`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-workflow</artifactId>
</dependency>
```

```yaml
vyforge:
  workflow:
    enabled: true          # mặc định
    # schema.migrate: false  # nếu DBA tự chạy META-INF/vyforge/sql/workflow.sql
```

```yaml
# src/main/resources/META-INF/vyforge/workflow/definitions/payment-request.yml
schemaVersion: 1
key: payment-request
name: { vi: Đề nghị thanh toán, en: Payment request }
category: finance
subject: { type: finance:payment-request, revisionCheck: STRICT }
variables:
  amount: { type: money, required: true }
flow:
  - approval: { id: head, name: Trưởng phòng, assignees: [{ manager: { of: requester, level: 1 } }], sla: { duration: 2bd } }
  - approval: { id: cfo, name: CFO, assignees: [{ group: cfo }], skipIf: "amount < money(100000000, 'VND')" }
```

```java
StartResult r = workflow.start(StartRequest.of("payment-request",
        SubjectRef.of("finance", "payment-request", id).withRevision("3"),
        SubjectSnapshot.of("DNTT 125.000.000 đ", Map.of("amount", Money.vnd(125_000_000)))));
tasks.act(taskId, TaskAction.Approve.of("Đồng ý"));
```

Frontend: `pnpm add @vyforge/workflow`, dùng `TaskInbox`/`TaskDetail`/`ApprovalPanel` trong app nghiệp vụ, client từ
`@vyforge/workflow/server`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.workflow.enabled` | `true` | Tắt toàn bộ starter |
| `vyforge.workflow.mode` | `embedded` | `remote` chưa có |
| `vyforge.workflow.bpmn.enabled` | `false` | Cho nhập BPMN thô; bật async executor Flowable |
| `vyforge.workflow.schema.migrate` | `true` | Tự chạy Flyway (`flyway_schema_history_workflow`) |
| `vyforge.workflow.defaults.*` | ESCALATE, KEEP_LAST, ASSIGN_PROCESS_ADMIN, … | Mặc định `settings` của VyFlow |
| `vyforge.workflow.delegation.max-duration` / `.allow-chain` / `.allow-self-service` | `366d` / `false` / `true` | Uỷ quyền |
| `vyforge.workflow.limits.*` | 60 bước, 200 việc mở, 100 việc/lần duyệt hàng loạt, snapshot 64 KB | Giới hạn tải |
| `vyforge.workflow.retention.default-period` / `.finance` | `P5Y` / `P10Y` | Thời hạn lưu (Luật Kế toán 2015 Đ.41) |
| `vyforge.workflow.sla.breach-scan` | `1h` | Chu kỳ quét quá hạn |
| `vyforge.workflow.static-org.*` | tắt | Tổ chức/nhóm tĩnh cho test, dự án nhỏ |
| `vyforge.workflow.api.base-path` | `/api/v1/workflow` | Gốc REST |
| `vyforge.workflow.integrations.<module>.enabled` | `true` | Tắt từng adapter |

## 6. Tích hợp với module khác

Workflow sở hữu port, module kia cài adapter (bật tự động khi có module trên classpath):

| Port | Adapter / module |
|---|---|
| `OrgDirectory` | `WorkflowOrgDirectory` ở **organization** |
| `BusinessCalendar` | `CalendarWorkflowCalendar` ở **calendar** (vắng thì 1bd = 24 giờ) |
| `DocumentNumbering` | `WorkflowDocumentNumbering` ở **sequence** (vắng thì `LocalNumbering`) |
| `FormRegistry` | `FormsFormRegistry` (**forms**) — publish kiểm `form` của bước tồn tại |
| `AttachmentStore` | `FilesAttachmentStore` (**files**) — đính kèm quyết định, slot `decisions` |
| `ApprovalPrinter` | `ReportApprovalPrinter` (**platform-report**) — template `workflow.approval-sheet` |
| `WorkflowNotifier`, `InboxPush`, `ParticipantDirectory` | **notification**, **realtime** (kênh `workflow.inbox`), **authz** (IAM) |
| privacy-dsr | `WorkflowPersonalData` (`PersonalDataContributor`) |

Automation dùng workflow qua action/trigger của nó (xem `automation.md`).

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: chỉ `embedded` (chung `DataSource` và transaction với app). `remote` (client HTTP, `/internal/**`, Kafka
  `workflow-events`) để bản sau.
- **IAM** (chỉ khi có `platform-starter-authz`): `workflow:task:Act` (policy chỉ để chặn — quyền duyệt theo danh tính),
  `workflow:instance:Start|Get|List|Admin`, `workflow:definition:List|Get|Create|Update|Delete|Simulate|Publish|Retire|DeployBpmn`, `workflow:matrix:List|Get|Update`,
  `workflow:delegation:Manage`, `workflow:settings:Get|Update`, `workflow:stats:Get`, `workflow:audit:Verify`.
- **Sự kiện** (`WorkflowEvent`, Spring event trong transaction, nghe bằng `@TransactionalEventListener`):
  `WorkflowStarted`, `WorkflowStepEntered`, `WorkflowTaskCreated`, `WorkflowTaskCompleted`, `WorkflowReturned`,
  `WorkflowInfoRequested`, `WorkflowSlaWarning`, `WorkflowSlaBreached`, `WorkflowEscalated`, `WorkflowCompleted`,
  `WorkflowIncident`, `WorkflowDelegationChanged`. Không chứa bình luận hay dữ liệu bản ghi.

## 8. Trạng thái & còn mở

- Đã có: VyFlow đầy đủ khối, ma trận hạn mức, SLA lịch làm việc, uỷ quyền, chuỗi băm quyết định, in phiếu, retention,
  admin MFE, package web.
- Chưa có: chế độ `remote`; di chuyển hồ sơ sang version mới (`migrate`); bước `http.call`; `formData` trên thao tác;
  adapter esign (`EnvelopeGate`, `esign.sign`) và ai (`ApprovalRouter`); API đặt `legal_hold`.
- Pháp lý: thời hạn lưu 10 năm nhóm `finance` cần pháp chế của khách xác nhận.
- `@vyforge/workflow` 0.1.0: zone admin đã kiểm bằng tarball local; lockfile cập nhật sau khi publish (cần xác nhận đã
  publish).
