# 3b-G — Thiết kế từ đầu: đặt lịch khám

> **Loại file:** G (greenfield). Tương đương bài Warehouse/Stock bỏ dở bên Saleor: cùng là bài "tài nguyên có hạn, nhiều người tranh nhau".
>
> Trả lời câu: *"2 bệnh nhân bấm đặt cùng 1 khung giờ cùng lúc, chỉ 1 người được giữ chỗ. Và trước đó: làm sao biết khung giờ nào còn trống?"*
>
> Đọc tiếp: [`3b-dat-lich.md`](./3b-dat-lich.md) (R) · [`3b-comparison-chong-dat-trung.md`](./3b-comparison-chong-dat-trung.md) (C).
>
> **Lưu ý trung thực:** viết sau khi đã lướt danh sách file `find.ts`, `hold.ts`, `book.ts`. Khi tự luyện, che Phần 5 trở đi.

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
| 💭 mô phỏng | Bối cảnh, câu trả lời, V1–V3 | Phần 1–4 |
| ✅ grounded | Code thật, chi tiết ở file R | Phần 5, 6 |
| 📘 | Chuẩn FHIR, docs chính thức | Rải rác |
| 🧠 suy luận | Đánh giá | Phần 7 |

---

## Phần 1 — Bối cảnh giả định 💭

> **Giám đốc Phòng khám An Bình** nói: *"Mình muốn bệnh nhân tự đặt lịch trên app như đặt vé máy bay. BS Minh khám nội tiết thứ 2, 4, 6 từ 8h đến 12h, mỗi lượt 20 phút. Siêu âm thì cần cả bác sĩ lẫn phòng siêu âm. Có lớp tư vấn dinh dưỡng nhóm 6 người. Bệnh nhân tiểu đường tái khám 4 tuần/lần."*

Kịch bản xuyên suốt:
- **Thứ Hai 9:00–9:20**, lịch của BS Minh.
- Bà **Lan** (app) và ông **Hùng** (gọi điện, lễ tân đặt hộ) cùng bấm "Đặt" lúc 7:59:59.
- Giữa 2 lượt khám có **5 phút** dọn phòng, lau máy đo.

---

## Phần 2 — BA view

### Bước 1 — Stakeholder map

1. *Ai nói?* → Giám đốc.
2. *Ai thao tác hằng ngày?* → Bệnh nhân (tự đặt), lễ tân (đặt hộ, đổi lịch), **điều phối viên** (khai báo lịch làm việc bác sĩ, phòng).
3. *Ai nhận kết quả?* → Bác sĩ (danh sách bệnh nhân trong ngày), bệnh nhân (tin nhắn nhắc).
4. *Bên thứ 3?* → Cổng đặt lịch đối tác, bảo hiểm (có bảo hiểm yêu cầu duyệt trước 1 số dịch vụ).
5. *Ai phát hiện nếu sai?* → **2 bệnh nhân cùng có mặt lúc 9:00** (đặt trùng), hoặc bác sĩ ngồi không (lịch trống giả).

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Bệnh nhân | Thấy giờ trống **đúng**, bấm là chắc chắn | "Đúng" ở đây gồm cả múi giờ, đổi giờ mùa hè nếu ở Mỹ |
| Lễ tân | Đôi khi **cố ý** chen thêm 1 ca cấp cứu | Yêu cầu "không trùng" của giám đốc mâu thuẫn với nhu cầu này |
| Điều phối viên | Khai báo lịch 1 lần, không phải tạo từng ô 20 phút | Người này quyết định mô hình dữ liệu nhưng ít khi được mời họp |
| Bác sĩ | Có thời gian dọn phòng, ghi hồ sơ giữa 2 ca | Buffer không được nói ra vì "ai cũng biết" |
| Bảo hiểm | Dịch vụ cần duyệt trước | Dẫn tới nhu cầu **giữ chỗ tạm** chờ duyệt |

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | BS Minh khám T2/4/6, 8h–12h, 20 phút/lượt | Nghỉ lễ, nghỉ phép khai báo thế nào? |
| 2 | Tường minh | Siêu âm cần bác sĩ **và** phòng | Nếu chỉ còn 1 trong 2 thì sao? Đặt được nửa? |
| 3 | Tường minh | Lớp dinh dưỡng 6 người/lượt | "Sức chứa" là của **dịch vụ** hay của **phòng**? |
| 4 | Tường minh | Tái khám 4 tuần/lần | Đặt cả chuỗi 1 lần? Nếu tuần thứ 3 bận thì sao? |
| 5 | Ẩn | **Không được đặt trùng** 1 khung giờ có sức chứa 1 | Kể cả khi 2 người bấm cách nhau vài mili-giây |
| 6 | Ẩn | Có 5 phút đệm giữa 2 ca | Đệm có được chồng lên đệm của ca khác không? |
| 7 | Ẩn | Giờ bắt đầu phải "tròn" (8:00, 8:20, 8:40), không phải 8:07 | Lưới thời gian đặt theo gì? |
| 8 | Ẩn | Bệnh nhân đang điền thông tin bảo hiểm thì chỗ phải được **giữ** vài phút | Giữ bao lâu? Hết hạn thì ai nhả? |
| 9 | Ẩn | Huỷ lịch phải còn dấu vết (thống kê no-show, tranh chấp) | Huỷ là xoá hay đổi trạng thái? |

**Cách tìm rule ẩn #5:** hỏi *"nếu 2 người cùng làm việc này cùng lúc thì sao?"* Giám đốc nói "chỉ 1 người được", nhưng chưa ai nghĩ tới **khoảng thời gian giữa lúc kiểm tra còn trống và lúc ghi**. Đây là lỗi kinh điển check-then-act.

**Cách tìm rule ẩn #8:** đi theo đúng 1 lượt đặt trên app: chọn giờ → nhập bảo hiểm → xác nhận. Giữa bước 1 và 3 có 2–3 phút. Không giữ chỗ thì bấm xác nhận mới biết đã mất chỗ.

### Bước 3 — 8 câu hỏi discovery

| # | Câu hỏi | Điểm rẽ nhánh thiết kế |
|---|---|---|
| Q1 | Lịch làm việc là **quy tắc lặp** hay **danh sách ô cụ thể**? | Sinh sẵn ô hay tính lúc tìm |
| Q2 | Có cho phép đặt chồng (overbook) không? Ai được? | Sức chứa và quyền vượt sức chứa |
| Q3 | Dịch vụ nào cần nhiều tài nguyên cùng lúc? | Đặt nguyên tử trên nhiều lịch |
| Q4 | Có buffer không? Ai cấu hình? | Buffer là 1 phần lịch hay 1 phần dịch vụ |
| Q5 | Có cần giữ chỗ tạm không? Bao lâu? | Trạng thái trung gian + cơ chế hết hạn |
| Q6 | Lịch lặp: đặt cả chuỗi "tất cả hoặc không" hay từng buổi? | Giao dịch nhiều lượt |
| Q7 | Bệnh viện có nhiều múi giờ / đổi giờ mùa hè không? | Lưu giờ địa phương hay UTC |
| Q8 | Đỉnh tải bao nhiêu? (mở đặt lịch tiêm vắc-xin lúc 8:00) | Chọn cơ chế chống trùng |

**Giả định trả lời:** Q1 quy tắc lặp · Q2 không overbook, trừ lớp nhóm · Q3 siêu âm cần 2 tài nguyên · Q5 giữ 10 phút · Q6 tất cả hoặc không · Q7 Việt Nam 1 múi giờ, không đổi giờ · Q8 ~50 lượt/phút lúc cao điểm.

### Sai lầm thường gặp khi làm BA view

1. **Không hỏi điều phối viên.** Thiết kế "sinh sẵn ô 20 phút cho 6 tháng" đúng về logic nhưng mỗi lần bác sĩ đổi giờ làm là phải xoá/sinh lại hàng nghìn ô.
2. **Coi "không trùng" là hiển nhiên** nên không đưa vào tiêu chí nghiệm thu. Lỗi chỉ lộ ra ở tải thật.
3. **Không hỏi Q5 "ai nhả chỗ".** Có giữ chỗ mà không có hết hạn thì lịch đầy chỗ "ma".

---

## Phần 3 — Mô hình khái niệm 💭

| Thực thể | Vì sao là thực thể thật |
|---|---|
| **Tài nguyên** (bác sĩ, phòng, máy) | Có lịch riêng, có sức chứa |
| **Lịch làm việc** của 1 tài nguyên | Quy tắc lặp + khoảng hiệu lực + múi giờ |
| **Loại dịch vụ** (khám nội tiết 20', siêu âm 30', lớp nhóm 60') | Quyết định thời lượng, buffer, sức chứa |
| **Ô thời gian bị chiếm** | Bằng chứng 1 khoảng thời gian đã bị dùng (bởi lịch hẹn, buffer, nghỉ) |
| **Lịch hẹn** | Ai, dịch vụ gì, lúc nào, trạng thái (giữ tạm / đã đặt / huỷ / đến / không đến) |

**Kiểm tra "ô trống có phải thực thể thật?"** Ô trống không có ai tạo, không có thuộc tính riêng, chỉ là **phần bù** của (lịch làm việc − ô bị chiếm). Nếu lưu nó thành dòng thì phải đồng bộ mỗi khi quy tắc đổi. → Ô trống nên được **tính**, ô bị chiếm mới cần **lưu**.

**Ma trận quan hệ:**

| | Tài nguyên | Lịch hẹn | Ô bị chiếm |
|---|---|---|---|
| **Lịch hẹn** | 1 lịch hẹn cần **1..N** tài nguyên (siêu âm = 2) | — | 1 lịch hẹn chiếm **1..N** ô (mỗi tài nguyên 1 ô + buffer) |
| **Ô bị chiếm** | Thuộc đúng 1 lịch (1 tài nguyên) | Thuộc 0..1 lịch hẹn (ô nghỉ phép không thuộc lịch hẹn nào) | — |

**Cầu nối sang FHIR:**

| Khái niệm | FHIR R4 | Lớp quyết định |
|---|---|---|
| Lịch làm việc của 1 tài nguyên | `Schedule` (`actor`, `planningHorizon`) | HL7 |
| Ô thời gian | `Slot` (`status`: `free` / `busy` / `busy-tentative` / `busy-unavailable`) | HL7 |
| Lịch hẹn | `Appointment` (`status`: `proposed` / `pending` / `booked` / `cancelled` / `noshow`…) | HL7 |
| Loại dịch vụ | `HealthcareService` + `Appointment.serviceType` | HL7 |
| Quy tắc lặp (giờ làm việc), thời lượng, buffer, sức chứa | **Không có trong R4** (R5 có `Availability`) → extension | **Medplum** |
| Tìm/giữ/đặt/xác nhận/huỷ | `$find`, `$hold`, `$book`, `$confirm`, `$cancel` | HL7 có IG Scheduling, **cách hiện thực là Medplum** |

📘 Lưu ý K1: FHIR R4 **không** nói Slot phải sinh sẵn. Spec chỉ nói `Slot` là "khoảng thời gian có thể đặt". Rất nhiều hệ thống FHIR sinh sẵn `Slot free` cho cả năm. Đây là quyết định của từng hệ thống.

---

## Phần 4 — Developer view: V1, V2, V3 💭

### V1 — 1 bảng lịch hẹn, kiểm tra rồi ghi

```sql
-- Bước 1: còn trống không?
SELECT count(*) FROM appointment
WHERE doctor_id = 'minh' AND status = 'booked'
  AND start_at < '2026-10-19 09:20' AND end_at > '2026-10-19 09:00';
-- Bước 2: nếu 0 thì ghi
INSERT INTO appointment (...) VALUES (...);
```

| Thời điểm | Request Lan | Request Hùng |
|---|---|---|
| t=0 ms | SELECT → 0 | |
| t=3 ms | | SELECT → 0 |
| t=8 ms | INSERT ✔ | |
| t=10 ms | | INSERT ✔ → **đặt trùng** |

**Dừng ở V1 khi nào:** chỉ lễ tân đặt, 1 người 1 máy, không có app. Đây chính là thực tế của OpenEMR (Phần 5).

### V2 — Sinh sẵn ô trống, "chiếm ô" bằng 1 câu UPDATE nguyên tử

```sql
-- Sinh sẵn mỗi đêm cho 90 ngày tới
INSERT INTO slot (schedule_id, start_at, end_at, status) VALUES ('minh', '09:00', '09:20', 'free'), ...;

-- Đặt: chỉ thành công nếu ô vẫn free
UPDATE slot SET status = 'busy', appointment_id = $1
WHERE id = $2 AND status = 'free';
-- rowcount = 0 → đã mất chỗ
```

Chống trùng tốt (Postgres khoá dòng khi UPDATE). Điểm mù:
- BS Minh đổi từ 20 phút sang 15 phút → sinh lại toàn bộ ô, lịch hẹn cũ lệch lưới.
- **Buffer 5 phút** không phải 1 ô: phải chiếm ô kế bên, nhưng ô kế bên là 9:20–9:40, chiếm cả ô thì mất 1 lượt khám.
- **Siêu âm 2 tài nguyên:** 2 câu UPDATE, cái đầu thành công cái sau thất bại → phải rollback. Dễ, nhưng phải nhớ bọc giao dịch.
- **Lớp nhóm 6 người:** ô cần `capacity` và `booked_count`. UPDATE thành `booked_count < capacity`.
- Số ô: 1 bác sĩ × 12 ô/buổi × 3 buổi/tuần × 52 tuần ≈ 1.900 ô/năm. 200 tài nguyên ≈ 380.000 dòng phần lớn **không bao giờ dùng**.

### V3 — Tính ô trống lúc tìm, chỉ lưu ô bị chiếm, kiểm tra lại trong giao dịch tuần tự hoá

1. **Quy tắc** (giờ làm việc, thời lượng, buffer, lưới, sức chứa) lưu 1 lần trên lịch/dịch vụ.
2. **Tìm** = (quy tắc trong khoảng) − (ô bị chiếm trong khoảng) → danh sách giờ trống ảo. Không ghi gì.
3. **Đặt** = trong 1 giao dịch:
   1. đọc lại ô bị chiếm trong khoảng
   2. kiểm tra còn chỗ
   3. ghi `Slot busy` + `Slot busy-unavailable` (buffer) + `Appointment`

   Isolation **SERIALIZABLE**: 2 giao dịch cùng "đọc thấy trống rồi ghi" thì CSDL buộc 1 cái thất bại, thử lại sẽ thấy hết chỗ.
4. **Giữ chỗ** = như đặt nhưng `Slot busy-tentative` + `Appointment pending`. **Xác nhận** = đổi trạng thái.
5. **Huỷ** = `Appointment cancelled` (giữ lại), xoá các `Slot` (nhả thời gian).
6. **Chuỗi lặp** = N lịch hẹn trong **1** giao dịch, tất cả hoặc không.

**Dừng ở V3 khi nào:** có app tự đặt, nhiều tài nguyên, lịch thay đổi thường xuyên.

### Điểm mù còn lại của V3

- **Hết hạn giữ chỗ**: V3 bước 4 chưa nói ai nhả `busy-tentative` khi bệnh nhân bỏ dở.
- **Lỗi tuần tự hoá ở tải cao**: 50 người tranh 1 ô thì 49 giao dịch bị huỷ và thử lại. Ổn với 50/phút, không ổn với 5.000/giây.

### 3 lối tắt "nghe hợp lý nhưng sai"

1. **"Đặt UNIQUE (doctor_id, start_at)."** Chặn được 2 lượt cùng 9:00, nhưng **không** chặn lượt 9:10–9:30 chồng lên 9:00–9:20. Trùng giờ là bài toán **khoảng chồng nhau**, không phải bằng nhau.
2. **"Khoá ở tầng ứng dụng bằng biến trong RAM."** Chạy 2 server là hỏng.
3. **"Kiểm tra trùng ở frontend, nút Đặt bị mờ nếu hết chỗ."** Frontend chỉ thấy dữ liệu lúc tải trang.

### Data flow

```
App bệnh nhân                  Server                                    DB
  │ $find(BS Minh, T2, nội tiết) │                                          │
  │ ─────────────────────────────▶ quy tắc − Slot bận → giờ trống ảo        │ (chỉ đọc)
  │ ◀── [9:00, 9:20, 9:40…] ─────│                                          │
  │ $hold(9:00)                  │ BEGIN SERIALIZABLE                       │
  │ ─────────────────────────────▶ đọc lại Slot bận → còn chỗ → ghi        │
  │                              │ Slot busy-tentative + Appointment pending│
  │                              │ COMMIT (hoặc 40001 → thử lại → 400)      │
  │ (nhập bảo hiểm 2 phút)       │                                          │
  │ $confirm                     │ busy-tentative → busy, pending → booked  │
```

---

## Phần 5 — Đối chiếu Medplum thật ✅

Chi tiết ở [`3b-dat-lich.md`](./3b-dat-lich.md).

| Thiết kế V3 | Medplum | ✅ (`packages/server/src/fhir/operations/`) |
|---|---|---|
| Quy tắc lưu 1 lần | Extension `SchedulingParameters` trên `Schedule`/`HealthcareService`: availability, duration, bufferBefore/After, alignmentInterval/Offset, slotCapacity, timezone | `utils/scheduling-parameters.ts:73-117` |
| Tìm = quy tắc − ô bận | `resolveAvailability` + `applyExistingSlots` (cộng thêm `Slot free` nếu có, trừ khoảng vượt sức chứa) | `utils/scheduling.ts:191-213`, `:458-480` |
| Đặt trong giao dịch SERIALIZABLE, kiểm tra lại | `createProposedAppointments`: validate trước, rồi trong `withTransaction({serializable: true})` gọi lại `validateAllAvailability` rồi mới ghi | `utils/scheduling.ts:1062-1115` |
| Thử lại khi xung đột | Mặc định 3 lần, backoff 50 ms × 2^n ± 25% | `../repository/repository-connection.ts:26-27`, `:516-583` |
| Buffer | Slot `busy-unavailable`, luôn tính sức chứa 1 | `utils/scheduling.ts:516-566`, `:867` |
| Giữ chỗ / xác nhận | `$hold`: `pending` + `busy-tentative`. `$confirm`: → `booked` + `busy` | `hold.ts:44-58`, `confirm.ts:83-91` |
| Huỷ | `cancelled` + **xoá** Slot | `cancel.ts:56-62` |
| Chuỗi lặp tất cả hoặc không | 2–6 lượt hằng tuần, 1 giao dịch | `utils/recurrence.ts:22-23` |
| **Hết hạn giữ chỗ** | **Không có.** Docs: "`$hold` has no built-in TTL" | 📘 `packages/docs/docs/decision-guides/scheduling.md` mục 3.8 |

OpenEMR (commit `ebe3ee8`): **dừng ở V1, và bỏ cả bước kiểm tra**.
- Giờ làm việc là sự kiện "In Office"/"Out of Office" trong **cùng bảng** `openemr_postcalendar_events` (`sql/database.sql:8275`), phân biệt bằng `pc_catid` 2/3.
- Màn hình tìm giờ trống tính bằng mảng bit (`interface/main/calendar/find_appt_popup.php:65-90`).
- Hàm lưu lịch hẹn chỉ kiểm tra định dạng trường, không kiểm tra chồng giờ (`src/Services/AppointmentService.php:90-120`, `:311+`).

---

## Phần 6 — Bảng so sánh tổng hợp

| Tiêu chí | V1 / OpenEMR | V2 (sinh sẵn ô) | V3 / Medplum |
|---|---|---|---|
| Chống đặt trùng | Không | Có (khoá dòng) | Có (SERIALIZABLE + kiểm tra lại) |
| Đổi giờ làm việc | Sửa sự kiện "In Office" | Sinh lại ô | Sửa 1 extension |
| Buffer | Không có khái niệm | Khó | Slot `busy-unavailable` |
| Nhiều tài nguyên | Không | Bọc giao dịch thủ công | Nhiều `Slot` trên nhiều `Schedule` trong 1 Appointment |
| Sức chứa > 1 | Không | Cột đếm | `slotCapacity`, quét sự kiện vào/ra |
| Giữ chỗ hết hạn | — | Tự viết | **Tự viết** |
| Chen ca cố ý (lễ tân) | Dễ (không chặn gì) | Khó | Khó: phải ghi Slot trực tiếp, không qua `$book` |

---

## Phần 7 — Đánh giá & luyện phản biện 🧠

**Vì sao có khoảng cách V2 → V3:** V2 coi **ô trống** là dữ liệu, V3 coi ô trống là **kết quả tính**. Đổi góc nhìn này làm biến mất cả 1 lớp đồng bộ (sinh lại ô khi đổi lịch), đổi lại phải trả bằng 1 thứ khó hơn: chống trùng khi không có dòng nào để khoá. SERIALIZABLE là cách "khoá cái chưa tồn tại".

**Khi nào KHÔNG nên nhảy thẳng lên V3:**
- Phòng khám nhỏ, lễ tân đặt là chính: V1 + màn hình cảnh báo trùng là đủ, và giữ được khả năng chen ca.
- Lịch rất cố định, ô ít (phòng mổ 4 ca/ngày): V2 đơn giản, dễ hiểu, dễ báo cáo.
- Đỉnh tải cực cao (mở đặt vắc-xin toàn thành phố): SERIALIZABLE trên 1 ô nóng sẽ huỷ và thử lại liên tục. Cần hàng đợi phía trước.

### Luyện phản biện

**Phản đối 1 (vận hành):** "Lễ tân cần chen ca cấp cứu, hệ thống chặn cứng thì họ quay về sổ giấy."

<details><summary>Đáp án gợi ý</summary>

Tách 2 đường: bệnh nhân tự đặt đi qua `$book` (chặn cứng). Lễ tân có quyền ghi `Appointment` trực tiếp (không qua `$book`), ghi rõ lý do. Phân quyền bằng AccessPolicy ([1c](./1c-access-control.md)): bệnh nhân chỉ được gọi operation, không được tạo `Slot`. Báo cáo cuối ngày liệt kê ca chen để quản lý duyệt.
</details>

**Phản đối 2 (kỹ thuật):** "SERIALIZABLE chậm, dùng `SELECT … FOR UPDATE` trên dòng Schedule cho nhanh."

<details><summary>Đáp án gợi ý</summary>

Khoá dòng Schedule tuần tự hoá **mọi** lượt đặt của bác sĩ đó, kể cả 2 lượt ở 2 ngày khác nhau không xung đột. SERIALIZABLE chỉ huỷ khi 2 giao dịch thật sự đọc–ghi chồng phạm vi. Với 50 lượt/phút trải trên nhiều bác sĩ, tỉ lệ xung đột thấp, SERIALIZABLE gần như không tốn gì. Ngược lại, 1 ô "nóng" thì khoá dòng lại tốt hơn (xếp hàng thay vì huỷ–thử lại). Chi tiết ở [C](./3b-comparison-chong-dat-trung.md).
</details>

**Phản đối 3 (kinh doanh):** "Giữ chỗ 10 phút làm giảm số lượt đặt vì chỗ bị khoá bởi người bỏ dở."

<details><summary>Đáp án gợi ý</summary>

Đo 2 con số: tỉ lệ bỏ dở sau khi giữ chỗ, và tỉ lệ "bấm xác nhận thì báo hết chỗ" nếu không giữ. Cái sau gây mất khách nặng hơn (trải nghiệm tệ ở bước cuối). Giảm hại của cái trước bằng TTL ngắn (3–5 phút) và nhả chỗ ngay khi bệnh nhân rời màn hình. Với Medplum, TTL phải tự viết (Bot cron quét `Appointment?status=pending&_lastUpdated=lt…` rồi `$cancel`).
</details>

---

## Phần 8 — Checklist tái sử dụng

- [ ] Ô trống là **dữ liệu** hay **kết quả tính**? Đã chọn có chủ đích
- [ ] Kiểm tra trùng là kiểm tra **khoảng chồng**, không phải giờ bằng nhau
- [ ] Bước kiểm tra và bước ghi nằm trong **cùng 1 giao dịch** có cơ chế chống race
- [ ] Buffer, sức chứa, nhiều tài nguyên đã có trong mô hình
- [ ] Giữ chỗ có **hết hạn** và có người/việc nhả chỗ
- [ ] Huỷ lịch giữ lại `Appointment` (thống kê no-show, tranh chấp)
- [ ] Có đường cho phép chen ca có kiểm soát
- [ ] Đã test 2 request song song thật, không chỉ test tuần tự

**Tự luyện tiếp theo:** thiết kế "danh sách chờ": khi có người huỷ ca 9:00, tự mời người đầu danh sách chờ, giữ chỗ 15 phút cho họ. Cần thêm thực thể gì, và chỗ nào có race?

**Nguồn:**
- 📘 FHIR R4 Schedule/Slot/Appointment: https://hl7.org/fhir/R4/appointment.html
- 📘 Docs Medplum: `packages/docs/docs/scheduling/`, `packages/docs/docs/decision-guides/scheduling.md`
- 📘 Postgres SERIALIZABLE: https://www.postgresql.org/docs/16/transaction-iso.html
