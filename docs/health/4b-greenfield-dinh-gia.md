# 4b-G — Thiết kế từ đầu: định giá dịch vụ & ghi nhận phí

> **Loại file:** G (greenfield).
>
> Trả lời câu: *"Cùng 1 lần khám, giá khác nhau theo payer, theo hợp đồng, theo mã CPT. Tính phí ở đâu, lúc nào, và con số nào mới là doanh thu?"*
>
> Đọc tiếp: [`4b-dinh-gia-ghi-phi.md`](./4b-dinh-gia-ghi-phi.md) (R) · [`4b-comparison-dinh-gia.md`](./4b-comparison-dinh-gia.md) (C).
>
> **Lưu ý trung thực:** viết sau khi đã đọc `chargeitemdefinitionapply.ts`. Che Phần 5 khi tự luyện.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bối cảnh giả định](#phần-1--bối-cảnh-giả-định-)
- [Phần 2 — BA view](#phần-2--ba-view)
- [Phần 3 — Mô hình khái niệm](#phần-3--mô-hình-khái-niệm-)
- [Phần 4 — Developer view: V1, V2, V3](#phần-4--developer-view-v1-v2-v3-)
- [Phần 5 — Đối chiếu thực tế](#phần-5--đối-chiếu-thực-tế-)
- [Phần 6 — Bảng so sánh tổng hợp](#phần-6--bảng-so-sánh-tổng-hợp)
- [Phần 7 — Đánh giá & luyện phản biện](#phần-7--đánh-giá--luyện-phản-biện-)
- [Phần 8 — Checklist tái sử dụng](#phần-8--checklist-tái-sử-dụng)

## Quy tắc nhãn nguồn

💭 bối cảnh, số liệu, V1–V3 · ✅ code thật · 📘 chuẩn, quy định · 🧠 suy luận.

---

## Phần 1 — Bối cảnh giả định 💭

> **Giám đốc tài chính An Bình (Mỹ)** nói: *"Bảng giá của mình ghi khám tái khám 200 đô. Nhưng Medicare chỉ trả 110, hợp đồng với Blue Cross là 140, bệnh nhân tự trả được giảm 40%. Kế toán ghi doanh thu 200 rồi cuối tháng điều chỉnh bằng tay, không ai biết mỗi payer thực sự trả bao nhiêu phần trăm. Mình cần 1 chỗ tính giá đúng ngay từ lúc ghi phí."*

Kịch bản: lần tái khám của bà Lan (Medicare chính) có 2 dòng phí:

| Dịch vụ | Mã | Giá niêm yết 💭 |
|---|---|---|
| Khám tái khám, mức độ vừa | CPT `99214` | 200$ |
| Xét nghiệm HbA1c | CPT `83036` | 40$ |

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

1. *Ai nói?* → Giám đốc tài chính.
2. *Ai thao tác hằng ngày?* → **Bác sĩ** (chọn mã dịch vụ), **coder** (kiểm tra mã), nhân viên billing.
3. *Ai nhận kết quả?* → Kế toán (doanh thu), payer (claim), bệnh nhân (hoá đơn).
4. *Ai trả tiền?* → Payer theo **hợp đồng**, bệnh nhân theo **phần còn lại**.
5. *Ai kiểm tra?* → Payer (kiểm tra mã có khớp chẩn đoán), **CMS** (quy định minh bạch giá), kiểm toán.

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Bác sĩ | Chọn mã nhanh | Không quan tâm giá, nhưng **chọn mã chính là định giá** |
| Coder | Mã khớp hồ sơ, có modifier đúng | Vai trò chuyên biệt, thường thuê ngoài |
| Kế toán | Doanh thu **thực nhận**, không phải niêm yết | Hệ thống lâm sàng hay chỉ có giá niêm yết |
| Người phụ trách hợp đồng payer | Bảng giá từng payer, hiệu lực từ ngày nào | Hợp đồng nằm ở file PDF, không ở hệ thống |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | Mỗi dịch vụ có giá niêm yết | Giá theo mã CPT, hay theo mã + modifier? |
| 2 | Tường minh | Mỗi payer trả theo hợp đồng | Hợp đồng là **bảng giá riêng** hay **% của Medicare**? |
| 3 | Tường minh | Tự trả được giảm giá | Giảm cố định hay theo thu nhập (sliding scale)? |
| 4 | Ẩn | Giá **đóng băng** tại ngày dịch vụ | Bảng giá đổi ngày 1/1, lần khám 31/12 lập claim ngày 3/1 thì dùng giá nào? |
| 5 | Ẩn | Số lượng (units) nhân giá | Tiêm 3 mũi, 15 phút × 4 |
| 6 | Ẩn | Chênh lệch niêm yết − hợp đồng là **điều chỉnh hợp đồng** (contractual adjustment), không phải nợ khó đòi | Ghi sổ thế nào? |
| 7 | Ẩn | Modifier đổi giá (VD `-25`, `-59`, `-50` hai bên) | Ai được thêm modifier? |
| 8 | Ẩn | Phải công bố giá (bệnh viện Mỹ) | Phạm vi áp dụng cho phòng khám? |

**Cách tìm rule ẩn #4:** hỏi *"bảng giá năm sau áp dụng cho lần khám nào?"* Câu trả lời đúng gần như luôn là **theo ngày dịch vụ**, không theo ngày lập hoá đơn.

**Cách tìm rule ẩn #6:** hỏi kế toán *"200 trừ 110 bằng 90, 90 đô đó đi đâu?"* Không phải nợ của bệnh nhân (Medicare cấm thu thêm phần vượt hợp đồng với nhà cung cấp tham gia). Đó là **khoản giảm doanh thu theo hợp đồng**.

📘 Rule #8: Mỹ có quy định **Hospital Price Transparency** (từ 2021) buộc bệnh viện công bố giá chuẩn, gồm giá thương lượng theo từng payer. 🧠 Áp cho bệnh viện, không áp mọi phòng khám, nhưng là lý do giá theo payer phải nằm trong **dữ liệu**, không nằm trong PDF.

### Bước 3 — 7 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh |
|---|---|---|
| Q1 | Có bao nhiêu payer, bao nhiêu hợp đồng? | Bảng giá riêng từng payer hay công thức |
| Q2 | Hợp đồng dạng gì: danh sách giá, % Medicare, gói theo lần khám? | Mô hình giá: bảng, hệ số, hay gói |
| Q3 | Tính giá lúc nào: lúc ghi phí, lúc lập claim, hay lúc payer trả? | Giá **dự kiến** vs giá **thực nhận** |
| Q4 | Bảng giá đổi bao lâu 1 lần? | Phiên bản theo thời gian |
| Q5 | Có giảm giá tự trả / theo thu nhập không? | Điều kiện theo bệnh nhân |
| Q6 | Ai được sửa giá tay? Có cần duyệt? | Ghi đè có kiểm soát |
| Q7 | Kế toán cần báo cáo gì? | Giá niêm yết, dự kiến, thực nhận, điều chỉnh |

### Sai lầm thường gặp khi làm BA view

1. **Chỉ hỏi "giá dịch vụ là bao nhiêu".** Có ít nhất 3 con số: niêm yết, payer cho phép (allowed), payer thực trả. Phần bệnh nhân là phần chênh giữa con số thứ 2 và thứ 3.
2. **Không hỏi Q3.** Tính lại giá mỗi lần mở màn hình thì ngày bảng giá đổi, toàn bộ lần khám cũ đổi theo.
3. **Coi giảm giá tự trả là "khuyến mãi".** Ở Mỹ có ràng buộc pháp lý (không được giảm giá để lôi kéo bệnh nhân Medicare). Ghi rõ lý do giảm.

---

## Phần 3 — Mô hình khái niệm 💭

| Thực thể | Vì sao là thực thể thật |
|---|---|
| **Mã dịch vụ** (CPT/HCPCS) | Danh mục quốc gia, có phiên bản theo năm ([1e](./1e-terminology.md)) |
| **Định nghĩa giá** cho 1 mã | Giá cơ sở + các điều chỉnh + điều kiện áp dụng + hiệu lực |
| **Hợp đồng payer** | Bảng giá cho phép theo mã, hoặc hệ số trên 1 bảng gốc, có hiệu lực |
| **Dòng phí** (charge) | 1 dịch vụ cụ thể cho 1 bệnh nhân trong 1 lần khám: mã, số lượng, **giá tại thời điểm**, liên kết chẩn đoán |
| **Tài khoản** (account) | Nơi gom phí của 1 bệnh nhân / 1 đợt điều trị, có người chịu trách nhiệm trả (guarantor) và danh sách bảo hiểm |
| **Điều chỉnh** | Chênh lệch có lý do: hợp đồng, giảm giá tự trả, xoá nợ |

**Phân biệt Định nghĩa giá và Dòng phí:** cùng mẫu "definition → instance" ở [3d](./3d-careplan-task.md#1-5-resource-2-tầng). Định nghĩa đổi theo năm, dòng phí phải **giữ nguyên** giá lúc phát sinh.

**Cầu nối FHIR:**

| Khái niệm | FHIR R4 | Lớp quyết định |
|---|---|---|
| Định nghĩa giá | `ChargeItemDefinition` (`propertyGroup[].applicability` + `priceComponent[]` type `base`/`surcharge`/`discount`/`tax`/`informational`/`deduction`) | HL7 |
| Dòng phí | `ChargeItem` (`code`, `quantity`, `context` = Encounter, `priceOverride`, `factorOverride`, `definitionCanonical`, `account`) | HL7 |
| Tài khoản | `Account` (`subject`, `guarantor`, `coverage`) | HL7 |
| Áp định nghĩa vào dòng phí | `ChargeItemDefinition/$apply` | HL7 định nghĩa, **hiện thực Medplum** |
| Hợp đồng payer | **Không có resource chuyên dụng** trong R4. Có thể là `ChargeItemDefinition` có điều kiện theo payer, hoặc `Contract` | Ứng dụng tự chọn |
| Điều chỉnh, số tiền payer trả | `ClaimResponse.item.adjudication`, `ExplanationOfBenefit` | HL7 ([4c](./4c-claim.md)) |

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — Giá trên bảng mã dịch vụ

```sql
CREATE TABLE service_code (code TEXT PRIMARY KEY, name TEXT, fee NUMERIC);
CREATE TABLE charge (id, encounter_id, code, units, fee NUMERIC);  -- chép fee lúc ghi
```

| Dòng phí bà Lan | fee |
|---|---|
| 99214 | 200 |
| 83036 | 40 |
| **Tổng** | **240** |

Giá payer, điều chỉnh: kế toán xử lý ngoài hệ thống khi nhận 835.

**Dừng ở V1 khi nào:** đa số khách tự trả, hoặc chỉ 1–2 payer.

### V2 — Ma trận giá: mã × mức giá

```sql
CREATE TABLE price (code TEXT, level TEXT, price NUMERIC, PRIMARY KEY (code, level));
-- level: 'standard', 'medicare', 'bluecross', 'selfpay'
```

| code | standard | medicare | bluecross | selfpay |
|---|---|---|---|---|
| 99214 | 200 | 110 | 140 | 120 |
| 83036 | 40 | 13 | 25 | 24 |

Dòng phí bà Lan (level = medicare): 110 + 13 = **123** dự kiến.

Điểm mù:
- **Bùng nổ ô:** 10.000 mã × 20 payer = 200.000 ô, đa số là "% Medicare". Hợp đồng "Blue Cross = 127% Medicare" phải chép thành 10.000 con số.
- **Không có hiệu lực theo thời gian**: bảng giá năm sau ghi đè năm nay.
- Level gắn với **bệnh nhân** hay với **bảo hiểm**? Bệnh nhân đổi bảo hiểm giữa năm thì sao?

### V3 — Định nghĩa giá có điều kiện + giá đóng băng trên dòng phí + 2 lớp số tiền

1. **Định nghĩa giá** cho mỗi mã: giá cơ sở (niêm yết) + các nhóm điều chỉnh có **điều kiện** (payer = Medicare → hệ số; tự trả → giảm 40%), có **hiệu lực**.
2. **Lúc ghi phí**: áp định nghĩa **có hiệu lực tại ngày dịch vụ**, ghi giá vào dòng phí, kèm phiên bản định nghĩa đã dùng. **Không tính lại** khi đọc.
3. **2 lớp số tiền** trên dòng phí: giá niêm yết (gửi trên claim) và giá **dự kiến** payer cho phép (để ước tính phần bệnh nhân, so với 835).
4. **Khi nhận 835**: ghi số tiền payer thực cho phép/trả. Chênh lệch niêm yết − cho phép = điều chỉnh hợp đồng. Chênh lệch dự kiến − thực tế = **tín hiệu hợp đồng bị payer trả sai**.
5. Sửa giá tay: ghi đè có lý do, có người duyệt.

**Dừng ở V3 khi nào:** nhiều payer, cần đo doanh thu thực theo payer, cần phát hiện payer trả thiếu.

### 3 lối tắt "nghe hợp lý nhưng sai"

1. **"Gửi claim với giá đã giảm theo hợp đồng."** Sai cách làm phổ biến ở Mỹ: claim thường gửi giá **niêm yết**, payer tự áp hợp đồng. Gửi giá thấp hơn có thể làm payer trả thấp hơn cả hợp đồng.
2. **"Tính giá lúc hiển thị, khỏi lưu."** Giá trôi theo bảng giá mới, báo cáo doanh thu tháng cũ thay đổi.
3. **"Mỗi payer 1 bảng giá đầy đủ."** Bùng nổ dữ liệu (V2), trong khi hợp đồng thật thường là công thức.

---

## Phần 5 — Đối chiếu thực tế ✅

Chi tiết: [`4b-dinh-gia-ghi-phi.md`](./4b-dinh-gia-ghi-phi.md).

| Thiết kế V3 | Medplum | ✅ |
|---|---|---|
| Định nghĩa giá có điều kiện | `ChargeItemDefinition/$apply`: tìm giá `base` ở nhóm thoả điều kiện, cộng `surcharge`, trừ `discount` (số tiền hoặc hệ số × giá cơ sở) | `packages/server/src/fhir/operations/chargeitemdefinitionapply.ts:49-124` |
| Điều kiện | Biểu thức FHIRPath **trên chính ChargeItem** | `:53-59` |
| Ghi giá vào dòng phí | `ChargeItem.priceOverride` | `:123-126` |
| Nhân số lượng | **Không** | toàn hàm |
| `tax`, `deduction`, `informational` | **Bỏ qua** | `:92-120` |
| Hiệu lực theo thời gian | Không kiểm tra `effectivePeriod` | toàn hàm |
| **Không tính lại khi đọc** | App Provider **tính lại mỗi lần mở tab Billing** | `examples/medplum-provider/src/utils/chargeitems.ts:44-57`, `components/encounter/BillingTab.tsx:82` |

📘/✅ Lưu ý tên gọi: `$set-accounts` **không** thuộc bài này. Nó ghi `meta.accounts` để **phân quyền theo compartment** ([1c](./1c-access-control.md)), không đụng tới resource tài chính `Account` (`packages/server/src/fhir/operations/set-accounts.ts`, ghi `meta.accounts` ở `:162-166`; `repo.ts:2074-2080` biến nó thành compartment).

OpenEMR (commit `ebe3ee8`) ở **V2**:
- `codes.fee` (`sql/database.sql:1125`): giá trên mã.
- `prices(pr_id, pr_selector, pr_level, pr_price)` (`:8776`): ma trận mã × mức giá. Mức giá (`pricelevel`) gắn trên **hồ sơ bệnh nhân** (layout `DEM`, `:3714`).
- `billing` (`:246`): mỗi dòng phí **chép `fee`, `units`, `modifier`, `justify`** (liên kết chẩn đoán), `payer_id`, `billed`. Giá được **đóng băng** lúc ghi.

---

## Phần 6 — Bảng so sánh tổng hợp

| | V1 | V2 / OpenEMR | V3 thiết kế | Medplum thật |
|---|---|---|---|---|
| Giá theo payer | ✘ | ✔ (ma trận) | ✔ (điều kiện) | ✔ nếu ChargeItem mang thông tin payer |
| Hệ số (% Medicare) | ✘ | ✘ | ✔ | ✔ (`factor`) |
| Hiệu lực theo thời gian | ✘ | ✘ | ✔ | ✘ |
| Số lượng | ✔ | ✔ (`units`) | ✔ | ✘ |
| Giá đóng băng | ✔ | ✔ | ✔ | Server: ✔ (ghi 1 lần). App mẫu: ✘ (tính lại khi đọc) |
| Giá niêm yết + dự kiến | ✘ | ✘ | ✔ | ✘ (1 trường `priceOverride`) |

---

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Vì sao có khoảng cách V2 → V3:** V2 trả lời "giá là bao nhiêu", V3 trả lời "**giá theo quy tắc nào, tại thời điểm nào, và khác thực nhận bao nhiêu**". Câu sau là câu của người làm tài chính: đo **tỉ lệ thực thu** (net collection rate) theo payer, phát hiện payer trả thiếu hợp đồng.

**Góc nhìn kinh doanh:**
- Luồng tiền lần khám bà Lan 💭: niêm yết 240$ → Medicare cho phép 123$ → Medicare trả 80% = 98,40$ → bà Lan (hoặc Medigap) trả 20% = 24,60$ → điều chỉnh hợp đồng 117$.
- Điểm rò rỉ doanh thu:
  - **under-coding** (bác sĩ chọn `99213` thay vì `99214` "cho an toàn")
  - **quên ghi phí** (xét nghiệm làm tại chỗ không có dòng phí)
  - **payer trả thiếu hợp đồng** mà không ai so
  - **không thu phần bệnh nhân**
- Chỉ số cần có: doanh thu niêm yết, allowed, thực thu, điều chỉnh hợp đồng, tỉ lệ thực thu = thực thu / allowed.

**Khi nào KHÔNG cần V3:** khách tự trả là chính (spa, thẩm mỹ, phòng khám tư ở Việt Nam ngoài BHYT): V1 với bảng giá có hiệu lực là đủ.

### Luyện phản biện

**Phản đối 1 (tài chính):** "Ghi doanh thu theo giá niêm yết rồi điều chỉnh cuối kỳ là cách kế toán vẫn làm, sao phải đổi?"

<details><summary>Đáp án gợi ý</summary>

Về kế toán có thể vẫn đúng (doanh thu ghi theo giá dự kiến thu được, điều chỉnh là khoản giảm trừ). Vấn đề là **vận hành**: không có giá dự kiến theo hợp đồng ở dòng phí thì không so được với 835, không phát hiện được payer trả thiếu. Tỉ lệ trả thiếu chỉ vài % nhưng trên toàn doanh thu là con số lớn.
</details>

**Phản đối 2 (kỹ thuật):** "`$apply` của Medplum đã có, cứ gọi lại mỗi lần cần là luôn đúng giá mới nhất."

<details><summary>Đáp án gợi ý</summary>

"Giá mới nhất" là **sai** cho lần khám cũ. Claim đã gửi với giá cũ, tính lại thì dòng phí khác claim, đối soát 835 lệch. Chỉ gọi `$apply` **1 lần** lúc ghi phí, sau đó khoá dòng phí (status `billable` → `billed`).
</details>

**Phản đối 3 (pháp lý):** "Điều kiện theo payer nằm trong FHIRPath trên ChargeItem, vậy ai sửa ChargeItem là đổi được giá."

<details><summary>Đáp án gợi ý</summary>

Đúng, nên phải: (1) Bot gán thông tin payer lên ChargeItem từ `Coverage` thật, không để người nhập; (2) AccessPolicy không cho bác sĩ sửa `priceOverride`; (3) mọi thay đổi có `_history` ([1f](./1f-history-audit-deletion.md)) để kiểm toán.
</details>

---

## Phần 8 — Checklist tái sử dụng

- [ ] Phân biệt 3 con số: niêm yết, payer cho phép, thực trả
- [ ] Định nghĩa giá có hiệu lực theo thời gian
- [ ] Giá áp theo **ngày dịch vụ**, đóng băng trên dòng phí
- [ ] Số lượng × đơn giá
- [ ] Hợp đồng dạng công thức (% bảng gốc), không chép ma trận
- [ ] Ghi đè giá tay có lý do và người duyệt
- [ ] Có báo cáo thực thu / allowed theo payer
- [ ] Thông tin payer trên dòng phí lấy từ `Coverage`, không nhập tay

**Tự luyện tiếp theo:** mô hình hợp đồng "Blue Cross trả 127% Medicare Physician Fee Schedule năm hiện hành" bằng `ChargeItemDefinition`. Cần những gì mà `$apply` hiện tại chưa có?

**Nguồn:**
- 📘 FHIR R4 ChargeItemDefinition `$apply`: https://www.hl7.org/fhir/chargeitemdefinition-operation-apply.html
- 📘 CMS Hospital Price Transparency: https://www.cms.gov/priorities/key-initiatives/hospital-price-transparency
