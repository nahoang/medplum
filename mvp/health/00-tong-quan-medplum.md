# 00 — Tổng quan Medplum: bản đồ để không lạc khi đào sâu

> **Loại file:** R (đọc code thật). **Commit Medplum:** `16cb130`. **Commit OpenEMR đối chứng:** `4c1eb8d`.
>
> File này trả lời đúng 1 câu: *"Medplum gồm những khối nào, 1 request đi qua đâu, và mỗi khối dạy mình phần domain nào?"* Không đào sâu khối nào cả. Đào sâu nằm ở các file `0a`, `0b`, `1a`… Đọc file này trước để biết file sau đang đứng ở đâu trên bản đồ.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [1. Medplum là gì — và KHÔNG là gì](#1-medplum-là-gì--và-không-là-gì)
- [2. Các khối trong monorepo](#2-các-khối-trong-monorepo)
- [3. Đường đi của 1 request ghi dữ liệu](#3-đường-đi-của-1-request-ghi-dữ-liệu)
- [4. Các nhóm API](#4-các-nhóm-api)
- [5. Đối chứng: OpenEMR khác Medplum ở tầng kiến trúc ra sao](#5-đối-chứng-openemr-khác-medplum-ở-tầng-kiến-trúc-ra-sao)
- [6. Đọc file nào tiếp](#6-đọc-file-nào-tiếp)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa |
|---|---|
| ✅ grounded | Đọc trực tiếp code, có `file:line` |
| 📘 spec | Chuẩn FHIR/HL7, quy định, hoặc docs chính thức `packages/docs/docs/...` |
| 💭 mô phỏng | Ví dụ tự dựng |
| 🧠 suy luận | Lập luận, không phải điều Medplum tự công bố |

---

## 1. Medplum là gì — và KHÔNG là gì

**Là:** 1 nền tảng backend lưu và phục vụ dữ liệu y tế theo chuẩn **FHIR R4**, kèm auth, phân quyền, tự động hoá (bot, subscription), và bộ component React. Người dùng của Medplum là **đội kỹ thuật** đang xây sản phẩm y tế, không phải phòng khám. 📘 `packages/docs/docs/fhir-basics.md`

**Không là:** 1 phần mềm EHR dùng ngay cho phòng khám. Không có sẵn màn hình viện phí, không có quy trình khám bệnh dựng sẵn. App `packages/provider` và các `examples/*` là bản mẫu, không phải sản phẩm hoàn chỉnh.

🧠 Hệ quả cho việc học: Medplum dạy rất tốt **"dữ liệu y tế có hình dạng gì và được lưu, tìm, bảo vệ ra sao"**. Nó dạy ít **"phòng khám vận hành ra sao"**. Phần sau phải lấy từ OpenEMR, docs, và phỏng vấn người làm thật. Xem đánh giá đầy đủ ở [`CLAUDE.md`](./CLAUDE.md#đánh-giá-nguồn-học-đã-chốt).

## 2. Các khối trong monorepo

✅ `ls packages/` tại `16cb130`.

| Tầng | Package | Vai trò | Domain học được |
|---|---|---|---|
| Từ vựng | `fhirtypes`, `definitions` | Type TypeScript và StructureDefinition/SearchParameter của FHIR R4 | Tên và hình dạng của mọi khái niệm y tế |
| Lõi dùng chung | `core` | Client SDK, FHIRPath, parse search, validate schema, tiện ích scheduling/pharmacy | Ngữ nghĩa FHIR chạy được |
| Server | `server` | API FHIR, lưu trữ, search→SQL, phân quyền, OAuth/SMART, bot, subscription, khoảng 90 operation | **Trung tâm** |
| Định tuyến | `fhir-router` | Định tuyến REST, batch/transaction | Ngữ nghĩa Bundle |
| Chuẩn cũ | `hl7`, `ccda`, `agent` | HL7 v2, C-CDA, agent cài tại bệnh viện | Bệnh viện thật trao đổi dữ liệu thế nào |
| UI | `app`, `react`, `react-hooks`, `react-scheduling`, `provider` | Admin app, component y tế, app bác sĩ | Màn hình người dùng thật |
| Đối tác | `health-gorilla-*`, `dosespot-*`, `scriptsure-react` | Xét nghiệm, kê đơn điện tử | Ai là bên thứ 3 trong hệ sinh thái |
| Hạ tầng | `cdk`, `cli`, `bot-layer`, `/terraform`, `/charts` | Triển khai | Self-host vs hosted |
| Ví dụ | `examples/*` | Khoảng 25 app mẫu | Kịch bản nghiệp vụ trọn vẹn |

## 3. Đường đi của 1 request ghi dữ liệu

Ví dụ: app gửi `PUT /fhir/R4/Patient/123`. ✅

```
HTTP request
  │
  ├─ packages/server/src/app.ts:200-250      middleware: header, CORS, rate limit, parse body
  ├─ packages/server/src/app.ts:262          /fhir/R4/ → fhirRouter
  ├─ packages/server/src/fhir/routes.ts:186  authenticateRequest (token OAuth → biết ai đang gọi)
  ├─ packages/server/src/fhir/routes.ts:225+ FhirRouter: CRUD chuẩn + các operation $xxx
  │
  └─ Repository.updateResource  (packages/server/src/fhir/repo.ts:941)
       └─ updateResourceImpl    (repo.ts:997)
            ├─ checkResourcePermissions   AccessPolicy cho phép không?
            ├─ preCommitValidation        repo.ts:1009  hook tuỳ biến trước khi ghi
            ├─ If-Match                   repo.ts:1026  chống ghi đè (optimistic locking)
            ├─ meta mới                   repo.ts:1040-1060  versionId mới, author, project, compartment
            ├─ validateRepositoryResource repo.ts:1063  kiểm tra đúng schema/profile FHIR
            ├─ isNotModified              repo.ts:1087  không đổi gì thì không tạo version mới
            ├─ handleStorage              repo.ts:1097 → 1164
            │    └─ writeToDatabase       repo.ts:1197  TRONG 1 TRANSACTION:
            │         ├─ writeResource          bảng "Patient"          (bản hiện tại + cột search)
            │         ├─ writeResourceVersion   bảng "Patient_History"  (lưu vết mọi phiên bản)
            │         └─ writeLookupTables      bảng HumanName, Address, Patient_References…
            └─ addBackgroundJobs          repo.ts:1102  sau commit: kích hoạt Subscription/Bot
```

🧠 3 điều domain đọc được ngay từ sơ đồ này:

1. **Không có "sửa đè"**. Mọi lần ghi đều sinh `versionId` mới và 1 dòng `_History`. Đây là yêu cầu pháp lý của hồ sơ y tế: phải truy vết ai sửa gì, khi nào.
2. **Phân quyền nằm trong tầng lưu trữ**, không nằm ở controller. Mọi đường ghi, kể cả từ bot, đều đi qua cùng 1 cửa kiểm tra.
3. **Tự động hoá chạy sau commit**. Dữ liệu được lưu chắc chắn trước, rồi mới báo cho hệ thống khác.

## 4. Các nhóm API

✅ `packages/server/src/app.ts:257-282`

| Đường dẫn | Mục đích |
|---|---|
| `/fhir/R4/` | API chính: CRUD, search, history, batch, khoảng 90 operation |
| `/auth/`, `/oauth2/` | Đăng nhập, MFA, OAuth2, SMART on FHIR |
| `/admin/` | Quản trị project, thành viên |
| `/scim/v2/` | Đồng bộ người dùng từ hệ thống doanh nghiệp |
| `/fhircast/` | Đồng bộ ngữ cảnh giữa nhiều app đang mở cùng lúc (VD 2 màn hình cùng hiện 1 bệnh nhân) |
| `/cds-services/` | CDS Hooks: gợi ý lâm sàng chèn vào workflow |
| `/shl/` | SMART Health Links: chia sẻ hồ sơ qua link/QR |
| `/storage/`, `/webhook/`, `/keyvalue/v1/`, `/email/v1/` | Hạ tầng phụ trợ |

Danh sách operation `$xxx` theo nghiệp vụ nằm ở [`0b`](./0b-ban-do-nghiep-vu-resource.md).

Mọi đường dẫn đều có thể nằm sau `/projects/:projectId/` (`app.ts:279-280`). Ngay từ URL đã thấy Medplum là hệ thống **nhiều tenant**.

## 5. Đối chứng: OpenEMR khác Medplum ở tầng kiến trúc ra sao

✅ OpenEMR tại `4c1eb8d`, clone ở `/home/user/openemr/openemr`.

| Khía cạnh | Medplum | OpenEMR |
|---|---|---|
| Nguồn sự thật | Tài liệu FHIR JSON (`content TEXT`) | Bảng quan hệ tự thiết kế: `patient_data` (132 cột, `sql/database.sql:8350`), `form_vitals`, `lists`… |
| FHIR | Lưu trữ gốc là FHIR | **Lớp dịch** tại thời điểm đọc: `src/Services/FHIR/*Service.php` chuyển bảng quan hệ sang FHIR. VD `FhirObservationVitalsService.php:95` gán mã LOINC `8480-6` cho cột `bps` |
| Người dùng | Đội dev xây sản phẩm | Phòng khám dùng ngay |
| Nghiệp vụ viện phí | Chỉ có operation khung (`Claim/$submit`) | Đầy đủ: `src/Billing/X125010837P.php`, `EDI270.php`, `ParseERA.php` |
| Tenant | Nhiều project trên 1 server | 1 cài đặt cho 1 tổ chức (có "sites" nhưng tách DB) |

🧠 Đây là 2 triết lý đối lập. **Medplum: "chuẩn trước, nghiệp vụ tự xây".** **OpenEMR: "nghiệp vụ trước, chuẩn là lớp xuất khẩu".** Một dự án MVP phải chọn đứng gần phía nào. Đây là câu hỏi chiến lược, không chỉ kỹ thuật.

## 6. Đọc file nào tiếp

1. [`0a-fhir-primer.md`](./0a-fhir-primer.md): 8 khái niệm FHIR phải biết trước khi đọc tiếp
2. [`0b-ban-do-nghiep-vu-resource.md`](./0b-ban-do-nghiep-vu-resource.md): nghiệp vụ → resource → operation → bảng OpenEMR
3. [`1a-fhir-datastore.md`](./1a-fhir-datastore.md): Medplum lưu 1 tài liệu y tế xuống Postgres thế nào
