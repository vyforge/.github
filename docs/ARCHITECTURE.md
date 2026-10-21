# Chuẩn kiến trúc source code Vyforge

Áp dụng cho mọi dự án mới từ 10/2026. Template hiện thực chuẩn này:

- Backend: [`vyforge/template-spring-boot`](https://github.com/vyforge/template-spring-boot)
- Frontend: [`vyforge/template-nextjs`](https://github.com/vyforge/template-nextjs)

Lý do lựa chọn: [báo cáo nghiên cứu](research/2026-10-kien-truc-template.md) và các [ADR](adr/).

## Nguyên tắc

1. **Mọi dự án có cùng hình dạng** — dev chuyển dự án không phải học lại.
2. **Dự án nhỏ không gánh chi phí của dự án lớn** — độ phức tạp tăng theo tier.
3. **Ranh giới được công cụ kiểm tra**, không chỉ ghi trong tài liệu (Spring Modulith, ArchUnit, ESLint).
4. **Chuẩn chung cập nhật một lần, mọi repo nhận được** — reusable workflow, Renovate preset, `copier update`.

## Backend — modular monolith (Spring Modulith)

Package gốc `com.vyforge.<du-an>`. **Mỗi package con trực tiếp là một module nghiệp vụ.** Type public ở package gốc của module là API của module; mọi package con là internal. `ModularityTests` (`ApplicationModules.verify()`) fail khi module khác dùng phần internal.

### Profile A — simple module (mặc định)

```
com.vyforge.crm
├── CrmApplication.java          @Modulithic(sharedModules = "shared")
├── shared/                      shared kernel rất nhỏ: lỗi, bảo mật, migration
├── customer/                    ← gốc module = public API
│   ├── package-info.java        @ApplicationModule @NullMarked
│   ├── CustomerService.java     facade public
│   ├── CustomerCreated.java     event public (có eventId)
│   ├── web/                     internal: controller
│   └── persistence/             internal: document, repository
└── notification/                lắng nghe CustomerCreated bằng @ApplicationModuleListener
```

### Profile B — hexagonal module (module lõi phức tạp, tier 2)

```
com.vyforge.acmeshop.order
├── package-info.java            @ApplicationModule(allowedDependencies = {"customer :: api", "shared"})
├── api/                         @NamedInterface("api"): OrderApi, OrderPlaced (@Externalized)
├── domain/                      aggregate, value object — KHÔNG import Spring/Mongo
├── application/
│   ├── port/in/                 @PrimaryPort — use case
│   ├── port/out/                @SecondaryPort — OrderStore, PaymentGateway
│   └── service/                 triển khai use case
└── adapter/
    ├── web/                     @PrimaryAdapter — REST
    ├── messaging/               Kafka listener / publisher
    ├── persistence/             @SecondaryAdapter — MongoDB
    └── cache/                   @SecondaryAdapter — Redis
```

### Tier

| Tier | Dự án | Bên trong module | Giữa các module |
|---|---|---|---|
| 0 — PoC / CRUD nhỏ | 1–2 dev, < 3 tháng | Phẳng | Gọi facade |
| 1 — Dự án khách hàng chuẩn | 2–6 dev | Profile A | Event qua Modulith (lưu trong MongoDB) |
| 2 — Product / domain phức tạp | 5+ dev, nhiều năm | Profile B cho module lõi, A cho phần còn lại | Event + `@Externalized` ra Kafka |
| 3 — Tách service | Module cần scale/release/team riêng | Giữ nguyên module | Kafka + `@HttpExchange` làm contract |

Ngưỡng số dev là ước lượng; điều chỉnh theo thực tế. Khung ngoài giống nhau ở mọi tier nên nâng tier chỉ là refactor từng module.

### Event và dữ liệu

- Event giữa module được Spring Modulith ghi trong cùng transaction MongoDB (outbox) → **MongoDB phải chạy replica set ở mọi môi trường**.
- Event ra Kafka: key = ID aggregate, có `eventId`, consumer idempotent (unique index `processed_messages` hoặc Redis `SETNX`).
- Index/schema MongoDB chỉ thay đổi qua Mongock `@ChangeUnit` trong `shared/migration` (ADR-0002).

## Frontend — Next.js App Router, feature-based, Next.js làm BFF

```
src/
├── app/                 CHỈ routing: page/layout/route handler
│   ├── [locale]/        /vi (mặc định), /en
│   └── api/             health, auth/[...all]
├── features/<ten>/      components, api/queries.ts (server-only), api/actions.ts ('use server'), schemas.ts, index.ts
├── shared/              ui, lib (env, cn), api (client server-only, Problem Details), auth (DAL)
├── i18n/                next-intl
└── proxy.ts             chỉ định tuyến ngôn ngữ
```

- Hướng phụ thuộc `app → features → shared`; import feature qua `@/features/<ten>` (ESLint chặn import sâu).
- **Đọc** dữ liệu trong Server Component qua client server-only; **ghi** bằng Server Action (validate Zod, kiểm tra quyền, map Problem Details về lỗi form).
- **Auth**: Better Auth + Keycloak, không database — session và token nằm trong cookie HttpOnly mã hoá; trình duyệt không bao giờ thấy access token (ADR-0006).
- TanStack Query chỉ khi cần dữ liệu tương tác (polling, infinite list); Zustand chỉ khi thật cần state toàn cục.
- Dự án có ≥ 2 frontend hoặc product lớn: cân nhắc Turborepo + pnpm workspace và biến thể Feature-Sliced Design.

## Delivery

- Mỗi dự án tách repo `<du-an>-backend`, `<du-an>-frontend` (và `<du-an>-infra` khi deploy Kubernetes).
- CI gọi reusable workflow của org: `java-ci.yml@v1`, `node-ci.yml@v1`. Mọi action bên thứ ba ghim theo commit SHA.
- Image multi-stage, chạy non-root; cấu hình qua biến môi trường runtime để một image dùng cho mọi môi trường.
- Dependency cập nhật bằng Renovate theo preset `github>vyforge/.github:renovate-config`.

## Quy tắc

### Bắt buộc (mọi dự án mới, mọi tier)

| # | Phạm vi | Quy tắc |
|---|---|---|
| B1 | Backend | Java 25, Spring Boot 4.1.x qua parent/BOM (sau này là `platform-parent`); không hard-code version trong module; wrapper (`mvnw`) được commit |
| B2 | Backend | Package gốc `com.vyforge.<du-an>`; mỗi package con trực tiếp là một module nghiệp vụ; không có `common`/`util` ở top-level, chỉ `shared` khai báo qua `sharedModules` |
| B3 | Backend | Test `ApplicationModules.verify()` + `Documenter` chạy trong CI; không dùng `Type.OPEN` trừ khi di chuyển code legacy, và phải có ADR |
| B4 | Backend | Module khác chỉ được dùng package gốc của module hoặc `@NamedInterface`; event dùng chéo module nằm trong API của module |
| B5 | Backend | Jackson 3; `RestClient` + `@HttpExchange` (không dùng `RestTemplate`); `@NullMarked` trong mọi `package-info.java`; record cho DTO và `@ConfigurationProperties` |
| B6 | Backend | Mongo chạy replica set ở mọi môi trường, kể cả local và Testcontainers; `@Document(collection=...)` tường minh; index chỉ tạo bằng migration Mongock, tắt auto-index ở mọi môi trường (ADR-0002) |
| B7 | Backend | Event có `eventId`; Kafka key = ID aggregate; consumer idempotent; DLT qua `DefaultErrorHandler`/`@RetryableTopic` |
| B8 | Backend | JUnit (theo BOM) + AssertJ + `@MockitoBean` + Testcontainers 2 `@ServiceConnection`; JaCoCo có ngưỡng (mặc định 60% dòng); Spotless (palantir-java-format) `check` trong CI |
| B9 | Backend | Resource server JWT Keycloak (`issuer-uri`, kiểm tra `aud`); Problem Details RFC 9457; springdoc 3.x; version API theo path `/api/v{n}` (ADR-0005) |
| B10 | Backend | Actuator liveness/readiness, starter OpenTelemetry, log JSON có cấu trúc ngoài môi trường local |
| B11 | Frontend | Next ≥ 16.4.1 (bản patch mới nhất), React 19, TypeScript 6 strict, ESLint 9, Node 24 LTS, pnpm 11 ghim qua `packageManager`, chỉ một lockfile, `--frozen-lockfile` |
| B12 | Frontend | `app/` chỉ chứa routing; code nằm trong `features/*` (có `index.ts` public) và `shared/`; ranh giới import kiểm tra bằng ESLint |
| B13 | Frontend | Gọi Spring qua client `server-only` sinh từ OpenAPI; token không bao giờ xuống trình duyệt hay vào props; mỗi Server Action tự kiểm tra authN/authZ; DAL là server-only; không dựa vào `proxy.ts` để phân quyền |
| B14 | Frontend | `proxy.ts` (không dùng `middleware.ts`); async request API; `output: 'standalone'`; không có cấu hình môi trường trong `NEXT_PUBLIC_*` |
| B15 | Frontend | ESLint flat config `eslint-config-next`; Prettier; Vitest cho unit; Playwright smoke E2E |
| B16 | Delivery | Image non-root, multi-stage, có SBOM + provenance, quét Trivy (ghim SHA); dùng một digest cho mọi môi trường |
| B17 | Delivery | CI gọi reusable workflow `vyforge/.github/...@v1`; mọi third-party action ghim theo SHA; `permissions: contents: read` mặc định |
| B18 | Delivery | Secret chỉ đi qua ESO / GitHub Environments; không có trong Git; bật secret scanning |
| B19 | Delivery | Dependency bot theo preset của org, có fast-track bảo mật cho `next`/`react`/`sharp`/Spring Boot |
| B20 | Repo | Tạo repo bằng Copier từ `template-*`; giữ `.copier-answers.yml`; tên repo theo `<du-an>-<thanh-phan>` |

### Khuyến nghị

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
| K9 | Frontend | `cacheComponents: true`; bật React Compiler sau khi lint sạch; oxlint pre-pass; TS 7 cho typecheck khi `typescript-eslint` hỗ trợ; MSW; Storybook cho dự án có design system |
| K10 | Frontend | Turborepo + pnpm workspace khi một dự án có ≥ 2 frontend; biến thể FSD cho product lớn |
| K11 | Delivery | Helm app chart chung (OCI) + Argo CD + repo `<du-an>-infra`; cân nhắc Kargo cho promotion |
| K12 | DX | Lefthook + commitlint; devcontainer (Java 25 + Node 24); `compose.yaml` có Mongo replica set, Kafka, Redis, Keycloak |

## Quyết định đã duyệt

| ADR | Quyết định |
|---|---|
| [0001](adr/0001-maven.md) | Maven 3.9 (wrapper) cho backend |
| [0002](adr/0002-mongodb-migrations-mongock.md) | Mongock 5.5.1 (driver thuần) cho migration MongoDB |
| [0003](adr/0003-modular-monolith.md) | Modular monolith (Spring Modulith); hexagonal chỉ cho module lõi tier 2 |
| [0004](adr/0004-renovate.md) | Renovate với preset của org thay cho Dependabot |
| [0005](adr/0005-api-versioning.md) | Version API theo path `/api/v{n}` |
| [0006](adr/0006-frontend-auth-bff.md) | Next.js làm BFF, Better Auth + Keycloak; phương án B là Spring Cloud Gateway |
