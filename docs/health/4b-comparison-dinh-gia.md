# 4b-C — So sánh 5 cách tổ chức giá dịch vụ y tế

> **Loại file:** C (theo [`../templates/saleor-design-technique-comparison-product-attribute.md`](../templates/saleor-design-technique-comparison-product-attribute.md)).
>
> Trả lời câu: *"Có 5 cách lưu giá dịch vụ cho nhiều payer. Mỗi cách chịu được loại hợp đồng nào, sai ở đâu, và Medplum (`ChargeItemDefinition`) đứng ở chỗ nào?"*
>
> Khác 2 file cùng module: [G](./4b-greenfield-dinh-gia.md) đi từ nghiệp vụ, [R](./4b-dinh-gia-ghi-phi.md) đọc code. File này đặt các cách cạnh nhau trên cùng 1 kịch bản. Đối chiếu bài Pricing bên Saleor (e-commerce): ở cuối file.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bài toán: 8 đặc tính](#phần-1--bài-toán-8-đặc-tính)
- [Phần 2 — 5 ứng viên](#phần-2--5-ứng-viên)
- [Phần 3 — Bảng so sánh trực diện](#phần-3--bảng-so-sánh-trực-diện)
- [Phần 4 — Vì sao không có ứng viên nào thắng tuyệt đối](#phần-4--vì-sao-không-có-ứng-viên-nào-thắng-tuyệt-đối)
- [Phần 5 — 4 lối tắt nghe hợp lý nhưng sai](#phần-5--4-lối-tắt-nghe-hợp-lý-nhưng-sai)
- [Phần 6 — Medplum tại từng ngã rẽ](#phần-6--medplum-tại-từng-ngã-rẽ-)
- [Phần 7 — Đánh đổi đo bằng số](#phần-7--đánh-đổi-đo-bằng-số-)
- [Phần 8 — Đối chứng: chỗ không cần định giá phức tạp](#phần-8--đối-chứng-chỗ-không-cần-định-giá-phức-tạp)
- [Phần 9 — Luyện phản biện](#phần-9--luyện-phản-biện)
- [Phần 10 — Checklist · So với Saleor · Nguồn](#phần-10--checklist--so-với-saleor--nguồn)

## Quy tắc nhãn nguồn

✅ code thật `file:line` · 📘 chuẩn/quy định · 💭 code minh hoạ, số liệu giả định · 🧠 suy luận.

**Kịch bản xuyên suốt 💭:** phòng khám có **2.000 mã** dịch vụ đang dùng, **8 payer**. Bà Lan, CPT `99214`, niêm yết 200$. Hợp đồng:

| Payer | Hợp đồng |
|---|---|
| Medicare | Theo biểu phí Medicare (MPFS) năm hiện hành: 110$ |
| Blue Cross | **127% MPFS**: 139,70$ |
| Aetna | Danh sách giá riêng cho 300 mã thường dùng, mã khác 60% niêm yết |
| Tự trả | Giảm 40%: 120$ |

Ngày 1/1 năm sau: Medicare công bố MPFS mới (`99214` = 113$), phòng khám tăng niêm yết 5%.

---

## Phần 1 — Bài toán: 8 đặc tính

| # | Đặc tính | Vì sao | Hậu quả nếu bỏ qua 💭 |
|---|---|---|---|
| P1 | Cùng mã, giá khác theo payer | Hợp đồng riêng từng payer | Ước tính phần bệnh nhân sai, không phát hiện payer trả thiếu |
| P2 | Hợp đồng thường là **công thức** (% bảng gốc) | Payer thương lượng hệ số, không thương lượng 10.000 con số | Chép ma trận: 2.000 × 8 = 16.000 ô, đổi MPFS là sửa 2.000 ô Blue Cross |
| P3 | Có **ngoại lệ** trong công thức (300 mã Aetna giá riêng) | Mã thường dùng được thương lượng kỹ | Công thức thuần không biểu diễn được |
| P4 | **Hiệu lực theo thời gian**, áp theo ngày dịch vụ | Bảng giá, MPFS đổi hằng năm | Lần khám 31/12 lập claim 3/1 bị tính giá mới |
| P5 | Số lượng, modifier | Tiêm 3 mũi, `-50` hai bên = 150% | Thu thiếu |
| P6 | **Đóng băng** giá trên dòng phí | Claim đã gửi phải khớp dòng phí | Đối soát 835 lệch |
| P7 | Điều kiện theo bệnh nhân (tự trả, theo thu nhập) | Chính sách giảm giá | Giảm sai đối tượng (rủi ro pháp lý với Medicare) |
| P8 | Pháp lý: **công bố giá** theo payer (bệnh viện Mỹ) | CMS Hospital Price Transparency | Phải xuất được "giá thương lượng từng payer cho từng mã" |

---

## Phần 2 — 5 ứng viên

### Ứng viên 1 — Giá phẳng trên mã (1 cột) 💭/✅

```sql
CREATE TABLE codes (code TEXT, fee NUMERIC);  -- OpenEMR: codes.fee (sql/database.sql:1125)
```

| Kịch bản | Kết quả |
|---|---|
| Bà Lan, Medicare | 200$ (niêm yết). Giá cho phép: không biết |
| P1–P3 | ✘ |

**Sai lầm thường gặp:** dùng cột `fee` như giá payer cho phép, rồi ngạc nhiên vì 835 trả 110.

### Ứng viên 2 — Ma trận mã × mức giá (OpenEMR) ✅

✅ OpenEMR `prices(pr_id, pr_selector, pr_level, pr_price)` (`sql/database.sql:8776`). Mức giá gắn ở bệnh nhân (`pricelevel`).

| pr_id (code) | pr_level | pr_price |
|---|---|---|
| 99214 | standard | 200 |
| 99214 | medicare | 110 |
| 99214 | bluecross | 139.70 |
| 99214 | aetna | 120 |
| 99214 | selfpay | 120 |

Mỗi mã 5 dòng → 2.000 mã × 5 = **10.000 dòng**. MPFS đổi → sửa 2.000 dòng `medicare` **và** 2.000 dòng `bluecross` (vì Blue Cross = 127% Medicare).

**Sai lầm thường gặp:** gắn mức giá vào **bệnh nhân** (như OpenEMR) thay vì vào **bảo hiểm được dùng cho lần khám**. Bà Lan đổi từ Blue Cross sang Medicare giữa năm thì mức giá trên hồ sơ cũ sai.

### Ứng viên 3 — Bảng giá riêng từng payer, có hiệu lực 💭

```sql
CREATE TABLE fee_schedule (id, payer_id, effective_from DATE, effective_to DATE);
CREATE TABLE fee_schedule_line (schedule_id, code TEXT, modifier TEXT, allowed NUMERIC);
```

Đúng P4 (hiệu lực), P1, P3. Nhưng Blue Cross vẫn phải chép 2.000 dòng = 127% Medicare (✘ P2). Đây là cấu trúc của nhiều hệ thống billing truyền thống.

### Ứng viên 4 — Công thức hợp đồng + bảng gốc + ngoại lệ 💭

```sql
CREATE TABLE base_schedule (name TEXT, year INT, code TEXT, amount NUMERIC);     -- MPFS 2026, MPFS 2027
CREATE TABLE contract (payer_id, effective_from, effective_to, base TEXT, factor NUMERIC, fallback_pct_of_charge NUMERIC);
CREATE TABLE contract_override (contract_id, code TEXT, amount NUMERIC);          -- 300 mã Aetna
```

| Payer | Lưu |
|---|---|
| Medicare | `contract(base='MPFS', factor=1.00)` |
| Blue Cross | `contract(base='MPFS', factor=1.27)` |
| Aetna | `contract(fallback_pct_of_charge=0.60)` + 300 dòng `contract_override` |

MPFS 2027 về → nạp 1 bảng gốc mới, **0 dòng hợp đồng phải sửa**. Đây là cách các phần mềm **contract management** (quản lý hợp đồng payer) làm.

**Sai lầm thường gặp:** quên bảng gốc cũng có năm. `base = 'MPFS'` phải hiểu là "MPFS của năm ngày dịch vụ".

### Ứng viên 5 — Quy tắc khai báo trên từng mã (Medplum `ChargeItemDefinition`) ✅

✅ `packages/server/src/fhir/operations/chargeitemdefinitionapply.ts:25-128`. 1 ChargeItemDefinition cho mỗi mã, có nhóm điều kiện:

| Nhóm | Điều kiện (FHIRPath trên ChargeItem) | Thành phần |
|---|---|---|
| 1 | — | `base` 200 |
| 2 | payer = medicare | `discount` amount 90 |
| 3 | payer = bluecross | `discount` amount 60.30 |
| 4 | payer = aetna | `discount` factor 0.40 |
| 5 | payer = selfpay | `discount` factor 0.40 |

| Kịch bản | Kết quả |
|---|---|
| Bà Lan, Medicare | 200 − 90 = 110 ✔ |
| Blue Cross = 127% MPFS | ✘ Không tham chiếu được bảng MPFS. Phải tính sẵn thành số và **chép vào từng mã** (lại thành ma trận) |
| MPFS đổi | Sửa 2.000 ChargeItemDefinition (nhóm Medicare + Blue Cross) |
| P4 hiệu lực | ✘ `effectivePeriod` không được xét (R mục 4) |
| P5 số lượng | ✘ Không nhân `quantity` |
| P6 đóng băng | Server ✔ (ghi 1 lần). App mẫu ✘ (tính lại khi đọc, R mục 5) |

**Sai lầm thường gặp:** nghĩ rằng vì có điều kiện FHIRPath nên "biểu diễn được mọi hợp đồng". Điều kiện chỉ chọn nhóm, **không tính toán dựa trên bảng khác**.

---

## Phần 3 — Bảng so sánh trực diện

| | 1 Phẳng | 2 Ma trận (OpenEMR) | 3 Bảng/payer | 4 Công thức | 5 Quy tắc/mã (Medplum) |
|---|---|---|---|---|---|
| P1 theo payer | ✘ | ✔ | ✔ | ✔ | ✔ |
| P2 công thức % bảng gốc | ✘ | ✘ | ✘ | ✔ | ✘ (chỉ % của giá **niêm yết**) |
| P3 ngoại lệ | — | ✔ | ✔ | ✔ | ✔ |
| P4 hiệu lực | ✘ | ✘ | ✔ | ✔ | ✘ (server), tự chọn URL có version |
| P5 số lượng | — | ✔ (billing.units) | tuỳ | tuỳ | ✘ |
| P6 đóng băng | ✔ | ✔ | ✔ | ✔ | ✔ server / ✘ app mẫu |
| P7 điều kiện bệnh nhân | ✘ | ✘ (mức giá trên bệnh nhân) | ✘ | ⚠ thêm cột | ✔ FHIRPath |
| P8 xuất giá theo payer | ✘ | ✔ | ✔ | ✔ (tính ra) | ⚠ phải đánh giá từng nhóm |
| Số dòng dữ liệu 💭 | 2.000 | 10.000 | 16.000 | ~2.000 × số bảng gốc + 300 | 2.000 resource, mỗi cái 5 nhóm |
| Đổi MPFS hằng năm | — | Sửa 4.000 dòng | Sửa 4.000 dòng | Nạp 1 bảng | Sửa 2.000 resource |

---

## Phần 4 — Vì sao không có ứng viên nào thắng tuyệt đối

Khác các bài C trước (có 1 thiết kế thật "thắng"), ở đây **không** ứng viên nào thoả cả 8 đặc tính:
- Ứng viên 4 đúng nhất về nghiệp vụ, nhưng là **phần mềm chuyên dụng** (quản lý hợp đồng payer), không phải chức năng 1 nền tảng dữ liệu lâm sàng.
- Ứng viên 5 (Medplum) là **hiện thực chuẩn FHIR tối thiểu**: đủ cho tự trả và giảm giá đơn giản, không đủ cho hợp đồng payer.

🧠 Đây là ranh giới sản phẩm có chủ đích: Medplum để **định giá theo hợp đồng** cho đối tác billing (Candid Health có Bot mẫu ở `examples/medplum-demo-bots/src/candid-health/`, [4c](./4c-claim.md)). Phòng khám gửi claim với **giá niêm yết**, payer tự áp hợp đồng. Giá cho phép chỉ cần để **ước tính** và **đối soát**, việc đó đối tác làm.

---

## Phần 5 — 4 lối tắt nghe hợp lý nhưng sai

**3 lối dừng quá sớm:**
1. **"1 cột giá là đủ, payer tự áp hợp đồng."** Đúng cho claim, sai cho **ước tính phần bệnh nhân** tại quầy ([4a](./4a-quyen-loi-bao-hiem.md)) và cho **phát hiện trả thiếu**.
2. **"Ma trận mã × payer, có Excel nhập vào là xong."** Mỗi năm nhập lại 16.000 ô, sai 1 ô không ai biết.
3. **"Dùng `ChargeItemDefinition` cho hợp đồng payer."** Thiếu tham chiếu bảng gốc, thiếu hiệu lực, thiếu số lượng.

**1 lối leo quá xa:**
4. **"Tự xây engine định giá hợp đồng đầy đủ (DRG, APC, gói phẫu thuật, giảm giá nhiều thủ thuật)."** Đó là sản phẩm hàng chục người-năm của các công ty contract management. Phòng khám ngoại trú chỉ cần ứng viên 4 rút gọn.

---

## Phần 6 — Medplum tại từng ngã rẽ ✅

| Ngã rẽ | Medplum chọn | Ghi chú |
|---|---|---|
| Giá lưu ở đâu? | `ChargeItemDefinition` (chuẩn FHIR) | Lớp HL7 |
| Điều kiện viết bằng gì? | FHIRPath trên `ChargeItem` (`:53-59`) | Không truy cập Coverage, Patient |
| Thành phần nào được tính? | `base`, `surcharge`, `discount` (`:93-119`) | Bỏ `tax`, `deduction` |
| Giá ghi vào đâu? | `ChargeItem.priceOverride` (`:123`) | 1 con số |
| Hợp đồng payer? | Không làm. Để Bot/đối tác | Ranh giới sản phẩm |

---

## Phần 7 — Đánh đổi đo bằng số 🧠

Chi phí bảo trì hằng năm (💭, giả định 1 lần sửa dòng tay = 30 giây, nhập file = 2 giờ chuẩn bị):

| | Dòng phải đụng khi MPFS đổi | Ước tính công |
|---|---|---|
| 2 Ma trận | ~4.000 (Medicare + Blue Cross) | Nhập file: 2 giờ + rủi ro sai lệch hệ số |
| 3 Bảng/payer | ~4.000 | 2 giờ |
| 4 Công thức | 1 bảng gốc mới (2.000 dòng nạp tự động) | ~15 phút |
| 5 ChargeItemDefinition | 2.000 resource × 2 nhóm | Viết script, 2–4 giờ, mỗi lần ghi tạo `_history` mới |

Giá trị phát hiện trả thiếu (💭): 20.000 lượt/năm × 150$ allowed trung bình × 2% payer trả thiếu = **60.000$/năm**. Chỉ phát hiện được nếu hệ thống có giá cho phép dự kiến (ứng viên 2–4, hoặc 5 nếu bổ sung).

---

## Phần 8 — Đối chứng: chỗ không cần định giá phức tạp

- **Bot superbill** (`examples/medplum-demo-bots/src/billing-bots/`): bệnh nhân tự trả rồi tự nộp bảo hiểm. Chỉ cần **giá niêm yết** từ ChargeItemDefinition theo `serviceType` → ứng viên 1 là đủ.
- **Bot Stripe** (`examples/medplum-demo-bots/src/stripe-bots/stripe-create-invoice.ts`): thu tiền tự trả, giá cố định.
- **Phòng khám tư ở Việt Nam không BHYT**: ứng viên 1 + hiệu lực theo ngày. Có BHYT thì giá BHYT là **giá nhà nước quy định**, không thương lượng (xem [4e](./4e-phu-luc-bhyt-viet-nam.md)): gần ứng viên 2 với 2 mức (dịch vụ, BHYT).

---

## Phần 9 — Luyện phản biện

**Phản đối 1:** "FHIR có `ChargeItemDefinition`, dùng chuẩn là tốt nhất."

<details><summary>Đáp án gợi ý</summary>

Chuẩn cho phép biểu diễn **kết quả** (giá cuối cùng của 1 mã cho 1 điều kiện), không phải **cách sinh ra** kết quả (127% MPFS năm X). Có thể dùng chuẩn làm **đầu ra**: engine hợp đồng (ứng viên 4) xuất ra ChargeItemDefinition đã tính sẵn mỗi năm. Chuẩn ở biên, công thức ở trong.
</details>

**Phản đối 2:** "Gửi claim với giá cho phép cho gọn, khỏi phải điều chỉnh hợp đồng."

<details><summary>Đáp án gợi ý</summary>

Nhiều hợp đồng trả "giá thấp hơn giữa niêm yết và biểu phí". Gửi giá cho phép thì không bao giờ được trả hơn, kể cả khi hợp đồng tăng. Và mất số liệu niêm yết cho báo cáo minh bạch giá.
</details>

**Phản đối 3:** "Mức giá gắn ở bệnh nhân như OpenEMR đơn giản hơn."

<details><summary>Đáp án gợi ý</summary>

Đơn giản nhưng sai khi bệnh nhân có 2 bảo hiểm hoặc đổi bảo hiểm. Mức giá phải theo **bảo hiểm áp cho lần khám** (Coverage order 1 tại ngày dịch vụ).
</details>

---

## Phần 10 — Checklist · So với Saleor · Nguồn

**Checklist chọn cách định giá:**
- [ ] Hợp đồng là danh sách hay công thức? Có ngoại lệ?
- [ ] Có bảng gốc quốc gia (MPFS, giá BHYT nhà nước)? Theo năm?
- [ ] Giá áp theo ngày dịch vụ, đóng băng trên dòng phí?
- [ ] Số lượng, modifier?
- [ ] Cần giá cho phép dự kiến để ước tính và đối soát?
- [ ] Ai sửa giá, có nhật ký?

**So với Saleor (e-commerce) 🧠:** Saleor có giá theo **kênh** (channel) và khuyến mãi theo **điều kiện** (voucher, promotion rule). Tương ứng: kênh ≈ payer, promotion rule ≈ nhóm điều kiện của ChargeItemDefinition. Khác biệt lớn: trong e-commerce **người bán đặt giá cuối**, khách trả đúng giá đó. Trong y tế Mỹ **người mua (payer) đặt giá cuối** theo hợp đồng, người bán chỉ gửi giá niêm yết và chấp nhận điều chỉnh. Định giá y tế vì vậy là bài **đối soát** nhiều hơn bài **tính giá**.

**Nguồn:**
- 📘 FHIR R4 ChargeItemDefinition: https://hl7.org/fhir/R4/chargeitemdefinition.html
- 📘 CMS Physician Fee Schedule: https://www.cms.gov/medicare/payment/fee-schedules/physician
- 📘 CMS Hospital Price Transparency: https://www.cms.gov/priorities/key-initiatives/hospital-price-transparency
- ✅ R: [`4b-dinh-gia-ghi-phi.md`](./4b-dinh-gia-ghi-phi.md)
