# 3c — Charting & Encounter: ghi chép 1 lần khám

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Bà Lan vào phòng khám BS Minh. Từ lúc ngồi xuống tới lúc bác sĩ ký hồ sơ, dữ liệu nào được tạo, nằm ở resource nào, và 'ký khoá hồ sơ' nghĩa là gì?"*

## Mục lục

- [1. 2 lớp dữ liệu: theo người và theo lần khám](#1-2-lớp-dữ-liệu-theo-người-và-theo-lần-khám)
- [2. SOAP → FHIR](#2-soap--fhir)
- [3. Vòng đời Encounter](#3-vòng-đời-encounter)
- [4. Ký và khoá hồ sơ](#4-ký-và-khoá-hồ-sơ)
- [5. Tóm tắt bệnh nhân: resource nào vào mục nào](#5-tóm-tắt-bệnh-nhân-resource-nào-vào-mục-nào)
- [6. Đối chiếu OpenEMR](#6-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 2 lớp dữ liệu: theo người và theo lần khám

📘 `packages/docs/docs/charting/chart-data-model.mdx` chia hồ sơ thành:

| Lớp | Resource | Trỏ về | Ví dụ bà Lan 💭 |
|---|---|---|---|
| **Theo người** (sống qua nhiều lần khám) | `Condition` (problem list), `AllergyIntolerance`, `MedicationRequest`/`MedicationStatement`, `Immunization`, `Device` | `subject`/`patient` = Patient | Tiểu đường type 2 (ICD-10 E11), dị ứng Penicillin |
| **Theo lần khám** | `Encounter`, `Observation`, `ClinicalImpression`, `ServiceRequest`, `Condition` (chẩn đoán lần khám) | `encounter` = Encounter **và** `subject` = Patient | HA 150/95, ghi chú "đường huyết chưa kiểm soát", chỉ định HbA1c |

📘 Cùng là `Condition`, phân biệt bằng `category`: `problem-list-item` (bệnh nền) hay `encounter-diagnosis` (chẩn đoán của lần này). 1 chẩn đoán lần khám có thể được "nâng" thành bệnh nền.

🧠 Ý nghĩa thiết kế: resource lâm sàng luôn trỏ **trực tiếp** về Patient, không chỉ qua Encounter. Nhờ vậy:
- "Mọi HbA1c của bà Lan" là 1 search `Observation?subject=…&code=4548-4`, không cần join qua Encounter ([1b](./1b-search.md)).
- Compartment Patient ([1c](./1c-access-control.md)) gom được mọi thứ để phân quyền.

## 2. SOAP → FHIR

📘 `packages/docs/docs/charting/visit-templates.md`: Medplum khuyên tách SOAP thành **dữ liệu có cấu trúc** (S, O, P) và **văn xuôi** (A).

| SOAP | Nghĩa | FHIR | Phân biệt S và O |
|---|---|---|---|
| **S** Subjective | Bệnh nhân kể | `Observation`, `performer` = Patient | `performer` |
| **O** Objective | Đo, khám | `Observation`, `performer` = Practitioner/Device, `category = vital-signs` | |
| **A** Assessment | Nhận định của bác sĩ | `ClinicalImpression` (văn xuôi) + `Condition` (mã ICD-10) | |
| **P** Plan | Chỉ định, kê đơn | `ServiceRequest`, `MedicationRequest`, `CarePlan` | |

**Visit template** = 1 `PlanDefinition` cho mỗi loại lần khám. Mở Encounter thì `$apply` sinh sẵn các `Task` (điền phiếu này, đo chỉ số kia, kê đơn) cho bác sĩ. Cơ chế `$apply` ở [3d](./3d-careplan-task.md).

🧠 Lý do tách A ra văn xuôi: phán đoán lâm sàng khó mã hoá hết, ép bác sĩ chọn mã cho mọi ý thì họ bỏ dùng hệ thống. Phần cần mã (chẩn đoán để thanh toán) vẫn có `Condition`.

## 3. Vòng đời Encounter

📘 FHIR R4 `Encounter.status`: `planned → arrived → triaged → in-progress → onleave → finished`, hoặc `cancelled`, `entered-in-error`.

✅ App mẫu `examples/medplum-provider/src/components/encounter/EncounterHeader.tsx:83-98` cho chuyển `planned → arrived / in-progress / finished / cancelled`, `arrived → in-progress …`. Server **không** ép thứ tự này. FHIR cũng không bắt buộc: `Encounter.statusHistory` ghi lại các lần đổi.

🧠 Quan hệ với lịch hẹn ([3b](./3b-dat-lich.md)):

```
Appointment (booked) ──basedOn/appointment──▶ Encounter (planned → arrived → in-progress → finished)
                                                    ▲
             Observation, ClinicalImpression, ServiceRequest, Condition ── encounter
                                                    │
                                     ChargeItem / Claim (Giai đoạn 4)
```

Encounter là **điểm neo của tiền**: claim được lập theo lần khám ([4c](./4c-claim.md)).

## 4. Ký và khoá hồ sơ

✅ App Provider (`examples/medplum-provider/src/components/encounter/EncounterChart.tsx`) có 3 trạng thái ghi chép (`types/encounter.ts:4-8`): `unsigned`, `signed`, `signedAndLocked`.

| Bước khi bác sĩ bấm "Ký & khoá" | ✅ |
|---|---|
| Đánh dấu mọi `Task` chưa xong của lần khám là `completed` | `EncounterChart.tsx:149-163` |
| `ClinicalImpression.status = completed` | `:165-171` |
| Tạo `Provenance`: `target` = Encounter, `reason` = `SIGN`, `signature.type` = `LA` (legally authenticated), `who` = bác sĩ | `:174-216` |
| Trạng thái "đã khoá" = có Provenance **và** ClinicalImpression `completed` | `:67-76` |
| Khoá = **nút sửa bị vô hiệu trên giao diện** | `:257`, `:266` |
| Sửa sau khi khoá = thêm **addendum** (`DocumentReference`, LOINC `55107-7`) | `SignAddendum.tsx:82-100` |

🧠 3 điểm cần biết:
1. **Khoá chỉ ở giao diện.** Server không biết "đã khoá". Gọi API `PUT Observation/…` vẫn sửa được. Muốn khoá thật phải dùng Bot pre-commit ([2b](./2b-bots.md#5-bot-pre-commit-chặn-hoặc-sửa-dữ-liệu-trước-khi-lưu)) từ chối sửa resource có `encounter` đã ký, hoặc AccessPolicy.
2. **`Provenance.signature` không có chữ ký số.** Trường `data` (chữ ký thật) để trống. Đây là chữ ký **thủ tục** (ai, lúc nào), không chứng minh nội dung không bị sửa sau đó. Chứng minh nội dung dựa vào `_history` ([1f](./1f-history-audit-deletion.md)).
3. **Ký không nguyên tử.** 3 bước trên là 3+ request riêng. Mạng đứt giữa chừng thì Task đã đóng mà chưa có Provenance. Có thể gói vào 1 Bundle `transaction`.

📘 Pháp lý (K3): ở Mỹ, hồ sơ đã ký phải giữ nguyên, chỉ được bổ sung (addendum). Ở Việt Nam, hồ sơ bệnh án điện tử có quy định riêng về chữ ký số (Thông tư 46/2018/TT-BYT về hồ sơ bệnh án điện tử, **cần kiểm tra văn bản hiện hành**). Chữ ký "thủ tục" như app mẫu nhiều khả năng **không đủ**.

## 5. Tóm tắt bệnh nhân: resource nào vào mục nào

`Patient/$summary` (đã nói ở [2d](./2d-interoperability.md)) cho thấy cách Medplum **phân loại** dữ liệu chart thành các mục chuẩn (LOINC section) của bản tóm tắt quốc tế IPS:

✅ `packages/server/src/fhir/operations/patientsummary.ts`:

| Resource | Rơi vào mục | Quy tắc |
|---|---|---|
| `Condition` | Health concerns **hoặc** Problems | `category` có mã LOINC health-concerns → mục đó, còn lại → Problems (`:374-381`) |
| `Observation` | Social history / Functional status / Vital signs / **Results** | Theo `category`: `social-history`, `survey` (khuyết tật → functional), `vital-signs`, còn lại → Results (`:383-404`) |
| `ClinicalImpression` | Notes hoặc Assessments | Mã đầu tiên là LOINC note document → Notes (`:365-372`) |
| `ServiceRequest` | Plan of treatment hoặc Reason for referral | Mã SNOMED `310449005` (giới thiệu) → referral (`:406+`) |

Lọc theo **ngày chăm sóc**, không phải ngày ghi: mỗi loại resource có 1 trường ngày riêng (`Observation.issued`, `Encounter.period.start`, `MedicationRequest.authoredOn`…) (`utils/caredate.ts:10-24`).

🧠 Bài học: `category` không phải trường trang trí. Ghi Observation thiếu `category = vital-signs` thì huyết áp rơi vào mục "Results" của bản tóm tắt gửi đi.

## 6. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Lần khám | `Encounter` | `form_encounter` (`sql/database.sql:2026`) |
| Ghi chép | Nhiều resource FHIR trỏ về Encounter | Bảng `forms` nối tới 38 bảng `form_*` (đa hình, [1a](./1a-fhir-datastore.md)) (`:2464`) |
| Ký | `Provenance` (app mẫu) | `esign_signatures` (`:12564-12576`): `table` + `tid` trỏ dòng được ký, `is_lock`, `amendment`, **`hash` của dữ liệu đã ký** và `signature_hash` |
| Khoá | Giao diện (app mẫu) | Logic khoá trong `library/ESign/` |

🧠 OpenEMR lưu **hash của dữ liệu lúc ký**. Sau này so hash là biết nội dung có bị sửa sau khi ký không. Provenance của app mẫu Medplum không có tương đương. Đây là chỗ hiếm hoi OpenEMR làm kỹ hơn, vì nó là EHR hoàn chỉnh phải qua chứng nhận, còn Medplum để việc này cho ứng dụng.

## Sai lầm thường gặp

1. **Chỉ gắn `encounter`, quên `subject`.** Search theo bệnh nhân bỏ sót.
2. **Dùng 1 `Condition` cho cả chẩn đoán lần khám lẫn bệnh nền.** Sửa bệnh nền làm đổi lịch sử chẩn đoán của lần khám cũ.
3. **Tin rằng "đã khoá" trên app là đã khoá dữ liệu.** Cần pre-commit Bot hoặc policy.
4. **Thiếu `Observation.category`.** Phân mục sai trong bản tóm tắt và trong các màn hình lọc theo category.
5. **Ép mọi phần Assessment thành mã.** Bác sĩ bỏ hệ thống, viết vào ô ghi chú tự do không ai đọc được.

## Tự kiểm tra

1. HA 150/95 của bà Lan nên là 1 Observation hay 2? (Gợi ý: xem "Multi-component Observations" trong `chart-data-model.mdx`.)
2. Viết pseudo-code Bot pre-commit chặn sửa Observation khi Encounter của nó đã có Provenance `SIGN` và ClinicalImpression `completed`.
3. Vì sao app mẫu dùng addendum là `DocumentReference` mới, thay vì sửa `ClinicalImpression`?
