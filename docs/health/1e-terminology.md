# 1e — Terminology: lưu, tra cứu và ánh xạ mã y tế

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Trả lời câu: *"Hệ thống làm sao biết 'E11' là đái tháo đường type 2, 'E11.9' là con của 'E11', mã nội bộ 'DTD2' của phòng khám tương đương mã nào, và màn hình chọn chẩn đoán gợi ý những mã nào?"* So sánh kỹ thuật lưu cây mã: [`1e-comparison-terminology.md`](./1e-comparison-terminology.md).
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. Vì sao terminology là xương sống của dữ liệu y tế](#1-vì-sao-terminology-là-xương-sống-của-dữ-liệu-y-tế)
- [2. 3 resource: CodeSystem, ValueSet, ConceptMap](#2-3-resource-codesystem-valueset-conceptmap)
- [3. Bảng lưu trữ: Coding và Coding_Property](#3-bảng-lưu-trữ-coding-và-coding_property)
- [4. Các operation](#4-các-operation)
- [5. Cây mã: is-a và truy vấn đệ quy](#5-cây-mã-is-a-và-truy-vấn-đệ-quy)
- [6. Gợi ý mã khi gõ: $expand có filter](#6-gợi-ý-mã-khi-gõ-expand-có-filter)
- [7. Giới hạn đáng biết](#7-giới-hạn-đáng-biết)
- [8. Đối chiếu OpenEMR và HAPI](#8-đối-chiếu-openemr-và-hapi)
- [9. Hệ quả cho dự án Việt Nam](#9-hệ-quả-cho-dự-án-việt-nam)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vì sao terminology là xương sống của dữ liệu y tế

🧠 Mọi việc có giá trị với dữ liệu y tế đều cần mã chuẩn:

| Việc | Cần mã gì | Thiếu mã thì sao |
|---|---|---|
| Thanh toán bảo hiểm | ICD-10 (chẩn đoán), mã dịch vụ kỹ thuật | Hồ sơ bị từ chối |
| Cảnh báo lâm sàng | RxNorm/danh mục thuốc (tương tác thuốc), LOINC (ngưỡng xét nghiệm) | Không cảnh báo được |
| Báo cáo chất lượng, dịch tễ | ICD-10, SNOMED | Đếm sai |
| Liên thông | Mọi hệ mã chuẩn | Bên nhận không hiểu |

📘 Docs Medplum (`packages/docs/docs/terminology/common-terminologies.md:5-16`) liệt kê hệ mã chính theo lĩnh vực:

| Lĩnh vực | Hệ mã | URL `system` |
|---|---|---|
| Xét nghiệm, dấu hiệu sinh tồn | LOINC | `http://loinc.org` |
| Thuốc | RxNorm | `http://www.nlm.nih.gov/research/umls/rxnorm` |
| Thủ thuật (tính tiền) | CPT | `http://www.ama-assn.org/go/cpt` |
| Chẩn đoán | ICD-10-CM | `http://hl7.org/fhir/sid/icd-10-cm` |
| Vắc-xin | CVX | `http://hl7.org/fhir/sid/cvx` |
| Dị ứng, khái niệm lâm sàng | SNOMED CT | `http://snomed.info/sct` |

## 2. 3 resource: CodeSystem, ValueSet, ConceptMap

| Resource 📘 | Vai trò | Ví dụ 💭 |
|---|---|---|
| `CodeSystem` | **Từ điển**: danh sách mã, tên, thuộc tính, quan hệ cha-con | ICD-10 với khoảng 70.000 mã |
| `ValueSet` | **Danh sách được phép** cho 1 trường, định nghĩa bằng liệt kê hoặc bộ lọc | "Chẩn đoán đái tháo đường" = mọi mã con của `E11` |
| `ConceptMap` | **Từ điển dịch** giữa 2 hệ mã | Mã nội bộ `DTD2` → ICD-10 `E11.9` |

📘 Thuộc tính quan trọng của CodeSystem:
- `hierarchyMeaning = is-a`: mã con là "1 loại" của mã cha.
- `content` có giá trị `complete`, `fragment` hoặc `example`. Nó cho biết đã nhập đủ hay chỉ 1 phần (docs `filtering-large-code-systems.md:51-56`).

## 3. Bảng lưu trữ: Coding và Coding_Property

✅ Bảng sinh trong `migrations/migrate.ts:733-881`:

| Bảng | Cột | Vai trò |
|---|---|---|
| `Coding` (`:733-782`) | `id`, `system` (UUID của resource CodeSystem), `code`, `display`, `isSynonym`, `synonymOf`, `language` | **1 dòng / mã**. Tên đồng nghĩa và bản dịch là dòng riêng trỏ `synonymOf` về dòng gốc |
| `Coding_Property` (`:784-810`) | `coding`, `property`, `target`, `value` | **EAV cho thuộc tính của mã**. Quan hệ cha-con cũng là 1 thuộc tính: `property` = parent, `target` = id mã cha |
| `CodeSystem_Property` (`:812-825`) | `system`, `code`, `type`, `uri`, `description` | Định nghĩa các thuộc tính mà 1 CodeSystem có |
| `CodingSystem` (`:857-866`) | `id`, `system` (URL) | Bảng tra URL → id số, để `ConceptMapping` lưu gọn |
| `ConceptMapping` (`:827-855`) | `conceptMap`, `sourceSystem`, `sourceCode`, `targetSystem`, `targetCode`, `relationship` | 1 dòng / 1 cặp ánh xạ. Index 2 chiều (`map_forward_idx`, `map_reverse_idx`) |
| `ConceptMapping_Attribute` (`:868-881`) | `mapping`, `uri`, `type`, `value`, `kind` | Thuộc tính thêm của ánh xạ |

**Index đáng chú ý trên `Coding`** (`:745-780`):
- Unique `(system, code)` chỉ cho dòng gốc (`WHERE "synonymOf" IS NULL`).
- **GIN trigram trên `medplum_unaccent(display)`**: tìm "systeme" khớp "Système" (comment gốc ở `:765`).
- B-tree `lower(code) text_pattern_ops`: gõ "e11" gợi ý "E11.9".

🧠 Chú ý: ở 1a, Medplum **tránh** EAV cho dữ liệu lâm sàng. Ở đây lại **dùng** EAV (`Coding_Property`) cho thuộc tính mã. Lý do: thuộc tính của mã khác nhau hoàn toàn giữa các hệ (RxNorm có `TTY`, SNOMED dựa vào cây, LOINC có `COMPONENT`, `SYSTEM`…). Mã cũng không cần so sánh khoảng như kết quả xét nghiệm. Đây đúng là điều kiện mà EAV phù hợp (bài Saleor, Phần 10).

**Chỉ CodeSystem có `content` = `complete`, `example` hoặc `fragment` mới được nạp vào `Coding`** ✅ `fhir/lookups/coding.ts:52-53`.

## 4. Các operation

✅ `fhir/routes.ts:269-307`:

| Operation | Câu hỏi nghiệp vụ | File |
|---|---|---|
| `CodeSystem/$lookup` | "Mã E11.9 tên gì, thuộc tính gì?" | `operations/codesystemlookup.ts` |
| `CodeSystem/$validate-code` | "Mã này có tồn tại trong hệ ICD-10 không?" | `operations/codesystemvalidatecode.ts` |
| `CodeSystem/$subsumes` | "E11.9 có phải 1 loại của E11 không?" | `operations/subsumes.ts` |
| `ValueSet/$expand` | "Liệt kê mã hợp lệ cho ô chẩn đoán, lọc theo chữ người dùng gõ" | `operations/expand.ts` |
| `ValueSet/$validate-code` | "Mã người dùng chọn có thuộc danh sách cho phép không?" | `operations/valuesetvalidatecode.ts` |
| `ConceptMap/$translate` | "Mã nội bộ DTD2 tương ứng mã ICD-10 nào?" | `operations/conceptmaptranslate.ts` |
| `CodeSystem/$import`, `ConceptMap/$import` | Nạp hàng loạt mã/ánh xạ | `operations/codesystemimport.ts`, `conceptmapimport.ts` |

**Ai được nạp mã** ✅ `operations/codesystemimport.ts:95-110`: chỉ **admin project** hoặc **super admin**. Admin project chỉ nạp vào CodeSystem của project mình (`ownProjectOnly`). 🧠 Bảng mã chuẩn (ICD-10, LOINC) thường do super admin nạp 1 lần, các project dùng chung qua `Project.link` ([1c mục 2](./1c-access-control.md#2-tầng-1--project-tenant-cứng)).

## 5. Cây mã: is-a và truy vấn đệ quy

📘 Docs (`filtering-large-code-systems.md:17-27`): SNOMED có cây *Clinical finding* → *Diabetes mellitus* → *Type 2 diabetes mellitus*. Chọn mã cha nghĩa là bao gồm mọi mã con. **1 bộ lọc thay cho hàng nghìn mã.** LOINC thì gần như không có cây, phải lọc theo thuộc tính.

✅ Cách Medplum duyệt cây, `operations/utils/terminology.ts`:
- `parentProperty = 'http://hl7.org/fhir/concept-properties#parent'` (`:27`).
- `getParentProperty` (`:227-236`): chỉ chấp nhận CodeSystem có `hierarchyMeaning = 'is-a'`, nếu không thì lỗi *"does not have an is-a hierarchy"*.
- `findAncestor` (`:190-225`): `WITH RECURSIVE cte_ancestors` đi **lên** qua `Coding_Property.target`. Dùng cho `$subsumes`.
- `addDescendants` (`:265-312`): `WITH RECURSIVE cte_descendants` đi **xuống**. Dùng cho filter `is-a` / `descendent-of` trong `$expand` (`operations/expand.ts:418-419`).

Đây là mô hình **danh sách kề (adjacency list) + CTE đệ quy**: mỗi mã chỉ lưu cha trực tiếp, cây được duyệt lúc truy vấn. So sánh với cách khác ở [`1e-comparison`](./1e-comparison-terminology.md).

## 6. Gợi ý mã khi gõ: $expand có filter

🧠 Đây là tính năng người dùng chạm vào nhiều nhất: ô "chẩn đoán" gõ "đái tháo" thì gợi ý mã.

✅ `operations/expand.ts`:
- Giới hạn **1000 mã** mỗi lần (`MAX_EXPANSION_SIZE`, `:117`). Vượt thì trả 1000 mã đầu và `total = 1001` (`:160-164`).
- Tối đa 10 từ trong chuỗi lọc (`MAX_FILTER_TOKENS`, `:49`).
- Lọc theo mã: `LOWER(code) LIKE 'abc%'` (`:527`).
- Lọc theo tên: **mỗi từ** phải xuất hiện, so khớp **không dấu** (`UNACCENT_ILIKE`, `:535-545`). Thử `ILIKE` thường trước vì rẻ hơn (`:544`).
- Xếp hạng theo độ giống `strict_word_similarity` trên chuỗi đã bỏ dấu (`:725`).
- Bản dịch: dòng `Coding` có `language` khác được trả thành `designation` (`:343-354`). 📘 Docs `medplum-terminology-services.mdx:40-67` hướng dẫn nạp và tìm theo bản dịch.

## 7. Giới hạn đáng biết

| Giới hạn | Bằng chứng | Cách làm thay 🧠 |
|---|---|---|
| **Search resource theo ValueSet (`code:in=`) và theo cây (`code:below=`) không được hỗ trợ** | `fhir/token-column.ts:240-255` ném lỗi `invalidSearchOperator` | `$expand` lấy danh sách mã rồi search `code=a,b,c`. Hoặc lưu thêm mã cha vào `Condition.category` khi ghi |
| `$expand` tối đa 1000 mã | `expand.ts:117` | Lọc hẹp hơn, phân trang (`offset`, `count`) |
| Cây chỉ cho `is-a` | `terminology.ts:227-236` | Hệ không có cây (LOINC): lọc theo thuộc tính |
| Nạp bảng mã cần quyền admin | `codesystemimport.ts:95-110` | Super admin nạp 1 lần, chia sẻ qua `Project.link` |

## 8. Đối chiếu OpenEMR và HAPI

**OpenEMR** ✅ `sql/database.sql`:
- Bảng `codes` (`:1125`): `code`, `code_type`, `code_text`, **`fee`**, `superbill`, `related_code`, `revenue_code`, `financial_reporting`…
- Bảng riêng cho từng hệ ICD-10: `icd10_dx_order_code` (`:3094`), `icd10_pcs_order_code` (`:3114`).

🧠 Ở OpenEMR, **mã dịch vụ và giá tiền nằm cùng 1 dòng**. Đó là góc nhìn của phần mềm phòng khám: mã tồn tại để **tính tiền**. Medplum tách hẳn: mã nằm ở `CodeSystem`, giá nằm ở `ChargeItemDefinition` (module 4b). Tách thì 1 mã có nhiều mức giá theo payer. Gộp thì đơn giản cho phòng khám 1 bảng giá.

**HAPI FHIR** ✅ `hapi-fhir-jpaserver-base/.../jpa/entity/`:
- `TRM_CONCEPT` (`TermConcept.java:102`).
- `TRM_CONCEPT_PC_LINK`: cha-con (`TermConceptParentChildLink.java:52-58`), giống `Coding_Property` của Medplum.
- **Thêm** cột `PARENT_PIDS_VC`, danh sách **mọi tổ tiên** của mã, đánh index full-text (`TermConcept.java:226-233`), tính sẵn lúc nạp (`:529-551`).
- **Mở rộng ValueSet tính trước** (pre-expansion): bảng `TRM_VALUESET_CONCEPT` (`TermValueSetConcept.java:66`), trạng thái `EXPANSION_STATUS` (`TermValueSet.java:146`).

🧠 HAPI trả giá lúc **nạp** (tính tổ tiên, mở rộng trước) để **đọc** nhanh. Medplum tính lúc **đọc** (CTE đệ quy) để nạp đơn giản. Chi tiết ở [`1e-comparison`](./1e-comparison-terminology.md).

## 9. Hệ quả cho dự án Việt Nam

🧠 Cần kiểm chứng trên môi trường thật trước khi dựa vào:

1. **Danh mục dùng chung của Việt Nam** (thuốc, dịch vụ kỹ thuật, vật tư theo quy định BHYT) không có sẵn. Phải tạo `CodeSystem` riêng và nạp bằng `$import`. Ánh xạ mã nội bộ của từng phòng khám sang danh mục chuẩn bằng `ConceptMap`.
2. **ICD-10 bản tiếng Việt**: nạp như bản dịch (`Coding.language = vi`). `$expand` tìm không dấu được, vì bộ lọc SQL dùng `medplum_unaccent` dựa trên extension `unaccent` của Postgres.
3. **Chữ "Đ"**: ký tự `Đ`/`đ` **không có dạng tách dấu** trong Unicode (kiểm bằng `unicodedata.decomposition('Đ')` ra chuỗi rỗng). Hệ quả:
   - Hàm `foldText` phía JS của Medplum (`util/text.ts:14-16`, NFD rồi bỏ dấu) **không** đổi "Đái" thành "dai". Docs trong code tự ghi chú hàm này không tương đương `medplum_unaccent` (`util/text.ts:9-10`).
   - Bộ chuẩn hoá của HAPI (`StringUtil.java:42-70`) cũng chỉ bỏ ký tự U+0300–036F, nên "Đặng" thành "ĐANG", không thành "DANG".
   - Phía SQL (`unaccent` của Postgres) theo hiểu biết của mình có quy tắc `Đ → D`. **Cần test.**

## Sai lầm thường gặp

1. **Lưu chẩn đoán chỉ bằng chữ** (`Condition.code.text = "Tiểu đường"`) không có `coding`. Không thanh toán, không thống kê được.
2. **Dùng 1 ValueSet "toàn bộ ICD-10" cho ô chẩn đoán.** 70.000 mã, gồm cả mã nhóm không được dùng để thanh toán (docs `filtering-large-code-systems.md:28-50`: ICD-10-CM có mã nhóm `Z01` trên mã lá `Z01.411`). Phải lọc theo cây hoặc thuộc tính.
3. **Tự đặt URL `system` lung tung cho mã nội bộ.** Cùng 1 hệ nội bộ mà 2 URL khác nhau thì thành 2 hệ, không gộp được. Chốt 1 URL ổn định.
4. **Kỳ vọng search `Condition?code:below=E11`.** Tại commit này không hỗ trợ (mục 7).
5. **Gộp giá vào bảng mã** như OpenEMR khi bán cho nhiều payer. Mỗi payer 1 giá thì phải tách (module 4b).

## Tự kiểm tra

1. Phòng khám dùng mã nội bộ `KHAM-01` cho "Khám nội tổng quát". Cần những resource gì để báo cáo BHYT theo mã chuẩn?
2. Màn hình chọn chẩn đoán chỉ nên hiện mã đái tháo đường. Viết `ValueSet.compose` dùng filter `is-a` trên ICD-10.
3. Vì sao Medplum dùng EAV cho `Coding_Property` nhưng tránh EAV cho `Observation`?
4. Muốn lấy mọi bệnh nhân có chẩn đoán thuộc nhóm E11. Viết 2 bước gọi API.
