# platform-bom và platform-parent

## 1. Mục đích

Hai POM giúp mọi backend Vyforge dùng chung một bộ version và một bộ kiểm chất lượng (B1). Dự án kế thừa
`platform-parent` thì chỉ khai dependency, không ghi version. Các module platform-java được ghim theo một version duy
nhất, cùng Java 25, Spring Boot 4.1.1 và thư viện bên thứ ba đã vá CVE. Build tự kiểm định dạng, độ phủ test, dependency
bị cấm và sinh SBOM.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-bom` (import scope), `com.vyforge.platform:platform-parent` (parent POM, kế thừa `spring-boot-starter-parent` 4.1.1) |
| Package web | không áp dụng. Phía web có `@vyforge/eslint-config`, `@vyforge/prettier-config`, `@vyforge/tsconfig`, `@vyforge/test-config` |
| Zone admin | không có |
| App liên quan | `template-spring-boot`, `platform-gateway` (parent 0.16.0), `platform-auth-server` (parent 0.15.0 ở v0.2.1), mọi backend dự án |

## 3. Tính năng chính

- **`platform-bom`** ghim version cho:
  - mọi thư viện `platform-*` (trừ hai app `platform-iam-server` và `platform-services`);
  - Spring Modulith, springdoc, ArchUnit, jMolecules (Profile B), Pact (contract test K5);
  - Resilience4j, ShedLock, Bucket4j;
  - thư viện riêng của từng starter: Flowable, CEL, Spring AI, Tika, ical4j, Liquid…
  - test-jar `platform-starter-ekyc` (hợp đồng adapter provider).
- **Bản vá bảo mật trong bom**: Guava 33.7.2-jre (CVE-2026-102554, bản 0.16.0), snappy-java 1.1.10.9 (CVE-2026-108106,
  CVE-2026-93451, bản 0.16.1), lz4-java. Có chú thích "xoá khi thư viện gốc đã nâng".
- **`platform-parent`**:
  - Java 25.
  - **Enforcer**: Java ≥ 25, Maven ≥ 3.9, `dependencyConvergence`, cấm `junit:junit` và `com.h2database:h2`.
  - **Spotless** (palantir-java-format) chạy `check` ở `verify`.
  - **JaCoCo**: độ phủ dòng tối thiểu 60% (`jacoco.minimum.line.coverage`).
  - **CycloneDX SBOM**: `makeBom` (thay `makeAggregateBom` vì chạy song song nhanh hơn) ghi
    `META-INF/sbom/application.cdx.json` vào jar, Actuator đọc được.
  - **Profile `-Derrorprone`**: Error Prone + NullAway (K3). Mặc định WARN; `-Dnullaway.severity=ERROR` khi đã sạch.
  - Ghim bản vá Tomcat/Jackson mới hơn bản Boot quản lý.
- **Quy tắc ArchUnit `NO_SYSTEM_TIME_ZONE`** (nằm trong `PlatformArchitectureRules.ALL` của `platform-test`, không nằm
  trong POM):
  - Cấm dùng múi giờ mặc định của JVM: `LocalDate.now()`, `ZoneId.systemDefault()`, `Clock.systemDefaultZone()`,
    `TimeZone.getDefault()`.
  - Ngày và kỳ nghiệp vụ tính theo zone của tenant hoặc module (mặc định `Asia/Ho_Chi_Minh`) với `Clock` inject được.
  - Miễn bằng `@SystemTimeZoneExempt("lý do")`.

## 4. Cách dùng nhanh

```xml
<parent>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-parent</artifactId>
    <version>0.16.1</version>
    <relativePath/>
</parent>
```

Dự án không dùng parent thì import bom:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.vyforge.platform</groupId>
            <artifactId>platform-bom</artifactId>
            <version>0.16.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

POM không khai `<repositories>` (ADR-0014). Registry nằm trong `settings.xml`, sinh bằng
`vyforge-ci maven-settings` từ `VYFORGE_MAVEN_URL`, `REGISTRY_USER`, `REGISTRY_TOKEN`.

## 5. Cấu hình quan trọng

| Thuộc tính / cờ | Mặc định | Ý nghĩa |
|---|---|---|
| `java.version` | `25` | |
| `jacoco.minimum.line.coverage` | `0.60` | Ngưỡng độ phủ dòng |
| `-Derrorprone` | tắt | Bật Error Prone + NullAway; platform-java luôn bật qua `.mvn/maven.config` |
| `nullaway.severity` | `WARN` | `ERROR` khi đã sạch |
| `revision` | `.mvn/maven.config` | Version CI-friendly khi build repo platform-java |

## 6. Tích hợp với module khác

- `platform-test` cung cấp luật ArchUnit để thực thi quy ước. POM chỉ kiểm build.
- CI dùng `vyforge-ci java-verify` (repo platform-ci). Tag `vX.Y.Z` chạy `maven-publish` cho mọi module thư viện; app
  phát hành dạng image.

## 7. Chế độ triển khai, quyền, sự kiện

- Không áp dụng.

## 8. Trạng thái & còn mở

- Phiên bản hiện tại **0.16.1** (chỉ thêm ghim snappy-java so với 0.16.0).
- Theo semver: đổi hành vi mặc định hoặc xoá API public là major; trước 1.0 vẫn có thay đổi phá vỡ ở minor (vd. 0.16:
  starter có phí cần giấy phép, `EntityRef` của files).
- Các dòng ghim bản vá cần gỡ khi Spring Boot hoặc thư viện gốc đã nâng.
