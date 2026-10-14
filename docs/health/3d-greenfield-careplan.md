# 3d-G — Thiết kế từ đầu: kế hoạch chăm sóc bệnh mạn tính

> **Loại file:** G (greenfield).
>
> Trả lời câu: *"Bệnh nhân tiểu đường cần 1 chuỗi việc lặp lại: HbA1c mỗi 3 tháng, khám mắt hằng năm, khám bàn chân mỗi lần tái khám. Ai giao việc, ai làm, ai biết việc nào bị bỏ sót?"*
>
> Đọc tiếp: [`3d-careplan-task.md`](./3d-careplan-task.md) (R).
>
> **Lưu ý trung thực:** viết sau khi đã đọc tên các hàm trong `plandefinitionapply.ts`. Che Phần 5 khi tự luyện.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bối cảnh giả định](#phần-1--bối-cảnh-giả-định-)
- [Phần 2 — BA view](#phần-2--ba-view)
- [Phần 3 — Mô hình khái niệm](#phần-3--mô-hình-khái-niệm-)
- [Phần 4 — Developer view: V1, V2, V3](#phần-4--developer-view-v1-v2-v3-)
- [Phần 5 — Đối chiếu Medplum thật](#phần-5--đối-chiếu-medplum-thật-)
- [Phần 6 — Bảng so sánh tổng hợp](#phần-6--bảng-so-sánh-tổng-hợp)
- [Phần 7 — Đánh giá & luyện phản biện](#phần-7--đánh-giá--luyện-phản-biện-)
- [Phần 8 — Checklist tái sử dụng](#phần-8--checklist-tái-sử-dụng)

## Quy tắc nhãn nguồn

💭 bối cảnh và V1–V3 tự dựng · ✅ code thật (chi tiết ở R) · 📘 chuẩn/docs · 🧠 suy luận.

---

## Phần 1 — Bối cảnh giả định 💭

> **Trưởng khoa Nội tiết An Bình** nói: *"Mình có 800 bệnh nhân tiểu đường. Theo phác đồ thì ai cũng phải xét nghiệm HbA1c 3 tháng/lần, khám mắt 1 năm/lần, khám bàn chân mỗi lần tái khám, người trên 40 tuổi thêm mỡ máu. Thực tế không ai theo dõi được ai đang trễ. Mình muốn mỗi điều dưỡng sáng ra có danh sách việc."*

Kịch bản: bà Lan được chẩn đoán tiểu đường type 2 hôm nay. BS Minh bấm "Áp dụng phác đồ tiểu đường".

📘 Các mốc phác đồ trên là ví dụ minh hoạ dựa trên khuyến cáo phổ biến (ADA). Không dùng làm chỉ dẫn lâm sàng.

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

1. *Ai nói?* → Trưởng khoa.
2. *Ai thao tác hằng ngày?* → **Điều dưỡng điều phối** (gọi bệnh nhân đi xét nghiệm), bác sĩ (duyệt kết quả), bệnh nhân (tự đo đường huyết ở nhà).
3. *Ai nhận kết quả?* → Trưởng khoa (tỉ lệ tuân thủ phác đồ), **bảo hiểm** (trả thưởng theo chất lượng, [4d](./4d-do-luong-chat-luong.md)).
4. *Bên thứ 3?* → Phòng xét nghiệm, phòng khám mắt đối tác.
5. *Ai phát hiện nếu sai?* → Bệnh nhân bị biến chứng mắt vì 2 năm không ai nhắc khám mắt. Phát hiện **muộn nhiều năm**.

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Điều dưỡng điều phối | Danh sách việc **của mình hôm nay**, theo thứ tự ưu tiên | Không phải người ra quyết định mua phần mềm |
| Bác sĩ | Không phải tự nhớ phác đồ | Họ nói "tôi nhớ được" |
| Trưởng khoa | % bệnh nhân đúng phác đồ | Câu hỏi này cần dữ liệu **việc chưa làm**, không chỉ việc đã làm |
| Bệnh nhân | Biết mình cần làm gì, khi nào | App bệnh nhân thường làm sau |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | HbA1c mỗi 3 tháng | "Mỗi 3 tháng" tính từ lần trước hay theo lịch cố định? |
| 2 | Tường minh | Khám mắt mỗi năm | Bệnh nhân đã khám ở nơi khác thì sao? |
| 3 | Tường minh | Khám bàn chân **mỗi lần tái khám** | Việc gắn với **sự kiện**, không gắn với thời gian |
| 4 | Tường minh | Trên 40 tuổi thêm mỡ máu | **Điều kiện** theo đặc điểm bệnh nhân |
| 5 | Ẩn | Phác đồ thay đổi (hội nội tiết cập nhật) | Bệnh nhân đang theo phác đồ cũ thì sao? |
| 6 | Ẩn | Việc phải có **người chịu trách nhiệm** | Giao cho người hay cho vai trò ("điều dưỡng ca sáng")? |
| 7 | Ẩn | Việc chưa làm **phải nhìn thấy được** | Quá hạn bao lâu thì báo ai? |
| 8 | Ẩn | Kết quả HbA1c về thì việc "xét nghiệm HbA1c" tự đóng | Ai nối kết quả với việc? |

**Cách tìm rule ẩn #5:** hỏi *"phác đồ này sống bao lâu?"* Phác đồ là **văn bản có phiên bản**. Kế hoạch của bà Lan phải nhớ nó sinh ra từ phiên bản nào.

**Cách tìm rule ẩn #7:** hỏi trưởng khoa *"báo cáo bạn muốn xem cuối tháng là gì?"* Câu trả lời ("ai đang trễ") đòi hỏi việc chưa làm phải **tồn tại thành dữ liệu**, không chỉ là "không có kết quả".

### Bước 3 — 7 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh |
|---|---|---|
| Q1 | Phác đồ do ai soạn, sửa bao lâu 1 lần? | Phác đồ là dữ liệu (cấu hình được) hay code |
| Q2 | Áp phác đồ là **1 lần** (sinh hết việc) hay **liên tục** (đánh giá lại định kỳ)? | Sinh việc 1 lần hay engine chạy nền |
| Q3 | Việc giao cho người cụ thể hay hàng đợi theo vai trò? | `owner` vs `performerType` |
| Q4 | Có điều kiện theo bệnh nhân (tuổi, bệnh kèm)? | Phải đánh giá biểu thức lúc áp |
| Q5 | Việc có thứ tự (A xong mới tới B)? | Phụ thuộc giữa việc |
| Q6 | Việc hoàn thành bằng cách nào: bấm tay hay tự động khi có kết quả? | Liên kết việc ↔ dữ liệu lâm sàng |
| Q7 | Bệnh nhân có thấy và tự làm 1 số việc không? | Việc có người làm là bệnh nhân |

**Giả định trả lời:** Q1 hội đồng chuyên môn sửa ~1 lần/năm · Q2 chưa rõ · Q3 hàng đợi theo vai trò · Q4 có · Q5 ít · Q6 muốn tự động · Q7 có (tự đo đường huyết).

### Sai lầm thường gặp khi làm BA view

1. **Chỉ mô hình hoá "đã làm".** Kết quả xét nghiệm có sẵn trong hệ thống. "Chưa làm" thì không. Không có thực thể "việc" thì không đo được tuân thủ.
2. **Coi phác đồ là danh sách cố định.** Bỏ qua điều kiện (#4) và sự kiện (#3).
3. **Không hỏi Q2.** Đây là câu quyết định kiến trúc: sinh việc 1 lần thì đơn giản nhưng không tự sinh lần HbA1c tiếp theo.

---

## Phần 3 — Mô hình khái niệm 💭

| Thực thể | Vì sao là thực thể thật |
|---|---|
| **Phác đồ** (mẫu) | Có phiên bản, có người soạn, dùng cho nhiều bệnh nhân |
| **Bước trong phác đồ** | Có tên, điều kiện, chu kỳ, vai trò thực hiện |
| **Định nghĩa hoạt động** dùng lại (xét nghiệm HbA1c, phiếu đánh giá bàn chân) | Dùng chung giữa nhiều phác đồ |
| **Kế hoạch của 1 bệnh nhân** | Sinh ra từ phác đồ, có trạng thái riêng (đang theo / tạm dừng / kết thúc) |
| **Việc** (1 lần làm cụ thể) | Có người làm, hạn, trạng thái, kết quả |
| **Chỉ định** (yêu cầu xét nghiệm gửi lab) | Là văn bản y khoa, khác "việc của điều dưỡng gọi bệnh nhân đi xét nghiệm" |
| **Mục tiêu** (HbA1c < 7%) | Đo được, đánh giá theo thời gian |

**Phân biệt Việc và Chỉ định:** Chỉ định = *bác sĩ yêu cầu lab làm xét nghiệm X* (pháp lý, gửi đi). Việc = *ai đó trong phòng khám phải làm cho chỉ định đó xảy ra* (vận hành). 1 chỉ định có thể sinh nhiều việc (gọi bệnh nhân, lấy mẫu, nhập kết quả).

**Cầu nối FHIR:**

| Khái niệm | FHIR R4 | Lớp quyết định |
|---|---|---|
| Phác đồ | `PlanDefinition` (`action[]`, `version`, `url`) | HL7 |
| Định nghĩa hoạt động | `ActivityDefinition`, `Questionnaire` | HL7 |
| Điều kiện / thứ tự / chu kỳ | `action.condition`, `action.relatedAction`, `action.timing[x]` | HL7 |
| Áp phác đồ | `PlanDefinition/$apply` → `RequestGroup` (R4) / `RequestOrchestration` (R5) | HL7 định nghĩa, **mức hỗ trợ do server** |
| Kế hoạch bệnh nhân | `CarePlan` | HL7 |
| Việc | `Task` (`status`, `businessStatus`, `owner`, `performerType`, `focus`) | HL7 |
| Chỉ định | `ServiceRequest` | HL7 |
| Mục tiêu | `Goal` | HL7 |
| Mẫu phác đồ y khoa | Có thể theo **CPG IG** (Clinical Practice Guidelines) | IG quốc tế |

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — Danh sách việc tay

Bác sĩ tự tạo `Task` "xét nghiệm HbA1c, hạn 3 tháng nữa". Không có phác đồ.

**Dừng ở V1 khi nào:** vài chục bệnh nhân mạn tính, 1 bác sĩ.

### V2 — Phác đồ là mẫu, áp 1 lần sinh danh sách việc

```
PlanDefinition "Tiểu đường v2026"
  ├─ action: Xét nghiệm HbA1c  → ActivityDefinition(kind=ServiceRequest, code=LOINC 4548-4)
  ├─ action: Khám mắt          → ActivityDefinition(kind=ServiceRequest)
  ├─ action: Đánh giá bàn chân → Questionnaire
  └─ action: Mỡ máu            (condition: tuổi > 40)

$apply(bà Lan) → CarePlan ─▶ RequestGroup ─▶ 4 Task (+ ServiceRequest nháp)
```

Điểm mù của V2:
- **Chu kỳ:** sinh 1 Task HbA1c. 3 tháng sau ai sinh Task tiếp theo?
- **Sự kiện:** "mỗi lần tái khám" không phải việc sinh lúc áp.
- **Điều kiện:** nếu engine không đánh giá `condition`, bà Lan 68 tuổi vẫn đúng, nhưng bệnh nhân 30 tuổi cũng bị giao xét nghiệm mỡ máu.
- **Nửa chừng:** áp được 2/4 việc thì lỗi mạng. Còn lại 1 kế hoạch dở.

### V3 — Áp 1 lần + "động cơ" chạy nền

1. `$apply` sinh **việc trước mắt**, trong 1 giao dịch.
2. **Bot cron** hằng đêm ([2b](./2b-bots.md)): với mỗi CarePlan `active`, tính việc đến hạn (HbA1c lần cuối > 80 ngày, chưa có Task mở) → sinh Task.
3. **Subscription** ([2a](./2a-subscriptions.md)) trên `Observation` code HbA1c → đóng Task tương ứng, cập nhật `Goal`.
4. **Subscription** trên `Encounter` loại "tái khám" → sinh Task "đánh giá bàn chân".
5. Hàng đợi = search `Task?status=requested&performer-type=điều-dưỡng&_sort=priority,period`.

**Dừng ở V3 khi nào:** nhiều bệnh mạn tính, cần báo cáo tuân thủ, có hợp đồng trả theo chất lượng.

### 3 lối tắt "nghe hợp lý nhưng sai"

1. **"Sinh sẵn Task HbA1c cho 5 năm (20 Task)."** Phác đồ đổi ở năm 2 thì 12 Task sai. Bệnh nhân chuyển viện thì 20 Task mồ côi.
2. **"Không cần Task, cứ search 'bệnh nhân tiểu đường chưa có HbA1c 90 ngày qua' là ra danh sách trễ."** Ra danh sách, nhưng không có **người chịu trách nhiệm**, không có trạng thái "đã gọi, bệnh nhân hẹn tuần sau". Đây là báo cáo, không phải hàng đợi việc.
3. **"Viết engine CQL đầy đủ."** Đúng chuẩn nhất, nhưng tốn nhiều tháng. Phần lớn phác đồ phòng khám chỉ cần FHIRPath đơn giản + cron.

---

## Phần 5 — Đối chiếu Medplum thật ✅

Chi tiết: [`3d-careplan-task.md`](./3d-careplan-task.md). Đường dẫn từ `packages/server/src/fhir/operations/`.

| Thiết kế | Medplum | ✅ |
|---|---|---|
| `$apply` sinh CarePlan + RequestGroup + Task | Có, đúng cấu trúc V2 | `plandefinitionapply.ts:66-134` |
| Action trỏ Questionnaire → Task có input Questionnaire | Có | `:164-172` |
| Action trỏ ActivityDefinition `kind=ServiceRequest` → ServiceRequest **nháp** + Task | Có. Kind khác (MedicationRequest…) chỉ sinh Task | `:247-286` |
| `dynamicValue` (FHIRPath) điền giá trị vào ServiceRequest | Có, chỉ `text/fhirpath` | `:340-362` |
| Giao cho vai trò / người | Extension `task-elements` (`owner` bằng biểu thức, `performerType`) | `:220-245` |
| **`condition`** (tuổi > 40) | **Không đánh giá** (không có chỗ nào đọc `action.condition`) | grep toàn file |
| **Action lồng nhau**, `relatedAction`, `timing` | **Không xử lý**, chỉ duyệt `planDefinition.action` cấp 1 | `:93` |
| Giao dịch (tất cả hoặc không) | **Không**: mỗi resource 1 `createResource` riêng | toàn file |
| Chu kỳ, sự kiện (V3 bước 2–4) | Không có trong server. Làm bằng Bot/Subscription | — |

📘 Docs `packages/docs/docs/careplans/protocols.md` hướng dẫn **cách khai báo** `condition`, `relatedAction`, action lồng nhau, nhưng bảng "Action Definition Processing" chỉ mô tả 3 kiểu sinh Task. Người đọc docs dễ tưởng server đánh giá điều kiện.

OpenEMR: dùng **Clinical Decision Rules** (`src/ClinicalDecisionRules/`, bảng `clinical_rules` `sql/database.sql:640`, `rule_action` `:8894`, `patient_reminders` `:8547`). Quy tắc chạy định kỳ, sinh **nhắc nhở** (reminder), không sinh việc có người chịu trách nhiệm. Gần với lối tắt #2.

---

## Phần 6 — Bảng so sánh tổng hợp

| | V1 | V2 / Medplum `$apply` | V3 | OpenEMR CDR |
|---|---|---|---|---|
| Phác đồ là dữ liệu | ✘ | ✔ | ✔ | ✔ |
| Điều kiện theo bệnh nhân | — | ✘ (Medplum chưa đánh giá) | ✔ (Bot) | ✔ |
| Chu kỳ lặp | Tay | ✘ | ✔ cron | ✔ |
| Việc có người chịu trách nhiệm | ✔ | ✔ | ✔ | ✘ (nhắc nhở) |
| Tự đóng việc khi có kết quả | ✘ | ✘ | ✔ Subscription | — |
| Nguyên tử | — | ✘ | ✔ (nếu tự gói transaction) | — |

---

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Vì sao có khoảng cách V2 → V3:** `$apply` là **phép biến đổi 1 lần** (mẫu → bản sao cho 1 bệnh nhân). Chăm sóc mạn tính là **quá trình liên tục**. FHIR cố ý tách 2 việc này: spec định nghĩa `$apply`, còn việc "đánh giá lại liên tục" để cho hệ thống. Medplum giữ đúng ranh giới đó, và để phần liên tục cho Bot.

**Khi nào KHÔNG nên lên V3:** phác đồ ngắn hạn (chuẩn bị mổ: 5 việc trong 1 tuần). V2 là đủ.

### Luyện phản biện

**Phản đối 1 (kinh doanh):** "Bảo hiểm trả thêm tiền khi đạt tỉ lệ HbA1c, sao không chỉ làm báo cáo?"

<details><summary>Đáp án gợi ý</summary>

Báo cáo cho biết **đã trễ**, Task cho biết **ai phải làm gì hôm nay để không trễ**. Tiền thưởng chất lượng ([4d](./4d-do-luong-chat-luong.md)) trả theo kết quả cuối năm. Chỉ có báo cáo thì tới tháng 12 mới biết thiếu, không kịp sửa. Task biến chỉ số chất lượng thành việc vận hành hằng ngày.
</details>

**Phản đối 2 (kỹ thuật):** "`$apply` không nguyên tử, vậy đừng dùng, tự viết Bot."

<details><summary>Đáp án gợi ý</summary>

Có 2 cách khắc phục nhẹ hơn: (1) gọi `$apply` rồi kiểm tra số Task sinh ra bằng số action, sai thì huỷ; (2) Bot tự đọc PlanDefinition, dựng 1 Bundle `transaction`. Cách 2 cũng là chỗ để đánh giá `condition`. Nhưng `$apply` vẫn đáng dùng cho phác đồ đơn giản vì giữ đúng liên kết chuẩn (`instantiatesCanonical`, `basedOn`).
</details>

**Phản đối 3 (lâm sàng):** "Sinh Task tự động làm điều dưỡng bị ngập việc."

<details><summary>Đáp án gợi ý</summary>

Đúng, nên hàng đợi phải có `priority`, `businessStatus` (đã gọi / hẹn lại / từ chối), và quy tắc gộp (1 lần gọi bệnh nhân giải quyết 3 việc). Đo tải hằng ngày trước khi bật: 800 bệnh nhân × (4 HbA1c + 1 mắt + ~4 bàn chân)/năm ≈ 7.200 việc/năm ≈ 30 việc/ngày làm việc. Chia cho 2 điều dưỡng là chấp nhận được.
</details>

---

## Phần 8 — Checklist tái sử dụng

- [ ] Phân biệt **Việc** (vận hành) và **Chỉ định** (y khoa)
- [ ] Phác đồ có phiên bản, kế hoạch bệnh nhân nhớ phiên bản sinh ra nó
- [ ] Đã xác định việc nào sinh **1 lần**, việc nào **theo chu kỳ**, việc nào **theo sự kiện**
- [ ] Điều kiện theo bệnh nhân được đánh giá ở đâu (server không làm thì Bot làm)
- [ ] Mỗi việc có `owner` hoặc `performerType`
- [ ] Việc tự đóng khi dữ liệu lâm sàng tương ứng xuất hiện
- [ ] Báo cáo tuân thủ đọc từ Task, không chỉ từ kết quả

**Tự luyện:** viết Bot cron cho rule "HbA1c mỗi 3 tháng" theo V3 bước 2. Xử lý: bệnh nhân vừa xét nghiệm ở nơi khác (Observation do lab ngoài gửi về qua HL7, [2c](./2c-agent-hl7v2.md)).

**Nguồn:**
- 📘 FHIR R4 PlanDefinition `$apply`: https://hl7.org/fhir/R4/plandefinition-operation-apply.html
- 📘 Docs Medplum `packages/docs/docs/careplans/`
