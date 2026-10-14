# 4e — Phụ lục: đối chiếu revenue cycle Mỹ với BHYT Việt Nam

> **Loại file:** phụ lục (không theo template). Viết để trả lời câu đã nêu ở [CLAUDE.md](../CLAUDE.md#đánh-giá-nguồn-học-đã-chốt): *"Nếu dự án ở Việt Nam thì Giai đoạn 4 dùng lại được bao nhiêu?"*
>
> **Cảnh báo nguồn:** mọi thông tin pháp lý dưới đây lấy từ trang tổng hợp và báo chí (tra cứu 10/2026), **chưa đối chiếu văn bản gốc**. Mỗi dòng có link. Trước khi dùng cho dự án thật, phải đọc bản gốc trên Cổng thông tin Bộ Y tế, Cổng BHXH Việt Nam hoặc Công báo.

## Mục lục

- [1. Khác biệt cấu trúc: 1 payer, giá nhà nước](#1-khác-biệt-cấu-trúc-1-payer-giá-nhà-nước)
- [2. Bảng ánh xạ từng bước](#2-bảng-ánh-xạ-từng-bước)
- [3. Văn bản cần đọc](#3-văn-bản-cần-đọc)
- [4. Dựng trên Medplum thì thế nào](#4-dựng-trên-medplum-thì-thế-nào)
- [5. Phần nào của Giai đoạn 4 dùng lại được](#5-phần-nào-của-giai-đoạn-4-dùng-lại-được)

Nhãn: 📘 văn bản/báo chí (có link) · ✅ code Medplum · 💭 ví dụ · 🧠 suy luận.

---

## 1. Khác biệt cấu trúc: 1 payer, giá nhà nước

🧠 3 khác biệt quyết định mọi thứ phía sau:

| | Mỹ | Việt Nam (BHYT) |
|---|---|---|
| **Số payer** | Hàng trăm (Medicare, Medicaid, công ty tư) | **1**: Quỹ BHYT do BHXH Việt Nam quản lý (bảo hiểm tư nhân là kênh phụ) |
| **Giá** | Thương lượng theo hợp đồng từng payer ([4b-C](./4b-comparison-dinh-gia.md)) | **Nhà nước quy định**, thống nhất theo hạng bệnh viện |
| **Phần bệnh nhân** | Copay, đồng chi trả, khấu trừ, khác theo gói | **Mức hưởng** theo **nhóm đối tượng** (80/95/100%) và theo **tuyến/nơi đăng ký** |
| **Trung gian** | Clearinghouse (phí theo giao dịch) | Không. Cơ sở gửi thẳng lên **Cổng tiếp nhận dữ liệu Hệ thống thông tin giám định BHYT** |
| **Chuẩn dữ liệu** | X12 270/271, 837, 835 | **XML theo Quyết định 130/QĐ-BYT** và các quyết định sửa đổi |

🧠 Hệ quả: bài toán "nhiều hợp đồng, nhiều clearinghouse" ([4a](./4a-quyen-loi-bao-hiem.md), [4b](./4b-dinh-gia-ghi-phi.md)) gần như **biến mất**. Thay vào đó là bài toán "**khớp đúng danh mục và quy tắc giám định của 1 payer duy nhất**", nơi sai 1 mã là bị **xuất toán** (không thanh toán).

## 2. Bảng ánh xạ từng bước

| # | Bước Mỹ ([4a mục 1](./4a-quyen-loi-bao-hiem.md#1-góc-nhìn-kinh-doanh-4a-nằm-ở-đâu-trong-dòng-tiền)) | Tương đương Việt Nam | Ghi chú |
|---|---|---|---|
| 1 | Ghi nhận bảo hiểm (`Coverage`) | Thẻ BHYT: mã số, thời hạn, **nơi đăng ký khám chữa bệnh ban đầu**, mức hưởng, thời gian tham gia liên tục | Có thể dùng CCCD gắn chip thay thẻ giấy (📘 [hướng dẫn BHXH Đắk Lắk](https://daklak.baohiemxahoi.gov.vn:4545/File_Server_BHXH/documents/39437_Phụ%20lục_hướng%20dẫn%20tra%20cứu%20thẻ%20BHYT%20bằng%20CCCD%20gắn%20chíp.pdf)) |
| 2 | Kiểm tra quyền lợi (270/271) | **Tra cứu thông tin thẻ** trên Cổng tiếp nhận dữ liệu giám định, bằng mã thẻ + họ tên + ngày sinh, hoặc CCCD. Có **hàm tra cứu tự động (API)** để tích hợp vào phần mềm đón tiếp | 📘 [CV 468/BHXH, tra cứu thẻ theo QĐ 1666](https://binhthuan.baohiemxahoi.gov.vn/content/tintuc/Lists/News/Attachments/4549/CV%20468%20BHXH%20VN%20HD%20Tra%20cuu%20thong%20tin%20the%20BHYT%20theo%20QD%201666.pdf). Tài liệu kỹ thuật API không công khai, phải xin BHXH |
| 3 | Duyệt trước (278) | Ít gặp với dịch vụ thông thường. Một số thuốc/kỹ thuật có điều kiện thanh toán riêng | 🧠 Kiểm tra theo danh mục, không theo yêu cầu duyệt |
| 4 | Mã chẩn đoán, dịch vụ | ICD-10 (bản Bộ Y tế), **mã dịch vụ kỹ thuật, thuốc, vật tư theo danh mục dùng chung** | Không có CPT. Mã danh mục là điều kiện để được thanh toán |
| 5 | Định giá | **Giá do nhà nước quy định**: Thông tư 22/2023/TT-BYT thống nhất giá dịch vụ KCB BHYT giữa các bệnh viện cùng hạng (📘 [caselaw.vn](https://caselaw.vn/van-ban-phap-luat/413195-thong-tu-so-22-2023-tt-byt-ngay-17-11-2023-cua-bo-truong-bo-y-te-ve-quy-dinh-thong-nhat-gia-dich-vu-kham-benh-chua-benh-bao-hiem-y-te-giua-cac-benh-vien-cung-hang-tren-toan-quoc-va-huong-dan-ap-dung-gia-thanh-toan-chi-phi-kham-benh-chua-benh-trong-mot-so-truong-hop)). Giá khám, ngày giường theo hạng bệnh viện, giá kỹ thuật dùng chung (📘 [PLO](https://plo.vn/benh-vien-dong-hang-se-ap-dung-cung-mot-muc-vien-phi-post317559.html)) | Có nguồn nói Thông tư 21/2024/TT-BYT (phương pháp định giá) liên quan/thay thế 1 phần, **chưa xác minh** |
| 6 | Gửi claim (837) | Gửi **dữ liệu XML** từng lượt khám lên cổng giám định theo các "Bảng chỉ tiêu" của **QĐ 130/QĐ-BYT** (18/01/2023), sửa bởi **QĐ 4750/QĐ-BYT** (29/12/2023, chính thức từ 01/7/2024) và **QĐ 3176/QĐ-BYT** (2024, sửa tạm thời) | 📘 [luatvietnam QĐ 130](https://luatvietnam.vn/y-te/quyet-dinh-130-qd-byt-241607-d1.html), [CV 1245/BHXH-CNTT](https://luatvietnam.vn/bao-hiem/cong-van-1245-bhxh-cntt-2024-trien-khai-kiem-thu-tiep-nhan-du-lieu-dien-tu-xml-413387-d6.html), [QĐ 3176](https://hethongphapluat.com/quyet-dinh-3176-qd-byt-6ff29c2a9402ae578ee567be63935ec1.html) |
| 7 | Kết quả thanh toán (835) | **Kết quả giám định**: chấp nhận / **xuất toán** từng khoản; quyết toán định kỳ với BHXH | 🧠 Xuất toán ≈ denial, nhưng không có hợp đồng để "điều chỉnh CO" |
| 8 | Phần bệnh nhân | **Cùng chi trả** theo mức hưởng + phần ngoài phạm vi BHYT + chênh lệch dịch vụ theo yêu cầu | |
| 9 | Chất lượng | Bộ tiêu chí chất lượng bệnh viện, không gắn tiền trực tiếp như value-based care Mỹ | 🧠 |

**Các bảng XML** (📘 theo [tóm tắt QĐ 130](https://luatvietnam.vn/y-te/quyet-dinh-130-qd-byt-241607-d1.html) và [CV 3434/BYT-BH](https://luatvietnam.vn/y-te/cong-van-3434-byt-bh-bo-y-te-115161-d6.html), chưa đối chiếu danh sách cột gốc):

| Bảng | Nội dung | Tương đương FHIR 🧠 |
|---|---|---|
| XML1 | Chỉ tiêu tổng hợp lượt khám chữa bệnh | `Encounter` + `Claim` (đầu) + `Coverage` |
| XML2 | Chi tiết thuốc | `MedicationRequest`/`MedicationDispense` + `Claim.item` |
| XML3 | Chi tiết dịch vụ kỹ thuật, vật tư y tế | `Procedure`/`ServiceRequest` + `ChargeItem` + `Claim.item` |
| XML4 | Chi tiết dịch vụ cận lâm sàng (kết quả) | `DiagnosticReport` + `Observation` |
| XML5 | Diễn biến lâm sàng | `ClinicalImpression`, ghi chép ([3c](./3c-charting-encounter.md)) |
| Khác | Hồ sơ bệnh án HIV/AIDS, giấy chuyển tuyến, giấy ra viện… | `DocumentReference`, `Composition` |

🧠 Khác biệt lớn với 837: XML BHYT mang cả **dữ liệu lâm sàng** (kết quả cận lâm sàng, diễn biến) để **giám định tự động** tính hợp lý. 837 Mỹ chỉ mang mã, phần hồ sơ lâm sàng chỉ gửi khi payer yêu cầu.

**Mức hưởng** (📘 theo [ebh.vn](https://ebh.vn/bao-hiem-y-te/muc-huong-bao-hiem-y-te-cho-nguoi-lao-dong-nhu-the-nao), [accgroup.vn](https://accgroup.vn/bao-hiem-y-te-duoc-giam-bao-nhieu-phan-tram/), [lsvn.vn](https://lsvn.vn/cac-truong-hop-duoc-huong-100-chi-phi-kham-chua-benh-khi-dong-bhyt-05-nam-lien-tuc-1708504378-a141113.html); dựa trên Điều 22 Luật BHYT trước sửa đổi):
- **100%**: quân nhân, công an, người có công, trẻ dưới 6 tuổi, hộ nghèo…; người tham gia **5 năm liên tục** và cùng chi trả trong năm > 6 tháng lương cơ sở.
- **95%**: người hưởng lương hưu, hộ cận nghèo…
- **80%**: còn lại.

**Luật sửa đổi**: Luật số 51/2024/QH15 sửa đổi Luật BHYT, **hiệu lực 01/7/2025**, 1 số điều áp dụng từ 01/01/2025. Thay đổi về chuyển người bệnh giữa cơ sở theo yêu cầu chuyên môn, mở rộng đối tượng (📘 [luatvietnam: 3 điểm mới về chuyển tuyến](https://luatvietnam.vn/bao-hiem/3-diem-moi-ve-chuyen-tuyen-bhyt-tu-ngay-01-7-2025-563-100345-article.html), [Báo Lào Cai](https://baolaocai.vn/sua-doi-luat-bao-hiem-y-te-theo-huong-mo-rong-quyen-loi-thuan-tien-cho-nguoi-tham-gia-post399387.html)). Có nguồn nhắc Nghị định 02/2025/NĐ-CP về giới hạn thanh toán 100%. **Mức hưởng theo tuyến sau 01/7/2025 chưa xác minh**.

## 3. Văn bản cần đọc

Theo thứ tự ưu tiên cho 1 dự án phần mềm phòng khám có BHYT:

| # | Văn bản | Vì sao | Trạng thái kiểm tra |
|---|---|---|---|
| 1 | QĐ 130/QĐ-BYT + QĐ 4750/QĐ-BYT + QĐ 3176/QĐ-BYT | Định dạng dữ liệu phải gửi. **Quyết định cấu trúc dữ liệu** của cả hệ thống | Có thể đã có sửa đổi sau 3176 |
| 2 | Luật 51/2024/QH15 + nghị định hướng dẫn (VD NĐ 02/2025) | Mức hưởng, tuyến, đối tượng | Chưa đọc bản gốc |
| 3 | Thông tư giá (22/2023/TT-BYT, 21/2024/TT-BYT) | Bảng giá BHYT | Quan hệ thay thế chưa rõ |
| 4 | Thông tư 13/2025/TT-BYT (bệnh án điện tử, thay TT 46/2018) | Ký số, định danh ([3c](./3c-charting-encounter.md#4-ký-và-khoá-hồ-sơ)) | Nguồn tổng hợp |
| 5 | Thông tư về đơn thuốc điện tử (27/2021, 04/2022, 26/2025) | Liên thông đơn thuốc ([3e](./3e-thuoc-ke-don.md#7-liên-hệ-việt-nam)) | Nguồn tổng hợp |
| 6 | Tài liệu kỹ thuật API cổng giám định (BHXH cấp) | Tra cứu thẻ, gửi XML | Không công khai |

## 4. Dựng trên Medplum thì thế nào

🧠 Thiết kế phác thảo, theo đúng điểm mở rộng Medplum đã có:

| Việc | Cách làm trên Medplum | Dựa trên |
|---|---|---|
| Thẻ BHYT | `Coverage`: `identifier` system `urn:vn:bhyt` (💭 chưa có URI chuẩn), `period`, `payor` = Organization BHXH, `class` = nơi đăng ký ban đầu, `costToBeneficiary` = mức hưởng (%) | [4a mục 2](./4a-quyen-loi-bao-hiem.md#2-coverage-1-hợp-đồng-bảo-hiểm) |
| Tra cứu thẻ | `CoverageEligibilityRequest/$submit` + setting `ELIGIBILITY_SUBMIT_OPERATION` → Bot gọi API cổng giám định | ✅ `packages/server/src/fhir/operations/utils/submit.ts:21-51` |
| Bảng giá BHYT | `ChargeItemDefinition` mỗi mã dịch vụ, 2 nhóm: giá BHYT, giá dịch vụ (tự nguyện). **Giá theo hạng bệnh viện** → mỗi cơ sở 1 bộ, hoặc điều kiện theo extension hạng trên ChargeItem | [4b](./4b-dinh-gia-ghi-phi.md). 🧠 Giá nhà nước 1 mức nên `ChargeItemDefinition` **đủ dùng** hơn so với bối cảnh Mỹ |
| Gửi XML | `Claim/$submit` + `CLAIM_SUBMIT_OPERATION` → Bot đọc Encounter, ChargeItem, MedicationRequest, DiagnosticReport, dựng XML1–XML5, ký số, gửi | ✅ `claimsubmit.ts:11`, `:42-45` |
| Kết quả giám định | Bot ghi `ClaimResponse` (từng khoản chấp nhận/xuất toán) + `Task` cho khoản bị xuất toán | [4c](./4c-claim.md), [3d](./3d-careplan-task.md) |
| Danh mục mã | `CodeSystem` cho mã dịch vụ kỹ thuật, thuốc, vật tư theo danh mục dùng chung | [1e](./1e-terminology.md) |
| Định danh | `Patient.identifier` CCCD (bắt buộc với bệnh án điện tử và đơn thuốc) | [3a](./3a-dinh-danh-tiep-nhan.md) |

🧠 Điểm khó nhất **không** nằm ở Medplum mà ở **Bot dựng XML**: phải ánh xạ đúng hàng trăm trường, đúng danh mục, đúng quy tắc giám định. Đây là chỗ các phần mềm HIS trong nước cạnh tranh, và là rào cản gia nhập thị trường.

## 5. Phần nào của Giai đoạn 4 dùng lại được

| Module | Dùng lại ở Việt Nam |
|---|---|
| [4a](./4a-quyen-loi-bao-hiem.md) | **Cơ chế** (Coverage, `$submit` → Bot) dùng lại 100%. **Nội dung** (X12, clearinghouse, COB nhiều payer) không dùng, trừ khi có bảo hiểm tư nhân bổ sung |
| [4b](./4b-dinh-gia-ghi-phi.md) | Dùng được và **khớp hơn** Mỹ: giá nhà nước là bảng giá 1 mức. Vẫn cần hiệu lực theo ngày (giá đổi theo thông tư) và nhân số lượng (Medplum chưa có) |
| [4c](./4c-claim.md) | Cơ chế `Claim/$submit` dùng lại. CMS-1500 không dùng. Sổ công nợ vẫn phải tự xây |
| [4d](./4d-do-luong-chat-luong.md) | Ít liên quan tiền trực tiếp |

🧠 Kết luận cho người làm chiến lược: ở Việt Nam, **lợi thế cạnh tranh** của phần mềm phòng khám có BHYT là **độ chính xác giám định** (tỉ lệ xuất toán thấp) và **tốc độ cập nhật** khi văn bản đổi. Medplum cho nền dữ liệu chuẩn và điểm mở rộng đúng chỗ, nhưng không có sẵn bất kỳ phần nào của lớp nghiệp vụ BHYT.
