# 4a-G — Thiết kế từ đầu: kiểm tra quyền lợi bảo hiểm trước khi khám

> **Loại file:** G (greenfield). Giai đoạn 4 (revenue cycle) học kỹ hơn mức trung bình vì sát chuyên môn tài chính.
>
> Trả lời câu: *"Lễ tân cần biết trước khi khám: bảo hiểm còn hiệu lực không, dịch vụ này có được chi trả không, bệnh nhân tự trả bao nhiêu?"*
>
> Đọc tiếp: [`4a-quyen-loi-bao-hiem.md`](./4a-quyen-loi-bao-hiem.md) (R) · [`4e-phu-luc-bhyt-viet-nam.md`](./4e-phu-luc-bhyt-viet-nam.md) (đối chiếu BHYT Việt Nam).
>
> **Lưu ý trung thực:** viết sau khi đã biết `$submit` của Medplum chỉ là bộ điều phối. Che Phần 5 khi tự luyện.

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

💭 bối cảnh, số liệu giả định, V1–V3 · ✅ code thật (chi tiết ở R) · 📘 chuẩn, quy định, docs · 🧠 suy luận.

---

## Phần 1 — Bối cảnh giả định 💭

Bối cảnh đặt ở **Mỹ** để khớp với Medplum. Bản Việt Nam ở [phụ lục 4e](./4e-phu-luc-bhyt-viet-nam.md).

> **Giám đốc tài chính chuỗi phòng khám An Bình (chi nhánh Mỹ)** nói: *"Tỉ lệ claim bị từ chối của mình là 12%, gần một nửa vì 'bảo hiểm hết hiệu lực' hoặc 'dịch vụ không thuộc quyền lợi'. Bệnh nhân khám xong mới biết phải tự trả 180 đô, nhiều người không trả. Mình muốn lễ tân kiểm tra trước, và thu phần bệnh nhân tự trả ngay tại quầy."*

Kịch bản: bà Lan (đã sang Mỹ, 68 tuổi) có **2 bảo hiểm**:
1. **Medicare** (liên bang, cho người ≥ 65 tuổi), là bảo hiểm **chính**.
2. 1 gói **bổ sung** (Medigap) qua công ty con trai, con trai là **người đứng tên hợp đồng**, bà là người thụ hưởng.

Lần khám: tái khám tiểu đường + xét nghiệm HbA1c.

🧠 Con số 12% và "gần một nửa do quyền lợi" là **giả định** của bối cảnh. Các báo cáo ngành ở Mỹ hay nêu tỉ lệ từ chối ban đầu ở mức 2 chữ số và lỗi đăng ký/quyền lợi là nhóm nguyên nhân lớn, nhưng file này không trích nguồn cụ thể. Khi làm thật, lấy số từ dữ liệu từ chối (mã CARC trên 835, [4c](./4c-claim.md)) của chính khách hàng.

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

Áp thêm 2 câu hỏi riêng của y tế (K6): **ai trả tiền**, **ai kiểm tra**.

1. *Ai nói?* → Giám đốc tài chính.
2. *Ai thao tác hằng ngày?* → **Lễ tân** (kiểm tra, thu tiền), **nhân viên billing** (xử lý từ chối).
3. *Ai nhận kết quả?* → Bệnh nhân (biết trước số tiền), bác sĩ (biết dịch vụ nào cần xin duyệt trước).
4. *Ai trả tiền?* → **Payer** (Medicare, công ty bảo hiểm), **bệnh nhân** (copay, đồng chi trả, khấu trừ). Có khi **người thứ 3** (người đứng tên hợp đồng).
5. *Ai kiểm tra?* → Payer (kiểm tra claim), cơ quan quản lý (minh bạch giá).

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Lễ tân | Kết quả trong **vài giây**, hiểu được | Phản hồi của payer rất kỹ thuật (mã X12) |
| Nhân viên billing | Biết **lý do** từ chối để sửa | Không có mặt ở buổi họp tiếp nhận |
| Bệnh nhân | Số tiền **trước khi** khám | Thường chỉ được nghĩ tới như người nhận hoá đơn |
| Người đứng tên hợp đồng | Thông tin của họ phải đúng (tên, ngày sinh) | **Không phải bệnh nhân**, không có mặt |
| Clearinghouse | Trung gian chuyển 270/271 giữa phòng khám và hàng trăm payer | "Hạ tầng vô hình", phí theo giao dịch |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | Kiểm tra bảo hiểm còn hiệu lực trước khi khám | Bao lâu trước? Lúc đặt lịch hay lúc đến? |
| 2 | Tường minh | Biết dịch vụ có được chi trả không | Dịch vụ cụ thể (CPT) hay **loại** dịch vụ? |
| 3 | Tường minh | Biết bệnh nhân tự trả bao nhiêu | Copay cố định, đồng chi trả %, hay còn khấu trừ (deductible)? |
| 4 | Ẩn | Bệnh nhân có **nhiều** bảo hiểm, có **thứ tự** | Ai trả trước? (Coordination of Benefits) |
| 5 | Ẩn | Người đứng tên ≠ người được bảo hiểm | Payer tra theo **người đứng tên** |
| 6 | Ẩn | Quyền lợi thay đổi trong năm (khấu trừ đã dùng bao nhiêu) | Kết quả kiểm tra **có hạn dùng** |
| 7 | Ẩn | Kết quả kiểm tra là **bằng chứng** khi tranh chấp | Lưu nguyên văn phản hồi của payer, bao lâu? |
| 8 | Ẩn | Kiểm tra tốn phí (clearinghouse tính theo giao dịch) | Kiểm tra lại bao nhiêu lần là đủ? |

**Cách tìm rule ẩn #4:** hỏi *"bệnh nhân có thể có mấy bảo hiểm?"* Trả lời "1" là sai ở Mỹ: Medicare + gói bổ sung, bảo hiểm của bố mẹ + của bản thân, bảo hiểm tai nạn lao động.

**Cách tìm rule ẩn #6:** hỏi *"kết quả kiểm tra hôm qua còn đúng hôm nay không?"* Khấu trừ (deductible) giảm dần mỗi lần khám. Đầu năm bà Lan tự trả 100%, cuối năm có thể 0%.

### Bước 3 — 8 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh |
|---|---|---|
| Q1 | Kết nối payer trực tiếp hay qua clearinghouse? | Bao nhiêu giao thức phải hỗ trợ |
| Q2 | Kiểm tra theo **loại** dịch vụ (X12 service type) hay **từng mã CPT**? | Độ chi tiết của request |
| Q3 | Kiểm tra lúc nào: đặt lịch, 2 ngày trước, lúc đến? | Tự động (cron) hay theo sự kiện |
| Q4 | Có bao nhiêu bảo hiểm mỗi bệnh nhân? | Danh sách có thứ tự |
| Q5 | Kết quả hiển thị cho ai, mức chi tiết nào? | Diễn giải mã X12 sang ngôn ngữ người |
| Q6 | Lưu kết quả bao lâu, làm gì khi kết quả cũ? | Bản ghi kết quả có ngày, có liên kết |
| Q7 | Dịch vụ cần **xin duyệt trước** (prior authorization)? | 1 luồng riêng, chậm hơn nhiều (ngày, không phải giây) |
| Q8 | Ai chịu chi phí giao dịch kiểm tra? | Giới hạn tần suất |

### Sai lầm thường gặp khi làm BA view

1. **Coi "kiểm tra bảo hiểm" là 1 câu có/không.** Thực tế là 3 tầng: còn hiệu lực → chi trả khám cơ bản → chi trả loại dịch vụ cụ thể (📘 `packages/docs/docs/billing/insurance-eligibility-checks.md`, mục "Use Cases").
2. **Lưu 1 bảo hiểm trên hồ sơ bệnh nhân** (các cột trong bảng bệnh nhân). Không có chỗ cho bảo hiểm thứ 2, không có thứ tự.
3. **Không hỏi nhân viên billing.** Họ biết lý do từ chối phổ biến nhất, tức là biết kiểm tra trước cần bắt được cái gì.

---

## Phần 3 — Mô hình khái niệm 💭

| Thực thể | Vì sao là thực thể thật |
|---|---|
| **Payer** (công ty bảo hiểm, Medicare) | Ký hợp đồng, có mã định danh riêng (payer ID ở clearinghouse) |
| **Hợp đồng bảo hiểm** của 1 người | Có số thành viên, nhóm, gói, ngày hiệu lực, **người đứng tên**, **người thụ hưởng**, thứ tự |
| **Yêu cầu kiểm tra** | Có thời điểm, loại dịch vụ hỏi, gửi tới đâu. Là **bằng chứng đã hỏi** |
| **Kết quả kiểm tra** | Có thời điểm, nguyên văn phản hồi, quyền lợi từng loại dịch vụ. Là **bằng chứng payer đã trả lời** |
| **Quyền lợi** (từng dòng trong kết quả) | Loại dịch vụ, mức copay, %, khấu trừ còn lại, cần duyệt trước không |

**Ma trận quan hệ:**

| | Người | Hợp đồng | Payer |
|---|---|---|---|
| **Hợp đồng** | Có 1 **người đứng tên** và 1 **người thụ hưởng** (có thể khác nhau), kèm quan hệ (con, vợ/chồng) | — | Thuộc đúng 1 payer |
| **Người** | — | 1 người có **N** hợp đồng, có thứ tự 1, 2, 3 | — |
| **Kết quả kiểm tra** | — | Về đúng 1 hợp đồng, tại 1 thời điểm | Do 1 payer trả |

**Cầu nối FHIR:**

| Khái niệm | FHIR R4 | Lớp quyết định |
|---|---|---|
| Payer | `Organization` | HL7 |
| Hợp đồng | `Coverage` (`subscriber`, `beneficiary`, `relationship`, `payor`, `order`, `period`, `costToBeneficiary`, `class` nhóm/gói) | HL7 |
| Số thành viên | `Coverage.subscriberId` hoặc `identifier` type `MB` | **US Core** |
| Loại gói (HMO, PPO, Medicare…) | `Coverage.type` theo **SOPT** | **US Core** |
| Yêu cầu kiểm tra | `CoverageEligibilityRequest` (`purpose`, `item.category` = X12 service type) | HL7 |
| Kết quả | `CoverageEligibilityResponse` (`insurance.inforce`, `item.benefit`, `authorizationRequired`) | HL7 |
| Gửi kiểm tra | Operation `$submit` | HL7 có định nghĩa, **cách xử lý là Medplum** |
| Giao dịch thật với payer | **X12 270 / 271** (chuẩn EDI của Mỹ) | Quy định HIPAA |

📘 Lớp quy định (K3): HIPAA quy định **X12 270/271** là chuẩn bắt buộc cho kiểm tra quyền lợi điện tử ở Mỹ. FHIR chỉ là cách Medplum biểu diễn bên trong. Ra ngoài vẫn phải đổi sang X12, thường do clearinghouse làm.

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — Cột bảo hiểm trên hồ sơ, gọi điện cho payer

```sql
ALTER TABLE patient ADD insurance_company TEXT, ADD member_id TEXT, ADD copay TEXT;
```
Lễ tân gọi điện cho payer hoặc vào cổng web của payer, ghi tay kết quả. Mỗi lần ~10 phút.

**Dừng ở V1 khi nào:** phòng khám nhỏ, tự trả tiền là chính.

### V2 — Bảng hợp đồng riêng + 1 lần gọi API tới 1 clearinghouse

```
coverage(id, patient_id, payer_id, member_id, subscriber_id, relationship, order, period)
eligibility_check(id, coverage_id, service_type, checked_at, raw_response, active, copay, deductible_remaining)
```
1 nút "Kiểm tra" → gọi API clearinghouse → lưu kết quả.

Điểm mù của V2:
- **Gắn chặt 1 clearinghouse.** Đổi đối tác (vì giá) là viết lại.
- **Không có bản ghi "yêu cầu"** riêng, chỉ có kết quả. Gọi thất bại thì không có dấu vết đã hỏi.
- **Kiểm tra đồng bộ** trong request của lễ tân. Payer chậm 30 giây thì màn hình treo.

### V3 — Yêu cầu và kết quả là 2 bản ghi, gửi qua 1 cổng điều phối cấu hình được

1. Lễ tân (hoặc cron 2 ngày trước lịch hẹn, [3b](./3b-dat-lich.md)) tạo **Yêu cầu** kiểm tra cho từng hợp đồng theo thứ tự.
2. Gọi `$submit`. Server **không biết** clearinghouse nào. Nó đọc **cấu hình project** để biết chuyển cho bộ xử lý nào (Bot).
3. Bot đổi sang X12 270 (hoặc API JSON của clearinghouse), gửi, nhận 271, đổi ngược thành **Kết quả**.
4. Kết quả gắn vào Yêu cầu và Hợp đồng. Hiển thị cho lễ tân bằng ngôn ngữ người.
5. Thất bại thì **Yêu cầu vẫn còn**, có trạng thái lỗi, thử lại được.

**Dừng ở V3 khi nào:** nhiều payer, muốn đổi clearinghouse, cần bằng chứng khi tranh chấp.

### 3 lối tắt "nghe hợp lý nhưng sai"

1. **"Kiểm tra 1 lần lúc đăng ký là đủ."** Khấu trừ và hiệu lực đổi theo tháng (rule #6).
2. **"Tự kết nối thẳng từng payer cho rẻ."** Hàng trăm payer, mỗi nơi 1 hợp đồng kết nối, 1 bộ thông tin đăng nhập. Clearinghouse tồn tại chính vì việc này.
3. **"Lưu kết quả đã diễn giải (copay = 20$), bỏ nguyên văn."** Khi payer từ chối claim và nói "lúc đó bà không có quyền lợi", nguyên văn 271 là bằng chứng duy nhất.

---

## Phần 5 — Đối chiếu thực tế ✅

Chi tiết: [`4a-quyen-loi-bao-hiem.md`](./4a-quyen-loi-bao-hiem.md).

| Thiết kế V3 | Medplum | ✅ |
|---|---|---|
| Yêu cầu và Kết quả là 2 resource | `CoverageEligibilityRequest` / `CoverageEligibilityResponse` (chuẩn FHIR) | — |
| Cổng điều phối cấu hình được | `$submit` đọc setting `ELIGIBILITY_SUBMIT_OPERATION` của project rồi chuyển cho operation tuỳ biến (Bot) | `packages/server/src/fhir/operations/coverageeligibilityrequestsubmit.ts:11`, `:45`; `utils/submit.ts:21-51` |
| Chưa cấu hình thì báo lỗi rõ | `400 … is not configured: set the … project setting` | `utils/submit.ts:29-32` |
| Yêu cầu còn khi gửi lỗi | App Provider: tạo Request trước, `$submit` sau. Lỗi `$submit` được báo nhưng Request được giữ | `examples/medplum-provider/src/components/insurance/CheckEligibilityModal.tsx:48`, `:107-118` |
| Bot đổi sang clearinghouse | Bot mẫu chỉ **giả lập** phản hồi, chưa nối clearinghouse thật | `examples/medplum-eligibility-demo/src/bots/process-eligibility-request.ts` |
| Nhiều bảo hiểm, có thứ tự | `Coverage.order` (1, 2, 3) | 📘 `packages/docs/docs/billing/patient-insurance.md` |

OpenEMR (commit `ebe3ee8`) **dừng giữa V1 và V2**:
- Bảo hiểm ở bảng `insurance_data` (`sql/database.sql:3310`): `type` = primary/secondary/tertiary, thông tin người đứng tên **chép thẳng vào từng dòng** (`subscriber_fname`, `subscriber_ss`, `subscriber_employer`…), `copay` là `varchar`.
- Tự dựng chuỗi X12 270 bằng code (`src/Billing/EDI270.php`), gửi tới endpoint của đối tác X12 (`:776+`). Dòng `EQ` **cố định** service type `30` (`:272-280`).

---

## Phần 6 — Bảng so sánh tổng hợp

| | V1 | V2 / OpenEMR | V3 / Medplum |
|---|---|---|---|
| Nhiều bảo hiểm có thứ tự | ✘ | ✔ (3 hàng cố định) | ✔ (`order`) |
| Người đứng tên là thực thể riêng | ✘ | ✘ (chép vào dòng) | ✔ (`subscriber` → Patient/RelatedPerson) |
| Yêu cầu là bản ghi riêng | ✘ | ✘ | ✔ |
| Đổi clearinghouse không sửa code lõi | — | ✘ (sửa PHP) | ✔ (đổi Bot + setting) |
| Kiểm tra theo loại dịch vụ | Tay | ✘ (cố định `30`) | ✔ (`item.category`) |
| Có sẵn kết nối clearinghouse | — | ✔ (X12 trực tiếp) | ✘ (phải tự viết Bot) |

---

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Vì sao có khoảng cách V2 → V3:** V2 coi kiểm tra quyền lợi là **1 cuộc gọi API**. V3 coi nó là **1 giao dịch có bằng chứng** (yêu cầu, phản hồi, thời điểm) đi qua **1 điểm tích hợp thay thế được**. Đó là cách nhìn của người làm tài chính: mỗi giao dịch với đối tác trả tiền phải đối soát được.

**Góc nhìn kinh doanh 📘/🧠:**
- Luồng tiền: bệnh nhân trả copay tại quầy → payer trả phần còn lại sau khi duyệt claim ([4c](./4c-claim.md)) → phần bệnh nhân còn nợ được gửi hoá đơn sau.
- Điểm rò rỉ doanh thu mà 4a chặn được:
  - claim bị từ chối vì bảo hiểm hết hiệu lực
  - sai thứ tự bảo hiểm (gửi nhầm payer phụ trước)
  - không thu copay tại quầy (thu sau tốn chi phí, tỉ lệ thu thấp hơn)
  - dịch vụ cần duyệt trước mà không xin
- Ước lượng 💭 cho An Bình: 20.000 lượt khám/năm × 12% bị từ chối × 45% do lỗi quyền lợi × 150$ trung bình × 30% không đòi lại được ≈ **48.600$/năm** mất trắng. Phí clearinghouse ~0,1–0,3$/lượt kiểm tra × 20.000 × 2 lần ≈ 4.000–12.000$/năm. Tỉ suất hoàn vốn dương nếu giả định đúng. **Mọi con số là giả định**, phải thay bằng dữ liệu từ chối thật của khách hàng.

### Luyện phản biện

**Phản đối 1 (tài chính):** "Phí mỗi giao dịch kiểm tra cộng lại rất lớn, sao không kiểm tra ngẫu nhiên?"

<details><summary>Đáp án gợi ý</summary>

Phân tầng theo rủi ro: kiểm tra tự động 100% với bệnh nhân mới, bảo hiểm mới khai, đầu năm (khấu trừ reset), dịch vụ đắt. Bệnh nhân cũ, cùng bảo hiểm, đã kiểm tra trong 30 ngày thì bỏ qua. Đo tỉ lệ từ chối theo từng nhóm trước và sau để tinh chỉnh.
</details>

**Phản đối 2 (kỹ thuật):** "Server Medplum không tự gửi được 270, vậy `$submit` có ích gì?"

<details><summary>Đáp án gợi ý</summary>

`$submit` cố định **giao diện** (app luôn gọi cùng 1 operation chuẩn FHIR) và để **hiện thực** thay đổi theo project. Đổi clearinghouse chỉ đổi Bot và 1 setting, app không đổi. Với nền tảng phục vụ nhiều khách hàng, mỗi khách 1 clearinghouse khác nhau, đây là điểm mở rộng đúng chỗ.
</details>

**Phản đối 3 (vận hành):** "Lễ tân không hiểu kết quả kiểm tra, toàn mã."

<details><summary>Đáp án gợi ý</summary>

Lưu nguyên văn để làm bằng chứng, nhưng hiển thị 3 dòng: **Còn hiệu lực? Dịch vụ hôm nay được chi trả? Bệnh nhân trả tại quầy bao nhiêu?** Bot diễn giải phần này lúc tạo `CoverageEligibilityResponse` (`item.benefit` có `allowedMoney`, `usedMoney`).
</details>

---

## Phần 8 — Checklist tái sử dụng

- [ ] Bảo hiểm là **danh sách có thứ tự**, không phải vài cột trên hồ sơ
- [ ] Người đứng tên và người thụ hưởng là 2 vai trò, có quan hệ
- [ ] Yêu cầu và kết quả kiểm tra là 2 bản ghi có thời điểm
- [ ] Lưu **nguyên văn** phản hồi payer
- [ ] Kiểm tra theo **loại** dịch vụ, có tầng
- [ ] Có lịch kiểm tra lại (trước lịch hẹn, đầu năm)
- [ ] Điểm tích hợp với clearinghouse thay thế được
- [ ] Có luồng riêng cho dịch vụ cần duyệt trước

**Tự luyện tiếp theo:** thiết kế luồng **prior authorization** (xin duyệt trước) cho MRI. Gợi ý: Medplum dùng `Claim` với `use = preauthorization` và setting `PRIOR_AUTH_SUBMIT_OPERATION` (`packages/server/src/fhir/operations/claimsubmit.ts:12`, `:42-43`). Luồng này mất nhiều ngày: cần Task theo dõi ([3d](./3d-careplan-task.md)).

**Nguồn:**
- 📘 X12 Service Type Codes: https://x12.org/codes/service-type-codes
- 📘 Docs Medplum `packages/docs/docs/billing/`
- 📘 FHIR R4 CoverageEligibilityRequest: https://hl7.org/fhir/R4/coverageeligibilityrequest.html
