# platform-starter-security

## 1. Mục đích

Biến mọi backend thành OIDC resource server theo cùng một cách (ADR-0008). Starter kiểm JWT của `platform-auth-server`
(hoặc Keycloak) và bắt buộc kiểm `aud`. Sau khi xác thực, starter điền tenant vào ngữ cảnh. Khi cần gọi service khác
thay mặt người dùng, starter đổi token sang service đích (ADR-0010). App không phải tự viết `SecurityFilterChain`.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-security` (kéo theo `platform-core`) |
| Package web | `@vyforge/auth` 0.4.0 (Better Auth + BFF phía Next.js, ADR-0006) |
| Zone admin | không có |
| App liên quan | `platform-auth-server` (issuer), `platform-gateway`, mọi backend |

## 3. Tính năng chính

- **Resource server stateless**:
  - Mọi path cần JWT, trừ health, info, OpenAPI và `vyforge.security.public-paths`.
  - Thiếu cấu hình audience thì app không khởi động (B9).
  - Token không có `sub` → 401.
- **Role**: claim `roles` (đổi được, kể cả đường dẫn lồng như `realm_access.roles` của Keycloak) thành `ROLE_*`; dùng
  `@PreAuthorize`.
- **Lỗi 401/403** trả dạng Problem Details, câu chữ theo `Accept-Language`.
- **Tham số controller `CurrentUser`**: `sub`, role, `hasRole(...)`.
- **Tenant**: claim `tenant_id` được đưa vào `TenantContext`. Claim không phải slug hợp lệ → 403 `tenant_invalid`.
  Header `X-Tenant-Id` không bao giờ được đọc.
- **Gọi service khác**:
  - `vyforge.web.client.relay-token=true` gửi tiếp access token của người dùng, khi service kia nhận cùng audience.
  - `TokenExchangeClient` (RFC 8693): đổi token sang `aud` của service đích. Token mới giữ `sub` và `roles`, có
    `act.sub` = service đang gọi, được cache tới gần hết hạn.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-security</artifactId>
</dependency>
```

```yaml
spring.security.oauth2.resourceserver.jwt:
  issuer-uri: ${OIDC_ISSUER_URI}
  audiences: ${OIDC_AUDIENCE}          # bắt buộc
vyforge.security.public-paths: /api/v1/public/**
```

```java
@PostMapping
OrderDto create(@Valid @RequestBody CreateOrder body, CurrentUser user) { ... }

RestClient billing = builder.baseUrl("http://billing-api")
        .requestInterceptor(tokenExchange.interceptor("billing-api"))   // registration vyforge-token-exchange
        .build();
```

Test: `TestJwt.user("u-1", "SALES")`, `TestJwt.tenantUser(...)` của `platform-test`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.security.public-paths` | rỗng | Path không cần token, cộng thêm vào health, info, `/v3/api-docs/**`, `/swagger-ui/**` |
| `vyforge.security.roles-claim` | `roles` | Claim chứa role |
| `vyforge.security.require-audience` | `true` | Thiếu `…jwt.audiences` thì app không khởi động |
| `vyforge.tenant.required` | `false` | Nên bật khi auth server đã phát claim `tenant_id` |
| `vyforge.web.client.relay-token` | `false` | Gửi tiếp token người dùng khi gọi service khác |
| `spring.security.oauth2.client.registration.vyforge-token-exchange.*` | — | Client confidential có grant token exchange ở auth server |

## 6. Tích hợp với module khác

- `platform-starter-authz` dựa trên `Authentication` mà starter này dựng.
- `platform-starter-ratelimit` chạy sau filter tenant (`-99`), nên khoá theo được `sub` và tenant.
- `platform-starter-ops` đọc claim `amr` và `auth_time` để đòi MFA và step-up.
- Ở `platform-auth-server`: client cần `audiences` gồm backend. Muốn token exchange thì client cần grant
  `token_exchange`.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng.
- Không có action IAM riêng, không phát sự kiện.

## 8. Trạng thái & còn mở

- Ổn định.
- App tự khai báo `SecurityFilterChain` thì cấu hình mặc định lùi lại.
- Theo ADR-0008, production vẫn dùng Keycloak cho tới khi `platform-auth-server` qua review bảo mật. Starter không phụ
  thuộc issuer cụ thể.
- `actor_token` (delegation) chưa hỗ trợ ở phía auth server.
