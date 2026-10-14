# 0b — Bản đồ nghiệp vụ: quy trình → resource FHIR → operation Medplum → bảng OpenEMR

> **Loại file:** R. **Commit Medplum:** `16cb130`. **Commit OpenEMR:** `4c1eb8d`.
>
> Trả lời câu: *"Mỗi việc phòng khám làm hằng ngày nằm ở đâu trong FHIR, trong Medplum, và trong OpenEMR?"* Đây là bảng tra dùng lại ở **mọi** module sau, tương đương bảng "Cầu nối sang tên kỹ thuật" của Template G. Khi đọc 1 module cụ thể, quay lại đây để biết mình đang ở khúc nào của chuỗi giá trị.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [1. Chuỗi giá trị 1 lượt khám](#1-chuỗi-giá-trị-1-lượt-khám)
- [2. Bảng tra theo nghiệp vụ](#2-bảng-tra-theo-nghiệp-vụ)
- [3. Ai là stakeholder của từng khúc](#3-ai-là-stakeholder-của-từng-khúc)
- [4. Quan sát rút ra](#4-quan-sát-rút-ra)
- [5. Khoảng trống: Medplum không có gì](#5-khoảng-trống-medplum-không-có-gì)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa |
|---|---|
| ✅ | Code thật: Medplum `packages/server/src/fhir/routes.ts:<dòng>` hoặc OpenEMR `sql/database.sql:<dòng>` |
| 📘 | Resource FHIR do spec R4 quy định |
| 🧠 | Suy luận |

---

## 1. Chuỗi giá trị 1 lượt khám

🧠 Dựng lại từ các resource FHIR và operation có thật trong Medplum. Tên bước là ngôn ngữ nghiệp vụ.

```
 ① Đăng ký      ② Đặt lịch     ③ Kiểm tra BH   ④ Tiếp đón       ⑤ Khám
 Patient   →    Slot      →    Coverage   →    Encounter   →    Observation
 Identifier     Appointment    Eligibility     (check-in)       Condition
                                                                Procedure
                                                                     │
      ┌──────────────────────────────────────────────────────────────┤
      ▼                         ▼                         ▼          ▼
 ⑥ Chỉ định XN/CĐHA        ⑦ Kê đơn               ⑧ Kế hoạch    ⑨ Ghi phí
 ServiceRequest            MedicationRequest       CarePlan      ChargeItem
   → DiagnosticReport        → (nhà thuốc)         Task            │
   → Observation (kết quả)                                         ▼
                                                    ⑩ Hồ sơ BH → ⑪ Thu tiền
                                                    Claim         ClaimResponse
                                                                  PaymentReconciliation
```

Bước ①–⑧ là **lâm sàng** (dòng thông tin). Bước ⑨–⑪ là **tài chính** (dòng tiền). 🧠 Trong y tế, 2 dòng này chạy **lệch pha**: khám hôm nay, bảo hiểm có thể trả sau nhiều tuần, hoặc từ chối. Ecommerce không có độ lệch này, vì khách trả tiền trước khi nhận hàng.

## 2. Bảng tra theo nghiệp vụ

### A. Hành chính & tiếp nhận

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Hồ sơ bệnh nhân | `Patient`, `RelatedPerson` | CRUD chuẩn. Tên/địa chỉ index qua bảng `HumanName`/`Address` | `patient_data` (`:8350`, 132 cột) |
| Ghép trùng bệnh nhân | `Patient.link` | `/Patient/$match` (`routes.ts:387`) | Điểm trùng lưu sẵn ở cột `patient_data.dupscore` (`:8468`), tính bởi `src/Services/Patient/DuplicatePatientService.php`; gộp bằng `interface/patient_file/merge_patients.php` |
| Nhân viên y tế, cơ sở | `Practitioner`, `PractitionerRole`, `Organization`, `Location` | CRUD chuẩn | `users` (`:9810`), `facility` (`:1849`) |
| Phiếu tiếp nhận/khảo sát | `Questionnaire`, `QuestionnaireResponse` | `/QuestionnaireResponse/$extract` (`routes.ts:410-412`): chuyển câu trả lời thành resource có cấu trúc | `questionnaire_repository` (`:14352`), `questionnaire_response` (`:14377`), form tự cấu hình `layout_options` (`:3661`) + `lbf_data` (`:10262`) |

### B. Đặt lịch

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Lịch làm việc bác sĩ | `Schedule`, `Slot` | CRUD chuẩn | Lịch nằm chung trong `openemr_postcalendar_events` (`:8275`) |
| Tìm giờ trống, giữ chỗ, đặt, xác nhận, huỷ, dời | `Appointment` | `$find`, `$hold`, `$book`, `$confirm`, `$cancel`, `$reschedule` (`routes.ts:433-438`) | `openemr_postcalendar_events` |
| Theo dõi bệnh nhân trong phòng khám (đã đến, đang khám, xong) | `Encounter.status` | CRUD chuẩn | `patient_tracker` (`:8574`) |

### C. Khám & ghi nhận lâm sàng

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Lượt khám | `Encounter` | CRUD chuẩn | `form_encounter` (`:2026`) + `forms` (`:2464`) làm sổ đăng ký các form của lượt khám |
| Dấu hiệu sinh tồn | `Observation` (category `vital-signs`, mã LOINC) | CRUD chuẩn | `form_vitals` (`:2422`), 1 cột cho mỗi chỉ số |
| Chẩn đoán, bệnh nền, dị ứng | `Condition`, `AllergyIntolerance` | CRUD chuẩn | **1 bảng chung** `lists` (`:7685`), phân biệt bằng cột `type` (`medical_problem`, `allergy`, `medication`…) |
| Thủ thuật | `Procedure` | CRUD chuẩn | Ghép từ 2 nguồn: `procedure_order` và `lists` type `surgery` (`src/Services/FHIR/FhirProcedureService.php:76-77`) |
| Ghi chú lâm sàng | `ClinicalImpression`, `DocumentReference`, `Composition` | CRUD chuẩn | `form_clinical_notes` (`:1976`), `pnotes` (`:8686`) |
| Tóm tắt hồ sơ | Bundle tổng hợp | `/Patient/:id/$summary` (`routes.ts:394`), `$everything` (`:390`) | Báo cáo PHP |
| Tiêm chủng | `Immunization` | CRUD chuẩn | `immunizations` (`:3239`) |

### D. Chỉ định & kết quả

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Chỉ định xét nghiệm/chẩn đoán hình ảnh | `ServiceRequest` | CRUD chuẩn. Đối tác: `packages/health-gorilla-*` | `procedure_order` (`:10395`) |
| Kết quả | `DiagnosticReport` + `Observation` | CRUD chuẩn | `procedure_report` (`:10493`), `procedure_result` (`:10519`) |
| Ảnh DICOM | `ImagingStudy` + resource Medplum `DicomStudy/Series/Instance` | `packages/server/src/dicom/` | Không có PACS tích hợp |

### E. Thuốc

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Kê đơn | `MedicationRequest`, `Medication` | CRUD chuẩn. Đối tác eRx: `packages/dosespot-*`, `scriptsure-react` | `prescriptions` (`:8714`), `lists` type `medication` |
| Cấp phát | `MedicationDispense` | CRUD chuẩn | Module `drugs` (`interface/drugs/`) |

### F. Kế hoạch chăm sóc & công việc

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Phác đồ mẫu → kế hoạch cho 1 bệnh nhân | `PlanDefinition` → `CarePlan`/`RequestGroup` | `/PlanDefinition/:id/$apply` (`routes.ts:372`) | `form_care_plan` (`:12832`) |
| Việc cần làm (gọi lại, nhắc tái khám, duyệt kết quả) | `Task` | CRUD chuẩn. Ví dụ: `examples/medplum-task-demo` | Nhắc việc: `src/Reminder/` |
| Gợi ý lâm sàng | CDS Hooks | `/cds-services/` (`app.ts:260`) | `src/ClinicalDecisionRules/` |

### G. Tài chính (revenue cycle)

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Thông tin bảo hiểm | `Coverage` | CRUD chuẩn | `insurance_data` (`:3310`), `insurance_companies` (`:3283`) |
| Kiểm tra quyền lợi | `CoverageEligibilityRequest/Response` | `/CoverageEligibilityRequest/$submit` (`routes.ts:357-358`) | X12 270/271: `src/Billing/EDI270.php`, `interface/billing/edi_271.php` |
| Bảng giá dịch vụ | `ChargeItemDefinition` | `/ChargeItemDefinition/:id/$apply` (`routes.ts:375`) | `codes` (`:1125`) + `prices` (`:8776`) |
| Ghi nhận phí | `ChargeItem`, `Account` | `/:resourceType/:id/$set-accounts` (`routes.ts:381`) | `billing` (`:246`): 1 dòng = 1 mã dịch vụ trong 1 lượt khám |
| Hồ sơ thanh toán BH | `Claim` | `/Claim/$submit`, `/Claim/$export` (`routes.ts:349-354`) | `claims` (`:379`), X12 837: `src/Billing/X125010837P.php` |
| Kết quả thanh toán, đối soát | `ClaimResponse`, `ExplanationOfBenefit`, `PaymentReconciliation` | CRUD chuẩn | ERA 835: `src/Billing/ParseERA.php`; công nợ `ar_session` (`:10184`), `ar_activity` (`:10214`), `payments` (`:8611`) |
| Đo lường chất lượng | `Measure`, `MeasureReport` | `/Measure/:id/$evaluate-measure` (`routes.ts:369`) | `src/Cqm/` |

### H. Liên thông & quyền dữ liệu của bệnh nhân

| Nghiệp vụ | Resource FHIR 📘 | Medplum ✅ | OpenEMR ✅ |
|---|---|---|---|
| Xuất dữ liệu hàng loạt | Bulk Data `$export` | `/$export`, `/Patient/$export`, `/Group/:id/$export` (`routes.ts:246,361,365`) | `FhirExportJobService.php` |
| Tài liệu tóm tắt chuẩn C-CDA | — | `/Patient/:id/$ccda-export` (`routes.ts:398`) | `ccr/`, `src/Cda/` |
| Chia sẻ hồ sơ qua QR/link | SMART Health Cards/Links | `routes.ts:402-407` | — |
| Kết nối HL7 v2 với bệnh viện | — | `packages/agent`, `/Agent/$status`… (`routes.ts:311-331`) | — |

### I. Nền tảng (không phải nghiệp vụ y tế nhưng bắt buộc có)

| Nghiệp vụ | Medplum ✅ | OpenEMR ✅ |
|---|---|---|
| Tenant | `Project`, `/Project/$init`, `$clone`, `$archive` (`routes.ts:250-260`) | 1 cài đặt = 1 tổ chức, multisite tách DB (`sites/<tên>/sqlconf.php`) |
| Phân quyền | `AccessPolicy`, `ProjectMembership`, compartment | phpGACL: `gacl_aro` (`:2639`) |
| Nhật ký truy cập | `AuditEvent` | `log` (`:7772`), `api_log` (`:92`) |
| Logic tuỳ biến | `Bot` + `/Bot/:id/$deploy` (`routes.ts:346`) | Sửa code PHP, module `interface/modules/` |
| Xoá vĩnh viễn | `/:resourceType/:id/$expunge` (`routes.ts:415`) | Xoá mềm qua cột `activity`/`deleted` |

## 3. Ai là stakeholder của từng khúc

🧠 Áp câu hỏi Stakeholder map của Template G vào chuỗi ở mục 1:

| Khúc | Thao tác hằng ngày | Nhận kết quả | Trả tiền / kiểm tra | Phát hiện nếu sai |
|---|---|---|---|---|
| ①② Đăng ký, đặt lịch | Lễ tân | Bệnh nhân | — | Bệnh nhân (đến không có lịch) |
| ③ Kiểm tra BH | Lễ tân / biller | Phòng khám | **Bảo hiểm** | Biller (bị từ chối sau này) |
| ④⑤ Khám | Bác sĩ, điều dưỡng | Bệnh nhân | — | Bác sĩ lượt sau, **cơ quan thanh tra** |
| ⑥ Xét nghiệm | Bác sĩ chỉ định, **lab** thực hiện | Bác sĩ | Bảo hiểm | Lab (mẫu không khớp chỉ định) |
| ⑦ Thuốc | Bác sĩ kê, **nhà thuốc** cấp | Bệnh nhân | Bảo hiểm | Dược sĩ (tương tác thuốc) |
| ⑨–⑪ Tài chính | Biller / kế toán | Phòng khám | **Bảo hiểm**, bệnh nhân (đồng chi trả) | Kế toán (công nợ không khớp) |

Cột "Trả tiền / kiểm tra" chính là khác biệt K6 trong [kế hoạch](./plan.md#1-khác-biệt-cốt-lõi-khi-chuyển-từ-saleor-sang-medplum). Ecommerce gần như không có cột này.

## 4. Quan sát rút ra

🧠 Nhìn 2 cột Medplum và OpenEMR cạnh nhau:

1. **Khúc lâm sàng (A–F): Medplum chủ yếu là "CRUD chuẩn".** Logic nghiệp vụ nằm ở cách dùng resource (do spec FHIR định nghĩa), không nằm trong code server. Muốn học quy tắc lâm sàng phải đọc spec, docs `packages/docs/docs/<domain>/` và `examples/`, không phải `packages/server`.
2. **Ngoại lệ là đặt lịch** (`$find`/`$hold`/`$book`…, 6 operation riêng). 🧠 Đặt lịch có bài toán **tranh chấp đồng thời** (2 người cùng giữ 1 slot), mà CRUD chuẩn không giải được. Đây là module có logic server dày nhất ở khúc lâm sàng, đáng làm Template G + C (module 3b).
3. **Khúc tài chính (G): OpenEMR dày hơn Medplum rất nhiều.** Medplum có operation khung (`$submit`) và để đối tác xử lý phần còn lại. OpenEMR tự sinh file X12 837 và đọc ERA 835. Muốn hiểu revenue cycle ở mức quy tắc, nguồn chính phải là OpenEMR.
4. **OpenEMR có đủ 4 kỹ thuật lưu trữ trong cùng 1 schema:**
   - cột cố định: `patient_data`, `form_vitals`
   - STI: `lists.type`
   - liên kết đa hình: `forms.formdir` + `form_id` trỏ sang nhiều bảng `form_*`, có 38 bảng `form_*`
   - EAV: `lbf_data`, `shared_attributes`

   Đây là kho đối chứng tuyệt vời cho file so sánh [`1a`](./1a-fhir-datastore.md).

## 5. Khoảng trống: Medplum không có gì

Ghi lại để không tưởng nhầm "đọc hết Medplum là đủ":

| Thiếu | Lấy từ đâu |
|---|---|
| Quy tắc thanh toán BHYT Việt Nam | Chuẩn dữ liệu XML BHYT của Bộ Y tế (cần kiểm tra văn bản hiện hành) |
| Quy tắc lâm sàng (phác đồ, ngưỡng cảnh báo) | Hướng dẫn chẩn đoán điều trị của Bộ Y tế, CDS Hooks mẫu |
| Danh mục dùng chung (thuốc, dịch vụ kỹ thuật) của Việt Nam | Danh mục của Bộ Y tế / BHXH |
| Vận hành thật ở quầy lễ tân, phòng kế toán | Phỏng vấn người làm thật |
| Revenue cycle Mỹ ở mức quy tắc | OpenEMR `src/Billing/` |
