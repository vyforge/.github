# Architecture Decision Records

Mỗi quyết định kiến trúc ảnh hưởng nhiều dự án được ghi thành một ADR: bối cảnh, quyết định, hệ quả. Không sửa ADR đã chấp nhận — tạo ADR mới thay thế và đánh dấu ADR cũ là *Superseded*.

Mẫu: `NNNN-tieu-de.md` với các mục **Trạng thái · Bối cảnh · Quyết định · Hệ quả**.

## Danh sách

- [ADR-0001: Maven 3.9 cho backend](0001-maven.md)
- [ADR-0002: Mongock cho migration MongoDB](0002-mongodb-migrations-mongock.md)
- [ADR-0003: Modular monolith với Spring Modulith; hexagonal chỉ cho module lõi](0003-modular-monolith.md)
- [ADR-0004: Renovate thay cho Dependabot](0004-renovate.md)
- [ADR-0005: Version API theo path](0005-api-versioning.md)
- [ADR-0006: Next.js làm BFF, Better Auth + Keycloak](0006-frontend-auth-bff.md)
- [ADR-0007: PostgreSQL là database chính, migration bằng Flyway](0007-postgresql-primary-database.md)
- [ADR-0008: Identity provider riêng `platform-auth-server` trên Spring Authorization Server](0008-platform-auth-server.md)
- [ADR-0009: Phân quyền nghiệp vụ theo mô hình IAM (policy JSON) trong app](0009-iam-authorization.md)
- [ADR-0010: API Gateway, dịch vụ IAM tập trung và token exchange khi có nhiều service](0010-api-gateway-central-iam.md)
- [ADR-0011: Tenant ở platform-auth-server, claim `tenant_id` và danh bạ người dùng](0011-tenant-and-user-directory.md)
- [ADR-0012: IdP mở rộng — passkey, SAML 2.0, LDAP/AD, Zalo — cấu hình trong database](0012-idp-extensions.md)
- [ADR-0014: CI/CD và registry trung lập với nền tảng git](0014-neutral-ci-cd.md)
- [ADR-0015: Tenant ở tầng dữ liệu, `platform-core` và mặc định cho lời gọi ra ngoài](0015-tenant-data-isolation-platform-core.md)
- [ADR-0020: Module ứng dụng dùng chung (mỗi module một mục)](0020-shared-application-modules.md)
- [ADR-0022: Cơ cấu tổ chức dùng chung](0022-shared-organization-structure.md)
- [ADR-0024: Workflow phê duyệt nhúng Flowable 8](0024-workflow-flowable.md)
- [ADR-0025: Module AI: use case, cổng dữ liệu, ngân sách, Spring AI 2.0](0025-ai-gateway.md)
- [ADR-0026: API key, ứng dụng đối tác và phantom token qua gateway](0026-api-access.md)

Câu hỏi pháp chế mà các ADR và thiết kế đang chờ: [legal-open-questions.md](../legal-open-questions.md).
