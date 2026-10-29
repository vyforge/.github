# platform-starter-messaging

## 1. Mục đích

Đặt chuẩn cho event giữa module và giữa service, để event không mất và không bị xử lý hai lần khi chạy nhiều replica:

- outbox của Spring Modulith, có tự bảo trì;
- Kafka retry rồi dead-letter;
- consumer idempotent;
- tenant đi theo header.

Starter cũng cung cấp phần dữ liệu cho ops-console: xem, gửi lại hoặc cất event của outbox, và nạp dead-letter rồi replay.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-messaging` (kéo `platform-core`) |
| Package web | không có. Giao diện vận hành nằm trong `@vyforge/ops-console` |
| Zone admin | qua `ops-console-admin` (outbox, dead-letter) |
| App liên quan | mọi backend có event, `platform-services` |

## 3. Tính năng chính

- **`IntegrationEvent`** và outbox Spring Modulith (`event_publication`). `@Externalized("topic::#{key}")` đẩy event ra
  Kafka. Event nên implement `TenantAwareEvent`.
- **Bảo trì outbox**:
  - Xoá event đã xong (`completion-mode=delete`).
  - Event treo quá `stale-after` → FAILED, rồi tự gửi lại theo `retry-delay` và `max-attempts`.
  - An toàn khi nhiều replica. Không gửi lại hàng loạt lúc khởi động.
- **Kafka**:
  - Producer idempotent.
  - Retry rồi dead-letter `<topic>-dlt`. Lỗi 4xx của `PlatformException` sang DLT ngay, không retry.
  - Tenant đi theo header `tenant_id`. Listener có `@RequiresTenant` mà bản ghi thiếu header thì sang DLT ngay.
  - `TenantRecordInterceptor` và `TenantBatchInterceptor` khôi phục tenant cho consumer.
- **Consumer idempotent**:
  - Consumer ghi database: `ProcessedMessages.markProcessed(consumer, eventId)` trong transaction nghiệp vụ. Bảng
    `processed_messages` tự dọn định kỳ.
  - Consumer không ghi database: `RedisMessageDeduplicator.firstDelivery(...)` / `release(...)`.
- **Ops** (`messaging.ops`):
  - `OutboxOps`: đọc outbox, gửi lại event đã bỏ cuộc, cất (park) event sang `vyforge_outbox_parked`, khôi phục.
  - `DeadLetterIngester`: nạp `<topic>-dlt` của các group thuộc app vào `vyforge_dead_letter`.
  - Replay về topic gốc kèm header `vyforge-replay-group`. `ReplayTargetInterceptor` làm group khác cùng topic bỏ qua
    bản ghi đó.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-messaging</artifactId>
</dependency>
```

Tạo bảng theo SQL mẫu `processed_messages` và `event_publication` của starter. Có `platform-starter-jpa` thì schema
`messaging` tự tạo bảng ops.

```java
@Externalized("customer-events::#{customerId()}")
public record CustomerCreated(String tenantId, UUID customerId) implements TenantAwareEvent {}

@KafkaListener(topics = "payment-events", groupId = "orders")
@RequiresTenant
@Transactional
void on(PaymentCaptured event) {
    if (!processed.markProcessed("orders", event.eventId())) return;
    ...
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.messaging.retry.attempts` / `.interval` | `3` / `1s` | Số lần retry trước khi sang `<topic>-dlt` |
| `vyforge.messaging.processed-messages.retention` | `7d` | Phải dài hơn thời gian một event có thể bị giao lại |
| `spring.modulith.events.completion-mode` | `delete` | Đặt `archive` nếu cần giữ vết |
| `vyforge.messaging.outbox.stale-after` / `.retry-delay` / `.max-attempts` | `5m` / `1m` / `10` | Bảo trì outbox |
| `vyforge.messaging.outbox.parked-retention` | `30d` | Thời gian giữ event đã cất |
| `vyforge.messaging.dlt.ingest-enabled` / `.topic-pattern` / `.owned-groups` | `true` / `.*-dlt` / rỗng | Nạp dead-letter |
| `vyforge.messaging.dlt.payload-retention` / `.retention` / `.replay-rate` | `14d` / `30d` / `500` | Thời gian giữ payload, metadata và tốc độ replay |
| `vyforge.messaging.redis-deduplication.retention` | `7d` | Thời gian giữ khoá của `RedisMessageDeduplicator` |

## 6. Tích hợp với module khác

- `platform-starter-ops` bọc `OutboxOps` và `DeadLetterIngester` bằng REST `/api/v1/ops/admin/outbox|dead-letters`.
- Hầu hết starter nghiệp vụ phát event qua outbox.
- `platform-test`:
  - `SharedKafka`/`KafkaTestcontainer`. Mỗi context test đặt `vyforge.messaging.dlt.topic-pattern` theo topic của
    mình.
  - App tự khai `RecordInterceptor` thì gộp với interceptor của platform bằng `CompositeRecordInterceptor`.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Kafka là tuỳ chọn: không có Kafka thì event chỉ đi trong process qua outbox.
- Không có action IAM riêng; quyền vận hành là `ops:outbox:*` và `ops:deadletter:*` của `-ops`.
- Metric: `vyforge.outbox.publications{status}`, `vyforge.outbox.oldest.incomplete.age`, `vyforge.outbox.parked`,
  `vyforge.dlt.records{topic,status}`, `vyforge.dlt.ingested{topic}`.

## 8. Trạng thái & còn mở

- Ổn định. Phần ops có từ 0.13–0.15.
- Thiếu bảng ops thì ingester không chạy.
- Để bản sau (thiết kế ops-console §9.4):
  - partition `vyforge_dead_letter` theo tháng;
  - sửa payload rồi gửi lại (cần hai người duyệt).
