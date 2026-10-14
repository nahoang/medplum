# 1e-C — So sánh 6 cách lưu và truy vấn cây mã y tế

> **Loại file:** C. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Nguồn ✅ cho Medplum: [`1e-terminology.md`](./1e-terminology.md). Câu hỏi của file: *"Bảng mã có hàng trăm nghìn mã, xếp thành cây nhiều tầng, và 1 mã có thể có **nhiều cha**. Lưu thế nào để vừa nạp được, vừa hỏi nhanh 'mọi mã con của X'?"*

## Mục lục

- [Phần 1 — Bài toán: 6 đặc tính](#phần-1--bài-toán-6-đặc-tính)
- [Phần 2 — 6 ứng viên](#phần-2--6-ứng-viên)
- [Phần 3 — Bảng so sánh](#phần-3--bảng-so-sánh)
- [Phần 4 — Vì sao loại / khi nào đúng](#phần-4--vì-sao-loại--khi-nào-đúng)
- [Phần 5 — Lối tắt sai](#phần-5--lối-tắt-sai)
- [Phần 6 — Medplum và HAPI tại từng ngã rẽ](#phần-6--medplum-và-hapi-tại-từng-ngã-rẽ)
- [Phần 7 — Đánh đổi bằng số](#phần-7--đánh-đổi-bằng-số)
- [Phần 8 — Đối chứng nội bộ](#phần-8--đối-chứng-nội-bộ)
- [Phần 9 — Phản biện](#phần-9--phản-biện)
- [Phần 10 — Checklist cho MVP](#phần-10--checklist-cho-mvp)

Nhãn: ✅ code thật · 📘 chuẩn · 💭 phản thực tế / ước tính · 🧠 suy luận.

**Kịch bản cố định:** cây ICD-10 nhánh đái tháo đường, và 1 đoạn SNOMED có **2 cha**:

```
ICD-10:  E08-E13 (nhóm)
          └─ E11  Đái tháo đường type 2
               ├─ E11.2  …có biến chứng thận
               │    └─ E11.21 …bệnh thận do ĐTĐ
               └─ E11.9  …không biến chứng

SNOMED (💭 rút gọn): "Bệnh thận do ĐTĐ type 2" là con của CẢ
          "Đái tháo đường type 2"  VÀ  "Bệnh thận mạn"
```

---

## Phần 1 — Bài toán: 6 đặc tính

| # | Đặc tính | Vì sao |
|---|---|---|
| T1 | Hỏi **mọi mã con** (mọi tầng) của X nhanh | ValueSet `is-a E11`, báo cáo "mọi bệnh nhân đái tháo đường" |
| T2 | Hỏi **X có phải tổ tiên của Y** | `$subsumes`, kiểm tra mã nhập có hợp lệ với ràng buộc |
| T3 | **Đa cha** (polyhierarchy) | 📘 SNOMED CT cho phép 1 khái niệm có nhiều cha. ICD-10 thì không |
| T4 | Nạp lại cả bảng mã định kỳ | ICD, LOINC, SNOMED ra bản mới hằng năm hoặc nửa năm |
| T5 | Tìm theo tên, không dấu, nhiều ngôn ngữ | Ô gợi ý chẩn đoán |
| T6 | Thuộc tính mã khác nhau giữa các hệ | RxNorm `TTY`, LOINC `COMPONENT`, ICD "có thanh toán được không" |

**Quy mô (📘 + 💭 ước tính):**

| Hệ mã | Số mã | Cây |
|---|---|---|
| ICD-10-CM | khoảng 70.000 | Cây đơn cha, sâu khoảng 5–7 tầng |
| SNOMED CT | khoảng 350.000 khái niệm đang dùng | Đa cha, sâu tới khoảng 20–30 tầng |
| LOINC | khoảng 100.000 | Gần như không có cây `is-a` (docs Medplum `filtering-large-code-systems.md:17-27`) |

## Phần 2 — 6 ứng viên

### 2.1 Bảng phẳng gộp giá — OpenEMR ✅

`sql/database.sql:1125`, bảng `codes`: `code`, `code_type`, `code_text`, `fee`, `superbill`, `related_code`, `revenue_code`…

| code | code_type | code_text | fee |
|---|---|---|---|
| E11.9 | ICD10 | Type 2 diabetes without complications | — |
| 99213 | CPT4 | Office visit, est. patient | 120.00 |

- **T1, T2, T3:** không có cây, chỉ so khớp tiền tố chuỗi (`LIKE 'E11%'`). Cách này **tình cờ đúng với ICD-10** vì mã ICD mã hoá cây vào chính chuỗi. Sai với SNOMED, vì mã SNOMED là số không mang cấu trúc.
- Gộp giá vào mã → 1 mã 1 giá.

### 2.2 Danh sách kề + CTE đệ quy — Medplum ✅

`Coding` (1 dòng/mã) + `Coding_Property` (`property = parent`, `target` = id mã cha). Duyệt bằng `WITH RECURSIVE` lúc hỏi (`operations/utils/terminology.ts:190-225` đi lên, `:265-312` đi xuống).

`Coding_Property` (rút gọn):

| coding | property | target |
|---|---|---|
| E11.2 | parent | E11 |
| E11.21 | parent | E11.2 |
| E11.9 | parent | E11 |
| BệnhThậnĐTĐ2 | parent | ĐTĐ2 |
| BệnhThậnĐTĐ2 | parent | BệnhThậnMạn |

- **T3 tự nhiên:** đa cha chỉ là nhiều dòng.
- **T4 rẻ:** mỗi mã ghi đúng các cạnh cha trực tiếp.
- **T1:** chi phí tỉ lệ với **số mã con**, chạy lại **mỗi lần** hỏi.

### 2.3 Danh sách kề + tổ tiên tính sẵn — HAPI ✅

`TRM_CONCEPT_PC_LINK` (cạnh cha-con, `TermConceptParentChildLink.java:52-58`) **cộng** cột `PARENT_PIDS_VC` trên `TRM_CONCEPT`, chứa **mọi tổ tiên**, đánh index full-text (`TermConcept.java:226-233`). Tính lúc nạp (`:529-551`, hàm đệ quy `parentPids`).

| concept | PARENT_PIDS_VC |
|---|---|
| E11.21 | `E08-E13 E11 E11.2` |
| BệnhThậnĐTĐ2 | `… ĐTĐ2 … BệnhThậnMạn …` (gộp tổ tiên của cả 2 nhánh) |

- **T1, T2:** 1 truy vấn index "PARENT_PIDS chứa X", không đệ quy.
- **T3:** gộp tổ tiên mọi nhánh vào 1 tập, nên vẫn đúng.
- **T4:** phải tính lại tổ tiên cho mọi mã khi nạp. SNOMED sâu và đa cha thì tốn.

### 2.4 Closure table 💭

Bảng `(ancestor, descendant, depth)` chứa **mọi cặp** tổ tiên–con cháu.

| ancestor | descendant | depth |
|---|---|---|
| E11 | E11 | 0 |
| E11 | E11.2 | 1 |
| E11 | E11.21 | 2 |
| E11.2 | E11.21 | 1 |

- **T1, T2:** `WHERE ancestor = 'E11'` trên B-tree, nhanh nhất.
- **T3:** đúng.
- **Dung lượng:** số dòng ≈ số mã × độ sâu trung bình × hệ số đa cha. Xem Phần 7.

### 2.5 ValueSet mở rộng sẵn — HAPI ✅

Đây không phải cách lưu **cây**. Đây là lớp lưu **kết quả** `$expand`. Bảng `TRM_VALUESET_CONCEPT` (`TermValueSetConcept.java:66`), trạng thái `EXPANSION_STATUS` trên `TRM_VALUESET` (`TermValueSet.java:146`).

| valueset | code |
|---|---|
| VS/dtd | E11 |
| VS/dtd | E11.2 |
| VS/dtd | E11.21 |
| VS/dtd | E11.9 |

- **T1 cho ValueSet đã định nghĩa:** đọc thẳng, không tính.
- **Phải mở rộng lại** khi bảng mã hoặc ValueSet đổi. Cần job nền và trạng thái "đang mở rộng".

### 2.6 Terminology server bên ngoài 💭📘

Không lưu trong FHIR server. Gọi API sang 1 server chuyên dụng (VD Snowstorm cho SNOMED, hoặc dịch vụ thương mại).

- Mọi đặc tính tốt, do bên chuyên làm.
- **Phụ thuộc mạng và nhà cung cấp.** Mỗi lần validate mã là 1 lời gọi ra ngoài. Có thể vướng giấy phép SNOMED theo quốc gia.

## Phần 3 — Bảng so sánh

| Ứng viên | T1 con cháu | T2 tổ tiên | T3 đa cha | T4 nạp lại | T5 tìm tên | T6 thuộc tính |
|---|---|---|---|---|---|---|
| 2.1 Phẳng (OpenEMR) | ◐ chỉ ICD (tiền tố) | ◐ chỉ ICD | ✗ | ✅ | ◐ `LIKE` | ✗ cột cố định |
| 2.2 Kề + CTE (Medplum) | ◐ đệ quy mỗi lần | ◐ đệ quy | ✅ | ✅ **rẻ nhất** | ✅ trigram + unaccent | ✅ EAV |
| 2.3 Kề + tổ tiên sẵn (HAPI) | ✅ | ✅ | ✅ | ◐ tính tổ tiên | ✅ full-text | ✅ |
| 2.4 Closure | ✅ **nhanh nhất** | ✅ | ✅ | ✗ đắt nhất | — | — |
| 2.5 Mở rộng sẵn (HAPI) | ✅ cho VS cố định | — | — | ✗ mở rộng lại | ✅ | — |
| 2.6 Server ngoài | ✅ | ✅ | ✅ | ✅ (của bên khác) | ✅ | ✅ |

## Phần 4 — Vì sao loại / khi nào đúng

| Ứng viên | Khi nào đúng | Khi nào sai |
|---|---|---|
| 2.1 | Phòng khám chỉ dùng ICD-10 + mã dịch vụ, cần gắn giá | Có SNOMED, có nhiều payer |
| 2.2 | Nền tảng nhiều tenant, nhiều hệ mã nạp/xoá thường xuyên, ít truy vấn cây sâu | Truy vấn "mọi con cháu" của nút gần gốc SNOMED mỗi giây |
| 2.3 | Server chuyên về terminology, truy vấn cây dày đặc | Nạp lại thường xuyên |
| 2.4 | Cây ổn định, truy vấn cây cực nhiều | Bảng mã lớn đa cha (dung lượng) |
| 2.5 | ValueSet dùng đi dùng lại (ô nhập liệu cố định) | ValueSet tạo động theo từng truy vấn |
| 2.6 | Dùng SNOMED đầy đủ, có ngân sách | MVP, offline, hoặc chỉ dùng ICD-10 |

🧠 Không ứng viên nào thắng tuyệt đối. Khác biệt giữa Medplum (2.2) và HAPI (2.3 + 2.5) là **chọn trả giá lúc nạp hay lúc đọc**, đúng mô thức đã thấy ở [1a-C](./1a-comparison-fhir-datastore.md#phần-7--ma-trận-đánh-đổi-có-số) và [1c-C](./1c-comparison-phan-quyen.md#phần-7--ma-trận-đánh-đổi).

## Phần 5 — Lối tắt sai

**A — *"Mã ICD có cấu trúc, cứ `LIKE 'E11%'` là ra con cháu."*** Đúng với ICD-10 ở hầu hết trường hợp. Nhưng áp cùng logic cho SNOMED (`LIKE '44054%'`) thì sai hoàn toàn, vì mã SNOMED là định danh số không mang cây. Ngay với ICD, các phiên bản quốc gia có thể thêm mã mở rộng không theo quy tắc tiền tố.

**B — *"Materialized path `/E08-E13/E11/E11.2/` cho nhanh."*** Vỡ ở T3: 1 mã đa cha có 2 đường dẫn, không biết lưu đường nào. HAPI tránh được vì lưu **tập** tổ tiên (2.3), không phải **1 đường dẫn**.

**C — *"Mở rộng sẵn mọi ValueSet cho nhanh."*** Mỗi project tạo ValueSet riêng. Bảng mã cập nhật thì mọi bản mở rộng cũ thành sai. Không có trạng thái "đang mở rộng lại" thì người dùng thấy danh sách thiếu mà không ai biết.

**D (leo quá xa) — *"MVP cần nạp đủ SNOMED + server terminology riêng."*** Nếu MVP chỉ cần ICD-10 cho chẩn đoán và 1 danh mục dịch vụ nội bộ, 1 `CodeSystem` + vài `ValueSet` liệt kê tay là đủ. SNOMED còn kéo theo vấn đề giấy phép.

## Phần 6 — Medplum và HAPI tại từng ngã rẽ

| Ngã rẽ | Medplum | HAPI |
|---|---|---|
| Lưu cây | Cạnh cha trực tiếp trong bảng EAV chung `Coding_Property` (✅ `migrate.ts:784-810`) | Bảng cạnh riêng `TRM_CONCEPT_PC_LINK` |
| Tăng tốc con cháu | Không tính sẵn. CTE đệ quy (✅ `terminology.ts:265-312`) | Tính sẵn tập tổ tiên `PARENT_PIDS_VC` |
| ValueSet | Mở rộng lúc gọi, tối đa 1000 (✅ `expand.ts:117`) | Có pre-expansion |
| Thuộc tính mã | EAV `Coding_Property` | `TRM_CONCEPT_PROPERTY` |
| Tìm tên | Trigram + `medplum_unaccent` (✅ `migrate.ts:765-773`) | Full-text (Lucene/Elastic qua Hibernate Search) |
| Search resource theo cây / ValueSet (`code:below`, `code:in`) | **Không hỗ trợ** (✅ `token-column.ts:240-255`) | ✅ Hỗ trợ: server tự `$expand` ValueSet hoặc tìm mã con, rồi đổi thành danh sách mã để so khớp (`hapi-fhir-jpaserver-base/.../search/builder/predicate/BaseTokenPredicateBuilder.java:185-206`). Chính là cách làm thay đề xuất cho Medplum, nhưng chạy phía server |

🧠 Hình mẫu chung: Medplum là **nền tảng ứng dụng**, terminology là tính năng phụ trợ, giữ đơn giản. HAPI là **FHIR server tham chiếu**, terminology là năng lực lõi, đầu tư sâu.

## Phần 7 — Đánh đổi bằng số

💭 Ước tính cho SNOMED, khoảng 350.000 khái niệm, độ sâu trung bình khoảng 10, hệ số đa cha khoảng 1,5:

| Cách | Số dòng thêm | Ghi chú |
|---|---|---|
| 2.2 Kề | khoảng 350k × 1,5 ≈ **525 nghìn** cạnh | Rẻ nhất khi nạp |
| 2.3 Tổ tiên sẵn | 350k chuỗi, mỗi chuỗi khoảng 15–30 id | Dung lượng vừa, cần full-text index |
| 2.4 Closure | khoảng 350k × 10 × 1,5 ≈ **5 triệu** dòng | Lớn nhất, nhưng mọi truy vấn là B-tree |

Truy vấn "mọi con cháu của *Clinical finding*" (nút gần gốc, khoảng 100k con cháu):
- **2.2:** CTE đệ quy đi khoảng 10 tầng, mỗi tầng 1 lần JOIN.
- **2.3 / 2.4:** 1 lần đọc index.

Nhưng **Medplum giới hạn `$expand` 1000 kết quả**. Với ô gợi ý thực tế (người dùng gõ chữ, lọc thêm), CTE chỉ cần đi tới khi đủ kết quả. Chi phí đệ quy chỉ đau khi cần **toàn bộ** cây con. Đó là việc báo cáo, nên làm ở kho phân tích, không làm ở API giao dịch.

## Phần 8 — Đối chứng nội bộ

Ngay trong Medplum, phần terminology **dùng EAV** (`Coding_Property`), trong khi phần dữ liệu lâm sàng ở 1a **tránh** EAV. 🧠 Đây là cùng 1 nguyên tắc chọn kỹ thuật theo đặc tính dữ liệu:

| | Dữ liệu lâm sàng (1a) | Thuộc tính mã (1e) |
|---|---|---|
| Cần so sánh khoảng (`> 140`)? | Có | Không |
| Tập thuộc tính có cố định theo loại không? | Có (do SearchParameter quy định) | Không (mỗi hệ mã 1 kiểu) |
| → Kỹ thuật | Cột có kiểu | EAV chuỗi |

Khớp hoàn toàn với Phần 10 của bài Saleor: *"có cần giữ kiểu gốc để lọc khoảng? Không → EAV thuần đủ."*

## Phần 9 — Phản biện

**1 — Kiến trúc sư dữ liệu:** *"Closure table là chuẩn mực cho cây trong SQL, sao Medplum dùng CTE?"*

<details><summary>Gợi ý</summary>

Closure tối ưu đọc, nhưng nạp SNOMED thì sinh khoảng 5 triệu dòng mỗi lần cập nhật phiên bản (Phần 7). Medplum cho mỗi project tự nạp CodeSystem riêng, nên chi phí nạp nhân lên theo số project. Với ô gợi ý giới hạn 1000 kết quả, CTE đủ nhanh. Câu hỏi đúng là: tần suất nạp so với tần suất truy vấn cây toàn phần là bao nhiêu?
</details>

**2 — Bác sĩ trưởng:** *"Tôi muốn lọc 'mọi bệnh nhân có bất kỳ chẩn đoán đái tháo đường nào' ngay trên màn hình danh sách."*

<details><summary>Gợi ý</summary>

Medplum hiện không hỗ trợ `code:below=E11` khi search. 2 cách:
- **Lúc đọc:** `$expand` ValueSet ĐTĐ rồi search `Condition?code=…` theo lô.
- **Lúc ghi:** Bot gắn thêm 1 `category` "dtd" vào mọi Condition có mã thuộc nhóm ĐTĐ. Search 1 tham số là xong.

Cách lúc ghi nhanh hơn khi đọc, đúng mô thức "chép nhãn lúc ghi" của compartment ở 1c.
</details>

**3 — Pháp chế:** *"Dùng SNOMED có cần giấy phép không?"*

<details><summary>Gợi ý</summary>

📘 SNOMED CT cấp phép qua SNOMED International và các nước thành viên. Quyền dùng ở 1 nước phụ thuộc nước đó có là thành viên hay không. Đây là câu hỏi pháp lý cần kiểm tra cho Việt Nam trước khi nạp, không phải câu hỏi kỹ thuật. ICD-10 (WHO) và LOINC có điều khoản sử dụng riêng, thường dễ hơn.
</details>

## Phần 10 — Checklist cho MVP

| Câu hỏi | Không → | Có → |
|---|---|---|
| Cần SNOMED (đa cha)? | ICD-10 + danh mục nội bộ. Kề + CTE (Medplum) thừa đủ | (xuống) |
| Truy vấn toàn bộ cây con xảy ra trên API giao dịch thường xuyên? | Kề + CTE | Tổ tiên tính sẵn (HAPI) hoặc server ngoài |
| Cần search resource theo nhóm mã (`:below`, `:in`)? | (bỏ qua) | Medplum: chép nhãn lúc ghi bằng Bot. Hoặc chọn server hỗ trợ |
| Giá dịch vụ khác nhau theo payer? | Gộp giá vào mã như OpenEMR được | Tách: CodeSystem + ChargeItemDefinition (module 4b) |
| Có ngân sách, giấy phép cho terminology server? | Tự nạp vào FHIR server | Server ngoài |
