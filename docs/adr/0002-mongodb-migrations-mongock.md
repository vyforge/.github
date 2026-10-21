# ADR-0002: Mongock cho migration MongoDB

- **Trạng thái:** *Superseded* bởi [ADR-0007](0007-postgresql-primary-database.md) cho dự án mới — chỉ còn áp dụng cho dự án đang chạy MongoDB

## Bối cảnh

Cần cách quản lý index/schema MongoDB có version. Mongock là công cụ phổ biến nhất nhưng không ra bản mới từ 5.5.1 (03/2025) và không có driver Spring Data cho Spring Boot 4. Liquibase/Flyway với MongoDB còn thử nghiệm.

**Spike (21/10/2026):** Mongock 5.5.1 + `mongodb-sync-v4-driver` (driver thuần, khai báo bean `ConnectionDriver` thủ công) chạy được trên Spring Boot 4.1.1 / MongoDB 8 replica set: tạo index, ghi changelog, lock — test và chạy container đều qua.

## Quyết định

- Dùng **Mongock 5.5.1 với driver thuần**, ChangeUnit đặt trong `shared/migration`, tên `V<NNN>...`.
- Tắt `auto-index-creation` ở **mọi môi trường**; index chỉ tạo bằng migration (tránh xung đột tên index giữa hai cơ chế — đã gặp trong spike).
- Không sửa ChangeUnit đã chạy; thay đổi mới là ChangeUnit mới.

## Hệ quả

- Rủi ro: Mongock không còn được phát triển tích cực. **Phương án rút lui:** runner nội bộ `platform-mongo-migrations` (interface ChangeUnit, collection lịch sử, document lock — vài trăm dòng). Kích hoạt khi Mongock lỗi với bản Spring Boot/driver mới, hoặc có lỗ hổng không được vá.
- Renovate sẽ báo khi có bản Mongock mới; CI của template phát hiện lỗi tương thích.
