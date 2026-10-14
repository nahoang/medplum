# 4b — Định giá & ghi nhận phí: `ChargeItemDefinition/$apply`, `ChargeItem`, `Account`

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Medplum tính giá 1 dịch vụ thế nào, giá được lưu ở đâu, và có gì khác cách phòng khám truyền thống làm?"* Thiết kế từ đầu: [`4b-greenfield-dinh-gia.md`](./4b-greenfield-dinh-gia.md). So sánh: [`4b-comparison-dinh-gia.md`](./4b-comparison-dinh-gia.md).

## Mục lục

- [1. Góc nhìn kinh doanh: 3 con số giá](#1-góc-nhìn-kinh-doanh-3-con-số-giá)
- [2. 3 resource tài chính](#2-3-resource-tài-chính)
- [3. `$apply` từng bước](#3-apply-từng-bước)
- [4. Những gì `$apply` không làm](#4-những-gì-apply-không-làm)
- [5. App Provider: tính lại giá mỗi lần mở](#5-app-provider-tính-lại-giá-mỗi-lần-mở)
- [6. Nhầm tên: `$set-accounts` không phải tài chính](#6-nhầm-tên-set-accounts-không-phải-tài-chính)
- [7. Đối chiếu OpenEMR](#7-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Góc nhìn kinh doanh: 3 con số giá

📘/🧠 Mỗi dòng dịch vụ ở Mỹ có 3 con số. Bà Lan, CPT `99214` 💭:

| Con số | Ai đặt | Ví dụ | Xuất hiện ở đâu |
|---|---|---|---|
| **Giá niêm yết** (charge, chargemaster) | Phòng khám | 200$ | Claim 837 ([4c](./4c-claim.md)) |
| **Giá payer cho phép** (allowed amount) | Hợp đồng với payer | 110$ | 835 / ExplanationOfBenefit |
| **Payer thực trả** | Payer, sau khi trừ phần bệnh nhân | 88$ (80%) | 835 |

- Niêm yết − cho phép = **điều chỉnh hợp đồng** (90$), không thu được của ai.
- Cho phép − payer trả = **phần bệnh nhân** (22$: đồng chi trả, khấu trừ, copay).

🧠 Mã CPT được chọn quyết định cả 3 con số. Dòng phí sai mã là sai doanh thu, nên **coding** là 1 nghề riêng trong revenue cycle.

## 2. 3 resource tài chính

| Resource | Vai trò | Trường chính |
|---|---|---|
| `ChargeItemDefinition` | **Quy tắc giá** cho 1 dịch vụ | `url`, `code`, `effectivePeriod`, `propertyGroup[]` = { `applicability[]` (điều kiện), `priceComponent[]` (`type`: `base`, `surcharge`, `deduction`, `discount`, `tax`, `informational`; `amount` hoặc `factor`) } |
| `ChargeItem` | **1 dòng phí** của 1 bệnh nhân | `code` (CPT), `quantity`, `subject`, `context` (Encounter), `occurrence[x]`, `definitionCanonical`, `priceOverride`, `factorOverride`, `account`, `status` (`planned`, `billable`, `billed`…) |
| `Account` | **Tài khoản** gom phí | `subject`, `guarantor` (người chịu trách nhiệm trả), `coverage[]` (kèm `priority`) |

📘 Cùng mẫu "định nghĩa → thể hiện" như PlanDefinition → Task ([3d](./3d-careplan-task.md#1-5-resource-2-tầng)). Lớp quyết định: HL7.

✅ Trong mã nguồn mở Medplum, `Account` hầu như chỉ thấy ở Bot mẫu Stripe (`examples/medplum-demo-bots/src/stripe-bots/stripe-create-invoice.ts`). 🧠 Tài khoản bệnh nhân (A/R, công nợ) thường giao cho hệ thống billing chuyên dụng (Candid Health…), không dựng trên Medplum.

## 3. `$apply` từng bước

✅ `packages/server/src/fhir/operations/chargeitemdefinitionapply.ts` (`chargeItemDefinitionApplyHandler`, `:25-128`). Gọi `POST ChargeItemDefinition/:id/$apply` với tham số `chargeItem` (tham chiếu).

| Bước | ✅ |
|---|---|
| Đọc ChargeItemDefinition và ChargeItem | `:28-34` |
| (Vòng 0) Gán `priceOverride` = giá `base` **đầu tiên tìm thấy, không xét điều kiện** | `:39-47` |
| Vòng 1: duyệt nhóm, đánh giá `applicability` bằng FHIRPath **trên ChargeItem** (`%resource` = ChargeItem). Nhóm đầu tiên thoả điều kiện và có `base` → giá cơ sở | `:49-72` |
| Vòng 2: với mọi nhóm thoả điều kiện, cộng `surcharge`, trừ `discount`. Mỗi thành phần là số tiền, hoặc **hệ số × giá cơ sở** (không cộng dồn kép) | `:74-121` |
| Ghi `priceOverride` = giá cuối, `updateResource(ChargeItem)` | `:123-126` |

💭 Ví dụ ChargeItemDefinition cho `99214`:

| Nhóm | Điều kiện (FHIRPath trên ChargeItem) | Thành phần |
|---|---|---|
| 1 | (không) | `base` 200$ |
| 2 | `extension.where(url='…/payer').value = 'medicare'` | `discount` factor 0.45 |
| 3 | `extension.where(url='…/payer').value = 'selfpay'` | `discount` factor 0.40 |

ChargeItem có extension payer = medicare → 200 − 200 × 0.45 = **110$**.

🧠 Điều kiện chỉ nhìn được **trường trên ChargeItem**. FHIRPath ở đây không đi theo tham chiếu sang `Coverage` hay `Patient`. Muốn giá theo payer thì phải **chép thông tin payer lên ChargeItem** trước khi `$apply` (Bot làm việc này từ `Coverage.order = 1`).

## 4. Những gì `$apply` không làm

| Thiếu | Hệ quả | ✅ |
|---|---|---|
| **Không nhân `quantity`** | Tiêm 3 mũi vẫn ra giá 1 mũi | Không chỗ nào đọc `quantity` trong `:25-128` |
| **Bỏ qua `tax`, `deduction`, `informational`** | Thuế không được tính | Chỉ xử lý `base`, `surcharge`, `discount` (`:93-119`) |
| **Không xét `effectivePeriod`** | Bảng giá 2027 áp cho lần khám 2026 nếu cùng resource | Toàn hàm |
| **Nhiều điều kiện trong 1 nhóm: điều kiện sau ghi đè điều kiện trước** | `isGroupApplicable` gán lại mỗi vòng (`:53-58`, `:80-85`), nên chỉ điều kiện **cuối** quyết định, không phải "tất cả đều đúng" | `:53-58` |
| **Vòng 0 không xét điều kiện** | Nếu không nhóm nào có `base` thoả điều kiện, ChargeItem vẫn mang giá `base` của nhóm đầu tiên dù nhóm đó không áp dụng | `:39-47` |
| **1 trường giá duy nhất** | `priceOverride` vừa là niêm yết vừa là dự kiến, không tách được 3 con số ở mục 1 | `:123` |

🧠 Đây là hiện thực **tối thiểu** của 1 chuẩn rộng. Dùng được cho bảng giá đơn giản (giá cơ sở + giảm giá tự trả), không đủ cho hợp đồng payer thật. Cùng tình trạng với `$apply` của PlanDefinition ([3d mục 3](./3d-careplan-task.md#3-apply-không-làm-gì)).

## 5. App Provider: tính lại giá mỗi lần mở

✅ `examples/medplum-provider/src/`:
- Thêm dòng phí: tạo `ChargeItem` (`status: planned`, `quantity: 1`, `code` + extension `ServiceBillingCode` = CPT, `definitionCanonical`), rồi gọi `$apply` ngay (`components/ChargeItem/ChargeItemList.tsx:66-87`).
- **Mở tab Billing**: `getChargeItemsForEncounter` search mọi ChargeItem của Encounter rồi **gọi `$apply` lại cho từng cái** (`utils/chargeitems.ts:44-57`, gọi từ `components/encounter/BillingTab.tsx:82`).
- Tổng tiền = cộng `priceOverride`, **không nhân số lượng** (`utils/chargeitems.ts:59-61`).

🧠 Hệ quả tài chính của "tính lại khi đọc":
1. Sửa ChargeItemDefinition (tăng giá 2027) → mở lại lần khám cũ → **giá lần khám cũ đổi theo**, lệch với claim đã gửi.
2. Mỗi lần mở tab là 1 lần **ghi** (`updateResource`) cho mỗi dòng phí, sinh 1 phiên bản `_history` mới ([1f](./1f-history-audit-deletion.md)) dù giá không đổi.
3. Đúng lối tắt #2 ở [4b-G](./4b-greenfield-dinh-gia.md#3-lối-tắt-nghe-hợp-lý-nhưng-sai). Sửa: chỉ `$apply` khi `status = planned`, chuyển `billable` sau khi chốt, không áp lại.

✅ Bot mẫu khác dùng ChargeItemDefinition: superbill (`examples/medplum-demo-bots/src/billing-bots/`), ánh xạ `Encounter.serviceType` → `ChargeItemDefinition.useContext` để lấy phí, xuất PDF bảng kê dịch vụ cho bệnh nhân tự nộp bảo hiểm.

## 6. Nhầm tên: `$set-accounts` không phải tài chính

Kế hoạch ([plan.md](./plan.md), Giai đoạn 4) xếp `$set-accounts` vào 4b. Đọc code thì **không đúng**:

✅ `packages/server/src/fhir/operations/set-accounts.ts`:
- Ghi `meta.accounts` (và `meta.account` = phần tử đầu) của 1 resource (`:162-166`).
- `propagate = true` trên Patient: lan sang mọi resource trong compartment bệnh nhân, tối đa 1.000 resource mỗi trang, có chế độ async qua `Prefer: respond-async` (`:94-108`, `:174-195`).

✅ `packages/server/src/fhir/repo.ts:2074-2080`: mỗi phần tử `meta.accounts` (trừ `Project/…`) được thêm vào cột **`compartments`**, tức là dùng cho **phân quyền** ([1c](./1c-access-control.md)).

🧠 "Account" ở đây là **"tài khoản sở hữu dữ liệu"** (VD 1 Organization đối tác được thấy bệnh nhân này), không phải resource FHIR `Account` (tài khoản công nợ). Trùng tên, khác hẳn nghĩa. Đã ghi vào câu hỏi mở của [CLAUDE.md](../CLAUDE.md).

## 7. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Giá trên mã | `ChargeItemDefinition` base | `codes.fee` (`sql/database.sql:1125`) |
| Giá theo mức/payer | Điều kiện FHIRPath + hệ số | Ma trận `prices(pr_id, pr_selector, pr_level, pr_price)` (`:8776`). Mức giá gắn ở **hồ sơ bệnh nhân** (`pricelevel`, layout `DEM` `:3714`) |
| Dòng phí | `ChargeItem.priceOverride` | `billing` (`:246`): `code`, `modifier`, **`units`**, **`fee`** (chép lúc ghi), **`justify`** (chẩn đoán đi kèm), `payer_id`, `billed`, `bill_date` |
| Đóng băng giá | Server: có. App mẫu: không | Có (`billing.fee` không tính lại) |
| Số lượng | Không | Có (`units`) |
| Liên kết dòng phí ↔ chẩn đoán | Không có trên ChargeItem (`Claim.item.diagnosisSequence`, [4c](./4c-claim.md)) | `billing.justify` |

🧠 Ở mảng này OpenEMR **trưởng thành hơn**: đóng băng giá, có số lượng, có liên kết chẩn đoán ngay trên dòng phí. Vì OpenEMR sinh ra để **lập claim** cho phòng khám Mỹ. Medplum để phần này cho đối tác billing.

## Sai lầm thường gặp

1. **Gọi `$apply` mỗi lần đọc.** Giá trôi, lịch sử phình (mục 5).
2. **Tin `$apply` nhân số lượng.** Không.
3. **Đặt nhiều điều kiện trong 1 nhóm** với ý "tất cả phải đúng". Chỉ điều kiện cuối có hiệu lực.
4. **Dùng `effectivePeriod` để phân năm giá.** Không được xét. Tách mỗi năm 1 ChargeItemDefinition (URL có version), chọn đúng cái theo ngày dịch vụ ở Bot.
5. **Nhầm `$set-accounts` với tài khoản công nợ.**
6. **Để người dùng nhập payer lên ChargeItem.** Điều kiện giá đọc từ đó, nên phải do Bot gán từ `Coverage`.

## Tự kiểm tra

1. Viết ChargeItemDefinition cho `83036` (HbA1c): niêm yết 40$, Medicare giảm 67,5%, tự trả giảm 40%. Kết quả `$apply` cho 2 ChargeItem quantity 1 và 2?
2. Vì sao điều kiện giá phải nằm trên ChargeItem chứ không đọc thẳng từ Coverage?
3. Nêu 2 cách khắc phục "tính lại khi đọc" mà không sửa server.
