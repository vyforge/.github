# platform-starter-realtime

## 1. Mục đích

Đẩy tin từ server tới client đang mở (K15): thông báo mới, tiến độ job, bản ghi vừa đổi, ai đang xem cùng hồ sơ.
Mặc định dùng SSE; WebSocket khi cần hai chiều.

- Topic luôn thuộc tenant của token.
- Quyền subscribe kiểm bằng IAM, và kiểm lại khi IAM đổi.
- Kết nối lại thì replay theo `Last-Event-ID`.

Realtime là **kênh báo tin, không phải nguồn dữ liệu**: client nhận tín hiệu rồi gọi API có kiểm quyền để lấy dữ liệu
thật.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-realtime` (kéo `-authz`; `-jpa`, `-ratelimit`, `spring-boot-starter-data-redis` tuỳ chọn) |
| Package web | `@vyforge/realtime` 0.1.0: `createRealtimeClient`, `/react` (`useTopic`, `useLatest`, `usePresence`, `ConnectionIndicator`), `/query` (`useInvalidateOnEvent`), `/server` (`createRealtimeRoute`, `createTicketAction`), `/websocket` |
| Zone admin | chưa có zone `realtime-admin` (thiết kế §9.3 để bản sau). Giới hạn của tenant sửa qua REST `/admin/settings` |
| App liên quan | `platform-services` (module `realtime`, nên tách deployment `platform-services-realtime`), `platform-gateway` |

## 3. Tính năng chính

- **Gửi tin**:
  - `RealtimePublisher.publish(topic, type, data)` gửi sau khi transaction commit, rollback thì bỏ.
  - `toUser(sub, …)` gửi cho một người dùng.
  - `job(id).owner(sub).progress(…)` báo tiến độ job.
  - Hoặc đánh dấu event Spring bằng `@RealtimeEvent`.
  - Gửi là best-effort: `RealtimeUnavailableException`, hoặc `publishQuietly` chỉ ghi log.
- **Topic `<namespace>:<path>`**:
  - `user`: chỉ chính người dùng.
  - `tenant`: cần `realtime:tenant-topic:Subscribe`.
  - `job`: chủ job, hoặc `realtime:job:Subscribe`.
  - `res`: action `Get` trên resource tương ứng, vd. `res:crm:order/42` cần `crm:order:Get`.
  - Namespace riêng khai bằng `TopicNamespace.builder(...)`: chế độ recovery, presence, cho client publish (qua
    `ClientMessageHandler`), cờ dữ liệu nhạy cảm, action subscribe.
- **Kết nối**:
  - `GET /api/v1/realtime/stream?topic=…` (SSE).
  - `POST /tickets`: vé dùng một lần 30 s cho WebSocket và SSE trực tiếp.
  - `GET /ws?ticket=` (WebSocket, tắt mặc định).
  - `GET /presence`.
  - Kết nối đóng trước `exp` của token và sau `max-connection-lifetime`.
- **Replay**: lịch sử 5 phút / 5.000 tin mỗi tenant (trần 1 giờ). Không vá được khoảng mất thì gửi `reset`. Tin điều
  khiển: `reset`, `presence`, `unsubscribed`, `bye` (`token_expiring`, `slow_consumer`, `shutdown`…).
- **Fan-out giữa replica**: Valkey (Stream + pub/sub, một instance hoặc Sentinel), Kafka, hoặc bộ nhớ khi chỉ có một
  replica.
- **Giới hạn theo tenant**: số kết nối, kết nối mỗi người dùng, topic mỗi kết nối, tin/giây (bảng
  `realtime_tenant_settings`). Vượt thì 429 `realtime.connection_limit` hoặc `realtime.publish_rate_exceeded`.
- **REST quản trị**:
  - `/api/v1/realtime/admin/settings` (`If-Match`)
  - `/admin/connections`, đóng kết nối của một người dùng
  - `/admin/namespaces`, `/admin/info`
  - `/admin/ops/stats` (người vận hành, xuyên tenant)
- **Hook**: `RealtimeInterceptor` (`beforePublish`, `beforeSubscribe`, `onDisconnect`).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-realtime</artifactId>
</dependency>
<dependency>                                   <!-- nhiều replica -->
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring.mvc.async.request-timeout: -1   # luồng SSE chạy lâu
```

```java
realtime.publish("res:orders:branch/" + branchId, "order.created", new OrderSummary(id, total));

@RealtimeEvent(topic = "'res:orders:branch/' + branchId()", type = "order.created")
public record OrderCreated(String tenantId, UUID orderId, String branchId) {}
```

```tsx
// Next.js: route BFF chuyển SSE kèm token, client dùng hook
const { data } = useTopic('user:inbox');
useInvalidateOnEvent('res:orders:branch/1', ['orders']);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.realtime.enabled` | `true` | `false`: tắt cả module (không bean, không migration) |
| `vyforge.realtime.mode` | `hub` | `publisher`: worker chỉ publish, hub nằm ở service khác cùng Valkey; `off` |
| `vyforge.realtime.broker` | `auto` | `valkey` khi có Redis, không có thì `memory`; hoặc `kafka` |
| `vyforge.realtime.cluster` | `spring.application.name` | Nhóm replica dùng chung luồng tin |
| `vyforge.realtime.heartbeat` | `15s` | Phải nhỏ hơn timeout của mọi proxy trên đường đi |
| `vyforge.realtime.max-connection-lifetime` | `1h` | |
| `vyforge.realtime.transports.websocket.enabled` | `false` | Kiểm `Origin` theo `allowed-origins` |
| `vyforge.realtime.queue.max-events` / `.max-bytes` | `256` / `1MB` | Hàng chờ mỗi kết nối; vượt thì đóng `slow_consumer` |
| `vyforge.realtime.defaults.max-connections` / `.max-connections-per-user` | `2000` / `10` | Giới hạn mặc định của tenant |
| `vyforge.realtime.defaults.history-ttl` / `.history-max-events` | `5m` / `5000` | |
| `vyforge.realtime.max-message-bytes` | `16KB` | |

## 6. Tích hợp với module khác

- `platform-starter-authz`: kiểm quyền subscribe và kiểm lại khi nhận `vyforge:iam:changed`.
- Starter có phụ thuộc tuỳ chọn tới realtime: collaboration, workflow. notification đẩy inbox in-app qua SPI
  `InboxRealtimeBridge` (topic `user:inbox`).
- `platform-gateway`:
  - SSE đi qua gateway nhờ Apache HttpClient 5 (read-timeout tính giữa hai gói).
  - Rule `realtime-stream` mặc định 10 lần mở luồng mỗi phút.
  - WebSocket chưa qua gateway được (cần Spring Cloud Gateway 5.1), phải dùng route ingress riêng.
- Ingress: `proxy_buffering off`, `proxy_read_timeout` ≥ 60s.

## 7. Chế độ triển khai, quyền, sự kiện

- Có hai cách chạy:
  - **Nhúng trong app** (`mode=hub`).
  - **Tách hub**: worker chạy `mode=publisher`; hub là `platform-services-realtime` (module `realtime`, route
    `/api/v1/realtime/**`, biến `REALTIME_URL` ở gateway).
- Quyền: `realtime:tenant-topic:Subscribe`, `realtime:job:Subscribe`, `realtime:presence:List`,
  `realtime:connection:List`, `realtime:connection:Disconnect`, `realtime:settings:Get`, `realtime:settings:Update`.
- Sự kiện Spring: `RealtimeConnectionOpened`, `RealtimeConnectionClosed`, `RealtimeSubscriptionRejected`,
  `PresenceChanged`, `RealtimeSettingsChanged`.
- Metric `vyforge.realtime.*` (connections, messages, latency, lag, replay, queue depth). Health `realtimeBroker` không
  nằm trong readiness.
- Module có phí (`licensed: true`, từ 0.13.0).

## 8. Trạng thái & còn mở

- Đã làm từ 0.13.
- Để bản sau (thiết kế §9.3):
  - broker Centrifugo (ngưỡng > 20.000 kết nối hoặc > 2.000 tin/giây);
  - chia sẻ kết nối giữa nhiều tab;
  - wildcard topic;
  - `RemoteTopicAuthorizer`;
  - broker PostgreSQL `LISTEN/NOTIFY`;
  - màn quản trị trong platform-admin.
- Chưa hỗ trợ Valkey Cluster. Với broker Kafka, dedup và tin mới nhất nằm trong bộ nhớ từng replica.
- Không làm adapter Mercure vì giấy phép AGPL.
- Điểm chờ người dùng xác nhận (thiết kế §10.3):
  - ngưỡng chuyển sang Centrifugo có hợp với quy mô pilot không;
  - việc thêm Apache HttpClient 5 vào gateway đã làm.
