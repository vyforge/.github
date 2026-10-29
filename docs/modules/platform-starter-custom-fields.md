# platform-starter-custom-fields — trường tuỳ biến theo tenant

## 1. Mục đích

Mỗi khách hàng muốn thêm vài trường riêng vào bản ghi ("Mã khách hàng cũ", "Nhóm máu", "Hạn mức tín dụng") mà không phải
sửa code, thêm cột hay deploy lại. Custom-fields cho tenant admin tự thêm trường cho các loại bản ghi app khai báo mở
rộng được, kiểm giá trị khi lưu cùng transaction, lọc/sắp xếp theo trường đó, và đổi kiểu có chuyển dữ liệu an toàn. Giá
trị nằm ở khoá `cf` của cột `attributes jsonb` có sẵn: không DDL lúc chạy, không bảng EAV.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-custom-fields` | Từ 0.14.0; có phí (`licensed: true`). Tên `customfields` trong Flyway, IAM, metric |
| Lõi kiểu trường | `platform-forms-core` | Cùng JSON Schema, bộ kiểm, mã lỗi `forms.field.*` với forms |
| Package web | `@vyforge/custom-fields` 0.1.1 | Client, cột/bộ lọc/schema cho admin-kit, khối form, khu quản trị |
| Zone admin | `apps/custom-fields-admin` 0.1.0 (`/m/custom-fields`) | |

Thiết kế: [docs/design/custom-fields.md](../design/custom-fields.md).

## 3. Tính năng chính

- **Khai báo loại bản ghi**: entity `@ExtensibleType("crm.customer")` implement `Extensible` (cột `attributes jsonb`) +
  bean `EntityTypeDeclaration` (nhãn, bảng, action đọc/ghi, search index, `maxFields`, `conditionAttributes`).
- **Kiểu trường**: chữ, đoạn văn, email, URL, SĐT VN, CCCD, MST, số nguyên, số, tiền VND, ngày, ngày giờ, chọn một/nhiều,
  có/không, địa chỉ VN (khi có master-data). Thêm kiểu bằng `FieldStorage`.
- **Kiểm khi lưu** (listener Hibernate gọi `CustomFieldValues`): chuẩn hoá, JSON Schema, quyền `writeAction` theo trường,
  trường ngừng dùng chỉ đọc, khoá lạ bị từ chối, mặc định, `unique` (chỉ mục unique partial), kích thước, mã hoá trường
  `encrypted`. **Kiểm khi chạm**: bản ghi cũ chỉ lỗi ở trường bị đổi, trường khác sai luật mới thành cảnh báo (header
  `Vyforge-Warnings`). Điều kiện `visibleWhen`/`requiredWhen` tham chiếu thuộc tính gốc của bản ghi.
- **Đọc**: `CustomFieldValues.read(...)` lọc trường theo quyền đọc, giải mã (audit khi đọc dữ liệu nhạy cảm), `display` theo
  locale; `@ExtensibleView` lọc `cf` khi ghi JSON.
- **Lọc, sắp xếp** theo quy ước admin-kit (`cf.source=fb`, `cf.areaFrom=50`, `sort=cf.area,desc`) qua
  `queries.specification(...)` (JPA), `queries.sql(...)` (JDBC/jOOQ), `queries.sort(...)`. Chỉ mục biểu thức partial theo
  tenant (`cfx_*`) dựng nền `concurrently`; hỗ trợ bảng phân vùng.
- **Đổi định nghĩa**: phân loại trước (`plan`): `SAFE`, `VALIDATING` (chấp nhận, `BACKFILL`, `REMAP`), `CONVERTING` (đổi kiểu
  trong ma trận, `ENCRYPT`/`DECRYPT`, `rekey`), `FORBIDDEN`. Giá trị không chuyển được: `BLOCK`, `CLEAR`, `KEEP_AS_LEGACY`.
  `COPY` sang trường mới khi đổi kiểu ngoài ma trận. Migration nền theo lô, hoàn tác được trong 30 ngày.
- Xoá trường là xoá mềm 15 ngày, khôi phục được, hết hạn thì job dọn giá trị.
- **Gói mẫu** `META-INF/vyforge/custom-fields/packs/*.pack.json`; xuất/nhập định nghĩa giữa tenant/môi trường.
- `lookup-failure: ACCEPT_UNVERIFIED`: nguồn lựa chọn không trả lời thì vẫn nhận, job kiểm lại sau.
- REST `${api.base-path}/custom-fields`: `GET /entity-types/{type}/schema` (ETag), `/admin/**` (fields, plan, change-type,
  copy, rekey, migrations, indexes, packs, export/import, lookups/unverified).

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-custom-fields</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/custom-fields
```

```java
@Entity
@Table(name = "crm_customer")
@ExtensibleType("crm.customer")
class Customer extends TenantEntity implements Extensible {
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "attributes", nullable = false)
    private Map<String, Object> attributes = new LinkedHashMap<>();
}

@Bean
EntityTypeDeclaration customerType() {
    return EntityTypeDeclaration.builder("crm.customer")
            .label("Khách hàng", "Customer")
            .table("crm_customer")
            .actions("crm:customer:Get", "crm:customer:Update")
            .build();
}
```

Migration của app:

```sql
alter table crm_customer add column if not exists attributes jsonb not null default '{}'
    check (jsonb_typeof(attributes) = 'object');
create index if not exists ix_crm_customer_cf on crm_customer using gin ((attributes -> 'cf') jsonb_path_ops);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.custom-fields.enabled` | `true` | |
| `vyforge.custom-fields.jpa-listener` | `true` | Kiểm tự động khi entity lưu |
| `vyforge.custom-fields.api.base-path` | `/api/v1` | App có thể đặt dưới service của mình (vd. `/api/v1/crm`) |
| `vyforge.custom-fields.limits.max-fields-per-type` | `300` | Trần mỗi loại bản ghi |
| `vyforge.custom-fields.limits.max-cf-bytes` | `32768` | |
| `vyforge.custom-fields.index.mode` | `AUTO` | `MANUAL`: chỉ sinh DDL vào `cf_index` cho DBA chạy |
| `vyforge.custom-fields.index.max-per-table` | `50` | Hạ từ 200 ở 0.16.0 (benchmark ghi chậm) |
| `vyforge.custom-fields.migration.plan-token-secret` | — | Phải chung cho mọi replica |
| `vyforge.custom-fields.defaults.validation-mode` | `TOUCH` | `STRICT`: kiểm đủ mọi lần lưu |
| `vyforge.custom-fields.defaults.delete-retention` / `backup-retention` | `15d` / `30d` | |
| `vyforge.custom-fields.defaults.index-min-rows` | `20000` | Từ ngưỡng này mới dựng chỉ mục `cfx_` |
| `vyforge.custom-fields.defaults.lookup-failure` | `REJECT` | `ACCEPT_UNVERIFIED` |
| `vyforge.customfields.schema.migrate` | `true` | Bảng `cf_*` |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: audit, features.

- **forms (`platform-forms-core`)**: dùng chung kiểu trường, `ChoiceSource` (danh mục master-data, đơn vị tổ chức).
- **jpa**: listener Hibernate, `SaveContext`, `ExtensionGuard` của platform-core (mở rộng ở 0.16.0).
- **features**: hạn mức `custom-fields.fields`, `custom-fields.indexed`, `custom-fields.encryption`.
- **admin-kit**: cột, bộ lọc, schema cho `DataTable` qua `@vyforge/custom-fields`.
- **search**: event `CustomFieldSchemaChanged` mang `searchImpact` (`NONE`/`FILTER`/`REINDEX`).
- **privacy-dsr**: event `CustomFieldPersonalDataChanged`; app dùng `CustomFieldValues.erasePersonalData` trong contributor.
- Valkey (kênh `vyforge:customfields:changed`): xoá cache schema mọi replica ngay.
- SPI: `EntityTypeDeclaration`, `FieldStorage`, `ValueConverter`, `CustomFieldInterceptor`, `ExtensibleDataAccess`,
  `ConditionAttributeSource`, `FieldPackProvider`, `CustomFieldsAuthorizer`, `CustomFieldsQuota`, `CustomFieldsSettings`.

## 7. Chế độ triển khai, quyền, sự kiện

- **Chỉ nhúng** trong app sở hữu bảng (kiểm cùng transaction, migration ghi bảng của app). Không có chế độ remote.
- **Quyền IAM**: `customfields:field:List|Get|Create|Update|ChangeType|Delete|Restore|Purge|ManageAccess`,
  `customfields:migration:List|Cancel|Retry|Rollback`, `customfields:pack:Apply`, `customfields:definition:Export|Import`,
  `customfields:index:Manage`. Từng trường còn có `readAction`/`writeAction` riêng.
- **Sự kiện** (không chứa giá trị bản ghi): `CustomFieldSchemaChanged`, `CustomFieldMigrationChanged`,
  `CustomFieldIndexChanged`, `CustomFieldPersonalDataChanged`.
- Metric `vyforge.customfields.*`; health `customFields` (`DEGRADED` khi thiếu chỉ mục GIN).

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm `ENCRYPT`/`DECRYPT`/`rekey`, `COPY`, `KEEP_AS_LEGACY`, `conditionAttributes`,
  `ACCEPT_UNVERIFIED`, invalidation qua Valkey, header `Vyforge-Warnings`, chỉ mục cho bảng phân vùng.
- **Chưa làm** (README): tích hợp search (`dynamicFields`), export, privacy-dsr, master-data, settings ngoài event (bộ nối
  nằm ở module đó); kiểu `file`; trường tính toán.
- **Chờ xác nhận** (thiết kế 11.3): chấp nhận "chỉ nhúng"; DDL chỉ mục lúc chạy (khách on-prem không cho `CREATE INDEX`
  thì dùng `index.mode: MANUAL`); số `delete-retention` 15 ngày và `backup-retention` 30 ngày.
- **Số đo** (29/10/2026, máy build dùng chung): kiểm 100 trường p95 0,66 ms, 300 trường p99 9,68 ms; 50 chỉ mục `cfx_` làm
  ghi chậm 25–60 %. Trách nhiệm phân loại dữ liệu cá nhân của trường thuộc tenant.
