# 1a — FHIR datastore: Medplum lưu 1 tài liệu y tế xuống Postgres thế nào

> **Loại file:** R (đọc code thật, tương đương file `1b` "Saleor làm gì" trong template). **Commit Medplum:** `16cb130`. **Commit OpenEMR đối chứng:** `4c1eb8d`.
>
> Trả lời câu: *"Khi lưu 1 Patient hay 1 Observation, dữ liệu nằm ở những bảng nào, cột nào, kiểu gì, và vì sao?"* File này chỉ mô tả **thiết kế thật** kèm `file:line`. Phần so sánh 5 ứng viên ngang hàng theo Template C để dành cho file `1a-comparison-fhir-datastore.md` (chưa viết). File đó sẽ dùng file này làm nguồn ✅.
>
> Đọc trước: [`0a`](./0a-fhir-primer.md) mục 7 (search parameter) và mục 8 (compartment).

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Kịch bản xuyên suốt](#kịch-bản-xuyên-suốt)
- [1. Bức tranh: 1 resource được ghi xuống bao nhiêu bảng](#1-bức-tranh-1-resource-được-ghi-xuống-bao-nhiêu-bảng)
- [2. Bảng resource: phần cố định](#2-bảng-resource-phần-cố-định)
- [3. Phần sinh tự động: 4 chiến lược index search parameter](#3-phần-sinh-tự-động-4-chiến-lược-index-search-parameter)
  - [3.1 range-column: ngày, số, số lượng có đơn vị](#31-range-column-ngày-số-số-lượng-có-đơn-vị)
  - [3.2 token-column: mã chuẩn, định danh, liên lạc](#32-token-column-mã-chuẩn-định-danh-liên-lạc)
  - [3.3 lookup-table: mảng có cấu trúc](#33-lookup-table-mảng-có-cấu-trúc)
  - [3.4 column: phần còn lại](#34-column-phần-còn-lại)
- [4. Dữ liệu thật trông thế nào](#4-dữ-liệu-thật-trông-thế-nào)
- [5. Ghi, sửa, xoá: lịch sử không bao giờ mất](#5-ghi-sửa-xoá-lịch-sử-không-bao-giờ-mất)
- [6. Con đường tiến hoá: Medplum cũng từng lưu số dạng chuỗi](#6-con-đường-tiến-hoá-medplum-cũng-từng-lưu-số-dạng-chuỗi)
- [7. Multi-tenant ngay trong schema](#7-multi-tenant-ngay-trong-schema)
- [8. Đối chiếu OpenEMR: cùng bà Lan, schema khác hẳn](#8-đối-chiếu-openemr-cùng-bà-lan-schema-khác-hẳn)
- [9. Đặc tính bài toán rút ra (nháp cho file so sánh)](#9-đặc-tính-bài-toán-rút-ra-nháp-cho-file-so-sánh)
- [10. Câu hỏi còn mở](#10-câu-hỏi-còn-mở)
- [Sai lầm thường gặp khi đọc phần này](#sai-lầm-thường-gặp-khi-đọc-phần-này)
- [Tự kiểm tra](#tự-kiểm-tra)
- [Nguồn](#nguồn)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa | Dùng ở đâu |
|---|---|---|
| ✅ grounded | Code Medplum/OpenEMR, có `file:line` | Mục 1–8 |
| 📘 spec | FHIR R4, docs chính thức | Rải rác |
| 💭 minh hoạ | Giá trị dữ liệu tự dựng. **Tên cột/bảng vẫn là thật** | Mục 4, 8 |
| 🧠 suy luận | Lập luận "vì sao", Medplum không tự công bố | Mục 3, 6, 9 |

Đường dẫn Medplum viết tắt từ `packages/server/src/`. Đường dẫn OpenEMR tính từ `/home/user/openemr/openemr/`.

## Kịch bản xuyên suốt

💭 Giống [`0a`](./0a-fhir-primer.md): bà **Nguyễn Thị Lan**, có 2 tên (tên khai sinh, tên thường gọi "Lan Nguyễn"), 1 số CCCD, 1 số điện thoại. Bác sĩ đo **huyết áp tâm thu 150 mmHg** (LOINC `8480-6`). Mọi bảng dữ liệu trong file dùng đúng 2 resource này.

---

## 1. Bức tranh: 1 resource được ghi xuống bao nhiêu bảng

✅ `fhir/repo.ts:1197-1206` (`writeToDatabase`): 3 bước ghi chạy **trong cùng 1 transaction**:

```
PUT Patient/8f2c (bà Lan)
   │
   ├─ writeResource          → "Patient"            1 dòng: JSON đầy đủ + cột search phi chuẩn hoá
   ├─ writeResourceVersion   → "Patient_History"    1 dòng mới cho mỗi lần ghi
   └─ writeLookupTables      → "HumanName"          2 dòng (2 tên)
                             → "Address"            n dòng (nếu có địa chỉ)
                             → "Patient_References" 1 dòng / reference (VD generalPractitioner)
```

Danh sách lookup table cố định ở `fhir/searchparameter.ts:205-211`: `AddressTable`, `HumanNameTable`, `ReferenceTable`, `CodingTable`, `ConceptMappingTable`.

🧠 Đọc nhanh thiết kế: **1 bản gốc + nhiều bản chiếu**. Bản gốc là JSON trong cột `content`. Mọi thứ khác (cột search, lookup table) là **bản chiếu suy ra từ bản gốc** để phục vụ tìm kiếm. Hệ quả của nguyên tắc này xuất hiện lại ở mục 5 và 6.

## 2. Bảng resource: phần cố định

✅ `migrations/migrate.ts:247-280` (`buildCreateTables`). Mọi bảng resource, dù là `Patient` hay `Claim`, đều bắt đầu với đúng các cột này:

| Cột | Kiểu | Vai trò | Index |
|---|---|---|---|
| `id` | `UUID` PK | Khoá kỹ thuật | PK |
| `content` | `TEXT NOT NULL` | **Toàn bộ resource dạng JSON**, nguồn sự thật | — |
| `lastUpdated` | `TIMESTAMPTZ` | Thời điểm ghi cuối | btree |
| `deleted` | `BOOLEAN` | Xoá mềm | — |
| `projectId` | `UUID NOT NULL` | Tenant | btree, và `(projectId, lastUpdated)` |
| `__version` | `INTEGER` | Phiên bản **logic index** đã dùng để sinh dòng này (xem mục 6) | btree |
| `_source`, `_profile` | `TEXT`, `TEXT[]` | Nguồn gốc dữ liệu, profile áp dụng | btree, GIN |
| `compartments` | `UUID[]` | Bệnh nhân/account mà resource thuộc về (trừ `Binary`) | GIN |

Kèm 2 bảng phụ cho **mỗi** loại resource (`migrate.ts:293-318`):

| Bảng | Cột |
|---|---|
| `<Type>_History` | `versionId` PK, `id`, `content`, `lastUpdated` |
| `<Type>_References` | `resourceId`, `targetId`, `code`; PK gồm cả 3 |

🧠 Cột `content` là `TEXT`, không phải `JSONB`. Medplum **không bao giờ query vào bên trong JSON** để search. Mọi search đi qua cột chiếu. Lý do chính thức chưa tìm thấy, xem [mục 10](#10-câu-hỏi-còn-mở).

## 3. Phần sinh tự động: 4 chiến lược index search parameter

Phần còn lại của bảng **không ai viết tay**. ✅ `migrate.ts:323-367` (`buildSearchColumns`) duyệt mọi search parameter của loại resource đó, rồi hỏi `getSearchParameterImplementation` xem cần cột gì. Quyết định nằm trong 1 hàm duy nhất, ✅ `fhir/searchparameter.ts:116-182` (`buildSearchParameterImplementation`), đi theo thứ tự ưu tiên:

```
search parameter
  │
  ├─ type là date / number / quantity?             → range-column   (dòng 127-136)
  │
  ├─ type token VÀ element là Identifier /
  │  CodeableConcept / Coding / ContactPoint?       → token-column   (dòng 138-157)
  │
  ├─ thuộc 1 lookup table (tên người, địa chỉ)?     → lookup-table   (dòng 160-170)
  │
  └─ còn lại (string, uri, reference, token kiểu code đơn giản) → column  (dòng 172-183)
```

Bảng kiểu cột Postgres tương ứng, ✅ `migrate.ts:371-422` (`getSearchParameterColumns`):

| Chiến lược | Cột sinh ra | Kiểu Postgres |
|---|---|---|
| `range-column` (date) | `__<tên>`, `__<tên>Sort`, kèm cột cũ | `DATERANGE` / `DATEMULTIRANGE`, `DATE` |
| `range-column` (dateTime, period) | như trên | `TSTZRANGE` / `TSTZMULTIRANGE`, `TIMESTAMPTZ` |
| `range-column` (number, quantity) | như trên | `NUMRANGE` / `NUMMULTIRANGE`, `NUMERIC` |
| `token-column` | `__<tên>`, `__<tên>Text`, `__<tên>Sort` | `UUID[]`, `TEXT[]`, `TEXT` |
| `lookup-table` | chỉ cột sort (nếu có) | `TEXT` |
| `column` | `<tên>` | tuỳ kiểu: `TEXT`, `TEXT[]`, `BOOLEAN`, `DATE`… |

### 3.1 range-column: ngày, số, số lượng có đơn vị

✅ `searchparameter.ts:127-136`.

**Vì sao cần 🧠:** 📘 Trong FHIR, 1 giá trị ngày thực chất là **1 khoảng**. `birthDate = 1958` nghĩa là "khoảng 1958-01-01 đến 1958-12-31". `effectivePeriod` của Observation là khoảng thời gian thật. Câu hỏi `date=ge2026-01-01` là phép so sánh **khoảng**, không phải so sánh điểm. Kiểu `TSTZRANGE`/`NUMRANGE` của Postgres biểu diễn đúng ngữ nghĩa này và có index riêng.

**Đang ở giữa quá trình chuyển đổi ✅:** `migrate.ts` ghi chú `// Keep original column during migration` ở cả 3 nhánh. `row-builder.ts` vẫn ghi cả cột range mới lẫn cột cũ, kèm `// TODO: return here once migration is complete`. Đường search theo range còn bật/tắt theo cấu hình hoặc theo project: `repo.ts:2103` (`supportsRangeSearch`: `config.rangeSearch` hoặc `project.features` chứa `'range-search'`).

### 3.2 token-column: mã chuẩn, định danh, liên lạc

✅ Quy tắc chọn: `fhir/tokens.ts:33-82` (`getTokenIndexType`):
- Element kiểu `ContactPoint` (email, điện thoại) → token, **không phân biệt hoa thường**
- `Identifier`, `CodeableConcept`, `Coding` → token, phân biệt hoa thường
- Token chỉ là `code`/`string`/`boolean` đơn giản (VD `Patient.gender`, `Observation.status`) → **không** dùng token-column mà rơi xuống `column` thường

**Cách mã hoá ✅** `fhir/token-column.ts:27-124` (`buildTokenColumns`). Với 1 Coding `{system: "http://loinc.org", code: "8480-6"}`, hàm sinh **nhiều token**, mỗi token ứng 1 kiểu câu hỏi FHIR cho phép:

| Câu hỏi FHIR | Chuỗi token trước khi băm |
|---|---|
| `code=http://loinc.org\|` | `http://loinc.org` |
| `code=http://loinc.org\|8480-6` | `http://loinc.org` + DELIM + `8480-6` |
| `code=8480-6` (không cần system) | DELIM + `8480-6` |
| `code=\|8480-6` (bắt buộc không có system) | chỉ sinh khi Coding thật sự không có system |

Mỗi chuỗi được băm thành UUID v5 (`token-column.ts:130-132`, `hashTokenColumnValue`) và lưu vào mảng `UUID[]` có index GIN. Search = "mảng có giao với mảng băm của điều kiện không" (toán tử `ARRAY_OVERLAPS`, tức `&&` trong Postgres).

🧠 Vì sao băm: token y tế có thể rất dài (URL system + mã). Băm về UUID 16 byte cố định giúp index nhỏ và đều. Đổi lại, cột token **không đọc được bằng mắt**. Muốn đọc phải xem `content`.

**Cột riêng hay cột dùng chung ✅** `searchparameter.ts:265-285` (`hasDedicatedTokenColumns`):
- Mặc định mỗi token parameter có **cột riêng** (`__identifier`, `__email`, `__code`…).
- Ngoại lệ: tham số kết thúc bằng `:identifier` và `_security` dồn vào **cột dùng chung** `__sharedTokens`. Token có tiền tố là tên tham số để không lẫn nhau.
- Có danh sách override cho tham số hay dùng, VD `Observation|patient:identifier` được cột riêng (`searchparameter.ts:256-262`).

🧠 Đây là đánh đổi **số cột ↔ tốc độ**: cột riêng thì index nhỏ, search nhanh, nhưng bảng thêm 3 cột. Cột dùng chung thì ít cột hơn, nhưng index phình to. Medplum chọn cột riêng làm mặc định, chỉ dồn chung cho tham số ít dùng.

### 3.3 lookup-table: mảng có cấu trúc

✅ `fhir/lookups/`:

| Bảng | Index cho | Vì sao không nhét vào bảng resource |
|---|---|---|
| `HumanName` | `name`, `given`, `family` của `Patient`, `Person`, `Practitioner`, `RelatedPerson` (`lookups/humanname.ts:21,33-40`) | 1 người nhiều tên. Cần search "có tên nào chứa X" bằng `ILIKE` (`humanname.ts:60`). Mỗi tên 1 dòng: `resourceId`, `projectId`, `name`, `given`, `family` (`humanname.ts:26-31`) |
| `Address` | `address`, `address-city`, `address-postalcode`… | 1 người nhiều địa chỉ, nhiều phần (đường/thành phố/mã bưu chính) |
| `<Type>_References` | Mọi reference (trừ `_compartment` và `:identifier`) (`lookups/reference.ts:305-311`) | **Không dùng cho search thường** (`isIndexed() → false`, `reference.ts:61`). Chỉ dùng làm bảng JOIN cho chained search (`fhir/search.ts:1689`) |
| `Coding` | Nội dung các `CodeSystem` | Không dùng cho search resource (`coding.ts:37`). Phục vụ terminology: `$lookup`, `$validate-code`, `$expand` |
| `ConceptMapping` | Nội dung các `ConceptMap` | Phục vụ `$translate` |

Riêng với tên người, Medplum vẫn ghi thêm 1 cột sort (`__nameSort`, `__familySort`…) lên bảng resource để sắp xếp không cần JOIN (`row-builder.ts:197-201`).

### 3.4 column: phần còn lại

✅ `searchparameter.ts:172-183`. Mỗi tham số 1 cột kiểu tương ứng. Tham số là mảng thì cột kiểu mảng (`TEXT[]`…). Giá trị trong mảng được sắp xếp trước khi ghi (`row-builder.ts:236-238`).

**Đối chứng nội bộ, nơi Medplum cố tình chọn cách đơn giản 🧠:**
- **Reference chỉ trỏ đúng 1 loại đích** (VD tham số chỉ nhận `Patient`): `singleTargetType` (`searchparameter.ts:175-181`). Đây vẫn là cột thường, chỉ ghi nhớ loại đích để JOIN thẳng. Không cần dispatch theo loại.
- **Token đơn giản** (`gender`, `status`): không băm, không cột `Text`/`Sort` riêng, chỉ 1 cột `TEXT`.

Đây là bản tương đương `ShippingMethod` trong bài Saleor: **cơ chế phức tạp chỉ được bật khi kiểu dữ liệu thật sự đòi hỏi**.

## 4. Dữ liệu thật trông thế nào

💭 Giá trị tự dựng. Tên cột ✅ lấy từ các migration trong `migrations/schema/` (VD `__identifier`, `__identifierText`, `__identifierSort`, `__birthdate`, `__birthdateSort`, `__nameSort`, `__phone`, `__email` có trong các file `v*.ts` cho bảng `"Patient"`). Chỉ trích các cột liên quan.

**Bảng `"Patient"`, 1 dòng:**

| id | projectId | content | gender | __birthdate | birthdate | __identifier | __nameSort | __phone | compartments | __version | deleted |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `8f2c…` | `p-anbinh…` | `{"resourceType":"Patient","name":[…],…}` | `female` | `[1958-03-12,1958-03-13)` | `1958-03-12` | `{a91e…, 03bd…, 7c44…}` | `Nguyễn Thị Lan` | `{5e02…, d1f9…}` | `{8f2c…}` | 19 | false |

Đọc dòng này:
- `gender`: token đơn giản → cột `TEXT` thường (mục 3.4).
- `__birthdate` + `birthdate`: range mới và cột cũ **cùng tồn tại** (mục 3.1).
- `__identifier`: 3 UUID băm từ 1 số CCCD, theo các dạng câu hỏi ở mục 3.2.
- `compartments`: Patient nằm trong compartment của chính nó.
- `__version = 19`: khớp `Repository.VERSION` hiện tại (`repo.ts:292`).

**Bảng `"HumanName"`, 2 dòng cho 2 tên:**

| resourceId | projectId | name | given | family |
|---|---|---|---|---|
| `8f2c…` | `p-anbinh…` | `Nguyễn Thị Lan` | `Thị Lan` | `Nguyễn` |
| `8f2c…` | `p-anbinh…` | `Lan Nguyễn` | `Lan` | `Nguyễn` |

**Bảng `"Observation"`, huyết áp tâm thu:**

| id | content | status | __code | __valueQuantity | valueQuantity | subject | __date | compartments |
|---|---|---|---|---|---|---|---|---|
| `b71a…` | `{"resourceType":"Observation","code":{"coding":[{"system":"http://loinc.org","code":"8480-6"}]},"valueQuantity":{"value":150,"unit":"mmHg",…},…}` | `final` | `{…UUID băm của loinc\|8480-6…}` | `[150,150]` | `150` | `Patient/8f2c…` | `[2026-10-14 08:00,…]` | `{8f2c…}` |

`compartments` của Observation chứa id bà Lan. Đây là nhờ `getPatients` đọc `subject` theo CompartmentDefinition (`fhir/patient.ts:56-76`). Nhờ vậy, câu "bác sĩ chỉ xem bệnh nhân được phân" quy về điều kiện `compartments && ARRAY[...]`. Xem `repo.ts:1961`.

💭 Câu `Observation?code=http://loinc.org|8480-6&value-quantity=gt140` thành SQL có dạng gần đúng:

```sql
SELECT "Observation"."content" FROM "Observation"
WHERE "Observation"."projectId" = $1
  AND "Observation"."deleted" = false
  AND "Observation"."__code" && ARRAY[$2]::UUID[]      -- $2 = uuid_v5('http://loinc.org' + DELIM + '8480-6')
  AND "Observation"."valueQuantity" > 140               -- hoặc so sánh trên "__valueQuantity" khi bật range-search
```

**Kết quả trả về luôn là cột `content`.** Cột search chỉ dùng để lọc, không dùng để dựng lại dữ liệu.

## 5. Ghi, sửa, xoá: lịch sử không bao giờ mất

| Thao tác | Bảng resource | Bảng `_History` | Nguồn ✅ |
|---|---|---|---|
| Tạo / sửa | `INSERT … ON CONFLICT` (merge) với `versionId` mới | Thêm 1 dòng | `repo.ts:2007-2051` |
| Sửa nhưng không đổi gì | Không ghi | Không ghi | `repo.ts:1087` (`isNotModified`) |
| Sửa có `If-Match` sai phiên bản | Từ chối `412` | — | `repo.ts:1026` |
| Xoá | Dòng giữ lại: `deleted = true`, `content = ''`, `__version = -1`, cột search rỗng | Thêm 1 dòng "đã xoá" | `row-builder.ts:105-125`, `repo.ts:1546-1565` |
| Expunge (xoá vĩnh viễn) | Xoá thật | Xoá cả lịch sử | `repo.ts:1713`, operation `$expunge` |

🧠 Ý nghĩa nghiệp vụ:
- **Xoá thường không xoá.** Hồ sơ bệnh án phải giữ được bằng chứng "đã từng có, đã bị xoá bởi ai, khi nào". Bảng `_History` vẫn còn đủ phiên bản cũ.
- **`$expunge` tồn tại vì luật cũng có lúc buộc xoá thật**: dữ liệu nhập nhầm người, yêu cầu xoá dữ liệu cá nhân. Đây là operation riêng, không phải `DELETE` thường, để việc xoá thật luôn là hành động có chủ đích.

## 6. Con đường tiến hoá: Medplum cũng từng lưu số dạng chuỗi

Phần đáng học nhất của module. Nó cho thấy thiết kế ở mục 2–3 **không ra đời 1 lần** mà sửa dần qua sự cố. Đây đúng là bài học "bất đối xứng thông tin theo thời gian" ở Phần 7 của Template G.

**Chặng 1: v1, mọi thứ là TEXT ✅** `migrations/schema/v1.ts:2359-2391`. Bảng `"Observation"` ban đầu có `"code" TEXT` (dòng 2364), `"date" TEXT` (dòng 2365), `"valueQuantity" TEXT` (dòng 2391).

🧠 Đây **chính xác** là ứng viên "EAV thuần / ép mọi kiểu về chuỗi" trong bài Saleor, với đúng bug sort `"16" < "4"`. Với `valueQuantity = "9"` và `"10"`, sắp xếp chuỗi cho `"10" < "9"`. Trong y tế, đó là danh sách huyết áp hoặc đường huyết bị xếp sai thứ tự.

**Chặng 2: v18, đổi kiểu ✅** `migrations/schema/v18.ts:102-108`:

```sql
ALTER TABLE "Observation"
  ALTER COLUMN "date" TYPE TIMESTAMP WITH TIME ZONE USING NULL,
  ...
  ALTER COLUMN "valueQuantity" TYPE DOUBLE PRECISION USING NULL
```

Chú ý `USING NULL`: dữ liệu cũ trong cột bị **bỏ đi**, không chuyển đổi. 🧠 Làm vậy được vì cột search chỉ là bản chiếu. Bản gốc vẫn còn nguyên trong `content`, nên chỉ cần **reindex** là dựng lại cột với kiểu đúng. Nếu cột này là nguồn sự thật (như `form_vitals.bps` của OpenEMR), `USING NULL` sẽ là thảm hoạ mất dữ liệu.

**Chặng 3: v105, chuyển sang kiểu khoảng ✅** `migrations/schema/v105.ts:268,276`: thêm `"__date" TSTZRANGE`, `"__valueQuantity" NUMRANGE`, giữ cột cũ song song (mục 3.1).

**Cơ chế cho phép tiến hoá liên tục ✅** `repo.ts:261-292` (`Repository.VERSION = 19`). Mỗi lần đổi logic sinh cột, Medplum tăng số version và ghi vào cột `__version` của từng dòng. Job reindex (`workers/reindex.ts:285-312`) tìm các dòng có `__version` cũ hơn, đọc lại `content`, sinh lại cột. Lịch sử 19 version ghi ngay trong comment, trích vài mốc:

| Version | Ngày | Thay đổi |
|---|---|---|
| 1 | 02/2025 | Thêm cột `__version` |
| 3 | 04/2025 | Thêm cột `__tokens` cho chiến lược `token-column` |
| 6 | 06/2025 | Tách cột token riêng cho từng tham số |
| 10 | 08/2025 | Thêm cột sort cho HumanName |
| 11 | 09/2025 | Thêm lookup table `ConceptMapping` |
| 15 | 05/2026 | Thêm chiến lược `range-column` |
| 19 | 10/2026 | Thêm search param `CoverageEligibilityRequest-coverage` |

🧠 Bài học thiết kế quan trọng nhất của module: **tách "nguồn sự thật" khỏi "cấu trúc phục vụ truy vấn"** thì đổi được cấu trúc truy vấn bao nhiêu lần cũng được, mà không cần migration dữ liệu nghiệp vụ. Đây là lý do Medplum tiến hoá được từ TEXT → DOUBLE → NUMRANGE mà không mất 1 kết quả xét nghiệm nào.

## 7. Multi-tenant ngay trong schema

✅
- Mọi bảng resource có `projectId UUID NOT NULL` (`migrate.ts:255`), index `(projectId, lastUpdated)` (`migrate.ts:262`).
- Lookup table `HumanName` cũng có `projectId` (`humanname.ts:61`, `hasProjectIdColumn = true`).
- Với các tham số search dùng nhiều, index được **đặt `projectId` lên đầu** (biến thể `ProjectScoped`, `migrate.ts:437+`). VD `Observation.code`, `Observation.status`, `Task.status` + `lastUpdated`, `Appointment.status`.
- Commit gần nhất của repo tại thời điểm viết: "Restrict chained search joins by projectId (#10894)". Chained search cũng phải lọc tenant ở **từng bước JOIN**.

🧠 Ý nghĩa: 1 phòng khám không bao giờ thấy dữ liệu phòng khám khác, kể cả qua lỗi truy vấn. Index đặt `projectId` lên đầu còn giúp hiệu năng: tenant nhỏ không phải quét dữ liệu của tenant lớn.

## 8. Đối chiếu OpenEMR: cùng bà Lan, schema khác hẳn

✅ OpenEMR `sql/database.sql`. Cùng 2 thông tin "bà Lan" và "huyết áp tâm thu 150":

**`patient_data`** (`:8350`, **132 cột**), 1 dòng:

| pid | fname | mname | lname | DOB | phone_cell | street | … 125 cột khác |
|---|---|---|---|---|---|---|---|
| 1 | Lan | Thị | Nguyễn | 1958-03-12 | 09… | … | phần lớn rỗng |

→ **Cột cố định** (ứng viên 2.1 trong bài Saleor). Tên thứ hai "Lan Nguyễn" không có chỗ lưu, vì mỗi trường chỉ 1 cột. Có 1 cột `dupscore` (`:8468`) phục vụ ghép trùng bệnh nhân.

**`form_vitals`** (`:2422`), 1 dòng cho cả lần đo:

| id | pid | date | bps | bpd | weight | pulse | … |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 2026-10-14 08:00 | `'150'` | `'95'` | 0.00 | 0.00 | … |

→ `bps`/`bpd` là **`varchar(40)`** (`:2431-2432`), trong khi `weight`, `pulse` là `DECIMAL`. Huyết áp lưu dạng chuỗi. Khi xuất FHIR, lớp dịch phải gắn mã LOINC (`src/Services/FHIR/Observation/FhirObservationVitalsService.php:95`, `8480-6`), và có cả luật ngầm "`bps` và `bpd` cùng bằng 0 nghĩa là không có dữ liệu" (`:880-882`).

**Các kỹ thuật khác cùng tồn tại trong OpenEMR:**

| Kỹ thuật | Bảng | Ghi chú |
|---|---|---|
| STI | `lists` (`:7685`) | Cột `type` phân biệt `medical_problem`, `allergy`, `medication`, `surgery`… trong cùng 1 bảng |
| Liên kết đa hình | `forms` (`:2464`) | `formdir` + `form_id` trỏ sang 1 trong 38 bảng `form_*` |
| EAV | `lbf_data` (`:10262`) | `(form_id, field_id, field_value LONGTEXT)` cho form tự cấu hình. Mọi giá trị là chuỗi |
| EAV có phạm vi | `shared_attributes` (`:12603`) | `(pid, encounter, field_id, field_value)`; `encounter = 0` nghĩa là thuộc tính cấp bệnh nhân |

🧠 So sánh trực diện:

| Câu hỏi | Medplum | OpenEMR |
|---|---|---|
| Nguồn sự thật | JSON FHIR (`content`) | Bảng quan hệ tự thiết kế |
| Thêm 1 trường mới | `extension` trong JSON, không migration. Muốn search được thì thêm `SearchParameter` rồi reindex | `ALTER TABLE` hoặc dùng form LBF (EAV) |
| Đổi kiểu cột search | Migration `USING NULL` + reindex, không mất dữ liệu | Migration chuyển đổi dữ liệu thật, rủi ro mất/sai |
| Nhiều tên/địa chỉ | Tự nhiên (mảng + lookup table) | Không có chỗ, trừ khi thêm bảng |
| Xuất FHIR | Không cần dịch | Lớp dịch `src/Services/FHIR/*` cho từng resource |
| Đọc DB bằng mắt / báo cáo SQL | Khó: token băm, JSON trong TEXT | Dễ: cột có tên nghiệp vụ |
| Tốc độ ra sản phẩm cho 1 phòng khám | Chậm hơn: phải tự xây nghiệp vụ | Nhanh: nghiệp vụ có sẵn |

Dòng "Đọc DB bằng mắt" là cái giá thật của Medplum. Team BI hoặc kế toán muốn viết SQL báo cáo trực tiếp sẽ gặp khó, phải đi qua API, `$export`, hoặc SQL-on-FHIR (`packages/core/src/sql-on-fhir/`).

## 9. Đặc tính bài toán rút ra (nháp cho file so sánh)

🧠 Đầu vào cho `1a-comparison-fhir-datastore.md`. Mỗi đặc tính ứng với 1 thứ đã thấy ở trên:

| # | Đặc tính | Bằng chứng trong thiết kế |
|---|---|---|
| P1 | Khoảng 150 loại tài liệu lồng sâu, có `extension` tự do | `content TEXT` + bảng sinh tự động (mục 2, 3) |
| P2 | Search đúng kiểu và đúng ngữ nghĩa khoảng (ngày, số có đơn vị) | `range-column`; bài học v1→v18→v105 (mục 3.1, 6) |
| P3 | Mảng có cấu trúc search được từng phần tử | `HumanName`, `Address` (mục 3.3) |
| P4 | Mã chuẩn `system\|code` với nhiều dạng câu hỏi | `token-column` băm (mục 3.2) |
| P5 | Lịch sử bất biến, xoá mềm, xoá thật có chủ đích | `_History`, `deleted`, `$expunge` (mục 5) |
| P6 | Tách tenant tuyệt đối | `projectId`, `compartments`, index `ProjectScoped` (mục 7) |
| P7 | Đổi cấu trúc index liên tục mà không migration dữ liệu | `__version` + reindex (mục 6) |

Ứng viên dự kiến cho file so sánh:
1. Cột cố định: **OpenEMR thật** `patient_data`
2. Document thuần JSONB + GIN: Fhirbase
3. EAV: **OpenEMR thật** `lbf_data`, và chính Medplum v1
4. Bảng index riêng theo kiểu: **HAPI FHIR** (`HFJ_SPIDX_*`), cần clone để grounding
5. Hybrid của Medplum (file này)

Khác bài Saleor: ứng viên 1, 3, 5 đều là **hệ thống thật** (✅), không phải 💭 phản thực tế.

## 10. Câu hỏi còn mở

| Câu hỏi | Giả thuyết 🧠 | Cách kiểm chứng |
|---|---|---|
| Vì sao `content` là `TEXT`, không phải `JSONB`? | Không bao giờ query vào trong. TEXT giữ nguyên chuỗi gốc (thứ tự key, khoảng trắng), tránh chi phí chuẩn hoá khi ghi | Tìm trong issue/PR Medplum, `packages/docs/docs/self-hosting/` |
| Khi nào `range-column` thành mặc định và bỏ cột cũ? | Sau khi mọi deployment reindex xong lên version ≥ 15 | Theo dõi `TODO` trong `row-builder.ts` và `migrate.ts` |
| Reindex 1 bảng `Observation` hàng trăm triệu dòng mất bao lâu, có khoá bảng không? | Chạy theo trang, có timeout (`workers/reindex.ts`) | Đọc kỹ `reindex.ts` và docs self-hosting |
| `__sharedTokens` có còn được mở rộng không, hay đang thu hẹp dần? | Đang thu hẹp: version 6 đã tách cột riêng | Đếm override theo thời gian trong `searchparameter.ts` |

## Sai lầm thường gặp khi đọc phần này

1. **Tưởng Medplum "lưu JSON nên không có schema".** Sai. Bảng `Observation` có hàng chục cột search có kiểu chặt. JSON chỉ là nguồn sự thật, không phải nơi truy vấn.
2. **Viết SQL báo cáo đọc thẳng cột `__code`.** Cột này là UUID băm, không đọc được. Báo cáo phải đi qua API, `$export` hoặc SQL-on-FHIR.
3. **Sửa thẳng `content` trong DB** (VD sửa dữ liệu nhanh bằng `UPDATE`). Cột search, lookup table và `_History` sẽ lệch khỏi `content`. Mọi thay đổi phải đi qua `Repository`.
4. **Nhầm `deleted = true` là đã xoá.** Lịch sử vẫn còn trong `_History`. Muốn xoá thật phải `$expunge`.
5. **Chép thiết kế này cho MVP chỉ có 1 phòng khám, 5 loại dữ liệu.** 🧠 Cơ chế sinh cột tự động, băm token, reindex theo version chỉ đáng giá khi có khoảng 150 loại resource và nhiều tenant. MVP nhỏ nên **dùng Medplum** chứ không **chép cách Medplum lưu trữ**. Xem lại Phần 7 của Template G: "khi nào KHÔNG nên nhảy thẳng lên thiết kế thật".

## Tự kiểm tra

1. Thêm 1 search parameter tuỳ biến `Patient.dan-toc` (kiểu `token`, element là `CodeableConcept`). Chiến lược nào được chọn? Bảng `Patient` có thêm những cột nào, kiểu gì?
2. Vì sao migration v18 dám dùng `USING NULL`? Nếu là OpenEMR thì có dám không?
3. Bà Lan có 2 tên. Tìm `Patient?name=Lan Nguyễn` đi qua bảng nào? Sắp xếp theo tên đi qua cột nào?
4. Xoá 1 Observation nhập nhầm, rồi thanh tra hỏi "đã từng có kết quả này không". Trả lời bằng bảng nào?
5. Vì sao `Patient.gender` không dùng `token-column` dù là tham số kiểu token?

## Nguồn

- Medplum `16cb130`:
  - `packages/server/src/fhir/repo.ts`
  - `packages/server/src/fhir/searchparameter.ts`
  - `packages/server/src/fhir/tokens.ts`
  - `packages/server/src/fhir/token-column.ts`
  - `packages/server/src/fhir/repository/row-builder.ts`
  - `packages/server/src/fhir/lookups/*.ts`
  - `packages/server/src/fhir/patient.ts`
  - `packages/server/src/migrations/migrate.ts`
  - `packages/server/src/migrations/schema/v1.ts`, `v18.ts`, `v105.ts`
  - `packages/server/src/workers/reindex.ts`
- OpenEMR `4c1eb8d`:
  - `sql/database.sql`
  - `src/Services/FHIR/Observation/FhirObservationVitalsService.php`
- 📘 FHIR R4 Search: hl7.org/fhir/R4/search.html (ngữ nghĩa khoảng của date/quantity, token `system|code`)
