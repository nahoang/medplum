# 3b — Đặt lịch: `$find`, `$hold`, `$book`, `$confirm`, `$cancel`, `$reschedule`

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Medplum tính khung giờ trống thế nào, và làm sao bảo đảm 2 người bấm cùng lúc thì chỉ 1 người được?"* Thiết kế từ đầu: [`3b-greenfield-dat-lich.md`](./3b-greenfield-dat-lich.md). So sánh kỹ thuật: [`3b-comparison-chong-dat-trung.md`](./3b-comparison-chong-dat-trung.md).
>
> Đường dẫn tính từ `packages/server/src/fhir/operations/` nếu không ghi khác. API đang ở trạng thái **beta** (`@experimental - Scheduling Beta API` trong mọi handler).

## Mục lục

- [1. Mô hình: 4 resource + 1 extension](#1-mô-hình-4-resource--1-extension)
- [2. `SchedulingParameters`: quy tắc thay cho ô sinh sẵn](#2-schedulingparameters-quy-tắc-thay-cho-ô-sinh-sẵn)
- [3. `$find`: tính giờ trống](#3-find-tính-giờ-trống)
- [4. Sức chứa: quét sự kiện vào/ra](#4-sức-chứa-quét-sự-kiện-vàora)
- [5. `$hold` / `$book`: ghi trong giao dịch SERIALIZABLE](#5-hold--book-ghi-trong-giao-dịch-serializable)
- [6. Vòng đời sau khi đặt](#6-vòng-đời-sau-khi-đặt)
- [7. Lịch lặp](#7-lịch-lặp)
- [8. Kịch bản Lan và Hùng](#8-kịch-bản-lan-và-hùng)
- [9. Đối chiếu OpenEMR](#9-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Mô hình: 4 resource + 1 extension

| Resource | Vai trò | Ghi chú Medplum |
|---|---|---|
| `HealthcareService` | Loại dịch vụ (khám nội tiết 20') | Mang tham số mặc định của dịch vụ |
| `Schedule` | Lịch của **1** tài nguyên | 📘 Medplum bắt buộc đúng 1 `actor` (FHIR cho phép nhiều). Actor phải có extension timezone (`packages/docs/docs/scheduling/index.md`, mục "Design Decisions") |
| `Slot` | Khoảng thời gian **bị chiếm** (`busy`, `busy-tentative`, `busy-unavailable`) hoặc **thêm giờ** 1 lần (`free`) | Không sinh sẵn |
| `Appointment` | Lịch hẹn, trỏ tới các `Slot` | Đánh dấu bằng extension `SchedulingBookedByOperation` (`utils/scheduling.ts:1082-1085`) |
| Extension `SchedulingParameters` | Quy tắc: giờ làm việc, thời lượng, buffer, lưới, sức chứa, múi giờ | Lớp quyết định **Medplum** (K1). R4 không có, R5 có kiểu `Availability` và Medplum mã hoá lại dạng extension R4 (`utils/scheduling-parameters.ts:54-71`) |

📘 Quyết định thiết kế chính được Medplum tự công bố: *"Recurring availability does not require pre-generated slots"* (`packages/docs/docs/scheduling/index.md`).

## 2. `SchedulingParameters`: quy tắc thay cho ô sinh sẵn

✅ `utils/scheduling-parameters.ts:73-117`:

| Tham số | Đơn vị | Ý nghĩa |
|---|---|---|
| `availability` | ngày trong tuần + giờ bắt đầu/kết thúc | Giờ làm việc lặp hằng tuần. `end <= start` nghĩa là qua đêm (`:119-121`) |
| `duration` | phút | Thời lượng 1 lượt |
| `bufferBefore`, `bufferAfter` | phút | Đệm trước/sau |
| `alignmentInterval`, `alignmentOffset` | phút | Lưới giờ bắt đầu (VD mỗi 20 phút, lệch 0) |
| `slotCapacity` | số nguyên ≥ 1 | Số lượt đồng thời. `1` = không overbook (`:105`) |
| `service` | Reference | Tham số này áp cho dịch vụ nào |
| `timezone`, `alignmentTimezone` | IANA | Múi giờ tính lịch |

Đơn vị thời gian chỉ cho phép `min`, `h`, `d`, `wk`. Cấm `ms`, `s` (quá mịn) và `mo`, `a` (độ dài không cố định) (`:36-39`).

Tham số được đặt ở cả `Schedule` lẫn `HealthcareService`, ghép thành 1 `LayeredDict` (lớp lịch đè lên lớp dịch vụ) (`utils/scheduling.ts:580+`, `getSchedulingParametersGroup`).

## 3. `$find`: tính giờ trống

✅ `find.ts` (handler `:100-363`, entry `:365`). Đầu vào: `start`, `end`, `service-type-reference`, 1..N `schedule`, `ignore-appointment`, `occurrence-count`, `_count` (`:55-62`).

Thuật toán:
1. Đọc các `Slot` chồng lên khoảng tìm (`slotsOverlappingInterval`, `utils/scheduling.ts:661-692`). **Đủ 1 trang tối đa thì báo lỗi** ("try searching with smaller bounds", `:686-689`) thay vì trả kết quả thiếu.
2. Bỏ các Slot của `ignore-appointment` (dùng khi đổi lịch, để lịch hẹn không tự chặn chính nó) (`find.ts:155-158`).
3. Cắt khoảng tìm theo `Schedule.planningHorizon` (`:174-195`).
4. **Giờ trống = giờ làm việc ∪ Slot `free` − khoảng đã đầy** (`applyExistingSlots`, `utils/scheduling.ts:458-480`).
5. Sinh giờ bắt đầu khớp lưới, đủ thời lượng + buffer (`utils/find.ts:33+`, `findAlignedSlotTimes`).
6. Trả về `Appointment` **ảo** (`status: proposed`) có `contained` Slot. Không ghi gì vào DB.

📘 Nhiều `schedule` = tìm giờ mà **mọi** tài nguyên cùng rảnh (bác sĩ + phòng siêu âm). Lịch lặp yêu cầu mọi lịch chung 1 múi giờ (`find.ts:197-201`).

## 4. Sức chứa: quét sự kiện vào/ra

✅ `intervalsExceedingCapacity` (`utils/scheduling.ts:384-432`):
- `occupiesTime`: mọi trạng thái trừ `free`, `entered-in-error` đều chiếm chỗ (`:322-324`). Nghĩa là **`busy-tentative` (đang giữ) cũng chiếm chỗ**.
- Sức chứa 1 có đường nhanh riêng (`:389-391`, `:345`).
- Sức chứa > 1: sắp các mốc bắt đầu/kết thúc theo thời gian, mỗi mốc cập nhật danh sách đang mở. Khoảng bị chặn khi số lượt đang mở ≥ **sức chứa nhỏ nhất** trong các lượt đang mở và của lượt định đặt (`:418-421`).
- Lượt kết thúc đúng lúc lượt khác bắt đầu thì không tính chồng (`:403-407`).

🧠 Vì sao lấy **nhỏ nhất**: lớp dinh dưỡng (sức chứa 6) và khám riêng (sức chứa 1) cùng dùng 1 phòng. Đã có 1 lượt khám riêng thì không được nhét thêm 5 người lớp nhóm. Mỗi Slot được "đóng dấu" sức chứa lúc đặt (`stampSlotCapacity`, `:492-506`). Slot không có dấu được coi là sức chứa 1, tức **mặc định chặn chặt** (`getSlotCapacity`, `:441-444`). Test `book.test.ts:846+` ("a capacity-1 booking cannot be overbooked by a capacity-2 service").

Buffer (`busy-unavailable`) luôn tính sức chứa 1 (`utils/scheduling.ts:867`): thời gian dọn phòng không chia sẻ.

## 5. `$hold` / `$book`: ghi trong giao dịch SERIALIZABLE

`$hold` và `$book` dùng chung `createProposedAppointments` (`utils/scheduling.ts:1062-1121`), chỉ khác hàm tuỳ biến:

| Operation | Appointment | Slot `busy` | ✅ |
|---|---|---|---|
| `$book` | `booked` | giữ `busy` | `book.ts:49` |
| `$hold` | `pending` | đổi thành `busy-tentative` | `hold.ts:49-56` |

Các bước:

| Bước | Ở đâu | Trong giao dịch? |
|---|---|---|
| 1. Kiểm tra hình thức: đúng 1 service, có Slot contained, đúng 1 Slot `busy` mỗi lịch, thời lượng khớp, giờ khớp lưới, buffer đúng (`validateProposedAppointment`, `validateSlots`) | `:905-972`, `:695-799` | Không |
| 2. Cấm client tự gắn `Appointment.slot` | `:1075-1080` | Không |
| 3. **Đọc lại Slot và kiểm tra còn chỗ** (`validateAllAvailability` → `validateAvailability`) | `:1097-1099`, `:811-903` | **Có** |
| 4. Tạo Slot, rồi tạo Appointment trỏ tới Slot | `:1100-1108` | **Có** |
| Giao dịch: `{ serializable: true, resourceTypes: ['Appointment', 'Slot'] }` | `:1114` | |

Hết chỗ thì trả `400` với `diagnostics` là JSON liệt kê các Slot đang chặn (`:871-901`), để ứng dụng hiển thị "đã có người đặt".

### Vì sao SERIALIZABLE đủ chống trùng

✅ `repository/repository-connection.ts`:
- Mặc định giao dịch là `REPEATABLE READ`, `serializable: true` thì đổi sang `SERIALIZABLE` (`:516`).
- Lỗi Postgres `40001` (serialization failure) được coi là **thử lại được** (`../sql.ts:681`, `:696-704`).
- Thử lại **toàn bộ callback** tối đa `transactionAttempts` lần, mặc định **3** (`:26`, `:519-521`), chờ `50 ms × 2^n × [0.75, 1.25]` giữa các lần (`:27`, `:566-572`).
- Giao dịch lồng (savepoint) không được thử lại (`:556`).

🧠 Cơ chế: 2 giao dịch cùng **đọc** "Slot trong 9:00–9:20 của BS Minh" (không có gì) rồi cùng **ghi** 1 Slot rơi vào đúng phạm vi đó. Postgres SSI phát hiện vòng phụ thuộc đọc–ghi, buộc 1 giao dịch lỗi `40001`. Giao dịch đó chạy lại từ đầu, bước 3 lần này đọc thấy Slot của người thắng → báo hết chỗ. Không cần khoá dòng nào, vì **chưa có dòng nào để khoá**.

✅ Test bắn 2 `$book` song song vào chỗ cuối cùng: đúng 1 trả `201`, 1 bị từ chối, số Slot không vượt sức chứa (`book.test.ts:781-843`).

## 6. Vòng đời sau khi đặt

| Operation | Điều kiện | Làm gì | ✅ |
|---|---|---|---|
| `$confirm` | `pending` hoặc `proposed` | Kiểm tra `HealthcareService` và `Schedule` còn `active`. `busy-tentative` → `busy`, Appointment → `booked`. SERIALIZABLE | `confirm.ts:33-107` |
| `$cancel` | `pending` hoặc `booked` | Appointment → `cancelled` (+ lý do), **xoá mềm** mọi Slot | `cancel.ts:39-73`, xoá ở `:62` |
| `$reschedule` | `pending` hoặc `booked` | Nhả Slot cũ **trước**, rồi kiểm tra giờ mới, tạo Slot mới, sửa Appointment tại chỗ. **Giữ nguyên `status`**: lịch đang giữ thì Slot mới vẫn `busy-tentative` | `reschedule.ts:104+`, `:128-131`, `:190-194`, `:208`, `:246` |

📘 **Không có hết hạn giữ chỗ.** *"Define expiry tracking and cleanup in the application; `$hold` has no built-in TTL"* (`packages/docs/docs/decision-guides/scheduling.md`, mục 3.8). Lịch `pending` bị bỏ dở sẽ chiếm chỗ mãi (vì `busy-tentative` chiếm chỗ, mục 4) cho tới khi ai đó gọi `$cancel`.

🧠 Đổi lịch không đổi được **loại dịch vụ**: `serviceType` của Appointment quyết định thời lượng, buffer. Đổi loại dịch vụ là huỷ và đặt mới (`reschedule.ts:86-92` docblock).

🧠 `$cancel` xoá Slot nhưng giữ Appointment `cancelled`. Thống kê huỷ, no-show đọc từ Appointment, còn "thời gian đã nhả" thể hiện bằng việc Slot biến mất (vẫn còn trong `_history`, [1f](./1f-history-audit-deletion.md)).

## 7. Lịch lặp

✅ `utils/recurrence.ts`:
- Chuỗi hằng tuần **2–6 lượt** (`:22-23`).
- Giữ **giờ địa phương** qua đổi giờ mùa hè. Giờ không tồn tại (rơi vào khoảng nhảy giờ) thì bỏ, giờ lặp 2 lần thì lấy lần đầu (`projectWeeksForward`, `:38-42`).
- `$book`/`$hold` nhận `recurrenceTemplate`, mở ra N Appointment rồi đặt **tất cả hoặc không** trong cùng 1 giao dịch (`hold.ts:44-47`, vòng lặp tuần tự ở `utils/scheduling.ts:1093-1111`).
- `$find` với `occurrence-count` kiểm tra cùng khoảng ở các tuần sau, nới thêm 1 giờ phòng đổi giờ (`find.ts:111-119`).

🧠 Giới hạn 6 lượt hợp với "tái khám 4 tuần/lần trong nửa năm" nhưng **không** đủ cho "vật lý trị liệu 2 buổi/tuần × 12 tuần" (24 lượt). Phải chia nhiều chuỗi.

## 8. Kịch bản Lan và Hùng

💭 BS Minh: availability T2 8:00–12:00, duration 20, bufferAfter 5, alignmentInterval 25 (20 + 5), slotCapacity 1.

| Thời điểm | Lan (`$book` 9:00) | Hùng (`$book` 9:00) |
|---|---|---|
| t=0 | Kiểm tra hình thức ✔ | |
| t=2 ms | BEGIN SERIALIZABLE, đọc Slot 9:00–9:25 → rỗng | Kiểm tra hình thức ✔ |
| t=4 ms | | BEGIN SERIALIZABLE, đọc Slot 9:00–9:25 → rỗng |
| t=6 ms | INSERT Slot busy 9:00–9:20, Slot busy-unavailable 9:20–9:25, Appointment | |
| t=8 ms | COMMIT ✔ → `201` | INSERT … |
| t=9 ms | | COMMIT → **40001** |
| t≈60 ms | | Thử lại lần 2: đọc thấy Slot của Lan → `400 Requested time slot is not available` |

Kết quả DB: 1 Appointment `booked`, 2 Slot. Hùng nhận lỗi kèm danh sách Slot đang chặn.

## 9. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Lưu lịch hẹn | `Appointment` + `Slot` | 1 dòng `openemr_postcalendar_events` (`sql/database.sql:8275-8319`) |
| Giờ làm việc | Extension quy tắc | **Cùng bảng**: sự kiện loại "In Office" (`pc_catid = 2`) / "Out of Office" (`3`) |
| Tìm giờ trống | `$find` (server) | Popup tính mảng bit: 1 = trong giờ, 2 = ngoài giờ, 4 = đã có hẹn (`interface/main/calendar/find_appt_popup.php:65-90`) |
| Chống trùng khi ghi | SERIALIZABLE + kiểm tra lại | **Không**: `AppointmentService::validate` chỉ kiểm tra trường bắt buộc, định dạng (`src/Services/AppointmentService.php:90-120`). `insert` ghi thẳng (`:311+`) |
| Lặp | Chuỗi 2–6 lượt, tạo N Appointment | 1 dòng + `pc_recurrtype`, `pc_recurrspec` (quy tắc lặp lưu trong sự kiện) |
| Buffer, sức chứa | Có | Không |

🧠 OpenEMR coi lịch là **sổ hẹn cho lễ tân**: cho đặt trùng là tính năng (chen ca), cảnh báo là việc của con người. Medplum coi lịch là **API cho bệnh nhân tự đặt**: phải chặn ở server vì không có người nào đứng giữa.

## Sai lầm thường gặp

1. **Dùng `$hold` mà không có việc nhả chỗ.** Chỗ bị chiếm vĩnh viễn. Cần Bot cron ([2b](./2b-bots.md)) quét `Appointment?status=pending&_lastUpdated=lt<now-10m>` rồi `$cancel`. Và phải xử lý race giữa Bot nhả chỗ với `$confirm` của bệnh nhân (docs nhắc rõ ở mục 3.8).
2. **Tạo `Slot free` cho mọi giờ làm việc** theo thói quen FHIR. Không cần, và Slot `free` được **cộng** vào giờ trống: tạo nhầm thì mở ra giờ ngoài giờ làm việc.
3. **Ghi thẳng `Appointment` + `Slot` bằng API CRUD** thay vì `$book`. Bỏ qua toàn bộ kiểm tra. Nên chặn bằng AccessPolicy cho tài khoản bệnh nhân.
4. **Tìm trong khoảng quá rộng** ở lịch dày: chạm trần 1 trang Slot là lỗi (`utils/scheduling.ts:686-689`), không phải kết quả thiếu.
5. **Quên timezone trên actor.** Bị từ chối, hoặc tính giờ lệch.
6. **Coi `busy-tentative` là còn trống.** Nó chiếm chỗ như `busy`.

## Tự kiểm tra

1. Vì sao `$find` không cần giao dịch, còn `$book` thì cần SERIALIZABLE?
2. Lớp nhóm sức chứa 6 đã có 5 người. 2 người cùng đặt chỗ cuối. Điều gì xảy ra?
3. Lan giữ chỗ lúc 7:59, rồi tắt app. 10 phút sau Hùng tìm giờ. Hùng có thấy 9:00 không? Cần thêm gì?
4. Đổi lịch của Lan từ 9:00 sang 9:25 khi 9:25 đang trống. Vì sao Slot 9:00 của chính Lan không chặn việc này?
