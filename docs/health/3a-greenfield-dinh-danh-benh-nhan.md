# 3a-G — Thiết kế từ đầu: định danh & tiếp nhận bệnh nhân

> **Loại file:** G (greenfield, theo [`../templates/saleor-greenfield-design-practice-product-attribute.md`](../templates/saleor-greenfield-design-practice-product-attribute.md)).
>
> Trả lời câu: *"Cùng 1 bà cụ đăng ký 3 lần với 3 cách viết tên. Làm sao biết là 1 người, và khi biết rồi thì gộp thế nào mà không làm mất hay lẫn hồ sơ?"*
>
> Đọc tiếp: [`3a-dinh-danh-tiep-nhan.md`](./3a-dinh-danh-tiep-nhan.md) (R, Medplum làm thật thế nào).
>
> **Lưu ý trung thực:** file này viết sau khi đã lướt tên file `patientmatch.ts`. Phần 2–4 cố ý không dùng chi tiết code. Khi tự luyện, che Phần 5 trở đi.

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

| Nhãn | Nghĩa | Dùng ở đâu |
|---|---|---|
| 💭 mô phỏng | Bối cảnh, câu trả lời của khách, thiết kế V1–V3 | Phần 1–4 |
| ✅ grounded | Code Medplum/OpenEMR thật, chi tiết ở file R | Phần 5, 6 |
| 📘 | Chuẩn FHIR, quy định, docs chính thức | Rải rác |
| 🧠 suy luận | Đánh giá, khuyến nghị | Phần 7 |

---

## Phần 1 — Bối cảnh giả định 💭

> **Trưởng phòng hành chính Phòng khám An Bình** nói: *"Tháng trước bà Lan đến khám, lễ tân không tìm thấy hồ sơ nên tạo mới. Bác sĩ Minh kê đơn mà không thấy bà đang dùng Metformin từ đợt trước. Mình muốn: bệnh nhân điền phiếu trên máy tính bảng ở quầy, hệ thống tự nhận ra người cũ."*

3 bản ghi của cùng 1 người, dùng xuyên suốt file:

| # | Ai tạo | Họ tên | Ngày sinh | SĐT | CCCD | Địa chỉ |
|---|---|---|---|---|---|---|
| A | Lễ tân, 2024 | Nguyễn Thị Lan | 1958-03-12 | 0912 345 678 | 001158012345 | 15 Trần Duy Hưng, Cầu Giấy |
| B | Máy tính bảng, 2025 | Nguyen Thi Lan | 1958-03-12 | 0912345678 | (bỏ trống) | 15 Tran Duy Hung |
| C | Kết nối từ bệnh viện tuyến trên (HL7 v2, [2c](./2c-agent-hl7v2.md)) | NGUYEN THI LAN | 1958-03-13 | (không có) | 001158012345 | (không có) |

Và 1 bản ghi **khác người** nhưng rất giống:

| # | Họ tên | Ngày sinh | SĐT | Ghi chú |
|---|---|---|---|---|
| D | Nguyễn Thị Lan | 1985-07-02 | 0912 345 678 | **Con gái** bà Lan, cùng tên mẹ, dùng chung SĐT nhà |

🧠 Ở Việt Nam, trùng họ tên rất phổ biến (họ Nguyễn chiếm tỉ lệ rất lớn dân số), nên **tên gần như không phân biệt được người**. Ngày sinh và giấy tờ mới là tín hiệu mạnh.

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

1. *Ai nói?* → Trưởng phòng hành chính.
2. *Ai thao tác hằng ngày?* → Lễ tân (tìm/tạo hồ sơ), bệnh nhân (tự điền phiếu), **nhân viên quản lý hồ sơ** (gộp trùng).
3. *Ai nhận kết quả?* → Bác sĩ (cần **toàn bộ** tiền sử), bệnh nhân (app xem hồ sơ của mình).
4. *Bên thứ 3?* → Bệnh viện tuyến trên gửi dữ liệu sang, **bảo hiểm** (đối chiếu số thẻ), cổng dữ liệu quốc gia (nếu có).
5. *Ai phát hiện nếu sai?* → Bác sĩ (thấy thuốc lạ trong tiền sử), **bệnh nhân** (thấy kết quả xét nghiệm của người khác trong app).

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Lễ tân | Tìm nhanh, không bắt bệnh nhân khai lại | Được hỏi nhiều nhất, nên yêu cầu "tìm nhanh" lấn át "tìm đúng" |
| Bác sĩ | Đủ tiền sử dị ứng, thuốc đang dùng | Hậu quả của **thiếu ghép** (bỏ sót thuốc) nằm ở bác sĩ, không ở lễ tân |
| Bệnh nhân | Không thấy dữ liệu người khác | Hậu quả của **ghép nhầm** (lộ dữ liệu) nằm ở bệnh nhân, thường chỉ lộ ra khi có app |
| Quản lý hồ sơ | Hàng đợi "nghi trùng" xử lý được trong ngày | Vai trò này thường **chưa tồn tại** cho tới khi số bản trùng lớn |
| Pháp chế | Ai gộp, khi nào, gộp sai thì tách lại được không | Chỉ xuất hiện khi có khiếu nại |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | Nhận ra bệnh nhân cũ khi họ điền phiếu | Nhận ra **ngay lúc điền** hay xử lý sau? |
| 2 | Tường minh | Không bắt bệnh nhân khai lại | Nếu thông tin mới khác thông tin cũ (đổi địa chỉ) thì lấy cái nào? |
| 3 | Ẩn | **Ghép nhầm tệ hơn bỏ sót** khi bệnh nhân xem được hồ sơ (app) | Chấp nhận tỉ lệ bỏ sót bao nhiêu để tỉ lệ ghép nhầm gần 0? |
| 4 | Ẩn | Gộp phải **tách lại được** | Gộp xong 3 tháng mới phát hiện sai thì dữ liệu phát sinh sau đó thuộc về ai? |
| 5 | Ẩn | Dữ liệu từ hệ thống khác không được sửa ở gốc | Bản ghi C thuộc bệnh viện tuyến trên, mình chỉ có bản sao |
| 6 | Ẩn | Cùng tên, cùng SĐT, **khác** ngày sinh rất có thể là người nhà | Có rule "chặn ghép" nào bắt buộc không (khác giới, khác thế hệ)? |
| 7 | Ẩn | Song sinh: cùng ngày sinh, cùng địa chỉ, cùng SĐT | Khi nào bắt buộc tên phải khớp tuyệt đối? |
| 8 | Ẩn | Mọi quyết định "trả hồ sơ cho ai" phải để lại dấu vết | Ai được xem nhật ký này? |

**Cách tìm rule ẩn #3:** hỏi *"sai theo hướng nào đắt hơn?"* Có 2 kiểu sai:
- **Bỏ sót** (2 bản ghi cùng người không được gộp): bác sĩ thiếu tiền sử.
- **Ghép nhầm** (2 người bị gộp làm 1): người này thấy và bị điều trị theo dữ liệu người kia.

Ở y tế, kiểu sai thứ 2 thường đắt hơn, nên ngưỡng tự động phải rất chặt, phần còn lại đẩy cho người duyệt.

**Cách tìm rule ẩn #6:** lấy bản ghi D ở Phần 1. Mọi rule "khớp 3/4 trường" đều ghép A với D.

### Bước 3 — 8 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh thiết kế |
|---|---|---|
| Q1 | Có mã định danh nào **bắt buộc** và **đáng tin** không (CCCD, số thẻ BHYT)? | Có thì ghép theo mã là chính, không thì ghép theo xác suất |
| Q2 | Bao nhiêu % bệnh nhân có mã đó? | Quyết định tỉ trọng của nhánh "không có mã" |
| Q3 | Dữ liệu đến từ bao nhiêu nguồn? Nguồn nào là gốc? | Có cần "bản ghi chính" (master) tách khỏi bản ghi nguồn không |
| Q4 | Ghép lúc nhập (đồng bộ) hay quét định kỳ (bất đồng bộ)? | Độ trễ và chi phí tính toán |
| Q5 | Ai được quyết định gộp? Tự động tới mức nào? | Ngưỡng tự động + hàng đợi người duyệt |
| Q6 | Gộp xong có phải tách lại được không? | Gộp **logic** (liên kết) hay gộp **vật lý** (chuyển dữ liệu, xoá bản cũ) |
| Q7 | Ứng dụng nào dùng kết quả? App bệnh nhân hay báo cáo thống kê? | Ngưỡng khác nhau theo mục đích |
| Q8 | Quy định nào yêu cầu nhật ký truy cập/công bố dữ liệu? | Audit cho từng quyết định ghép |

**Giả định trả lời:**
- **Q1:** có CCCD nhưng chỉ ~60% bệnh nhân khai (💭).
- **Q3:** 2 nguồn (quầy + bệnh viện tuyến trên).
- **Q5:** tự động chỉ khi "chắc chắn", còn lại người duyệt.
- **Q6:** phải tách lại được.

### Sai lầm thường gặp khi làm BA view

1. **Chỉ hỏi lễ tân.** Lễ tân tối ưu "tìm nhanh", không cảm nhận được hậu quả ghép nhầm. Phải hỏi bác sĩ và người phụ trách app bệnh nhân.
2. **Coi CCCD là khoá tuyệt đối.** Thực tế có gõ nhầm 1 số, có người dùng CCCD của người nhà để khám. Mã định danh là tín hiệu **mạnh**, không phải **chân lý**.
3. **Không hỏi Q6.** Đến khi gộp nhầm mới biết thiết kế đã xoá bản gốc.

---

## Phần 3 — Mô hình khái niệm 💭

| Thực thể | Vì sao là thực thể thật |
|---|---|
| **Bản ghi bệnh nhân** (từng nguồn) | Có định danh riêng trong nguồn, có vòng đời riêng (nguồn sửa thì bản sao sửa) |
| **Người** (danh tính thật) | Thứ mình muốn biết. Không bao giờ quan sát trực tiếp, chỉ suy ra từ bản ghi |
| **Mã định danh** | 1 người có nhiều mã (CCCD, BHYT, mã bệnh viện), mỗi mã thuộc 1 **hệ thống cấp mã** |
| **Ứng viên trùng** | Cặp (bản ghi, bản ghi) + điểm + lý do. Có trạng thái: chờ duyệt / đã gộp / đã bác |
| **Danh sách "không phải trùng"** | Quyết định của người duyệt. Phải nhớ, nếu không lần quét sau đề xuất lại |
| **Liên kết gộp** | Cặp (bản ghi nguồn → bản ghi chính), có người tạo, thời điểm. Gỡ được |
| **Phiếu tiếp nhận** | Bản khai thô của bệnh nhân. Là **bằng chứng**, không phải hồ sơ |

**Kiểm tra "thực thể thật?" với "Người":** không có thuộc tính riêng ngoài các bản ghi. Có 2 cách hiện thực:
1. Không lưu, chỉ suy ra bằng liên kết giữa bản ghi.
2. Lưu thành "bản ghi chính" (golden record) riêng.

Đây là ngã rẽ lớn nhất của bài.

**Ma trận quan hệ (hỏi 2 chiều):**

| | Người | Bản ghi | Mã định danh |
|---|---|---|---|
| **Người** | — | 1 người có **nhiều** bản ghi | 1 người có nhiều mã |
| **Bản ghi** | 1 bản ghi thuộc **đúng 1** người (nếu ghép đúng) | gộp vào 1 bản ghi chính | 1 bản ghi khai 0..N mã |
| **Mã định danh** | 1 mã **nên** chỉ thuộc 1 người, nhưng dữ liệu bẩn vi phạm | — | — |

**Cầu nối sang tên kỹ thuật (FHIR):**

| Khái niệm | FHIR | Lớp quyết định |
|---|---|---|
| Bản ghi bệnh nhân | `Patient` | HL7 |
| Mã định danh + hệ thống cấp mã | `Patient.identifier` (`system` + `value`) | HL7 (cấu trúc), quốc gia (URI của CCCD, BHYT) |
| Người (khi muốn tách riêng) | `Person` hoặc `Patient` chính | HL7 cho phép cả 2 |
| Liên kết gộp | `Patient.link` (`replaced-by`, `replaces`, `refer`, `seealso`) | HL7 |
| Bản ghi không còn dùng | `Patient.active = false` | HL7 |
| Ứng viên trùng + điểm | Không có resource riêng. Có thể dùng `RiskAssessment` hoặc `Task` | Ứng dụng tự chọn |
| Phiếu tiếp nhận | `Questionnaire` + `QuestionnaireResponse` | HL7 + SDC IG |
| Tìm ứng viên | Operation `Patient/$match` | HL7 định nghĩa giao diện, **thuật toán do server tự chọn** |

📘 Chuẩn FHIR cố ý **không** quy định thuật toán `$match`. Spec chỉ quy định đầu vào (1 Patient), đầu ra (Bundle có `search.score` và extension `match-grade`: `certain` / `probable` / `possible` / `certainly-not`).

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — "Khoá tự nhiên": ghép khi khớp CCCD, không có thì ghép theo họ tên + ngày sinh

```sql
SELECT id FROM patient
WHERE project_id = $1
  AND (cccd = $2 OR (lower(full_name) = lower($3) AND birth_date = $4));
```

| Cặp | Kết quả V1 | Đúng? |
|---|---|---|
| A–B | "Nguyễn Thị Lan" ≠ "Nguyen Thi Lan" | **Bỏ sót** |
| A–C | Khớp CCCD | Đúng |
| A–D | Khác ngày sinh | Đúng (không ghép) |

**Dừng ở đây khi nào:** chỉ 1 nguồn dữ liệu, bệnh nhân luôn khai CCCD.

### V2 — Chuẩn hoá + chấm điểm

1. **Chuẩn hoá** trước khi so: bỏ dấu, bỏ khoảng trắng, chữ thường, SĐT chỉ giữ số, bỏ mã quốc gia.
2. **Chấm điểm** theo trọng số: CCCD +6, ngày sinh +6, họ tên +6, SĐT +1…
3. Ngưỡng: ≥ 12 → đưa vào hàng đợi duyệt.

| Cặp | Điểm V2 💭 | Kết quả |
|---|---|---|
| A–B | tên 6 + ngày sinh 6 + SĐT 1 = 13 | Đề xuất gộp ✔ |
| A–C | CCCD 6 + tên 6 = 12 (ngày sinh lệch 1 ngày, 0 điểm) | Đề xuất gộp ✔ |
| A–D | tên 6 + SĐT 1 = 7 | Không đề xuất ✔ |

Điểm mù của V2:
- **Điểm cộng dồn che được mâu thuẫn.** 2 người khác nhau có cùng tên + cùng email gia đình + cùng giới = vượt ngưỡng.
- **Không giải thích được** vì sao 12 điểm là "trùng". Khi bị khiếu nại không trả lời được "dựa trên quy tắc nào".
- **Bỏ dấu kiểu ngây thơ** có thể làm hỏng chữ đặc thù. VD chữ **Đ** không có dạng tách dấu trong Unicode (xem Phần 5).

### V3 — Tổ hợp quy tắc + 3 tầng quyết định + gộp bằng liên kết

1. **Thu ứng viên** bằng vài truy vấn rẻ (theo mã, theo SĐT, theo tên + ngày sinh ±1 ngày), không so với cả bảng.
2. **Quyết định theo tổ hợp**, không cộng điểm: danh sách các **tổ hợp trường đủ để kết luận chắc chắn** (VD "tên + họ + ngày sinh + số nhà", "tên + ngày sinh + SĐT"). Chỉ khớp 1 tổ hợp mới được "chắc chắn".
3. **Chặn cứng**: khác hậu tố thế hệ (Jr/Sr), song sinh thì tên phải khớp tuyệt đối.
4. **3 tầng:**
   - "chắc chắn": tự động
   - "có thể": hàng đợi người duyệt
   - "không phải": bỏ
5. **Gộp bằng liên kết**: bản phụ `active = false` + `link → bản chính`. Không xoá. Tách lại = gỡ liên kết.
6. **Ghi audit** mỗi quyết định trả hồ sơ.

**Dừng ở V3 khi nào:** nhiều nguồn, có app bệnh nhân, có yêu cầu pháp lý về công bố dữ liệu.

### 3 lối tắt "nghe hợp lý nhưng sai"

1. **"Đặt unique constraint trên CCCD là xong."** Bản ghi C và A cùng CCCD sẽ làm nguồn tuyến trên **không đẩy được dữ liệu vào**. Dữ liệu bị chặn ở cửa còn tệ hơn trùng.
2. **"Gộp vật lý cho sạch: chuyển hết dữ liệu sang bản chính rồi xoá bản phụ."** Gộp nhầm thì không biết dòng nào vốn của ai. Không tách lại được.
3. **"Dùng machine learning chấm điểm."** Hợp lý ở quy mô lớn, nhưng mất khả năng giải thích. Ở bối cảnh pháp lý, "mô hình bảo vậy" không phải câu trả lời cho thanh tra.

### Data flow

```
Máy tính bảng ── QuestionnaireResponse ──▶ [Bot tiếp nhận]
                                              │ 1. $extract → Patient, Coverage, AllergyIntolerance…
                                              │ 2. $match(Patient mới)
                                              ▼
                            certain ── 1 kết quả ──▶ gắn vào bản cũ
                            probable/possible ─────▶ Task "duyệt trùng" cho quản lý hồ sơ
                            không có ──────────────▶ tạo Patient mới
```

---

## Phần 5 — Đối chiếu Medplum thật ✅

Chi tiết ở [`3a-dinh-danh-tiep-nhan.md`](./3a-dinh-danh-tiep-nhan.md). Tóm tắt:

| Thiết kế | Medplum | ✅ |
|---|---|---|
| V3 bước 1 (thu ứng viên rẻ) | Đúng 3 chiến lược: identifier, telecom, tên + ngày sinh ±1 ngày. Tối đa 100/lượt search. **Không chuẩn hoá**: SĐT so nguyên văn, tên không bỏ dấu, nên với dữ liệu Phần 1, A **không lọt vào** ứng viên của B | `packages/server/src/fhir/operations/patientmatch.ts:140-200` |
| V3 bước 2 (tổ hợp) | Bảng 2 của **CMS Patient Matching framework v3.4.0** (Mỹ), quy tắc 01–22, 29, 30 | `utils/cms-patient-match.ts:95-152` |
| V3 bước 3 (chặn cứng) | Hậu tố thế hệ khác nhau → `certainly-not`. Có > 1 ứng viên trùng ngày sinh → tên phải khớp tuyệt đối | `patientmatch.ts:111-117`, `:219-221` |
| V3 bước 4 (3 tầng) | `certain` (khớp 1 tổ hợp) / `probable` ≥ 0.65 / `possible` ≥ 0.2 | `patientmatch.ts:42-43`, `:215-235` |
| V3 bước 5 (gộp bằng liên kết) | **Không có trong server**. Bot mẫu dùng `Patient.link` `replaces`/`replaced-by` + `active=false` | `examples/medplum-demo-bots/src/deduplication/merge-matching-patients.ts:123-132` |
| V3 bước 6 (audit) | Khi gọi `onlyCertainMatches`: ghi AuditEvent `released` / `no-match` / `ambiguous` / `gather-truncated` | `patientmatch.ts:272-308` |
| V2 điểm mù "bỏ dấu" | `foldString` dùng NFD + bỏ dấu. **"Đặng" → "ang"**, chữ Đ bị xoá chứ không thành D | `utils/cms-patient-match.ts:260-262` + kiểm thử trong file R |
| Data flow (Bot tiếp nhận) | Bot mẫu **luôn tạo Patient mới**, không gọi `$match` | `examples/medplum-patient-intake-demo/src/bots/core/intake-form.ts:131` |

OpenEMR (commit `ebe3ee8`) dừng ở **V2**: điểm cộng dồn, gộp vật lý.

| | OpenEMR | ✅ |
|---|---|---|
| Chấm điểm | email 6 + DOB 6 + họ tên 6 + giới 6 + soundex họ 2 + SĐT 1, ngưỡng 12 | `src/Services/Patient/DuplicatePatientService.php:93-112`, `:50` |
| Lưu điểm | Cột `patient_data.dupscore` (`-9` chờ tính, `-1` đã xác nhận không trùng) | `sql/database.sql:8468`, `DuplicatePatientService.php:44-47` |
| Gộp | Chuyển mọi dòng sang bệnh nhân đích rồi **xoá dòng định danh** của bản nguồn | `src/Services/Patient/PatientMergeService.php:1-20` (docblock), `:298` |

---

## Phần 6 — Bảng so sánh tổng hợp

| Tiêu chí | V1 💭 | V2 💭 / OpenEMR ✅ | V3 💭 / Medplum ✅ |
|---|---|---|---|
| Cách kết luận | Khoá tự nhiên | Cộng điểm, 1 ngưỡng | Tổ hợp quy tắc + 3 tầng |
| Giải thích được | Có | Kém ("12 điểm") | Có ("quy tắc 02: tên + họ + DOB + SĐT") |
| Chặn A–D (mẹ/con) | Có (tình cờ) | Có ở ví dụ này, nhưng dễ vỡ khi thêm email chung | Có: không tổ hợp nào chỉ gồm tên + SĐT, quy tắc 29 cần ZIP và tự tắt khi 2 DOB đều có mà khác nhau |
| Gộp tách lại được | — | Không (xoá bản nguồn) | Có (liên kết), nhưng **phải tự viết Bot** |
| Tiếng Việt | Kém | Kém (soundex là thuật toán cho tiếng Anh) | Tốt với dấu thường, **lỗi với Đ** |

---

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Vì sao có khoảng cách V2 → V3:** V2 trả lời "giống bao nhiêu", V3 trả lời "**đủ bằng chứng chưa**". Câu hỏi thứ 2 mới là câu pháp lý hỏi. Cơ quan quản lý Mỹ (CMS) chuẩn hoá đúng câu thứ 2 thành 1 bảng tổ hợp. Medplum chỉ việc hiện thực bảng đó.

**Khi nào KHÔNG nên nhảy thẳng lên V3:**
- Phòng khám 1 chi nhánh, nhập liệu tập trung, có CCCD gần 100%: V1 + 1 màn hình "nghi trùng" là đủ.
- Bảng tổ hợp CMS dùng định danh Mỹ (SSN 4 số cuối, MBI). Ở Việt Nam phải tự dựng bảng tổ hợp với CCCD, số thẻ BHYT, SĐT. **Không bê nguyên bảng.**

### Luyện phản biện

**Phản đối 1 (kinh doanh):** "Cứ để lễ tân tìm tay, tự động hoá tốn công."

<details><summary>Đáp án gợi ý</summary>

Chi phí của bỏ sót không nằm ở lễ tân mà ở bác sĩ (thiếu tiền sử) và ở **doanh thu**: hồ sơ trùng làm hỏng đối soát bảo hiểm (cùng thẻ BHYT, 2 hồ sơ, 2 lượt khám trong ngày bị từ chối thanh toán). Hỏi tỉ lệ trùng hiện tại: tài liệu Medplum nêu mức từ dưới 1% tới 10–50% tuỳ hệ thống (📘 `packages/docs/docs/fhir-datastore/patient-deduplication/architecture-overview.md`). Ở mức vài %, hàng đợi duyệt thủ công đã đủ. Tự động hoá chỉ đáng khi tỉ lệ cao hoặc có nhiều nguồn.
</details>

**Phản đối 2 (kỹ thuật):** "Ghép xác suất thì dùng thư viện ML có sẵn, viết bảng quy tắc làm gì."

<details><summary>Đáp án gợi ý</summary>

Bảng quy tắc là **hợp đồng kiểm toán được**. Mỗi lần trả hồ sơ, AuditEvent ghi đúng mã quy tắc đã thoả (`criteria` trong `patientmatch.ts:285-293`). ML có thể dùng ở tầng `probable`/`possible` để **xếp thứ tự hàng đợi duyệt**, không dùng để tự động trả hồ sơ.
</details>

**Phản đối 3 (pháp lý):** "Gộp bằng liên kết thì dữ liệu vẫn còn 2 nơi, có vi phạm nguyên tắc 1 hồ sơ duy nhất không?"

<details><summary>Đáp án gợi ý</summary>

Không. Bản phụ `active = false` và trỏ `replaced-by` bản chính. Ứng dụng chỉ hiển thị bản chính. Còn bản phụ là **bằng chứng** của việc gộp, cần để tách lại và để giải trình. Gộp vật lý mới là rủi ro pháp lý: không chứng minh được dữ liệu nào vốn thuộc ai.
</details>

---

## Phần 8 — Checklist tái sử dụng

- [ ] Đã hỏi "sai theo hướng nào đắt hơn": bỏ sót hay ghép nhầm?
- [ ] Đã có bản ghi kiểu "D" (người nhà cùng tên, cùng SĐT) trong bộ test
- [ ] Mã định danh có `system` (hệ thống cấp mã), không chỉ `value`
- [ ] Chuẩn hoá tiếng Việt đã test **Đ/đ**
- [ ] Có 3 tầng: tự động / người duyệt / bỏ
- [ ] Gộp bằng liên kết, có thao tác tách lại
- [ ] Danh sách "không phải trùng" được lưu để không đề xuất lại
- [ ] Mỗi quyết định trả hồ sơ có audit kèm quy tắc đã thoả

**Tự luyện tiếp theo:** dựng bảng tổ hợp cho Việt Nam với 6 trường (họ tên, ngày sinh, giới, CCCD, số thẻ BHYT, SĐT). Với mỗi tổ hợp, tìm 1 cặp người thật **khác nhau** vẫn thoả. Tổ hợp nào không tìm được phản ví dụ thì mới đủ "chắc chắn".

**Nguồn:**
- 📘 FHIR R4 `Patient/$match`: https://hl7.org/fhir/R4/patient-operation-match.html
- 📘 Docs Medplum: `packages/docs/docs/fhir-datastore/patient-deduplication/`
- ✅ File R: [`3a-dinh-danh-tiep-nhan.md`](./3a-dinh-danh-tiep-nhan.md)
