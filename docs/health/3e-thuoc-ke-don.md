# 3e — Thuốc & kê đơn điện tử (eRx)

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"BS Minh kê Metformin cho bà Lan. Đơn thuốc đi từ hệ thống tới nhà thuốc thế nào, và Medplum tự làm phần nào?"*

## Mục lục

- [1. 5 resource về thuốc](#1-5-resource-về-thuốc)
- [2. Mã thuốc: RxNorm, NDC](#2-mã-thuốc-rxnorm-ndc)
- [3. Kê đơn điện tử: Medplum không tự gửi](#3-kê-đơn-điện-tử-medplum-không-tự-gửi)
- [4. Bài toán "không biết đơn đã gửi chưa"](#4-bài-toán-không-biết-đơn-đã-gửi-chưa)
- [5. Nhà thuốc ưa thích](#5-nhà-thuốc-ưa-thích)
- [6. Đối chiếu OpenEMR](#6-đối-chiếu-openemr)
- [7. Liên hệ Việt Nam](#7-liên-hệ-việt-nam)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 5 resource về thuốc

📘 FHIR R4, lớp quyết định HL7:

| Resource | Câu hỏi nó trả lời | Ví dụ bà Lan 💭 |
|---|---|---|
| `Medication` | Thuốc **là gì** (hoạt chất, dạng, hàm lượng) | Metformin 500 mg viên |
| `MedicationRequest` | **Ai yêu cầu** dùng thuốc (đơn/y lệnh) | BS Minh kê 2 viên/ngày × 30 ngày |
| `MedicationDispense` | Nhà thuốc **đã phát** gì | Nhà thuốc Long Châu phát 60 viên |
| `MedicationAdministration` | **Đã cho dùng** (nội trú: điều dưỡng cho uống) | — |
| `MedicationStatement` | Bệnh nhân **nói** mình đang dùng gì | "Tôi đang uống Amlodipine của bác sĩ khác" |

📘 `MedicationRequest.category` phân biệt **đơn ngoại trú** (prescription, bệnh nhân tự mua) và **y lệnh nội trú** (medication order, khoa dược cấp) (`packages/docs/docs/medications/representing-prescriptions-and-medication-orders.mdx`, mục "Distinguishing…").

🧠 Đối chiếu thuốc (medication reconciliation) chính là so `MedicationRequest` (bác sĩ ở đây kê) với `MedicationStatement` (bệnh nhân nói) và `MedicationDispense` (nhà thuốc phát). Lỗi kê trùng thuốc thường nằm ở khoảng chênh giữa 3 nguồn này. Đây là lý do Bot tiếp nhận [3a](./3a-dinh-danh-tiep-nhan.md#bot-tiếp-nhận-mẫu-làm-gì) tạo `MedicationStatement` từ phiếu khai.

## 2. Mã thuốc: RxNorm, NDC

📘 `packages/docs/docs/medications/medication-codes.md`:
- **RxNorm** (Thư viện Y khoa Quốc gia Mỹ): mã khái niệm thuốc lâm sàng (hoạt chất + hàm lượng + dạng), không phụ thuộc nhà sản xuất. Dùng để kê đơn, kiểm tra tương tác.
- **NDC** (FDA): mã **sản phẩm đóng gói** cụ thể (nhà sản xuất, quy cách hộp). Dùng ở khâu phát thuốc, thanh toán.
- Cơ sở dữ liệu thương mại: First Databank (FDB), Medi-Span… cung cấp tương tác thuốc, liều dùng.

✅ Hằng số `NDC`, `RXNORM` được dùng lại trong core (`packages/core/src/medication-order-utils.ts:17`).

🧠 1 RxNorm ↔ nhiều NDC. Kê đơn theo RxNorm, phát theo NDC, thanh toán theo NDC. Đây là chỗ terminology ([1e](./1e-terminology.md)) gặp dòng tiền.

## 3. Kê đơn điện tử: Medplum không tự gửi

📘 Ở Mỹ, đơn điện tử đi qua mạng **Surescripts** (chuẩn NCPDP SCRIPT) tới nhà thuốc. Thuốc gây nghiện (EPCS) cần thêm xác thực 2 yếu tố cho người kê. Medplum **không** kết nối Surescripts trực tiếp mà tích hợp **nhà cung cấp eRx**:

| Gói | Đối tác | Cách tích hợp |
|---|---|---|
| `packages/dosespot-core`, `packages/dosespot-react` | DoseSpot | iframe kê đơn của DoseSpot nhúng vào app + Bot đồng bộ. Hỗ trợ EPCS (📘 `packages/docs/docs/medications/e-prescibe.md`) |
| `packages/scriptsure-react` | ScriptSure | iframe + giỏ thuốc (`useScriptSureCart.ts`) + Task cho tin nhắn từ nhà thuốc (`useScriptSureMessageTask.ts`) |
| `examples/medplum-photon-integration` | Photon Health | Ví dụ |

✅ Core định nghĩa giao diện **trung lập nhà cung cấp** để app không phụ thuộc 1 đối tác (`packages/core/src/medication-order-utils.ts`):
- `MedicationOrderDrugInput` (`:84-97`): NDC, RxNorm, mã nội bộ của vendor, số lượng, số lần tái kê (`refill`), hướng dẫn dùng (`sigLine3`), cho phép thay thuốc tương đương (`useSubstitution`).
- `MedicationOrderRequest` (`:118-141`): bệnh nhân, thuốc, **chẩn đoán ICD**, **bảo hiểm** (`coverageId`, `payerOrganizationId`), **nhà thuốc** (mã NCPDP), số ngày dùng.
- `MedicationOrderResponse` (`:150-156`): mã đơn bên vendor, `launchUrl` (mở iframe ký đơn).

🧠 Đơn thuốc chứa **chẩn đoán và bảo hiểm** vì nhà thuốc Mỹ phải xin bảo hiểm duyệt giá thuốc lúc phát. Đơn thuốc là 1 giao dịch tài chính, không chỉ y khoa.

📘 Docs ghi: không lấy được giá thuốc qua API Medplum. Nhiều vendor eRx không cho nhập bảo hiểm trực tiếp mà lấy từ clearinghouse (`e-prescibe.md`, mục FAQ).

## 4. Bài toán "không biết đơn đã gửi chưa"

✅ `packages/core/src/medication-order-utils.ts:36-82`:
- Luồng: tạo `MedicationRequest` **nháp** trong Medplum → gọi Bot "order medication" → Bot gọi API vendor.
- Nếu gọi vendor lỗi **mà không biết vendor đã nhận chưa** (mất kết nối sau khi gửi), **không xoá** MedicationRequest. Đổi `status: 'unknown'` và gắn `statusReason` = `response-not-received` (`:44-54`, `buildMedicationRequestResponseLostStatusReason` `:68-82`).
- Webhook từ vendor sau đó dùng `statusReason` này để đối soát (`:39-42`).

🧠 Đây là bài toán **giao dịch phân tán** kinh điển (2 hệ thống, 1 thao tác). Xoá bản nháp thì có thể mất dấu 1 đơn **đã gửi tới nhà thuốc thật**. Giữ lại ở trạng thái "không rõ" là lựa chọn an toàn cho người bệnh: thà có 1 bản ghi thừa cần đối soát còn hơn mất 1 đơn thật. Cùng tinh thần với Subscription ở [2a](./2a-subscriptions.md): khi không chắc, giữ dấu vết để đối soát sau.

## 5. Nhà thuốc ưa thích

✅ `packages/core/src/pharmacy-utils.ts`:
- Extension chuẩn HL7 `patient-preferredPharmacy` trên `Patient` (`:9`), trỏ tới `Organization` nhà thuốc.
- Phân loại `primary` / `preferred` bằng CodeSystem riêng của Medplum (`:15-18`).
- Thêm/xoá/đọc (`:109`, `:196`, `:236`).

## 6. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Đơn thuốc | `MedicationRequest` | `prescriptions` (`sql/database.sql:8714`) |
| Kho thuốc tại phòng khám | Không có sẵn (tự mô hình bằng `Medication` + `MedicationDispense`/`SupplyDelivery`) | `drugs` (`:1600`), `drug_inventory` (`:1520`): lô, hạn dùng, tồn kho. Phòng khám **bán thuốc trực tiếp** |
| eRx | DoseSpot / ScriptSure / Photon qua Bot + iframe | Module `oe-module-weno` (Weno eRx), `src/Rx/` |

🧠 OpenEMR có kho thuốc vì phòng khám tư nhỏ ở nhiều nước vừa khám vừa bán thuốc. Medplum không có, vì mô hình ngoại trú Mỹ tách khám và bán thuốc. **Ở Việt Nam, phòng khám tư nhân bán thuốc rất phổ biến**: phần kho thuốc phải tự xây trên Medplum.

## 7. Liên hệ Việt Nam

📘 Văn bản liên quan (tra cứu 10/2026 qua trang tổng hợp, **chưa đối chiếu bản gốc**):
- Thông tư 27/2021/TT-BYT: kê đơn thuốc bằng hình thức điện tử. Đơn điện tử được lập, ký số, chia sẻ, lưu trữ điện tử có giá trị pháp lý như đơn giấy ([luatvietnam](https://luatvietnam.vn/y-te/thong-tu-27-2021-tt-byt-bo-y-te-214399-d1.html?layout=amp), [lsvn](https://lsvn.vn/bo-y-te-quy-dinh-don-thuoc-dien-tu-co-gia-tri-phap-ly-nhu-don-thuoc-giay1657923730-a121292.html)).
- Thông tư 04/2022/TT-BYT: sửa Thông tư 52/2017, 18/2018 và 27/2021, hiệu lực 15/9/2022 ([luatvietnam](https://luatvietnam.vn/y-te/thong-tu-04-2022-tt-byt-225827-d1.html)).
- Thông tư 26/2025/TT-BYT: đơn thuốc ngoại trú, bổ sung trường bắt buộc như số định danh cá nhân. Theo [VnEconomy](https://vneconomy.vn/so-hoa-viec-ke-don-thuoc-tang-tinh-minh-bach-han-che-lam-dung-thuoc.htm), lộ trình hướng tới kê đơn điện tử toàn quốc từ 2026. Chưa xác minh điều khoản thay thế và ngày hiệu lực.
- Quyết định 425/QĐ-BYT (2025): quy chế an ninh mạng của hệ thống quản lý kê đơn, bán thuốc theo đơn ([luatvietnam](https://luatvietnam.vn/y-te/quyet-dinh-425-qd-byt-2025-quy-che-an-ninh-mang-he-thong-quan-ly-ke-don-thuoc-ban-thuoc-theo-don-391009-d1.html)).

🧠 Nếu dùng Medplum ở Việt Nam, phần "gửi đơn" tương đương vai trò DoseSpot ở trên: 1 Bot gọi API của hệ thống đơn thuốc quốc gia, nhận mã đơn về gắn vào `MedicationRequest.identifier`, xử lý lỗi theo đúng mẫu "không biết đã gửi chưa" ở mục 4. Đơn phải có số định danh cá nhân, tức là `Patient.identifier` với `system` cho CCCD ([3a](./3a-dinh-danh-tiep-nhan.md)) trở thành **bắt buộc**.

🧠 Mã thuốc Việt Nam (số đăng ký, mã dược quốc gia) không có sẵn trong Medplum. Cần tự nạp `CodeSystem` ([1e](./1e-terminology.md)).

## Sai lầm thường gặp

1. **Dùng `MedicationRequest` cho thuốc bệnh nhân khai.** Đó là `MedicationStatement`. Dùng nhầm thì bác sĩ "kê" thuốc mà mình không kê.
2. **Xoá `MedicationRequest` nháp khi gọi vendor lỗi.** Mất dấu đơn có thể đã tới nhà thuốc (mục 4).
3. **Lưu mỗi NDC, không lưu RxNorm.** Không kiểm tra tương tác hay đổi thuốc tương đương được.
4. **Tưởng Medplum gửi đơn tới nhà thuốc.** Cần hợp đồng với 1 vendor eRx (Mỹ) hoặc tích hợp cổng quốc gia (Việt Nam).
5. **Không ghi chẩn đoán vào đơn.** Thanh toán bảo hiểm cho thuốc bị từ chối.
