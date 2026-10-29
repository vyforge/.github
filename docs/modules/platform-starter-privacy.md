# platform-starter-privacy — nền bảo vệ dữ liệu cá nhân

## 1. Mục đích

Mọi ứng dụng xử lý dữ liệu cá nhân phải tuân Luật 91/2025/QH15 và NĐ 356/2025/NĐ-CP. Starter này là **phần nền tối thiểu**
mà mọi service đều nhúng: đánh dấu field chứa dữ liệu cá nhân, che PII trong log và audit, danh mục tài sản dữ liệu, cùng
các SPI (contributor, kiểm mục đích xử lý, nơi lưu dữ liệu) để engine `platform-starter-privacy-dsr` điều phối yêu cầu của
chủ thể. Bản thân starter không có bảng, không có REST công khai.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter nền | `platform-starter-privacy` | Từ 0.11.0 (bản tối thiểu, B21); không thu phí (starter nền) |
| Engine | [`platform-starter-privacy-dsr`](platform-starter-privacy-dsr.md) | DSR, consent, RoPA… (có phí, key `privacy`) |
| Package web | — | Không có package riêng; `@vyforge/privacy` thuộc privacy-dsr |
| Kiểm kiến trúc | `platform-test`: `PlatformArchitectureRules.PERSONAL_DATA_ANNOTATED` | Bắt entity có cột email, phone, cccd, address thiếu `@PersonalData` |

Thiết kế: [docs/design/privacy-dsr.md](../design/privacy-dsr.md) (phần SPI); không có README module (cần bổ sung).

## 3. Tính năng chính

- **`@PersonalData`** trên field/record component: `category` (`BASIC`, `SENSITIVE`), `redaction` (`MASK`, `HASH`…),
  `type` (khoá taxonomy NĐ 356 Điều 3–4, `META-INF/vyforge/privacy/taxonomy.yml`), `erase` (`ErasureMethod`),
  `replacement`. `@NotPersonalData` để ghi rõ ngoại lệ.
- **Che PII**: `LogSafeJson` thay field `@PersonalData` bằng `Redaction` khi ghi log/audit (JSON trả cho client không
  đổi); `PiiMasker` + `PiiLogMasking` che email, số điện thoại, CCCD trong message log; message converter che PII.
- **`@PersonalDataOwner`** trên entity JPA (+ `@RetentionTimestamp`): privacy-dsr tự sinh contributor `app.<entity>` để
  xuất và xoá mà app không phải viết code.
- **`PersonalDataInventory`**: quét class của app lúc khởi động, kiểm `type` có trong taxonomy, dùng cho danh mục tài sản
  dữ liệu và RoPA.
- **SPI cho engine**: `PersonalDataContributor` (xuất, xoá, sửa, hạn chế; idempotent theo `requestId`, trả checkpoint khi
  dài), `RetentionCapable`, `ThirdPartyDeclaration`, `DataTaxonomy`.
- **`PrivacyGuard`** + `@RequiresPurpose(value, subject)`: kiểm legal basis, consent, hạn chế/phản đối, purpose bị tắt ở
  tenant; từ chối → `ProcessingNotAllowedException`. Mặc định `PermitAllPrivacyGuard` khi chưa có privacy-dsr.
- **`DataResidency`**: kiểm nơi lưu/nơi gửi theo quy tắc của tenant (chỉ kiểm ý định module khai báo). Mặc định
  `PermissiveDataResidency`.
- **Contributor endpoint** (`privacy-contributor/v1`): phơi contributor local ở `/api/v1/privacy/internal/contributor/v1/**`
  cho engine chạy ở service khác.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-privacy</artifactId>
</dependency>
```

```yaml
vyforge.privacy:
  hash-key: ${PRIVACY_HASH_KEY}       # ≥ 32 byte, từ secret store
```

```java
@Entity
@PersonalDataOwner(subjectType = "customer", subjectField = "id", onErase = ErasureStrategy.ANONYMIZE)
class Customer extends TenantEntity {
    @PersonalData(type = "basic.identity.name", erase = ErasureMethod.REPLACE, replacement = "Đã xoá")
    String fullName;

    @PersonalData(type = "basic.contact.phone", erase = ErasureMethod.NULL)
    String phone;
}

@RequiresPurpose(value = "credit.scoring", subject = "#application.subjectRef()")
public Score score(LoanApplication application) { ... }
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.privacy.hash-key` | — | Khoá HMAC cho `Redaction.HASH` và `subject_hash` của privacy-dsr. Không đặt thì sinh ngẫu nhiên mỗi lần khởi động (không so được giữa replica) |
| `vyforge.privacy.json.mask-text` | `true` | Che email/SĐT/CCCD trong mọi chuỗi của JSON log/audit |
| `vyforge.privacy.logging.mask` | `true` | Che PII trong message log |
| `vyforge.privacy.inventory.enabled` | `true` | Quét `@PersonalData` lúc khởi động |
| `vyforge.privacy.contributor-endpoint.enabled` | `false` | Phơi contributor cho engine ở service khác |
| `vyforge.privacy.contributor-endpoint.call-timeout` | `2m` | Thời gian tối đa một lời gọi |

## 6. Tích hợp với module khác

- **privacy-dsr**: thay `PermitAllPrivacyGuard` và `PermissiveDataResidency` bằng bản thật, điều phối mọi
  `PersonalDataContributor`.
- **audit**: audit log dùng `LogSafeJson` nên không lưu giá trị PII.
- Các module có dữ liệu cá nhân cài `PersonalDataContributor` (settings, sequence, organization, search, notification,
  files, collaboration, automation…).
- **jpa**: `@PersonalDataOwner` làm việc với entity `TenantEntity`.

## 7. Chế độ triển khai, quyền, sự kiện

- Luôn **embedded** trong mọi service. Khi engine privacy-dsr chạy ở service khác, bật `contributor-endpoint`
  (`/internal/**`, gateway chặn, chỉ gọi trong cluster).
- Không có action IAM, không phát sự kiện.

## 8. Trạng thái & còn mở

- **Đã làm**: bản tối thiểu 0.11.0; mở rộng SPI (contributor, guard, residency, taxonomy, endpoint) cùng privacy-dsr ở
  0.14.0.
- **Còn mở**: chưa có README module trên `origin/main`. Taxonomy theo NĐ 356 cần pháp chế rà khi có dự án thật (cần xác
  nhận).
- **Giới hạn đã biết**: `DataResidency` chỉ kiểm ý định module khai báo, không chặn code tự gọi HTTP ra ngoài (việc của
  lớp mạng). Che PII trong log dựa trên mẫu (email, SĐT, CCCD), không bắt được mọi dạng PII tự do.
