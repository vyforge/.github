# ADR-0008: Identity provider riêng `platform-auth-server` trên Spring Authorization Server

- **Trạng thái:** Chấp nhận — 21/10/2026. Thay phần Keycloak trong [ADR-0006](0006-frontend-auth-bff.md); mô hình BFF của ADR-0006 giữ nguyên.

## Bối cảnh

Vyforge muốn kiểm soát toàn bộ luồng đăng nhập, giao diện và dữ liệu người dùng, tích hợp với database PostgreSQL (ADR-0007). Spring Authorization Server (SAS) cung cấp phần giao thức OAuth2/OIDC; phần sản phẩm (UI, quản lý user, MFA, admin) phải tự xây và tự chịu trách nhiệm bảo mật.

## Quyết định

- Xây **một sản phẩm duy nhất `platform-auth-server`** (repo riêng, semver, image riêng), triển khai riêng cho từng khách hàng hoặc dùng chung cho product nội bộ. Dự án không tự dựng authorization server.
- Database PostgreSQL **riêng** của auth server, không dùng chung với database của app.
- **App chỉ phụ thuộc chuẩn OIDC:** backend là resource server (`issuer-uri`, kiểm tra `aud`); Next.js dùng Better Auth `genericOAuth` với discovery URL của auth server. Không dùng API riêng của auth server trong app, trừ module quản trị user.
- **Phân quyền nghiệp vụ nằm trong app.** Token chỉ mang `sub`, `aud`, `scope` và role thô; profile và quyền chi tiết lưu ở app, khoá theo `sub`.
- Access token JWT sống ngắn (5–15 phút); refresh token xoay vòng, thu hồi được; key ký (JWK) xoay định kỳ.

### Phạm vi MVP

| Nhóm | Nội dung |
|---|---|
| Giao thức | Authorization code + PKCE, refresh token rotation, client credentials, OIDC discovery, userinfo, RP-initiated logout |
| Tài khoản | Đăng nhập mật khẩu (`DelegatingPasswordEncoder`, Argon2/bcrypt), đăng ký, xác thực email, quên mật khẩu, khoá tạm khi sai nhiều lần |
| Bảo mật | MFA TOTP, rate limit, CSRF, header bảo mật, audit log sự kiện đăng nhập |
| Liên kết | Đăng nhập Google, Microsoft |
| Quản trị | API + giao diện cơ bản quản lý user, client, role |

Ngoài MVP (làm khi có dự án cần): passkey, SAML, LDAP, đăng nhập Zalo, multi-tenant.

### Vận hành và trách nhiệm

- Có team/owner cố định (tối thiểu 2 người), ghi trong `CODEOWNERS`.
- Bản vá bảo mật của Spring Security/SAS được phát hành trong **7 ngày**.
- **Review bảo mật hoặc pentest trước lần đầu lên production**, sau đó mỗi năm và sau mỗi thay đổi lớn về luồng đăng nhập.

## Hệ quả

- Cho đến khi `platform-auth-server` qua review bảo mật, dự án cần lên production dùng **Keycloak (phương án B)**; app không phải đổi code khi chuyển, chỉ đổi issuer và client.
- Keycloak cũng là phương án cho khách cần SAML/IdP doanh nghiệp hoặc khi không đủ người duy trì auth server.
- Chi phí lâu dài: phát triển và duy trì UI, MFA, admin, vá bảo mật — tính như một product nội bộ.
- `compose.yaml` và Testcontainers của template thay Keycloak bằng image `platform-auth-server` khi có bản đầu tiên.
