# 3d — Care plan & Task: `PlanDefinition/$apply` và hàng đợi việc

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Phác đồ biến thành việc cụ thể cho 1 bệnh nhân thế nào, và việc được giao, nhận, theo dõi ra sao?"* Thiết kế từ đầu: [`3d-greenfield-careplan.md`](./3d-greenfield-careplan.md).
>
> Đường dẫn tính từ `packages/server/src/fhir/operations/` nếu không ghi khác.

## Mục lục

- [1. 5 resource, 2 tầng](#1-5-resource-2-tầng)
- [2. `$apply` làm gì, từng bước](#2-apply-làm-gì-từng-bước)
- [3. `$apply` không làm gì](#3-apply-không-làm-gì)
- [4. Task là động cơ quy trình](#4-task-là-động-cơ-quy-trình)
- [5. Nhận việc an toàn khi 2 người cùng bấm](#5-nhận-việc-an-toàn-khi-2-người-cùng-bấm)
- [6. Đối chiếu OpenEMR](#6-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 5 resource, 2 tầng

| Tầng | Resource | Vai trò |
|---|---|---|
| **Định nghĩa** (dùng chung) | `PlanDefinition` | Phác đồ: danh sách `action` |
| | `ActivityDefinition` | 1 hoạt động dùng lại (xét nghiệm HbA1c) |
| | `Questionnaire` | 1 phiếu dùng lại (đánh giá bàn chân) |
| **Thể hiện** (của 1 bệnh nhân) | `CarePlan` | Kế hoạch của bà Lan, trỏ về phác đồ bằng `instantiatesCanonical` |
| | `RequestGroup` | Nhóm các yêu cầu sinh ra cùng lúc |
| | `Task` | 1 việc cụ thể, có người làm |
| | `ServiceRequest` | Chỉ định y khoa (nếu hoạt động là chỉ định) |

📘 Đây là mẫu "definition → instance" chuẩn của FHIR (nhóm Clinical Reasoning). Lớp quyết định: **HL7**.

## 2. `$apply` làm gì, từng bước

✅ `plandefinitionapply.ts:66-134` (`planDefinitionApplyHandler`). Đầu vào: `subject` (bệnh nhân), tuỳ chọn `encounter`, `practitioner`, `organization` (`:50-55`).

| Bước | ✅ |
|---|---|
| Đọc PlanDefinition, bệnh nhân (chỉ lấy `subject[0]`), encounter, practitioner, organization | `:66-90` |
| Với **mỗi action cấp 1**, gọi `createAction` | `:92-106` |
| Tạo `RequestGroup` (`status: active`, `intent: order`, `instantiatesCanonical` = URL phác đồ) | `:108-119` |
| Tạo `CarePlan` (`status: active`, `intent: plan`, `activity` → RequestGroup) | `:121-131` |
| Trả về CarePlan | `:133` |

`createAction` (`:148-189`) chọn theo loại định nghĩa mà `action.definitionCanonical` trỏ tới:

| Định nghĩa | Sinh ra | ✅ |
|---|---|---|
| `Questionnaire` | `Task` có `input` = Questionnaire (để giao diện mở phiếu) | `:164-172` |
| `ActivityDefinition` `kind = ServiceRequest` | `ServiceRequest` **`status: draft`** + `Task` có `focus` = ServiceRequest | `:247-279` |
| `ActivityDefinition` kind khác | Chỉ `Task` | `:282-283` |
| Không có định nghĩa | Chỉ `Task` | `:186-187` |

Mỗi `Task` (`createTask`, `:300-338`): `status: requested`, `intent: order`, `for` = bệnh nhân, `code.text` = tiêu đề action, `basedOn` = PlanDefinition, `owner`, `performerType`.

**Giao việc** qua extension `https://medplum.com/fhir/StructureDefinition/task-elements` trên ActivityDefinition (`:47`, `:220-245`):
- `owner`: biểu thức FHIRPath, VD `%practitioner` → giao cho bác sĩ được truyền vào `$apply`.
- `performerType`: vai trò (VD điều dưỡng) → vào hàng đợi chung.

**`dynamicValue`** (`:340-362`): điền giá trị vào ServiceRequest bằng FHIRPath. Ngôn ngữ khác `text/fhirpath` (như CQL) bị từ chối (`:351-356`).

🧠 ServiceRequest sinh ra ở trạng thái **nháp** (`draft`), không phải `active`. Nghĩa là `$apply` **đề xuất**, bác sĩ vẫn phải duyệt chỉ định. Đúng với trách nhiệm pháp lý: phác đồ không tự ra y lệnh.

## 3. `$apply` không làm gì

Kiểm tra bằng grep toàn file `plandefinitionapply.ts`: không có chỗ nào đọc các trường sau.

| Trường PlanDefinition (FHIR R4) | Ý nghĩa | Medplum `$apply` |
|---|---|---|
| `action.condition` (`applicability`) | Chỉ áp nếu biểu thức đúng (tuổi > 40) | **Bỏ qua**: mọi action đều sinh Task |
| `action.action` (lồng nhau) | Nhóm hành động | **Bỏ qua**: chỉ duyệt cấp 1 (`:93`) |
| `action.relatedAction` | A xong mới tới B, sau X ngày | **Bỏ qua** |
| `action.timing[x]` | Khi nào làm | **Bỏ qua** |
| `action.selectionBehavior` | Chọn 1 trong N | **Bỏ qua** |
| Giao dịch | Tất cả hoặc không | **Không**: mỗi `createResource` riêng lẻ, lỗi giữa chừng để lại Task mồ côi không có RequestGroup/CarePlan |

📘 Docs `packages/docs/docs/careplans/protocols.md` có ví dụ khai báo `condition`, `relatedAction`, action lồng nhau ở mục "Advanced Workflow Logic", rồi mục "Executing Protocols" chỉ liệt kê 3 kiểu sinh Task. 🧠 Đọc nhanh docs dễ tưởng server đánh giá điều kiện. Phần logic đó phải tự viết bằng Bot ([2b](./2b-bots.md)).

🧠 Hệ quả cho bài tiểu đường: phác đồ có action "Mỡ máu nếu > 40 tuổi" sẽ sinh Task mỡ máu cho **mọi** bệnh nhân. Cách làm đúng trên Medplum: tách phác đồ (PlanDefinition riêng cho nhóm > 40), hoặc Bot áp phác đồ và tự đánh giá điều kiện.

## 4. Task là động cơ quy trình

📘 `packages/docs/docs/careplans/tasks/tasks.md` coi `Task` là đơn vị công việc chung cho mọi quy trình (duyệt kết quả, gọi bệnh nhân, trả lời tin nhắn, xử lý claim bị từ chối).

| Trường | Dùng cho | Ví dụ bà Lan 💭 |
|---|---|---|
| `code` | Loại việc | "Gọi bệnh nhân đi xét nghiệm HbA1c" |
| `status` | Vòng đời chuẩn FHIR: `requested → accepted → in-progress → completed`, hoặc `on-hold`, `cancelled`, `failed`… | `requested` |
| `businessStatus` | **Phễu vận hành riêng** của tổ chức | "Đã gọi lần 1 – không nghe máy" |
| `priority` | Khẩn | `routine` |
| `owner` | Người chịu trách nhiệm hiện tại | Điều dưỡng Hoa |
| `performerType` | Vai trò cần có | Điều dưỡng |
| `focus` | Bản ghi cần xử lý | ServiceRequest HbA1c |
| `for` | Bệnh nhân | bà Lan |
| `restriction.period` | Hạn | Trong 14 ngày |
| `partOf` | Việc cha | — |

📘 *"A queue is a filtered view of Tasks, not a separate FHIR resource"* (`tasks.md`). Hàng đợi = 1 câu search: `Task?status=requested&performer=…&_sort=priority`.

🧠 `businessStatus` là chỗ **chuyên môn tài chính/vận hành** đi vào FHIR: đo tỉ lệ chuyển đổi giữa các bước (gọi → hẹn → làm xét nghiệm), y như phễu bán hàng.

✅ Bot mẫu sinh Task từ sự kiện (`examples/medplum-task-demo/src/bots/example/`):
- `create-review-report-task.ts`: có DiagnosticReport mới → Task "duyệt kết quả" cho bác sĩ.
- `create-respond-to-message-task.ts`: có Communication mới từ bệnh nhân → Task "trả lời tin nhắn".

Đây đúng là V3 bước 3–4 của file G: việc sinh theo **sự kiện**, bằng Subscription + Bot.

## 5. Nhận việc an toàn khi 2 người cùng bấm

✅ `examples/medplum-task-demo/src/components/actions/ClaimTask.tsx:25-34`:

```ts
const ops: PatchOperation[] = [
  { op: 'test', path: '/meta/versionId', value: task.meta?.versionId },
  { op: 'add', path: '/owner', value: createReference(currentUser) },
];
await medplum.patchResource('Task', taskId, ops);
```

JSON Patch `test` trên `versionId`: nếu Task đã bị người khác sửa (versionId đổi), cả patch thất bại. 2 điều dưỡng cùng bấm "Nhận việc" thì chỉ 1 người thành công.

🧠 Đối chiếu với [3b](./3b-dat-lich.md): ở đây **có 1 dòng cụ thể** (Task) để so phiên bản, nên khoá lạc quan theo version là đủ. Đặt lịch không có dòng nào (Slot chưa tồn tại), nên phải dùng SERIALIZABLE. Đây là đối chứng nội bộ đẹp cho [3b-C Phần 8](./3b-comparison-chong-dat-trung.md#phần-8--đối-chứng-trong-chính-medplum).

## 6. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Phác đồ | `PlanDefinition` (dữ liệu FHIR) | **Clinical Decision Rules**: `clinical_rules` (`sql/database.sql:640`), `rule_action` (`:8894`), `rule_reminder` (`:9359`), code ở `src/ClinicalDecisionRules/` |
| Đầu ra | `Task` có người chịu trách nhiệm | **Nhắc nhở**: `patient_reminders` (`:8547`), hiển thị trên hồ sơ / gửi bệnh nhân |
| Điều kiện | Không đánh giá (tự viết Bot) | Có (quy tắc lọc theo tuổi, chẩn đoán, kết quả) |
| Đo chất lượng | `Measure/$evaluate-measure` ([4d](./4d-do-luong-chat-luong.md)) | `src/ClinicalDecisionRules/AMC/` (chỉ số "Meaningful Use" của Mỹ) |

🧠 Ngược nhau: OpenEMR **đánh giá điều kiện tốt** nhưng đầu ra chỉ là **lời nhắc**. Medplum có **đầu ra tốt** (Task, hàng đợi, người chịu trách nhiệm) nhưng **không đánh giá điều kiện**. Sản phẩm thật cần cả 2.

## Sai lầm thường gặp

1. **Tin `$apply` đánh giá `condition`.** Mọi action đều sinh Task.
2. **Dùng action lồng nhau** để nhóm phác đồ. Action con bị bỏ.
3. **Gọi `$apply` không kiểm tra kết quả.** Không nguyên tử. Lỗi giữa chừng để lại Task không có CarePlan.
4. **Coi ServiceRequest `draft` là đã chỉ định.** Lab không nhận được gì cho tới khi chuyển `active`.
5. **Hàng đợi chỉ dùng `status`.** Thiếu `businessStatus` thì không biết việc đang kẹt ở đâu.
6. **Nhận việc bằng `PUT` cả Task** không có If-Match. 2 người cùng nhận, người sau ghi đè.

## Tự kiểm tra

1. Áp phác đồ tiểu đường cho bà Lan sinh ra bao nhiêu resource nếu phác đồ có 4 action (2 ActivityDefinition kind ServiceRequest, 1 Questionnaire, 1 không định nghĩa)?
2. Viết search cho hàng đợi "điều dưỡng ca sáng, việc chưa nhận, quá hạn".
3. Làm sao để Task "xét nghiệm HbA1c" tự đóng khi Observation HbA1c của bà Lan về?
