# platform-starter-master-data — danh mục dùng chung

## 1. Mục đích

Ứng dụng nghiệp vụ VN nào cũng cần cùng một bộ danh mục: đơn vị hành chính (đã đổi lớn từ 01/07/2025), ngân hàng, quốc
gia, tiền tệ, ngày lễ, ngành kinh tế, dân tộc, đơn vị tính, cùng danh mục riêng của từng tenant. Master-data cung cấp danh
mục **có version và ngày hiệu lực**, đọc nhanh từ snapshot trong bộ nhớ, sửa theo quy trình nháp → kiểm → (duyệt) → công
bố, kèm xử lý địa chỉ VN (chuyển địa chỉ cũ sang mới), tỷ giá và geocoding.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-master-data` | Từ 0.14.0; có phí (`licensed: true`). Tên `masterdata` trong Flyway, IAM, metric, cấu hình |
| Package web | `@vyforge/master-data` 0.2.0 | Client server-only, component chọn ĐVHC/địa chỉ/ngân hàng/tiền tệ, hook, component quản trị |
| Zone admin | `apps/master-data-admin` 0.1.0 (`/m/master-data`) | |
| Công cụ | `tools/masterdata-packs` (`PackBuilder`) | Dựng và ký data pack |
| Service | `platform-services` | Bật module `master-data`; app khác dùng chế độ `remote` |

Thiết kế: [docs/design/master-data.md](../design/master-data.md).

## 3. Tính năng chính

- **Danh mục SYSTEM** đóng sẵn: ĐVHC VN (34 tỉnh, 3.321 xã từ 01/07/2025, thay đổi 2026, dữ liệu 63 tỉnh/3 cấp trước đó,
  đồ thị kế thừa xã cũ → mới), ngân hàng (BIN NAPAS, SWIFT, VietQR), quốc gia, tiền tệ, ngôn ngữ (ISO/CLDR), lịch nghỉ
  quốc gia (`VN`, `VN-GOV`), VSIC 2025/2018, dân tộc, đơn vị tính. Tenant tạo thêm danh mục riêng.
- **Facade đọc**: `AdministrativeUnits` (`provinces`, `wards`, `get(code, asOf)`), `AddressMigration.convert` (kết quả
  `Exact` hoặc `DefaultOfSplit` cần người xác nhận), `Addresses.parse|normalize|format`, `Banks`, `Currencies`,
  `PublicHolidays`, `Lookups.validate` (mọi danh mục, kể cả của tenant).
- Đọc từ snapshot bất biến theo (danh mục, revision), không I/O; đọc theo `asOf` bất kỳ hoặc revision cũ (báo cáo tái
  lập). Mã không bao giờ bị xoá: hết hiệu lực thì `validate` trả `EXPIRED` kèm mã kế thừa.
- Tenant thấy danh mục của mình + danh mục dùng chung đã áp **overlay** (ẩn, sắp xếp, tên gọi khác).
- **Sửa** (`MasterDataAdmin`): `put`, `diff`, `validate`, `publish` (kèm căn cứ pháp lý), `revert`, `compact`; duyệt
  `submit` → `approve` cho danh mục `requiresApproval`. Một nháp mỗi danh mục.
- **Import/export** CSV, JSON, XLSX (chạy nền vào nháp).
- **Data pack** ký Ed25519 trên JSON chuẩn hoá RFC 8785: pack đóng trong starter nạp khi danh mục rỗng; pack mới nhập qua
  `POST /api/v1/master-data/admin/packs`, phải ký bằng khoá tin cậy.
- **Tỷ giá** (`ExchangeRates.quote|convert` theo mục đích `INVOICE`, `DISPLAY`…): nguồn `vietcombank`, `ecb`, `sbv`
  (`CENTRAL`, `CUSTOMS`), `file`, `manual`; tính chéo qua VND/EUR/USD; lưu `rateIds` để tái lập.
- **Geocoding**: mặc định `local` (không gửi địa chỉ ra ngoài); Nominatim tự host, `vietmap`, `goong`, `google` (cần tài
  khoản). Kết quả luôn ánh xạ về mã ĐVHC.
- REST `/api/v1/master-data`: `lists/**` (ETag, 304), `vn/admin-units/**`, `vn/addresses:parse|:normalize|:format|:migrate`,
  `banks`, `holidays`, `fx/quote`, `fx/rates`, `geo/*`, `admin/**`, `internal/**`. Danh mục `PUBLIC` đọc không cần token
  khi app mở đường đó.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-master-data</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/master-data
```

Cần `DataSource`, `platform-starter-authz`, `platform-starter-audit`; `platform-starter-scheduling` cho import nền và job
tỷ giá. Có extension `btree_gist` thì có thêm ràng buộc không chồng hiệu lực ở DB.

```java
List<AdminUnit> provinces = administrativeUnits.provinces(LocalDate.now());          // 34 tỉnh
MigrationResult r = addressMigration.convert(new VnAddress.LegacyPart("79", "764", "26881"), null, LocalDate.now());
banks.byBin("970436");                                                               // Vietcombank
lookups.validate("crm.lead-source", "WEB", LocalDate.now());
FxQuote q = exchangeRates.quote(CurrencyPair.of("USD", "VND"), Instant.now(), FxPurpose.INVOICE);
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.masterdata.mode` | `embedded` | `remote`: đọc snapshot từ `platform-services` (client `vyforge-masterdata`) |
| `vyforge.masterdata.operator-tenant` | `default` | Tenant được sửa danh mục dùng chung |
| `vyforge.masterdata.sync.revision-poll` | `60s` | Poll revision khi không có Valkey |
| `vyforge.masterdata.packs.seed` | `true` | Nạp pack đóng sẵn khi danh mục rỗng |
| `vyforge.masterdata.packs.trusted-keys.<kid>` | — | Khoá công khai Ed25519 để kiểm pack |
| `vyforge.masterdata.packs.https-allowed-hosts` | — | Host được tải pack qua URL |
| `vyforge.masterdata.packs.require-signature-for-classpath` | `false` | Bắt chữ ký cả với pack đóng sẵn |
| `vyforge.masterdata.lists.default-max-lists` / `default-max-items` | `50` / `20000` | Hạn mức khi chưa có entitlement |
| `vyforge.masterdata.fx.sources.<nguồn>.enabled` | `false` | Bật nguồn tỷ giá; `min-interval` mặc định 5 phút |
| `vyforge.masterdata.fx.default-policies` | — | Chính sách tỷ giá theo mục đích |
| `vyforge.masterdata.geo.providers.<tên>.enabled` | `false` | Bật nhà cung cấp geocoding (`nominatim`, `vietmap`, `goong`, `google`) |
| `vyforge.masterdata.authz.operator-role` | `MASTERDATA_ADMIN` | Role ở tenant vận hành nhận policy quản trị danh mục dùng chung |
| `vyforge.masterdata.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/masterdata.sql` |

## 6. Tích hợp với module khác

- **calendar**: dữ liệu lịch nghỉ quốc gia ở master-data; calendar dùng qua SPI `NationalHolidaySource`.
- **payment**: trường merchant `lookup:bank` lấy từ danh sách ngân hàng (0.16.0).
- **features**: hạn mức `masterdata.lists.max`, `masterdata.items.max` theo entitlement.
- **forms**: lựa chọn ĐVHC lấy từ master-data (`forms.integration.masterdata`). **custom-fields**: trường lookup tới danh mục (cần xác nhận phạm vi).
- **settings**: khoá nhà cung cấp geocoding của tenant mã hoá bằng `SecretCipher`.
- App nhúng `AddressColumns` (street, toạ độ có `@PersonalData`) để lưu địa chỉ; module không lưu địa chỉ người dùng.
- SPI: `PublishInterceptor`, `ItemValidator`, `ListDefinitionContributor`, `ExchangeRateSource`.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote** (`Lookups` và facade đọc snapshot tải từ server; `ExchangeRates` gọi server; không
  migration, không REST ở app).
- **Quyền IAM**: `masterdata:list:List|Get|Create|Update|Delete|Compact`, `masterdata:item:Edit`, `masterdata:overlay:Put`,
  `masterdata:release:Submit|Approve|Publish|Revert`, `masterdata:pack:Import`, `masterdata:fx:Read|Record|Import|ManageSources`,
  `masterdata:geo:Use`, `masterdata:settings:Get|Update`.
- **Sự kiện** (outbox Spring Modulith): `MasterDataPublished`, `AdministrativeUnitsChanged` (khi `vn-admin-unit` có cạnh kế
  thừa mới).
- Metric `vyforge.masterdata.*`; health `masterdata` (`OUT_OF_SERVICE` khi danh mục SYSTEM chưa có revision).

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm adapter địa lý, nguồn tỷ giá, XLSX, compaction, giới hạn theo entitlement, ký
  gói dữ liệu.
- **Chờ** (thiết kế 10.3): Q3 ai giữ khoá ký pack, pack phát hành ở đâu (repo chưa có secret thì pack không ký); Q4 pháp
  chế xác nhận việc dùng dữ liệu phụ lục văn bản pháp luật và BIN/SWIFT để phát hành pack ra ngoài.
- **Giới hạn đã biết**: geocoding `google` không cache theo điều khoản; Nominatim công cộng chỉ 1 req/s, không dùng cho
  autocomplete. Compaction làm revision cũ không đọc lại được (410).
