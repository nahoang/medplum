# 0a — FHIR primer: 10 khái niệm phải có trước khi đọc code Medplum

> **Loại file:** R (phần lớn 📘, có neo ✅ vào Medplum). **Commit Medplum:** `16cb130`.
>
> Trả lời câu: *"Dữ liệu y tế theo FHIR có hình dạng gì?"* Giống 1 cuốn từ điển rút gọn. Mỗi khái niệm có 3 phần: **nó là gì** (📘), **vì sao y tế cần nó** (🧠, góc nghiệp vụ), **Medplum xử lý nó ở đâu** (✅). Không học hết 150 loại resource ở đây. Resource cụ thể học ở module dùng đến nó (xem [`0b`](./0b-ban-do-nghiep-vu-resource.md)).

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Kịch bản xuyên suốt](#kịch-bản-xuyên-suốt)
- [1. Resource](#1-resource)
- [2. Element: kiểu nguyên thuỷ và kiểu phức](#2-element-kiểu-nguyên-thuỷ-và-kiểu-phức)
- [3. Reference](#3-reference)
- [4. CodeableConcept và Coding](#4-codeableconcept-và-coding)
- [5. Identifier](#5-identifier)
- [6. Extension và Profile](#6-extension-và-profile)
- [7. Search parameter](#7-search-parameter)
- [8. Compartment](#8-compartment)
- [9. Bundle: batch và transaction](#9-bundle-batch-và-transaction)
- [10. Terminology: CodeSystem, ValueSet, ConceptMap](#10-terminology-codesystem-valueset-conceptmap)
- [Phần riêng của Medplum: meta mở rộng và resource tự định nghĩa](#phần-riêng-của-medplum-meta-mở-rộng-và-resource-tự-định-nghĩa)
- [Sai lầm thường gặp khi mới học FHIR](#sai-lầm-thường-gặp-khi-mới-học-fhir)
- [Tự kiểm tra](#tự-kiểm-tra)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa |
|---|---|
| ✅ grounded | Code Medplum, có `file:line` |
| 📘 spec | FHIR R4 (hl7.org/fhir/R4) hoặc docs `packages/docs/docs/...` |
| 💭 mô phỏng | Ví dụ tự dựng |
| 🧠 suy luận | Lập luận |

## Kịch bản xuyên suốt

💭 Bà **Nguyễn Thị Lan**, 68 tuổi, tiểu đường type 2, khám tại **Phòng khám An Bình**. Hôm nay bác sĩ **Trần Văn Minh** đo huyết áp 150/95 và chỉ định xét nghiệm HbA1c. Mọi ví dụ trong file dùng đúng câu chuyện này.

---

## 1. Resource

**Là gì 📘:** đơn vị dữ liệu cơ bản của FHIR, giống 1 "object". FHIR R4 định nghĩa khoảng 150 loại (`Patient`, `Observation`, `Encounter`, `Claim`…). Mỗi resource có `resourceType`, `id` (server cấp), và `meta` (phiên bản, thời điểm sửa cuối).

**Vì sao y tế cần 🧠:** y tế có quá nhiều tổ chức (bệnh viện, phòng khám, lab, bảo hiểm, nhà thuốc) dùng phần mềm khác nhau. Không có 1 bộ "object" chung thì mỗi cặp hệ thống phải tự thoả thuận định dạng. Với N hệ thống là N×(N−1)/2 cặp. Resource là bộ từ vựng chung để giảm số thoả thuận đó.

**Ví dụ 💭:**

```json
{
  "resourceType": "Patient",
  "id": "8f2c...",
  "meta": { "versionId": "a1b2...", "lastUpdated": "2026-10-14T08:00:00Z" },
  "name": [{ "family": "Nguyễn", "given": ["Thị", "Lan"] }],
  "gender": "female",
  "birthDate": "1958-03-12"
}
```

**Medplum ✅:** mỗi loại resource là 1 bảng Postgres riêng, toàn bộ JSON nằm trong cột `content TEXT` (`packages/server/src/migrations/migrate.ts:247-257`). Chi tiết ở [`1a`](./1a-fhir-datastore.md).

## 2. Element: kiểu nguyên thuỷ và kiểu phức

**Là gì 📘:** mỗi trường trong resource là 1 element. Element có thể là kiểu nguyên thuỷ (`string`, `boolean`, `date`, `decimal`, `code`…) hoặc kiểu phức dùng lại nhiều nơi (`HumanName`, `Address`, `ContactPoint`, `Period`, `Quantity`…).

**Vì sao 🧠:** tên người Việt (họ + tên đệm + tên), địa chỉ (phường/quận/tỉnh) có cấu trúc. Lưu thành 1 chuỗi tự do thì không tìm theo họ được, không thống kê theo tỉnh được.

**Điểm cần nhớ:** rất nhiều element là **mảng**. 1 bệnh nhân có nhiều tên (tên khai sinh, tên sau kết hôn), nhiều địa chỉ, nhiều số điện thoại. Chính "mảng có cấu trúc" là thứ làm việc lưu trữ khó. Xem [`1a`](./1a-fhir-datastore.md), đặc tính P3.

**`Quantity`** đáng nhớ riêng: giá trị số **luôn đi kèm đơn vị**, theo chuẩn UCUM.

```json
{ "value": 150, "unit": "mmHg", "system": "http://unitsofmeasure.org", "code": "mm[Hg]" }
```

Trong y tế, số không có đơn vị là số vô nghĩa và nguy hiểm: 5 mg khác 5 g.

## 3. Reference

**Là gì 📘:** con trỏ từ resource này sang resource khác, dạng `"ResourceType/id"`. Tương đương khoá ngoại.

```json
{
  "resourceType": "Observation",
  "subject":   { "reference": "Patient/8f2c...",      "display": "Nguyễn Thị Lan" },
  "performer": [{ "reference": "Practitioner/77ab...", "display": "BS Trần Văn Minh" }],
  "encounter": { "reference": "Encounter/c3d4..." }
}
```

**Vì sao 🧠:** hồ sơ y tế không phải 1 file lớn. Nó là **mạng lưới** các sự kiện nhỏ trỏ về nhau: lượt khám → các chỉ số đo → chẩn đoán → đơn thuốc → hoá đơn. Câu hỏi lâm sàng luôn đi theo mạng này. VD: "các kết quả HbA1c của bệnh nhân này trong những lượt khám với BS Minh".

**Medplum ✅:** reference được index vào bảng `<ResourceType>_References` (`migrate.ts:308-317`) để phục vụ search lồng (chained search, `packages/server/src/fhir/search.ts:1689`).

## 4. CodeableConcept và Coding

**Là gì 📘:** cách FHIR ghi 1 khái niệm y khoa bằng **mã chuẩn**. `Coding` = 1 cặp `(system, code)`. `CodeableConcept` = text hiển thị + mảng nhiều `Coding`, vì cùng 1 khái niệm có thể có mã trong nhiều hệ.

```json
"code": {
  "text": "Huyết áp tâm thu",
  "coding": [
    { "system": "http://loinc.org", "code": "8480-6", "display": "Systolic blood pressure" }
  ]
}
```

**Vì sao 🧠:** đây là **khái niệm quan trọng nhất** của dữ liệu y tế. Chữ "tiểu đường", "ĐTĐ type 2", "diabetes mellitus type 2" là 3 chuỗi khác nhau nhưng cùng 1 bệnh. Máy chỉ gộp, thống kê, thanh toán bảo hiểm được khi cùng trỏ về 1 mã (VD ICD-10 `E11`). So với Saleor: đây chính là bài toán P2 "dedup giá trị". Có điều y tế không tự dựng bảng giá trị mà dùng **bảng mã quốc tế có sẵn**.

**Các hệ mã thường gặp 📘** (`packages/docs/docs/terminology/`):

| Hệ mã | Dùng cho | Ghi chú |
|---|---|---|
| ICD-10 | Chẩn đoán | Việt Nam dùng ICD-10 cho BHYT |
| SNOMED CT | Khái niệm lâm sàng tổng quát | Có phí bản quyền theo quốc gia |
| LOINC | Xét nghiệm, chỉ số đo | Miễn phí |
| RxNorm, NDC | Thuốc (Mỹ) | Việt Nam có danh mục thuốc riêng của Bộ Y tế |
| CPT, HCPCS | Dịch vụ y tế để tính tiền (Mỹ) | CPT thuộc bản quyền AMA |

**Medplum ✅:** search theo `system|code` đi qua chiến lược `token-column`, mã hoá băm vào cột `UUID[]` (`packages/server/src/fhir/token-column.ts:27-124`).

## 5. Identifier

**Là gì 📘:** mã định danh nghiệp vụ, khác với `id` kỹ thuật. Dạng `(system, value)`. 1 resource có thể có nhiều identifier.

```json
"identifier": [
  { "system": "https://cccd.gov.vn",           "value": "001058012345" },
  { "system": "https://bhyt.gov.vn",           "value": "HT2790112345678" },
  { "system": "https://phongkham-anbinh.vn/mrn", "value": "AB-000123" }
]
```

💭 `system` URL ở trên là tự đặt cho ví dụ, không phải URL chính thức.

**Vì sao 🧠:** cùng 1 người có số CCCD, số thẻ BHYT, mã bệnh nhân ở từng bệnh viện. Ghép đúng người (patient matching) là bài toán sống còn: ghép sai là xem nhầm hồ sơ người khác. 📘 docs `packages/docs/docs/fhir-basics.md` (mục Identifiers) có ví dụ tương tự cho Mỹ (SSN, MRN).

## 6. Extension và Profile

**Extension 📘:** cách thêm trường **không có trong chuẩn** mà không phá chuẩn. Mỗi extension có `url` định danh.

```json
"extension": [{ "url": "https://phongkham-anbinh.vn/fhir/dan-toc", "valueString": "Kinh" }]
```

**Profile 📘:** bộ ràng buộc thêm lên resource gốc cho 1 bối cảnh. VD US Core bắt `Patient` phải có `identifier`, `name`, `gender`. Mỗi quốc gia thường có bộ profile riêng.

**Vì sao 🧠:** FHIR cố tình làm resource gốc "lỏng" (hầu hết trường là tuỳ chọn) để dùng được ở mọi nước. Profile là nơi từng nước/tổ chức "siết" lại. 🧠 Với dự án ở Việt Nam: khi Bộ Y tế ban hành profile chính thức thì dự án phải tuân theo. Cần kiểm tra hiện trạng trước khi thiết kế.

**Medplum ✅📘:** validate theo profile khai báo trong `meta.profile`. Cột `_profile TEXT[]` có index GIN trên mọi bảng (`migrate.ts:258,265`). Docs: `packages/docs/docs/fhir-datastore/profiles.md`.

## 7. Search parameter

**Là gì 📘:** FHIR **không cho tìm theo trường bất kỳ**. Mỗi resource có danh sách search parameter được định nghĩa sẵn (VD `Patient?name=`, `Observation?code=&date=ge2026-01-01`). Mỗi parameter có kiểu: `string`, `token`, `reference`, `date`, `number`, `quantity`, `uri`, `composite`, `special`.

**Vì sao 🧠:** đổi tự do lấy hiệu năng dự đoán được. Server biết trước sẽ bị hỏi theo trường nào nên index trước được.

**Medplum ✅:** đây là **trục chính** quyết định cách lưu trữ. Mỗi kiểu parameter được gán 1 chiến lược lưu khác nhau (`packages/server/src/fhir/searchparameter.ts:116-182`). File [`1a`](./1a-fhir-datastore.md) dành trọn cho chuyện này.

## 8. Compartment

**Là gì 📘:** nhóm logic các resource "thuộc về" 1 thực thể. Quan trọng nhất là **Patient compartment**: mọi `Observation`, `Encounter`, `MedicationRequest`… có tham chiếu tới bệnh nhân X đều nằm trong compartment của X. Danh sách trường nào xác định "thuộc về" do spec quy định (CompartmentDefinition).

**Vì sao 🧠:** đây là nền của **phân quyền theo bệnh nhân**. "Bệnh nhân chỉ xem được hồ sơ của mình", "bác sĩ chỉ xem bệnh nhân mình phụ trách": cả 2 quy về "được truy cập compartment nào".

**Medplum ✅:**
- `packages/server/src/fhir/patient.ts:56-76` (`getPatients`) tìm bệnh nhân của 1 resource theo CompartmentDefinition.
- `repo.ts:2061-2101` (`getCompartments`) ghi danh sách compartment vào `meta.compartment`.
- Cột `compartments UUID[]` có index GIN (`migrate.ts:277-280`) để lọc nhanh.

## 9. Bundle: batch và transaction

**Là gì 📘:** gói nhiều resource. Có các loại `searchset` (kết quả tìm kiếm), `batch` (nhiều request độc lập), `transaction` (nhiều request **cùng thành công hoặc cùng thất bại**), `history`, `document`.

**Vì sao 🧠:** 1 lượt khám tạo ra cùng lúc Encounter + nhiều Observation + Condition + MedicationRequest. Lưu nửa chừng (có chỉ số nhưng mất chẩn đoán) là hồ sơ sai. Transaction đảm bảo tất cả hoặc không gì cả.

**Medplum ✅:** `packages/fhir-router` (`batch`, `transaction`), `packages/server/src/fhir/batch/`. Docs: `packages/docs/docs/fhir-datastore/fhir-batch-requests.mdx`.

## 10. Terminology: CodeSystem, ValueSet, ConceptMap

**Là gì 📘:**
- **CodeSystem**: bảng mã (VD toàn bộ LOINC).
- **ValueSet**: tập con mã hợp lệ cho 1 trường (VD "các mã LOINC dành cho dấu hiệu sinh tồn").
- **ConceptMap**: bảng ánh xạ giữa 2 hệ mã (VD mã nội bộ phòng khám → ICD-10).

**Vì sao 🧠:** dữ liệu cũ của phòng khám thường dùng mã tự đặt. Muốn liên thông hay thanh toán BHYT phải ánh xạ sang mã chuẩn. ConceptMap là "từ điển dịch" đó.

**Medplum ✅:** operation `$expand`, `$lookup`, `$validate-code`, `$subsumes`, `$translate` (`packages/server/src/fhir/routes.ts:269-307`). Bảng `Coding` (`fhir/lookups/coding.ts:20-37`) và `ConceptMapping` lưu nội dung bảng mã.

---

## Phần riêng của Medplum: meta mở rộng và resource tự định nghĩa

FHIR chuẩn không đủ cho 1 nền tảng SaaS nhiều tenant. Medplum thêm 2 thứ:

**1. Trường mở rộng trong `meta` ✅** (`repo.ts:1040-1060`):

| Trường | Ý nghĩa |
|---|---|
| `meta.project` | Resource thuộc tenant nào |
| `meta.author` | Ai ghi phiên bản này |
| `meta.onBehalfOf` | Ghi thay mặt ai (VD trợ lý nhập thay bác sĩ) |
| `meta.account` / `meta.accounts` | Thuộc nhóm truy cập nào (dùng cho phân quyền theo tổ chức con) |
| `meta.compartment` | Danh sách compartment đã tính (xem mục 8) |

**2. Resource tự định nghĩa ✅** (`packages/definitions/src/fhir/r4/profiles-medplum.json`):

`AccessPolicy`, `Agent`, `AsyncJob`, `Bot`, `BulkDataExport`, `ClientApplication`, `Cron`, `DicomInstance`, `DicomSeries`, `DicomStudy`, `DomainConfiguration`, `Enterprise`, `JsonWebKey`, `Login`, `Package`, `PackageInstallation`, `PackageRelease`, `Project`, `ProjectMembership`, `SmartAppLaunch`, `SmartHealthLink`, `User`, `UserConfiguration`, `UserSecurityRequest`

🧠 Quan sát: toàn bộ danh sách là **hạ tầng nền tảng** (tenant, người dùng, auth, bot, DICOM). Không có resource nghiệp vụ lâm sàng nào. Medplum giữ đúng nguyên tắc **"nghiệp vụ y tế để FHIR định nghĩa, Medplum chỉ thêm phần vận hành nền tảng"**. Đây là bằng chứng cho khác biệt K1 trong [kế hoạch](../../docs/domain/medplum-deep-dive-plan.md#1-khác-biệt-cốt-lõi-khi-chuyển-từ-saleor-sang-medplum).

---

## Sai lầm thường gặp khi mới học FHIR

1. **Coi `id` là mã bệnh nhân.** `id` là khoá kỹ thuật do server cấp, đổi khi chuyển server. Mã nghiệp vụ nằm ở `identifier`. Dấu hiệu sai: in `Patient.id` lên phiếu khám.
2. **Lưu số không kèm đơn vị.** Dùng `valueString: "150/95"` thay vì 2 `Quantity` có đơn vị. Hậu quả: không vẽ biểu đồ được, không cảnh báo tăng huyết áp tự động được. OpenEMR có đúng vết này: cột `bps`/`bpd` là `varchar(40)` (`/home/user/openemr/openemr/sql/database.sql:2431-2432`).
3. **Chỉ ghi `text`, bỏ `coding`.** "Tiểu đường" không có mã ICD-10 thì không thanh toán BHYT được, không thống kê được.
4. **Tạo trường mới bằng cách sửa resource gốc.** Phải dùng `extension`, không thì phá khả năng liên thông.
5. **Nghĩ FHIR là 1 database.** FHIR là **chuẩn trao đổi và mô hình dữ liệu**. Lưu nó xuống đĩa thế nào là việc của từng server. Medplum, HAPI, OpenEMR mỗi bên làm 1 kiểu.

## Tự kiểm tra

Viết ra giấy trước, rồi đối chiếu lại các mục trên:

1. Kết quả HbA1c 8.2% của bà Lan: resource nào? Trường `code` dùng hệ mã nào? Trường `value` kiểu gì?
2. Bà Lan đổi số thẻ BHYT sang năm mới. Sửa trường nào? Có xoá số cũ không? (Gợi ý: `Identifier.period`)
3. Bác sĩ chỉ được xem bệnh nhân của phòng khám An Bình. Đây là bài toán compartment, project, hay cả hai?
4. Lượt khám tạo 1 Encounter và 3 Observation. Gửi bằng Bundle loại gì để không bao giờ lưu nửa chừng?
