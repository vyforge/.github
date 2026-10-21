# ADR-0004: Renovate thay cho Dependabot

- **Trạng thái:** Chấp nhận — 21/10/2026

## Quyết định

Mọi repo dùng **Renovate** với preset chung `github>vyforge/.github:renovate-config`:

- Gom minor/patch thành một PR mỗi tuần; major cần duyệt trên Dependency Dashboard.
- Chờ 3 ngày trước khi đề xuất bản mới (khớp cổng 24 giờ của pnpm); cảnh báo bảo mật đi ngay.
- Fast-track bản vá cho `next`, `react`, `sharp`, `better-auth`, Spring Boot, Spring Security.
- Ghim GitHub Actions theo commit SHA (sau vụ `trivy-action` bị chiếm quyền 03/2026).

## Hệ quả

Cần cài Renovate GitHub App cho org (hoặc self-host bản ≥ 44.3.1). Sửa preset một lần áp dụng cho mọi repo.
