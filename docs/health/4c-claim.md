# 4c — Claim: lập, gửi, in CMS-1500, nhận kết quả thanh toán

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Lần khám của bà Lan biến thành 1 yêu cầu thanh toán gửi Medicare thế nào, kết quả quay về ra sao, và phần nào Medplum tự làm, phần nào giao đối tác?"*

## Mục lục

- [1. Góc nhìn kinh doanh: vòng 837 → 835](#1-góc-nhìn-kinh-doanh-vòng-837--835)
- [2. 3 resource: Claim, ClaimResponse, ExplanationOfBenefit](#2-3-resource-claim-claimresponse-explanationofbenefit)
- [3. Lập Claim từ lần khám (app Provider)](#3-lập-claim-từ-lần-khám-app-provider)
- [4. `Claim/$submit`: điều phối giống eligibility](#4-claimsubmit-điều-phối-giống-eligibility)
- [5. `Claim/$export`: in mẫu CMS-1500](#5-claimexport-in-mẫu-cms-1500)
- [6. Đối tác billing: Candid, Stedi](#6-đối-tác-billing-candid-stedi)
- [7. Sau khi gửi: 835, điều chỉnh, công nợ (Medplum không có)](#7-sau-khi-gửi-835-điều-chỉnh-công-nợ-medplum-không-có)
- [8. Đối chiếu OpenEMR](#8-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Góc nhìn kinh doanh: vòng 837 → 835

📘 Giao dịch HIPAA X12 cho claim ngoại trú:

```
Phòng khám ──837P (claim)──▶ Clearinghouse ──▶ Payer
     ▲                          │ 277CA (đã nhận / bị trả lại vì lỗi định dạng)
     │                          ▼
     └────────── 835 (ERA: trả bao nhiêu, điều chỉnh vì sao) ──── Payer
```

| Mốc | Ý nghĩa tài chính |
|---|---|
| 837P gửi | Ghi nhận **khoản phải thu** (A/R) theo giá niêm yết |
| 277CA từ chối | Claim **không vào được** payer (sai định dạng, sai số thành viên). Chưa phải từ chối thanh toán |
| 835 | Payer trả tiền + **điều chỉnh** theo từng dòng, mỗi điều chỉnh có **nhóm** và **mã lý do** |

📘 Nhóm điều chỉnh trong 835 (segment `CAS`):

| Nhóm | Nghĩa | Ai chịu | Ví dụ bà Lan 💭 |
|---|---|---|---|
| `CO` Contractual Obligation | Theo hợp đồng, phòng khám **không được thu** của bệnh nhân | Phòng khám | 117$ (240 − 123) |
| `PR` Patient Responsibility | Phần bệnh nhân (khấu trừ, đồng chi trả) | Bệnh nhân / bảo hiểm phụ | 24,60$ |
| `OA` Other Adjustment | Khác | | |
| `PI` Payer Initiated | Payer tự điều chỉnh | | |

Mỗi điều chỉnh kèm **CARC** (Claim Adjustment Reason Code, VD `45` vượt biểu phí, `1` khấu trừ, `2` đồng chi trả) và có thể có **RARC** giải thích thêm.

🧠 Chỉ số revenue cycle đọc được từ vòng này:
- **Clean claim rate**: % claim qua 277CA ngay lần đầu.
- **Denial rate**: % dòng bị từ chối (CARC thuộc nhóm từ chối).
- **Days in A/R**: số ngày từ 837 tới khi thu đủ.
- **Net collection rate**: thực thu / (niêm yết − điều chỉnh CO).

Đây là phần người làm tài chính đọc được ngay, và là chỗ **Medplum (mã nguồn mở) gần như không có gì** (mục 7).

## 2. 3 resource: Claim, ClaimResponse, ExplanationOfBenefit

| Resource | Ai tạo | Tương ứng X12 | Trường chính |
|---|---|---|---|
| `Claim` | Phòng khám | 837 (claim), 278 (`use = preauthorization`) | `use`, `type` (`professional` = ngoại trú/CMS-1500, `institutional` = UB-04), `patient`, `provider`, `insurance[]` (`sequence`, `focal`), `diagnosis[]` (ICD-10, `sequence`), `item[]` (`productOrService` CPT, `modifier`, `quantity`, `net`, **`diagnosisSequence`**), `total` |
| `ClaimResponse` | Payer (hoặc Bot đọc 835) | 835 cho 1 claim | `outcome`, `item[].adjudication[]` (`category`: submitted, eligible, benefit, copay, deductible… + `amount`), `payment` |
| `ExplanationOfBenefit` | Payer | Gộp claim + kết quả | Dùng cho **bệnh nhân xem**, theo CARIN Blue Button (luật Mỹ buộc payer cung cấp, [2d](./2d-interoperability.md)) |

📘 `Claim.diagnosisSequence` trên mỗi dòng là **con trỏ** sang `Claim.diagnosis[].sequence`: dòng dịch vụ này được làm **vì** chẩn đoán nào. Payer dùng nó kiểm tra **tính cần thiết y khoa** (medical necessity). Thiếu là lý do từ chối phổ biến.

## 3. Lập Claim từ lần khám (app Provider)

✅ `examples/medplum-provider/src/utils/claims.ts`, `buildClaimFromEncounter` (`:44-69`):
- Claim **nháp, chưa lưu** (docblock `:31-43`). Chỉ lưu lúc export hoặc submit.
- `type = professional`, `use = claim`, `status = draft` (`:47-50`).
- `provider` = Organization billing nếu có, không thì bác sĩ (billing cá nhân) (`:53`). Bác sĩ khám luôn là `careTeam` `primary` (`:54-62`).
- `insurance[]`: theo thứ tự truyền vào, phần tử đầu `focal` (`:64`).
- `diagnosis[]` từ `Condition` của lần khám, mã đầu `principal`, còn lại `secondary` (`createDiagnosisArray`, `:94-112`).
- `item[]` từ `ChargeItem` có mã CPT: `productOrService`, `modifier` (từ extension `chargeitem-modifier`), **`net` = `priceOverride`** (`getCptChargeItems`, `:71-91`).
- `total` = tổng `priceOverride` (`:67`).

🧠 3 điểm cần sửa trước khi dùng thật:
1. **Không gán `diagnosisSequence`** cho dòng nào (grep cả app không thấy). Ô 24E trên CMS-1500 sẽ trống, payer không biết dòng nào gắn chẩn đoán nào.
2. **Không có `quantity`** trên dòng claim, và `net` lấy từ `priceOverride` vốn không nhân số lượng ([4b mục 4](./4b-dinh-gia-ghi-phi.md#4-những-gì-apply-không-làm)).
3. Mã chẩn đoán lấy từ system `…/icd-10-cm` rồi **ghi lại system thành `…/icd-10`** (`:96-104`) để khớp bộ in CMS-1500 (`packages/server/src/fhir/operations/utils/cms1500pdf.ts:73-77`). Về ngữ nghĩa, ICD-10-CM (bản Mỹ) và ICD-10 (WHO) là 2 hệ mã khác nhau ([1e](./1e-terminology.md)). Đây là thoả hiệp để 2 phần khớp nhau, không phải cách mã hoá chuẩn.

## 4. `Claim/$submit`: điều phối giống eligibility

✅ `packages/server/src/fhir/operations/claimsubmit.ts`:
- Cùng khuôn với `CoverageEligibilityRequest/$submit` ([4a mục 4](./4a-quyen-loi-bao-hiem.md#4-submit-bộ-điều-phối-không-phải-bộ-gửi)).
- Setting `CLAIM_SUBMIT_OPERATION` cho claim thường, **`PRIOR_AUTH_SUBMIT_OPERATION`** cho `use = preauthorization` (`:11-12`, `:42-45`).
- Gọi `dispatchSubmitOperation` → OperationDefinition tuỳ biến → Bot (`utils/submit.ts:21-51`).

🧠 1 cơ chế cho 3 loại giao dịch (eligibility, claim, prior auth): mỗi project nối với clearinghouse/đối tác của mình bằng 3 setting.

## 5. `Claim/$export`: in mẫu CMS-1500

📘 **CMS-1500** là mẫu giấy chuẩn cho claim ngoại trú ở Mỹ (bản điện tử tương đương là 837P). Vẫn dùng cho payer nhỏ, bồi thường lao động, hoặc làm bản xem trước.

✅ `packages/server/src/fhir/operations/claimexport.ts`:
- `GET Claim/:id/$export` hoặc `POST Claim/$export` (`:108-122`, `:137+`).
- Dựng PDF (`getClaimPDFDocDefinition`, `utils/cms1500pdf.ts:33`), lưu thành `Binary` (`:63-72`), trả về `Media` trỏ tới Binary (`:75-88`). `Media` **không được lưu**, chỉ trả trong response.

✅ `utils/cms1500pdf.ts` ánh xạ ô:
- Bảo hiểm chính/phụ từ `insurance[0]`, `insurance[1]` (`:38`, `:42`). Số duyệt trước (ô 23) từ `insurance[0].preAuthRef` (`:68`).
- Ô 21 chẩn đoán: chỉ lấy mã có system `…/icd-10` (`:73-77`).
- Ô 24 dòng dịch vụ: ngày, nơi khám (mã Place of Service của CMS, `:346-348`), cấp cứu, CPT, modifier, **con trỏ chẩn đoán** đổi 1→A, 2→B (`formatDiagnosisPointers`, `:361+`), số tiền, số lượng (`:241-268`).
- Ô 25: mã số thuế của **phòng khám**, không phải của payer (ghi chú `:97`).

🧠 Mẫu giấy CMS-1500 chỉ có **6 dòng dịch vụ** và **12 chẩn đoán**. Code vẽ mọi `claim.item` theo toạ độ tăng dần (`y = 542 + index * 25`, `:242`), không chia trang. Claim 8 dòng sẽ in tràn ra ngoài khung.

## 6. Đối tác billing: Candid, Stedi

✅ App Provider có 2 nút gửi: **"Submit to Candid"** và **"Submit to Stedi"** (`examples/medplum-provider/src/components/encounter/SubmitClaimModal.tsx:253-270`). Candid Health là dịch vụ billing (RCM), Stedi là clearinghouse API.

✅ Bot Candid (`examples/medplum-demo-bots/src/candid-health/`):
- `send-to-candid.ts`: chạy theo **Encounter** (không theo Claim), dựng "coded encounter" (bệnh nhân, thẻ bảo hiểm, chẩn đoán, dịch vụ) rồi gọi API Candid (`:19`, `:126-153`).
- `sync-candid-tasks.ts`: chạy **cron**, kéo danh sách việc từ Candid (claim cần sửa, bị từ chối) về thành `Task` trong Medplum, gắn với Encounter theo `identifier` (`:9-61`).

🧠 Mô hình phân vai rõ:
- **Medplum** giữ dữ liệu lâm sàng chuẩn (Encounter, Condition, Coverage, Practitioner có NPI) và hàng đợi việc (Task, [3d](./3d-careplan-task.md)).
- **Đối tác RCM** giữ toàn bộ vòng 837 → 835, công nợ, xử lý từ chối.
- Chỗ nối là Bot. README Candid nhấn mạnh: chất lượng claim phụ thuộc **dữ liệu nguồn được chú thích đúng** (NPI, địa chỉ, mã).

## 7. Sau khi gửi: 835, điều chỉnh, công nợ (Medplum không có)

Grep `packages/server/src` không thấy code đọc 835, tạo `ClaimResponse`/`ExplanationOfBenefit` từ ERA, hay sổ công nợ. 🧠 Trong mã nguồn mở Medplum:

| Việc | Có? |
|---|---|
| Đọc 835, ghi điều chỉnh CO/PR/OA theo dòng | ✘ (đối tác, hoặc Bot tự viết) |
| Sổ cái công nợ bệnh nhân / payer | ✘ |
| Đẩy phần còn lại sang bảo hiểm phụ (crossover) | ✘ |
| Hoá đơn cho bệnh nhân | Bot Stripe mẫu (`examples/medplum-demo-bots/src/stripe-bots/`) |
| Hàng đợi xử lý từ chối | `Task` (đồng bộ từ đối tác) |

## 8. Đối chiếu OpenEMR

OpenEMR có **cả vòng**, vì nó là phần mềm billing phòng khám:

| Việc | OpenEMR ✅ |
|---|---|
| Bảng claim | `claims` (`sql/database.sql:379`): `patient_id`, `encounter_id`, **`version`** (mỗi lần gửi lại tăng), `payer_id`, `payer_type`, `status`, `bill_time`, `x12_partner_id`, `submitted_claim` (lưu nguyên claim đã gửi) |
| Sinh 837P | `src/Billing/X125010837P.php` (1.680 dòng) |
| Đọc 835 | `src/Billing/ParseERA.php` (561 dòng): đọc `CAS` ở cấp claim và dòng (`:269`, `:394`), xử lý riêng nhóm `PR` và `CO` (`:45`, `:402`) |
| Phiên nhận tiền | `ar_session` (`:10184`): 1 séc / 1 lần chuyển khoản của payer |
| Bút toán | `ar_activity` (`:10214`): mỗi dòng theo **(bệnh nhân, lần khám, mã dịch vụ, payer_type 0 = bệnh nhân, 1 = BH chính, 2 = BH phụ)**, có `pay_amount` **hoặc** `adj_amount`, `reason_code` (CARC), `payer_claim_number` (CLP07), `follow_up`, `deleted` (huỷ mềm) |

🧠 `ar_activity` là **sổ cái kế toán** thu nhỏ: mỗi bút toán là thanh toán hoặc điều chỉnh, có lý do chuẩn, gắn với dòng dịch vụ và người trả. Đây là cấu trúc mà người làm tài chính nhận ra ngay. Medplum không có tương đương. Nếu xây trên Medplum, có 2 lựa chọn: (1) thuê RCM bên ngoài như mô hình Candid, (2) tự mô hình bằng `ClaimResponse.item.adjudication` + `PaymentReconciliation` của FHIR, kèm 1 kho dữ liệu kế toán riêng.

## Sai lầm thường gặp

1. **Không có `diagnosisSequence`.** Payer từ chối vì không chứng minh được tính cần thiết.
2. **Gửi bảo hiểm phụ trước chính.** `insurance[].sequence` và `focal` phải theo `Coverage.order`.
3. **Claim > 6 dòng in CMS-1500.** Tràn khung, phải tách claim.
4. **Coi 277CA từ chối là bị từ chối thanh toán.** Đó là lỗi định dạng, sửa và gửi lại ngay, không phải kháng nghị.
5. **Ghi doanh thu theo giá niêm yết, không ghi điều chỉnh CO.** Báo cáo doanh thu phồng.
6. **Mong Medplum xử lý 835.** Cần đối tác hoặc Bot riêng.

## Tự kiểm tra

1. Viết `Claim` cho lần khám bà Lan: 2 chẩn đoán (E11.9 tiểu đường, I10 tăng huyết áp), 2 dòng (`99214` gắn cả 2 chẩn đoán, `83036` gắn E11.9).
2. 835 trả `99214`: allowed 110, paid 88, `CO-45` 90, `PR-2` 22. Dựng `ClaimResponse.item.adjudication` tương ứng.
3. Vì sao Bot Candid gửi theo Encounter thay vì theo Claim?
