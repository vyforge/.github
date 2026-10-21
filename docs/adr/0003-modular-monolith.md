# ADR-0003: Modular monolith với Spring Modulith; hexagonal chỉ cho module lõi

- **Trạng thái:** Chấp nhận — 21/10/2026. Phần MongoDB (outbox, replica set) được thay bởi [ADR-0007](0007-postgresql-primary-database.md)

## Bối cảnh

Vyforge làm nhiều dự án outsource (CRUD là chính) và một số product dài hạn. Layered/hexagonal/clean chỉ trả lời "tách kỹ thuật khỏi nghiệp vụ", không trả lời "chia hệ thống theo nghiệp vụ". Microservice từ đầu quá tốn kém cho đa số dự án.

## Quyết định

- Mọi backend là **modular monolith**: mỗi package con của `com.vyforge.<du-an>` là một module; `ApplicationModules.verify()` chạy trong CI.
- Bên trong module mặc định **Profile A** (simple). **Profile B** (hexagonal + jMolecules) chỉ cho module lõi của dự án tier 2.
- Giao tiếp giữa module ưu tiên event (Modulith outbox trên MongoDB); tier 2 đẩy event ra Kafka bằng `@Externalized`.
- Tách microservice chỉ khi có nhu cầu cụ thể (scale, release, team riêng).

## Hệ quả

- Cần MongoDB replica set ở mọi môi trường (đã cấu hình trong `compose.yaml` và Testcontainers).
- Dev phải học khái niệm module, named interface, event registry.
- Dự án lớn lên chỉ cần refactor từng module sang Profile B hoặc tách service, không viết lại.
