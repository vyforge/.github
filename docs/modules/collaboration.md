# Collaboration — trao đổi, dòng thời gian, việc cần làm trên bản ghi

## 1. Mục đích

Gắn vào bất kỳ bản ghi nghiệp vụ nào (đơn hàng, hợp đồng, bài viết…) một khung "chatter" dùng chung: thread trao đổi và
ghi chú nội bộ có mention, dòng thời gian thay đổi field, việc cần làm có hạn theo ngày làm việc, nhãn, theo dõi. Kèm
kiểm duyệt bình luận công khai và xử lý yêu cầu gỡ của cơ quan có thẩm quyền theo NĐ 147/2024. Quyền đi theo bản ghi
cha. Thiết kế: `docs/design/collaboration.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-collaboration` | platform-java 0.16.1 |
| Package web | `@vyforge/collaboration` 0.2.0 | `Chatter`, `CommentThread`, `Timeline`, `TodoList`…; `/server`, `/hooks`, `/editor` (Tiptap, `vf-doc/1`), `/messages`, `/admin` |
| Zone admin | `apps/collaboration-admin` (`/m/collaboration`) | kiểm duyệt, báo cáo vi phạm, yêu cầu gỡ, bộ lọc từ khoá, hạn chế người dùng, nhãn, loại việc, loại bản ghi |
| Phụ thuộc bắt buộc | PostgreSQL, `platform-starter-scheduling` (queue `collaboration`), `platform-starter-authz` | |

## 3. Tính năng chính

- **Loại bản ghi** (`META-INF/vyforge/collaboration/subject-types/*.yml` hoặc bean `SubjectTypeDefinition`): quyền theo
  action IAM của bản ghi cha (`read`/`contribute`/`write`, hoặc `access.mode: POLICY` + `SubjectAccessPolicy`),
  visibility `INTERNAL`/`PUBLIC`, `edit-window`, `moderation` (OFF/POST/…), `on-subject-deleted`, `public-profile:
  VN_SOCIAL` (chỉ tài khoản đã xác thực SĐT được bình luận — NĐ 147/2024). Không đọc được bản ghi → 404, không góp ý
  được → 403.
- **Trao đổi**: thread, comment, ghi chú, trả lời, reaction, mention `@user`/`GROUP`/`ORG_UNIT`, khoá/giải quyết/mở lại
  thread, lịch sử sửa, nội dung `vf-doc/1` (không HTML thô).
- **Dòng thời gian**: `ActivityRecorder` + `@TrackedEntity`/`@Tracked` trên entity JPA (thay đổi field thành một dòng
  mỗi transaction; field `SENSITIVE` chỉ ghi "đã cập nhật"); nguồn ngoài qua `TimelineContributor`, audit log tuỳ chọn.
- **Việc cần làm**: loại việc, giao người, hạn tương đối theo ngày làm việc (`DueSpec.relative(2, BUSINESS_DAY)`),
  nhắc sắp hạn/quá hạn, outcome (`APPROVED`/`REJECTED`).
- **Nhãn, theo dõi, đã đọc**, tóm tắt theo nhiều bản ghi (`/subjects/summary`).
- **Kiểm duyệt**: hàng đợi, báo cáo vi phạm, bộ lọc từ khoá, hạn chế người viết, yêu cầu gỡ có SLA (24h/6h/3h theo mức;
  khiếu nại người dùng 48h).
- **API Java**: `Comments` (có `asSystem(...)`), `Activities`/`ActivityRecorder`, `Todos`, `Tags`, `Timeline`, `Subjects`.
- **REST** (`/api/v1/collaboration`): `/subjects/{type}/{id}/threads|timeline|todos|tags|followers|read`,
  `/threads/{id}/**`, `/comments/{id}/**`, `/subjects/summary`, `/todos/**`, `/tags/**`, `/mentions/suggest`,
  `/admin/**`, `/internal/import|activities|access`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-collaboration</artifactId>
</dependency>
```

```yaml
# META-INF/vyforge/collaboration/subject-types/crm.yml
subject-types:
  crm:order:
    name: { vi: "Đơn hàng", en: "Order" }
    access: { mode: ACTIONS, read: crm:order:Get, contribute: crm:order:Comment, write: crm:order:Update }
    visibilities: [INTERNAL]
```

```java
comments.createThread(SubjectRef.of("crm:order", id),
        NewThread.comment(Visibility.INTERNAL, VfDocInput.markdown("Nhờ [@Bình](mention:USER:binh) kiểm")));
todos.create(NewTodo.of(subject, "approval", "Duyệt hạn mức", Assignee.user(managerSub))
        .due(DueSpec.relative(2, DueUnit.BUSINESS_DAY)));
```

Frontend: `<Chatter subject={{ type: 'crm:order', id }} … />` từ `@vyforge/collaboration`, Server Action từ
`@vyforge/collaboration/server`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.collaboration.enabled` | `true` | Tắt module |
| `vyforge.collaboration.mode` | `embedded` | `remote`: app không có bảng, gửi activity qua Kafka |
| `vyforge.collaboration.schema.migrate` | `true` | Flyway riêng; tắt thì chạy `META-INF/vyforge/sql/collaboration.sql` |
| `vyforge.collaboration.web.base-path` | `/api/v1/collaboration` | Gốc REST |
| `vyforge.collaboration.max-body-chars` / `max-mentions` / `max-group-expansion` | 20.000 / 50 / 200 | |
| `vyforge.collaboration.time-zone` | `Asia/Ho_Chi_Minh` | |
| `vyforge.collaboration.moderation.authority-normal|urgent|critical` | 24h / 6h / 3h | SLA yêu cầu gỡ |
| `vyforge.collaboration.moderation.user-complaint` | 48h | |
| `vyforge.collaboration.timeline.audit.enabled` | (cần xác nhận) | Đưa audit log vào timeline |
| `vyforge.collaboration.ingest.enabled` / `topic` | `false` / `collaboration-activities` | Server nhận activity từ app remote |
| `vyforge.collaboration.integrations.<notification|realtime|files|webhooks|privacy>.enabled` | `true` | Tắt từng adapter |

Tenant ghi đè qua `/admin/settings` và `/admin/subject-types/{type}/settings` (chỉ thu hẹp phần an toàn).

## 6. Tích hợp với module khác

| SPI | Adapter có sẵn / module |
|---|---|
| `CollaborationNotifier` | `NotificationCollaborationNotifier` (**notification**): mention, trả lời, theo dõi, việc được giao/sắp hạn/quá hạn |
| `LivePublisher` | `RealtimeLivePublisher` (**realtime**): kênh `collab:<service>/<resource>/<id>`, chỉ mang id |
| `AttachmentGateway` | `FilesAttachmentGateway` (**files**): file gắn vào `collaboration:comment/<id>` |
| `CollaborationErasure` | `CollaborationPersonalData` (**privacy-dsr**): xuất/ẩn danh comment, tôn trọng legal hold |
| webhooks | catalog `collaboration.comment.created`, `.todo.completed`, `.tag.assigned` (payload chỉ id; theo `egress.webhooks`) |
| `TodoDueCalculator` | lịch làm việc (**calendar**); mặc định bỏ T7, CN |
| `MentionTargetResolver` | `ORG_UNIT` (**organization**) |
| `ContentClassifier`, `CommentInterceptor` | phân loại nội dung (**ai**), luật riêng của app |
| `UserDirectory`, `PrincipalRoles`, `TimelineContributor` | tên hiển thị, role, nguồn timeline ngoài |
| automation | `todo.create`, `comment.post` gọi `Todos`/`Comments.asSystem` (adapter nằm ở automation) |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded`; `remote` một phần — app chỉ còn `ActivityRecorder` (`RemoteActivityRecorder` gửi Kafka sau
  commit, cần `spring-boot-starter-kafka`), frontend gọi REST của platform-services; server bật `ingest.enabled`.
- **IAM**: quyền chính theo action của bản ghi cha; action riêng của module: `collaboration:comment:ModerateAny|ViewHistory`,
  `collaboration:thread:Lock`, `collaboration:mention:Group`, `collaboration:follower:Manage`,
  `collaboration:todo:ListAll|ManageAny`, `collaboration:todo-type:Manage`, `collaboration:tag:Manage`,
  `collaboration:moderation:List|Decide`, `collaboration:report:List`, `collaboration:takedown:List|Manage`,
  `collaboration:filter:List|Manage`, `collaboration:restriction:List|Manage`, `collaboration:settings:Get|Update`,
  `collaboration:stats:Get`, `collaboration:task:Manage`, `collaboration:internal:Access`.
- **Sự kiện** (`com.vyforge.platform.collaboration.events`, không mang nội dung comment): `CommentCreated`,
  `CommentEdited`, `CommentDeleted`, `CommentHidden`, `CommentRestored`, `UserMentioned`, `ThreadLocked`,
  `ThreadResolved`, `ThreadReopened`, `ActivityRecorded`, `TodoCreated`, `TodoAssigned`, `TodoCompleted`,
  `TodoCancelled`, `TodoDueSoon`, `TodoOverdue`, `TagAssigned`, `TagUnassigned`, `SubjectFollowed`,
  `SubjectUnfollowed`, `ContentReported`, `ModerationDecided`, `TakedownOpened`, `TakedownOverdue`,
  `TakedownCompleted`.
- **Giấy phép**: module `licensed` (`collaboration`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: trao đổi, timeline `@Tracked`, việc cần làm, nhãn, theo dõi, kiểm duyệt + yêu cầu gỡ, adapter
  notification/realtime/files/privacy-dsr/webhooks, zone admin, editor rich text.
- Chế độ remote còn giới hạn: gửi Kafka trực tiếp chưa qua outbox (app dừng giữa commit và gửi thì mất activity); chưa
  có client remote cho `Comments`, `Todos`, `Tags`, `Timeline`, `Subjects`; chưa có `@Tracked`/`ActivityMapper` ở app
  remote.
- Test `ModerationApiTests` hay timeout khi máy build tải nặng (lỗi môi trường, chạy riêng thì xanh).
