# ADR-0007: PostgreSQL là database chính, migration bằng Flyway

- **Trạng thái:** Chấp nhận — 21/10/2026. Thay thế [ADR-0002](0002-mongodb-migrations-mongock.md); thay phần MongoDB trong hệ quả của [ADR-0003](0003-modular-monolith.md).

## Bối cảnh

Dự án của Vyforge phần lớn là nghiệp vụ có quan hệ chặt (đơn hàng, thanh toán, kho, phân quyền) và cần báo cáo. MongoDB có transaction nhiều document nhưng phải chạy replica set, có giới hạn thời gian transaction và thiếu ràng buộc toàn vẹn (foreign key, check). Công cụ migration Mongock không còn được phát triển (ADR-0002). `platform-auth-server` (ADR-0008) cũng cần database quan hệ.

## Quyết định

- **PostgreSQL 18** là database mặc định cho mọi dự án mới. Dữ liệu dạng document dùng cột `JSONB`.
- Truy cập dữ liệu: **Spring Data JPA** (Hibernate) mặc định; Spring Data JDBC hoặc jOOQ cho module cần SQL tường minh. `spring.jpa.hibernate.ddl-auto=validate`, `spring.jpa.open-in-view=false`.
- Schema chỉ thay đổi bằng **Flyway**: file `src/main/resources/db/migration/V<NNN>__<mo_ta>.sql`; không sửa migration đã chạy.
- **Ranh giới module trong database:** mỗi module sở hữu bảng của mình (tiền tố `<module>_`), không foreign key và không JOIN chéo module — module khác lấy dữ liệu qua API hoặc event của module. Nhờ vậy vẫn tách được service theo tier 3.
- Event Publication Registry của Spring Modulith chạy trên JPA (`spring-modulith-starter-jpa`); bảng `event_publication` tạo bằng Flyway, không bật tự khởi tạo schema.
- Consumer idempotent: bảng `processed_messages` (khoá chính `event_id`), ghi trong cùng transaction với xử lý nghiệp vụ.
- Entity có `@Version` (optimistic lock) và cột audit `created_at/created_by/updated_at/updated_by`. ID là UUID v7.
- Test: Testcontainers `PostgreSQLContainer` + `@ServiceConnection`, không dùng H2.
- MongoDB chỉ dùng cho từng module có lý do cụ thể (log, catalog schema linh hoạt…) và phải có ADR của dự án.

## Hệ quả

- Bỏ yêu cầu MongoDB replica set và phụ thuộc vào Mongock.
- `template-spring-boot` phải đổi: dependency, `compose.yaml` (Postgres thay Mongo), Testcontainers, thư mục migration, test mẫu.
- Dự án đang chạy MongoDB giữ nguyên và tiếp tục theo ADR-0002; không migrate nếu không có nhu cầu nghiệp vụ.
- Dev cần nắm thiết kế schema quan hệ, index và kế hoạch truy vấn (`EXPLAIN`); review migration như review code.
