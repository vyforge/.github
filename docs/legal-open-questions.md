# Câu hỏi pháp chế đang chờ

> Cập nhật 27/10/2026. Mỗi dòng là một điểm mà thiết kế đã chọn một mặc định để code chạy được, nhưng cần pháp chế xác
> nhận trước khi dự án dùng trên production. Mọi mặc định dưới đây đều là cấu hình: trả lời xong chỉ đổi giá trị, không
> đổi code. Trả lời thì ghi vào cột "Kết luận" (ngày, người xác nhận, văn bản căn cứ) và sửa mặc định ở module tương ứng.

## Dữ liệu cá nhân và yêu cầu của chủ thể (privacy-dsr)

Thiết kế: [privacy-dsr.md](design/privacy-dsr.md) mục 10; ADR: [ADR-0020 mục privacy-dsr](adr/0020-shared-application-modules.md#privacy-dsr).

Trong lúc chờ, engine chặn các phương án lỏng hơn mặc định (forge-115c). Trả lời xong: đặt
`vyforge.privacy.dsr.legal.<q>.status=CONFIRMED` và `reference=<văn bản>`, rồi mới đổi cấu hình.

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| privacy-dsr Q1 | "Ngày" trong hạn thực hiện yêu cầu của chủ thể (NĐ 356/2025 Điều 5, văn bản không ghi "làm việc") là ngày lịch hay ngày làm việc | Ngày lịch | Ngày làm việc (dài hơn, rủi ro hơn) | |
| privacy-dsr Q2 | Mốc bắt đầu tính hạn | Lúc nhận yêu cầu (`RECEIVED`) | Lúc đã xác minh danh tính (`VERIFIED`), có thể bị coi là kéo dài hạn | |
| privacy-dsr Q3 | Thời gian giữ bằng chứng đồng ý sau khi hết quan hệ | 5 năm rồi ẩn danh | Theo thời hiệu khởi kiện dân sự | |
| privacy-dsr Q6 | Ngưỡng "xử lý quy mô lớn" để mất miễn trừ ở Điều 38 | 100.000 chủ thể (theo báo cáo nghiên cứu) | Đối chiếu văn bản hướng dẫn | |
| privacy-dsr Q8 | Quy tắc lưu trữ trong nước (residency) mặc định: phạm vi "dữ liệu người dùng" và thời hạn lưu theo Luật An ninh mạng 2025 / NĐ 333/2026 | `basic.*`, `sensitive.*` lưu ở VN; gửi ra ngoài cần hồ sơ TRANSFER; chế độ `MONITOR` (chỉ ghi nhận) | Bật `ENFORCE` sau khi pháp chế xác nhận | |

## Hoá đơn và thanh toán (billing, einvoice)

Thiết kế: [billing.md](design/billing.md) mục 10.

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| billing Q3a | SaaS có thuộc "dịch vụ phần mềm" không chịu thuế GTGT (KCT) không | Cấu hình theo sản phẩm | — | |
| billing Q3b | Số định danh cá nhân trên hồ sơ khách là dữ liệu cá nhân nhạy cảm hay cơ bản | Cấu hình | — | |
| billing Q3c | Hoá đơn ngoại tệ có bắt buộc in tổng quy đổi VND không | Cấu hình | — | |

## Lưu trữ và thời hạn

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| workflow Q4 | Thời hạn lưu hồ sơ duyệt nhóm `finance` (suy từ Luật Kế toán Điều 16, 19, 41); kế toán của khách cũng cần xác nhận. [workflow.md](design/workflow.md) mục 11, [ADR-0024](adr/0024-workflow-flowable.md) | 10 năm | 5 năm nếu phiếu duyệt chỉ là tài liệu quản lý | |
| tenant-lifecycle Q3 | Thời hạn khi tenant ngừng dùng: cooling-off, giữ dữ liệu, giữ slug (tombstone), giữ biên bản xoá, hạn link tải dữ liệu. Hợp đồng mẫu phải ghi cùng số. [tenant-lifecycle.md](design/tenant-lifecycle.md) mục 10 | 14 ngày / 90 ngày / 180 ngày / biên bản 10 năm / link 30 ngày; ghi đè theo hợp đồng (`retention_days`) | Chủ sở hữu và pháp chế đặt | |
| organization Q3 | Thời hạn ẩn danh hoá phân công đã kết thúc (Luật 91/2025 Điều 25 khoản 2). [organization.md](design/organization.md) mục 10, [ADR-0022](adr/0022-shared-organization-structure.md) | 24 tháng, tenant chỉnh được | — | |

## Thời hạn tính theo ngày làm việc

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| calendar Q5 | Cách hiểu "N ngày làm việc" cho yêu cầu của chủ thể và giờ chốt nhận hồ sơ một cửa. [calendar.md](design/calendar.md) mục 10 | Quy tắc `LEGAL`: không tính ngày nhận, hạn hết 24:00 ngày làm việc thứ N (BLDS 2015 Điều 147–148); giờ chốt cấu hình (`CUTOFF`) | — | |

Cần trước khi dự án khu vực công đi vào vận hành.

## Tin nhắn quảng cáo (notification)

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| NĐ 91/2020 | Diễn giải NĐ 91/2020/NĐ-CP về chống tin nhắn rác: tối đa 3 tin quảng cáo/24 giờ cho một người nhận, khung giờ 7h–22h, nhãn `[QC]`, cách từ chối bằng tin nhắn. Con số lấy từ tóm tắt của cơ quan nhà nước, chưa đối chiếu toàn văn; cần theo dõi nghị định thay thế (Bộ Công an đang lấy ý kiến dự thảo). [notification.md](design/notification.md) mục 8.6 và 10, [ADR-0020 mục notification](adr/0020-shared-application-modules.md#notification) | `compliance.vn.enabled=true` cho kind `PROMOTIONAL`; mọi con số là cấu hình | — | |
| notification | Tracking mở/click cho tin quảng cáo | Tắt mặc định | Bật sau khi pháp chế xác nhận | |

## License thư viện LGPL

| # | Câu hỏi | Mặc định đang dùng | Phương án khác | Kết luận |
|---|---|---|---|---|
| OpenHTMLtoPDF | OpenHTMLtoPDF (LGPL-2.1+) dùng như thư viện, không sửa, đóng trong sản phẩm giao cho khách on-prem. MODULE_PLAN "Cần chủ sở hữu quyết định" mục 5, [export.md](design/export.md) mục 10.2 Q1, [ADR-0020 mục export](adr/0020-shared-application-modules.md#export) | Cô lập trong artifact riêng `platform-report-openhtmltopdf`; lõi `platform-report` chỉ Apache/MIT; export kéo adapter để giữ hành vi cũ ([ADR-0020 mục report-engine](adr/0020-shared-application-modules.md#report-engine)) | Mặc định dùng Gotenberg (MIT, thêm một service) | |
| EU DSS | EU DSS 6.5 (LGPL-2.1) cho chữ ký PAdES và bộ kiểm chuẩn ETSI ([ADR-0020 mục esign](adr/0020-shared-application-modules.md#esign)) | Không ở đường mặc định: chỉ trong `platform-starter-esign-dss`, app chủ động thêm; engine mặc định tự làm trên PDFBox + Bouncy Castle | Dùng EU DSS làm mặc định (ít code tự viết hơn) | |

Hai câu hỏi trên trả lời chung một lần (cùng loại license, cùng cách dùng). JobRunr (LGPL-3.0) và JMRTD (LGPL) đã không
dùng nên không cần hỏi.

## Câu hỏi khác đang ghi trong thiết kế

Không nằm trong danh sách chờ chính nhưng cũng cần pháp chế xem trước khi dùng thật:

- master-data Q4: dùng dữ liệu phụ lục văn bản pháp luật và dữ kiện BIN/SWIFT để phát hành data pack ra ngoài
  ([master-data.md](design/master-data.md)).
- ekyc: flow mẫu (`sbv-payment-account`…) phải được pháp chế của dự án duyệt ([ekyc.md](design/ekyc.md)).
- export Q3 (retention file kết quả, mặc định 7 ngày) và ai Q2 (`crossBorder` mặc định `DENY`) đã chọn phương án an toàn
  hơn, không cần chờ.
