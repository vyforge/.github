# ADR-0012: IdP mở rộng — passkey, SAML 2.0, LDAP/AD, Zalo — cấu hình trong database

- **Trạng thái:** Chấp nhận — 26/10/2026. Mở rộng [ADR-0008](0008-platform-auth-server.md) (auth server). Cột
  `tenant_id` để ghép với tenant của ADR-0011.

## Bối cảnh

ADR-0008 hỗ trợ mật khẩu + TOTP và IdP OIDC khai báo trong file cấu hình (Google, Microsoft, OIDC bất kỳ). Khách hàng
doanh nghiệp và sản phẩm cho người dùng Việt Nam cần thêm:

- **SAML 2.0**: nhiều doanh nghiệp chỉ mở SAML (ADFS, Entra ID cấu hình sẵn, Keycloak/Shibboleth).
- **LDAP / Active Directory**: hệ thống nội bộ không có IdP web, chỉ có thư mục.
- **Zalo Login**: kênh đăng nhập phổ biến nhất ở Việt Nam. Zalo dùng OAuth v4 không chuẩn và không trả email.
- **Passkey**: chống phishing, thay mật khẩu và mã TOTP.

Khai báo trong file thì mỗi lần thêm/sửa IdP phải triển khai lại, và secret nằm trong biến môi trường của từng môi
trường. Admin cần tự cấu hình qua admin console.

## Quyết định

### 0. Mặc định tắt

Mọi IdP mở rộng **mặc định tắt**; cài auth server mới không có thêm bề mặt đăng nhập nào:

- **SAML, LDAP, Zalo, OIDC** chỉ hoạt động khi có cấu hình trong database **và** cấu hình đó đã bật. IdP mới tạo luôn ở
  trạng thái tắt; không có IdP nào thì không có nút đăng nhập, `/saml2/authenticate/*` và `/oauth2/authorization/*`
  không chuyển đi đâu, form mật khẩu không gọi LDAP.
- **Passkey** bật bằng cờ `vyforge.account.idp.passkey.enabled` (biến `AUTH_PASSKEY_ENABLED`), mặc định `false`; chỉ
  profile `local` bật sẵn. Tắt thì không đăng ký filter WebAuthn (endpoint `/webauthn/**`, `/login/webauthn` không tồn
  tại) và trang đăng nhập/tài khoản không hiện passkey.
- Test `IdpDefaultsTests` giữ trạng thái này.

### 1. Thứ tự ưu tiên và rủi ro

| Thứ tự | Tính năng | Lý do | Rủi ro chính → cách giảm |
|---|---|---|---|
| 1 | Passkey | Có sẵn trong Spring Security 7 (WebAuthn + webauthn4j), lợi ích bảo mật lớn nhất | Mất thiết bị → vẫn còn mật khẩu/TOTP; admin thu hồi passkey. Đổi RP ID làm mọi passkey mất hiệu lực → RP ID cố định theo domain của issuer |
| 2 | SAML 2.0 (SP) | Yêu cầu bắt buộc của khách doanh nghiệp | OpenSAML chỉ phát hành trên repo Shibboleth; assertion POST chéo site mất cookie `SameSite=Lax` → trang gửi lại cùng site (mục 5) |
| 3 | LDAP / AD | Hệ thống nội bộ | Thư mục chậm/không truy cập được làm chậm đăng nhập → timeout 5s/10s, lỗi thì rơi về mật khẩu nội bộ |
| 4 | Zalo | Người dùng phổ thông ở VN | Giao thức không chuẩn, có thể đổi; không có email → luồng xác nhận email riêng (mục 3) |

### 2. Lưu cấu hình IdP trong database

- Bảng `account_identity_providers` (module account): `id`, `tenant_id` (`NULL` = dùng chung mọi tenant), `alias`
  (duy nhất toàn hệ thống, là registration id trong URL, không đổi sau khi tạo), `type` (`OIDC`/`ZALO`/`SAML`/`LDAP`,
  không đổi), `display_name`, `enabled`, `auto_register`, `role_mapping` (JSONB group → role), `settings` (JSONB, không
  chứa secret), `secret`, `version` (khoá lạc quan).
- **Secret** (client secret OIDC, app secret Zalo, mật khẩu bind LDAP) mã hoá **AES-256-GCM** bằng
  `vyforge.auth.keys.encryption-key` (cùng khoá với khoá ký và TOTP), ngữ cảnh `idp:<id>` nên bản mã không chép sang IdP
  khác được. API **chỉ ghi**: response có `secretSet`, sửa mà để trống `secret` = giữ secret cũ.
- IdP mới ở trạng thái **tắt**; không bật được khi loại đó cần secret mà chưa có. Metadata SAML được tải/đọc thử lúc lưu.
- Alias không được trùng registration id trong file (`vyforge.account.social.providers`). Xoá IdP thì xoá luôn mọi liên
  kết tài khoản với nó — alias tạo lại sau này không nhận lại liên kết cũ.
- API quản trị `/api/v1/admin/identity-providers` (CRUD, `enable`/`disable`, `test` cho LDAP); mọi thao tác ghi nhật ký
  quản trị (`@Audited`). Registration được cache theo `vyforge.account.idp.cache-ttl` (mặc định 30s): replica nhận thay
  đổi nạp lại ngay, replica khác chậm tối đa một TTL.

### 3. Liên kết tài khoản theo email đã xác thực, chống chiếm tài khoản

Quy tắc chung cho mọi IdP ngoài (OIDC, Zalo, SAML, LDAP):

1. Định danh ở IdP (`sub`, NameID/attribute SAML, DN LDAP, id Zalo) **đã liên kết** → đăng nhập vào tài khoản đó.
2. Chưa liên kết, email trùng tài khoản có sẵn → **chỉ tự liên kết khi cả hai phía đã xác thực email**: IdP được cấu
   hình tin email (`trustEmailVerified` của OIDC kèm claim `email_verified`, `trustEmail` của SAML/LDAP) **và** tài
   khoản có sẵn có `email_verified = true`. Thiếu một trong hai → từ chối (`EMAIL_IN_USE`). Điều kiện thứ hai chặn
   *pre-account takeover*: kẻ tấn công tạo trước tài khoản bằng email nạn nhân (chưa xác thực) rồi chờ nạn nhân đăng
   nhập bằng IdP.
3. Email chưa có tài khoản → JIT provisioning (mục 4).
4. Liên kết thủ công từ trang tài khoản (`/account/connections`) luôn được — người dùng đã chứng minh sở hữu cả hai.
5. **IdP không trả email (Zalo):** hồ sơ IdP giữ trong session; người dùng nhập email, auth server gửi link xác nhận.
   Link chỉ dùng được khi có **cả token trong email lẫn session đã đăng nhập IdP** (cùng trình duyệt, 30 phút). Kẻ
   tấn công nhập email nạn nhân thì không có token; nạn nhân bấm link (nếu có) thì không có session của kẻ tấn công. Bấm
   đúng → coi email đã xác thực rồi áp dụng quy tắc 2–3.
6. Tài khoản bật TOTP vẫn qua bước mã sau khi đăng nhập bằng IdP ngoài. Passkey thì không cần (mục 6).

### 4. JIT provisioning và role

- Mỗi IdP có `auto_register` (mặc định bật): tạo tài khoản khi đăng nhập lần đầu với role mặc định
  (`vyforge.account.registration.default-roles`), email đã xác thực theo quy tắc trên. Tắt → chỉ người đã có tài khoản
  (hoặc đã liên kết) mới vào được.
- **Role mapping** (SAML theo attribute group, LDAP theo group tìm được): mỗi lần đăng nhập, role nằm trong mapping được
  đặt lại theo group hiện tại — vào group thì có role, ra group thì mất. Role ngoài mapping (admin gán tay) giữ nguyên.
  Role trong mapping phải có trong danh mục role. Không đọc được group LDAP → từ chối đăng nhập (không gỡ role nhầm).

### 5. SAML 2.0, LDAP, Zalo — chi tiết

- **SAML** dùng `saml2Login` của Spring Security (OpenSAML 5): Vyforge là SP; entityId mặc định
  `<issuer>/saml2/metadata/<alias>` (đổi được), ACS `<issuer>/login/saml2/sso/<alias>` (HTTP-POST), bắt đầu từ
  `/saml2/authenticate/<alias>`. Spring Security kiểm chữ ký theo chứng chỉ trong metadata, audience, recipient, thời hạn
  và `InResponseTo`. IdP POST assertion chéo site nên cookie session `SameSite=Lax` không đi kèm: request ACS không có
  cookie được trả một trang tự gửi lại đúng tham số tới chính ACS (script cùng origin, hợp CSP). Chưa ký AuthnRequest và
  chưa giải mã assertion mã hoá (cần khoá riêng của SP) — làm khi có IdP yêu cầu.
- **LDAP/AD** dùng `BindAuthenticator` + `FilterBasedLdapUserSearch` của Spring Security LDAP: tìm người dùng bằng tài
  khoản dịch vụ (hoặc ẩn danh), bind bằng DN tìm được với mật khẩu người dùng. Chạy trên form mật khẩu, thử **trước**
  mật khẩu nội bộ; LDAP không có người dùng hoặc không kết nối được → mật khẩu nội bộ xử lý. Định danh liên kết là DN
  (chữ thường). AD: filter `(|(sAMAccountName={0})(userPrincipalName={0}))`, dùng `ldaps://`.
- **Zalo Login v4** (theo SDK chính thức zaloplatform/zalo-php-sdk): authorize `oauth.zaloapp.com/v4/permission` với
  `app_id` + PKCE S256; token `oauth.zaloapp.com/v4/access_token` với form `code, app_id, grant_type, code_verifier` và
  secret ở header `secret_key` (lỗi trả HTTP 200 kèm `error` âm, `expires_in` dạng chuỗi); hồ sơ
  `graph.zalo.me/v2.0/me?fields=id,name,picture` với token ở header `access_token`. Đi chung `oauth2Login` với
  Google/Microsoft; token client và user service riêng cho registration Zalo. Endpoint đổi được qua
  `vyforge.account.idp.zalo.*` (để test).
- **OIDC** cấu hình qua API dùng discovery như IdP OIDC trong file.

### 6. Passkey

- `webAuthn` của Spring Security 7, credential lưu PostgreSQL bằng `JdbcUserCredentialRepository` /
  `JdbcPublicKeyCredentialUserEntityRepository` (bảng `user_credentials`, `user_entities` theo schema của Spring
  Security). User entity có `name` = ID tài khoản, `displayName` = email.
- Discoverable credential (resident key **bắt buộc**) và **user verification bắt buộc** cả khi đăng ký lẫn đăng nhập.
  Vì vậy passkey là hai yếu tố (thiết bị + vân tay/PIN): `amr` = `["hwk","mfa"]`, không hỏi thêm TOTP, vào được admin
  console (yêu cầu `mfa`).
- RP ID mặc định là host của issuer (`vyforge.account.idp.passkey.rp-id`), origin là origin của issuer.
- Người dùng quản lý passkey ở `/account/passkeys`; admin xem/thu hồi qua `/api/v1/admin/users/{id}/passkeys` (và thu
  hồi liên kết IdP qua `/api/v1/admin/users/{id}/identities/{provider}`).

### 7. Audit

Audit log đăng nhập ghi nguồn: `password`, `mfa`, `passkey`, `<registration id>` (IdP trong file), `saml:<alias>`,
`ldap:<alias>`, `zalo:<alias>`, `oidc:<alias>` (cột `method` nới lên 100 ký tự).

## Hệ quả

- Thêm IdP không cần triển khai lại; secret không còn rải trong biến môi trường. Mất
  `vyforge.auth.keys.encryption-key` thì mất cả secret IdP (như khoá ký và TOTP).
- Build cần thêm repo Maven của Shibboleth (`build.shibboleth.net/maven/releases`) cho OpenSAML; mirror nội bộ phải
  proxy cả repo này.
- `tenant_id` chưa được dùng khi chọn IdP; khi có tenant (ADR-0011) thì IdP gắn tenant chỉ hiện cho tenant đó, `NULL`
  dùng chung.
- Chưa làm: SAML Single Logout, ký AuthnRequest/giải mã assertion, SCIM, đồng bộ LDAP định kỳ (chỉ đồng bộ khi đăng
  nhập), OIDC role mapping theo claim group.
