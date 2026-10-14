# 3g — Questionnaire: phiếu, bảng hỏi, thang điểm

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"Phiếu tiếp nhận, phiếu sàng lọc trầm cảm PHQ-9, câu hỏi AOE của lab đều là 'form'. Medplum mô hình hoá form thế nào, logic form chạy ở đâu, và câu trả lời biến thành dữ liệu lâm sàng ra sao?"*

## Mục lục

- [1. 2 resource: mẫu và bài làm](#1-2-resource-mẫu-và-bài-làm)
- [2. Logic form chạy ở client](#2-logic-form-chạy-ở-client)
- [3. Câu trả lời → dữ liệu lâm sàng: 2 cách](#3-câu-trả-lời--dữ-liệu-lâm-sàng-2-cách)
- [4. Form ở khắp nơi trong Medplum](#4-form-ở-khắp-nơi-trong-medplum)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 2 resource: mẫu và bài làm

| Resource | Là gì | Ví dụ |
|---|---|---|
| `Questionnaire` | **Mẫu** form: câu hỏi (`item`), kiểu (`string`, `choice`, `date`, `group`…), bắt buộc, điều kiện hiện | PHQ-9 (9 câu, mỗi câu 0–3 điểm) |
| `QuestionnaireResponse` | **Bài làm** của 1 người, trỏ về mẫu bằng URL `questionnaire` | Bà Lan trả lời PHQ-9 ngày 14/10 |

📘 Lớp quyết định: HL7 (resource) + **SDC IG** (Structured Data Capture: các extension cho điều kiện, tính toán, trích xuất).

🧠 `QuestionnaireResponse` trỏ tới mẫu bằng **URL canonical**, không bằng id. Khi mẫu đổi phiên bản, bài làm cũ vẫn trỏ đúng phiên bản cũ nếu URL có `|version`. Thiếu version thì không biết bài làm cũ trả lời bộ câu hỏi nào.

## 2. Logic form chạy ở client

✅ `packages/react-hooks/src/useQuestionnaireForm/utils.ts`:
- **Hiện/ẩn câu hỏi** theo câu trả lời khác (`enableWhen`, `enableBehavior = any | all`) (`isQuestionEnabled`, `:133+`, `:184-210`).
- **Tính toán** (`calculatedExpression` SDC, FHIRPath): tổng điểm PHQ-9, BMI từ cân nặng + chiều cao (`:62`, `:226+`).
- Chia trang (`itemControl = page`), ví dụ ở `examples/medplum-questionnaire-hooks/src/questionnaire.ts`.

Component giao diện: `packages/react/src/QuestionnaireForm/` (điền), `QuestionnaireBuilder/` (soạn), `QuestionnaireResponseDisplay/` (xem).

🧠 Grep `packages/server/src` không thấy chỗ nào kiểm tra `QuestionnaireResponse` có khớp `Questionnaire` (câu bắt buộc, kiểu dữ liệu, enableWhen). Server lưu bài làm như mọi resource khác, chỉ kiểm tra theo **schema FHIR**. Hệ quả: client tự viết (không dùng component Medplum) có thể gửi bài làm thiếu câu bắt buộc. Nếu cần chắc chắn, kiểm tra lại bằng Bot pre-commit ([2b](./2b-bots.md)). Ngoại lệ: AOE của lab có kiểm tra câu bắt buộc phía package (`packages/health-gorilla-core/src/lab-order.ts:152-168`, [3f](./3f-xet-nghiem-hinh-anh.md)).

## 3. Câu trả lời → dữ liệu lâm sàng: 2 cách

📘 `packages/docs/docs/questionnaires/parsing-questionnaire-responses.md` ("Choosing an Approach"):

| | `$extract` (SDC template) | Subscription + Bot |
|---|---|---|
| Logic nằm ở | Chính `Questionnaire` (extension) | Mã TypeScript của Bot |
| Hợp với | Ánh xạ 1 trường → 1 trường, form đổi thường xuyên | Chấm điểm (PHQ-9), điều kiện, gọi API ngoài |
| Đổi logic | Sửa Questionnaire, không deploy | Deploy lại Bot, giữ Bot và Questionnaire cùng phiên bản |
| Chi tiết | [3a mục 8](./3a-dinh-danh-tiep-nhan.md#8-extract-phiếu--hồ-sơ) | [2b](./2b-bots.md), Bot tiếp nhận ở [3a](./3a-dinh-danh-tiep-nhan.md#bot-tiếp-nhận-mẫu-làm-gì) |

💭 PHQ-9 của bà Lan:
1. Client tính tổng điểm bằng `calculatedExpression` → hiển thị ngay "12 điểm".
2. Bot nhận `QuestionnaireResponse` → **tính lại** tổng điểm (không tin client) → tạo `Observation` LOINC `44261-6` (PHQ-9 total score), `category = survey`.
3. Điểm ≥ 10 → Bot tạo `Task` cho bác sĩ ([3d](./3d-careplan-task.md)).

🧠 Bước 2 là lý do bảng trên khuyên dùng Bot cho thang điểm: điểm số là **dữ liệu lâm sàng có hậu quả** (quyết định điều trị, đo chất lượng [4d](./4d-do-luong-chat-luong.md)), không nên chỉ dựa vào tính toán phía client.

🧠 `category = survey` quyết định Observation rơi vào mục nào trong bản tóm tắt ([3c mục 5](./3c-charting-encounter.md#5-tóm-tắt-bệnh-nhân-resource-nào-vào-mục-nào)).

## 4. Form ở khắp nơi trong Medplum

| Nơi dùng | Vai trò của Questionnaire |
|---|---|
| Tiếp nhận bệnh nhân | Phiếu khai thông tin ([3a](./3a-dinh-danh-tiep-nhan.md)) |
| Visit template | Mỗi bước khám là 1 form, sinh qua `$apply` thành `Task` có input Questionnaire ([3d](./3d-careplan-task.md)) |
| Chỉ định xét nghiệm | AOE ([3f](./3f-xet-nghiem-hinh-anh.md)) |
| Gộp hồ sơ trùng | Người duyệt điền form quyết định ([3a mục 9](./3a-dinh-danh-tiep-nhan.md#9-gộp-hồ-sơ-không-có-trong-server)) |
| Consent | Form đồng ý → `Consent` ([3i](./3i-consent.md)) |

🧠 Questionnaire là **công cụ cấu hình nghiệp vụ không cần code** của Medplum: người vận hành soạn form, Bot hoặc `$extract` biến form thành dữ liệu. Đây là "bàn phím" của nền tảng headless.

## Sai lầm thường gặp

1. **Lưu điểm số chỉ trong `QuestionnaireResponse`.** Không search, không vẽ biểu đồ được. Phải trích ra `Observation`.
2. **Tin điểm do client tính.** Tính lại ở server.
3. **Sửa `Questionnaire` đang dùng mà không tăng version.** Bài làm cũ mất ngữ cảnh.
4. **Đổi `linkId` khi sửa form.** Bot và `$extract` dò theo `linkId`, đổi là trích sai âm thầm.
5. **Tưởng server kiểm tra câu bắt buộc.** Không (mục 2).
