# 5c — Vòng đời project: `$init`, `$clone`, `$archive`, `$restore`

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"1 khách hàng mới (phòng khám, chuỗi phòng khám) được tạo ra trên nền tảng thế nào, và khi họ muốn rời đi hoặc chuyển server thì dữ liệu đi đâu?"*

## Mục lục

- [1. Project = 1 tenant](#1-project--1-tenant)
- [2. `$init`: tạo tenant mới với 4 chính sách mặc định](#2-init-tạo-tenant-mới-với-4-chính-sách-mặc-định)
- [3. `$clone`: nhân bản project](#3-clone-nhân-bản-project)
- [4. `$archive` / `$restore`: mang project sang server khác](#4-archive--restore-mang-project-sang-server-khác)
- [5. Ý nghĩa kinh doanh: chi phí chuyển đổi](#5-ý-nghĩa-kinh-doanh-chi-phí-chuyển-đổi)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 docs · 💭 ví dụ · 🧠 suy luận. Đường dẫn từ `packages/server/src/fhir/operations/`.

---

## 1. Project = 1 tenant

Nhắc lại từ [1c](./1c-access-control.md): mọi dòng dữ liệu có `projectId`, mọi truy vấn bị lọc theo project. 1 `Project` là 1 khách hàng (hoặc 1 môi trường: dev/staging/prod của cùng khách hàng).

| Thao tác | Ai được gọi | Đồng bộ? | ✅ |
|---|---|---|---|
| `$init` | Người dùng đã đăng nhập | Đồng bộ, 1 giao dịch | `projectinit.ts:69` |
| `$clone` | **Chỉ super admin** (quản trị server) | Đồng bộ | `projectclone.ts:18-22` |
| `$archive` | **Admin của chính project đó**, project bật feature `archive`, có quyền đọc toàn bộ | **Bất đồng bộ** (AsyncJob) | `projectarchive.ts:54-72` |
| `$restore` | **Chỉ super admin** | Bất đồng bộ | `projectrestore.ts:35-53` |

## 2. `$init`: tạo tenant mới với 4 chính sách mặc định

✅ `createProject` (`projectinit.ts:109-210`), trong **1 giao dịch** (`:143`, `resourceTypes` ở `:210`):
1. Tạo 4 `AccessPolicy` mặc định:
   - **Patient**: chỉ thấy compartment của mình (`createPatientCompartmentAccessPolicy`, `:214+`)
   - **RelatedPerson**: tương tự (người nhà)
   - **Admin**: toàn quyền (`:251+`)
   - **Practitioner**: đọc/ghi lâm sàng, **chỉ đọc** các resource tri thức (terminology, định nghĩa, cấu hình) (`:266-280+`)
2. Tạo `Project` với `strictMode: true`, `features` = mặc định của server, `defaultPatientAccessPolicy`… gắn 4 policy trên theo `profileType` (`:166-180`).
3. Nếu có admin: tạo `Practitioner` + `ProjectMembership` admin (`:196-206`).

🧠 "Tenant mới có sẵn phân quyền đúng" là quyết định sản phẩm: khách hàng không phải tự viết AccessPolicy để có app bệnh nhân an toàn ngay ngày đầu. Đây là phần "onboarding" của 1 SaaS, viết bằng dữ liệu FHIR.

## 3. `$clone`: nhân bản project

✅ `projectclone.ts`:
- Chỉ super admin (`:20-22`).
- Tham số: `name`, `resourceTypes` (chỉ chép các loại này), `includeIds`, `excludeIds` (`:25-26`, lọc ở `:115-132`).
- Chép resource sang project mới, đổi id.

🧠 Dùng cho: tạo môi trường staging từ production (sau khi loại dữ liệu bệnh nhân bằng `resourceTypes`), hoặc nhân bản "project mẫu" (Questionnaire, PlanDefinition, ChargeItemDefinition, Bot) cho khách hàng mới của 1 công ty phần mềm dựng trên Medplum. Vì chỉ super admin gọi được, trên bản hosted đây là **dịch vụ do Medplum làm**, không phải tự phục vụ.

## 4. `$archive` / `$restore`: mang project sang server khác

✅ `projectarchive.ts`:
- Docblock: *"Exports every version of every resource in the project, preserving ids and meta, so the project can be restored into another Medplum server"* (`:45-48`).
- Điều kiện chặt (`:57-66`): đúng project của người gọi, **không** phải project super admin, người gọi là admin, project bật feature `archive`, và AccessPolicy cho đọc **toàn bộ không điều kiện** (`hasUnrestrictedReadAccess`). Lý do (docblock `:75-78`): *"A partial archive is useless for migration"*.
- Xuất **mọi phiên bản** (không chỉ bản mới nhất), phân trang theo (id, lastUpdated, versionId) (`:132+`, `:167-170`). Tuỳ chọn kèm `AuditEvent`.
- Bỏ qua `AsyncJob`, `BulkDataExport` (`:40`).

✅ `projectrestore.ts`:
- Chỉ super admin (`:36`). Đầu vào là kết quả AsyncJob của `$archive`, mỗi file là 1 `Binary` trên server đích.
- **Giữ nguyên id, versionId, meta** (`:26-28`).
- Chống ghi đè project của tenant khác: chỉ cho restore đè lên project đã tồn tại nếu archive chứa **1 versionId** của chính project đó. versionId là ngẫu nhiên nên không giả mạo được (`checkExistingProject`, `:117-149`).
- Cảnh báo nếu server đích thiếu feature mà project gốc dùng (`:88-95`).

🧠 So với Bulk `$export` ([2d](./2d-interoperability.md)): `$export` xuất **bản hiện tại** để **chia sẻ dữ liệu** (theo luật). `$archive` xuất **toàn bộ lịch sử** để **di chuyển tenant** giữa 2 server Medplum. Lịch sử phiên bản là 1 phần hồ sơ pháp lý ([1f](./1f-history-audit-deletion.md)), nên di chuyển mà mất lịch sử là không chấp nhận được.

## 5. Ý nghĩa kinh doanh: chi phí chuyển đổi

🧠 Với người làm chiến lược, 5c trả lời câu **"khách hàng có bị khoá vào nhà cung cấp không"**:

| Hướng di chuyển | Công cụ | Mất gì |
|---|---|---|
| Medplum hosted → Medplum self-host (hoặc ngược lại) | `$archive` + `$restore` | Gần như không mất (giữ id, phiên bản, meta). Cần super admin bên đích |
| Medplum → server FHIR khác (HAPI…) | Bulk `$export` | Mất lịch sử phiên bản, mất Bot, AccessPolicy, Subscription (là resource riêng của Medplum hoặc mang ngữ nghĩa riêng) |
| Medplum → EHR truyền thống | `$export` + chuyển đổi | Phải ánh xạ FHIR sang schema riêng |

- Dữ liệu lâm sàng: **khoá thấp** (chuẩn FHIR, `$export` theo luật).
- **Logic nghiệp vụ** (Bot, AccessPolicy, cấu hình `$submit`, ChargeItemDefinition…): **khoá cao hơn**, vì nằm trong mô hình riêng của Medplum ([2b](./2b-bots.md)).
- Chuyển giữa hosted và self-host **dễ**: đây vừa là cam kết với khách (không bị khoá vào bản hosted), vừa là đường lên hosted cho người đang self-host (docs mời rõ ở `self-hosting/considerations.md`).

## Sai lầm thường gặp

1. **Dùng `$export` để chuyển tenant.** Mất lịch sử.
2. **Gọi `$archive` bằng tài khoản admin có AccessPolicy hạn chế.** Bị từ chối: archive phải đầy đủ.
3. **Quên bật feature `archive`** trên project.
4. **Tạo project mới rồi tự viết lại AccessPolicy từ đầu.** `$init` đã có 4 chính sách mặc định, nên sửa từ đó.
5. **Clone production sang staging mà không lọc `resourceTypes`.** Mang dữ liệu bệnh nhân thật sang môi trường kém bảo vệ, vi phạm nguyên tắc tối thiểu (HIPAA minimum necessary).
