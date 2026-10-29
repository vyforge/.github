# platform-starter-privacy-dsr — engine quyền riêng tư (consent, DSR, retention, RoPA)

## 1. Mục đích

Luật Bảo vệ dữ liệu cá nhân 91/2025/QH15 và NĐ 356/2025/NĐ-CP buộc bên kiểm soát dữ liệu phải có consent có bằng chứng,
xử lý yêu cầu của chủ thể đúng hạn, lưu giữ có thời hạn, sổ hoạt động xử lý (RoPA), đánh giá tác động, sổ sự cố 72 giờ.
Privacy-dsr là engine làm các việc đó cho mọi app Vyforge, điều phối các `PersonalDataContributor` của từng module để xuất,
xoá, sửa, hạn chế dữ liệu của một chủ thể. SPI nằm ở [`platform-starter-privacy`](platform-starter-privacy.md) nên module
khác không phụ thuộc engine.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-privacy-dsr` | Từ 0.14.0; có phí (key `privacy`) |
| SPI nền | `platform-starter-privacy` | `@PersonalData`, contributor, `PrivacyGuard`, `DataResidency` |
| Package web | `@vyforge/privacy` 0.2.1 | Client API, Server Action, form đồng ý, banner cookie, trang "Quyền riêng tư của tôi", form yêu cầu của chủ thể; `/client` tín hiệu consent; `/admin` màn hình quản trị |
| Zone admin | `apps/privacy-admin` 0.2.1 (`/m/privacy`) | |
| Tài liệu vận hành | `platform-starter-privacy-dsr/OPERATIONS.md` | Checklist trước khi bật cho khách |
| Service | `platform-services` | Bật module `privacy`; app khác dùng `remote` |

Thiết kế: [docs/design/privacy-dsr.md](../design/privacy-dsr.md); câu hỏi pháp lý:
[docs/legal-open-questions.md](../legal-open-questions.md).

## 3. Tính năng chính

- **Consent có bằng chứng**: danh mục mục đích (`legalBasis`, `dataTypes`, `consentTtl`), văn bản thông báo có version,
  biên nhận chỉ ghi thêm có chuỗi băm theo tenant (kiểm hằng tuần), consent ẩn danh (cookie) ghép khi đăng nhập, người
  giám hộ, điểm thu thập.
- **Yêu cầu của chủ thể (DSR)**: 8 loại theo Luật Điều 4, hạn theo NĐ 356 Điều 5, xác minh danh tính (phiên đăng nhập, OTP,
  giấy tờ), duyệt, gia hạn một lần, điều phối contributor trên `DurableTaskQueue` (queue `privacy`), gói ZIP (JSON, CSV,
  README vi/en) tải về có giới hạn số lần và thời hạn.
- **Hạn chế/phản đối xử lý** + `PrivacyGuard` thật (consent + hạn chế + legal basis), `@RequiresPurpose`.
- **Retention trung tâm**: policy, mốc theo sự kiện (`Retention.startClock`), legal hold, quét theo lô, mục cần DPO duyệt.
- **Hồ sơ**: danh mục tài sản sinh từ `@PersonalData` và contributor, RoPA có version, bên xử lý, đánh giá tác động (Mẫu 10)
  và chuyển ra nước ngoài (Mẫu 09) theo JSON Schema, báo cáo tuân thủ.
- **Sổ sự cố** có đồng hồ 72 giờ, Mẫu 08, thông báo chủ thể/công khai, giữ ≥ 5 năm.
- **Residency**: quy tắc nơi lưu và nơi gửi, chế độ `MONITOR`/`ENFORCE`.
- **Offboarding tenant**: `TENANT_EXPORT`/`TENANT_ERASE`, duyệt 4 mắt.
- Contributor tự sinh `app.<entity>` từ `@PersonalDataOwner` + `@PersonalData(type, erase)`.
- REST `/api/v1/privacy`: `/me/**` (chủ thể; không tắt theo gói), `/public/{tenant}/**` (form công khai, consent ẩn danh,
  thông báo sự cố; rate limit), `/admin/**` (dashboard, purposes, notices, requests, restrictions, retention, legal-holds,
  assets, activities, processors, assessments, breaches, residency…), `/internal/**` (remote).
- Bảng bằng chứng có trigger chặn `UPDATE`/`DELETE`; chỉ hàm `SECURITY DEFINER` sửa được.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-privacy-dsr</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/privacy
```

```yaml
vyforge:
  privacy:
    hash-key: ${PRIVACY_HASH_KEY}             # ≥ 32 byte
  crypto:
    keys:
      k2026a: ${CRYPTO_KEY_2026A}             # SecretCipher
    active-key: k2026a
  security:
    public-paths: /api/v1/privacy/public/**
```

Cần PostgreSQL, `platform-starter-scheduling`, `platform-starter-authz`.

```java
@Bean
PrivacyCatalogCustomizer crmPrivacyCatalog() {
    return catalog -> catalog
            .purpose("service.account", p -> p.legalBasis("CONTRACT").required().dataTypes("basic.identity.name"))
            .purpose("marketing.zalo", p -> p.parent("marketing").dataTypes("basic.contact.phone")
                    .consentTtl(Period.ofYears(2)).title("vi", "Nhận ưu đãi qua Zalo", "en", "Offers via Zalo"))
            .retention("customer.closed", r -> r.after("RELATIONSHIP_ENDED", Period.ofYears(5)).anonymize())
            .collectionPoint("signup-web", "WEB", "signup-consent");
}
```

## 5. Cấu hình quan trọng

Mọi khoá dưới `vyforge.privacy.dsr.`.

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `mode` | `embedded` | `remote`: gọi engine ở `platform-services` (`remote.url`, `remote.registration-id`) |
| `sla.execute-day-kind` | `CALENDAR` | **Q1 chờ pháp chế**: `WORKING` cần `legal.q1` |
| `sla.clock-start` | `RECEIVED` | **Q2 chờ pháp chế**: `VERIFIED` cần `legal.q2` |
| `consent.evidence-retention` | `1826d` | **Q3 chờ pháp chế**: ngắn hơn cần `legal.q3` |
| `exemption.large-scale-threshold` | `100000` | **Q6 chờ pháp chế**: cao hơn cần `legal.q6` |
| `legal.q1` … `legal.q8` | `PENDING_LEGAL` | `CONFIRMED` kèm `reference=<số văn bản>` khi pháp chế trả lời |
| `packaging.store` | `auto` | Có files thì bucket `privacy-packages`; `jdbc` giữ trong DB (≤ 200 MB) |
| `packaging.ttl`, `max-downloads`, `link-ttl` | `14d`, `5`, `5m` | Gói tải về |
| `verification.ttl`, `otp-ttl` | `14d`, `15m` | |
| `guard.cache-ttl`, `fail-open` | `30s`, `false` | Lỗi DB thì từ chối |
| `integration.notification.enabled` | `true` | Thông báo qua notification |
| `integration.calendar.enabled` | `true` | Hạn theo lịch nghỉ của tenant |
| `requests.tombstones` | `false` | Ghi HMAC identifier của chủ thể đã xoá |
| `remote-contributors[*]` | — | Contributor ngoài process (`privacy-contributor/v1`) |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: ratelimit, notification, files, calendar.

- **calendar**: `CalendarWorkingDays` (quy tắc `LEGAL`; năm chưa có dữ liệu tính `CONSERVATIVE`; calendar lỗi chỉ trừ
  T7/CN). Không có: `WeekendOnlyWorkingDays`.
- **notification**: `NotificationPrivacyNotifier` (type `privacy.*`), `PrivacyPromotionalConsent` (tin quảng cáo chỉ gửi khi
  purpose `GRANTED`). Không có: chỉ log, OTP không khả dụng.
- **files**: `FilesPackageStore` (bucket `privacy-packages`, nhạy cảm, audit khi tải).
- **tenant-lifecycle**: gửi `TENANT_EXPORT`/`TENANT_ERASE`.
- **Contributor của module**: settings, sequence, organization, search, collaboration, automation, workflow… cài
  `PersonalDataContributor`.
- SPI: `WorkingDays`, `PrivacyNotifier`, `DsrPackageStore`, `SubjectIdentityResolver`, `SubjectProfileResolver`,
  `RequestTypeDefinition`, `RequestInterceptor`, `ConsentInterceptor`, `ConsentMirror`, `AssessmentRenderer`,
  `PrivacyCatalogCustomizer`.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote**: app remote không có bảng, job, API quản trị; `Consents`, `DataSubjectRequests`,
  `Restrictions`, `Retention.startClock`, `PrivacyGuard` (snapshot cục bộ, hết hạn mà không tới được server thì từ chối)
  gọi `/internal/**`. Đặt `remote.service`, `contributor-base-url` và bật `vyforge.privacy.contributor-endpoint.enabled`
  để engine gọi ngược contributor của app.
- **Quyền IAM**: `privacy:<resource>:<Verb>` (`PrivacyDsrActions`); mọi thay đổi và mọi lần xem consent/yêu cầu của người
  khác ghi audit. Token có `act` (thay mặt) không được gửi yêu cầu cho chủ thể.
- **Sự kiện**: `ConsentChanged`, `ReconsentRequired`, `DataSubjectRequestChanged`, `DataSubjectRequestDue`,
  `ProcessingRestricted`, `SubjectErased`, `RetentionClockStarted`, `RetentionExecuted`, `BreachChanged`,
  `BreachDeadlineApproaching`, `AssessmentReviewDue`, `ResidencyViolationDetected`.
- Job `privacy-sla` (15 phút), `privacy-retention` (02:00), `privacy-housekeeping` (03:15), `privacy-consent-verify` (Chủ
  nhật), `privacy-partitions`; metric `vyforge.privacy.*`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm adapter notification, files, calendar, `OPERATIONS.md`, và cơ chế "mặc định an
  toàn": cấu hình platform lỏng hơn thì app không khởi động, settings tenant lỏng hơn bị `422 privacy.legal_pending`.
- **Chờ pháp chế**: Q1 (ngày lịch hay ngày làm việc), Q2 (mốc tính hạn), Q3 (thời hạn giữ bằng chứng consent), Q6 (ngưỡng
  quy mô lớn), Q8 (residency `ENFORCE`).
- **Chưa làm**: adapter export (privacy tự đóng gói, Q5), nộp hồ sơ điện tử lên cổng A05, IAB TCF/GPP, ký số biên nhận.
  Màn quản trị còn thiếu editor JSON Schema cho DPIA, ActivityEditor cho RoPA, so sánh version thông báo, ghi consent thay
  chủ thể, quyết định review retention (API đã có).
- **Đã biết**: `DsrIntegrationTests` có thể timeout khi máy build tải nặng (chạy riêng thì xanh). Sau khi restore backup
  phải chạy lại các yêu cầu ERASE đã hoàn tất sau mốc backup.
