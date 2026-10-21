# ADR-0001: Maven 3.9 cho backend

- **Trạng thái:** Chấp nhận — 21/10/2026

## Bối cảnh

Cần một công cụ build chung cho mọi dự án Spring Boot. Maven 4 vẫn ở bản RC (4.0.0-rc-7, "không dùng cho production"). Gradle 9 ổn định, nhanh hơn, nhưng đa số dev quen Maven và CI mẫu đã dùng Maven.

## Quyết định

Dùng **Maven 3.9.x qua wrapper (`mvnw`)**, version do Spring Boot parent/BOM quản lý. Sau này gom cấu hình plugin chung vào `platform-parent`.

## Hệ quả

- Một bộ CI (`java-ci.yml`), một parent, một tài liệu.
- Nhóm đã quen Gradle có thể dùng Gradle 9 Kotlin DSL như **ngoại lệ có ADR riêng** của dự án.
- Đánh giá lại khi Maven 4 GA và có ít nhất một bản patch.
