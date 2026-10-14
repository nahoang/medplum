# 2d — Liên thông theo quy định: xuất dữ liệu, tóm tắt hồ sơ, chia sẻ cho bệnh nhân

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Luật buộc phải trả dữ liệu cho bệnh nhân, cho cơ sở y tế khác, cho cơ quan quản lý. Nền tảng có những 'cửa ra' nào, mỗi cửa dùng khi nào?"*
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. Vì sao liên thông là nghĩa vụ, không phải tính năng](#1-vì-sao-liên-thông-là-nghĩa-vụ-không-phải-tính-năng)
- [2. 6 cửa ra dữ liệu](#2-6-cửa-ra-dữ-liệu)
- [3. Bulk $export: dữ liệu hàng loạt](#3-bulk-export-dữ-liệu-hàng-loạt)
- [4. Patient/$everything: toàn bộ hồ sơ 1 người](#4-patienteverything-toàn-bộ-hồ-sơ-1-người)
- [5. $summary và C-CDA: tài liệu tóm tắt](#5-summary-và-c-cda-tài-liệu-tóm-tắt)
- [6. SMART Health Cards và Links: bệnh nhân tự mang hồ sơ](#6-smart-health-cards-và-links-bệnh-nhân-tự-mang-hồ-sơ)
- [7. Chứng nhận ONC của Medplum: góc nhìn kinh doanh](#7-chứng-nhận-onc-của-medplum-góc-nhìn-kinh-doanh)
- [8. Đối chiếu OpenEMR](#8-đối-chiếu-openemr)
- [9. Hệ quả cho dự án Việt Nam](#9-hệ-quả-cho-dự-án-việt-nam)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vì sao liên thông là nghĩa vụ, không phải tính năng

📘 Ở Mỹ, 21st Century Cures Act và quy định của ONC:
- Cấm **"chặn thông tin"** (information blocking).
- Bắt buộc EHR được chứng nhận phải có **API chuẩn** cho bệnh nhân và cho quần thể (tiêu chí `(g)(10)`).
- Bắt buộc có **xuất toàn bộ dữ liệu sức khoẻ điện tử** (tiêu chí `(b)(10)` EHI Export).

Bộ dữ liệu tối thiểu phải trao đổi được gọi là **USCDI**, biểu diễn bằng FHIR theo profile **US Core**. Docs `packages/docs/docs/fhir-datastore/understanding-uscdi-dataclasses.md:5-20`.

🧠 Hệ quả kinh doanh: ở Mỹ, khả năng liên thông là **điều kiện để được bán** cho cơ sở y tế nhận tiền từ chương trình liên bang, không phải điểm cộng. Ở Việt Nam, xu hướng hồ sơ sức khoẻ điện tử và liên thông dữ liệu khám chữa bệnh cũng đang đi theo hướng bắt buộc. Cần đọc văn bản hiện hành.

## 2. 6 cửa ra dữ liệu

✅ `fhir/routes.ts`:

| Cửa | Endpoint | Ai dùng | Định dạng |
|---|---|---|---|
| Bulk export toàn hệ thống | `/$export` (`:246-247`) | Kho dữ liệu, cơ quan báo cáo | NDJSON |
| Bulk export theo nhóm | `/Group/:id/$export` (`:361`) | Bảo hiểm (danh sách thành viên), nghiên cứu | NDJSON |
| Bulk export mọi bệnh nhân | `/Patient/$export` (`:365`) | Chuyển đổi hệ thống | NDJSON |
| Toàn bộ hồ sơ 1 người | `/Patient/:id/$everything` (`:390`) | Bệnh nhân, app bên thứ 3, xuất EHI | Bundle FHIR |
| Tóm tắt hồ sơ | `/Patient/:id/$summary` (`:394`), `/Patient/:id/$ccda-export` (`:398`) | Chuyển viện, giấy ra viện | Bundle FHIR / XML C-CDA |
| Bệnh nhân tự mang | `$generate-smart-health-card` (`:402`), `$generate-smart-health-link` (`:406`) | Bệnh nhân (QR, link) | JWS / JWE |

## 3. Bulk $export: dữ liệu hàng loạt

📘 Chuẩn **FHIR Bulk Data Access**: gửi yêu cầu → nhận URL trạng thái → server chạy nền → trả danh sách file NDJSON (mỗi dòng 1 resource).

✅ `fhir/operations/export.ts`:
- Tham số `_since` (chỉ dữ liệu đổi sau mốc), `_type` (chỉ các loại resource này), `_typeFilter` (`:36-38`).
- `_typeFilter` không hỗ trợ thì **từ chối** thay vì xuất không lọc (`:125-130`). Comment ghi rõ đó là khuyến nghị của chuẩn. 🧠 Đúng nguyên tắc tối thiểu: xuất thừa dữ liệu là rủi ro quyền riêng tư.
- Chạy nền qua `AsyncJob` và worker `workers/bulk-export.ts`. Kết quả ghi thành `Binary` (`operations/utils/bulkexporter.ts`).

🧠 Dùng khi nào:
- Đổ dữ liệu sang kho phân tích hằng đêm (`_since`) cho BI. Đây là cách trả lời "SQL báo cáo" của [1a-C phản biện 2](./1a-comparison-fhir-datastore.md#phần-9--luyện-phản-biện).
- Bảo hiểm lấy dữ liệu của mọi thành viên họ chi trả (`Group/$export`).

## 4. Patient/$everything: toàn bộ hồ sơ 1 người

✅ `fhir/operations/patienteverything.ts:94-115` (`getPatientEverything`):
- Hỗ trợ `_since`, `_count`, `_type`.
- Lõi là `searchPatientCompartment`: lấy mọi resource trong **Patient compartment** ([0a mục 8](./0a-fhir-primer.md#8-compartment)).
- Chính hàm này cũng được dùng để lan nhãn tenant trong `$set-accounts` ([1c mục 5](./1c-access-control.md#5-tầng-4--compartment-tenant-mềm-gắn-lên-dữ-liệu)).

📘 Docs ONC của Medplum (`packages/docs/docs/compliance/onc.md:43`) gắn tiêu chí `b10` (EHI Export) với `$everything`.

## 5. $summary và C-CDA: tài liệu tóm tắt

**`$summary`** ✅ `fhir/operations/patientsummary.ts`: dựng 1 `Composition` gồm các **mục** (section), mỗi mục có mã LOINC chuẩn (`:4-23`). Ví dụ:
- dị ứng (`LOINC_ALLERGIES_SECTION`)
- thuốc (`LOINC_MEDICATIONS_SECTION`)
- vấn đề sức khoẻ (`LOINC_PROBLEMS_SECTION`)
- kết quả (`LOINC_RESULTS_SECTION`)
- tiêm chủng (`LOINC_IMMUNIZATIONS_SECTION`)
- thủ thuật, kế hoạch điều trị, tiền sử xã hội, bảo hiểm…

📘 Đây là cấu trúc của **tài liệu tóm tắt bệnh nhân** (gần với International Patient Summary).

**C-CDA** ✅ `fhir/operations/ccdaexport.ts:48-61`: lấy bundle tóm tắt, chuyển sang C-CDA (`convertFhirToCcda` của package `packages/ccda`), xuất XML dạng `Binary`.

📘 C-CDA (Consolidated Clinical Document Architecture) là chuẩn **tài liệu XML** cũ hơn FHIR, vẫn bắt buộc cho chuyển tuyến ở Mỹ (`onc.md`, tiêu chí `b1` Transitions of Care).

🧠 Phân biệt **dữ liệu** và **tài liệu**:
- `$everything` trả **dữ liệu**: mọi resource, máy đọc.
- `$summary`/C-CDA trả **tài liệu**: chọn lọc, có cấu trúc mục, có người ký, người đọc được. Giấy chuyển viện là tài liệu, không phải dump dữ liệu.

## 6. SMART Health Cards và Links: bệnh nhân tự mang hồ sơ

| | SMART Health Card (SHC) | SMART Health Link (SHL) |
|---|---|---|
| Là gì 📘 | Dữ liệu nhỏ (VD tiêm chủng) **ký số**, nén, đặt trong QR | **Đường link** tới gói dữ liệu **mã hoá**, có thể kèm mật khẩu và hạn |
| Medplum ✅ | `fhir/operations/smarthealthcards.ts`: nén `deflateRaw` (`:209`), file `verifiableCredential` (`:297-300`), có endpoint xác minh `/$verify-smart-health-card` (`routes.ts:403`) | `fhir/operations/smarthealthlinks.ts`: `passcode` (`:74`, `:94`), cờ `U`/`P` (`:136-140`), mã hoá gói (`encryptSmartHealthLinkFile`, `:162`), lưu `Binary` kiểu `application/jose` (`:168`). Mở link qua `/shl/` (`app.ts:271`) |
| Dùng khi | Chứng nhận tiêm chủng, kết quả xét nghiệm ngắn | Gửi hồ sơ cho bác sĩ khác, du lịch, khám ở nơi khác |
| Ưu điểm | Xác minh offline được (chữ ký) | Không giới hạn kích thước, thu hồi được |

🧠 2 cách này đưa bệnh nhân lên làm **người vận chuyển dữ liệu**, thay vì 2 cơ sở y tế phải kết nối trực tiếp với nhau. Rất hợp với nơi chưa có mạng liên thông quốc gia.

## 7. Chứng nhận ONC của Medplum: góc nhìn kinh doanh

📘 `packages/docs/docs/compliance/onc.md:33-64` liệt kê tiêu chí Medplum đã chứng nhận, gồm:
- `a2` CPOE xét nghiệm, `a5` nhân khẩu học, `a14` thiết bị cấy ghép
- `b1` chuyển tuyến (C-CDA), `b10` xuất EHI, `b11` hỗ trợ quyết định
- `c1` đo lường chất lượng
- `d1`–`d13`: xác thực, audit (`d2`, `d3`), truy cập khẩn cấp (`d6`), MFA (`d13`)
- `g10` API chuẩn cho bệnh nhân và quần thể (US Core STU 5.0.1, `:25-29`)

🧠 Ý nghĩa với người làm chiến lược:
- Khách hàng Mỹ xây sản phẩm trên Medplum có thể **thừa hưởng** 1 phần chứng nhận thay vì tự làm. Đây là chi phí lớn và tốn thời gian. Là 1 phần **giá trị cốt lõi** mà nền tảng bán, không chỉ là phần mềm.
- Danh sách tiêu chí cũng là **bản đồ những gì 1 EHR ở Mỹ phải có**. Hữu ích để so khi đánh giá đối thủ.
- Cần xác minh trạng thái chứng nhận hiện hành trên danh sách CHPL của ONC trước khi dùng trong tư vấn.

## 8. Đối chiếu OpenEMR

✅ OpenEMR:
- Bulk export FHIR: `src/Services/FHIR/FhirExportJobService.php`.
- Xuất EHI (tiêu chí `b10`): module `interface/modules/custom_modules/oe-module-ehi-exporter/` (có `table.sql`, `src/`, `templates/`).
- C-CDA/CCR: `src/Cda/`, thư mục `ccr/`.
- Sự kiện khi nhập C-CDA từ bên ngoài: `src/Events/CDA/CDAPreParseEvent.php`, `CDAPostParseEvent.php`. Bộ chuyển dữ liệu nội bộ sang CDA: `src/Cda/InternalToCdaConverter.php`.

| | Medplum | OpenEMR |
|---|---|---|
| Dữ liệu gốc | FHIR, nên export gần như là đọc thẳng | Bảng quan hệ, phải dịch sang FHIR/C-CDA mỗi lần xuất |
| Xuất EHI | Dựa trên `$everything` | Module riêng xuất **bảng** dữ liệu gốc |
| Chia sẻ cho bệnh nhân | SHC, SHL, cổng bệnh nhân qua API | Cổng bệnh nhân (portal) PHP |

🧠 Cách OpenEMR xuất EHI bằng **bảng gốc** phản ánh đúng định nghĩa "toàn bộ dữ liệu" của luật: kể cả dữ liệu không có chỗ trong FHIR. Với Medplum, mọi thứ đã là FHIR nên `$everything` gần như đủ.

## 9. Hệ quả cho dự án Việt Nam

🧠 Cần kiểm chứng văn bản hiện hành:
- **Liên thông dữ liệu khám chữa bệnh với BHXH** đi theo chuẩn dữ liệu XML riêng, không theo FHIR. Bulk `$export` của Medplum không trực tiếp đáp ứng. Cần 1 lớp chuyển đổi (Bot hoặc job riêng) từ FHIR sang định dạng của cơ quan.
- **Sổ sức khoẻ điện tử / hồ sơ sức khoẻ cá nhân** quốc gia: nếu có API, dùng `$summary`/`$everything` làm nguồn rồi chuyển định dạng.
- SHL là cách nhẹ để bệnh nhân mang hồ sơ khi đi khám nơi khác, không cần chờ hạ tầng liên thông.

## Sai lầm thường gặp

1. **Dùng `$everything` để đồng bộ hằng đêm cho mọi bệnh nhân.** Mỗi lần 1 bệnh nhân, chậm và nặng. Dùng `$export` với `_since`.
2. **Gửi `$everything` làm "giấy chuyển viện".** Bên nhận cần tài liệu tóm tắt có cấu trúc (`$summary` hoặc C-CDA), không phải hàng nghìn resource.
3. **SHL không đặt mật khẩu và hạn** cho dữ liệu nhạy cảm. Link lộ là hồ sơ lộ.
4. **Xuất bulk không lọc `_type`** khi chỉ cần vài loại resource. Tốn tài nguyên và xuất thừa dữ liệu nhạy cảm.
