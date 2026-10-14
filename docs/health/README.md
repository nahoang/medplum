# Học y tế số qua Medplum — mục lục đọc theo vai trò

> 44 file trong thư mục này là kết quả đọc code Medplum (commit `16cb130`), đối chiếu OpenEMR và HAPI FHIR, theo [kế hoạch](./plan.md). File này giúp chọn **đọc gì trước** tuỳ mục đích. Bối cảnh và quy ước: [`../CLAUDE.md`](../CLAUDE.md).

## Mục lục

- [Cách đọc 1 file](#cách-đọc-1-file)
- [Lộ trình theo vai trò](#lộ-trình-theo-vai-trò)
- [Toàn bộ file theo giai đoạn](#toàn-bộ-file-theo-giai-đoạn)
- [10 phát hiện đáng nhớ nhất](#10-phát-hiện-đáng-nhớ-nhất)

---

## Cách đọc 1 file

| Ký hiệu trong tên | Loại | Đọc khi nào |
|---|---|---|
| `-greenfield-` (G) | Tự thiết kế từ đầu: BA → mô hình → V1/V2/V3 → đối chiếu code thật | Muốn luyện tư duy thiết kế. **Che Phần 5 trở đi** khi tự luyện |
| (không có) (R) | Đọc code thật, có `file:line` | Muốn biết hệ thống thật làm gì |
| `-comparison-` (C) | So sánh N cách làm trên 1 kịch bản | Muốn biết vì sao chọn cách này, khi nào không nên |

Nhãn nguồn trong mọi file: ✅ code thật · 📘 chuẩn/docs/quy định · 💭 ví dụ tự dựng · 🧠 suy luận. **Chỉ ✅ là bằng chứng.**

Kịch bản xuyên suốt: bà **Nguyễn Thị Lan**, 68 tuổi, tiểu đường type 2, khám tại Phòng khám An Bình với BS Trần Văn Minh.

---

## Lộ trình theo vai trò

### Consultant / chiến lược / đầu tư (≈ 3–4 giờ)

Mục tiêu: hiểu sản phẩm, mô hình kinh doanh, dòng tiền, rủi ro.

1. [`5d-tong-hop-chien-luoc.md`](./5d-tong-hop-chien-luoc.md): kết luận, vị trí chuỗi giá trị, open-core, cây quyết định MVP
2. [`00-tong-quan-medplum.md`](./00-tong-quan-medplum.md): bản đồ hệ thống
3. [`4a-quyen-loi-bao-hiem.md`](./4a-quyen-loi-bao-hiem.md) mục 1: toàn bộ revenue cycle Mỹ trong 1 bảng
4. [`4b-comparison-dinh-gia.md`](./4b-comparison-dinh-gia.md): 3 con số giá, hợp đồng payer, so với e-commerce
5. [`4c-claim.md`](./4c-claim.md) mục 1, 7, 8: vòng 837 → 835, chỉ số A/R
6. [`4e-phu-luc-bhyt-viet-nam.md`](./4e-phu-luc-bhyt-viet-nam.md): ánh xạ sang BHYT Việt Nam
7. [`4d-do-luong-chat-luong.md`](./4d-do-luong-chat-luong.md) mục 1: value-based care
8. [`5a-gioi-han-do-luong.md`](./5a-gioi-han-do-luong.md) mục 5, [`5b`](./5b-self-host-sharding-migration.md) mục 2, [`5c`](./5c-vong-doi-project.md) mục 5: đơn vị định giá, chi phí self-host, chi phí chuyển đổi
9. [`2d-interoperability.md`](./2d-interoperability.md): chứng nhận ONC và ý nghĩa kinh doanh

### BA / product (≈ 1–2 ngày)

Mục tiêu: hiểu nghiệp vụ y tế, luyện discovery.

1. [`0a-fhir-primer.md`](./0a-fhir-primer.md), [`0b-ban-do-nghiep-vu-resource.md`](./0b-ban-do-nghiep-vu-resource.md): từ vựng
2. **Toàn bộ file G** (luyện, che Phần 5): [`1c`](./1c-greenfield-phan-quyen.md), [`3a`](./3a-greenfield-dinh-danh-benh-nhan.md), [`3b`](./3b-greenfield-dat-lich.md), [`3d`](./3d-greenfield-careplan.md), [`4a`](./4a-greenfield-quyen-loi-bao-hiem.md), [`4b`](./4b-greenfield-dinh-gia.md)
3. File R ngắn Giai đoạn 3 để biết quy trình lâm sàng: [`3c`](./3c-charting-encounter.md) → [`3e`](./3e-thuoc-ke-don.md) → [`3f`](./3f-xet-nghiem-hinh-anh.md) → [`3g`](./3g-questionnaire.md) → [`3h`](./3h-giao-tiep.md) → [`3i`](./3i-consent.md)
4. [`4e-phu-luc-bhyt-viet-nam.md`](./4e-phu-luc-bhyt-viet-nam.md) nếu dự án ở Việt Nam

### Developer / kiến trúc (≈ 3–5 ngày)

Mục tiêu: hiểu lõi kỹ thuật, quyết định thiết kế, chỗ yếu.

1. [`00`](./00-tong-quan-medplum.md) → [`1a`](./1a-fhir-datastore.md) → [`1a-C`](./1a-comparison-fhir-datastore.md) → [`1b`](./1b-search.md)
2. [`1c`](./1c-access-control.md) + [`1c-C`](./1c-comparison-phan-quyen.md), [`1d`](./1d-auth-smart.md)
3. [`2a`](./2a-subscriptions.md) + [`2a-C`](./2a-comparison-event-delivery.md), [`2b`](./2b-bots.md) + [`2b-C`](./2b-comparison-extensibility.md)
4. [`3b`](./3b-dat-lich.md) + [`3b-C`](./3b-comparison-chong-dat-trung.md): bài toán đồng thời hay nhất repo
5. [`3a`](./3a-dinh-danh-tiep-nhan.md): thuật toán ghép hồ sơ
6. [`5b`](./5b-self-host-sharding-migration.md): migration, sharding
7. Mục "không làm gì" của [`3d`](./3d-careplan-task.md#3-apply-không-làm-gì), [`4b`](./4b-dinh-gia-ghi-phi.md#4-những-gì-apply-không-làm), [`4d`](./4d-do-luong-chat-luong.md#4-4-giới-hạn-khiến-con-số-không-dùng-được-cho-hợp-đồng): biết phải tự viết gì

---

## Toàn bộ file theo giai đoạn

| Giai đoạn | File |
|---|---|
| Kế hoạch | [`plan.md`](./plan.md) |
| 0 — Nền tảng FHIR | [`00`](./00-tong-quan-medplum.md) · [`0a`](./0a-fhir-primer.md) · [`0b`](./0b-ban-do-nghiep-vu-resource.md) |
| 1 — Lõi nền tảng | [`1a`](./1a-fhir-datastore.md) · [`1a-C`](./1a-comparison-fhir-datastore.md) · [`1b`](./1b-search.md) · [`1c-G`](./1c-greenfield-phan-quyen.md) · [`1c`](./1c-access-control.md) · [`1c-C`](./1c-comparison-phan-quyen.md) · [`1d`](./1d-auth-smart.md) · [`1e`](./1e-terminology.md) · [`1e-C`](./1e-comparison-terminology.md) · [`1f`](./1f-history-audit-deletion.md) |
| 2 — Tự động hoá & tích hợp | [`2a`](./2a-subscriptions.md) · [`2a-C`](./2a-comparison-event-delivery.md) · [`2b`](./2b-bots.md) · [`2b-C`](./2b-comparison-extensibility.md) · [`2c`](./2c-agent-hl7v2.md) · [`2d`](./2d-interoperability.md) |
| 3 — Nghiệp vụ lâm sàng | [`3a-G`](./3a-greenfield-dinh-danh-benh-nhan.md) · [`3a`](./3a-dinh-danh-tiep-nhan.md) · [`3b-G`](./3b-greenfield-dat-lich.md) · [`3b`](./3b-dat-lich.md) · [`3b-C`](./3b-comparison-chong-dat-trung.md) · [`3c`](./3c-charting-encounter.md) · [`3d-G`](./3d-greenfield-careplan.md) · [`3d`](./3d-careplan-task.md) · [`3e`](./3e-thuoc-ke-don.md) · [`3f`](./3f-xet-nghiem-hinh-anh.md) · [`3g`](./3g-questionnaire.md) · [`3h`](./3h-giao-tiep.md) · [`3i`](./3i-consent.md) |
| 4 — Revenue cycle | [`4a-G`](./4a-greenfield-quyen-loi-bao-hiem.md) · [`4a`](./4a-quyen-loi-bao-hiem.md) · [`4b-G`](./4b-greenfield-dinh-gia.md) · [`4b`](./4b-dinh-gia-ghi-phi.md) · [`4b-C`](./4b-comparison-dinh-gia.md) · [`4c`](./4c-claim.md) · [`4d`](./4d-do-luong-chat-luong.md) · [`4e`](./4e-phu-luc-bhyt-viet-nam.md) |
| 5 — Vận hành & chiến lược | [`5a`](./5a-gioi-han-do-luong.md) · [`5b`](./5b-self-host-sharding-migration.md) · [`5c`](./5c-vong-doi-project.md) · [`5d`](./5d-tong-hop-chien-luoc.md) |

---

## 10 phát hiện đáng nhớ nhất

Mỗi dòng đều có bằng chứng `file:line` trong file được dẫn.

1. **Lưu 1 bản gốc + nhiều bản chiếu**: nhờ vậy Medplum đổi kiểu cột search (TEXT → RANGE) bằng reindex, không mất dữ liệu ([1a](./1a-fhir-datastore.md)).
2. **Đặt lịch chống trùng bằng SERIALIZABLE**, vì giờ trống là kết quả tính, không có dòng để khoá. Có test bắn 2 request song song ([3b](./3b-dat-lich.md#5-hold--book-ghi-trong-giao-dịch-serializable)). `$hold` **không có hạn giữ chỗ**.
3. **Ghép hồ sơ theo khung CMS v3.4.0**: tổ hợp quy tắc thay vì cộng điểm. Nhưng bước thu ứng viên không chuẩn hoá SĐT, không bỏ dấu, và chữ **Đ bị xoá** khi chuẩn hoá ("Đặng" → "ang") ([3a](./3a-dinh-danh-tiep-nhan.md#6-kịch-bản-bà-lan-chạy-qua-match)).
4. **3 operation "tri thức" là hiện thực tối thiểu**: `PlanDefinition/$apply` bỏ qua điều kiện, `ChargeItemDefinition/$apply` không nhân số lượng, `$evaluate-measure` đếm resource theo ngày sửa ([3d](./3d-careplan-task.md), [4b](./4b-dinh-gia-ghi-phi.md), [4d](./4d-do-luong-chat-luong.md)).
5. **`$submit` không gửi gì**: chỉ điều phối sang Bot theo setting project. Eligibility, claim, prior auth đều do đối tác ([4a](./4a-quyen-loi-bao-hiem.md#4-submit-bộ-điều-phối-không-phải-bộ-gửi)).
6. **App Provider tính lại giá mỗi lần mở tab Billing**: giá lần khám cũ trôi theo bảng giá mới ([4b](./4b-dinh-gia-ghi-phi.md#5-app-provider-tính-lại-giá-mỗi-lần-mở)). Claim thiếu con trỏ chẩn đoán ([4c](./4c-claim.md#3-lập-claim-từ-lần-khám-app-provider)).
7. **Consent không chặn truy cập**, "khoá hồ sơ" chỉ ở giao diện ([3i](./3i-consent.md#4-consent-không-tự-chặn-truy-cập), [3c](./3c-charting-encounter.md#4-ký-và-khoá-hồ-sơ)).
8. **`$set-accounts` không phải tài chính**: gán `meta.accounts` cho phân quyền compartment ([4b](./4b-dinh-gia-ghi-phi.md#6-nhầm-tên-set-accounts-không-phải-tài-chính)).
9. **OpenEMR mạnh hơn ở billing**: sổ cái `ar_activity`, đọc 835, X12 270/837, engine CQL, kho thuốc ([4c](./4c-claim.md#8-đối-chiếu-openemr), [4d](./4d-do-luong-chat-luong.md#5-đối-chiếu-openemr-engine-cql-đầy-đủ)).
10. **Free tier không có Bot**, mà Bot là nơi chứa mọi logic nghiệp vụ: chỗ đặt paywall chính xác của mô hình open-core ([5d](./5d-tong-hop-chien-luoc.md#4-mô-hình-kinh-doanh-open-core)).
