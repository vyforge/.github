# platform-starter-ops

## 1. Mục đích

ops-console cho người vận hành: thấy và xử lý tay mọi thứ chạy ngầm trên **mỗi** backend, gồm:

- việc nền `DurableTaskQueue`;
- job `@ClusterScheduled`;
- outbox Spring Modulith;
- dead-letter Kafka;
- cache theo tenant.

Mọi thao tác ghi đều phải có lý do và được audit. Thao tác nguy hiểm đòi xác thực lại (step-up). Người vận hành không
phải vào database để retry hay replay.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-ops` (cần `-web`, `-security`, `-authz`, `-audit`, `-privacy`; `-scheduling`, `-messaging` tuỳ chọn) |
| Package web | `@vyforge/ops-console` 0.1.0: `createOpsAdminClient`, `loadOpsTargets`, `createTokenExchanger`, component `/admin` |
| Zone admin | `apps/ops-console-admin` (zone `/m/ops-console`, chỉ người vận hành, cổng dev 3103) |
| App liên quan | mọi backend có việc ngầm, `platform-services` (luôn bật) |

## 3. Tính năng chính

- **Không có bảng riêng.** Dữ liệu nằm ở starter sở hữu nó:
  - `-scheduling`: `TaskOps`, `QueueControl`, lịch sử job;
  - `-messaging`: `OutboxOps`, `DeadLetterIngester`;
  - `platform-core`: `TenantCacheAdmin`.

  Thiếu starter nào thì không có nhóm API tương ứng. `GET /info` trả `capabilities` (`tasks`, `jobs`, `outbox`,
  `deadletter`, `cache`).
- **REST `/api/v1/ops/admin/**`** (apiVersion 1):
  - `info`, `overview`
  - việc nền: `tasks` (lọc xuyên tenant, nhóm lỗi, payload đã che, retry, cancel, bulk retry/cancel/purge),
    `bulk/{taskId}`
  - queue: `queues`, pause/resume theo queue, tenant hoặc kind
  - job: `jobs` (lịch sử, chạy tay cho tối đa 500 tenant, pause, nhả khoá)
  - outbox: `outbox` (summary, gửi lại, cất, khôi phục, xoá event đã cất)
  - dead-letter: `dead-letters` (replay nhắm đúng consumer group, discard)
  - cache: `caches` (xoá theo tên và tenant, xoá cả tenant)
  - `audit`
- **An ninh ba lớp**:
  - Token phải thuộc `operator-tenant`, có một role trong `operator-roles`, và `amr` có `mfa`.
  - Sau đó kiểm action `ops:<resource>:<Verb>` qua `Authz`.
  - Thiếu điều kiện → 403 `ops.operator_required`.
- **Step-up** (RFC 9470): purge, discard, park, nhả khoá, đọc payload, xoá cache mọi tenant, bulk từ 10.000 dòng.
  `auth_time` phải trong 15 phút, không thì 401 `ops.step_up_required`.
- **Lý do** 10–500 ký tự cho mọi thao tác ghi, ghi vào `AuditLog`, kể cả mỗi lần xem payload. Payload luôn được che PII.
- **Bulk** cùng dạng `BulkRequest` của admin-kit:
  - `ids` (≤ 500) chạy ngay;
  - hoặc `filter` + `expectedCount`, chạy nền ở queue `ops`;
  - retry giãn nhịp `spreadOver` (≤ 6 giờ).
- **Không qua gateway**: gateway chặn `/api/v*/ops/**`. Zone gọi URL nội bộ của từng backend bằng token đã đổi theo
  audience của backend đó (RFC 8693).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-ops</artifactId>
</dependency>
```

```yaml
vyforge:
  ops:
    operator-tenant: default
    operator-roles: [platform-admin]
    require-mfa: true          # chỉ tắt ở dev
    target: crm                # tên backend trong OPS_TARGETS, ghi vào audit
```

Ở platform-admin, khai backend vào `OPS_TARGETS` (JSON do infra sinh) của zone `ops-console-admin`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.ops.enabled` | `true` | |
| `vyforge.ops.operator-tenant` / `.operator-roles` | `default` / `[platform-admin]` | Ai là người vận hành |
| `vyforge.ops.require-mfa` | `true` | |
| `vyforge.ops.step-up-max-age` | `15m` | |
| `vyforge.ops.bulk.sync-limit` / `.max-items` / `.max-spread` / `.step-up-threshold` | `500` / `200000` / `6h` / `10000` | |
| `vyforge.ops.payload.max-view-bytes` | `262144` | |
| `vyforge.task.queues.ops.max-running` | `2` | Bulk chạy nền |

## 6. Tích hợp với module khác

- **`OpsQueueSource`**: module có hàng đợi riêng (vd. webhooks `webhook_delivery`) khai bean để hiện số liệu và link
  trong tổng quan.
- **`META-INF/vyforge/ops/<module>.yml`**: nhãn queue/kind, runbook, gợi ý xử lý theo loại lỗi.
- **`OpsActionGuard`**: chặn thao tác theo chính sách của bản triển khai (vd. đòi số ticket cho queue `payment`), trả
  422 `ops.action_rejected`.
- **`platform-starter-licensing`**: ops luôn bật và không tính phí (descriptor `licensed: false`).

## 7. Chế độ triển khai, quyền, sự kiện

- Nhúng trong mỗi backend. Zone `ops-console-admin` gộp nhiều backend ("target"); target chết thì các target khác vẫn
  hiện.
- Quyền: `ops:overview:Get`, `ops:task:*`, `ops:queue:Pause`, `ops:job:*`, `ops:outbox:*`, `ops:deadletter:*`,
  `ops:cache:*`, `ops:audit:List`. Vai gợi ý: `ops-viewer`, `ops-operator`, `ops-admin`.
- Sự kiện `OpsActionPerformed` cho mọi thao tác ghi. Counter `vyforge.ops.actions{action,outcome}`.

## 8. Trạng thái & còn mở

- Đã làm từ 0.13–0.15. 0.16.1 không đổi API.
- Để bản sau (thiết kế ops-console §9.4):
  - đổi cấu hình queue lúc chạy;
  - partition bảng DLT và lịch sử job;
  - tổng quan realtime thay polling;
  - bản ops cho tenant admin;
  - sửa payload rồi gửi lại (hai người duyệt);
  - xuất việc lỗi ra Excel.
- Token exchange ở auth-server phải giữ `amr` và `auth_time` (thiết kế §11.3 mục 4). Hiện auth-server giữ `amr` khi
  đổi token; việc giữ `auth_time` (cần xác nhận).
- NetworkPolicy chỉ mở API ops cho namespace của platform-admin; việc này phụ thuộc chart triển khai.
