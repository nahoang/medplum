# 3f — Xét nghiệm & chẩn đoán hình ảnh

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"BS Minh chỉ định HbA1c cho bà Lan. Chỉ định đi tới phòng xét nghiệm, kết quả quay về, bác sĩ duyệt. Mỗi bước là resource nào, ai trả tiền xét nghiệm, và ảnh X-quang lưu ở đâu?"*

## Mục lục

- [1. Vòng đời chỉ định → kết quả](#1-vòng-đời-chỉ-định--kết-quả)
- [2. Đặt xét nghiệm qua đối tác: Health Gorilla](#2-đặt-xét-nghiệm-qua-đối-tác-health-gorilla)
- [3. Kết quả: trạng thái và cờ bất thường](#3-kết-quả-trạng-thái-và-cờ-bất-thường)
- [4. Ảnh y khoa: DICOMweb](#4-ảnh-y-khoa-dicomweb)
- [5. Đối chiếu OpenEMR](#5-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vòng đời chỉ định → kết quả

📘 `packages/docs/docs/labs-imaging/results-and-review.md`:

```
ServiceRequest (chỉ định HbA1c, LOINC 4548-4)
   ▲ basedOn
DiagnosticReport (phiếu kết quả)  ──result──▶ Observation (HbA1c = 8.2 %)
   │                                           Observation (…)
   └─ specimen ──▶ Specimen (mẫu máu, giờ lấy)
```

| Bước | Resource | Trạng thái | Ai |
|---|---|---|---|
| Bác sĩ chỉ định | `ServiceRequest` | `draft → active` | Bác sĩ |
| Lấy mẫu | `Specimen` | | Điều dưỡng/lab |
| Lab nhận, làm | (bên lab) | | Lab |
| Kết quả sơ bộ | `DiagnosticReport` + `Observation` | `preliminary` | Lab |
| Kết quả chính thức | | `final` | Lab |
| Sửa sau khi chính thức | | `amended` / `corrected` | Lab |
| Bác sĩ duyệt | `Task` "duyệt kết quả" ([3d](./3d-careplan-task.md#4-task-là-động-cơ-quy-trình)) | `completed` | Bác sĩ |

🧠 1 `ServiceRequest` panel (VD sinh hoá máu) → 1 `DiagnosticReport` → nhiều `Observation`. Phân biệt `amended` và `corrected` quan trọng về pháp lý: `corrected` nghĩa là **kết quả cũ sai**, bác sĩ đã ra quyết định dựa trên nó phải được báo lại.

## 2. Đặt xét nghiệm qua đối tác: Health Gorilla

📘 Health Gorilla là mạng kết nối tới các phòng xét nghiệm lớn ở Mỹ (Quest, Labcorp…). Medplum không tự gửi chỉ định tới lab mà tích hợp qua đối tác này, giống eRx ở [3e](./3e-thuoc-ke-don.md#3-kê-đơn-điện-tử-medplum-không-tự-gửi).

✅ `packages/health-gorilla-core/src/lab-order.ts`:

| Khái niệm nghiệp vụ | Code |
|---|---|
| **AOE** (Ask at Order Entry): câu hỏi lab bắt buộc hỏi lúc chỉ định (VD "nhịn ăn chưa?", "đang mang thai?"). Mỗi xét nghiệm có 1 `Questionnaire` riêng | `aoe.ts`, kiểm tra thiếu câu bắt buộc `lab-order.ts:152-168` |
| **Ai trả tiền xét nghiệm** (`billTo`): `patient`, `insurance`, `customer-account` (phòng khám trả), `self` | `types.ts:38` |
| Trả bằng bảo hiểm thì phải kèm **1–3 `Coverage`** (bảo hiểm chính, phụ, thứ 3) | `lab-order.ts:178-186` |
| Đóng gói thành **1 Bundle `transaction`**: `QuestionnaireResponse` (AOE) + `ServiceRequest` (`draft`, `intent: order`) + `Specimen` nếu đã lấy mẫu | `createLabOrderBundle`, `:217+` |
| Định danh bác sĩ chỉ định bằng **NPI** (mã định danh nhà cung cấp quốc gia Mỹ) | `constants.ts:20` |

🧠 3 điểm nghiệp vụ:
1. **Chỉ định xét nghiệm là quyết định tài chính**: `billTo` phải chọn ngay lúc đặt. Đây là chỗ lâm sàng gặp revenue cycle ([Giai đoạn 4](./4a-quyen-loi-bao-hiem.md)).
2. **Ngược với `$apply`** ([3d](./3d-careplan-task.md#3-apply-không-làm-gì)), đơn xét nghiệm được gói thành 1 transaction, tất cả hoặc không.
3. AOE là 1 `Questionnaire` ([3g](./3g-questionnaire.md)): cùng 1 công cụ phiếu dùng cho tiếp nhận, khám, và chỉ định.

📘 Kết quả trả về qua Health Gorilla có thể sinh `DetectedIssue` cho kết quả **không được yêu cầu** hoặc **không khớp bệnh nhân** (`results-and-review.md`, mục "Critical and Abnormal Results"). Không khớp bệnh nhân chính là bài toán [3a](./3a-dinh-danh-tiep-nhan.md).

Ngoài Health Gorilla, kết quả lab tại chỗ thường về bằng **HL7 v2 ORU** qua Agent ([2c](./2c-agent-hl7v2.md)).

## 3. Kết quả: trạng thái và cờ bất thường

📘 `Observation.interpretation` (`results-and-review.md`): `N` bình thường, `H`/`L` cao/thấp, **`HH`/`LL` nguy kịch**, `A` bất thường.

🧠 Kết quả nguy kịch (VD kali máu 6.8) là nghĩa vụ an toàn người bệnh: phải báo bác sĩ trong thời gian quy định. Medplum không có cơ chế sẵn. Cách làm: Subscription ([2a](./2a-subscriptions.md)) với `criteria` = `Observation?interpretation=HH,LL` → Bot tạo `Task` priority `stat` + gửi tin nhắn ([3h](./3h-giao-tiep.md)). Nhớ rằng Subscription có thể mất sự kiện ([CLAUDE.md, phát hiện 7](../CLAUDE.md#phát-hiện-quan-trọng-đến-nay)), nên cần cron quét lại làm lưới an toàn.

## 4. Ảnh y khoa: DICOMweb

📘 Ảnh y khoa (X-quang, CT, MRI, siêu âm) dùng chuẩn **DICOM**. Mỗi ca chụp = Study → nhiều Series → nhiều Instance (ảnh). Trên web dùng **DICOMweb** gồm 3 dịch vụ: STOW-RS (lưu), QIDO-RS (tìm), WADO-RS (lấy).

✅ `packages/server/src/dicom/`. README tự ghi rõ chỉ hiện thực **1 phần** chuẩn, route chưa làm trả `200` rỗng (`README.md:3-5`):

| Dịch vụ | Đã có | ✅ `routes.ts` |
|---|---|---|
| STOW-RS | `POST /studies`, `POST /studies/{uid}` | `:16`, `:20` |
| QIDO-RS | `GET /studies` (**không lọc** theo tham số), `GET /studies/{uid}/series` | `:17`, `:22` |
| WADO-RS | metadata của series, **1 frame** của 1 instance | `:25`, `:30-33` |
| WADO-URI | Không | — |

✅ STOW-RS (`stow-rs.ts`): mỗi file DICOM được lưu thành `Binary` (dùng chung cơ chế lưu file của Medplum, `:11`, `:196`) và dựng các resource `DicomStudy`, `DicomSeries`, `DicomInstance` (resource riêng của Medplum, không phải `ImagingStudy` của FHIR) (`:4`). Có worker riêng `workers/dicom.ts`. Tích hợp trình xem OHIF.

🧠 Medplum có DICOM ở mức "lưu và xem cơ bản", **không thay được PACS** của bệnh viện. Với phòng khám có siêu âm, X-quang: máy chụp vẫn đẩy về PACS, Medplum chỉ cần `ImagingStudy` + `DiagnosticReport` (kết luận của bác sĩ chẩn đoán hình ảnh) và link sang PACS.

## 5. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Chỉ định | `ServiceRequest` | `procedure_order` (`sql/database.sql:10395`) |
| Phiếu kết quả | `DiagnosticReport` | `procedure_report` (`:10493`) |
| Từng chỉ số | `Observation` | `procedure_result` (`:10519`) |
| Kết nối lab | Health Gorilla (Bot/package), HL7 v2 qua Agent | HL7 v2 trực tiếp tới lab ([2c](./2c-agent-hl7v2.md)) |

🧠 Cấu trúc 3 tầng (chỉ định → phiếu → chỉ số) giống hệt nhau. Đây là cấu trúc của **nghiệp vụ** xét nghiệm, cả HL7 v2 (ORC → OBR → OBX) lẫn FHIR đều phản ánh nó.

## Sai lầm thường gặp

1. **Không gắn `basedOn`** từ DiagnosticReport về ServiceRequest. Không biết chỉ định nào đã có kết quả, chỉ định nào còn treo.
2. **Bỏ qua `corrected`.** Hiển thị kết quả mới mà không báo bác sĩ đã từng dựa trên kết quả sai.
3. **Không hỏi `billTo` lúc chỉ định.** Lab gửi hoá đơn nhầm người.
4. **Tưởng Medplum là PACS.** DICOMweb chỉ có 1 phần.
5. **Tin Subscription cho kết quả nguy kịch** mà không có lưới an toàn.
