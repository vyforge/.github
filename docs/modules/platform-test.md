# platform-test

## 1. Mục đích

Bộ công cụ test dùng chung (scope `test`), để mọi service test cùng một cách và giữ được các quy ước của platform:

- container dùng chung cho cả JVM test;
- JWT và IAM giả mà không phải mock từng lệnh `check`;
- luật ArchUnit cho B1–B22;
- kiểm schema jpa-extras, danh mục mã lỗi, tính module, cách ly tenant.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-test` (scope `test`; kéo `platform-core`, Testcontainers PostgreSQL/Kafka, ArchUnit, Spring Modulith; `-web`, `-jpa`, `-authz`, `-privacy` tuỳ chọn) |
| Package web | không áp dụng. Phía web dùng `@vyforge/test-config` 0.2.0 |
| Zone admin | không có |
| App liên quan | mọi repo Java (platform-java, gateway, auth-server, template-spring-boot) |

## 3. Tính năng chính

- **Container**:
  - `PostgresTestcontainer`, `RedisTestcontainer`, `KafkaTestcontainer`: một container cho mỗi context Spring.
  - **`SharedPostgres`**: một container cho cả JVM test, mỗi context một database mới, nên schema và dữ liệu vẫn tách
    riêng.
  - **`SharedRedis`**: mỗi context một số database Redis. Pub/sub không tách theo database: test dùng pub/sub, hoặc
    dừng container, thì giữ container riêng.
  - **`SharedKafka`**: một broker cho cả JVM test. Mỗi context dùng topic và consumer group riêng, và đặt
    `vyforge.messaging.dlt.topic-pattern` theo topic của mình.
  - `PlatformKafkaContainer`: image JVM `apache/kafka`, không dùng `kafka-native` (hay chết với mã 139). Broker chết
    thì báo ngay mã thoát và log.
  - Không bật `testcontainers.reuse` trên máy build dùng chung; Ryuk dọn container khi JVM kết thúc.
- **Bảo mật và IAM**:
  - `TestJwt.user(sub, roles…)`, `TestJwt.tenantUser(...)`.
  - `TestAuthz.builder()`: dựng `Authz` thật với policy JSON trong bộ nhớ (role, user, attribute, resource,
    `boundaries`).
  - `TestTenants.withTenant("a", …)` chạy code test trong một tenant.
- **Luật kiến trúc** (`PlatformArchitectureRules`):
  - `ALL`: không field injection, không `RestTemplate`, không package `common/util`, controller nằm trong `web` và không
    gọi repository, domain Profile B là Java thuần, không JUL, không stdout, không Joda, và **`NO_SYSTEM_TIME_ZONE`**.
  - Thêm riêng khi cần: `TENANT_ISOLATION`, `PERSONAL_DATA_ANNOTATED`, `NO_PLAIN_SCHEDULED`, `JPA_EXTRAS`,
    `SENSITIVE_PERSONAL_DATA_ENCRYPTED`.
- **`NO_SYSTEM_TIME_ZONE`**: cấm `LocalDate.now()` và `now()` không tham số của các kiểu `java.time`,
  `ZoneId.systemDefault()`, `Clock.systemDefaultZone()`, `TimeZone.getDefault()`.
  - Dùng `LocalDate.now(clock.withZone(zone))` với zone của tenant hoặc module (mặc định `Asia/Ho_Chi_Minh`).
  - Miễn bằng `@SystemTimeZoneExempt("lý do")` của platform-core.
- **Kiểm khác**:
  - `ModularityAssertions.verify(App.class)`: Spring Modulith `verify()` + `Documenter`, chặn `Type.OPEN` không có ADR.
  - `ErrorCatalogAssertions.assertComplete(...)`.
  - `TenantIsolationTests<ID>` (lớp con cho mỗi repository).
  - `AdminListContractTests` (API danh sách quản trị).
  - `JpaExtrasSchemaChecks.verify`, `JpaDdl.forEntity`, `JpaExtrasAssertions.assertVietnameseTextContract`.
  - `ContributorTestKit` (contributor của privacy-dsr).
- **Giấy phép trong test**:
  - `DevLicenseModeForTests` (JUnit `LauncherSessionListener`) đặt `vyforge.license.mode=dev`, nên starter có phí bật
    được trong test.
  - `@WithLicense` và `LicenseBuilder` cho test cần giấy phép cụ thể.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
class OrderApiTests {
    @DynamicPropertySource
    static void db(DynamicPropertyRegistry registry) {
        SharedPostgres.register(registry);          // hoặc @Bean DynamicPropertyRegistrar db() { return SharedPostgres.registrar(); }
    }
}

@AnalyzeClasses(packages = "com.vyforge.crm")
class ArchitectureTests {
    @ArchTest static final ArchRule all = PlatformArchitectureRules.ALL;
    @ArchTest static final ArchRule tenant = PlatformArchitectureRules.TENANT_ISOLATION;
}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.license.mode` | `dev` (do `DevLicenseModeForTests` đặt) | Không ghi đè giá trị đã đặt qua system property hay biến môi trường |
| `vyforge.scheduling.enabled` | (app tự đặt `false` trong test) | Không lên lịch job |
| `vyforge.http.retry.enabled` | (app tự đặt `false` khi giả lập 5xx) | Tránh request gửi 3 lần với `MockRestServiceServer` |
| `spring.test.context.cache.pause` | `ALWAYS` (chỉ trong POM gốc của platform-java) | Context cũ dừng worker ngay khi không còn lớp nào dùng |

## 6. Tích hợp với module khác

- `TestAuthz` thay `Authz` thật của `platform-starter-authz`.
- Luật `NO_PLAIN_SCHEDULED` đi với `platform-starter-scheduling`; `PERSONAL_DATA_ANNOTATED` đi với
  `platform-starter-privacy`; `JPA_EXTRAS` đi với `platform-starter-jpa`.
- Module viết adapter provider dùng test-jar hợp đồng riêng (vd. `EkycProviderContractTests` của ekyc,
  `RealtimeBrokerContractTests` của realtime).

## 7. Chế độ triển khai, quyền, sự kiện

- Chỉ dùng trong test. Không có action IAM hay sự kiện.

## 8. Trạng thái & còn mở

- Ổn định. 0.16 thêm `SharedPostgres`, `SharedRedis`, `SharedKafka`, `PlatformKafkaContainer` và chế độ dev license.
- Luật ArchUnit là xấp xỉ:
  - Luật JDBC theo tenant không đọc được câu SQL.
  - Luật PII bắt theo tên cột.

  Không thay được review.
- Role PostgreSQL là của cả cluster. Test tạo role phải xử lý `duplicate_object` khi dùng `SharedPostgres`.
