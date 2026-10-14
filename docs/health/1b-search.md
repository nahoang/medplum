# 1b — Search: 1 câu hỏi lâm sàng biến thành SQL thế nào

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Bác sĩ hỏi 'các lần đo huyết áp > 140 của bà Lan trong năm nay, kèm tên bác sĩ đo' thì hệ thống chạy những bước nào?"* File [`1a`](./1a-fhir-datastore.md) nói dữ liệu **nằm ở đâu**. File này nói dữ liệu **được lấy ra thế nào**.
>
> Đường dẫn Medplum tính từ `packages/server/src/` trừ khi ghi khác.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [1. Vì sao search là nghiệp vụ, không chỉ là kỹ thuật](#1-vì-sao-search-là-nghiệp-vụ-không-chỉ-là-kỹ-thuật)
- [2. Đường đi của 1 câu search](#2-đường-đi-của-1-câu-search)
- [3. Từ URL thành bộ lọc: ngôn ngữ search của FHIR](#3-từ-url-thành-bộ-lọc-ngôn-ngữ-search-của-fhir)
- [4. Từ bộ lọc thành SQL: dispatch theo chiến lược lưu trữ](#4-từ-bộ-lọc-thành-sql-dispatch-theo-chiến-lược-lưu-trữ)
- [5. Bộ lọc luôn có: tenant, xoá mềm, phân quyền](#5-bộ-lọc-luôn-có-tenant-xoá-mềm-phân-quyền)
- [6. Search lồng (chained) và search ngược (_has)](#6-search-lồng-chained-và-search-ngược-_has)
- [7. _include / _revinclude: lấy kèm dữ liệu liên quan](#7-_include--_revinclude-lấy-kèm-dữ-liệu-liên-quan)
- [8. Phân trang và đếm tổng](#8-phân-trang-và-đếm-tổng)
- [9. Hai bản hiện thực của cùng 1 ngôn ngữ search](#9-hai-bản-hiện-thực-của-cùng-1-ngôn-ngữ-search)
- [10. Đối chiếu OpenEMR](#10-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa |
|---|---|
| ✅ | Code thật, có `file:line` |
| 📘 | FHIR R4 Search (hl7.org/fhir/R4/search.html), docs `packages/docs/docs/search/` |
| 💭 | Ví dụ tự dựng, SQL minh hoạ gần đúng |
| 🧠 | Suy luận |

---

## 1. Vì sao search là nghiệp vụ, không chỉ là kỹ thuật

🧠 Trong ecommerce, search sai thì khách không thấy sản phẩm, mất 1 đơn hàng. Trong y tế, các câu hỏi hằng ngày đều là search:
- "Danh sách bệnh nhân hẹn hôm nay"
- "Kết quả xét nghiệm chưa ai duyệt"
- "Bệnh nhân tiểu đường có HbA1c > 9 chưa tái khám 3 tháng"

Search **sót** 1 kết quả bất thường là bác sĩ không biết để xử lý. Search **thừa** (lộ bệnh nhân không thuộc quyền) là vi phạm quyền riêng tư. Vì vậy search phải đúng ngữ nghĩa và phải mang theo phân quyền ở **mọi** câu truy vấn. Mục 5 cho thấy Medplum làm điều đó ra sao.

## 2. Đường đi của 1 câu search

💭 Câu ví dụ, dùng xuyên suốt file:

```
GET /fhir/R4/Observation?subject=Patient/8f2c&code=http://loinc.org|8480-6
    &value-quantity=gt140&date=ge2026-01-01
    &_include=Observation:performer&_sort=-date&_count=20
```

✅

```
packages/core/src/search/search.ts:141   parseSearchRequest      URL → SearchRequest {filters, sortRules, include, count…}
        │
packages/server/src/fhir/search.ts:113   searchImpl
        ├─ :219  applyCountAndOffsetLimits   mặc định 20, tối đa 1000 (core/search/search.ts:11-12)
        ├─ :281  getSelectQueryForSearch
        │     └─ :432 getBaseSelectQueryForResourceType
        │           ├─ chọn cột id, lastUpdated, content
        │           ├─ addDeletedFilter          (repo.ts:1877)
        │           ├─ addSecurityFilters        (repo.ts:1887)  ← tenant + AccessPolicy
        │           └─ addSearchFilters          (:848) → mỗi filter 1 biểu thức WHERE
        ├─ :1526 addSortRules
        ├─ :319  getSearchEntries   chạy SQL, parse cột content thành resource
        │     └─ :383 getExtraEntries (packages/fhir-router/src/search-include.ts:79)  ← _include
        ├─ :735  getCount            (nếu _total=accurate|estimate)
        └─ getSearchLinks            first / next / previous
```

**SQL không viết bằng ORM.** Medplum có bộ dựng SQL riêng, `fhir/sql.ts` (`SelectQuery`, `Condition`, `Disjunction`…). 🧠 Lý do: cần dùng trực tiếp các toán tử riêng của Postgres như `&&` trên mảng, range, `EXISTS` lồng, mà ORM thường không diễn đạt gọn.

## 3. Từ URL thành bộ lọc: ngôn ngữ search của FHIR

📘 Mỗi tham số trong URL có dạng `tên[:modifier]=[prefix]giá-trị`. Danh sách toán tử Medplum hỗ trợ, ✅ `packages/core/src/search/search.ts:55-92`:

| Loại | Toán tử | Ví dụ nghiệp vụ |
|---|---|---|
| Prefix so sánh | `eq ne gt lt ge le sa eb ap` | `value-quantity=gt140`: huyết áp trên 140. `date=ge2026-01-01`: từ đầu năm |
| Modifier chuỗi | `:contains :exact` (mặc định là "bắt đầu bằng") | `name:contains=Lan` |
| Modifier token | `:text :not` (dùng được). `:above :below :in :not-in :of-type` có trong danh sách của core nhưng **server từ chối** với tham số `token-column` (`fhir/token-column.ts:240-255`) | `code:not=8480-6` |
| Có / không có | `:missing :present` | `Observation?interpretation:missing=true`: kết quả chưa diễn giải |
| Định danh | `:identifier` | `subject:identifier=https://bhyt.gov.vn\|HT27…`: tìm theo số thẻ BHYT thay vì id |
| Lặp | `:iterate` | `_include:iterate`: lấy kèm nhiều tầng |

**Vài tham số đặc biệt** ✅ `fhir/search.ts:1088+` (`trySpecialSearchParameter`): `_id`, `_lastUpdated`, `_deleted`, `_project`, `_compartment`, `_filter`.

📘 `_filter` là ngôn ngữ biểu thức đầy đủ (có `and`/`or`/ngoặc) cho câu hỏi mà cú pháp URL thường không diễn đạt được. VD "huyết áp > 140 **hoặc** nhịp tim > 120". ✅ `search.ts:1124-1204`.

**Giới hạn đáng biết: không search được theo ValueSet.** 📘 Chuẩn FHIR cho phép `code:in=ValueSet/tim-mach` (mọi Condition có mã thuộc nhóm bệnh tim mạch) và `code:below=…` (mã con trong cây). ✅ Tại commit này, Medplum **từ chối** cả 2 cho tham số token (`fhir/token-column.ts:240-255` ném `invalidSearchOperator`), và bộ so khớp trong bộ nhớ (`packages/core/src/search/match.ts`) cũng không xử lý `IN`/`BELOW`. 🧠 Cách làm thay: gọi `ValueSet/$expand` lấy danh sách mã, rồi search `code=sys|a,sys|b,…` (dấu phẩy = OR). Danh sách dài thì chia lô. Xem module [1e](./1e-terminology.md).

## 4. Từ bộ lọc thành SQL: dispatch theo chiến lược lưu trữ

✅ `fhir/search.ts:903-997` (`buildSearchFilterExpression`). Thứ tự xử lý 1 filter:

1. Giá trị chứa `\0` → từ chối (`:915-917`).
2. Là chained search → mục 6 (`:919-922`).
3. Là tham số đặc biệt (`_id`, `_filter`…) → xử lý riêng (`:924-934`).
4. Tham số không tồn tại → lỗi `Unknown search parameter` (`:936-939`). 📘 FHIR **không cho tìm theo trường tuỳ ý**.
5. Còn lại: **dispatch theo chiến lược lưu trữ** đã chọn ở [`1a`](./1a-fhir-datastore.md#3-phần-sinh-tự-động-4-chiến-lược-index-search-parameter) (`:950-984`):

| Chiến lược | Hàm dựng SQL | SQL ra dạng (💭) |
|---|---|---|
| `token-column` | `buildTokenColumnsSearchFilter` (`fhir/token-column.ts:170`) | `"__code" && ARRAY[uuid_v5('http://loinc.org'+DELIM+'8480-6')]::UUID[]` |
| `lookup-table` | `lookupTable.buildWhere` (`fhir/lookups/lookuptable.ts`) | `EXISTS (SELECT 1 FROM "HumanName" WHERE "resourceId" = "Patient"."id" AND "name" ILIKE '%Lan%')` |
| `range-column` | `buildRangeColumnsSearchFilter` (`fhir/range-column.ts`), **chỉ khi** `supportsRangeSearch()` | `"__valueQuantity" && numrange(140, NULL, '()')` |
| `range-column` khi chưa bật | quay về `buildNormalSearchFilterExpression` (`:963-972`) | `"valueQuantity" > 140` |
| `column` | `buildNormalSearchFilterExpression` (`:999-1036`) | `"subject" = 'Patient/8f2c'` |

**Câu ví dụ ở mục 2 thành SQL (💭, gần đúng):**

```sql
SELECT "Observation"."id", "Observation"."lastUpdated", "Observation"."content"
FROM "Observation"
WHERE "Observation"."deleted" = false                                   -- xoá mềm
  AND "Observation"."projectId" IN ($p)                                 -- tenant
  AND ( "Observation"."compartments" && ARRAY[$c]::UUID[] )             -- AccessPolicy (nếu có)
  AND "Observation"."subject" = 'Patient/8f2c'                          -- column
  AND "Observation"."__code" && ARRAY[$h]::UUID[]                       -- token-column
  AND "Observation"."valueQuantity" > 140                               -- column / range
  AND "Observation"."date" >= '2026-01-01'
ORDER BY "Observation"."date" DESC
LIMIT 21;                                                               -- 20 + 1 để biết còn trang sau
```

`LIMIT 21` ✅ `search.ts:291`: lấy thừa 1 dòng để biết có trang tiếp theo mà không cần đếm.

## 5. Bộ lọc luôn có: tenant, xoá mềm, phân quyền

✅ `fhir/search.ts:451-454` gắn 2 bộ lọc vào **mọi** câu search, trước cả filter của người dùng:

| Bộ lọc | Code | Ý nghĩa |
|---|---|---|
| `deleted = false` | `repo.ts:1877-1879` | Không trả resource đã xoá mềm (trừ khi hỏi `_deleted`) |
| `projectId IN (...)` | `repo.ts:1919-1924` (`addProjectFilters`) | Chỉ trong các project người gọi được phép. Super admin không bị giới hạn (`:1897-1903`) |
| AccessPolicy | `repo.ts:1932-1999` (`addAccessPolicyFilters`) | Thu hẹp tiếp theo chính sách của người dùng |

**Điều đáng học nhất:** AccessPolicy có trường `criteria` viết **bằng chính ngôn ngữ search FHIR**, VD `Observation?subject.organization=Organization/an-binh`. Medplum parse chuỗi đó rồi gọi lại **đúng hàm dựng SQL của search** (`repo.ts:1974-1983`: `parseSearchRequest` → `buildSearchExpression`). Các policy được nối bằng `OR` (`Disjunction`, `repo.ts:1996-1998`).

🧠 Ý nghĩa:
- **Phân quyền = 1 câu search được gắn thêm vào mọi câu search khác.** Người viết chính sách không cần biết SQL, chỉ cần biết search FHIR.
- Không có đường nào lấy dữ liệu mà né được bộ lọc này, vì nó gắn ở `getBaseSelectQueryForResourceType`, nơi mọi search, count, include đều đi qua.

Chi tiết phân quyền ở module 1c.

## 6. Search lồng (chained) và search ngược (_has)

📘 **Chained search** đi xuôi theo reference: `Observation?subject:Patient.name=Lan` nghĩa là "Observation có subject là Patient tên Lan". **Reverse chain** (`_has`) đi ngược: `Patient?_has:Observation:subject:code=4548-4` nghĩa là "bệnh nhân có ít nhất 1 Observation HbA1c".

🧠 Ví dụ nghiệp vụ của `_has`: "danh sách bệnh nhân **đã từng** làm HbA1c". Đây là câu hỏi lọc quần thể bệnh nhân, nền của quản lý bệnh mạn tính.

✅ Cách Medplum dựng (`fhir/search.ts:1628-1763`):
- Tối đa **3 bước** lồng (`:1635-1637`).
- Trường hợp đặc biệt `subject._id=X` được viết lại thành `subject=Patient/X`, không cần JOIN (`:1644-1666`).
- Có thể **tắt** chained search cho từng loại resource qua cấu hình `disableChainedSearch` (`fhir/lookups/reference.ts:35-39`). 🧠 Đây là van an toàn hiệu năng cho bảng rất lớn.
- Còn lại: dựng 1 subquery `EXISTS (...)` đi qua bảng `<Type>_References` ở mỗi bước (`:1689-1763`, kết thúc bằng `SqlFunction('EXISTS', …)` ở `:1762`).
- Mỗi bước JOIN đều gắn lọc `projectId` (`withProjectFilter`, `:1874`). Commit mới nhất của repo, *"Restrict chained search joins by projectId (#10894)"*, cho thấy đây là chỗ vừa được siết thêm.

## 7. _include / _revinclude: lấy kèm dữ liệu liên quan

📘 `_include=Observation:performer` nghĩa là trả kèm luôn các `Practitioner` được tham chiếu, để app không phải gọi thêm N lần.

✅ Logic nằm ở package dùng chung `packages/fhir-router/src/search-include.ts:79` (`getExtraEntries`):
- `:iterate` tối đa **5 vòng** (`DEFAULT_MAX_INCLUDE_DEPTH`, `:28`).
- Tổng kết quả vượt `DEFAULT_MAX_SEARCH_COUNT` (1000) thì từ chối (`:96-97`).

Server cung cấp hàm chạy sub-search (`fhir/search.ts:498-515`). Mỗi sub-search lại đi qua `getSelectQueryForSearch`, nên **vẫn mang đủ bộ lọc tenant và phân quyền** ở mục 5.

🧠 Ví dụ nghiệp vụ: màn hình "chi tiết lượt khám" gọi 1 lần `Encounter?_id=X&_revinclude=Observation:encounter&_revinclude=Condition:encounter&_include=Encounter:participant` thay vì 4 request riêng.

## 8. Phân trang và đếm tổng

**Phân trang** ✅ `fhir/search.ts:569-577` (`canUseCursorLinks`). Medplum dùng **cursor** thay vì offset khi:
- không có offset,
- `_count ≥ 20`,
- sắp xếp duy nhất theo `_lastUpdated` tăng dần.

Cursor lưu mốc `lastUpdated` và danh sách id đã trả (`:297-306`). 🧠 Phù hợp cho đồng bộ dữ liệu sang hệ thống khác: duyệt hàng triệu bản ghi mà không chậm dần như `OFFSET`.

**Đếm tổng** ✅ `fhir/search.ts:735-765` (`getCount`):

| Tình huống | Cách đếm |
|---|---|
| Trang hiện tại chưa đầy | Biết chính xác ngay: `offset + số dòng` (`:743-752`), không chạy thêm query |
| Còn lại | Chạy `EXPLAIN` để lấy **ước lượng** từ query planner của Postgres (`:799-818`) |
| Ước lượng < `accurateCountThreshold` (mặc định 1.000.000, `:729`) | Chạy `COUNT(*)` thật (`:774-789`) |

🧠 Đây là đánh đổi chính xác ↔ tốc độ. Màn hình hiển thị "khoảng 1,2 triệu kết quả" không cần đếm chính xác. Báo cáo gửi cơ quan quản lý thì phải dùng `_total=accurate`.

## 9. Hai bản hiện thực của cùng 1 ngôn ngữ search

✅ Medplum có **2 bộ máy** hiểu cùng cú pháp search FHIR:

| Bộ máy | Ở đâu | Dùng cho |
|---|---|---|
| Dịch sang SQL | `packages/server/src/fhir/search.ts` | API search, AccessPolicy `criteria` |
| So khớp trong bộ nhớ | `packages/core/src/search/match.ts:27` (`matchesSearchRequest`) | Subscription: "resource vừa ghi có khớp `criteria` không?" (`workers/subscription.ts:545`). Client cũng dùng được |

🧠 Hệ quả: 1 câu `criteria` có thể cho kết quả **khác nhau** giữa 2 bộ máy nếu 1 bên hỗ trợ modifier mà bên kia chưa. Khi viết Subscription hay AccessPolicy phức tạp, phải test cả 2 đường. Chi tiết ở module 2a.

## 10. Đối chiếu OpenEMR

✅ OpenEMR cũng có API search FHIR, nhưng **ánh xạ tay** từng tham số sang cột SQL. VD `src/Services/FHIR/FhirPatientService.php`:

```php
// :130
'name'      => new FhirSearchParameterDefinition('name', SearchFieldType::STRING, ['fname', 'mname', 'lname', 'title']),
// :131
'birthdate' => new FhirSearchParameterDefinition('birthdate', SearchFieldType::DATE, ['DOB']),
// :161
'family'    => new FhirSearchParameterDefinition('family', SearchFieldType::STRING, ['lname']),
```

Bộ dựng câu `WHERE` dùng chung ở `src/Services/Search/` (`FhirSearchWhereClauseBuilder.php`, `TokenSearchField.php`, `DateSearchField.php`…).

| | Medplum | OpenEMR |
|---|---|---|
| Tham số search | Sinh từ định nghĩa `SearchParameter` của spec, đủ mọi tham số | Khai báo tay trong từng `Fhir*Service.php`, chỉ những tham số đã map |
| Thêm tham số tuỳ biến | Tạo resource `SearchParameter` + reindex | Sửa code PHP |
| `name` khớp tên nào | Mọi tên trong mảng `name` (lookup table) | Chỉ 1 bộ `fname/mname/lname` |

🧠 Đây lại là triết lý "chuẩn trước" và "nghiệp vụ trước" ([`00` mục 5](./00-tong-quan-medplum.md#5-đối-chứng-openemr-khác-medplum-ở-tầng-kiến-trúc-ra-sao)), lần này thể hiện ở tầng search.

## Sai lầm thường gặp

1. **Thêm điều kiện lọc ở client sau khi lấy dữ liệu về.** VD lấy 20 Observation rồi lọc "> 140" bằng JavaScript. Kết quả: trang có thể trống trong khi trang sau có dữ liệu, và tổng đếm sai. Mọi điều kiện phải nằm trong câu search.
2. **Dùng `_count=1000` để "lấy hết".** Giới hạn cứng là 1000 (`DEFAULT_MAX_SEARCH_COUNT`). Đồng bộ hàng loạt phải dùng cursor hoặc Bulk `$export` (module 2d).
3. **Viết AccessPolicy `criteria` bằng chained search nhiều tầng.** Mỗi câu search của người dùng đó sẽ kéo thêm `EXISTS` nhiều tầng → chậm toàn hệ thống. Ưu tiên `compartment` hoặc tham số trực tiếp.
4. **Tin `total` là số chính xác.** Trên bảng lớn đó là ước lượng từ `EXPLAIN`, trừ khi yêu cầu `_total=accurate`.
5. **Tìm `name=nguyen` và kết luận "không có bệnh nhân".** Search tên của Medplum nhiều khả năng phân biệt dấu (xem [Phụ lục 1a-C](./1a-comparison-fhir-datastore.md#phụ-lục--hệ-quả-riêng-cho-dự-án-ở-việt-nam)).

## Tự kiểm tra

1. Câu `Patient?_has:Observation:subject:code=4548-4&_has:Observation:subject:value-quantity=gt9` có nghĩa giống "bệnh nhân có HbA1c > 9" không? Vì sao không? (Gợi ý: 2 điều kiện `_has` có thể khớp 2 Observation **khác nhau**.)
2. 1 user có AccessPolicy `Observation?subject.organization=Organization/an-binh`. Câu search `Observation?code=8480-6` của họ thành SQL có mấy tầng `EXISTS`?
3. Vì sao Medplum lấy `LIMIT count + 1` thay vì chạy `COUNT(*)`?
4. Muốn Subscription báo mọi Observation có mã thuộc ValueSet "tiểu đường". Vì sao không viết được `criteria = Observation?code:in=ValueSet/tieu-duong`, và nên viết thế nào?
