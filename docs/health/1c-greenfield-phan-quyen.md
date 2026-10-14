# 1c-G — Thiết kế từ đầu: phân quyền cho chuỗi phòng khám

> **Loại file:** G (greenfield, theo [`../templates/saleor-greenfield-design-practice-product-attribute.md`](../templates/saleor-greenfield-design-practice-product-attribute.md)).
>
> Giả định **chưa có 1 dòng code nào**. Đi đúng trình tự: discovery (BA) → mô hình khái niệm → thiết kế V1/V2/V3 (dev) → đối chiếu với Medplum thật. Đọc tiếp:
> - [`1c-access-control.md`](./1c-access-control.md): Medplum làm thật thế nào (R)
> - [`1c-comparison-phan-quyen.md`](./1c-comparison-phan-quyen.md): so sánh 5 kỹ thuật (C)
>
> **Lưu ý trung thực:** quy ước là viết G trước khi đọc code thật. File này được viết **sau** khi đã đọc `fhir/search.ts` và `repo.ts` ở module 1b, nên V1–V3 có thể bị "nhìn trước đáp án". Khi tự luyện, hãy che Phần 5 trở đi và tự làm lại Phần 2–4.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bối cảnh giả định](#phần-1--bối-cảnh-giả-định-)
- [Phần 2 — BA view](#phần-2--ba-view)
  - [Bước 1 — Stakeholder map](#bước-1--stakeholder-map)
  - [Bước 2 — Bảng business rule](#bước-2--bảng-business-rule)
  - [Bước 3 — 8 câu hỏi discovery](#bước-3--8-câu-hỏi-discovery)
  - [Sai lầm thường gặp khi làm BA view](#sai-lầm-thường-gặp-khi-làm-ba-view)
- [Phần 3 — Mô hình khái niệm](#phần-3--mô-hình-khái-niệm-)
- [Phần 4 — Developer view: V1, V2, V3](#phần-4--developer-view-v1-v2-v3-)
- [Phần 5 — Đối chiếu Medplum thật](#phần-5--đối-chiếu-medplum-thật-)
- [Phần 6 — Bảng so sánh tổng hợp](#phần-6--bảng-so-sánh-tổng-hợp)
- [Phần 7 — Đánh giá & luyện phản biện](#phần-7--đánh-giá--luyện-phản-biện-)
- [Phần 8 — Checklist tái sử dụng](#phần-8--checklist-tái-sử-dụng)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa | Dùng ở đâu |
|---|---|---|
| 💭 mô phỏng | Bối cảnh, câu trả lời của khách, thiết kế V1–V3 đều tự dựng | Phần 1–4 |
| ✅ grounded | Code Medplum thật, chi tiết ở file R | Phần 5, 6 |
| 📘 | Quy định, chuẩn | Rải rác |
| 🧠 suy luận | Đánh giá, khuyến nghị | Phần 7 |

---

## Phần 1 — Bối cảnh giả định 💭

> **Giám đốc vận hành chuỗi phòng khám "An Bình Health"** nói trong buổi kickoff: *"Mình có 12 phòng khám ở Hà Nội và TP.HCM, mỗi phòng là 1 chi nhánh. Bác sĩ phòng nào chỉ xem bệnh nhân phòng đó. Bác sĩ trưởng chuyên môn thì xem được hết để hội chẩn. Năm sau mình ra app cho bệnh nhân xem kết quả xét nghiệm. Và bên bảo hiểm đối tác cần xem hồ sơ để duyệt thanh toán."*

Kịch bản dữ liệu dùng xuyên suốt: bà **Nguyễn Thị Lan** khám ở **An Bình Cầu Giấy**. BS **Trần Văn Minh** làm việc ở Cầu Giấy. BS **Lê Thu Hà** làm việc ở **An Bình Quận 3**.

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

Áp đúng 5 câu hỏi của phương pháp gốc:

1. *Ai nói?* → Giám đốc vận hành.
2. *Ai thao tác hằng ngày?* → Bác sĩ, điều dưỡng, lễ tân, **quản trị hệ thống của từng phòng khám**.
3. *Ai nhận kết quả?* → Bệnh nhân (qua app), bác sĩ trưởng chuyên môn.
4. *Bên thứ 3?* → **Công ty bảo hiểm**. Sau này có thể có app đối tác (đặt lịch, telehealth).
5. *Ai phát hiện nếu sai?* → **Bệnh nhân** (thấy người lạ biết bệnh của mình), **thanh tra Sở Y tế**, bộ phận pháp chế.

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Giám đốc vận hành | Mở chi nhánh nhanh, hội chẩn liên chi nhánh | Người nói duy nhất |
| Bác sĩ | Xem đủ hồ sơ bệnh nhân **mình đang điều trị**, không bị chặn lúc cấp cứu | Thường được hỏi về "quyền", ít khi được hỏi về **lúc khẩn cấp** |
| Lễ tân | Đặt lịch, xem thông tin hành chính, **không** xem chẩn đoán | Hay bị cấp quyền giống bác sĩ "cho tiện" |
| Quản trị chi nhánh | Thêm/bớt nhân viên của chi nhánh mình | Không được nhắc trong câu nói gốc |
| Bệnh nhân | Chỉ xem hồ sơ **của mình** (và con nhỏ, cha mẹ già nếu được uỷ quyền) | "Uỷ quyền gia đình" hiếm khi được nghĩ tới ở buổi đầu |
| Công ty bảo hiểm | Xem **hồ sơ thanh toán** của bệnh nhân **có hợp đồng** với họ | Câu nói gốc nói "xem hồ sơ", rất dễ bị hiểu thành xem **toàn bộ** hồ sơ |
| Pháp chế / thanh tra | Truy vết **ai đã xem** hồ sơ nào | Không ai nhắc, chỉ xuất hiện khi có sự cố |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | Bác sĩ chỉ xem bệnh nhân **phòng khám mình** | Bệnh nhân khám ở 2 chi nhánh thì thuộc chi nhánh nào? |
| 2 | Tường minh | Bác sĩ trưởng chuyên môn xem **tất cả** | "Tất cả" gồm cả ghi chú tâm lý, xét nghiệm HIV không? |
| 3 | Tường minh | Bệnh nhân xem kết quả xét nghiệm qua app | Chỉ kết quả, hay cả ghi chú của bác sĩ? Xem ngay hay sau khi bác sĩ duyệt? |
| 4 | Tường minh | Bảo hiểm xem hồ sơ để duyệt thanh toán | Xem **những loại dữ liệu nào**, của **những bệnh nhân nào**? |
| 5 | Ẩn | Bác sĩ có thể làm việc ở **nhiều** chi nhánh (VD thứ 2–4 Cầu Giấy, thứ 5–6 Quận 3) | 1 người, nhiều nơi làm thì quyền là hợp hay giao? |
| 6 | Ẩn | Cấp cứu: bệnh nhân chi nhánh A ngất ở chi nhánh B | Có cơ chế "mở khẩn cấp" (break-glass) không, ai duyệt sau? |
| 7 | Ẩn | Mọi lần xem hồ sơ phải được ghi lại | Lưu bao lâu? Bệnh nhân có quyền hỏi "ai đã xem tôi" không? |
| 8 | Ẩn | Lễ tân không được sửa chẩn đoán dù được xem lịch hẹn | Phân quyền theo **loại dữ liệu** và theo **hành động** (xem/sửa/xoá) |
| 9 | Ẩn | Mở chi nhánh thứ 13 không cần dev | Ai tạo chi nhánh, ai gán nhân viên? |

**Cách tìm rule ẩn #5:** hỏi *"để rule #1 đúng, phải giả định gì?"* → giả định mỗi bác sĩ chỉ thuộc 1 phòng khám. Hỏi lại khách thì giả định này **sai** ở hầu hết chuỗi phòng khám thật. Rule ẩn #5 lật ngược quan hệ Nhân viên ↔ Phòng khám từ 1-N thành **N-N**.

**Cách tìm rule ẩn #6:** hỏi *"tình huống nào mà tuân thủ rule #1 lại gây hại?"* → cấp cứu. Đây là đặc thù y tế mà ecommerce không có: **chặn đúng quy tắc vẫn có thể là sai về mặt đạo đức và pháp lý**.

### Bước 3 — 8 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh thiết kế |
|---|---|---|
| Q1 | Ranh giới tenant là **pháp nhân** (công ty) hay **chi nhánh**? | 1 hay nhiều "kho dữ liệu" tách cứng |
| Q2 | Bệnh nhân có thể thuộc nhiều chi nhánh cùng lúc không? | Nhãn tenant trên dữ liệu là 1 giá trị hay mảng |
| Q3 | Nhân viên có làm ở nhiều chi nhánh không? | Quyền là 1 vai trò hay danh sách (vai trò, phạm vi) |
| Q4 | Có loại dữ liệu nhạy cảm cần tách riêng không? (tâm thần, HIV, sinh sản) | Phân quyền theo **loại resource / trường** ngoài theo tenant |
| Q5 | Có cần break-glass không? | Cơ chế nâng quyền tạm thời + audit bắt buộc |
| Q6 | Bên ngoài (bảo hiểm, app đối tác) truy cập thế nào? | Phạm vi uỷ quyền (scope) cho ứng dụng, không chỉ cho người |
| Q7 | Bệnh nhân có được uỷ quyền cho người nhà không? | Quan hệ người–người (RelatedPerson) trong phân quyền |
| Q8 | Quy định nào áp dụng? Thời hạn lưu nhật ký truy cập? | Thiết kế audit, không chỉ thiết kế "chặn" |

**Giả định thực tế:** buổi kickoff chỉ hỏi được Q1–Q3. Các trả lời giả định:
- **Q1:** 1 pháp nhân, 12 chi nhánh.
- **Q2:** có, bệnh nhân có thể khám nhiều chi nhánh.
- **Q3:** có, bác sĩ làm nhiều chi nhánh.

Q4–Q8 hỏi dần qua các buổi sau.

### Sai lầm thường gặp khi làm BA view

1. **Chỉ hỏi "ai được xem", quên hỏi "ai được sửa" và "ai đã xem".** Hậu quả: thiết kế chỉ có cờ `can_view`, đến khi thanh tra hỏi nhật ký thì không có.
2. **Coi "bảo hiểm xem hồ sơ" là 1 quyền đơn giản.** Thực tế là: chỉ bệnh nhân có hợp đồng, chỉ dữ liệu liên quan claim, chỉ trong thời hạn hợp đồng. Bỏ qua thì 1 tài khoản bảo hiểm thấy toàn bộ hồ sơ toàn chuỗi.
3. **Không hỏi Q5 (khẩn cấp).** Hệ thống "đúng quy tắc" nhưng bác sĩ không xem được dị ứng thuốc của bệnh nhân đang cấp cứu. Đây là lỗi an toàn người bệnh, không phải lỗi phân quyền.

---

## Phần 3 — Mô hình khái niệm 💭

**Thực thể** (ngôn ngữ nghiệp vụ):

| Thực thể | Vì sao là thực thể thật (định danh, thuộc tính riêng, quan hệ độc lập) |
|---|---|
| Tổ chức (pháp nhân) | Ký hợp đồng, chịu trách nhiệm pháp lý |
| Chi nhánh | Có địa chỉ, giấy phép riêng, mở/đóng độc lập |
| Nhân viên | Có chứng chỉ hành nghề, làm nhiều nơi |
| Vai trò | "Bác sĩ", "Lễ tân", "Bác sĩ trưởng": tập quyền dùng lại cho nhiều người |
| Phân công | **Nhân viên X làm vai trò Y tại chi nhánh Z**. Mang dữ liệu riêng của cặp (từ ngày, đến ngày) → thực thể, không chỉ là quan hệ |
| Bệnh nhân | Hiển nhiên |
| Hồ sơ y tế | Mọi dữ liệu lâm sàng của bệnh nhân |
| Ứng dụng bên ngoài | App bảo hiểm, app đối tác: **không phải người** nhưng cũng cần quyền |
| Nhật ký truy cập | Ai, xem gì, lúc nào, lý do gì |

**Near-miss (không phải thực thể riêng):** "Bác sĩ trưởng" không phải 1 loại nhân viên khác. Đó là 1 **vai trò** gán cho nhân viên. Nhầm chỗ này sẽ dẫn tới bảng `chief_doctors` riêng, lặp lại toàn bộ logic của bảng `doctors`.

**Ma trận quan hệ (hỏi 2 chiều):**

| A | Câu hỏi 2 chiều | B | Kết luận |
|---|---|---|---|
| Tổ chức | 1 tổ chức nhiều chi nhánh? Có. / 1 chi nhánh nhiều tổ chức? Không | Chi nhánh | 1-N |
| Nhân viên | 1 nhân viên nhiều chi nhánh? **Có (Q3)**. / 1 chi nhánh nhiều nhân viên? Có | Chi nhánh | **N-N**, qua Phân công |
| Bệnh nhân | 1 bệnh nhân nhiều chi nhánh? **Có (Q2)**. / 1 chi nhánh nhiều bệnh nhân? Có | Chi nhánh | **N-N** |
| Bệnh nhân | — | Hồ sơ | 1-N. Hồ sơ **thừa hưởng** chi nhánh từ bệnh nhân |
| Vai trò | 1 vai trò nhiều quyền? Có. / 1 quyền thuộc nhiều vai trò? Có | Quyền | N-N |

🧠 Điểm mấu chốt ở tầng khái niệm: **hồ sơ thừa hưởng phạm vi từ bệnh nhân**. Phân quyền không cần gắn từng Observation vào chi nhánh bằng tay. Gắn bệnh nhân vào chi nhánh, rồi mọi thứ "thuộc về" bệnh nhân đi theo.

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — Cột `clinic_id` + vai trò cố định (chỉ có Q1)

```sql
-- 💭
ALTER TABLE patient     ADD COLUMN clinic_id int NOT NULL;
ALTER TABLE observation ADD COLUMN clinic_id int NOT NULL;   -- lặp trên mọi bảng
ALTER TABLE app_user    ADD COLUMN clinic_id int, ADD COLUMN role text; -- 'doctor' | 'receptionist' | 'chief'
```

```ts
// 💭 mỗi API tự nhớ thêm điều kiện
if (user.role !== 'chief') query.where('clinic_id', user.clinic_id);
```

**Dữ liệu:**

| user | clinic_id | role |
|---|---|---|
| BS Minh | 1 (Cầu Giấy) | doctor |
| BS Hà | 2 (Quận 3) | doctor |

**Vỡ ở đâu:**
- **Q3:** BS Minh làm thêm thứ 5 ở Quận 3 thì `clinic_id` chỉ chứa được 1 giá trị.
- **Q2:** bà Lan tái khám ở Quận 3 thì `patient.clinic_id` chỉ chứa được 1 giá trị.
- **Lỗ hổng lớn nhất:** điều kiện `where clinic_id` nằm rải trong từng API. **Quên ở 1 API là lộ dữ liệu toàn chuỗi.**

### V2 — Bảng phân công N-N + bảng gán bệnh nhân (sau khi hỏi Q2, Q3)

```sql
-- 💭
CREATE TABLE staff_assignment (user_id int, clinic_id int, role text, valid_from date, valid_to date);
CREATE TABLE patient_clinic   (patient_id int, clinic_id int);
-- Hồ sơ không mang clinic_id. Quyền = JOIN qua patient_clinic.
```

**Câu kiểm tra quyền:**

```sql
-- 💭 BS Minh có xem được Observation X không?
SELECT 1 FROM observation o
JOIN patient_clinic pc   ON pc.patient_id = o.patient_id
JOIN staff_assignment sa ON sa.clinic_id = pc.clinic_id
WHERE o.id = :x AND sa.user_id = :minh AND now() BETWEEN sa.valid_from AND sa.valid_to;
```

**Tốt hơn V1:** đúng N-N ở cả 2 phía. Hồ sơ thừa hưởng phạm vi từ bệnh nhân.

**Điểm mù (lộ ra khi hỏi Q4, Q6):**
- "Lễ tân xem lịch hẹn nhưng không xem chẩn đoán" thì không có chỗ diễn đạt. `role` chỉ là 1 chuỗi, logic "role nào xem resource nào" lại nằm rải trong code.
- **Bảo hiểm** không phải "nhân viên của chi nhánh". Phạm vi của bảo hiểm là "bệnh nhân **có hợp đồng** với tôi". Đó là 1 **điều kiện trên dữ liệu**, không phải 1 chi nhánh.

### V3 — Chính sách = điều kiện trên dữ liệu (sau khi hỏi Q4, Q6)

Ý tưởng: thay vì code cứng "role → bảng → điều kiện", lưu chính sách thành **dữ liệu**. Mỗi chính sách là danh sách luật `(loại dữ liệu, hành động, điều kiện lọc)`. Điều kiện lọc viết bằng **cùng ngôn ngữ truy vấn** mà hệ thống dùng để search.

```json
// 💭 chính sách "Bác sĩ chi nhánh", có tham số %clinic
{
  "name": "bac-si-chi-nhanh",
  "rules": [
    { "type": "Patient",     "actions": ["read","search","update"], "filter": "Patient?clinic=%clinic" },
    { "type": "Observation", "actions": ["read","search","create"], "filter": "Observation?patient.clinic=%clinic" }
  ]
}
// 💭 chính sách "Lễ tân": chỉ Patient (hành chính) và Appointment, không có Observation/Condition
// 💭 chính sách "Bảo hiểm X": chỉ Claim, Coverage, filter "Claim?insurer=%payer"
```

**Gán:** `staff_assignment` trỏ tới chính sách + giá trị tham số (`%clinic = Cầu Giấy`). 1 người làm 2 chi nhánh thì có 2 dòng gán, quyền là **hợp** của 2 chính sách.

**Vì sao V3 là bước nhảy chất lượng:**
- Thêm vai trò mới (VD "Điều dưỡng trưởng") = thêm dữ liệu, không deploy. Đây là P1 của bài Saleor, áp sang phân quyền.
- **Điều kiện lọc được ghép tự động vào mọi câu truy vấn** ở 1 chỗ duy nhất. Không còn lỗ hổng "quên 1 API" của V1.

**Còn mở:**
- Q5 (khẩn cấp) và Q8 (nhật ký) chưa giải. Đó là 2 tính năng **bên cạnh** chính sách, không nằm trong chính sách.
- Hiệu năng: `Observation?patient.clinic=X` là 1 JOIN qua bệnh nhân **mỗi lần** search.

### 3 lối tắt "nghe hợp lý nhưng sai"

**Lối tắt A — *"Mỗi chi nhánh 1 database riêng, khỏi phải phân quyền."***

Vỡ ngay ở Q2 và yêu cầu hội chẩn: bà Lan có 2 hồ sơ ở 2 DB, bác sĩ trưởng phải gộp tay. Cách này đúng cho **khác pháp nhân** (Q1), sai cho **khác chi nhánh cùng pháp nhân**.

**Lối tắt B — *"Dùng Row-Level Security của Postgres, chính sách nằm trong DB."***

Hấp dẫn vì DB tự chặn. Nhưng chính sách RLS viết bằng SQL, cần quyền DBA để sửa, và không hiểu ngữ nghĩa FHIR (compartment, reference). Vận hành không tự thêm vai trò được. Xem so sánh đầy đủ ở [`1c-comparison`](./1c-comparison-phan-quyen.md).

**Lối tắt C — *"Bảng `user_patient_access(user_id, patient_id)`, gán từng cặp."***

Đúng tuyệt đối, nhưng 200 nhân viên × 50.000 bệnh nhân = tới 10 triệu dòng. Mỗi bệnh nhân mới phải gán cho mọi nhân viên chi nhánh. Nhân viên mới phải gán 5.000 dòng. Bảng này phình nhanh và lệch khỏi thực tế khi nhân sự thay đổi.

---

## Phần 5 — Đối chiếu Medplum thật ✅

Chi tiết và `file:line` ở [`1c-access-control.md`](./1c-access-control.md). Ánh xạ khái niệm:

| Khái niệm Phần 3 | Medplum |
|---|---|
| Tổ chức (ranh giới cứng) | `Project`, cột `projectId` trên mọi bảng |
| Chi nhánh | Resource tuỳ chọn: `Organization`, `HealthcareService` hoặc `CareTeam` (theo docs `packages/docs/docs/access/multi-tenant-access-policy.md`) |
| Phân công (người × vai trò × phạm vi) | `ProjectMembership.access[]`, mỗi phần tử = 1 `AccessPolicy` + `parameter` |
| Vai trò / chính sách | `AccessPolicy` (dữ liệu, không phải code) |
| Điều kiện lọc | `AccessPolicy.resource[].criteria`, **viết bằng search FHIR**, có biến `%organization`, `%patient`, `%profile` |
| Hồ sơ thừa hưởng từ bệnh nhân | `meta.compartment` + `$set-accounts` với `propagate=true`, lan theo Patient CompartmentDefinition |
| Ứng dụng bên ngoài | `ClientApplication` + SMART scopes, **giao** với AccessPolicy |
| Lễ tân không thấy chẩn đoán | Không có luật cho `Condition` trong policy của lễ tân. Mặc định là cấm |
| Ẩn 1 phần dữ liệu | `hiddenFields`, `readonlyFields` |
| Ràng buộc khi ghi | `writeConstraint` (FHIRPath với `%before`, `%after`) |
| Nhật ký | `AuditEvent` (module 1f) |

**Khớp với V3 ở điểm cốt lõi:** chính sách là dữ liệu, điều kiện là câu search, được ghép tự động vào mọi truy vấn (`repo.ts:1932-1999`, đã thấy ở [`1b` mục 5](./1b-search.md#5-bộ-lọc-luôn-có-tenant-xoá-mềm-phân-quyền)).

**Medplum đi xa hơn V3 ở 3 điểm:**
1. **Hiệu năng của "thừa hưởng từ bệnh nhân"**: không JOIN qua bệnh nhân mỗi lần. Nhãn được **chép sẵn** vào `meta.compartment` và cột `compartments UUID[]` của từng resource khi ghi. Search chỉ cần `compartments && ARRAY[...]`.
2. **2 tầng tenant**: `Project` (cứng, khác pháp nhân) + compartment (mềm, khác chi nhánh). Giải đúng cả Q1 lẫn lối tắt A.
3. **Quyền của app = giao (∩) quyền của người và scope của app** (`fhir/smart.ts:196`). App bảo hiểm không bao giờ vượt quá người đã uỷ quyền cho nó.

**Break-glass (Q5): Medplum có hướng dẫn, nhưng không có sẵn tính năng.** 📘 Docs `packages/docs/docs/access/access-policies.md:380-404` (mục "Emergency Access", gắn với tiêu chí ONC d6) mô tả 1 **quy trình** dựng từ các khối có sẵn:
- thêm tạm `Practitioner` vào `Patient.generalPractitioner`, kèm 1 policy `Patient?general-practitioner=%profile`; hoặc
- thêm tạm 1 phần tử `ProjectMembership.access`;
- bắt buộc ghi lý do, người duyệt, thời điểm thu hồi;
- rà `AuditEvent` sau khi hết hạn.

🧠 Không thấy trong code server 1 cơ chế tự hết hạn hay nút "truy cập khẩn cấp". Phần quy trình (duyệt, tự thu hồi) dự án phải tự xây, thường bằng Bot + Task.

## Phần 6 — Bảng so sánh tổng hợp

| Tiêu chí | V1 cột + role | V2 bảng N-N | V3 chính sách = dữ liệu | Medplum thật |
|---|---|---|---|---|
| Nhân viên nhiều chi nhánh (Q3) | ✗ | ✅ | ✅ | ✅ nhiều `access[]` |
| Bệnh nhân nhiều chi nhánh (Q2) | ✗ | ✅ | ✅ | ✅ `meta.compartment` là mảng |
| Phân quyền theo loại dữ liệu (Q4) | ✗ trong code | ✗ trong code | ✅ | ✅ |
| Bên thứ 3 (Q6) | ✗ | ✗ | ◐ | ✅ SMART scopes ∩ policy |
| Quên 1 API → lộ dữ liệu | **Có** | Có | Không | Không |
| Thêm vai trò không deploy | ✗ | ✗ | ✅ | ✅ |
| Hiệu năng lọc theo chi nhánh | Tốt (cột) | JOIN 2 bảng | JOIN mỗi search | Tốt (mảng + GIN) |
| Break-glass (Q5) | ✗ | ✗ | ✗ | ◐ có hướng dẫn quy trình, không có tính năng tự hết hạn |

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Khi nào KHÔNG nên nhảy thẳng lên mức Medplum:**

| Câu trả lời discovery | Mức nên chọn |
|---|---|
| 1 phòng khám, < 10 nhân viên, không app bệnh nhân | V1. Thậm chí chỉ cần vai trò, không cần `clinic_id` |
| Nhiều chi nhánh, nhân viên/bệnh nhân không đi lại giữa các chi nhánh | V1 vẫn được, nhưng phải gom điều kiện `clinic_id` vào **1 lớp dùng chung**, không rải theo API |
| Q2 hoặc Q3 = có | Tối thiểu V2 |
| Q4 hoặc Q6 = có | V3, hoặc dùng nền tảng có sẵn kiểu Medplum |

**Phản đối 1 — Founder:** *"Cứ cho bác sĩ xem hết đi, toàn người nhà cả, phân quyền làm chậm dự án."*

<details><summary>Gợi ý</summary>

Không tranh luận về lòng tin. Hỏi 2 câu: *"Khi bệnh nhân hỏi ai đã xem hồ sơ của họ, mình trả lời bằng gì?"* và *"Bảo hiểm đối tác sẽ dùng tài khoản nào?"*. Câu thứ 2 làm "xem hết" trở thành **lộ toàn bộ hồ sơ chuỗi cho 1 công ty bên ngoài**. Đề xuất bước nhỏ: bắt đầu với 3 chính sách (bác sĩ, lễ tân, bảo hiểm), thêm dần sau.
</details>

**Phản đối 2 — Dev:** *"Policy viết bằng câu search thì khó test. Code `if` rõ ràng hơn."*

<details><summary>Gợi ý</summary>

Đúng là khó test hơn 1 câu `if`. Nhưng code `if` phải **lặp ở mọi API**. Policy dạng dữ liệu chỉ có **1 điểm ghép** (`addAccessPolicyFilters`). Đề xuất: viết test theo dạng "user X search resource Y thì phải thấy N kết quả" cho mỗi policy, chạy trên dữ liệu mẫu (bà Lan, BS Minh, BS Hà).
</details>

**Phản đối 3 — Pháp chế:** *"Bác sĩ Quận 3 cấp cứu cho bà Lan mà hệ thống chặn thì ai chịu trách nhiệm?"*

<details><summary>Gợi ý</summary>

Thừa nhận V3 chưa giải điểm này. Medplum chỉ có hướng dẫn quy trình (Phần 5). Đề xuất break-glass: nút "truy cập khẩn cấp" cấp quyền tạm thời (VD 4 giờ), bắt nhập lý do, tạo `AuditEvent` đặc biệt, thông báo cho trưởng chi nhánh, và duyệt hậu kiểm trong 24 giờ.
</details>

## Phần 8 — Checklist tái sử dụng

Cho bất kỳ bài toán phân quyền nào trong y tế:

- [ ] Đã tách **tenant cứng** (khác pháp nhân, khác hợp đồng dữ liệu) khỏi **phạm vi mềm** (chi nhánh, nhóm chăm sóc)?
- [ ] Đã hỏi "1 người nhiều nơi" và "1 bệnh nhân nhiều nơi"?
- [ ] Dữ liệu lâm sàng **thừa hưởng** phạm vi từ bệnh nhân, không gán tay từng bản ghi?
- [ ] Điều kiện phân quyền được ghép ở **1 chỗ duy nhất**, không rải theo API?
- [ ] Có phân biệt **xem / sửa / xoá** và **theo loại dữ liệu**?
- [ ] Ứng dụng bên thứ 3 bị giới hạn bởi **cả** người uỷ quyền **và** scope của app?
- [ ] Có break-glass và nhật ký truy cập?
- [ ] Trước mỗi lần leo từ V1 → V2 → V3: có bằng chứng (câu trả lời Q2/Q3/Q4/Q6), không phải đoán?
