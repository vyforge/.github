# platform-expression

## 1. Mục đích

Thư viện biểu thức CEL dùng chung, có sandbox, cho mọi chỗ cần cấu hình logic không phải deploy:

- workflow: điều kiện rẽ nhánh, người duyệt theo hạn mức;
- automation: điều kiện, tham số `$expr`, chuỗi `{{ }}`;
- forms: hiện/ẩn, validate.

Biểu thức được kiểm kiểu ngay lúc lưu, chạy có giới hạn tài nguyên, và có sẵn hàm cho nghiệp vụ Việt Nam (bỏ dấu, tiền,
ngày theo múi giờ tenant).

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-expression` (**thư viện**, không phải starter; kéo `platform-core`, `dev.cel:cel` 0.14.0) |
| Package web | không có package riêng. Editor biểu thức nằm trong `@vyforge/automation` |
| Zone admin | không có |
| App liên quan | `platform-starter-workflow`, `platform-starter-automation` (phụ thuộc trực tiếp). Theo pom hiện tại, forms và `platform-forms-core` chưa phụ thuộc thư viện này; `Conditions.toCel` dùng cùng dạng điều kiện của forms (cần xác nhận kế hoạch nối) |

## 3. Tính năng chính

- **Bean `Expressions`**:
  - `compile(expr, ctx, Boolean.class)`: biên dịch và kiểm kiểu, có cache.
  - `test`, `evaluate`, `template("… {{ record.code }} …")`.
  - `trace`: giá trị từng vế, dùng cho chạy thử trong editor.
  - `catalog(ctx)`: biến, kiểu và hàm, dùng cho gợi ý trong editor.
- **Kiểu biến sinh từ JSON Schema** (`SchemaTypes.fromJsonSchema`): `object` thành struct, `format: date-time` thành
  `timestamp`, `x-expression-type: money` thành `money`. Trường sai tên báo lỗi ngay lúc biên dịch.
- **Hàm `core`**:
  - chuỗi: `fold` (bỏ dấu), `isEmpty`, `replace`;
  - tiền: `money`, `formatMoney`, `formatNumber` (dựa trên `Money` của core);
  - thời gian: `now`, `date`, `dateOf`, `startOfDay`, `addDays`, `daysBetween`, `daysUntil`, `hourOf`, `dayOfWeek`,
    `formatDate`, tính theo múi giờ tenant (mặc định `Asia/Ho_Chi_Minh`);
  - cộng thêm extension `strings`, `math`, `lists`, `cel.bind` của CEL.
- **Hàm của module khác**: cài qua bean `ExpressionFunctions` theo nhóm (`calendar`: `businessDaysBetween`, `org`:
  `inGroup`, `privacy`: `hasConsent`), bật bằng `ctx.withFunctionGroups(...)`.
- **Điều kiện dạng cây → CEL** (`Conditions.toCel`): toán tử `eq`, `in`, `gt`, `contains`, `anyOf`, `hiddenValue`…
  Giá trị thành literal nên không chèn được mã.
- **Sandbox**: giới hạn độ dài nguồn, số nút AST, độ sâu, số vòng lặp, số bước, kích thước chuỗi/collection, regex RE2J
  và timeout 2 s.
- **Lỗi** `ExpressionException`: có `phase`, `code` ổn định (`expression.type`, `expression.limit.steps`…), dòng/cột và
  message vi/en.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-expression</artifactId>
</dependency>
```

```java
ExpressionContext ctx = ExpressionContext.of("workflow:purchase-request:v2")
        .withVariable("record", SchemaTypes.fromJsonSchema(schema, "crm.PurchaseRequest"));
CompiledExpression rule = expressions.compile(
        "record.total >= money(20000000, 'VND') && fold(record.department) == 'ke toan'", ctx, Boolean.class);
boolean needsDirector = expressions.test(rule, Map.of("record", request),
        EvaluationOptions.of(ZoneId.of("Asia/Ho_Chi_Minh"), locale));
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.expression.limits.max-source-length` | `4096` | Ký tự |
| `vyforge.expression.limits.max-ast-nodes` / `.max-recursion-depth` | `500` / `32` | |
| `vyforge.expression.limits.max-comprehension-iterations` | `1000` | Cộng dồn trong một lần đánh giá |
| `vyforge.expression.limits.max-evaluation-steps` | `10000` | |
| `vyforge.expression.limits.max-result-size` / `.max-collection-size` | `65536` / `10000` | |
| `vyforge.expression.limits.timeout` | `2s` | Tính cả hàm có I/O |
| `vyforge.expression.cache.compiled` / `.environments` | `10000` / `1000` | Kích thước cache |

## 6. Tích hợp với module khác

- **SPI `ExpressionFunctions`**: module cài hàm của mình. Hàm phải thuần và tôn trọng
  `ExpressionScope.current().checkDeadline()`.
- **SPI `TextFolder`**: thay cách bỏ dấu (vd. dùng `VietnameseText` của core).
- **`ExpressionObserver`**: app tự cài khi cần metric theo ngữ cảnh.
- workflow và automation chuyển mã lỗi và vị trí lỗi vào Problem Details của API kiểm biểu thức.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Không có bảng, REST, action IAM hay sự kiện.
- Metric: `vyforge.expression.compile`, `vyforge.expression.evaluate` (tag `outcome`), `vyforge.expression.limit_exceeded`.

## 8. Trạng thái & còn mở

- Đã làm từ 0.13.
- 0.16: `platform-bom` ghim Guava 33.7.2-jre vì CEL 0.14.0 kéo Guava có CVE-2026-102554.
- Giới hạn: CEL không tự cộng `int` với `double` (phải viết `double(x)`). `format: date` của JSON Schema vẫn là `string`.
