# 4a — Bảo hiểm & kiểm tra quyền lợi: `Coverage`, `CoverageEligibilityRequest/$submit`

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Medplum lưu bảo hiểm của bệnh nhân thế nào, và khi lễ tân bấm 'kiểm tra quyền lợi' thì request đi đâu?"* Thiết kế từ đầu: [`4a-greenfield-quyen-loi-bao-hiem.md`](./4a-greenfield-quyen-loi-bao-hiem.md). Đối chiếu Việt Nam: [`4e-phu-luc-bhyt-viet-nam.md`](./4e-phu-luc-bhyt-viet-nam.md).

## Mục lục

- [1. Góc nhìn kinh doanh: 4a nằm ở đâu trong dòng tiền](#1-góc-nhìn-kinh-doanh-4a-nằm-ở-đâu-trong-dòng-tiền)
- [2. `Coverage`: 1 hợp đồng bảo hiểm](#2-coverage-1-hợp-đồng-bảo-hiểm)
- [3. Kiểm tra quyền lợi: 3 tầng câu hỏi](#3-kiểm-tra-quyền-lợi-3-tầng-câu-hỏi)
- [4. `$submit`: bộ điều phối, không phải bộ gửi](#4-submit-bộ-điều-phối-không-phải-bộ-gửi)
- [5. Bot mẫu và app Provider](#5-bot-mẫu-và-app-provider)
- [6. Đối chiếu OpenEMR: X12 270 tự dựng](#6-đối-chiếu-openemr-x12-270-tự-dựng)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Góc nhìn kinh doanh: 4a nằm ở đâu trong dòng tiền

📘/🧠 Chu trình doanh thu (revenue cycle) ngoại trú Mỹ, và module nào của Giai đoạn 4 phụ trách bước nào:

| # | Bước | Giao dịch X12 (HIPAA) | Resource FHIR | Module |
|---|---|---|---|---|
| 1 | Đăng ký, ghi nhận bảo hiểm | — | `Coverage` | **4a** |
| 2 | Kiểm tra quyền lợi | **270 / 271** | `CoverageEligibilityRequest` / `Response` | **4a** |
| 3 | Xin duyệt trước (nếu cần) | 278 | `Claim` (`use = preauthorization`) | [4c](./4c-claim.md) |
| 4 | Khám, ghi mã chẩn đoán (ICD-10-CM) và dịch vụ (CPT/HCPCS) | — | `Encounter`, `Condition`, `Procedure` | [3c](./3c-charting-encounter.md) |
| 5 | Ghi nhận phí, định giá | — | `ChargeItem`, `ChargeItemDefinition`, `Account` | [4b](./4b-dinh-gia-ghi-phi.md) |
| 6 | Gửi claim | **837P** (ngoại trú) | `Claim` | [4c](./4c-claim.md) |
| 7 | Nhận kết quả thanh toán | **835** | `ClaimResponse`, `ExplanationOfBenefit` | [4c](./4c-claim.md) |
| 8 | Thu phần bệnh nhân, xử lý từ chối | — | `Invoice`, `Task` | [4c](./4c-claim.md) |
| 9 | Đo chất lượng (trả theo giá trị) | — | `Measure`, `MeasureReport` | [4d](./4d-do-luong-chat-luong.md) |

🧠 Bước 1–2 là **cửa đầu tiên** của dòng tiền. Sai ở đây thì mọi bước sau đều thành công về kỹ thuật nhưng thất bại về tiền: claim gửi đi đúng định dạng, payer từ chối vì bảo hiểm không còn hiệu lực.

## 2. `Coverage`: 1 hợp đồng bảo hiểm

📘 `packages/docs/docs/billing/patient-insurance.md`:

| Trường | Ý nghĩa | Ví dụ bà Lan 💭 | Lớp quyết định |
|---|---|---|---|
| `beneficiary` | Người **được** bảo hiểm | bà Lan | HL7 |
| `subscriber` | Người **đứng tên** hợp đồng | con trai bà (RelatedPerson) | HL7 |
| `relationship` | Quan hệ subscriber ↔ beneficiary | `parent` | HL7 + US Core value set |
| `subscriberId` hoặc `identifier` (type `MB`) | Số thành viên | | **US Core** bắt buộc |
| `payor` | Ai trả tiền (`Organization`) | Medicare | HL7 |
| `type` | Loại gói (HMO, PPO, Medicare…) theo **SOPT** | Medicare | **US Core** |
| `class` | Nhóm, tên gói, RXBIN/RXPCN (cho thuốc) | | HL7 |
| `costToBeneficiary` | Copay, đồng chi trả. Mã theo **CARIN** | 20$ khám chuyên khoa | CARIN IG |
| `order` | Thứ tự áp dụng: 1 chính, 2 phụ… | Medicare = 1, Medigap = 2 | HL7 |
| `period` | Hiệu lực | | HL7 |

📘 2 chi tiết đáng nhớ trong docs:
- **Thẻ giảm giá của hãng thuốc** (co-pay card) được mô hình như 1 `Coverage` nữa trong chồng bảo hiểm. Medplum khuyên dùng **giá trị âm** ở `costToBeneficiary` cho khoản hoàn lại, vì cộng đồng FHIR chưa có chuẩn (mục "Primary vs. Secondary Coverage").
- **Tự trả** (self-pay) cũng có cách biểu diễn riêng (mục "Plan Types and Payors").

✅ Bot tiếp nhận ([3a](./3a-dinh-danh-tiep-nhan.md#bot-tiếp-nhận-mẫu-làm-gì)) tạo `Coverage` từ phiếu khai (`examples/medplum-patient-intake-demo/src/bots/core/intake-form.ts:234`).

## 3. Kiểm tra quyền lợi: 3 tầng câu hỏi

📘 `packages/docs/docs/billing/insurance-eligibility-checks.md`, mục "Use Cases":

| Tầng | Câu hỏi | X12 service type |
|---|---|---|
| 1 | Bảo hiểm còn hiệu lực? | — |
| 2 | Có chi trả khám cơ bản? | **`30`** Plan Coverage and General Benefits (phổ biến nhất) |
| 3 | Có chi trả **loại** dịch vụ này? | VD `12` Durable Medical Equipment |

📘 Docs nhấn mạnh: hỏi theo **loại** dịch vụ (service type), không theo **dịch vụ cụ thể** (CPT). 🧠 Lý do: 270/271 được thiết kế quanh service type. Hỏi theo CPT cụ thể là việc của duyệt trước (278).

Cấu trúc FHIR:
- `CoverageEligibilityRequest`: `purpose` (`benefits`, `validation`…), `patient`, `insurer`, `insurance[].coverage`, `item[].category` = service type.
- `CoverageEligibilityResponse`: `outcome`, `insurance[].inforce`, `insurance[].item[]` (từng loại dịch vụ: `excluded`, `benefit[]` với `allowedMoney`/`usedMoney`, `authorizationRequired`).

## 4. `$submit`: bộ điều phối, không phải bộ gửi

✅ `packages/server/src/fhir/operations/coverageeligibilityrequestsubmit.ts`:
- 2 cửa vào: `POST CoverageEligibilityRequest/$submit` (gửi resource hoặc Bundle) và `POST CoverageEligibilityRequest/:id/$submit` (gửi theo id) (`:59-90`).
- Kiểm tra có đúng 1 `CoverageEligibilityRequest` (`:33-43`).
- Chuyển cho `dispatchSubmitOperation` với tên setting `ELIGIBILITY_SUBMIT_OPERATION` (`:11`, `:45`).

✅ `utils/submit.ts:21-51` (`dispatchSubmitOperation`), dùng chung cho eligibility, claim, prior auth:
1. Đọc `Project.setting[name = ELIGIBILITY_SUBMIT_OPERATION]` → được 1 mã operation, VD `check-eligibility-availity` (`:29`).
2. Không có setting → `400` "… submit is not configured: set the … project setting" (`:30-32`).
3. Dựng request con `POST /CoverageEligibilityRequest/$<mã>` (`:36-41`) và gọi `tryCustomOperation` (`:42`).
4. `tryCustomOperation` (`custom.ts:20+`) tìm `OperationDefinition` có `code` = mã đó, đọc extension `operationDefinition-implementation` trỏ tới 1 **Bot**, rồi chạy Bot (`:31-60`).

```
App ──$submit──▶ coverageeligibilityrequestsubmit.ts
                     │ đọc Project.setting ELIGIBILITY_SUBMIT_OPERATION = "check-elig-x"
                     ▼
               custom.ts: OperationDefinition(code="check-elig-x") ──implementation──▶ Bot
                                                                                        │ X12 270 / API JSON
                                                                                        ▼
                                                                                  Clearinghouse ↔ Payer
```

🧠 3 lớp gián tiếp (setting → OperationDefinition → Bot) để **mỗi project chọn clearinghouse riêng** mà app không đổi. Đây là đúng mẫu "chuẩn trước, nghiệp vụ tự xây" ([CLAUDE.md phát hiện 9](../CLAUDE.md#phát-hiện-quan-trọng-đến-nay)): Medplum cố định **hợp đồng giao diện** (operation FHIR chuẩn), để **kết nối thật** cho khách hàng hoặc đối tác.

## 5. Bot mẫu và app Provider

✅ `examples/medplum-eligibility-demo/src/bots/process-eligibility-request.ts`:
- Docblock ghi rõ: Bot **chỉ minh hoạ luồng**, phải tự chọn clearinghouse và sửa code.
- `processRequest` dựng `CoverageEligibilityResponse` giả: `inforce: true`, `excluded` theo service type `30`.

✅ Bot mẫu khác: `examples/medplum-demo-bots/src/eligibility-check-opkit.ts` (đối tác Opkit, [2b](./2b-bots.md#6-bot-mẫu-cho-thấy-nghiệp-vụ-gì)).

✅ App Provider (`examples/medplum-provider/src/components/insurance/CheckEligibilityModal.tsx`):
- Tạo `CoverageEligibilityRequest` (`purpose: ['benefits']`) **trước** (`:107-110`), rồi `POST …/$submit` (`:118`).
- Docblock: *"A failed `$submit` is reported but the request is kept"* (`:48`).

🧠 Đúng V3 của file G: Request tồn tại độc lập với kết quả, nên mọi lần hỏi đều có dấu vết, kể cả lần payer không trả lời. Cùng tinh thần với `MedicationRequest` "không biết đã gửi chưa" ở [3e](./3e-thuoc-ke-don.md#4-bài-toán-không-biết-đơn-đã-gửi-chưa).

## 6. Đối chiếu OpenEMR: X12 270 tự dựng

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Lưu bảo hiểm | `Coverage` (1 resource / hợp đồng) | `insurance_data` (`sql/database.sql:3310`): `type` = primary/secondary/tertiary, ~20 cột `subscriber_*` **chép thẳng** thông tin người đứng tên, `copay varchar` |
| Người đứng tên | Tham chiếu tới Patient/RelatedPerson | Chép giá trị vào mỗi dòng. Người cha có 3 con cùng bảo hiểm = 3 bản chép |
| Kiểm tra quyền lợi | `$submit` → Bot | Code PHP **tự dựng chuỗi X12 270** từng segment ISA/GS/ST/BHT/HL/NM1/DMG/DTP/EQ (`src/Billing/EDI270.php:50-316`) |
| Service type | Theo `item.category` | **Cố định `30`** (`createEQ`, `:272-280`) |
| Gửi | Tuỳ Bot | HTTP tới `x12_eligibility_endpoint` của đối tác X12 (`requestEligibility`, `:776+`) |
| Kết quả | `CoverageEligibilityResponse` | Bảng `eligibility_verification` (`:1651`), `benefit_eligibility` (`:14109`). Phân tích 271 ở `interface/billing/edi_271.php` |

🧠 OpenEMR **làm được ngay** (có sẵn X12) nhưng **cứng**: chỉ hỏi tầng 2, đổi đối tác hay hỏi tầng 3 là sửa PHP. Medplum **chưa làm được gì** khi chưa có Bot, nhưng **linh hoạt**. Đánh đổi kinh điển giữa sản phẩm đóng gói và nền tảng.

## Sai lầm thường gặp

1. **Gọi `$submit` mà chưa cấu hình setting.** Nhận `400`, không phải lỗi mạng.
2. **Lưu số thành viên của beneficiary thay vì của subscriber.** Payer tra theo subscriber (US Core yêu cầu subscriber ID).
3. **Không đặt `Coverage.order`.** Claim gửi nhầm payer phụ trước, bị từ chối "COB".
4. **Kiểm tra theo CPT.** 270/271 trả lời theo service type.
5. **Xoá `CoverageEligibilityRequest` khi gửi lỗi.** Mất bằng chứng đã hỏi.
6. **Tin Bot mẫu.** Nó trả kết quả giả.

## Tự kiểm tra

1. Bà Lan có Medicare (order 1) và Medigap (order 2), con trai đứng tên Medigap. Viết 2 `Coverage`.
2. Vì sao `$submit` đi qua 3 lớp gián tiếp thay vì gọi thẳng Bot?
3. Lễ tân kiểm tra lúc 8h, bà Lan khám lúc 10h và dùng hết khấu trừ còn lại. Bệnh nhân kế tiếp cùng gói, cùng ngày có bị ảnh hưởng không? (Gợi ý: khấu trừ là theo **người**, không theo gói.)
