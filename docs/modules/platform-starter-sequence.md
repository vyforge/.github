# platform-starter-sequence — đánh số chứng từ

## 1. Mục đích

Ứng dụng nghiệp vụ nào cũng phải cấp số hiệu đọc được cho chứng từ: phiếu thu, phiếu chi, đề nghị thanh toán, hợp đồng,
số văn bản đi, mã hồ sơ một cửa, mã ticket, mã nhân viên. Sequence cho mẫu số có token ngày/chi nhánh/năm tài chính, reset
theo **ngày chứng từ** và múi giờ của tenant, hai chế độ `FAST` và `GAPLESS` (không lỗ, không trùng), đặt trước, xác nhận,
huỷ số có lý do, sổ cái số đã cấp và báo cáo tính liên tục. Không làm số hoá đơn điện tử (nhà cung cấp HĐĐT cấp) và khoá
chính kỹ thuật (UUID v7 của `-jpa`).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-sequence` | Từ 0.14.0; có phí (`licensed: true`), `apiPrefix` `/api/v1/sequences` |
| Package web | `@vyforge/sequence` 0.1.0 | Client server-only, `DocumentNumberBadge`, `useReservedNumber()`, component quản trị |
| Zone admin | `apps/sequence-admin` 0.1.0 (`/m/sequence`) | |
| Service | `platform-services` | Bật module `sequence`; khuyến nghị nhúng, chỉ dùng remote khi nhiều app chung dãy số |

Thiết kế: [docs/design/sequence.md](../design/sequence.md).

## 3. Tính năng chính

- **Khai báo** bằng YAML `META-INF/vyforge/sequence/*.yml` hoặc bean `SequenceSpec`; lúc chạy `SequenceCatalog.ensure(spec)`;
  tenant tự tạo bằng `SequenceAdmin.create`. Khai báo sai thì app không khởi động.
- **Mẫu số**: `{seq:N}`, `{prefix}`, `{yyyy}{yy}{mm}{dd}{q}`, `{fy}`, `{now.yyyy}`, `{<scopeKey>}`, `{ctx.<name>}`, token
  tuỳ biến (`TokenResolver`), số kiểm tra `{check:luhn|mod97|mod11}`, `|upper`/`|lower`. Kiểm tính duy nhất
  (`uniqueness: GLOBAL` hoặc `PERIOD` cho số văn bản NĐ 30/2020). Đổi mẫu tạo revision có `effectiveFrom`.
- **Reset**: `NEVER`, `YEARLY`, `MONTHLY`, `DAILY`, `FISCAL_YEAR`, `CUSTOM:<id>` (`PeriodPolicy`).
- **Cấp số** (`DocumentNumbers`): `next`, `nextBatch`, `nextAll` (nhiều bộ đếm, không deadlock), `preview`.
  - `GAPLESS`: tăng bộ đếm trong transaction của caller, rollback thì số quay về. Phải gọi trong transaction, nên cấp ở
    cuối transaction; chờ khoá quá `lock-timeout` → 503 `sequence.lock_timeout`.
  - `FAST`: cấp theo khối trong transaction riêng, có thể nhảy số; `blockSize` lớn cho tải cao.
- **Đặt trước**: `reserve`/`reserveBlock` → `confirm` hoặc `release`; quá hạn → `EXPIRED`.
- **Huỷ số**: `voidNumber(ref, reason, note)`, số giữ nguyên trong sổ cái, không bao giờ cấp lại.
- `issueManual`: nhập số từ sổ giấy/hệ thống cũ.
- **Sổ cái** `seq_issued` (bắt buộc với GAPLESS), báo cáo tính liên tục (JSON hoặc CSV), partition theo năm.
- **Đặt lại bộ đếm**: cần MFA, lý do; sequence `regulated` cần người thứ hai duyệt; trigger DB chặn hạ `next_value` bằng
  SQL tay.
- **REST cho frontend** `POST /api/v1/sequences/numbers` (header `Idempotency-Key` bắt buộc): FAST cấp ngay; GAPLESS qua
  REST luôn là đặt trước, xác nhận bằng `/numbers/{id}/confirm`. Có `/release`, `/void`, `/preview`.
- **Quản trị** `/api/v1/sequences/admin/**`: định nghĩa (`ETag`/`If-Match`), lịch sử mẫu, bộ đếm, đặt lại, sổ cái, báo cáo,
  lý do huỷ, cài đặt, việc nền, `GET /admin/info`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-sequence</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/sequence
```

Cần PostgreSQL, `platform-starter-authz`, `platform-starter-audit`, `platform-starter-scheduling`.

```yaml
# src/main/resources/META-INF/vyforge/sequence/finance.yml
sequences:
  - key: finance.cash-payment
    name: { vi: Phiếu chi, en: Cash payment voucher }
    mode: GAPLESS
    reset: YEARLY
    scopeKeys: [branch]
    prefix: PC
    pattern: "{prefix}-{branch}-{yyyy}-{seq:6}"
    regulated: true
```

```java
@Transactional
public CashPayment post(CashPayment payment) {
    repository.save(payment);
    IssuedNumber n = numbers.next(NumberRequest.of("finance.cash-payment")
            .documentDate(payment.date())
            .scope("branch", payment.branch())
            .reference("vrn:finance:cash-payment/" + payment.id())
            .idempotencyKey("post:" + payment.id())
            .build());                       // cấp số ở cuối transaction
    payment.setNumber(n.number());
    return payment;
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.sequence.enabled` | `true` | `false` tắt hẳn (không bean, migration, job) |
| `vyforge.sequence.mode` | `embedded` | `remote`: gọi `platform-services` (client `vyforge-sequence`, scope `sequence.internal`) |
| `vyforge.sequence.lock-timeout` | `5s` | GAPLESS chờ khoá tối đa |
| `vyforge.sequence.lock.hold-warn` | `200ms` | Log WARN khi giữ khoá lâu |
| `vyforge.sequence.fast.max-block-size` | `10000` | |
| `vyforge.sequence.reservation.default-ttl` / `max-ttl` | `PT30M` / `P90D` | Thời hạn số đặt trước |
| `vyforge.sequence.ledger.retention-floor-regulated-years` | `10` | Sàn lưu sổ cái sequence `regulated` |
| `vyforge.sequence.tenant-defaults.zone` / `fiscal-year-start` | `Asia/Ho_Chi_Minh` / `--01-01` | Khi tenant chưa đặt `core.timezone`, `core.fiscal-year-start` |
| `vyforge.sequence.adjust.require-mfa` | `true` | Đặt lại bộ đếm cần MFA |
| `vyforge.sequence.events.externalize` / `topic` | `false` / `vyforge.sequence` | Event ra Kafka |
| `vyforge.sequence.rate-limit.issue-per-minute` / `preview-per-minute` | `600` / `60` | Cần platform-starter-ratelimit |
| `vyforge.sequence.void-reasons` | — | Lý do huỷ thêm của dự án |
| `vyforge.sequence.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/sequence.sql` |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: organization, workflow, settings, ratelimit.

- **settings**: đọc `core.timezone`, `core.fiscal-year-start`; `PUT /admin/settings` ghi tầng TENANT (không có settings thì
  `409 sequence.settings_read_only`).
- **organization**: phạm vi `branch`/`legalEntity` lấy từ đơn vị cha gần nhất có cờ `BRANCH_LIKE`/`LEGAL_ENTITY` của người
  dùng hiện tại (`vyforge.sequence.organization.*`).
- **workflow**: cài port `DocumentNumbering` (`WorkflowDocumentNumbering`) để cấp số hồ sơ; rút/huỷ hồ sơ thì huỷ số.
- **privacy-dsr**: `SequencePersonalData` (`PersonalDataContributor`, khoá `sequence`); yêu cầu xoá trả
  `Retained(LEGAL_OBLIGATION, Luật Kế toán 88/2015/QH13 Điều 41)`.
- **ratelimit**: giới hạn cấp/xem trước theo người dùng.
- SPI: `TokenResolver`, `ScopeValueResolver`, `PeriodPolicy`, `PeriodGuard`, `NumberingInterceptor`, `ReferenceChecker`.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** (khuyến nghị) hoặc **remote**. Ở remote, GAPLESS thành ngữ nghĩa **ACCOUNTED** (đặt trước, xác nhận ở
  `beforeCommit`, rollback thì huỷ với lý do `rollback`); sequence `regulated` mặc định `EMBEDDED_ONLY`, không cấp qua
  remote (`sequence.gapless_requires_embedded`).
- **Quyền IAM** (resource `vrn:sequence:<type>/<key>`): `sequence:definition:List|Get|Create|Update|SetStatus`,
  `sequence:pattern:Update`, `sequence:counter:List|Adjust|ApproveAdjust`, `sequence:number:Issue|Void`,
  `sequence:ledger:List|Export`, `sequence:settings:Get|Update`, `sequence:task:Manage`. API Java `DocumentNumbers` không
  kiểm quyền (app tự kiểm nghiệp vụ).
- **Sự kiện** (`SequenceEvent`, `TenantAwareEvent`): `SequenceNumberVoided`, `SequenceReservationExpired`,
  `SequenceCounterAdjusted`, `SequenceDefinitionChanged`, `SequenceCapacityWarning`, `SequenceNumberIssued` (tắt mặc định).
- Job: `sequence-reservation-sweep`, `sequence-partitions`, `sequence-retention`. Metric `vyforge.sequence.*`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm event qua Kafka, rate limit REST, privacy-dsr, phạm vi theo organization,
  settings, detach partition.
- **Chờ xác nhận** (thiết kế 10.3): chấp nhận ngữ nghĩa `ACCOUNTED` cho GAPLESS ở remote; port `DocumentNumbering` của
  workflow có tách `documentDate`/`reference` không; các văn bản pháp lý đánh dấu "chưa kiểm chứng toàn văn" cần đối chiếu
  khi có dự án kế toán, văn thư, một cửa thật.
- **Giới hạn đã biết**: thông lượng GAPLESS ≈ 1 / thời gian giữ khoá (đo trên máy dev: ~107 số/giây khi giữ khoá 0 ms,
  14 số/giây khi 50 ms); FAST `blockSize: 50` ~3600 số/giây. Cần đo lại trên môi trường thật. Không đưa CCCD/số điện thoại
  vào `scope`/`ctx`.
