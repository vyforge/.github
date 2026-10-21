> Báo cáo nghiên cứu ngày 21/10/2026 — căn cứ cho [ARCHITECTURE.md](../ARCHITECTURE.md). Các quyết định cuối cùng (kể cả chỗ khác báo cáo, vd. kết quả spike Mongock, `localePrefix`) nằm trong [ADR](../adr/).

# Chuẩn hoá template Vyforge quanh module nghiệp vụ

Vyforge nên dùng **một khung chung cho mọi dự án, cấu hình theo quy mô**. Backend dùng **modular monolith với Spring Modulith**: package chia theo module nghiệp vụ, ranh giới module được kiểm tra bằng test, các module nói chuyện với nhau qua event. Bên trong mỗi module có hai profile: "simple" (phẳng hoặc layered) cho phần CRUD, và "hexagonal" chỉ cho domain lõi phức tạp ở dự án tier 2. Frontend dùng **App Router mỏng + `features/` + `shared/`**, ranh giới import được ESLint kiểm tra. Server Component gọi Spring qua client sinh từ OpenAPI. Next.js đóng vai BFF: giữ token phía server, còn Spring là OAuth2 resource server xác thực JWT từ Keycloak. Về hạ tầng chung, cần chuyển các workflow template hiện có trong `vyforge/.github` sang **reusable workflow có version**, sau đó thêm Renovate preset, Helm chart dùng chung và parent POM. Template `template-*` chỉ giữ phần khung repo, và được phân phối bằng **Copier** để các repo con nhận được bản cập nhật. GitHub template repo chỉ copy một lần nên không đáp ứng được việc này. Ba lý do khiến việc này cấp bách. Spring Boot 4 là một major break (Jackson 3, starter tách module, `spring.mongodb.*`). Next.js đã có ba lỗ hổng RCE nghiêm trọng liên quan RSC trong khoảng 11 tháng. Mongock chưa có dấu hiệu hỗ trợ Boot 4. Template là nơi khoá các quyết định này một lần cho mọi dự án. Còn bốn quyết định công ty phải chốt: Maven hay Gradle, công cụ migration MongoDB, hexagonal mặc định hay chỉ ở tier 2, và Renovate hay Dependabot. Báo cáo đưa khuyến nghị cụ thể cho từng quyết định và kết thúc bằng lộ trình triển khai 4 giai đoạn.

> Phạm vi và độ tin cậy. Báo cáo tổng hợp từ bốn bộ ghi chú nghiên cứu ngày 21/10/2026. Phiên bản thư viện Java được đối chiếu trực tiếp với Maven Central. Nhiều trang tài liệu chính thức (docs.spring.io, nextjs.org, feature-sliced.design) không truy cập được, nên một số khẳng định dựa vào bản tóm tắt từ kết quả tìm kiếm hoặc nguồn thứ cấp. Những chỗ đó được đánh dấu **(chưa kiểm chứng)**.

## Backend nên chia theo nghiệp vụ trước, chia theo tầng kỹ thuật sau

### Sáu kiểu kiến trúc trả lời hai câu hỏi khác nhau

Layered, hexagonal, clean và onion trả lời câu hỏi *"tách code kỹ thuật khỏi code domain thế nào"*. Package-by-feature, vertical slice và modular monolith trả lời câu hỏi *"cắt hệ thống theo năng lực nghiệp vụ thế nào"*. Các ý kiến có uy tín hiện nay đều xếp câu hỏi thứ hai lên trước:

- **Simon Brown**, trong chương 34 sách *Clean Architecture* ("the missing chapter"), so sánh package by layer, by feature, ports & adapters và package by component. Ông kết luận rằng encapsulation theo component mới là điểm mấu chốt ([O'Reilly ch.34](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/ch34.xhtml)). Hình tròn đồng tâm của Clean Architecture nói về hướng phụ thuộc, không phải về cách đặt thư mục ([Milan Jovanović](https://milanjovanovic.tech/blog/clean-architecture-the-missing-chapter)).
- **Oliver Drotbohm** (Spring team) nhận xét rằng hexagonal và onion "cho kết quả lẫn lộn" vì chúng không xử lý cấu trúc của domain. Nguồn của ý này chỉ là đoạn trích từ kết quả tìm kiếm, vì không tải được trang gốc ([JUG.ch 2026](https://www.jug.ch/html/events/2026/domain-centric.html); [tóm tắt talk](https://ishansoninitj.medium.com/creating-a-multi-module-monolith-using-spring-modulith-f83053736762)).
- **ThoughtWorks** đưa Spring Modulith vào vòng "Assess" năm 2023, như liều thuốc chữa "microservice envy". Blip này đã cũ và không còn trên radar hiện tại ([TW Radar](https://www.thoughtworks.com/radar/languages-and-frameworks/spring-modulith)).

| Kiểu | Điểm mạnh | Điểm yếu | Hợp với | Boilerplate |
|---|---|---|---|---|
| Layered cổ điển (controller/service/repository ở top-level) | Ai cũng biết, khởi động nhanh nhất | Code một tính năng rải khắp các package; mọi thứ phải `public`; không có ranh giới module | 1–3 dev, CRUD ngắn hạn | Rất thấp |
| Package-by-feature | Gắn kết cao; dùng được package-private; dễ xoá hoặc tách | Dễ bị gọi chéo giữa các feature nếu không có công cụ chặn | 2–8 dev | Thấp |
| Hexagonal (ports & adapters) | Domain độc lập với Mongo/Kafka/Redis; unit test domain thuần | Số type gấp đôi (port + mapper); thừa với CRUD; không nói gì về cách cắt domain | 4+ dev, domain phức tạp, sống lâu | Cao |
| Clean (use-case class, vòng đồng tâm) | Use case tường minh | Nhiều thủ tục nhất; thường bị áp dụng thành cấu trúc thư mục | Domain lớn | Cao nhất |
| **Modular monolith + Spring Modulith** | Ranh giới module được công cụ kiểm tra; event giữa module; test theo module; sinh tài liệu; sẵn sàng tách service | Phải thiết kế module từ đầu; cần học event, registry, named interface; event registry trên Mongo cần replica set | 3–30+ dev | Thấp–trung bình |
| Vertical slice | Mỗi use case một class hoặc thư mục; ít abstraction | Chấp nhận trùng lặp; domain chung yếu; dev Java ít quen | Lead có kinh nghiệm | Thấp |

Bảng trên là đánh giá định tính tổng hợp từ ghi chú. Không có nghiên cứu định lượng nào (thời gian onboarding, tỉ lệ lỗi) so sánh các kiểu này. Vertical slice không được khuyến nghị cho ứng dụng ít tính năng; nguồn này là thứ cấp ([issoh.co.jp](https://www.issoh.co.jp/tech/details/10356/)). Bogard, người đặt ra thuật ngữ, năm 2026 quảng bá VSA + DDD như "guardrail cho phát triển có AI hỗ trợ" ([sessionize](https://sessionize.com/s/jimmy-bogard/vertical-slice-architecture-effective-guardrails-f/185766)). Ý tưởng này đáng chú ý, nhưng chưa đủ cơ sở để Vyforge lấy làm mặc định.

**Kết luận: Spring Modulith không thay thế hexagonal mà nằm ở tầng trên.** Module là lát cắt ngoài cùng. Bên trong, từng module có thể phẳng, layered hoặc hexagonal. Hexagonal đặt bên trong module Modulith là một pattern cộng đồng đã được mô tả ([codecentric](https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture)). jMolecules cung cấp các annotation `@PrimaryPort`, `@SecondaryPort`, `@PrimaryAdapter`, `@SecondaryAdapter`, và Modulith tự nhận diện các thành phần jMolecules ([jMolecules README](https://github.com/xmolecules/jmolecules)). Hombergs, tác giả sách hexagonal cho Java được dùng nhiều nhất, cũng chấp nhận các "shortcut" có chủ đích khi một use case quá đơn giản ([Leanpub](https://leanpub.com/sh/PtkHnaSf)).

### Quy tắc Modulith quyết định hình dạng package

Mỗi package con trực tiếp của package chứa `@SpringBootApplication` là một module. **Các type public ở package gốc của module là API của module, còn mọi package con mặc định là internal.** Compiler Java không chặn được việc gọi một type public nằm trong package internal, nên phải dùng `verify()` để bắt lỗi này ([fundamentals.adoc](https://raw.githubusercontent.com/spring-projects/spring-modulith/main/src/docs/antora/modules/ROOT/pages/fundamentals.adoc)). Muốn mở một package con cho module khác dùng thì đánh dấu `@NamedInterface("api")` trong `package-info.java`, và module dùng khai báo `allowedDependencies = {"order :: api"}`. `Type.OPEN` chỉ dành cho việc di chuyển code legacy. Từ bản 1.3 có thêm nested module. Modulith 2.0 kiểm tra cấu trúc ngay khi ứng dụng khởi động và đọc mô tả module từ Javadoc của `package-info.java` ([Modulith 2.0 GA](https://spring.io/blog/2025/11/21/spring-modulith-2-0-ga-1-4-5-and-1-3-11-released/)). Spring chưa có sample chính thức nào quy định cấu trúc `{api,application,domain,infrastructure}` bên trong module. Hai cây dưới đây do báo cáo suy ra từ các quy tắc trên.

**Profile A — "simple module"** (mặc định cho mọi module CRUD):

```
com.vyforge.crm
├── CrmApplication.java              @SpringBootApplication @Modulithic(sharedModules = "shared")
├── shared/                          shared kernel rất nhỏ: Money, ID, error model
├── customer/                        ← gốc module = public API
│   ├── package-info.java            @ApplicationModule(displayName = "Customer") @NullMarked
│   ├── CustomerService.java         facade public
│   ├── CustomerCreated.java         event record public
│   ├── CustomerDto.java
│   ├── web/CustomerController.java              (internal)
│   └── persistence/
│       ├── CustomerDocument.java                (internal, @Document(collection="customers"))
│       └── CustomerRepository.java              (internal)
└── order/ ...
```

**Profile B — "hexagonal module"** (chỉ cho domain lõi phức tạp):

```
com.vyforge.acmeshop.order
├── package-info.java                @ApplicationModule(allowedDependencies = {"customer :: api", "shared"})
├── api/                             @NamedInterface("api"): OrderApi, OrderPlaced (event @Externalized)
├── domain/                          aggregate, value object, domain event — KHÔNG import Spring/Mongo
├── application/
│   ├── port/in/                     @PrimaryPort  — PlaceOrderUseCase
│   ├── port/out/                    @SecondaryPort — OrderStore, PaymentGateway
│   └── service/                     triển khai use case
└── adapter/
    ├── web/                         @PrimaryAdapter — REST controller, API versioning
    ├── messaging/                   @PrimaryAdapter (Kafka listener) / @SecondaryAdapter (publisher)
    ├── persistence/                 @SecondaryAdapter — Spring Data MongoDB document + repository
    └── cache/                       @SecondaryAdapter — Redis
```

Các package con `adapter/in/web`, `adapter/out/persistence`, `application/port/in|out` trong BuckPal của Hombergs đến từ kiến thức nền, **(chưa kiểm chứng)** với bản 2 của sách. Cây Profile B ở trên đã gộp in/out vào tên adapter cho gọn. Không được tạo package `common` hay `util` làm chỗ đổ đồ chung. Chỉ dùng một module `shared` nhỏ, khai báo qua `sharedModules`.

### Bốn tier theo quy mô dự án

| Tier | Dự án điển hình | Cấu trúc bên trong module | Giao tiếp giữa module | Bắt buộc thêm |
|---|---|---|---|---|
| **0 – PoC / CRUD nhỏ** (1–2 dev, < 3 tháng) | Demo, landing có admin | Phẳng: controller, service, repository, document cùng một package, phần lớn package-private | Gọi đồng bộ qua facade | `ModularityTests.verify()` |
| **1 – Dự án khách hàng chuẩn** (2–6 dev) | CRM, HRM outsourcing | Profile A | `@ApplicationModuleListener` + Mongo event registry; Redis cache adapter tuỳ chọn | ArchUnit house rules, sinh tài liệu module trong CI |
| **2 – Product / domain phức tạp** (5+ dev, nhiều năm) | Sản phẩm của Vyforge | Profile B cho module lõi, Profile A cho module phụ | `@Externalized` → Kafka, consumer idempotent | jMolecules DDD + hexagonal rules, staleness monitor, `spring-modulith-junit` (chỉ test module có thay đổi) |
| **3 – Tách service** | Module cần scale, release riêng hoặc đổi team sở hữu | Giữ nguyên module | Event Kafka + interface `@HttpExchange` trở thành contract | Repo mới theo quy ước `<du-an>-<ten-service>-service` |

Các ngưỡng số dev ở trên là phán đoán, vì không có dữ liệu thực tế về các dự án của Vyforge. Điều quan trọng là **cấu trúc ngoài giống hệt nhau ở mọi tier**. Nhờ vậy dev chuyển dự án không phải học lại, và một dự án tier 1 lớn lên thành tier 2 chỉ cần refactor từng module, không phải đập đi làm lại. Lý do tách microservice (scale độc lập, nhịp release riêng, team riêng, cách ly lỗi) cũng lấy từ kiến thức nền. Bài "MonolithFirst" của Fowler không truy cập được để trích dẫn.

## Spring Boot 4.1 buộc template phải khoá nhiều quyết định kỹ thuật

### Nền tảng phiên bản và những thay đổi phá vỡ tương thích

Ngày 21/10/2026, bản GA mới nhất là **Spring Boot 4.1.1** (phát hành 20/8/2026, 98 bug fix) và 4.2.0-M2 đang ở milestone ([Spring blog 4.1.1](https://spring.io/blog/2026/08/20/spring-boot-4-1-1-available-now/)). BOM của 4.1.1 quản lý các phiên bản sau ([spring-boot-dependencies-4.1.1.pom](https://repo1.maven.org/maven2/org/springframework/boot/spring-boot-dependencies/4.1.1/spring-boot-dependencies-4.1.1.pom)):

| Thành phần | Phiên bản |
|---|---|
| Spring Framework | 7.0.9 |
| Spring Security | 7.1.1 |
| Spring Data | 2026.0.1 |
| Spring Kafka / kafka-clients | 4.1.1 / 4.2.1 |
| MongoDB driver | 5.8.1 |
| Lettuce | 7.5.2 |
| Jackson | 3.1.5 |
| JUnit Jupiter | 6.0.3 |
| Testcontainers | 2.0.5 |
| OpenTelemetry | 1.62.0 |

Theo migration guide của Boot 4.0 ([Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)), các thay đổi template phải phản ánh là:

- Jar được tách module: `spring-boot-starter-web` đổi thành `spring-boot-starter-webmvc`, và mỗi công nghệ có starter test riêng.
- Jackson 3 chuyển sang package `tools.jackson`.
- `@MockBean` bị bỏ, thay bằng `@MockitoBean`.
- `@SpringBootTest` không còn tự cấp MockMvc.
- Property Mongo đổi từ `spring.data.mongodb.*` sang `spring.mongodb.*`.
- Kafka `backoff.random` đổi thành `backoff.jitter`.

Framework 7 đưa vào JSpecify, API versioning gốc, `@ImportHttpServices`, `@Retryable`/`@ConcurrencyLimit`. `RestTemplate` sắp bị deprecate. Jackson 2 sẽ bị tắt auto-detect ở 7.1 và gỡ bỏ ở 7.2 ([Framework 7.0 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)). Boot 4.1 bổ sung chống SSRF cho HTTP client qua `InetAddressFilter`, hỗ trợ gRPC và cập nhật OTel ([Boot 4.1.0](https://spring.io/blog/2026/06/10/spring-boot-4/)). Hệ quả cho template:

- Gọi ra ngoài bằng `RestClient` + interface `@HttpExchange`.
- Dùng Jackson 3 ngay từ đầu.
- Mỗi `package-info.java` có `@NullMarked`.
- Bật `spring.threads.virtual.enabled=true` trên Java 25. Khẳng định rằng property này làm `server.tomcat.threads.max` mất tác dụng là **(chưa kiểm chứng)**.

Boot 4.0 hết hỗ trợ OSS vào 12/2026, 4.1 được hỗ trợ đến khoảng giữa năm 2027. Bản 4.2 GA được *dự đoán* rơi vào khoảng 11/2026 theo nhịp 6 tháng, nhưng chưa được xác nhận ([versionlog](https://versionlog.com/spring-boot/4.1/)).

### Event giữa module: Modulith là outbox có sẵn, nhưng Mongo phải chạy replica set

Event Publication Registry của Modulith ghi mỗi listener đích một bản ghi, nằm trong chính transaction nghiệp vụ. Bản ghi được đánh dấu hoàn tất khi listener chạy thành công và được giữ lại để retry nếu thất bại. Thực chất đây là một **transactional outbox có sẵn**. Backend MongoDB của registry (`spring-modulith-starter-mongodb`) **yêu cầu replica set** ([events.adoc](https://raw.githubusercontent.com/spring-projects/spring-modulith/main/src/docs/antora/modules/ROOT/pages/events.adoc)). Đẩy event ra Kafka chỉ cần thêm `spring-modulith-events-kafka` và gắn `@Externalized("topic::#{aggregateId}")`; module phát event không phải đổi code.

Bản 2.0 làm lại vòng đời của publication:

- Các trạng thái: PUBLISHED, PROCESSING, COMPLETED, FAILED, RESUBMITTED.
- Staleness monitor, mặc định tắt.
- Các API `FailedEventPublications` và `IncompleteEventPublications`.
- Trạng thái `ABANDONED` chỉ có từ 2.2 (đang milestone).

Outbox mode Namastack ra mắt ở 2.1 chỉ hỗ trợ CSDL quan hệ, nên không áp dụng cho Vyforge. JobRunr kết hợp Mongo là **(chưa kiểm chứng)** ([Modulith 2.1 GA](https://spring.io/blog/2026/06/11/spring-modulith-2-1-ga-2-0-7-and-1-4-12-released/)). Vì giao nhận là at-least-once, template phải bắt buộc ba điều:

- Mỗi event mang `eventId`.
- Consumer chống trùng bằng unique index trong collection `processed_messages`, hoặc Redis `SETNX` kèm TTL.
- Key Kafka là ID của aggregate, để giữ thứ tự theo từng aggregate.

Đây là thực hành chung chứ không phải hướng dẫn riêng của Modulith. Modulith chỉ lo độ tin cậy ở chiều gửi đi. Hệ quả vận hành: `compose.yaml` ở môi trường local và Testcontainers đều phải chạy **Mongo single-node replica set**. Ghi chú gợi ý Postgres trong compose của template backend là không đúng với stack này. Template phải dùng Mongo, Kafka, Redis và Keycloak.

### Migration MongoDB là lỗ hổng lớn nhất của stack

**Mongock không ra bản mới nào kể từ 5.5.1 (03/2025), và không có artifact driver nào cho Spring Data MongoDB 5** ([mongock-springboot-v3 metadata](https://repo1.maven.org/maven2/io/mongock/mongock-springboot-v3/maven-metadata.xml)). Liquibase và Flyway chỉ hỗ trợ MongoDB "rất hạn chế hoặc thử nghiệm" ([boottechsolutions](https://blog.boottechsolutions.com/2024/10/28/spring-data-easy-mongodb-migration-using-mongock/)). Ghi chú cũng không tìm được tuyên bố nào từ nhóm Mongock về Boot 4. Có ba phương án:

- **(A)** Mongock 5.5.1 với `mongodb-sync-v4-driver`. Phương án này dùng raw driver, không phụ thuộc Spring Data, và cần một spike để kiểm tra tương thích.
- **(B)** Liquibase MongoDB extension.
- **(C)** Một runner nội bộ khoảng vài trăm dòng: interface `ChangeUnit`, lịch sử lưu trong collection `_migrations`, khoá bằng một document lock, đóng gói thành `platform-mongo-migrations`.

Khuyến nghị: **chạy spike cho A trong giai đoạn 0. Nếu có bất kỳ lỗi nào thì chọn C**, vì C không phụ thuộc vào một dự án đã ngừng phát triển. Dù chọn phương án nào, quy ước sau là bắt buộc:

- Index được khai báo bằng code nhưng tắt auto-index-creation ở prod; migration chịu trách nhiệm tạo index.
- Đặt tên collection tường minh bằng `@Document(collection=...)`.
- Dùng `@Version` cho optimistic locking.

### Build, chất lượng code và test

**Maven 3.9 + wrapper + parent POM của công ty** là lựa chọn ít rủi ro nhất, và khớp với `ci-java-spring.yml` hiện có (`cache: maven`, `mvn -B verify`). Maven 4 vẫn ở 4.0.0-rc-7 và bản RC được ghi là "không phù hợp cho production" ([maven-core metadata](https://repo1.maven.org/maven2/org/apache/maven/maven-core/maven-metadata.xml); [Maven 4 rc-1 notes](https://maven.apache.org/docs/4.0.0-rc-1/release-notes.html)). Gradle 9.8.2 đã trưởng thành và được Boot 4 hỗ trợ đầy đủ ([Gradle current](https://services.gradle.org/versions/current); [Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)). Gradle chỉ thực sự có lợi khi build multi-module lớn hoặc cần cache build. Parent POM (`platform-parent`) làm các việc sau:

- Import BOM của Boot, `spring-modulith-bom` 2.1.x và `jmolecules-bom` 2025.0.x.
- Cấu hình `maven-enforcer`: `requireJavaVersion 25`, dependency convergence, banned dependencies.
- Đặt `project.build.outputTimestamp` để build có thể tái lập.
- Cấu hình Spotless, JaCoCo và CycloneDX.

Chất lượng code:

- **Spotless + palantir-java-format** là formatter duy nhất. Checkstyle tuỳ chọn.
- **Error Prone + NullAway ở chế độ JSpecify** trên các package `@NullMarked`. Ban đầu để mức warning, sau mới nâng lên error. Đây là cách chính Spring Framework 7 kiểm tra code của nó ([spring.io null-safety](https://spring.io/blog/2025/11/12/null-safe-applications-with-spring-boot-4/)). Các flag `--add-exports` cho JDK 25 **(chưa kiểm chứng)**.
- **Lombok** chỉ được dùng hạn chế: từ 1.18.40 mới hỗ trợ JDK 25, và bản đó đổi hành vi copy annotation Jackson ([Lombok #3859](https://github.com/projectlombok/lombok/issues/3859)). Quy tắc: DTO, value object và `@ConfigurationProperties` dùng `record`.

Test dùng các phiên bản do BOM quản lý: JUnit 6.0.3 (không tự nâng lên 6.1.x), AssertJ, Mockito với `@MockitoBean`, **Testcontainers 2** với `@ServiceConnection` cho Mongo, Kafka và Redis. Testcontainers 2 bỏ hỗ trợ JUnit 4 và đổi tên module thành `testcontainers-mongodb`; thông tin này từ nguồn thứ cấp ([issoh.co.jp](https://www.issoh.co.jp/tech/details/4629/)). Trạng thái module Redis chính thức trong TC 2 **(chưa kiểm chứng)**. Slice test dùng các starter test tách riêng của Boot 4 như `spring-boot-starter-data-mongodb-test` và `-kafka-test`. Test HTTP dùng `MockMvcTester` hoặc `RestTestClient`; chọn cái nào chủ yếu là sở thích ([Dan Vega](https://www.danvega.dev/blog/mock-vs-rest)). Kiến trúc được kiểm tra ở ba lớp:

1. `ApplicationModules.of(App.class).verify()` kèm `Documenter` để sinh tài liệu module.
2. Rule jMolecules cho các module Profile B. Tên class `JMoleculesArchitectureRules.ensureHexagonal()` **(chưa kiểm chứng)**.
3. ArchUnit 1.5.x cho house rules: không field injection, quy ước đặt tên.

Module test dùng `@ApplicationModuleTest` và `Scenario` API. Từ Modulith 2.1, `Scenario` thấy được event phát từ mọi thread ([testing.adoc](https://raw.githubusercontent.com/spring-projects/spring-modulith/main/src/docs/antora/modules/ROOT/pages/testing.adoc)). Có một cạm bẫy đã biết: gọi `.apiVersion()` trên MockMvc sẽ lỗi "No ApiVersionInserter" nếu thiếu cấu hình ([spring-boot#47373](https://github.com/spring-projects/spring-boot/issues/47373)). Template cần cấu hình sẵn `ApiVersionInserter`. PIT (mutation testing) chạy nightly cho domain tier 2. Contract test: dùng Spring Cloud Contract nếu cả hai đầu đều là Spring, dùng Pact nếu consumer là frontend hoặc mobile.

### Observability, bảo mật, API

**Observability.** Boot 4 có `spring-boot-starter-opentelemetry` chính chủ, export metrics và traces qua OTLP. Micrometer Observation vẫn là API instrument ([Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)). Template dùng starter này thay cho OTel Java agent. Ngoài ra:

- Log JSON có cấu trúc (`logging.structured.format.console=ecs`) ở mọi profile trừ local.
- Probe `/actuator/health/liveness|readiness`.
- Management port riêng, chỉ expose `health,info,prometheus`.

Các property cụ thể này lấy từ kiến thức nền, **(chưa kiểm chứng trên tài liệu Boot 4)**.

**Bảo mật.**

- Dùng `spring-boot-starter-security-oauth2-resource-server` với `issuer-uri` trỏ đến realm Keycloak.
- Kiểm tra `aud`.
- Dùng một `JwtAuthenticationConverter` chung để ánh xạ realm role và client role.
- Security 7 đã bỏ `authorizeRequests` và `and()` ([Security 7 what's new](https://docs.spring.io/spring-security/reference/whats-new.html), qua bản tóm tắt).
- Secret lấy từ env, `configtree` hoặc Vault. Không bao giờ để trong `application.yml`.
- Có SBOM CycloneDX (plugin 2.9.3) và quét lỗ hổng.

**API.**

- springdoc-openapi **3.1.1** cho Boot 4 ([metadata](https://repo1.maven.org/maven2/org/springdoc/springdoc-openapi-starter-webmvc-ui/maven-metadata.xml)).
- Lỗi trả về theo **RFC 9457 Problem Details**, với `type` URI ổn định.
- Versioning dùng cơ chế gốc `spring.mvc.apiversion.*` ([API Versioning in Spring](https://spring.io/blog/2025/09/16/api-versioning-in-spring/)).
- Quy ước đặt tên: path dạng kebab-case với danh từ số nhiều, JSON camelCase, thời gian ISO-8601 UTC.

Khuyến nghị **versioning bằng path `/api/v{n}`** vì dễ đi qua gateway và WAF của khách hàng, và dễ đọc trong log. Đây là quyết định chọn một lần cho toàn công ty. Dự án code-first là mặc định; API công khai hoặc dùng chéo team thì làm API-first với openapi-generator 7.26.0. OpenAPI spec do springdoc sinh ra là đầu vào để sinh type cho frontend.

## Frontend: App Router mỏng, feature module, và Next.js làm BFF

### Cấu trúc thư mục: feature-based mặc định, FSD chỉ cho product lớn

Next.js cố ý không áp đặt cấu trúc. Framework chỉ cung cấp `src/`, route group `(group)`, private folder `_folder` và cho phép colocation an toàn: một segment chỉ thành route public khi có `page` hoặc `route` ([Next.js Project structure](https://nextjs.org/docs/app/getting-started/project-structure)). Bulletproof-react chia code theo feature và quy định hướng import một chiều shared → features → app. Các feature không import lẫn nhau, và quy tắc này được ép bằng ESLint `import/no-restricted-paths` ([DEV](https://dev.to/meijin/bulletproof-react-is-a-hidden-treasure-of-react-best-practices-3m19)). Feature-Sliced Design xung đột với App Router: layer `pages` trùng tên, và cấu trúc thư mục bị ràng buộc theo URL. Cách xử lý phổ biến là đổi `pages` thành `views` hoặc đặt `app/` ở thư mục gốc ([HackerNoon](https://sia.hackernoon.com/how-to-deal-with-the-nextjs-app-router-and-fsd-problem)). Guide FSD chính thức cho Next.js không tải được. Với một công ty làm nhiều app ngắn hạn, sáu layer của FSD tốn chi phí onboarding mà chỉ đáng bỏ ra khi app đủ lớn. Template mặc định vì vậy dùng cây sau:

```
crm-frontend/
├─ next.config.ts                 cacheComponents, output: 'standalone', reactCompiler (sau khi lint sạch)
├─ src/
│  ├─ app/                        CHỈ ROUTING: page/layout/loading/error/route.ts
│  │  ├─ [locale]/
│  │  │  ├─ (auth)/login/page.tsx
│  │  │  └─ (dashboard)/orders/{page.tsx,[id]/page.tsx}   → import từ @/features/orders
│  │  └─ api/{auth/[...all],health}/route.ts
│  ├─ features/orders/
│  │  ├─ components/              'use client' ở leaf
│  │  ├─ api/{queries.ts (server-only), actions.ts ('use server'), hooks.ts (TanStack)}
│  │  ├─ schemas.ts               Zod, dùng chung client/server
│  │  └─ index.ts                 public API của feature
│  ├─ shared/{ui (shadcn), api (client sinh từ OpenAPI), auth (DAL, server-only), lib (env.ts Zod)}
│  ├─ i18n/{routing.ts, request.ts}   + messages/{vi,en}.json
│  └─ proxy.ts
```

Dự án tier 2 có nhiều frontend (admin + portal) thì dùng biến thể FSD (`app/` ở gốc, `src/{app,views,widgets,features,entities,shared}`) trong một monorepo Turborepo + pnpm. Ranh giới được kiểm tra bằng `eslint-plugin-boundaries` hoặc Steiger.

### Next.js 16.4: bật Cache Components, dùng `proxy.ts`, và vá ngay

Next.js 16 có các thay đổi sau ([Next.js 16 blog](https://nextjs.org/blog/next-16)):

- Turbopack là bundler mặc định.
- Cache Components (`"use cache"` + PPR) được bật qua `cacheComponents`.
- `middleware.ts` đổi tên thành `proxy.ts`, chạy trên Node runtime.
- React Compiler đã ổn định nhưng vẫn là opt-in.
- `next lint` bị bỏ.

Bản 16.4 (6/10/2026) đưa Cache Components thành mặc định của create-next-app và đi kèm React 19.3. Thông tin này từ nguồn thứ cấp, vì không tìm thấy blog chính thức ([zeli.app](https://zeli.app/story/50028855)). Next 15 hết hỗ trợ đúng ngày 21/10/2026; 16.x dự kiến được hỗ trợ đến 21/10/2027 ([HeroDevs](https://www.herodevs.com/blog-posts/nextjs-eol-dates-version-support-timeline)).

**Bảo mật là lý do cấp bách nhất.** Bản 16.4.1 (14/10/2026) vá hai lỗ hổng RCE CVSS 9.5 và một lỗ hổng DoS. RCE thứ nhất nằm ở `use cache` khi hàm trả về nhiều typed array, và không có workaround. RCE thứ hai nằm ở sharp khi xử lý SVG trong Image Optimization. Các phiên bản bị ảnh hưởng là 16.0.0–16.4.0 ([Netlify](https://netlify.com/changelog/2026-10-14-nextjs-react-security-vulnerabilities); [CSA](https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/10/CSA_research_note_nextjs_use_cache_librsvg_rce_20261016-csa-styled.pdf)). Trước đó đã có React2Shell (CVE-2025-55182, CVSS 10, đã bị khai thác thực tế) ([Wiz](https://www.wiz.io/pt-br/blog/critical-vulnerability-in-react-cve-2025-55182)) và CVE-2025-29927, lỗi bypass auth qua middleware ([Help Net Security](https://www.helpnetsecurity.com/2025/03/24/critical-next-js-auth-bypass-vulnerability-opens-web-apps-to-compromise-cve-2025-29927/)). Như vậy là ba lỗ hổng nghiêm trọng liên quan RSC/Server Function trong khoảng 11 tháng. Vyforge tự host trên Docker nên không có WAF của Vercel che chắn. Template vì vậy phải:

- Đặt phiên bản tối thiểu **≥ 16.4.1** (lấy bản patch mới nhất) và **sharp ≥ 0.35.5**.
- Có rule Renovate fast-track cho `next`, `react`, `react-dom`, `sharp`.
- Có quy trình rebuild image trong ngày khi có advisory.

Theo ghi chú, `typedRoutes` đã ổn định ở 16, và async request API (`await cookies()`, `await params`) là bắt buộc. Cả hai điểm này **(chưa kiểm chứng)**, nhưng code mẫu trong template vẫn nên chỉ dùng dạng async.

### Lấy dữ liệu từ Spring: Server Component đọc, Server Action ghi, TanStack chỉ khi cần tương tác

Next.js khuyến cáo chỉ truy vấn dữ liệu trực tiếp trong component khi làm prototype, còn lại phải qua một **Data Access Layer** ([Next.js security blog](https://nextjs.org/blog/security-nextjs-server-components-actions)). Cách lấy dữ liệu mặc định của template:

- **Server Component** gọi Spring qua `shared/api/client.server.ts`. File này có `import 'server-only'` và gắn access token từ session ở phía server.
- **Server Action** xử lý mutation. Mỗi action parse bằng Zod, gọi Spring, rồi chạy `revalidateTag`.
- **TanStack Query v5** chỉ dùng cho dữ liệu cần tương tác: infinite list, polling, optimistic UI. Server prefetch dữ liệu, client nhận qua `HydrationBoundary` ([TanStack Advanced SSR](https://tanstack.com/query/v5/docs/framework/react/guides/advanced-ssr)). TanStack Query có tỉ lệ dùng 68% trong State of React 2025 ([heise](https://heise.de/-11197819)).

Type được sinh từ spec springdoc bằng **openapi-typescript + openapi-fetch**. Runtime khoảng 6 KB và sinh nhanh nhất trong benchmark ([pkgpulse](https://www.pkgpulse.com/guides/orval-vs-openapi-typescript-vs-kubb-openapi-client-2026)). Orval là lựa chọn thay thế khi muốn sinh sẵn hook TanStack, Zod schema và mock MSW. openapi-typescript-codegen đã ngừng phát triển. Route Handler chỉ dùng cho webhook, proxy file và bên thứ ba gọi vào. Hai cạnh sắc cần ghi vào template:

- `"use cache"` trên dữ liệu theo user hoặc tenant phải có key chứa user/tenant, nếu không thì không được cache.
- Một helper `toFormErrors(problemDetail)` dùng chung để ánh xạ lỗi field từ Problem Details của Spring vào form.

### State, form, auth, UI và i18n

**State.** Thứ tự ưu tiên: URL state bằng **nuqs** cho filter và phân trang → `useState`/context → **Zustand** khi cần chia sẻ state giữa nhiều nhánh cây component. Store Zustand phải tạo theo request hoặc qua provider để tránh rò state giữa các request. Template không cài sẵn store global nào, vì 34% dev React không dùng thư viện state ([certificates.dev](https://certificates.dev/blog/breaking-down-state-of-react-2025-results)).

**Form.** Mặc định là **React Hook Form + Zod**. RHF có tỉ lệ dùng 74%, TanStack Form 21% (cùng nguồn trên). Form đơn giản cần progressive enhancement thì dùng **Server Action + `useActionState` + Conform**, vì hỗ trợ Server Action của RHF bị mô tả là đang dừng phát triển ([Zenn](https://zenn.dev/akfm/articles/server-actions-with-conform)). Schema Zod đặt trong `features/*/schemas.ts` và dùng chung cho client lẫn server.

**Auth.**

- Next.js là confidential OIDC client với Keycloak. Session nằm trong cookie HttpOnly, SameSite=Lax. Access token chỉ đi giữa Next và Spring; trình duyệt không bao giờ giữ token.
- Thư viện: **Better Auth** cho dự án mới, vì Auth.js đã chuyển sang team Better Auth từ 22/9/2025 và chỉ còn nhận bản vá bảo mật ([Better Auth](https://www.better-auth.com/blog/authjs-joins-better-auth)). Chi tiết plugin generic OIDC kết nối Keycloak **(chưa kiểm chứng)**, nên phải spike ở giai đoạn 0. Auth.js v5 vẫn đang beta và đã nhận 4 advisory vào 7/2026 ([Better Auth security update](https://better-auth.com/blog/security-update-july-2026)). Dự án cũ đang dùng Auth.js thì giữ nguyên và vá đầy đủ.
- **Không bao giờ chỉ dựa vào `proxy.ts` để phân quyền.** Server Action là endpoint POST công khai, nên mỗi action phải tự kiểm tra authN và authZ. DAL phải là server-only và chỉ trả DTO tối thiểu ([Next.js Data Security](https://www.nextjs.org/docs/app/guides/data-security); [OWASP cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Nextjs_Security_Cheat_Sheet.html)).
- Không truyền token qua props cho client component. Cobalt từng gặp token Keycloak bị serialize vào hydration payload dù cookie đã được bảo vệ ([Cobalt](https://www.cobalt.io/blog/the-hidden-cost-of-client-side-convenience)).
- Phương án thay thế là để Spring Cloud Gateway làm BFF (`oauth2Login` + TokenRelay). Cách này hợp với dự án mà team backend muốn nắm toàn bộ bảo mật, đổi lại mất sự tiện lợi khi RSC gọi API trực tiếp.

**UI.** **Tailwind CSS v4 + shadcn/ui CLI v4**. Token đặt trong `@theme` dưới dạng CSS variable để mỗi khách hàng override được ([ssojet](https://ssojet.com/blog/what-is-shadcn-ui)). Thông tin "Base UI là mặc định từ 7/2026" đến từ blog của một đối thủ cạnh tranh, **(chưa kiểm chứng)** ([Aceternity](https://ui.aceternity.com/blog/shadcn-ui-alternatives)). AI Elements lại phụ thuộc vào Radix. Template chọn **Radix** cho đến khi xác minh được.

**i18n.** **next-intl v4** với segment `[locale]`, gắn vào `proxy.ts` ([buildwithmatija](https://www.buildwithmatija.com/blog/next-intl-nextjs-16-proxy-fix)):

- `locales: ['vi','en']`, `defaultLocale: 'vi'`.
- Font dùng `subsets: ['latin','vietnamese']`.
- Định dạng tiền bằng `Intl` với `currency: 'VND'` và 0 chữ số thập phân.

Tương thích giữa next-intl và Cache Components **(chưa kiểm chứng)**.

### Tooling và test frontend

- **pnpm 11** là package manager. Bản này cần Node ≥ 22 và mặc định giữ lại các version mới publish trong 24 giờ ([bytes.dev](https://bytes.dev/archives/484)). Ghim phiên bản qua `packageManager` và cài bằng `pnpm/action-setup`, **không dùng corepack**, vì Node 25+ không còn đóng gói nó ([Node docs](https://r2.nodejs.org/dist/v25.9.0/docs/api/corepack.json)). Thông tin pnpm 12 (viết lại bằng Rust) đã ổn định chỉ có một nguồn, **(chưa kiểm chứng)**, nên vẫn ghim 11.x. File `ci-node.yml` hiện tại còn dùng `cache: npm` và `npm ci`, cần đổi.
- **TypeScript.** TS 7.0, compiler native viết bằng Go, GA ngày 8/7/2026. Build nhanh hơn khoảng 10 lần theo số liệu của nhà phát hành, nhưng chưa có compiler API cho đến bản 7.1 ([VSM](https://visualstudiomagazine.com/articles/2026/07/08/typescript-7-arrives-to-rock-vs-code-with-go-powered-speed.aspx); [itwriting](https://www.itwriting.com/blog/?p=12438)). Vì vậy dùng TS 7 cho `tsc --noEmit`. Nếu rule type-aware của typescript-eslint xung đột thì giữ `typescript@~6.0` làm API provider cho ESLint; tương thích này **(chưa kiểm chứng)**.
- **Lint.** ESLint 9 flat config + `eslint-config-next/core-web-vitals` và `/typescript`, vì đến 4/2026 `eslint-config-next` chưa hỗ trợ ESLint 10 ([InfoQ](https://infoq.com/news/2026/04/eslint-10-release)). Format bằng Prettier 3 + `prettier-plugin-tailwindcss`. Oxlint là bước lint nhanh tuỳ chọn ([oxc.rs](https://oxc.rs/blog/2025-06-10-oxlint-stable)). Biome không làm mặc định vì sẽ mất các rule `@next/next/*`.
- **tsconfig.** `strict`, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, `moduleResolution: "bundler"`.

Test:

- **Vitest 4** + React Testing Library cho unit và component ([Vitest 4](https://voidzero.dev/posts/announcing-vitest-4)).
- **Playwright** cho E2E, chạy trên output standalone. Next.js nói rõ Vitest chưa hỗ trợ async Server Component ([Next.js Vitest guide](https://nextjs.org/docs/app/guides/testing/vitest)), nên logic lấy dữ liệu viết thành hàm thuần để unit test, còn RSC kiểm bằng E2E với MSW hoặc WireMock.
- **MSW 2** cho mock mạng.
- **Storybook 10** tuỳ chọn cho dự án có design system ([Storybook 10](https://storybook.js.org/blog/storybook-10/)).
- Không dùng Jest cho template mới.

## Delivery: chia sẻ qua artifact có version, chỉ copy phần khung

### Container và chuỗi cung ứng phần mềm

**Backend.** Mặc định là Dockerfile multi-stage: `java -Djarmode=tools -jar app.jar extract --layers`, image runtime JRE 25 chạy non-root. Boot 4.1 đã **gỡ bỏ jar mode `layertools`** ([Boot 4.1 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.1-Release-Notes)). Trên Java 25, **AOT Cache (JEP 483)** thay thế CDS: chạy train với `-XX:AOTCacheOutput`, chạy thật với `-XX:AOTCache` ([Spring Framework AOT cache](https://docs.spring.io/spring-framework/reference/integration/aot-cache.html)). Buildpacks có `BP_JVM_AOTCACHE_ENABLED=true` cho cùng mục đích ([Boot how-to](https://docs.spring.io/spring-boot/how-to/aot-cache.html), qua bản tóm tắt). Ghi chú có hai khuyến nghị mâu thuẫn: một nguồn chọn buildpacks làm mặc định, nguồn kia chọn Dockerfile. Báo cáo chọn **Dockerfile**, vì một so sánh cho thấy image buildpack lớn gấp đôi ([BellSoft](https://bell-sw.com/blog/how-to-dockerize-a-spring-boot-app-with-the-smallest-base-image)), và Dockerfile đi cùng một pattern với frontend. `spring-boot:build-image` vẫn là phương án thay thế được chấp nhận. Mức tăng tốc khởi động thực tế của AOT cache trên Boot 4 chưa có số đo. GraalVM native chỉ dùng khi dự án tự chọn, cho trường hợp scale-to-zero.

**Frontend.** Dùng `output: 'standalone'`, image runtime Node 24 chạy non-root. Có thể dùng Docker Hardened Images, như ví dụ chính thức của Docker ([Docker Next.js guide](https://docs.docker.com/guides/nextjs/)). Cài pnpm bằng `npm i -g pnpm@<pin>`. **`NEXT_PUBLIC_*` bị inline lúc build**, nên cấu hình runtime phải đến từ env phía server. Đây là điều kiện để dùng một image cho nhiều môi trường.

**Chuỗi cung ứng.** Mỗi image được push lên GHCR theo digest, đi kèm SBOM và provenance (`attest-build-provenance` hoặc cosign keyless), và được quét bằng Trivy. Ngày 19/3/2026, TeamPCP đã chiếm quyền `trivy-action` và force-push 75/76 tag để đánh cắp secret CI ([Snyk](https://snyk.io/articles/trivy-github-actions-supply-chain-compromise/)). Vì vậy **mọi action phải ghim theo commit SHA**. Các workflow template hiện có trong `vyforge/.github` đang ghim theo tag (`actions/checkout@v7`, `amannn/action-semantic-pull-request@v6`), cần sửa.

### Kubernetes và GitOps

- **Helm 4** (GA 11/2025) dùng cho một app chart chung của công ty. Helm 3 hết nhận bản vá bảo mật từ 11/11/2026 ([Helm blog](https://helm.sh/blog/helm-4-released)). Chart publish dạng OCI lên `ghcr.io/vyforge/charts/app` và gồm Deployment, Service, Ingress/HTTPRoute, HPA, PDB, ServiceMonitor, ExternalSecret, NetworkPolicy.
- **Argo CD 3.x** đọc từ repo `<du-an>-infra`, đúng quy ước đặt tên repo hiện có cho Terraform/Helm. CI chỉ bump digest ở `values-dev.yaml`. Lên staging và prod bằng PR, cùng một digest, không build lại. Kargo đáng cân nhắc ở giai đoạn sau: tỉ lệ dùng tăng từ 2% lên 20% trong khảo sát Argo CD năm 2026 ([tfir.io](https://tfir.io/argocd-adoption-kargo-multi-environment-promotion/)).
- **Secrets** qua External Secrets Operator với API `external-secrets.io/v1`, vì v1beta1 đã bị bỏ ([Sylva](https://gitlab.com/sylva-projects/sylva-core/-/issues/2506)).
- **Probe Spring:** liveness, readiness, cộng startupProbe cho thời gian JVM khởi động, và `server.shutdown=graceful`.
- **Probe Next:** `/api/health`.

### CI: chuyển workflow template thành reusable workflow

Hiện tại `vyforge/.github` chỉ có **workflow template**: file được copy một lần khi chọn "New workflow". Sau khi copy, sửa template không có tác dụng gì với repo đã tạo. Reusable workflow (`on: workflow_call`) giải quyết được việc này. Từ 11/2025, GitHub cho phép lồng 10 cấp và 50 reusable workflow trong một lần chạy, và permission chỉ có thể giữ nguyên hoặc giảm khi đi xuống chuỗi gọi ([GitHub docs](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations); [changelog 11/2025](https://github.blog/changelog/2025-11-06-new-releases-for-github-actions-november-2025)). Cấu trúc đề xuất:

- `vyforge/.github/.github/workflows/` chứa:
  - `java-ci.yml`: Maven verify, Spotless, Testcontainers, JaCoCo, ModularityTests.
  - `node-ci.yml`: pnpm frozen-lockfile, lint, typecheck, Vitest, Playwright.
  - `container.yml`: build, push, SBOM, attest, Trivy.
  - `gitops-bump.yml`.
- Các workflow này được phát hành theo tag `v1`.
- `workflow-templates/` giữ vai trò cũ: tạo caller workflow mỏng cho repo mới, dạng `uses: vyforge/.github/.github/workflows/java-ci.yml@v1`.
- Các check `ci / build`, `ci / test`, `container / scan` được thêm vào organization ruleset sẵn có.
- Mặc định `permissions: contents: read`. Dùng OIDC khi cần quyền cloud.

### Repo và phân phối template: Copier cho khung, artifact cho phần dùng chung

**Polyrepo** `<du-an>-backend` / `<du-an>-frontend` / `<du-an>-infra` khớp với quy ước hiện có, dễ phân quyền, và dễ bàn giao cho khách hàng. Chỉ dùng monorepo khi cùng một team sở hữu cả frontend lẫn backend và hai bên release cùng nhau. Quy tắc phân phối: **đưa càng ít càng tốt vào phần copy, càng nhiều càng tốt vào artifact có version** để Renovate tự cập nhật. Các artifact đó là:

- reusable workflow `@v1`
- Renovate preset
- Helm chart
- `platform-parent` (Maven)
- `@vyforge/eslint-config` và `@vyforge/tsconfig` trên GitHub Packages
- `platform-mongo-migrations`, nếu chọn phương án C

Repo `template-spring-boot` và `template-nextjs` chỉ chứa phần khung: Dockerfile, `compose.yaml`, lefthook, `.editorconfig`, devcontainer và caller workflow. **Copier** đọc Git tag của template, rồi chạy `copier update` để áp smart diff lên repo đã sinh. Xung đột hiện ra dưới dạng file `.rej` hoặc marker, và smart diff đôi khi cho kết quả khó đoán ([Copier docs](https://copier.readthedocs.io/en/v9.7.0/updating/)). Cruft chỉ theo dõi commit, không theo dõi thay đổi biến, và Cambridge UIS đã đánh giá là không hữu ích ([Cambridge UIS](https://gitlab.developers.cam.ac.uk/uis/devops/webapp-boilerplate/-/issues/89)). GitHub template repo, cách hiện đang ghi trong checklist của `REPOSITORY_GUIDELINES.md`, thì **không có đường cập nhật nào sau lần tạo**. Có một điểm kỹ thuật cần lưu ý: template Copier chứa cú pháp Jinja, nên nếu tạo repo bằng nút "Use this template" của GitHub sẽ ra file chưa render. Do đó repo `template-*` vẫn có thể bật cờ Template repository để dễ tìm, nhưng **quy trình chính thức là `copier copy`**. Việc đồng bộ chạy bằng một workflow định kỳ mở PR `copier update`. Renovate có manager cho Copier hay không thì **(chưa kiểm chứng)**. Backstage chỉ đáng đầu tư khi cần cổng self-service, và nó chỉ tạo repo, không đồng bộ về sau ([Backstage](https://backstage.io/docs/features/software-templates/builtin-actions)).

**Git hook:** dùng **Lefthook** chung cho cả Java và Node, chạy format và lint trên staged file, cùng commitlint. CI vẫn kiểm tra lại toàn bộ vì hook có thể bị bỏ qua bằng `--no-verify` ([pkgpulse](https://www.pkgpulse.com/guides/husky-vs-lefthook-vs-lint-staged-git-hooks-nodejs-2026)). **Dependency bot:** Renovate có preset dùng chung cho toàn org, grouping linh hoạt và `minimumReleaseAge`. Self-hosted Renovate phải ≥ 44.3.1 để vá CVE-2026-88884 ([Mend](https://mend.io/vulnerability-database/CVE-2026-88884/)). Thời gian chờ của bot phải khớp với cổng chờ của pnpm, nếu không bot sẽ mở PR mà không cài được dependency ([nesbitt.io](https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down)).

## Quy tắc: BẮT BUỘC và KHUYẾN NGHỊ

### BẮT BUỘC (mọi dự án mới, mọi tier)

| # | Phạm vi | Quy tắc |
|---|---|---|
| B1 | Backend | Java 25, Spring Boot 4.1.x qua `platform-parent`/BOM; không hard-code version trong module; wrapper (`mvnw`) được commit |
| B2 | Backend | Package gốc `com.vyforge.<du-an>`; mỗi package con trực tiếp là một module nghiệp vụ; không có `common`/`util` ở top-level, chỉ `shared` khai báo qua `sharedModules` |
| B3 | Backend | Test `ApplicationModules.verify()` + `Documenter` chạy trong CI; không dùng `Type.OPEN` trừ khi di chuyển code legacy, và phải có ADR |
| B4 | Backend | Module khác chỉ được dùng package gốc của module hoặc `@NamedInterface`; event dùng chéo module nằm trong API của module |
| B5 | Backend | Jackson 3; `RestClient` + `@HttpExchange` (không dùng `RestTemplate`); `@NullMarked` trong mọi `package-info.java`; record cho DTO và `@ConfigurationProperties` |
| B6 | Backend | Mongo chạy replica set ở mọi môi trường, kể cả local và Testcontainers; `@Document(collection=...)` tường minh; index tạo bằng migration, tắt auto-index ở prod |
| B7 | Backend | Event có `eventId`; Kafka key = ID aggregate; consumer idempotent; DLT qua `DefaultErrorHandler`/`@RetryableTopic` |
| B8 | Backend | JUnit (theo BOM) + AssertJ + `@MockitoBean` + Testcontainers 2 `@ServiceConnection`; JaCoCo có ngưỡng trên code thay đổi; Spotless (palantir-java-format) `check` trong CI |
| B9 | Backend | Resource server JWT Keycloak (`issuer-uri`, kiểm tra `aud`); Problem Details RFC 9457; springdoc 3.x; versioning theo một chuẩn chung của công ty |
| B10 | Backend | Actuator liveness/readiness, starter OpenTelemetry, log JSON có cấu trúc ngoài môi trường local |
| B11 | Frontend | Next ≥ 16.4.1 (bản patch mới nhất), React 19, TS strict, Node 24 LTS, pnpm 11 ghim qua `packageManager`, chỉ một lockfile, `--frozen-lockfile` |
| B12 | Frontend | `app/` chỉ chứa routing; code nằm trong `features/*` (có `index.ts` public) và `shared/`; ranh giới import kiểm tra bằng ESLint |
| B13 | Frontend | Gọi Spring qua client `server-only` sinh từ OpenAPI; token không bao giờ xuống trình duyệt hay vào props; mỗi Server Action tự kiểm tra authN/authZ; DAL là server-only; không dựa vào `proxy.ts` để phân quyền |
| B14 | Frontend | `proxy.ts` (không dùng `middleware.ts`); async request API; `output: 'standalone'`; không có cấu hình môi trường trong `NEXT_PUBLIC_*` |
| B15 | Frontend | ESLint flat config `eslint-config-next`; Prettier; Vitest cho unit; Playwright smoke E2E |
| B16 | Delivery | Image non-root, multi-stage, có SBOM + provenance, quét Trivy (ghim SHA); dùng một digest cho mọi môi trường |
| B17 | Delivery | CI gọi reusable workflow `vyforge/.github/...@v1`; mọi third-party action ghim theo SHA; `permissions: contents: read` mặc định |
| B18 | Delivery | Secret chỉ đi qua ESO / GitHub Environments; không có trong Git; bật secret scanning |
| B19 | Delivery | Dependency bot theo preset của org, có fast-track bảo mật cho `next`/`react`/`sharp`/Spring Boot |
| B20 | Repo | Tạo repo bằng Copier từ `template-*`; giữ `.copier-answers.yml`; tên repo theo `<du-an>-<thanh-phan>` |

### KHUYẾN NGHỊ

| # | Phạm vi | Quy tắc |
|---|---|---|
| K1 | Backend | Profile B (hexagonal + jMolecules rules) cho module lõi ở tier 2; Profile A cho phần còn lại |
| K2 | Backend | `@Externalized` → Kafka thay vì publisher viết tay; bật staleness monitor; `spring-modulith-junit` để chỉ test module có thay đổi (tier 2) |
| K3 | Backend | Error Prone + NullAway (warning trước, error sau); ArchUnit house rules; PIT nightly cho domain |
| K4 | Backend | `spring.threads.virtual.enabled=true`; `@Retryable` của Framework cho các trường hợp retry đơn giản; AOT cache trong Dockerfile |
| K5 | Backend | Lombok chỉ dùng cho `@Slf4j`/`@RequiredArgsConstructor`; Spring Cloud Contract hoặc Pact cho API dùng chéo team |
| K6 | Frontend | TanStack Query v5 cho dữ liệu cần tương tác; nuqs cho filter; Zustand chỉ khi thật cần |
| K7 | Frontend | RHF + Zod (form phức tạp), Server Action + `useActionState` + Conform (form đơn giản); helper ánh xạ Problem Details |
| K8 | Frontend | Better Auth (dự án mới) + Keycloak; Tailwind v4 + shadcn/ui (Radix); next-intl `vi` mặc định |
| K9 | Frontend | `cacheComponents: true`; bật React Compiler sau khi lint sạch; oxlint pre-pass; TS 7 cho typecheck; MSW; Storybook cho dự án có design system |
| K10 | Frontend | Turborepo + pnpm workspace khi một dự án có ≥ 2 frontend; biến thể FSD cho product lớn |
| K11 | Delivery | Helm app chart chung (OCI) + Argo CD + repo `<du-an>-infra`; cân nhắc Kargo cho promotion |
| K12 | DX | Lefthook + commitlint; devcontainer (Java 25 + Node 24); `compose.yaml` có Mongo replica set, Kafka, Redis, Keycloak |

## Sáu quyết định công ty phải chốt

| # | Quyết định | Khuyến nghị | Căn cứ và rủi ro |
|---|---|---|---|
| Q1 | **Maven hay Gradle** | **Maven 3.9 + `platform-parent`**; xem lại sau khi Maven 4 GA thêm một bản patch (khoảng 2027) | Khớp với CI hiện có và với số đông dev. Nếu một nhóm đã quen Gradle thì cho phép Gradle 9 Kotlin DSL + version catalog như một ngoại lệ có ADR, không phải lựa chọn ngang hàng |
| Q2 | **Migration MongoDB** | Spike Mongock 5.5.1 (raw driver) trong 1 tuần; nếu có lỗi hoặc rủi ro thì chọn **runner nội bộ `platform-mongo-migrations`** | Mongock không ra bản mới từ 03/2025 và không có tín hiệu hỗ trợ Boot 4; Liquibase/Flyway với Mongo chỉ ở mức thử nghiệm |
| Q3 | **Hexagonal mặc định hay chỉ tier 2** | **Chỉ cho module lõi ở tier 2.** Mặc định là Profile A | Drotbohm và Brown đều ưu tiên cắt theo domain; hexagonal tăng gấp đôi số type, không đáng với CRUD outsourcing. Hombergs cũng chấp nhận shortcut |
| Q4 | **Renovate hay Dependabot** | **Renovate** với preset org; sửa mục 5 của `REPOSITORY_GUIDELINES.md` (hiện ghi Dependabot) | Grouping, preset chung, `minimumReleaseAge`. Nếu tự host thì phải ≥ 44.3.1 |
| Q5 | **Versioning API** | Path `/api/v{n}` bằng cơ chế gốc của Framework 7 | Dễ đi qua gateway khách hàng; cấu hình `ApiVersionInserter` cho test |
| Q6 | **BFF cho auth** | Next.js + Better Auth làm mặc định; Spring Cloud Gateway BFF là phương án B | Better Auth + Keycloak chưa được kiểm chứng; nếu spike thất bại thì dùng Auth.js v5 (đã vá đầy đủ) tạm thời |

Ngoài ra cần chốt thêm: palantir hay google-java-format (báo cáo đề xuất palantir), nơi chạy Keycloak cho từng khách hàng, và có dùng TS 7 trong CI ngay hay không, vì phụ thuộc vào tương thích của typescript-eslint.

## Lộ trình 4 giai đoạn

| Giai đoạn | Thời gian | Kết quả | Tiêu chí hoàn thành |
|---|---|---|---|
| **0 – Spike và quyết định** | Tuần 1–2 (đến ~6/11/2026) | Chốt Q1–Q6 bằng ADR. Ba spike: Mongock vs runner nội bộ, Better Auth + Keycloak + Spring resource server, Modulith Mongo registry + Kafka externalization trên Testcontainers replica set | Mỗi spike có repo demo chạy được; ADR được merge vào `vyforge/.github/docs/adr/` |
| **1 – Nền tảng dùng chung** | Tuần 3–5 | Reusable workflow `java-ci`, `node-ci`, `container`, `gitops-bump` phát hành `v1` (ghim SHA); chuyển `workflow-templates/` thành caller mỏng (pnpm); Renovate preset; `platform-parent`; `@vyforge/eslint-config`, `@vyforge/tsconfig`; Helm app chart OCI | Một repo thử nghiệm chạy CI xanh hoàn toàn bằng artifact dùng chung |
| **2 – Template Copier** | Tuần 5–8 | `template-spring-boot` (Profile A, câu hỏi chọn tier, tuỳ chọn Kafka/Redis, Profile B mẫu), `template-nextjs` (feature-based, BFF, i18n vi/en), `template-infra` (values và Argo CD app); workflow định kỳ `copier update`; cập nhật checklist mục 6 của `REPOSITORY_GUIDELINES.md` | Sinh một cặp `demo-backend`/`demo-frontend` từ đầu đến khi deploy lên dev trong dưới 1 ngày |
| **3 – Pilot và mở rộng** | Tuần 9–14 | Một dự án outsourcing mới (tier 1) và một product (tier 2) chạy pilot; thu thập phản hồi; phát hành template v1.1; lên kế hoạch nâng cấp dự án cũ (Boot 3.5 đã hết OSS, Next 15 hết hỗ trợ) | Hai dự án pilot ra production; sửa template qua `copier update` thành công trên cả hai |
| **Liên tục** | Mỗi quý | Rà soát bảng chuẩn công nghệ; nâng template khi Boot 4.2 / Modulith 2.2 / Node 26 LTS GA; đánh giá lại Maven 4, ESLint 10, pnpm 12 | Changelog template; Renovate PR được merge trong SLA |

## Kết luận

Vấn đề cốt lõi của Vyforge là làm sao để hàng chục repo đã được sinh ra **vẫn tiếp tục nhận được cải tiến** sau ngày tạo. Chọn kiến trúc nào là câu hỏi thứ yếu hơn. Vì vậy thiết kế quan trọng nhất trong báo cáo này là tách ra hai lớp. Lớp thứ nhất gồm các artifact có version (reusable workflow, parent POM, chart, config package), được cập nhật bằng Renovate. Lớp thứ hai là phần khung mỏng sinh bằng Copier, đồng bộ bằng `copier update`. Nếu không có cách tách này, mọi quy tắc BẮT BUỘC ở trên sẽ chỉ đúng vào ngày repo được tạo. Thực tế hiện nay cho thấy rõ điều đó: workflow template đang dùng npm và tag có thể bị đổi, trong khi chuẩn mới đòi hỏi pnpm và ghim SHA.

Điểm thứ hai: Spring Modulith cho phép Vyforge **hoãn quyết định microservice mà không phải trả giá**. Một dự án outsourcing tier 1 và một product tier 2 có cùng hình dạng bên ngoài. Module nào thực sự cần tách thì đã có sẵn contract là event Kafka và interface `@HttpExchange`. Rủi ro thực sự của stack nằm ở những chỗ ít ai để ý: migration Mongo không có công cụ được bảo trì, Mongo phải chạy replica set ở mọi môi trường, và nhịp vá bảo mật của Next.js. Đây là ba thứ cần giải quyết trong hai tuần đầu, trước khi bắt tay vào viết template.
