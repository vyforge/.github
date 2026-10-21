# ADR-0006: Next.js làm BFF, Better Auth + Keycloak

- **Trạng thái:** Chấp nhận — 21/10/2026

## Bối cảnh

Token trong trình duyệt dễ bị đánh cắp qua XSS. Auth.js đã chuyển sang team Better Auth (09/2025) và chỉ còn nhận bản vá bảo mật.

## Quyết định

- Next.js là **OIDC confidential client** của Keycloak qua **Better Auth** (plugin `genericOAuth` + helper `keycloak`), **không dùng database**: session và token nằm trong cookie HttpOnly mã hoá (JWE).
- Server Component / Server Action lấy access token ở server (`getAccessToken`) và gọi Spring Boot với `Authorization: Bearer`. Spring là resource server (kiểm tra `iss`, `aud`).
- Phân quyền ở DAL (`requireUser`) và đầu mỗi Server Action; `proxy.ts` không dùng để phân quyền.
- **Phương án B:** Spring Cloud Gateway làm BFF (`oauth2Login` + TokenRelay) cho dự án muốn backend nắm toàn bộ bảo mật.

## Hệ quả

- Đã kiểm chứng: build, typecheck, khởi động và smoke E2E của template với cấu hình này. **Chưa kiểm chứng end-to-end đăng nhập thật với Keycloak** — cần làm ở dự án pilot đầu tiên (realm, client confidential, redirect `<BETTER_AUTH_URL>/api/auth/callback/keycloak`).
- Token Keycloak lớn có thể chạm giới hạn kích thước cookie (Better Auth tự chia cookie); nếu gặp, chuyển sang lưu session trong database/Redis.
