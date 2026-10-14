# 3b-C — So sánh 6 cách chống đặt trùng lịch

> **Loại file:** C (theo [`../templates/saleor-design-technique-comparison-product-attribute.md`](../templates/saleor-design-technique-comparison-product-attribute.md)).
>
> Trả lời câu: *"Có ít nhất 6 cách bảo đảm 1 khung giờ không bị đặt quá sức chứa. Vì sao Medplum chọn giao dịch SERIALIZABLE + kiểm tra lại, và khi nào cách đó không còn đúng?"*
>
> Khác 2 file cùng module: [G](./3b-greenfield-dat-lich.md) đi từ nghiệp vụ tới thiết kế, [R](./3b-dat-lich.md) đọc code thật. File này đặt các cách làm cạnh nhau trên **cùng 1 kịch bản**.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bài toán: 8 đặc tính](#phần-1--bài-toán-8-đặc-tính)
- [Phần 2 — 6 ứng viên](#phần-2--6-ứng-viên)
- [Phần 3 — Bảng so sánh trực diện](#phần-3--bảng-so-sánh-trực-diện)
- [Phần 4 — Vì sao 5 ứng viên bị loại](#phần-4--vì-sao-5-ứng-viên-bị-loại)
- [Phần 5 — 4 lối tắt nghe hợp lý nhưng sai](#phần-5--4-lối-tắt-nghe-hợp-lý-nhưng-sai)
- [Phần 6 — Thiết kế thật tại từng ngã rẽ](#phần-6--thiết-kế-thật-tại-từng-ngã-rẽ-)
- [Phần 7 — Đánh đổi đo bằng số](#phần-7--đánh-đổi-đo-bằng-số-)
- [Phần 8 — Đối chứng trong chính Medplum](#phần-8--đối-chứng-trong-chính-medplum)
- [Phần 9 — Luyện phản biện](#phần-9--luyện-phản-biện)
- [Phần 10 — Checklist · Tự luyện · Nguồn](#phần-10--checklist--tự-luyện--nguồn)

## Quy tắc nhãn nguồn

✅ code thật `file:line` · 📘 chuẩn/docs · 💭 code minh hoạ, số liệu giả định · 🧠 suy luận.

**Kịch bản xuyên suốt 💭:** BS Minh, Thứ Hai 9:00–9:20, buffer sau 5 phút. Lan (app) và Hùng (lễ tân) cùng `$book` lúc 7:59:59. Thêm 1 biến thể: **lớp dinh dưỡng** sức chứa 6, đã có 5 người, 3 người tranh chỗ cuối.

---

## Phần 1 — Bài toán: 8 đặc tính

| # | Đặc tính | Vì sao tồn tại | Hậu quả nếu bỏ qua 💭 |
|---|---|---|---|
| P1 | Trùng là **khoảng chồng nhau**, không phải giờ bằng nhau | Lượt 9:10–9:30 (đổi lịch tay) chồng lên 9:00–9:20 | 2 bệnh nhân trong phòng. Với 30 bác sĩ × 15 lượt/ngày, 1% lỗi = ~4,5 ca trùng/ngày |
| P2 | Sức chứa N > 1, và **lấy sức chứa nhỏ nhất** khi 2 loại dịch vụ chung tài nguyên | Lớp nhóm và khám riêng chung 1 phòng | Nhét 5 người lớp nhóm vào giờ đã có 1 ca khám riêng |
| P3 | 1 lịch hẹn chiếm **nhiều tài nguyên** cùng lúc, tất cả hoặc không | Siêu âm cần bác sĩ + phòng | Giữ được bác sĩ, mất phòng → bác sĩ ngồi không 30' |
| P4 | Giờ trống là **quy tắc**, không có sẵn dòng để khoá | Đổi giờ làm việc không phải sinh lại ô ([G](./3b-greenfield-dat-lich.md#phần-3--mô-hình-khái-niệm-)) | Cơ chế dựa trên "khoá dòng ô trống" không áp dụng được |
| P5 | Có trạng thái **giữ tạm** cũng chiếm chỗ | Bệnh nhân nhập bảo hiểm 2–3 phút | Bấm xác nhận mới biết hết chỗ |
| P6 | Chuỗi lặp **tất cả hoặc không** | Tái khám 4 tuần/lần × 6 | Đặt được 4/6 lượt, bệnh nhân không biết |
| P7 | **Điểm nóng** lúc mở đặt | Mở đặt vắc-xin 8:00, hàng trăm người cùng 1 khung | Cơ chế huỷ–thử lại sinh lỗi giả "conflict" |
| P8 | Pháp lý/vận hành: huỷ phải **còn dấu vết**, lễ tân được chen ca **có kiểm soát** | Thống kê no-show, tranh chấp, cấp cứu | Chặn cứng tuyệt đối thì nhân viên quay về sổ giấy |

---

## Phần 2 — 6 ứng viên

### Ứng viên 1 — Không kiểm tra, để con người xử lý (OpenEMR) ✅

Nguồn gốc: phần mềm quản lý phòng khám thời kỳ "lễ tân là người duy nhất đặt lịch".

✅ OpenEMR `src/Services/AppointmentService.php:90-120` (`validate`) chỉ kiểm tra trường bắt buộc và định dạng. `insert` (`:311+`) ghi thẳng 1 dòng `openemr_postcalendar_events`. Popup tìm giờ trống (`interface/main/calendar/find_appt_popup.php:65-90`) chỉ để **gợi ý**.

| Kịch bản | Kết quả |
|---|---|
| Lan & Hùng cùng 9:00 | Cả 2 được ghi |
| Lớp nhóm | Không có khái niệm sức chứa |

**Sai lầm thường gặp:** coi đây là "lỗi". Với phòng khám chỉ lễ tân đặt, đây là lựa chọn hợp lý: chen ca là nhu cầu thật (P8).

### Ứng viên 2 — Kiểm tra rồi ghi, isolation mặc định 💭

```sql
-- READ COMMITTED (mặc định Postgres)
SELECT count(*) FROM slot WHERE schedule_id = 'minh'
  AND start_at < '09:20' AND end_at > '09:00' AND status <> 'free';
-- 0 → INSERT
```

| t | Lan | Hùng |
|---|---|---|
| 0 | SELECT → 0 | |
| 3 ms | | SELECT → 0 |
| 6 ms | INSERT, COMMIT | |
| 9 ms | | INSERT, COMMIT → **trùng** |

Tỉ lệ lỗi phụ thuộc khoảng cách giữa 2 request so với thời gian 1 giao dịch. Test tuần tự **luôn xanh**, nên lỗi chỉ lộ ra ở production.

**Sai lầm thường gặp:** nghĩ `REPEATABLE READ` sửa được. Không: cả 2 vẫn thấy snapshot rỗng và cùng INSERT dòng **mới** (không ai sửa cùng 1 dòng nên không có xung đột ghi).

### Ứng viên 3 — Sinh sẵn ô, chiếm ô bằng UPDATE có điều kiện 💭

```sql
UPDATE slot SET status = 'busy', appointment_id = $1
WHERE id = 'minh-2026-10-19-0900' AND status = 'free';
-- rowcount 0 → đã mất chỗ
```

Đây là cách "chuẩn sách" nhiều hệ thống FHIR dùng: `Slot free` sinh sẵn cho cả planning horizon. Postgres khoá dòng khi UPDATE nên chỉ 1 người thắng.

| Kịch bản | Kết quả |
|---|---|
| Lan & Hùng | ✔ 1 thắng (khoá dòng) |
| Lớp nhóm | Thêm cột `booked_count`, `WHERE booked_count < capacity` ✔ |
| Buffer | ✘ Buffer 5' không khớp lưới ô 20' |
| Đổi giờ làm việc | ✘ Sinh lại ô, lịch cũ lệch lưới |

**Sai lầm thường gặp:** sinh ô bằng cron đêm, quên múi giờ/đổi giờ mùa hè → ô lệch 1 giờ 2 lần/năm.

### Ứng viên 4 — Ràng buộc loại trừ của Postgres (exclusion constraint) 💭

```sql
CREATE EXTENSION btree_gist;
ALTER TABLE slot ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (schedule_id WITH =, tstzrange(start_at, end_at) WITH &&)
  WHERE (status IN ('busy', 'busy-tentative', 'busy-unavailable'));
```

CSDL tự từ chối mọi Slot chồng khoảng trên cùng lịch, **bất kể** ứng dụng viết gì.

| Kịch bản | Kết quả |
|---|---|
| Lan & Hùng | ✔ Người sau nhận lỗi `23P01` |
| Chồng khoảng (P1) | ✔ Đúng bản chất (`&&` = chồng nhau) |
| Lớp nhóm (P2) | ✘ Ràng buộc chỉ biết "0 hoặc 1", không đếm được N |
| Ghi thẳng qua API CRUD | ✔ Vẫn bị chặn (ưu điểm lớn nhất) |

**Sai lầm thường gặp:** quên `WHERE status …`, làm Slot đã huỷ vẫn chặn.

### Ứng viên 5 — Khoá bi quan theo tài nguyên 💭

```sql
BEGIN;
SELECT pg_advisory_xact_lock(hashtext('schedule:minh'));   -- hoặc SELECT … FROM schedule WHERE id='minh' FOR UPDATE
-- kiểm tra còn chỗ, INSERT
COMMIT;  -- khoá tự nhả
```

Mọi lượt đặt trên lịch BS Minh **xếp hàng**.

| Kịch bản | Kết quả |
|---|---|
| Lan & Hùng | ✔ Hùng chờ ~6 ms, rồi thấy hết chỗ |
| Lớp nhóm, 3 người tranh chỗ cuối | ✔ Xếp hàng, người 1 được, 2 và 3 thấy hết chỗ. **Không có lỗi giả** |
| Nhiều tài nguyên (P3) | ⚠ Phải khoá theo **thứ tự cố định** (sắp id) để tránh deadlock |
| 2 lượt khác ngày cùng bác sĩ | ⚠ Vẫn xếp hàng dù không xung đột |

**Sai lầm thường gặp:** khoá theo thứ tự request gửi lên (bác sĩ rồi phòng / phòng rồi bác sĩ) → deadlock.

### Ứng viên 6 — SERIALIZABLE + kiểm tra lại trong giao dịch + thử lại (Medplum) ✅

✅ `packages/server/src/fhir/operations/utils/scheduling.ts:1093-1114`: trong `withTransaction({ serializable: true })`, đọc lại Slot chồng khoảng, kiểm tra sức chứa, rồi tạo Slot + Appointment. Lỗi `40001` thử lại tối đa 3 lần (`packages/server/src/fhir/repository/repository-connection.ts:26-27`, `:516-583`).

| Kịch bản | Kết quả |
|---|---|
| Lan & Hùng | ✔ Hùng lỗi `40001` → thử lại → `400 not available` (test thật: `book.test.ts:781-843`) |
| Lớp nhóm, 3 người tranh chỗ cuối | ✔ Vòng 1: 1 người commit, 2 người `40001`. Vòng 2: cả 2 thấy đầy → 400 |
| Nhiều tài nguyên | ✔ Cùng 1 giao dịch, không cần thứ tự khoá |
| 2 lượt khác ngày | ✔ Không đọc chồng phạm vi → không xung đột |
| Điểm nóng (P7) | ⚠ Xem Phần 7 |

**Sai lầm thường gặp:** quên rằng mọi đường ghi khác (API CRUD tạo `Slot` trực tiếp) **không** đi qua kiểm tra này.

---

## Phần 3 — Bảng so sánh trực diện

| | 1 Không kiểm tra | 2 Check-then-insert | 3 Ô sinh sẵn + UPDATE | 4 Exclusion constraint | 5 Khoá tài nguyên | 6 SERIALIZABLE |
|---|---|---|---|---|---|---|
| P1 khoảng chồng | ✘ | ✘ (race) | ⚠ chỉ trong lưới ô | ✔ | ✔ | ✔ |
| P2 sức chứa N, min | ✘ | ✘ | ⚠ đếm, không có "min" | ✘ | ✔ | ✔ |
| P3 nhiều tài nguyên | ✘ | ✘ | ✔ cần giao dịch | ✔ | ⚠ thứ tự khoá | ✔ |
| P4 không sinh sẵn ô | ✔ | ✔ | ✘ | ✔ | ✔ | ✔ |
| P5 giữ tạm chiếm chỗ | — | ⚠ | ✔ | ✔ | ✔ | ✔ |
| P6 chuỗi tất cả/không | ✘ | ✘ | ✔ | ✔ | ✔ | ✔ |
| P7 điểm nóng | ✔ (không chặn) | ✘ | ✔ khoá dòng | ✔ | ✔ xếp hàng | ⚠ huỷ–thử lại |
| P8 chen ca có kiểm soát | ✔ | ✔ | ⚠ | ✘ chặn cả API | ✔ | ✔ qua đường CRUD |
| Phụ thuộc DB | Không | Không | Không | **Postgres** (gist) | Postgres (advisory) hoặc `FOR UPDATE` | Postgres SSI (DB khác hành vi khác) |

---

## Phần 4 — Vì sao 5 ứng viên bị loại

| Ứng viên | Loại vì |
|---|---|
| 1 | Medplum phục vụ **bệnh nhân tự đặt qua API**: không có lễ tân đứng giữa (P1). |
| 2 | Race thật, đã chứng minh ở bảng thời gian (P1). |
| 3 | Mâu thuẫn trực tiếp với quyết định "không sinh sẵn Slot" (P4), là quyết định Medplum tự công bố (📘 `packages/docs/docs/scheduling/index.md`). Buffer không khớp lưới. |
| 4 | Không đếm được sức chứa N (P2). Và chặn cả đường ghi CRUD, mất đường chen ca (P8). Lý do phụ 🧠: Slot của Medplum lưu theo mô hình bảng chung ([1a](./1a-fhir-datastore.md)), `start`/`end` là cột search phái sinh, gắn ràng buộc nghiệp vụ lên bản chiếu là đi ngược kiến trúc "content là nguồn sự thật". |
| 5 | Hợp lệ, nhưng tuần tự hoá **mọi** lượt đặt của 1 tài nguyên và cần thứ tự khoá khi nhiều tài nguyên. 🧠 Với tải phân tán (nhiều bác sĩ, ít xung đột), SERIALIZABLE rẻ hơn và code đơn giản hơn. |

---

## Phần 5 — 4 lối tắt nghe hợp lý nhưng sai

**3 lối dừng quá sớm:**
1. **"Đặt UNIQUE (schedule, start)."** Chặn 9:00 vs 9:00, không chặn 9:10 vs 9:00 (P1).
2. **"Dùng REPEATABLE READ là đủ an toàn."** Không có xung đột ghi khi 2 bên cùng INSERT dòng mới (Ứng viên 2).
3. **"Kiểm tra lại ở frontend ngay trước khi gửi."** Vẫn là check-then-act, chỉ dời chỗ race.

**1 lối leo quá xa:**
4. **"Đưa mọi lượt đặt qua 1 hàng đợi tuần tự (Kafka, 1 consumer/tài nguyên)."** Đúng ở quy mô bán vé hòa nhạc, nhưng biến đặt lịch thành bất đồng bộ: app phải chờ kết quả qua kênh khác. Thêm hạ tầng, thêm điểm hỏng, cho bài toán 50 lượt/phút.

---

## Phần 6 — Thiết kế thật tại từng ngã rẽ ✅

| Ngã rẽ | Medplum chọn | Vì sao không đi nhánh kia |
|---|---|---|
| Ô trống lưu hay tính? | Tính (`resolveAvailability`, `utils/scheduling.ts:191-213`) | Ứng viên 3 phải đồng bộ ô với quy tắc |
| Kiểm tra ở đâu? | **2 lần**: ngoài giao dịch (hình thức, `:905-972`) và trong giao dịch (còn chỗ, `:1097-1099`) | Kiểm tra hình thức trong giao dịch làm giao dịch dài, tăng xung đột |
| Isolation? | `SERIALIZABLE` chỉ cho các operation lịch (`:1114`, `confirm.ts:96`, `cancel.ts:68`). Còn lại `REPEATABLE READ` (`repository-connection.ts:516`) | Bật SERIALIZABLE toàn hệ thống thì mọi ghi FHIR chịu chi phí SSI |
| Sức chứa? | Đóng dấu sức chứa lên từng Slot, quét lấy min (`:384-432`, `:492-506`) | Ứng viên 4 không đếm được |
| Thử lại? | 3 lần, backoff 50 ms × 2^n (`repository-connection.ts:26-27`, `:566-572`) | Không thử lại thì lỗi `40001` thành `409` cho người dùng dù còn chỗ |
| Giữ tạm hết hạn? | **Không làm**, để ứng dụng (📘 decision guide mục 3.8) | 🧠 Hết hạn là chính sách kinh doanh (3 phút? 24 giờ chờ duyệt bảo hiểm?) |

---

## Phần 7 — Đánh đổi đo bằng số 🧠

Ước lượng thô, giả định 1 giao dịch đặt lịch mất ~10 ms.

| Tình huống | Ứng viên 5 (khoá) | Ứng viên 6 (SERIALIZABLE, 3 lần thử) |
|---|---|---|
| 50 lượt/phút trên 30 bác sĩ | ~0 chờ | ~0 xung đột |
| 2 người tranh 1 chỗ (sức chứa 1) | Người sau chờ 10 ms → "hết chỗ" | Người sau `40001` → thử lại sau ~50 ms → "hết chỗ" |
| 3 người tranh 1 chỗ cuối (lớp nhóm) | Xếp hàng, ~20 ms | Vòng 1: 1 thắng. Vòng 2: 2 người thấy đầy → 400 |
| **50 người tranh 6 chỗ** (mở đặt lớp nhóm) | Xếp hàng ~500 ms, 6 được, 44 "hết chỗ" | Mỗi vòng chỉ ~1 giao dịch commit trong nhóm xung đột. Sau **3 vòng** chỉ ~3 chỗ được lấp. ~47 người nhận **`409 conflict`**, trong đó nhiều người lẽ ra còn chỗ |

Dòng cuối là điểm yếu thật của ứng viên 6 ở điểm nóng: lỗi trả về là **"xung đột, thử lại"** chứ không phải **"hết chỗ"**. Ứng dụng phải tự thử lại (ở client) hoặc tăng `transactionAttempts` (cấu hình server, `repository-connection.ts:519`).

---

## Phần 8 — Đối chứng trong chính Medplum

Chỗ Medplum **không** dùng SERIALIZABLE + kiểm tra lại, dù cũng là "tài nguyên có hạn":
- **Ghi FHIR thường** (`createResource`, `updateResource`) dùng `REPEATABLE READ`. Chống ghi đè đồng thời bằng **If-Match / version** (optimistic, [1f](./1f-history-audit-deletion.md)). Đó là bài toán "2 người sửa cùng 1 dòng", không phải "2 người tạo 2 dòng chồng nhau", nên không cần SSI.
- **Tạo có điều kiện** (`If-None-Exist`) chống trùng theo **khoá bằng nhau**, đủ cho "1 bệnh nhân 1 CCCD" ([3a](./3a-dinh-danh-tiep-nhan.md)), không đủ cho khoảng thời gian.
- **`$expunge`** cũng dùng SERIALIZABLE (`packages/server/src/fhir/repo.ts:1756`): xoá thật cần chắc không ai đang ghi dở.

🧠 Quy tắc rút ra: SERIALIZABLE dành cho **bất biến trên tập hợp** ("tổng số lượt chồng ≤ N", "không còn tham chiếu") mà không có 1 dòng đại diện để khoá.

---

## Phần 9 — Luyện phản biện

**Phản đối 1:** "Exclusion constraint chặn ở tầng CSDL, an toàn tuyệt đối, sao không dùng?"

<details><summary>Đáp án gợi ý</summary>

Nó không đếm được sức chứa N và chặn luôn đường chen ca có kiểm soát. Có thể dùng **kết hợp**: constraint cho tài nguyên sức chứa 1 (phòng mổ), SERIALIZABLE cho phần còn lại. Nhưng ở Medplum, cột `start`/`end` của Slot là bản chiếu search, không phải nơi đặt bất biến nghiệp vụ.
</details>

**Phản đối 2:** "SERIALIZABLE trên Postgres có thể báo xung đột giả (false positive), làm lỗi oan."

<details><summary>Đáp án gợi ý</summary>

Đúng, SSI có thể huỷ giao dịch không thật sự xung đột (khoá vị từ ở mức trang index). Chi phí là 1 lần thử lại ~50 ms. Đổi lại không có deadlock và không cần thứ tự khoá. Cái giá chỉ đáng lo ở điểm nóng (Phần 7).
</details>

**Phản đối 3:** "Sao không dùng Redis lock cho nhanh, Medplum có sẵn Redis?"

<details><summary>Đáp án gợi ý</summary>

Khoá Redis nằm **ngoài** giao dịch Postgres: giao dịch có thể commit sau khi khoá hết hạn, hoặc khoá nhả mà giao dịch chưa commit. Phải dùng thêm "fencing token" mới đúng. Khoá trong cùng CSDL với dữ liệu (ứng viên 5 hoặc 6) tránh được cả lớp lỗi này.
</details>

---

## Phần 10 — Checklist · Tự luyện · Nguồn

**Checklist chọn cơ chế chống trùng:**
- [ ] Bất biến là "khoá bằng nhau" (UNIQUE đủ) hay "tổng trên khoảng" (cần khoá/SSI/constraint)?
- [ ] Có dòng đại diện để khoá không? Không có → SSI hoặc advisory lock
- [ ] Sức chứa N > 1? → loại exclusion constraint
- [ ] Nhiều tài nguyên? → khoá phải có thứ tự, hoặc dùng SSI
- [ ] Điểm nóng? → khoá xếp hàng, hoặc tăng số lần thử lại, hoặc hàng đợi phía trước
- [ ] Mọi đường ghi (kể cả CRUD trực tiếp) đã đi qua cùng 1 kiểm tra, hoặc đã chặn bằng quyền?
- [ ] Đã có test bắn song song thật (như `book.test.ts:781`)?

**Tự luyện:** viết lại Ứng viên 5 cho siêu âm (bác sĩ + phòng) sao cho không bao giờ deadlock. Rồi tính: nếu 1 bác sĩ có 20 lượt đặt/phút trải trên 5 ngày khác nhau, Ứng viên 5 làm chờ trung bình bao lâu so với Ứng viên 6?

**Nguồn:**
- 📘 Postgres Transaction Isolation & SSI: https://www.postgresql.org/docs/16/transaction-iso.html
- 📘 Postgres exclusion constraint: https://www.postgresql.org/docs/16/ddl-constraints.html#DDL-CONSTRAINTS-EXCLUSION
- 📘 Medplum `packages/docs/docs/scheduling/appointment-book.md` (đoạn về SERIALIZABLE và capacity)
- ✅ File R: [`3b-dat-lich.md`](./3b-dat-lich.md)
