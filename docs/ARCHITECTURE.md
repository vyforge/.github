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
├── shared/                      shared kernel rất nhỏ: lỗi, bảo mật, cấu hình
├── customer/                    ← gốc module = public API
│   ├── package-info.java        @ApplicationModule @NullMarked
│   ├── CustomerService.java     facade public
│   ├── CustomerCreated.java     event public (có eventId)
│   ├── web/                     internal: controller
│   └── persistence/             internal: entity, repository
└── notification/                lắng nghe CustomerCreated bằng @ApplicationModuleListener
```

### Profile B — hexagonal module (module lõi phức tạp, tier 2)

```
com.vyforge.acmeshop.order
├── package-info.java            @ApplicationModule(allowedDependencies = {"customer", "shared", "shared :: persistence"})
├── api/                         @NamedInterface("api"): OrderApi (@PrimaryPort, truy vấn), OrderView, OrderPlaced (@Externalized)
├── domain/                      @AggregateRoot Order, @ValueObject OrderLine/OrderId — KHÔNG import Spring/JPA
├── application/
│   ├── port/in/                 @PrimaryPort — use case (PlaceOrderUseCase)
│   ├── port/out/                @SecondaryPort — OrderStore, CustomerDirectory, PaymentGateway
│   └── service/                 triển khai use case: kiểm tra dữ liệu ngoài aggregate → domain → lưu → phát event
└── adapter/
    ├── web/                     @PrimaryAdapter — REST
    ├── messaging/               Kafka listener / publisher
    ├── persistence/             @SecondaryAdapter — PostgreSQL (JPA), ánh xạ entity ↔ aggregate
    ├── customer/                @SecondaryAdapter — gọi API của module khác
    └── cache/                   @SecondaryAdapter — Redis
```

- Lệnh đi qua `port/in`; module khác chỉ thấy `api/` (truy vấn + event). Tách module khác thành service (tier 3) chỉ cần thay
  adapter tương ứng.
- Luật hexagonal (port không phụ thuộc adapter, ...) do `jmolecules-archunit` kiểm tra, cả trong `ApplicationModules.verify()`;
  "domain không import Spring/JPA" là luật ArchUnit riêng.
- Test: domain test JUnit thuần; `@ApplicationModuleTest` thay secondary port của module khác bằng mock; adapter web test bằng
  `@WebMvcTest(<Controller>.class)` với mock primary port.
- Mẫu đầy đủ: module `order` của `template-spring-boot` (sinh khi tier = 2).

### Tier

| Tier | Dự án | Bên trong module | Giữa các module |
|---|---|---|---|
| 0 — PoC / CRUD nhỏ | 1–2 dev, < 3 tháng | Phẳng | Gọi facade |
| 1 — Dự án khách hàng chuẩn | 2–6 dev | Profile A | Event qua Modulith (lưu trong PostgreSQL) |
| 2 — Product / domain phức tạp | 5+ dev, nhiều năm | Profile B cho module lõi, A cho phần còn lại | Event + `@Externalized` ra Kafka |
| 3 — Tách service | Module cần scale/release/team riêng | Giữ nguyên module | Kafka + `@HttpExchange` làm contract |

Ngưỡng số dev là ước lượng; điều chỉnh theo thực tế. Khung ngoài giống nhau ở mọi tier nên nâng tier chỉ là refactor từng module.

### Event và dữ liệu

- **PostgreSQL** là database chính (ADR-0007). Mỗi module sở hữu bảng của mình (tiền tố `<module>_`); không foreign key, không JOIN chéo module.
- Event giữa module được Spring Modulith ghi vào bảng `event_publication` trong cùng transaction nghiệp vụ (outbox).
- Event ra Kafka: key = ID aggregate, có `eventId`, consumer idempotent (bảng `processed_messages` hoặc Redis `SETNX`).
- Schema chỉ thay đổi qua migration Flyway, đặt cạnh module sở hữu bảng: `db/migration/<module>/V<NNN>__*.sql` (version duy nhất trên toàn ứng dụng); `ddl-auto=validate`.

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
- **Auth**: Better Auth + `platform-auth-server` (OIDC), không database — session và token nằm trong cookie HttpOnly mã hoá; trình duyệt không bao giờ thấy access token (ADR-0006, ADR-0008).
- TanStack Query chỉ khi cần dữ liệu tương tác (polling, infinite list); Zustand chỉ khi thật cần state toàn cục.
- Dự án có ≥ 2 frontend hoặc product lớn: cân nhắc Turborepo + pnpm workspace và biến thể Feature-Sliced Design.

## Thư viện platform

Phần dùng chung nằm trong artifact có version (Renovate tự cập nhật), không copy vào từng dự án:

| Repo | Artifact | Nội dung |
|---|---|---|
| [`platform-java`](https://github.com/vyforge/platform-java) | `com.vyforge.platform:*` (GitHub Packages) | `platform-parent`, `platform-bom`, `platform-starter-web` (Problem Details có `code`, request id, `RestClient` gửi request id và đổi lỗi service khác thành `RemoteServiceException`), `-security` (resource server OIDC, `CurrentUser`, chuyển tiếp token tuỳ chọn), `-authz` (IAM: policy JSON, group, Policy Simulator — ADR-0009), `-jpa` (PostgreSQL/Flyway, `BaseEntity`, audit), `-messaging` (consumer idempotent theo bảng hoặc Redis, Kafka DLT), `-audit` (audit log nghiệp vụ, `@Audited`), `-observability`, `platform-test` (ArchUnit house rules, `TestJwt`); `platform-parent` có profile Error Prone + NullAway (K3), `platform-bom` quản lý jMolecules (K1) và Pact (K5) |
| [`platform-web`](https://github.com/vyforge/platform-web) | `@vyforge/*` (GitHub Packages) | `tsconfig`, `eslint-config` (ranh giới B12), `prettier-config`, `ui`, `api-client` (token ở server, Problem Details → lỗi form, `createFormAction`), `auth` (Better Auth BFF, `requireRole`, `can`/`requirePermission`), `i18n` (next-intl vi/en), `test-config` (Vitest, Playwright, MSW), `iam` (giao diện quản trị IAM), `query` (TanStack Query, K6), `server` (`createEnv` Zod lúc runtime, `healthRoute`); Storybook cho `ui` |
| [`platform-helm-charts`](https://github.com/vyforge/platform-helm-charts) | `oci://ghcr.io/vyforge/charts/app` | App chart: digest, non-root, probe, HPA, PDB, HTTPRoute/Ingress, ExternalSecret, NetworkPolicy |

Caller workflow của dự án cấp `permissions: packages: read` để CI tải được các artifact này. Package npm và chart: thêm repo dự án vào *Manage Actions access* (Read) của từng package. Package Maven chỉ cấp quyền theo repo nguồn, nên repo backend cần secret `PACKAGES_TOKEN` (PAT classic, scope `read:packages`); `java-ci.yml` dùng secret này khi có (`secrets: inherit`).

## Service nền tảng

Các service dùng chung. Mỗi service có một repo và một image OCI, đẩy lên registry do tổ chức cấu hình (`VYFORGE_OCI_REGISTRY`: Harbor on-prem, GHCR hoặc registry của cloud):

| Repo | Vai trò |
|---|---|
| `platform-auth-server` | Identity provider OIDC: đăng nhập, MFA, Google/Microsoft, token exchange RFC 8693 (ADR-0008) |
| `platform-iam-server` | IAM tập trung: lưu policy và group, API nội bộ cho chế độ `remote`, phát event xoá cache qua Redis (ADR-0010) |
| `platform-gateway` | Spring Cloud Gateway: kiểm JWT và `aud`, định tuyến, rate limit; không phân quyền (ADR-0010) |
| `platform-admin` | Console cho người vận hành platform (Next.js): tài khoản, client, role, tenant, IdP, phân quyền (IAM). Trước đây là `platform-auth-admin` |

Kế hoạch thêm, gộp, đổi tên và bỏ module đến cuối 2027 nằm ở [MODULE_PLAN.md](MODULE_PLAN.md). Kế hoạch đó gồm cả việc đưa CI/CD và registry ra khỏi GitHub.

## Delivery

- Mỗi dự án tách repo `<du-an>-backend`, `<du-an>-frontend` (và `<du-an>-infra` khi deploy Kubernetes).
- CI gọi reusable workflow của org: `java-ci.yml@v1`, `node-ci.yml@v1`; CD dùng `container.yml@v1` (build → Trivy → push GHCR kèm SBOM + provenance) và `gitops-bump.yml@v1` (bump digest môi trường dev trong repo infra). Thư viện `platform-*` phát hành bằng `maven-publish`, `npm-publish`, `helm-publish`. Mọi action bên thứ ba ghim theo commit SHA.
- Image multi-stage, chạy non-root; cấu hình qua biến môi trường runtime để một image dùng cho mọi môi trường.
- Dependency cập nhật bằng Renovate theo preset `github>vyforge/.github:renovate-config`.

## Quy tắc

### Bắt buộc (mọi dự án mới, mọi tier)

| # | Phạm vi | Quy tắc |
|---|---|---|
| B1 | Backend | Java 25, Spring Boot 4.1.x qua `com.vyforge.platform:platform-parent`; không hard-code version trong module; wrapper (`mvnw`) được commit |
| B2 | Backend | Package gốc `com.vyforge.<du-an>`; mỗi package con trực tiếp là một module nghiệp vụ; không có `common`/`util` ở top-level, chỉ `shared` khai báo qua `sharedModules` |
| B3 | Backend | Test `ApplicationModules.verify()` + `Documenter` chạy trong CI; không dùng `Type.OPEN` trừ khi di chuyển code legacy, và phải có ADR |
| B4 | Backend | Module khác chỉ được dùng package gốc của module hoặc `@NamedInterface`; event dùng chéo module nằm trong API của module |
| B5 | Backend | Jackson 3; `RestClient` + `@HttpExchange` (không dùng `RestTemplate`); `@NullMarked` trong mọi `package-info.java`; record cho DTO và `@ConfigurationProperties` |
| B6 | Backend | PostgreSQL là database chính, kể cả local và Testcontainers (không H2); schema chỉ đổi bằng Flyway, `ddl-auto=validate`, `open-in-view=false`; không FK/JOIN chéo module; entity có `@Version`; MongoDB chỉ khi có ADR (ADR-0007) |
| B7 | Backend | Event có `eventId`; Kafka key = ID aggregate; consumer idempotent; DLT qua `DefaultErrorHandler`/`@RetryableTopic` |
| B8 | Backend | JUnit (theo BOM) + AssertJ + `@MockitoBean` + Testcontainers 2 `@ServiceConnection`; JaCoCo có ngưỡng (mặc định 60% dòng); Spotless (palantir-java-format) `check` trong CI |
| B9 | Backend | Resource server JWT chuẩn OIDC (`issuer-uri` của `platform-auth-server`, kiểm tra `aud`), không phụ thuộc API riêng của IdP; phân quyền nghiệp vụ nằm trong app theo mô hình IAM của `platform-starter-authz` (ADR-0008, ADR-0009); Problem Details RFC 9457; springdoc 3.x; version API theo path `/api/v{n}` (ADR-0005) |
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
| K8 | Frontend | Better Auth (dự án mới) + `platform-auth-server`; Tailwind v4 + shadcn/ui (Radix); next-intl `vi` mặc định |
| K9 | Frontend | `cacheComponents: true`; bật React Compiler sau khi lint sạch; oxlint pre-pass; TS 7 cho typecheck khi `typescript-eslint` hỗ trợ; MSW; Storybook cho dự án có design system |
| K10 | Frontend | Turborepo + pnpm workspace khi một dự án có ≥ 2 frontend; biến thể FSD cho product lớn |
| K11 | Delivery | Helm app chart chung (OCI) + Argo CD + repo `<du-an>-infra`; cân nhắc Kargo cho promotion |
| K12 | DX | Lefthook + commitlint; devcontainer (Java 25 + Node 24); `compose.yaml` có PostgreSQL, Kafka, Redis, `platform-auth-server` (Keycloak cho tới khi có bản đầu tiên) |

## Quyết định đã duyệt

| ADR | Quyết định |
|---|---|
| [0001](adr/0001-maven.md) | Maven 3.9 (wrapper) cho backend |
| [0002](adr/0002-mongodb-migrations-mongock.md) | *Superseded* bởi 0007 — Mongock chỉ còn cho dự án MongoDB cũ |
| [0003](adr/0003-modular-monolith.md) | Modular monolith (Spring Modulith); hexagonal chỉ cho module lõi tier 2 |
| [0004](adr/0004-renovate.md) | Renovate với preset của org thay cho Dependabot |
| [0005](adr/0005-api-versioning.md) | Version API theo path `/api/v{n}` |
| [0006](adr/0006-frontend-auth-bff.md) | Next.js làm BFF với Better Auth; phương án B là Spring Cloud Gateway |
| [0007](adr/0007-postgresql-primary-database.md) | PostgreSQL 18 + Flyway là database chính; MongoDB chỉ khi có ADR |
| [0008](adr/0008-platform-auth-server.md) | Identity provider riêng `platform-auth-server` trên Spring Authorization Server; Keycloak là phương án dự phòng |
| [0009](adr/0009-iam-authorization.md) | Phân quyền nghiệp vụ theo mô hình IAM (policy JSON, Deny thắng Allow, multi-tenant) trong `platform-starter-authz` |
| [0010](adr/0010-api-gateway-central-iam.md) | Nhiều service: `platform-gateway` (xác thực JWT, định tuyến, rate limit — không phân quyền), `platform-iam-server` (policy tập trung, đánh giá tại từng service), xoá cache qua Redis, token exchange RFC 8693 |
| [0011](adr/0011-tenant-and-user-directory.md) | Tenant ở `platform-auth-server`: chọn tenant khi đăng nhập, claim `tenant_id` trong token, role theo tenant, danh bạ người dùng `/api/v1/directory` |
| [0012](adr/0012-idp-extensions.md) | IdP mở rộng cấu hình trong database (passkey, SAML 2.0, LDAP/AD, Zalo, OIDC), mặc định tắt; đăng nhập qua IdP cũng chọn tenant và phát `tenant_id` |
| [0014](adr/0014-neutral-ci-cd.md) | CI/CD trung lập với nền tảng git: logic trong CLI `vyforge-ci` (repo `platform-ci`, image `ci-toolbox`), file CI chỉ là adapter cho GitHub / Forgejo-Gitea / GitLab, registry và khoá ký là biến môi trường, provenance BuildKit + cosign không Rekor, GitOps pull; preset Renovate `local>vyforge/platform-ci:renovate` |
| [0015](adr/0015-tenant-data-isolation-platform-core.md) | Tenant ở tầng dữ liệu: `platform-core` (`TenantContext`, truyền ngữ cảnh, quy ước cache), tenant chỉ lấy từ token, `@TenantId` của Hibernate + RLS cho tier 2, header Kafka; mặc định timeout/retry cho `RestClient` |
