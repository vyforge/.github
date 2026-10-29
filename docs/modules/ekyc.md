# eKYC — định danh khách hàng điện tử

## 1. Mục đích

Cho app (ví, tín dụng, chứng khoán, bảo hiểm, dịch vụ công…) một luồng định danh điện tử dùng chung: đọc giấy tờ (OCR,
MRZ, QR CCCD), đọc chip NFC CCCD và xác thực chip theo ICAO 9303 ở server, so khớp khuôn mặt + liveness qua provider,
luật rủi ro, duyệt tay có SLA và bốn mắt, rồi cấp "mức định danh" cho chủ thể. Bằng chứng được mã hoá từng đối tượng,
có retention theo lớp và legal hold. Thiết kế: `docs/design/ekyc.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-ekyc` | platform-java 0.16.1; test-jar có `EkycProviderContractTests`, `ChipFixture` cho adapter mới |
| Package web | `@vyforge/ekyc` 0.2.0 | BFF route handler, `/capture` (camera, kiểm ảnh mờ/loá), `/react`, `/ui` (`VerificationWizard`, `StepUpDialog`, `KycBadge`…), `/admin` (`ReviewWorkbench`, `EvidenceViewer`…) |
| Zone admin | `apps/ekyc-admin` (`/m/ekyc`) | hàng đợi duyệt, phiên, chủ thể, flow, provider, cài đặt |
| Phụ thuộc bắt buộc | PostgreSQL, `vyforge.crypto.*` (`SecretCipher`), `platform-starter-scheduling` (queue `ekyc`) | production cần module files (store bằng chứng) |

## 3. Tính năng chính

- **Giấy tờ**: OCR qua provider, MRZ TD1/TD3 (kiểm chữ số), QR mặt trước CCCD.
- **Chip NFC** (ICAO 9303): passive authentication ở server trên BouncyCastle (hash DG ↔ SOD, chữ ký SOD, chuỗi DS →
  CSCA), active authentication với challenge do server cấp (RSA ISO/IEC 9796-2, ECDSA), xác thực chip với C06/RAR qua
  provider.
- **Khuôn mặt**: so khớp 1:1, liveness qua provider; ghi chứng nhận PAD của provider, flow đặt `minPadLevel`.
- **Flow có version** (YAML mẫu ở `META-INF/vyforge/ekyc/flows`, vd `sbv-payment-account`), luật rủi ro, kết quả cấp mức
  cho chủ thể (`EkycSubjects.hasLevel`), thu hồi/tạm khoá, giấy tờ sắp hết hạn.
- **Duyệt tay**: hàng đợi, SLA, bốn mắt, ghi đè có người thứ hai; hoặc duyệt qua workflow.
- **Step-up**: `StepUpAuthentication` (flow `step-up`) cho thao tác rủi ro cao.
- **Bằng chứng**: mã hoá từng đối tượng (DEK bọc bằng `SecretCipher`), store `files` hoặc `jdbc`, token xem có hạn 60s,
  retention theo lớp có mức sàn (vd `KYC_RECORD: P5Y` — Luật PCRT 14/2022, TT 27/2025/TT-NHNN), legal hold, xoá bằng
  huỷ khoá, ân hạn 7 ngày trước khi purge.
- **Provider**: `mock` (tắt mặc định), `fptai` (FPT.AI). Provider mới: bean `EkycProvider` + capability
  (`DocumentOcr`, `FaceMatcher`, `LivenessDetector`, `ChipVerifier`…).
- **API Java**: `EkycVerifications` (start, submit), `EkycSubjects`, `EkycReviews` (`decideExternal`,
  `detachExternal`), `StepUpAuthentication`.
- **SPI**: `RiskRule`, `RiskSignalProvider`, `EkycInterceptor`, `ReviewRouter`, `ReasonCatalogContributor`,
  `EkycQuotaGuard` (hạn mức theo gói), `ExternalReview`.
- **REST** (`/api/v1/ekyc`): `verifications/{id}/...` (nộp ảnh, chip, selfie), `/capture/{id}/...` (token chụp
  `X-Ekyc-Capture`), `/callbacks/**` (public), `/admin/**`, `/internal/**`; `GET /admin/info` cho admin MFE. Chi tiết:
  thiết kế §5.2.

## 4. Cách dùng nhanh

```xml
<dependency>
  <groupId>com.vyforge.platform</groupId>
  <artifactId>platform-starter-ekyc</artifactId>
</dependency>
```

```yaml
vyforge:
  ekyc:
    evidence.store: files          # production qua module files; dev/test: jdbc
    providers:
      fptai: { enabled: true }     # chỉ enabled + base-url; khoá API nhập qua admin (ekyc:provider:Manage, cần xác nhận)
    platform-accounts.shared: false
```

```java
StartedVerification started = verifications.start(StartVerification.of(sub, "sbv-payment-account", consents));
// app mobile/web nộp ảnh, chip, selfie qua REST
verifications.submit(started.verificationId(), Duration.ofSeconds(5));
if (subjects.hasLevel(sub, "sbv-payment-account")) { ... }
```

Frontend: route BFF `app/api/ekyc/[...op]/route.ts` dùng `createEkycRouteHandlers`, giao diện `<VerificationWizard>`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.ekyc.enabled` | `true` | Tắt toàn bộ module |
| `vyforge.ekyc.mode` | `embedded` | `remote`: gọi `/api/v1/ekyc/internal/**` của server |
| `vyforge.ekyc.deployment` | `embedded` | `services` khi chạy trong platform-services (báo qua `admin/info`) |
| `vyforge.ekyc.api-enabled` / `public-endpoints` | `true` / `true` | REST; tự mở `/api/v1/ekyc/callbacks/**` |
| `vyforge.ekyc.sync-wait` / `verification-ttl` / `capture-token-ttl` | 8s / 30m / 15m | |
| `vyforge.ekyc.evidence.store` | `files` | `files` hoặc `jdbc`; bucket `ekyc-evidence` |
| `vyforge.ekyc.evidence.max-upload-size` / `max-video-size` | 5MB / 20MB | |
| `vyforge.ekyc.evidence.purge-grace` | 7d | Ân hạn trước khi purge |
| `vyforge.ekyc.retention.defaults` / `floor` / `event-retention` | — / — / `P6Y` | Retention theo lớp |
| `vyforge.ekyc.providers.mock.enabled` | `false` | Provider giả cho dev |
| `vyforge.ekyc.defaults.review-external` | `false` | Mặc định đưa hồ sơ duyệt tay sang workflow (tenant ghi đè `review.externalReview`) |
| `vyforge.ekyc.integrations.workflow.enabled` / `definition-key` | `true` / `ekyc-review` | Cầu nối workflow |
| `vyforge.ekyc.auth-server.url` / `client-id` / `client-secret` | — | Đồng bộ cờ định danh lên auth-server |
| `vyforge.ekyc.schema.migrate` | `true` | Flyway riêng `flyway_schema_history_ekyc` |

## 6. Tích hợp với module khác

| Module | Tích hợp |
|---|---|
| files | Store bằng chứng mặc định (`evidence.store: files`); thiếu files thì phải đặt `jdbc` |
| workflow | Hồ sơ duyệt tay mở thành hồ sơ workflow `ekyc-review` (subject `vrn:ekyc:verification/<id>`, không chứa PII); `APPROVED`/`REJECTED`/rút → kết quả tương ứng |
| platform-auth-server | Việc `sync-identity` cập nhật claim `vyforge_identity_verified` (`PUT/DELETE /api/v1/admin/users/{sub}/identity-verification`, scope `auth.identity.verify`) |
| features / billing | Hạn mức theo gói qua bean `EkycQuotaGuard` |
| Kafka | Event ra topic `vyforge.ekyc` khi bật externalization |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded`; `remote` (`EkycSubjects`, `EkycVerifications`, `StepUpAuthentication` gọi REST nội bộ, kèm
  `X-Tenant-Id`) tới platform-services hoặc deployment ekyc riêng.
- **IAM**: `ekyc:verification:Start|Get|List|Cancel|Retry|Revoke`, `ekyc:review:List|Assign|Decide|Override`,
  `ekyc:evidence:View|Export|PutLegalHold`, `ekyc:subject:Get|Search|Manage`, `ekyc:identity:Reveal`,
  `ekyc:flow:List|Manage|Activate`, `ekyc:provider:Manage`, `ekyc:settings:Get|Update`, `ekyc:usage:Get`,
  `ekyc:task:Manage`.
- **Sự kiện** (`com.vyforge.platform.ekyc.event`, không chứa PII): `KycVerificationStarted`, `KycVerificationSubmitted`,
  `KycCheckCompleted`, `KycReviewRequired`, `KycReviewOverdue`, `KycResubmissionRequested`, `KycVerificationApproved`,
  `KycVerificationRejected`, `KycVerificationCancelled`, `KycVerificationExpired`, `KycVerificationRevoked`,
  `KycSubjectLevelChanged`, `KycStepUpCompleted`, `KycDocumentExpiring`, `KycEvidencePurged`, `KycTaskFailed`.
- **Giấy phép**: module `licensed` (`ekyc`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: OCR/MRZ/QR, chip NFC PA/AA, face match + liveness, flow + rủi ro + duyệt tay, duyệt qua workflow, step-up, cờ
  định danh ở auth-server, bằng chứng mã hoá + retention, remote, zone admin, package web.
- Provider thật mới có FPT.AI (theo tài liệu công khai); VNPT, Viettel… chờ tài liệu từ hợp đồng với đối tác. Xác thực
  chip với C06/RAR phụ thuộc provider.
- Pháp lý: flow mẫu (`sbv-payment-account`…) phải được pháp chế của dự án duyệt (`docs/legal-open-questions.md`).
- Cờ định danh ở auth-server theo tài khoản (mọi tenant): chủ thể bị thu hồi ở một tenant cũng gỡ cờ.
