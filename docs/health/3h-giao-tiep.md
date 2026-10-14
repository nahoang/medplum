# 3h — Giao tiếp: tin nhắn, luồng hội thoại, fax

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"Bà Lan nhắn hỏi bác sĩ về kết quả HbA1c. Tin nhắn được lưu thế nào, ai phải trả lời, và tại sao 1 cuộc nhắn tin có thể là 1 lần khám tính tiền?"*

## Mục lục

- [1. Thread = 2 tầng `Communication`](#1-thread--2-tầng-communication)
- [2. Ai phải trả lời: Communication + Task](#2-ai-phải-trả-lời-communication--task)
- [3. Kênh ngoài: SMS, email, fax](#3-kênh-ngoài-sms-email-fax)
- [4. Nhắn tin là 1 lần khám](#4-nhắn-tin-là-1-lần-khám)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Thread = 2 tầng `Communication`

📘 `packages/docs/docs/communications/messaging-data-model.mdx`: Medplum dùng **1 resource** `Communication` cho cả "cuộc hội thoại" lẫn "tin nhắn":

| Tầng | `payload` | `partOf` | Vai trò |
|---|---|---|---|
| **Thread header** | Không có | Không có | Nhóm hội thoại. `topic` = tiêu đề, `recipient` = mọi người tham gia, `subject` = bệnh nhân |
| **Message** | Có (text, file, tham chiếu resource) | → header | 1 tin nhắn |

Tìm mọi thread: `Communication?part-of:missing=true`. Tìm tin trong 1 thread: `Communication?part-of=Communication/<header>&_sort=sent`.

| Trường | Ý nghĩa | Ví dụ 💭 |
|---|---|---|
| `category` | Phân loại rộng (thông báo, cảnh báo) | notification |
| `reasonCode` | Lý do cụ thể (SNOMED hoặc mã nội bộ) | hỏi kết quả xét nghiệm |
| `medium` | Kênh (SMS, email, in-app) | in-app |
| `status` | `in-progress`, `completed`, … | completed |
| `inResponseTo` | Trả lời 1 tin cụ thể | — |

📘 Sửa tin nhắn đã gửi = **thu hồi** tin cũ + tạo tin mới (`message-editing-and-drafts.mdx`), không sửa tại chỗ. Cùng nguyên tắc với addendum ở [3c](./3c-charting-encounter.md#4-ký-và-khoá-hồ-sơ): giao tiếp với bệnh nhân là 1 phần hồ sơ y tế.

## 2. Ai phải trả lời: Communication + Task

📘 `message-response-tracking-and-routing.mdx`: tin nhắn không có "chủ". Muốn biết ai phải trả lời thì tạo 1 `Task` (`focus` = thread, `owner` = bác sĩ, hoặc `performerType` = vai trò để vào hàng đợi chung). Chuyển tin cho bác sĩ khác = đổi `owner`, lịch sử chuyển nằm trong `_history` của Task.

✅ Bot mẫu `examples/medplum-task-demo/src/bots/example/create-respond-to-message-task.ts`: có `Communication` mới từ bệnh nhân → tạo Task "trả lời tin nhắn". Đúng mẫu "việc theo sự kiện" ở [3d](./3d-careplan-task.md#4-task-là-động-cơ-quy-trình).

🧠 Tách "nội dung" (Communication) khỏi "trách nhiệm" (Task) cho phép đo **thời gian phản hồi** (Task tạo lúc nào, đóng lúc nào) mà không đụng vào tin nhắn. Đây là KPI vận hành chính của các dịch vụ chăm sóc từ xa.

## 3. Kênh ngoài: SMS, email, fax

📘 `external-messaging-integration-patterns.mdx`:
- **Gửi ra**: Subscription trên `Communication` mới → Bot gọi Twilio/SendGrid ([2a](./2a-subscriptions.md), [2b](./2b-bots.md)).
- **Nhận vào**: webhook công khai của Bot nhận tin → tạo `Communication`. Ghép vào thread nào? 3 chiến lược: theo mã hội thoại bên ngoài (tốt nhất), theo bệnh nhân + thread đang mở, hoặc mỗi tin 1 thread mới.
- Nhận diện người gửi bằng **tham chiếu có điều kiện** (VD `Patient?telecom=…`). Đây lại là bài toán [3a](./3a-dinh-danh-tiep-nhan.md): SĐT không chuẩn hoá thì không ghép được.

**Fax** (vẫn rất phổ biến ở y tế Mỹ để gửi hồ sơ giữa phòng khám):

📘 `packages/docs/docs/integration/efax/index.md`: tích hợp eFax Corporate, **cần đội Medplum bật** cho project. Các operation `$send-efax`, `$receive-efax`, `$sync-efax-status`. 🧠 Grep `packages/server/src` không thấy các operation này, nên chúng thuộc phần **dịch vụ hosted**, không nằm trong mã nguồn mở.

✅ App mẫu `examples/medplum-efax-demo/src/pages/SendFaxPage.tsx`: upload file → `DocumentReference` → `Organization` người nhận → `Communication` (`category` `fax-direction = outbound`, `payload.contentReference` → DocumentReference) (`:62-95`).

## 4. Nhắn tin là 1 lần khám

📘 `packages/docs/docs/communications/async-encounters/async-encounters.mdx`: trong y tế số, 1 chuỗi SMS hay 1 thread chat có thể là 1 **lần khám không đồng bộ**. Cách mô hình:
1. Định nghĩa "phiên": 1 thread, hoặc mọi tin trong 1 ngày, hoặc bệnh nhân chủ động mở phiên.
2. Tạo `Encounter` cho phiên, gắn `Communication.encounter` của thread header vào.
3. Nhiều bệnh nhân trong 1 phiên (nhóm) → Encounter cha + Encounter con cho từng người.

📘 Docs nói rõ lý do: Encounter có bác sĩ, chẩn đoán, `serviceType`, `reasonCode` đúng là *"critical to billing"* (mục "Handling Multiple-Patient Sessions").

🧠 Ở Mỹ có mã CPT riêng cho "khám qua tin nhắn trên cổng bệnh nhân" (online digital E/M, nhóm 9942x). Muốn thu tiền phải chứng minh tổng thời gian trong 7 ngày, nên **ranh giới phiên** là quyết định doanh thu, không chỉ kỹ thuật. Nối sang [4b](./4b-dinh-gia-ghi-phi.md), [4c](./4c-claim.md).

## Sai lầm thường gặp

1. **Đặt `payload` trên thread header.** Header và tin lẫn nhau khi search.
2. **Sửa nội dung tin đã gửi.** Phải thu hồi + gửi mới.
3. **Không có Task cho tin đến.** Không ai chịu trách nhiệm trả lời, không đo được thời gian phản hồi.
4. **Không gắn Encounter cho phiên nhắn tin có tư vấn y khoa.** Mất căn cứ tính tiền và đo chất lượng.
5. **Ghép tin SMS đến theo SĐT chưa chuẩn hoá.**
