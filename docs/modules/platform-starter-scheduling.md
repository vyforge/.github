# platform-starter-scheduling

## 1. Mục đích

Hai khối cho việc chạy ngầm khi app có nhiều replica (B22):

- **`@ClusterScheduled`**: job định kỳ chạy đúng một nơi, có thể lặp theo từng tenant.
- **`DurableTaskQueue`**: hàng đợi việc nền bền vững trong PostgreSQL, có retry, backoff, khoá, tiến độ, tạm dừng và API
  quản trị.

Mọi việc như quét virus, gửi tin, webhook, xuất file hay đối soát của các starter đều chạy trên hai khối này.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-scheduling` (kéo `platform-core`; `-jpa`, `-web` tuỳ chọn) |
| Package web | `@vyforge/admin-kit` 0.2.0 (`TaskMonitor` cho tenant admin); `@vyforge/ops-console` 0.1.0 cho người vận hành |
| Zone admin | `TaskMonitor` trong khu của từng module; `ops-console-admin` xem xuyên tenant |
| App liên quan | mọi backend, `platform-services`, `platform-auth-server` (job dọn dẹp) |

## 3. Tính năng chính

- **`@ClusterScheduled(name, cron | fixedDelay, zone, lockAtMostFor, lockAtLeastFor, perTenant)`**:
  - Chạy trên ShedLock JDBC, giờ của khoá lấy từ database.
  - Cấu hình sai (thiếu hoặc trùng tên, cron sai…) thì app không khởi động.
  - `perTenant = true`: chạy một lần cho mỗi tenant của `TenantDirectory`; một tenant lỗi không chặn các tenant sau.
- **Lịch sử và điều khiển job**:
  - Lịch sử lưu ở `vyforge_job_run`, outcome `SUCCEEDED`, `FAILED`, `PARTIAL` hoặc `ABANDONED`.
  - Tạm dừng job (`vyforge_job_control`), chạy tay (`ClusterJobRegistry.runNow`), nhả khoá kẹt.
- **`DurableTaskQueue`**:
  - Bảng `vyforge_task`, lấy việc bằng `FOR UPDATE SKIP LOCKED`.
  - `TaskType`, `TaskHandler`, `enqueue(type, payload, EnqueueOptions)`: ghi cùng transaction nghiệp vụ, có
    `idempotencyKey` theo tenant và queue.
  - Kết quả của handler:
    - chạy xong → `SUCCEEDED`;
    - ném exception → retry với backoff, hết lượt thì `DEAD`;
    - `RetryTaskLaterException(delay)` → chạy lại sau `delay`;
    - `PermanentTaskException` → `DEAD` ngay.
  - Khoá có `extendLock`. Khoá mất → `TaskLockLostException`, kết quả lần chạy đó không được ghi.
  - Tiến độ: `reportProgress(done, failed, total)`.
  - Mang ngữ cảnh sang việc nền qua `TaskContextPropagator`; có sẵn propagator `mdc`.
  - Giới hạn đồng thời theo queue và theo tenant trên cả cluster (`max-running`, `max-running-per-tenant`).
- **Quản trị**:
  - `TaskAdmin` theo tenant: `list`, `find`, `retry`, `cancel`, `backlog`, `retryAll`, `cancelAll`.
  - `TaskOps` xuyên tenant: nhóm lỗi theo `last_error_type`, retry hàng loạt giãn nhịp `spreadOver`, purge.
  - `QueueControl`: tạm dừng cả queue, một tenant hoặc một kind, tự tiếp tục được.
- **Job của jpa-extras** (khi có `-jpa`): `SoftDeletePurgeJob` và `EncryptionRotationJob`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-scheduling</artifactId>
</dependency>
```

```yaml
spring.flyway.locations: classpath:db/migration,classpath:META-INF/vyforge/db/scheduling   # bảng shedlock
```

```java
@ClusterScheduled(name = "invoice-reminder", cron = "0 0 8 * * *", zone = "Asia/Ho_Chi_Minh", perTenant = true)
void remind() { ... }

@Bean TenantDirectory tenants(JdbcClient jdbc) {
    return () -> jdbc.sql("select id from tenant where active").query(String.class).list();
}

public static final TaskType<ScanPayload> SCAN = TaskType.of("files", "scan", ScanPayload.class);
tasks.enqueue(SCAN, new ScanPayload(blobId), EnqueueOptions.defaults().idempotencyKey("scan:" + blobId));
```

Bảng `vyforge_task` do schema `task` của starter tạo khi có `-jpa`. Không có `-jpa` thì chạy
`META-INF/vyforge/sql/task.sql`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.scheduling.enabled` | `true` | `false`: không lên lịch job (dùng trong test) |
| `vyforge.scheduling.zone` | múi giờ JVM | Múi giờ mặc định của cron; nên đặt `Asia/Ho_Chi_Minh` |
| `vyforge.scheduling.default-lock-at-most-for` | `10m` | Phải dài hơn lần chạy lâu nhất |
| `vyforge.scheduling.history.retention` | `30d` | Thời gian giữ `vyforge_job_run` |
| `vyforge.task.enabled` / `.poll-interval` | `true` / `1s` | `false`: không chạy handler, vẫn enqueue được |
| `vyforge.task.defaults.concurrency` | `4` | Số việc song song mỗi instance cho mỗi queue |
| `vyforge.task.defaults.max-attempts` / `.lock-duration` | `8` / `5m` | |
| `vyforge.task.defaults.backoff.initial` / `.multiplier` / `.max` | `30s` / `2` / `1h` | |
| `vyforge.task.queues.<queue>.max-running` / `.max-running-per-tenant` | `0` / `0` | Trần trên cả cluster; `0` = không giới hạn |
| `vyforge.task.keep-succeeded` / `.keep-dead` | `7d` / `90d` | Dọn việc cũ; `keep-dead=0` thì giữ mãi |
| `vyforge.task.context.mdc-keys` | `request_id` | Khoá MDC mang sang việc nền |

## 6. Tích hợp với module khác

- `platform-starter-ops` bọc `TaskOps`, `QueueControl` và `ClusterJobRegistry` bằng REST.
- `platform-starter-web` dùng scheduling cho bulk chạy nền và `TaskAdminEndpoint`.
- Queue của các starter: `files`, `notification`, `webhooks`, `export`, `payment`, `ops`… (xem tài liệu từng module).
- Luật kiến trúc `NO_PLAIN_SCHEDULED` (`platform-test`).

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Mỗi backend có bảng task riêng.
- Quyền theo module (vd. `files:task:Retry`); người vận hành dùng `ops:task:*`, `ops:job:*`, `ops:queue:Pause`.
- Metric:
  - Job: `vyforge.scheduling.runs{job,outcome}`, `.skipped`, `.tenant.failures`, `.last.success`,
    `.expected.interval`, `.paused`.
  - Việc nền: `vyforge.task.enqueued`, `.executions{outcome}`, `.backlog{status}`, `.oldest.due.age`,
    `.lock.lost`.

## 8. Trạng thái & còn mở

- Ổn định.
- 0.16 sửa lỗi: điều kiện tạm dừng queue nay kiểm ngay trong câu claim. Trước đó worker có thể nhận việc của queue vừa
  bị dừng.
- Giới hạn đã biết:
  - Không dùng virtual thread thì scheduler chỉ có 1 thread.
  - Job `perTenant` dùng chung một khoá cho cả lượt, nên `lockAtMostFor` phải đủ cho mọi tenant.
- Để bản sau (ops-console §9.4):
  - đổi cấu hình queue lúc chạy từ console;
  - partition `vyforge_job_run` theo tháng.
