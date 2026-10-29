# platform-starter-jpa

## 1. Mục đích

Chuẩn hoá tầng dữ liệu PostgreSQL + JPA + Flyway (ADR-0007) và cách ly tenant ở mức Hibernate (ADR-0015). Starter cũng
gom bốn nhu cầu mà app nghiệp vụ Việt Nam nào cũng gặp (jpa-extras):

- tìm kiếm tiếng Việt không dấu;
- dữ liệu đa ngôn ngữ;
- xoá mềm có thùng rác;
- mã hoá cột chứa dữ liệu cá nhân.

App chỉ khai báo entity và migration, không tự viết lại các phần này.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-jpa` (phụ thuộc tuỳ chọn `-web`, `-authz` cho REST thùng rác) |
| Package web | `@vyforge/trash` 0.1.0 (client + trang thùng rác) |
| Zone admin | `apps/trash-admin` (zone `/m/trash`, đọc được thùng rác của nhiều app) |
| App liên quan | mọi backend có database, `platform-services`, `platform-iam-server` |

## 3. Tính năng chính

- **Nền JPA**:
  - Mặc định `ddl-auto=validate`, `open-in-view=false`, giờ UTC.
  - `BaseEntity`: id UUID v7, `@Version`, cột audit. Auditor là `sub` của token; ngoài request là `system`.
  - `TenantEntity` (`@TenantId`): Hibernate tự lọc mọi truy vấn và tự gán tenant khi insert. Ngoài phạm vi tenant thì
    đọc không ra dòng nào và insert bị từ chối.
  - Row-Level Security tuỳ chọn: `set_config('app.tenant_id', …)` ở đầu mỗi transaction.
- **Flyway riêng của starter** (`StarterSchemaMigrator`, `StarterSchema.of("files")`): mỗi starter có bảng lịch sử
  `flyway_schema_history_<module>` và đánh số migration độc lập với app.
- **SQL nền schema `vyforge`**: extension `unaccent`, `pg_trgm`; hàm `vyforge.fold`, `vyforge.localized…`; cấu hình FTS
  `vyforge.vi`. Starter tự cài trước Flyway của app, có advisory lock.
- **Tìm tiếng Việt không dấu**: `VietnameseSearch.searchVietnamese(fields, q)` và `orderBySimilarity`. Hàm HQL: `fold`,
  `localized`, `trgm_word_similarity`. Từ khoá ngắn hơn 3 ký tự thì so tiền tố, hoặc trả 400 `jpa.search_too_short`.
- **`LocalizedText`**: cột `jsonb` dạng `{"vi": …, "en": …}`.
  - Fallback giống nhau ở Java và SQL.
  - Sắp xếp theo collation `vi-VN-x-icu` bằng `LocalizedSpecs.orderBy`.
  - Validation: `@LocalizedRequired`, `@LocalizedSize`.
- **Xoá mềm và thùng rác**:
  - Đánh dấu bằng `@SoftDeletable(type, retention, purge)` và kế thừa `SoftDeletableTenantEntity`.
  - `SoftDeletes`: `delete` (kèm lý do), `restore` (trùng unique → 409), `includingDeleted`, `purgeNow`.
  - Hibernate filter tự bỏ dòng đã xoá. Xoá cứng bị chặn (`HardDeleteNotAllowedException`).
  - `SoftDeletePurger` purge theo hạn: chia lô, có cursor, bỏ qua dòng vướng khoá ngoại.
  - Purge kiểu `ANONYMIZE`: ẩn danh rồi giữ dòng.
  - Điểm mở rộng: `SoftDeleteListener`, `PurgeVeto` (legal hold), `RetentionPolicyResolver` (theo tenant hoặc
    settings).
  - REST thùng rác dùng chung (opt-in qua bean `TrashResource`):
    - `GET /api/v1/trash/types`
    - `GET /{type}` (lọc `q`, `deletedBy`, `deletedFrom`/`deletedTo`)
    - `POST /{type}/{id}/restore`, `DELETE /{type}/{id}`
    - `GET /admin/info`

    Mỗi app một đường dẫn: `vyforge.jpa.trash.api.path`, `…api.app`.
- **Mã hoá cột**:
  - `@Encrypted("purpose")`: AES-GCM qua `SecretCipher`, AAD gồm `purpose` và tenant.
  - `@BlindIndex`: HMAC, chuẩn hoá `EMAIL`, `PHONE_VN`, `NATIONAL_ID` để tìm chính xác.
  - `EncryptionRotation` mã hoá lại theo lô khi xoay khoá.
- **`EntityRef`** map tự động sang hai cột (`entity_type`, `entity_id`).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-jpa</artifactId>
</dependency>
```

```java
@Entity
@SoftDeletable(type = "crm:customer", retention = "P90D")
public class Customer extends SoftDeletableTenantEntity {
    private LocalizedText displayName = LocalizedText.empty();
    @PersonalData @Encrypted("crm.customer.national_id") private String nationalId;
}

softDeletes.delete(customer, "Trùng hồ sơ");
customers.findAll(VietnameseSearch.searchVietnamese(List.of("nameFold", "codeFold"), q), pageable);

@Bean
TrashResource<Customer> customerTrash() {
    return TrashResource.of(Customer.class, LocalizedText.vi("Khách hàng", "Customers"), Customer::getCode)
            .searchingBy("nameFold");
}
```

Migration của app dùng hàm nền:
`name_fold text generated always as (vyforge.localized_fold(display_name)) stored` cùng index GIN trigram. Unique index
của bảng xoá mềm phải là partial `where deleted_at is null`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.jpa.auditing.enabled` | `true` | Tự điền cột audit |
| `vyforge.jpa.tenant.row-level-security` | `false` | Đặt `app.tenant_id` ở đầu mỗi transaction cho policy RLS |
| `vyforge.jpa.text.enabled` / `.install` | `true` / `true` | `install=false` khi DBA tự chạy `META-INF/vyforge/sql/jpa-text.sql` |
| `vyforge.jpa.text.search.min-length` | `3` | Từ khoá ngắn hơn thì xử lý theo `short-query` (`PREFIX` hoặc `REJECT`) |
| `vyforge.jpa.localized-text.fallback` / `.locales` | `vi, en` | Thứ tự fallback và ngôn ngữ được phép |
| `vyforge.jpa.soft-delete.retention` | `P30D` | Hạn mặc định của thùng rác; `P0D` = không tự purge |
| `vyforge.jpa.soft-delete.purge.cron` / `.enabled` | `0 30 2 * * *` / `true` | Job purge (cần `-scheduling`) |
| `vyforge.jpa.trash.api.enabled` / `.path` / `.app` | `true` / `/api/v1/trash` / `spring.application.name` | REST thùng rác |
| `vyforge.jpa.encryption.blind-index.keys.<kid>` | — | Keyring HMAC riêng cho blind index |
| `vyforge.jpa.encryption.accept-plaintext` | `false` | Bật tạm khi mã hoá cột đã có dữ liệu rõ |
| `vyforge.jpa.encryption.rotation.cron` | `0 0 3 * * *` | Job xoay khoá cột |
| `vyforge.<module>.schema.migrate` | `true` | `false`: DBA tự chạy migration của starter |

## 6. Tích hợp với module khác

- **`platform-starter-scheduling`**: có sẵn job `SoftDeletePurgeJob` và `EncryptionRotationJob`, chạy theo tenant qua
  `TenantDirectory`.
- **`platform-starter-authz`**: kiểm quyền cho REST thùng rác. Action chung là `trash:item:List|Restore|Purge`, hoặc
  action riêng khai qua `TrashResource.withActions`.
- **`platform-starter-files`**: xoá mềm bản ghi cha thì file đi theo; file đang legal hold thì chặn purge.
- **`platform-starter-settings`**: cung cấp retention thùng rác theo tenant.
- **`platform-starter-privacy-dsr`**: dùng `purgeNow` hoặc `ANONYMIZE` cho yêu cầu xoá dữ liệu. Xoá mềm không được tính
  là xoá theo Luật 91/2025.
- **Test** (`platform-test`): `JpaExtrasSchemaChecks.verify`, luật `JPA_EXTRAS`, `SENSITIVE_PERSONAL_DATA_ENCRYPTED`,
  `JpaDdl.forEntity`.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng trong app sở hữu dữ liệu.
- Thùng rác: mỗi app một API; gateway định tuyến theo tiền tố. Zone `trash-admin` gom được nhiều app qua
  `GET …/admin/info`.
- Quyền: `trash:item:List`, `trash:item:Restore`, `trash:item:Purge` trên `vrn:trash:item/<type>`.
- Sự kiện (qua outbox khi có Spring Modulith): `EntitySoftDeleted`, `EntityRestored`, `EntityPurged` (mang `EntityRef`
  và `tenantId`).
- Metric: `vyforge.jpa.softdelete.trash.size`, `…purge.blocked`, `vyforge.jpa.encryption.rotation.pending|rotated`,
  `vyforge.jpa.text.fold.mismatch`.

## 8. Trạng thái & còn mở

- jpa-extras có từ 0.12. 0.16 thêm lọc theo ngày, mỗi app một API thùng rác, retention theo settings, và file đi theo
  xoá mềm.
- Giới hạn đã biết:
  - AAD của cột mã hoá không gắn id dòng.
  - `deleteAllInBatch` (HQL) không bị chặn với entity xoá mềm.
  - Join bị Hibernate tối ưu bỏ thì không có điều kiện xoá mềm của bảng kia.
  - Cột `@Encrypted` không dùng được trong `where`, `order by` hay unique.
- Khi nâng PostgreSQL bản major phải kiểm lại khớp `fold` và sinh lại cột `*_fold` nếu lệch.
