# platform-starter-observability

## 1. Mục đích

Đặt sẵn các mặc định vận hành để backend chạy đúng trong Kubernetes mà không cần cấu hình thêm:

- health probe;
- OpenTelemetry (trace, metric) gắn tenant và request id;
- graceful shutdown;
- virtual threads;
- log JSON khi chạy trong cluster.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-observability` (kéo `platform-core`, `spring-boot-starter-actuator`, `spring-boot-starter-opentelemetry`) |
| Package web | không có |
| Zone admin | không có. Dashboard và alert nằm ở chart `platform-observability` (repo platform-helm-charts) |
| App liên quan | mọi backend, `platform-gateway`, `platform-auth-server`, `platform-iam-server`, `platform-services` |

## 3. Tính năng chính

- **Mặc định** (đặt ở nguồn cấu hình ưu tiên thấp nhất, app ghi đè được):
  - `management.endpoint.health.probes.enabled=true` (`/actuator/health/liveness`, `/readiness`).
  - `management.endpoints.web.exposure.include=health,info`.
  - Tag `application` = `spring.application.name` cho mọi observation.
  - `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=20s`.
  - `spring.threads.virtual.enabled=true`.
  - Log JSON định dạng ECS khi phát hiện Kubernetes (`KUBERNETES_SERVICE_HOST`), ép bằng `vyforge.logging.structured`.
- **Gắn ngữ cảnh vào span**: `ContextObservationFilter` thêm thuộc tính `tenant_id` và `request_id` cho span HTTP,
  Kafka, `@Async` và `RestClient`. Đây là thuộc tính high-cardinality nên không thành tag của metric.
- **`UserMdcFilter`**: đưa `userId` (`sub`) vào MDC.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-observability</artifactId>
</dependency>
```

```yaml
management:
  otlp.tracing.endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://otel-collector:4318/v1/traces}
  endpoints.web.exposure.include: health,info,prometheus
```

Không cần code. Endpoint OTLP và cổng management là cấu hình chuẩn của Spring Boot (cần xác nhận giá trị chuẩn của
chart `app`).

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.logging.structured` | `true` khi chạy trong Kubernetes | Ép bật/tắt log JSON (ECS) |
| `server.shutdown` | `graceful` | |
| `spring.lifecycle.timeout-per-shutdown-phase` | `20s` | |
| `spring.threads.virtual.enabled` | `true` | |
| `management.endpoints.web.exposure.include` | `health,info` | Thêm `prometheus`, `metrics` khi cần |

## 6. Tích hợp với module khác

- `platform-core` cung cấp MDC `tenant_id`/`request_id`. `platform-starter-privacy` che PII trong message log, cả log
  JSON.
- Metric của mọi starter (`vyforge.*`) đi qua Micrometer. Alert gợi ý (task backlog, job quá hạn, outbox, DLT, realtime)
  nằm ở chart `platform-observability`.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Không có action IAM, không phát sự kiện.

## 8. Trạng thái & còn mở

- Ổn định, module nhỏ (6 lớp).
- Exporter, endpoint collector và sampling là cấu hình của từng bản triển khai. Không có giá trị mặc định trỏ tới
  collector cụ thể.
