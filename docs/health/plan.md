# Kế hoạch đào sâu toàn bộ Medplum — học domain knowledge y tế số qua code thật

> Áp đúng phương pháp đã dùng cho Saleor (xem [`docs/templates/`](../templates/README.md)), chuyển sang domain **y tế số / FHIR**. Mục tiêu không phải thuộc lòng Medplum. Mục tiêu là hiểu **nghiệp vụ y tế** (đặt lịch, kê đơn, xét nghiệm, thanh toán bảo hiểm, tuân thủ pháp lý) và **vì sao một nền tảng thật hiện thực nó theo cách này**. Medplum là bằng chứng code (`file:line`) cho từng kết luận.
>
> File này là **kế hoạch**: nói rõ học gì, theo thứ tự nào, dùng template nào, đọc file nào. Kết quả học được ghi ngay trong thư mục này (`docs/health/`), mỗi module 1 đến 3 file. Tiến độ và bối cảnh để phiên sau làm tiếp nằm ở [`docs/CLAUDE.md`](../CLAUDE.md).

---

## Mục lục

- [1. Khác biệt cốt lõi khi chuyển từ Saleor sang Medplum](#1-khác-biệt-cốt-lõi-khi-chuyển-từ-saleor-sang-medplum)
- [2. Bản đồ project — Medplum gồm những gì](#2-bản-đồ-project--medplum-gồm-những-gì)
- [3. Quy ước chung cho mọi file](#3-quy-ước-chung-cho-mọi-file)
- [4. Lộ trình 6 giai đoạn](#4-lộ-trình-6-giai-đoạn)
  - [Giai đoạn 0 — Nền tảng FHIR](#giai-đoạn-0--nền-tảng-fhir-trước-khi-mở-code)
  - [Giai đoạn 1 — Lõi nền tảng](#giai-đoạn-1--lõi-nền-tảng-platform-core)
  - [Giai đoạn 2 — Tự động hoá & tích hợp](#giai-đoạn-2--tự-động-hoá--tích-hợp)
  - [Giai đoạn 3 — Nghiệp vụ lâm sàng](#giai-đoạn-3--nghiệp-vụ-lâm-sàng-clinical-workflows)
  - [Giai đoạn 4 — Chu trình doanh thu](#giai-đoạn-4--chu-trình-doanh-thu-revenue-cycle)
  - [Giai đoạn 5 — Vận hành & mô hình kinh doanh](#giai-đoạn-5--vận-hành-nền-tảng--mô-hình-kinh-doanh)
  - [Đối chứng OpenEMR theo module](#đối-chứng-openemr-theo-module)
- [5. Lịch trình đề xuất](#5-lịch-trình-đề-xuất)
- [6. Definition of Done cho mỗi file](#6-definition-of-done-cho-mỗi-file)
- [7. Bài mẫu đầu tiên — phác thảo chi tiết module 1a](#7-bài-mẫu-đầu-tiên--phác-thảo-chi-tiết-module-1a)
- [8. Rủi ro của chính kế hoạch này](#8-rủi-ro-của-chính-kế-hoạch-này)

---

## 1. Khác biệt cốt lõi khi chuyển từ Saleor sang Medplum

Đây là phần dễ bị bỏ qua nhất. Bê nguyên template sang mà không chỉnh sẽ cho ra kết luận sai.

| # | Saleor | Medplum | Hệ quả cho cách viết file |
|---|---|---|---|
| K1 | Mô hình dữ liệu do **Saleor tự thiết kế** (Product, Attribute…) | Mô hình dữ liệu do **HL7 FHIR R4 quy định** (Patient, Observation, Claim…). Medplum gần như không tự đặt ra entity nghiệp vụ | Template G phải tách 3 lớp quyết định: **HL7 spec quyết định** / **profile quốc gia (US Core…) quyết định** / **Medplum quyết định**. Câu "vì sao Medplum chọn X" chỉ hợp lệ với lớp thứ 3 |
| K2 | Quyết định thiết kế lớn nằm ở **domain model** (EAV+STI) | Quyết định thiết kế lớn nằm ở **tầng lưu trữ/index/phân quyền**: lưu 1 tài liệu FHIR bất kỳ thế nào để vẫn search nhanh, đúng kiểu, đúng tenant | Template C phát huy mạnh nhất ở Giai đoạn 1 (datastore, search, access). Template G phát huy mạnh nhất ở Giai đoạn 3–4 (nghiệp vụ) |
| K3 | Tuân thủ pháp lý là tuỳ chọn | Tuân thủ pháp lý là **ràng buộc bắt buộc**: HIPAA (audit, minimum necessary), ONC Cures Act (chống chặn thông tin, bulk export), 21 CFR Part 11 | Mỗi bảng đặc tính P1…Pn phải có ít nhất 1 dòng "pháp lý" nếu module chạm dữ liệu bệnh nhân |
| K4 | Django/Python | TypeScript/Node, SQL viết qua builder riêng (`packages/server/src/fhir/sql.ts`), Postgres + Redis | Code minh hoạ 💭 viết bằng TypeScript + SQL, không dùng Django ORM |
| K5 | Single-tenant (1 shop) là mặc định | Multi-tenant SaaS ngay từ đầu (`projectId` trên mọi dòng) | Mọi bảng dữ liệu trong bài phải có cột tenant. Thiếu cột này là "sai lầm thường gặp" có sẵn để phân tích |
| K6 | Stakeholder: founder, vận hành, khách mua | Stakeholder dày hơn hẳn: bệnh nhân, bác sĩ, điều dưỡng, lễ tân, biller, **payer (bảo hiểm)**, nhà thuốc, lab, cơ quan quản lý | Stakeholder map (Template G, Bước 1) phải hỏi thêm câu "bên thứ 3 nào **trả tiền**" và "cơ quan nào **kiểm tra**" |

---

## 2. Bản đồ project — Medplum gồm những gì

✅ grounded: liệt kê từ `ls packages/`, `ls examples/` ở commit `16cb130`.

| Lớp | Package | Vai trò | Domain học được |
|---|---|---|---|
| Chuẩn dữ liệu | `fhirtypes`, `definitions` | Type TypeScript + StructureDefinition/SearchParameter của FHIR R4 | Từ vựng chung của y tế số |
| Thư viện lõi | `core` | Client SDK, FHIRPath, parse search, schema/validation, scheduling utils, pharmacy utils | Ngữ nghĩa FHIR chạy trong code |
| Server | `server` | REST/GraphQL FHIR API, repository, search→SQL, access policy, OAuth/SMART, bots, subscriptions, ~90 operation (`$book`, `$submit`, `$expand`…) | **Trung tâm của kế hoạch** |
| Router | `fhir-router` | Định tuyến FHIR REST, batch/transaction | Ngữ nghĩa Bundle/transaction |
| Chuẩn cũ | `hl7`, `ccda`, `agent` | HL7 v2 (ADT/ORU), C-CDA, agent cài on-prem cầu nối bệnh viện | Vì sao bệnh viện vẫn chạy HL7 v2 |
| UI | `app`, `react`, `react-hooks`, `react-scheduling`, `provider` | Admin app, component y tế, app cho bác sĩ | Workflow người dùng thật |
| Tích hợp bên thứ 3 | `health-gorilla-*`, `dosespot-*`, `scriptsure-react` | Lab, kê đơn điện tử (eRx) | Hệ sinh thái đối tác |
| Hạ tầng | `cdk`, `cli`, `bot-layer`, `/terraform`, `/charts` | Triển khai AWS/K8s | Mô hình self-host vs hosted |
| Ví dụ | `examples/*` (~25 app) | `foomedical`, `medplum-patient-intake-demo`, `medplum-task-demo`, `medplum-eligibility-demo`… | Kịch bản nghiệp vụ hoàn chỉnh, dùng làm bối cảnh Template G |
| Tài liệu | `packages/docs/docs/*` (~37 mục) | Docs chính thức theo domain | Nguồn 📘 thứ hai sau spec FHIR |

---

## 3. Quy ước chung cho mọi file

### Nhãn nguồn (mở rộng thêm 📘 so với template Saleor)

| Nhãn | Nghĩa |
|---|---|
| ✅ grounded | Đọc trực tiếp code Medplum, có `file:line` và commit hash |
| 📘 spec | Trích từ chuẩn bên ngoài: FHIR R4 (hl7.org/fhir/R4), US Core, HL7 v2, quy định (HIPAA, ONC), hoặc docs chính thức `packages/docs/docs/...` |
| 💭 mô phỏng | Tự đóng vai, code phản-thực-tế, bối cảnh giả định |
| 🧠 suy luận | Lập luận "vì sao", không phải lý do Medplum tự công bố |

Không gắn nhãn ✅ cho thứ chỉ đọc trong docs. Docs có thể lệch so với code, nên docs dùng 📘.

### 3 loại file

| Mã | Loại | Tên file | Thứ tự |
|---|---|---|---|
| **R** | Đọc code thật (tương đương `1b` của Saleor) | `docs/health/<mã>-<chủ-đề>.md` (VD `1a-fhir-datastore.md`) | Viết **sau** G, trước C |
| **G** | Greenfield practice | `docs/health/<mã>-greenfield-<chủ-đề>.md` | Viết **đầu tiên**, không mở code thật trước khi xong V1–V3 |
| **C** | So sánh kỹ thuật | `docs/health/<mã>-comparison-<chủ-đề>.md` | Viết **cuối**, cần R làm nguồn ✅ |

Thứ tự **G → R → C** giữ đúng nguyên tắc của template: tự thiết kế trước, mở đáp án sau, để khoảng cách thật không bị che. Riêng Giai đoạn 0 và các module chỉ có R thì đọc thẳng.

### Cách grounding

- Ghi commit hash ở đầu mỗi file R (`git rev-parse --short HEAD`) vì `file:line` trôi theo thời gian.
- Ưu tiên đọc **file test** cạnh file code (`*.test.ts`). Test của Medplum là đặc tả nghiệp vụ dạng chạy được, thường rõ hơn chính code.
- Test trong `packages/core` chạy không cần DB: `npx vitest run packages/core/src/<file>.test.ts`. Test `packages/server` cần Postgres + Redis (`docker compose up`).

---

## 4. Lộ trình 6 giai đoạn

Ký hiệu cột "File": **R** / **G** / **C** như mục 3. ⭐ = module ưu tiên cao (domain dày hoặc quyết định thiết kế đáng học).

### Giai đoạn 0 — Nền tảng FHIR (trước khi mở code)

| Mã | Chủ đề | File | Nguồn |
|---|---|---|---|
| 0a | FHIR căn bản: Resource, Reference, Bundle, CodeableConcept/Coding, Extension, Identifier, `meta` | R (primer, phần lớn 📘) | `packages/docs/docs/fhir-basics.md`, `packages/docs/docs/fhir-datastore/`, spec R4 |
| 0b | Bản đồ resource theo nghiệp vụ: Administrative / Clinical / Financial / Workflow / Terminology | R | Spec R4 "Resource Index"; đối chiếu `packages/fhirtypes/dist` |

Kết quả cần có: 1 bảng "nghiệp vụ → resource FHIR" dùng lại ở mọi giai đoạn sau, tương đương bảng "Cầu nối sang tên kỹ thuật" của Template G.

### Giai đoạn 1 — Lõi nền tảng (platform core)

| Mã | Chủ đề | File | Câu hỏi trung tâm | Grounding chính ✅ |
|---|---|---|---|---|
| ⭐ 1a | **FHIR datastore**: lưu & index resource | R + C | "Lưu hàng trăm loại tài liệu JSON có cấu trúc lồng sâu, mà vẫn search đúng kiểu, nhanh, có lịch sử?" | `fhir/repository/row-builder.ts:130-141`, `fhir/searchparameter.ts:16-21,116-182,205-211`, `fhir/lookups/*.ts`, `fhir/token-column.ts`, `migrations/schema/v1.ts` |
| ⭐ 1b | **Search**: ngữ nghĩa FHIR search sang SQL (modifier, chain, `_include`, `_revinclude`, sort) | R | "`Observation?patient.name=Smith&date=ge2024` thành SQL thế nào?" | `fhir/search.ts`, `fhir/sql.ts`, `core/src/search/` |
| ⭐ 1c | **Multi-tenancy & phân quyền**: Project, ProjectMembership, AccessPolicy, compartment | G + R + C | "Nhiều phòng khám chung 1 server, bác sĩ chỉ thấy bệnh nhân của mình. Mô hình hoá thế nào?" | `fhir/accesspolicy.ts:139,199`, cột `compartments UUID[]` (`migrations/schema/v1.ts:16`), `packages/docs/docs/access/` |
| 1d | **Auth**: OAuth2, SMART on FHIR, ClientApplication, MFA, external IdP, token exchange | R | "App bên thứ 3 truy cập dữ liệu 1 bệnh nhân với đúng phạm vi được cấp thế nào?" | `server/src/auth/`, `server/src/oauth/`, `fhir/smart.ts`, `examples/medplum-smart-on-fhir-demo` |
| ⭐ 1e | **Terminology**: CodeSystem, ValueSet, ConceptMap; SNOMED/LOINC/ICD-10/RxNorm/CPT | R + C | "Lưu, tra cứu, kiểm tra mã y tế (có phân cấp cha-con) thế nào cho nhanh?" | `fhir/operations/expand.ts`, `codesystemlookup.ts`, `subsumes.ts`, `conceptmaptranslate.ts`, `fhir/lookups/coding.ts`, `conceptmapping.ts` |
| 1f | **Version, lịch sử, audit, xoá**: `_history`, AuditEvent, `$expunge`, If-Match | R | "Vì sao hồ sơ y tế gần như không bao giờ xoá thật, và khi buộc phải xoá thì sao?" | bảng `*_History`, `fhir/repo.ts`, `fhir/repo-auditevent.test.ts`, `fhir/operations/expunge.ts`, `packages/docs/docs/compliance/` |

### Giai đoạn 2 — Tự động hoá & tích hợp

| Mã | Chủ đề | File | Câu hỏi trung tâm | Grounding chính ✅ |
|---|---|---|---|---|
| ⭐ 2a | **Subscriptions** (rest-hook, websocket), worker queue | R + C | "Có kết quả xét nghiệm mới thì báo hệ thống khác ngay. Polling, webhook hay hàng đợi?" | `workers/subscription.ts`, `subscription-failure-tracker.ts`, `core/src/subscriptions/`, `examples/medplum-websocket-subscriptions-demo` |
| ⭐ 2b | **Bots**: logic tuỳ biến chạy server-side (`$execute`, `$deploy`), cron | R + C | "Mỗi phòng khám cần logic riêng mà không fork platform. Bots, hooks hay plugin?" | `server/src/bots/`, `fhir/operations/execute.ts`, `deploy.ts`, `workers/cron.ts`, `examples/medplum-demo-bots` |
| 2c | **Agent + HL7 v2**: cầu nối hệ thống bệnh viện on-prem | R | "Vì sao năm 2026 bệnh viện vẫn gửi ADT/ORU qua MLLP, và cloud nhận thế nào?" | `packages/agent`, `packages/hl7`, `server/src/agent/`, `packages/docs/docs/agent/` |
| 2d | **Interoperability theo quy định**: Bulk `$export`, `$everything`, C-CDA, SMART Health Cards/Links | R | "Luật buộc trả dữ liệu cho bệnh nhân/payer. Nền tảng đáp ứng bằng API nào?" | `fhir/operations/export.ts`, `patienteverything.ts`, `ccdaexport.ts`, `smarthealthlinks.ts`, `packages/ccda` |

### Giai đoạn 3 — Nghiệp vụ lâm sàng (clinical workflows)

Giai đoạn chính để học **domain**. Mỗi module trả lời 4 câu: *ai làm gì → resource FHIR nào → operation Medplum nào → app ví dụ nào*.

| Mã | Chủ đề | File | Bối cảnh greenfield gợi ý 💭 (cho Template G) | Grounding chính ✅ |
|---|---|---|---|---|
| ⭐ 3a | **Định danh & tiếp nhận bệnh nhân**: Patient, `$match`, intake form → `$extract` | G + R | *"Cùng 1 bà cụ đăng ký 3 lần với 3 cách viết tên. Làm sao biết là 1 người?"* | `fhir/operations/patientmatch.ts`, `extract.ts`, `fhir/lookups/humanname.ts`, `examples/medplum-patient-intake-demo` |
| ⭐ 3b | **Đặt lịch**: Schedule/Slot/Appointment, `$find`/`$hold`/`$book`/`$confirm`/`$cancel`/`$reschedule`, lịch lặp | G + R + C | *"2 bệnh nhân bấm đặt cùng 1 khung giờ cùng lúc, chỉ 1 người được giữ chỗ."* (tương đương bài Warehouse/Stock bỏ dở bên Saleor) | `fhir/operations/find.ts`, `hold.ts`, `book.ts`, `reschedule.ts` + `*.recurring.test.ts`, `core/src/scheduling.ts`, `packages/react-scheduling` |
| 3c | **Charting & encounter**: Encounter, Observation, Condition, DocumentReference, ClinicalImpression | R | — | `packages/docs/docs/charting/`, `packages/provider`, `fhir/operations/patientsummary.ts` |
| ⭐ 3d | **Care plan & task**: PlanDefinition `$apply`, CarePlan, RequestGroup, Task làm workflow engine | G + R | *"Bệnh nhân tiểu đường cần chuỗi việc: đo HbA1c mỗi 3 tháng, khám mắt hằng năm. Ai giao việc, ai theo dõi?"* | `fhir/operations/plandefinitionapply.ts`, `packages/docs/docs/careplans/`, `examples/medplum-task-demo` |
| 3e | **Thuốc & kê đơn điện tử**: Medication, MedicationRequest, eRx | R | — | `core/src/pharmacy-utils.ts`, `medication-order-utils.ts`, `packages/dosespot-*`, `packages/docs/docs/medications/` |
| 3f | **Xét nghiệm & chẩn đoán hình ảnh**: ServiceRequest → DiagnosticReport/Observation, DICOM | R | — | `packages/health-gorilla-*`, `server/src/dicom/`, `workers/dicom.ts`, `examples/medplum-health-gorilla-demo` |
| 3g | **Questionnaire/form** | R | — | `packages/docs/docs/questionnaires/`, `examples/medplum-questionnaire-hooks` |
| 3h | **Giao tiếp**: Communication, nhắn tin, fax | R | — | `packages/docs/docs/communications/`, `examples/medplum-efax-demo` |
| 3i | **Consent** | R | — | `packages/docs/docs/consent/` |

### Giai đoạn 4 — Chu trình doanh thu (revenue cycle)

Phần sát nhất với chuyên môn tài chính/chiến lược. Học kỹ hơn mức trung bình.

| Mã | Chủ đề | File | Bối cảnh greenfield gợi ý 💭 | Grounding chính ✅ |
|---|---|---|---|---|
| ⭐ 4a | **Bảo hiểm & kiểm tra quyền lợi**: Coverage, CoverageEligibilityRequest/Response | G + R | *"Lễ tân cần biết trước khi khám: bảo hiểm còn hiệu lực không, bệnh nhân tự trả bao nhiêu?"* | `fhir/operations/coverageeligibilityrequestsubmit.ts`, `examples/medplum-eligibility-demo` |
| ⭐ 4b | **Định giá dịch vụ & ghi nhận phí**: ChargeItemDefinition `$apply`, ChargeItem, Account, `$set-accounts` | G + R + C | *"Cùng 1 lần khám, giá khác nhau theo payer, theo hợp đồng, theo mã CPT. Tính phí ở đâu?"* (đối chiếu bài Pricing đã làm bên Saleor) | `fhir/operations/chargeitemdefinitionapply.ts`, `set-accounts.ts`, `workers/set-accounts.ts` |
| ⭐ 4c | **Claim**: Claim `$submit`/`$export`, ClaimResponse, ExplanationOfBenefit | R | — | `fhir/operations/claimsubmit.ts`, `claimexport.ts`, `packages/docs/docs/billing/` |
| 4d | **Đo lường chất lượng**: Measure `$evaluate-measure` (nền của value-based care) | R | — | `fhir/operations/evaluatemeasure.ts` |

Phần "góc nhìn kinh doanh" 📘/🧠 bắt buộc có trong mỗi file Giai đoạn 4: luồng tiền (bệnh nhân, payer, provider), các mã chuẩn (CPT/HCPCS, ICD-10-CM), giao dịch X12 tương ứng (270/271 eligibility, 837 claim, 835 remittance), và điểm rò rỉ doanh thu (denial, under-coding).

### Giai đoạn 5 — Vận hành nền tảng & mô hình kinh doanh

| Mã | Chủ đề | File | Grounding chính ✅ |
|---|---|---|---|
| 5a | Giới hạn & đo lường sử dụng: rate limit, FHIR quota, resource cap | R | `server/src/ratelimit.ts`, `fhir/fhirquota.ts`, `fhir/resource-cap.ts`, `fhir/operations/project-rate-limits.ts` |
| 5b | Self-host vs hosted, sharding, migration DB | R | `packages/cdk`, `/terraform`, `/charts`, `fhir/sharding.ts`, `server/src/migrations/`, `packages/docs/docs/self-hosting/` |
| 5c | Vòng đời project: `$init`, `$clone`, `$archive`, `$restore` | R | `fhir/operations/projectinit.ts`, `projectclone.ts`, `projectarchive.ts`, `projectrestore.ts` |
| 5d | **Tổng hợp chiến lược** 🧠: Medplum đứng ở đâu trong chuỗi giá trị health-tech (headless EHR / developer platform), so với EHR truyền thống và các platform FHIR khác; đòn bẩy của open-core | Bài phân tích, không theo template | Kết quả 1a–4d + docs; mọi nhận định về giá/khách hàng phải 📘 kèm nguồn công khai, không đoán |

---

### Đối chứng OpenEMR theo module

OpenEMR (PHP, phòng khám ngoại trú Mỹ) có phần nghiệp vụ mà Medplum để cho đối tác. Không cần fork: clone đọc trực tiếp khi cần.

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 https://github.com/openemr/openemr /home/user/openemr/openemr
```

Commit đã dùng để grounding: `4c1eb8d`. Đường dẫn bên dưới tính từ gốc repo OpenEMR.

| Module Medplum | Đối chiếu trong OpenEMR | Vì sao đáng xem |
|---|---|---|
| 1a datastore | `sql/database.sql`: `patient_data` (:8350), `form_vitals` (:2422), `lists` (:7685), `forms` (:2464), `lbf_data` (:10262), `shared_attributes` (:12603) | Đủ 4 kỹ thuật lưu trữ (cột cố định, STI, đa hình, EAV) trong 1 schema thật |
| 1c phân quyền | phpGACL `gacl_aro` (:2639), `src/Gacl/` | Mô hình ACL kiểu cũ so với AccessPolicy + compartment |
| 1f audit | `log` (:7772), `api_log` (:92) | Nhật ký dạng bảng log so với `AuditEvent` + `_History` |
| 3a định danh bệnh nhân | `patient_data.dupscore` (:8468), `src/Services/Patient/DuplicatePatientService.php`, `interface/patient_file/merge_patients.php` | Ghép trùng theo điểm so với `$match` |
| 3b đặt lịch | `openemr_postcalendar_events` (:8275), `patient_tracker` (:8574), `src/Appointment/` | 1 bảng sự kiện lịch so với Schedule/Slot/Appointment |
| 3e thuốc | `prescriptions` (:8714), `interface/drugs/`, `src/Rx/` | Kê đơn và kho thuốc tại phòng khám |
| 3f xét nghiệm | `procedure_order` (:10395), `procedure_report` (:10493), `procedure_result` (:10519) | Vòng đời chỉ định → kết quả |
| 4a quyền lợi BH | `src/Billing/EDI270.php`, `interface/billing/edi_271.php`, `insurance_data` (:3310) | Giao dịch X12 270/271 thật |
| 4b định giá | `codes` (:1125), `prices` (:8776), `billing` (:246) | Bảng giá theo mức giá và payer |
| 4c claim | `claims` (:379), `src/Billing/X125010837P.php`, `src/Billing/ParseERA.php`, `ar_session` (:10184), `ar_activity` (:10214) | Toàn bộ vòng 837 → 835 → công nợ |
| 4d chất lượng | `src/Cqm/`, `src/ClinicalDecisionRules/` | Đo lường chất lượng, quy tắc lâm sàng |
| Lớp FHIR | `src/Services/FHIR/*Service.php` | Cách 1 hệ thống "nghiệp vụ trước" dịch sang FHIR |

## 5. Lịch trình đề xuất

Giả định khoảng 6–8 giờ/tuần. Một file template đầy đủ (kiểu bài Saleor ~850 dòng) tốn 4–6 giờ, nên **không viết đủ cả G+R+C cho mọi module**. Chỉ module ⭐ có G/C.

| Tuần | Nội dung | Số file |
|---|---|---|
| 1 | 0a, 0b | 2 R |
| 2–3 | 1a (R + C, làm bài mẫu đầu tiên, xem mục 7), 1b | 2 R, 1 C |
| 4 | 1c (G → R → C) | 1 G, 1 R, 1 C |
| 5 | 1d, 1e (R), 1f | 3 R |
| 6 | 1e (C), 2a, 2b | 2 R, 1 C |
| 7 | 2c, 2d, ôn lại Giai đoạn 1–2 bằng phần "Tự luyện" | 2 R |
| 8 | 3a (G → R) | 1 G, 1 R |
| 9 | 3b (G → R → C) | 1 G, 1 R, 1 C |
| 10 | 3c, 3d (G → R), 3e | 1 G, 3 R |
| 11 | 3f–3i (đọc nhanh, R ngắn) | 4 R ngắn |
| 12 | 4a (G → R), 4b (G) | 2 G, 1 R |
| 13 | 4b (R + C), 4c, 4d | 3 R, 1 C |
| 14 | 5a–5c, 5d tổng hợp | 3 R, 1 bài tổng hợp |

Tổng: khoảng 28 R, 7 G, 5 C, 1 bài tổng hợp. Có thể cắt Giai đoạn 5 hoặc 3f–3i nếu thiếu thời gian, nhưng không cắt Giai đoạn 0 (mọi giai đoạn sau đều dựa vào nó).

---

## 6. Definition of Done cho mỗi file

Rút từ chính 2 template. File chưa đạt đủ checklist thì chưa coi là xong.

- [ ] Đầu file: khối `>` nói rõ file này trả lời câu hỏi nào, khác các file cùng module ở đâu (giống đầu template C)
- [ ] Mục lục có anchor
- [ ] Bảng nhãn nguồn ✅/📘/💭/🧠, và mỗi đoạn đều gắn đúng nhãn
- [ ] Mọi ✅ có `file:line`. File R ghi commit hash ở đầu
- [ ] Có **1 kịch bản dữ liệu xuyên suốt** (VD 1 bệnh nhân + 2 Observation) dùng lại ở mọi bảng
- [ ] Có bảng dữ liệu thật (dòng/cột), không chỉ code
- [ ] Đặc tính bài toán (P1…Pn) có giải thích **vì sao** tồn tại và hậu quả **bằng số** nếu bỏ qua
- [ ] Mục "Sai lầm thường gặp" + "Lối tắt nghe hợp lý nhưng sai" (tự đoán trước khi đọc)
- [ ] **Đối chứng nội bộ**: chỗ nào ngay trong Medplum KHÔNG dùng kỹ thuật này, và vì sao (giống `ShippingMethod` của Saleor)
- [ ] Luyện phản biện 3 kiểu phản đối, đáp án gói trong `<details>`
- [ ] Checklist tái sử dụng + Tự luyện tiếp theo + Liên hệ/Nguồn
- [ ] Riêng module chạm dữ liệu bệnh nhân: ít nhất 1 đặc tính pháp lý (K3)
- [ ] Riêng Template G: tách rõ lớp quyết định HL7 / profile / Medplum (K1)

---

## 7. Bài mẫu đầu tiên — phác thảo chi tiết module 1a

Module 1a được chọn làm bài đầu tiên vì nó **giống hệt cấu trúc bài Saleor** (JSONB vs EAV vs hybrid), nên chuyển template sang ít rủi ro nhất, và vì mọi module sau đều đi qua tầng này.

### Bài toán (P1–P6), nháp 💭/🧠

| # | Đặc tính | Vì sao đòi hỏi |
|---|---|---|
| P1 | ~150 loại resource, cấu trúc lồng sâu, có `extension` tự do | Không thể có 1 bảng "cột cứng" cho mọi field |
| P2 | Search phải đúng **kiểu**: `date=ge2024-01-01` là so sánh khoảng thời gian, `value-quantity=gt5.4\|http://unitsofmeasure.org\|mg` là so sánh số có đơn vị | Giống P3 của Saleor: sai kiểu không báo lỗi, chỉ âm thầm trả sai bệnh nhân. Ở y tế, đây là rủi ro an toàn người bệnh |
| P3 | Trường **mảng có cấu trúc** (nhiều tên, nhiều địa chỉ, nhiều identifier, nhiều coding) phải search được từng phần tử | 1 bệnh nhân có tên khai sinh + tên sau kết hôn |
| P4 | Reference giữa resource + chained search (`Observation?subject:Patient.name=...`) | Câu hỏi lâm sàng luôn đi qua quan hệ |
| P5 | Mỗi lần sửa phải giữ **phiên bản cũ** | Pháp lý (K3): hồ sơ bệnh án phải truy vết được ai sửa gì, khi nào |
| P6 | Admin thêm `SearchParameter` tuỳ biến lúc chạy | Tương đương P1 của Saleor ("không cần deploy") |

### Ứng viên cho Template C

1. **Cột cứng mỗi field**: 1 bảng/resource, 1 cột/field (giống 2.1 Fixed columns)
2. **Document thuần**: chỉ `JSONB` + GIN index (giống 2.4)
3. **EAV tổng quát**: `(resource_id, path, value TEXT)` (giống 2.5)
4. **Toàn bộ qua lookup table**: mỗi search param 1 bảng phụ
5. **Hybrid của Medplum** ✅: `content TEXT` (nguồn sự thật, `row-builder.ts:133-141`) + cột search **phi chuẩn hoá** trên bảng resource, chọn chiến lược theo kiểu (`searchparameter.ts:116-182`):
   - `range-column` cho date/number/quantity (dòng 127-136)
   - `token-column` cho token, có **cột riêng** hoặc **cột dùng chung `__sharedTokens`** (dòng 138-157)
   - `lookup-table` cho mảng có cấu trúc: Address, HumanName, Reference, Coding, ConceptMapping (dòng 205-211)
   - `column` thường cho phần còn lại (dòng 172-182)
   - bảng `*_History` cho P5 (`migrations/schema/v1.ts:26-29`)

### Đối chứng nội bộ cần tìm

- Reference chỉ trỏ 1 loại đích (`singleTargetType`, `searchparameter.ts:175-181`): đi đường `column` đơn giản, không cần lookup table. Đây là bản "ShippingMethod" của Medplum.
- `__sharedTokens` vs cột token riêng: khi nào đáng tốn cột riêng (`hasDedicatedTokenColumns`). Đây là đánh đổi "số cột vs tốc độ" cần đo bằng số giống Phần 7 của template C.

### Câu hỏi mở cần trả lời khi viết R

- Vì sao `content` là `TEXT` chứ không phải `JSONB`? (🧠 giả thuyết: không bao giờ query vào bên trong, chỉ đọc nguyên khối, TEXT tránh chi phí parse/normalize. Cần tìm bằng chứng trong code/commit/docs trước khi gắn ✅)
- Lookup table nào đã bị thay bằng token-column theo thời gian? Xem lịch sử `migrations/schema/v*.ts` để kể "con đường tiến hoá", tương đương mục "Thứ tự xây dựng thực tế" của template G.

---

## 8. Rủi ro của chính kế hoạch này

| Rủi ro | Dấu hiệu | Cách tránh |
|---|---|---|
| Học FHIR spec thay vì học Medplum (sa vào 150 resource) | Tuần 1 kéo sang tuần 3 | Giai đoạn 0 giới hạn 2 file. Resource nào chưa cần thì để tới module dùng nó |
| Gán nhầm quyết định của HL7 thành quyết định của Medplum | Viết "Medplum chọn mô hình Appointment–Slot…" | Áp K1. Trước khi viết "Medplum chọn", tra spec xem đó có phải điều FHIR bắt buộc không |
| `file:line` trôi khi repo cập nhật | Link trỏ sai dòng sau vài tuần | Ghi commit hash. Khi cập nhật file thì grep lại tên hàm |
| Over-engineering chính kế hoạch: viết G+R+C cho mọi module | Tuần 4 mới xong Giai đoạn 1 nửa chừng | Chỉ module ⭐ có G/C, còn lại R ngắn |
| Kết luận kinh doanh (5d) không có nguồn | Câu kiểu "Medplum thu phí theo…" không kèm link | Mọi nhận định thị trường phải 📘 kèm nguồn công khai, nếu không thì 🧠 và ghi rõ là giả thuyết |
