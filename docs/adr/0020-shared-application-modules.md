# ADR-0020: Module ứng dụng dùng chung

- **Trạng thái:** Chấp nhận dần theo từng module (mỗi module một mục). Thuộc kế hoạch trong
  [MODULE_PLAN.md](../MODULE_PLAN.md) mục 4 và 8.
- **Nền chung:** module-conventions của đợt P1; khối dùng chung `SecretCipher`, `DurableTaskQueue`,
  `StarterSchemaMigrator` (platform-java 0.12.1); đóng gói độc lập và admin microfrontend theo
  [platform-services](../design/platform-services.md).

Mỗi module ghi: quyết định, thư viện và license, điểm nối với module khác, việc còn mở. Thiết kế chi tiết nằm ở
`docs/design/<module>.md`.

## realtime

- **Thiết kế:** [realtime.md](../design/realtime.md). **Phát hành:** `platform-starter-realtime` (platform-java 0.13.0),
  `@vyforge/realtime` 0.1.0 (platform-web); `platform-gateway` thêm Apache HttpClient 5 (branch `feat/realtime-proxy`).

### Quyết định

1. **Realtime là kênh báo tin, không phải nguồn dữ liệu.** Tin chỉ giữ ngắn hạn (mặc định 5 phút / 5.000 tin mỗi tenant,
   trần 1 giờ). Không vá được khoảng mất tin thì client nhận `reset` và tải lại từ API.
2. **SSE mặc định (K15), WebSocket là transport thứ hai trên cùng hub,** tắt mặc định. Lý do: platform-gateway
   (Spring Cloud Gateway Server WebMVC 5.0) chưa proxy được WebSocket, phải chờ 5.1 (Spring Cloud 2026.0). Không dùng
   STOMP: simple broker không chạy nhiều node, còn broker relay cần thêm RabbitMQ.
3. **Topic `<namespace>:<path>`, tenant không nằm trong tên** (luôn lấy từ token). Namespace khai báo bằng code, mang
   quyền subscribe (action IAM + VRN), cách recovery (`STREAM`, `CACHE`, `NONE`), presence, publish từ client, cờ nhạy
   cảm. Có sẵn `user`, `tenant`, `job`, `res`.
4. **Một luồng tin cho mỗi tenant, `Last-Event-ID` là vị trí trong luồng** (học Mercure). Một kết nối gộp nhiều topic
   nên chỉ cần một vị trí.
5. **Broker theo deployment, không theo tenant** (đã duyệt): `memory` (một replica), `valkey` (Stream + pub/sub có shard;
   publish bằng một script Lua; id đã cắt ghi vào `…:trimmed` để phát hiện mất tin), `kafka` (key là tenant). Pub/sub
   đứt rồi nối lại thì hub vá tin từ Stream cho từng kết nối.
6. **Publish best-effort sau commit** (đã duyệt), không đi qua outbox. Tin phải tới chắc chắn thì publish từ listener
   outbox kèm `dedupKey`.
7. **Quyền kiểm khi subscribe và kiểm lại khi IAM đổi.** Kết nối đóng trước `exp` của token. Không cache quyền suốt kết
   nối (tránh lỗi của Supabase Realtime).
8. **Vé dùng một lần, sống 30 giây** cho WebSocket và SSE thẳng, thay cho access token trên URL.
9. **Cấu hình theo tenant** (`realtime_tenant_settings`, `realtime_namespace_override`) qua `StarterSchemaMigrator`
   (`flyway_schema_history_realtime`). Không có secret trong DB.
10. **Không phụ thuộc cứng vào module P1 nào.** Hợp đồng với notification, export, files, payment, features là topic,
    type và schema (thiết kế §5.3). Bên kia publish qua `ObjectProvider<RealtimePublisher>` hoặc `@RealtimeEvent`.
11. **Ngưỡng chuyển sang Centrifugo** (adapter để bản sau): > 20.000 kết nối đồng thời hoặc > 2.000 tin/giây.

### Thư viện

| Thư viện | Phiên bản | License | Dùng cho |
|---|---|---|---|
| Spring MVC `SseEmitter`, Spring WebSocket | Spring Framework 7.0 (Boot 4.1) | Apache-2.0 | Transport |
| Spring Data Redis (Lettuce) | theo Boot 4.1 | Apache-2.0 | Broker Valkey |
| kafka-clients | theo Boot 4.1 | Apache-2.0 | Broker Kafka |
| Apache HttpClient 5 | theo Boot 4.1 | Apache-2.0 | platform-gateway: read-timeout giữa hai gói cho SSE |
| eventsource-parser | 4.1.3 | MIT | `@vyforge/realtime`: đọc SSE trên `fetch` |
| Centrifugo (tham khảo, adapter sau) | v6.10.2 | Apache-2.0 | Khi vượt ngưỡng |
| Mercure (không dùng) | v1.1.5 | AGPL-3.0 | Hub AGPL, quyền trong JWT khó khớp IAM có `Condition` |

### Còn mở

- Adapter Centrifugo và transport `@vyforge/realtime/centrifugo`.
- App zone `platform-admin/apps/realtime-admin` (`pnpm create-admin-zone realtime`) sau khi forge-37 có template.
  Component đã có ở `@vyforge/realtime/admin`.
- Gateway: WebSocket qua gateway khi lên Spring Cloud 2026.0. Rule mở luồng hiện vẫn tính thêm vào bucket chung, vì
  `-ratelimit` chưa loại trừ path được.
- Valkey Cluster (sharded pub/sub) chưa hỗ trợ; Valkey một instance hoặc Sentinel.
