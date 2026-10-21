# ADR-0005: Version API theo path

- **Trạng thái:** Chấp nhận — 21/10/2026

## Quyết định

REST API dùng tiền tố **`/api/v{n}`** (vd. `/api/v1/customers`). Lỗi trả về **Problem Details (RFC 9457)**. Thay đổi phá vỡ tương thích → version mới, giữ version cũ tới khi client chuyển xong.

## Hệ quả

Dễ đi qua API gateway/reverse proxy của khách hàng và dễ đọc log. Template hiện khai báo version trực tiếp trong `@RequestMapping`; có thể chuyển sang cơ chế API versioning gốc của Spring Framework 7 (`spring.mvc.apiversion.*`) khi cần nhiều version song song (lưu ý cấu hình `ApiVersionInserter` cho test).
