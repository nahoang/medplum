# 1c — Phân quyền & nhiều tenant trong Medplum

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Khi BS Minh đăng nhập, hệ thống quyết định ông được thấy, sửa, xoá gì bằng những dữ liệu nào và ở những điểm chặn nào?"* Bài thiết kế từ đầu cho cùng bài toán: [`1c-greenfield-phan-quyen.md`](./1c-greenfield-phan-quyen.md). So sánh kỹ thuật: [`1c-comparison-phan-quyen.md`](./1c-comparison-phan-quyen.md).
>
> Đường dẫn Medplum tính từ `packages/server/src/` trừ khi ghi khác.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [1. 4 tầng quyết định quyền](#1-4-tầng-quyết-định-quyền)
- [2. Tầng 1 — Project: tenant cứng](#2-tầng-1--project-tenant-cứng)
- [3. Tầng 2 — ProjectMembership: ai, vai trò gì, trong project nào](#3-tầng-2--projectmembership-ai-vai-trò-gì-trong-project-nào)
- [4. Tầng 3 — AccessPolicy: chính sách là dữ liệu](#4-tầng-3--accesspolicy-chính-sách-là-dữ-liệu)
- [5. Tầng 4 — Compartment: tenant mềm gắn lên dữ liệu](#5-tầng-4--compartment-tenant-mềm-gắn-lên-dữ-liệu)
- [6. Lúc đăng nhập: ghép thành 1 chính sách cuối cùng](#6-lúc-đăng-nhập-ghép-thành-1-chính-sách-cuối-cùng)
- [7. Các điểm chặn khi truy cập](#7-các-điểm-chặn-khi-truy-cập)
- [8. Kịch bản đầy đủ: BS Minh, BS Hà, bà Lan](#8-kịch-bản-đầy-đủ-bs-minh-bs-hà-bà-lan)
- [9. Đối chiếu OpenEMR](#9-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa |
|---|---|
| ✅ | Code thật, có `file:line` |
| 📘 | Docs `packages/docs/docs/access/`, định nghĩa resource `packages/definitions/src/fhir/r4/profiles-medplum.json` |
| 💭 | Dữ liệu ví dụ |
| 🧠 | Suy luận |

---

## 1. 4 tầng quyết định quyền

```
 Tầng 1  Project               ranh giới cứng: khác project = không thấy nhau   (cột projectId)
   │
 Tầng 2  ProjectMembership     User X là thành viên project P, profile là Practitioner/minh,
   │                           có admin không, áp những AccessPolicy nào, với tham số gì
   │
 Tầng 3  AccessPolicy          luật theo loại resource: hành động nào, điều kiện (criteria) nào,
   │                           trường nào ẩn/chỉ đọc, ràng buộc khi ghi
   │
 Tầng 4  Compartment           nhãn trên từng resource: thuộc bệnh nhân nào, thuộc tenant mềm nào
                               (meta.compartment → cột compartments UUID[])
```

Kèm theo 2 lớp **cắt bớt**:
- **SMART scopes** khi đăng nhập qua app (`fhir/smart.ts:196`): quyền cuối = chính sách **giao** (∩) scope.
- **IP access rule** (`AccessPolicy.ipAccessRule`, kiểm tra ở `oauth/utils.ts`).

## 2. Tầng 1 — Project: tenant cứng

📘 `Project` là resource riêng của Medplum. Các trường đáng chú ý (`profiles-medplum.json`):

| Trường | Ý nghĩa nghiệp vụ |
|---|---|
| `superAdmin` | Project quản trị toàn server. Không giới hạn project |
| `strictMode` | Bật validate FHIR nghiêm ngặt |
| `checkReferencesOnWrite` | Bắt buộc reference phải trỏ tới resource có thật |
| `features` | Bật tính năng theo project (VD `range-search`, xem [`1a`](./1a-fhir-datastore.md#31-range-column-ngày-số-số-lượng-có-đơn-vị)) |
| `setting` / `secret` / `systemSetting` / `systemSecret` | Cấu hình và bí mật (API key đối tác…) |
| `link[]` | **Project liên kết**: đọc được dữ liệu của project khác |
| `defaultAccessPolicies[]` | Chính sách mặc định theo loại thành viên (`Practitioner`, `Patient`, `Admin`…) |
| `defaultProfile[]` | Profile FHIR mặc định gán cho resource mới |

✅ Mọi query đều lọc `projectId IN (...)` (`fhir/repo.ts:1919-1924`). Danh sách project được phép = project hiện tại + các project trong `link` (`fhir/accesspolicy.ts:104-132`, `getAllowedProjects`). Với project liên kết, chỉ được **đọc** (`fhir/repo.ts:2323-2345`, `canPerformInteraction`).

🧠 Dùng `Project` khi 2 bên là **2 pháp nhân / 2 hợp đồng dữ liệu khác nhau** (VD 2 khách hàng SaaS). `Project.link` dùng cho dữ liệu dùng chung chỉ đọc, VD 1 project chứa bảng mã thuốc dùng cho nhiều khách hàng.

## 3. Tầng 2 — ProjectMembership: ai, vai trò gì, trong project nào

📘 Các trường chính:

| Trường | Ý nghĩa |
|---|---|
| `user` | Tài khoản đăng nhập (`User`, có thể dùng chung nhiều project) |
| `profile` | **Danh tính nghiệp vụ** trong project: `Practitioner`, `Patient`, `RelatedPerson`, `Bot`, `ClientApplication` |
| `admin` | Quản trị project |
| `accessPolicy` | 1 chính sách |
| `access[]` | Nhiều chính sách **có tham số**: `{ policy, parameter[] }` |

🧠 Tách `User` khỏi `profile` là chi tiết domain quan trọng. 1 người thật có thể vừa là bác sĩ ở phòng khám này, vừa là bệnh nhân ở nơi khác. Cùng 1 tài khoản, 2 membership, 2 profile, 2 bộ quyền.

## 4. Tầng 3 — AccessPolicy: chính sách là dữ liệu

📘 Cấu trúc (`profiles-medplum.json`, resource `AccessPolicy`):

| Trường | Ý nghĩa | Ví dụ nghiệp vụ |
|---|---|---|
| `resource[].resourceType` | Loại resource (hoặc `*`) | `Observation` |
| `resource[].interaction` | Hành động được phép: `create`, `read`, `update`, `delete`, `search`, `history`, `vread` | Lễ tân: `Appointment` có `create`, `update`; `Patient` chỉ `read`, `search` |
| `resource[].criteria` | **Câu search FHIR**, resource phải khớp | `Observation?_compartment=%organization` |
| `resource[].hiddenFields` | Trường bị ẩn hoàn toàn | Ẩn `Patient.identifier` với nhân viên marketing |
| `resource[].readonlyFields` | Đọc được, không sửa được | `Patient.birthDate` chỉ sửa ở quầy lễ tân |
| `resource[].writeConstraint` | Biểu thức FHIRPath phải đúng khi ghi, có `%before` và `%after` | "Không được đổi `status` từ `final` về `preliminary`" |
| `ipAccessRule[]` | Cho phép/chặn theo IP | Chỉ truy cập từ mạng phòng khám |
| `compartment`, `resource[].compartment`, `resource[].readonly` | **Đã deprecated**, thay bằng `criteria` và `interaction` | — |

**Mặc định là cấm.** ✅ `packages/core/src/access.ts:121-130` (`satisfiedAccessPolicy`) tìm luật **đầu tiên** khớp. Không luật nào khớp thì không có quyền.

**Tham số (biến `%`)** ✅ `fhir/accesspolicy.ts:259-301`:
- Mỗi phần tử `ProjectMembership.access[]` mang `parameter[]`.
- Hệ thống **tự thêm** 2 biến: `%profile` = profile của người dùng (`:288`), và `%patient` = profile nếu chưa được khai báo (`:289-291`).
- Thay thế bằng **thay chuỗi trên JSON** của policy (`:293-300`): `%x.id` → id, `%x` → reference đầy đủ.

🧠 Nhờ `%patient` mặc định = profile, **cổng bệnh nhân** chỉ cần 1 policy dạng `Observation?subject=%patient`. Mỗi bệnh nhân đăng nhập tự động chỉ thấy dữ liệu của chính mình.

**Ví dụ chính sách theo chi nhánh** ✅ `packages/examples/src/access/multi-tenant-access-policy.ts:286-308`:

```ts
{
  resourceType: 'AccessPolicy',
  name: 'Organization Based Access Policy',
  resource: [
    { resourceType: 'Patient',       criteria: 'Patient?_compartment=%organization' },
    { resourceType: 'Observation',   criteria: 'Observation?_compartment=%organization' },
    { resourceType: 'Encounter',     criteria: 'Encounter?_compartment=%organization' },
    { resourceType: 'Communication', criteria: 'Communication?_compartment=%organization' },
  ],
}
```

## 5. Tầng 4 — Compartment: tenant mềm gắn lên dữ liệu

✅ Khi ghi, `fhir/repo.ts:2061-2101` (`getCompartments`) tính `meta.compartment` từ 3 nguồn:
1. Project (deprecated, đang chuyển sang cột `projectId`).
2. **`meta.accounts`**: các tenant mềm (`Organization`, `HealthcareService`, `CareTeam`…) gán qua `$set-accounts`.
3. **Bệnh nhân** mà resource thuộc về, theo Patient CompartmentDefinition (`fhir/patient.ts:56-76`).

Rồi ghi thành cột `compartments UUID[]` có index GIN ([`1a` mục 2](./1a-fhir-datastore.md#2-bảng-resource-phần-cố-định)). Search `_compartment=X` lọc trên cột này.

**Gán tenant cho bệnh nhân và lan ra hồ sơ** ✅ `fhir/operations/set-accounts.ts`:
- Tham số `propagate` (`:49`, `:69`).
- Chỉ lan khi đích là `Patient` (`:174-176`). Hàm lan dùng `searchPatientCompartment` của `$everything` (`:27`) để tìm mọi resource thuộc bệnh nhân.
- Có chế độ bất đồng bộ `Prefer: respond-async` cho bệnh nhân nhiều dữ liệu (`:94`), chạy ở `workers/set-accounts.ts:102`.

📘 Docs giải thích vì sao **chỉ Patient** lan được (`packages/docs/docs/access/multi-tenant-access-policy.md:378-382`): quan hệ "thuộc về" chỉ được chuẩn định nghĩa cho Patient compartment.

🧠 Đây là cách Medplum giải bài "hồ sơ thừa hưởng phạm vi từ bệnh nhân" ở [`1c-G` Phần 3](./1c-greenfield-phan-quyen.md#phần-3--mô-hình-khái-niệm-). **Chép nhãn lúc ghi** để **không phải JOIN lúc đọc**. Đổi lại, khi chuyển bệnh nhân sang chi nhánh khác phải chạy lại `$set-accounts` với `propagate`.

## 6. Lúc đăng nhập: ghép thành 1 chính sách cuối cùng

✅ `fhir/accesspolicy.ts:139-159` (`getAccessPolicyForLogin`), theo thứ tự:

| Bước | Hàm | Việc làm |
|---|---|---|
| 1 | `reconcileDefaultAccessPolicy` (`:175-191`) | Nếu cờ `admin` vừa đổi mà membership vẫn dùng policy mặc định của vai trò cũ, đổi sang policy mặc định đúng vai trò |
| 2 | `buildAccessPolicy` (`:199-249`) | Gộp `accessPolicy` + mọi `access[]`, thay biến `%`. Luật `readonly` cũ được đổi thành `interaction = search, read, history, vread` (`:224-226`). Thêm quyền đọc `SearchParameter`, `StructureDefinition` (`:312-322`) |
| 3 | `applySmartScopes` (`fhir/smart.ts:196-211`) | Nếu đăng nhập qua app có SMART scope: lấy **giao** chính sách và scope. Scope `patient/` mà không có ngữ cảnh bệnh nhân thì từ chối |
| 4 | `applyProjectAdminAccessPolicy` (`:332-417`) | Super admin: thêm mọi loại resource quản trị. Admin project: được sửa project, thành viên, policy **của project mình**, nhưng `superAdmin`, `systemSecret`, `strictMode` bị ẩn, còn `features`, `link`, `systemSetting` chỉ đọc. Thành viên thường: **không bao giờ** được ghi `AccessPolicy` |

Danh sách loại resource "quản trị" ✅ `packages/core/src/access.ts:21-31`: `AccessPolicy`, `Cron`, `Package`, `PackageRelease`, `PackageInstallation`, `Project`, `ProjectMembership`, `User`, `UserSecurityRequest`. Loại "bảo vệ", chỉ super admin (`:15`): `DomainConfiguration`, `Enterprise`, `JsonWebKey`, `Login`.

## 7. Các điểm chặn khi truy cập

| Thao tác | Điểm chặn | Cách đánh giá `criteria` |
|---|---|---|
| **Search / count / include** | `repo.ts:1932-1999` (`addAccessPolicyFilters`) | **Dịch sang SQL**, ghép vào `WHERE` (xem [`1b` mục 5](./1b-search.md#5-bộ-lọc-luôn-có-tenant-xoá-mềm-phân-quyền)) |
| **Đọc 1 resource** | `repo.ts:636` (`supportsInteraction`), `:649` (`canPerformInteraction`) | **So khớp trong bộ nhớ** (`core/access.ts:139-163`, dùng `matchesSearchRequest`) |
| **Tạo / sửa** | `repo.ts:967` (`checkResourcePermissions`), `:1022` (kiểm tra bản cũ), `:1092` (`isResourceWriteable` kiểm tra bản mới) | Trong bộ nhớ + `writeConstraint` FHIRPath (`repo.ts:2354-2377`) |
| **Xoá** | `repo.ts:1498` | Trong bộ nhớ |
| **Trả kết quả** | `removeHiddenFields` (`repo.ts:2402`) | Xoá các trường `hiddenFields` |
| **Khi ghi** | `restoreReadonlyFields` (`repo.ts:2427`, gọi ở `:1036`) | Ghi đè lại giá trị cũ cho `readonlyFields`, nên client gửi lên giá trị khác cũng không có tác dụng |

🧠 2 chi tiết đáng chú ý:
1. **Sửa được kiểm tra 2 lần**: bản **trước** khi sửa phải thuộc quyền (không sửa được thứ không được thấy), và bản **sau** khi sửa cũng phải thuộc quyền (không "đẩy" được resource ra khỏi phạm vi của mình, hay sang phạm vi người khác).
2. **2 bộ máy đánh giá cùng 1 `criteria`**: SQL cho search, trong bộ nhớ cho đọc/ghi từng resource. Đúng như [`1b` mục 9](./1b-search.md#9-hai-bản-hiện-thực-của-cùng-1-ngôn-ngữ-search). Policy dùng modifier mà 1 trong 2 bộ máy không hỗ trợ có thể cho hành vi lệch giữa search và read.

## 8. Kịch bản đầy đủ: BS Minh, BS Hà, bà Lan

💭 Dữ liệu:

`AccessPolicy/bac-si-chi-nhanh`: như ví dụ mục 4, thêm `Condition` và `MedicationRequest`.

`ProjectMembership` của BS Minh (làm 2 chi nhánh):

```json
{
  "resourceType": "ProjectMembership",
  "profile": { "reference": "Practitioner/minh" },
  "access": [
    { "policy": { "reference": "AccessPolicy/bac-si-chi-nhanh" },
      "parameter": [{ "name": "organization", "valueReference": { "reference": "Organization/cau-giay" } }] },
    { "policy": { "reference": "AccessPolicy/bac-si-chi-nhanh" },
      "parameter": [{ "name": "organization", "valueReference": { "reference": "Organization/quan-3" } }] }
  ]
}
```

Bà Lan sau khi gọi `Patient/lan/$set-accounts` với `accounts=[Organization/cau-giay]`, `propagate=true`:

| Resource | `meta.compartment` (rút gọn) |
|---|---|
| `Patient/lan` | `Organization/cau-giay`, `Patient/lan` |
| `Observation/bp-150` | `Organization/cau-giay`, `Patient/lan` |

| Ai | `Observation?code=8480-6` | Vì sao |
|---|---|---|
| BS Minh | Thấy `bp-150` | Policy sau khi thay biến có `Observation?_compartment=Organization/cau-giay` |
| BS Hà (chỉ Quận 3) | **Không thấy** | `compartments` không chứa `Organization/quan-3` |
| Bà Lan (cổng bệnh nhân, policy `Observation?subject=%patient`) | Thấy | `%patient` = `Patient/lan` |
| App bảo hiểm (SMART scope `user/Claim.rs`) | **Không thấy** | Scope không có `Observation`, giao với policy ra rỗng |

## 9. Đối chiếu OpenEMR

✅ OpenEMR dùng **phpGACL**: bảng `gacl_aro` (`sql/database.sql:2639`) và các bảng `gacl_*` khác, kiểm tra bằng `AclMain::aclCheckCore($section, $value)` (`src/Common/Acl/AclMain.php:170`). Code gọi rải rác ở từng màn hình. Đếm trong repo:

| Lời gọi | Số nơi |
|---|---|
| `aclCheckCore('patients', 'med')` (hồ sơ y tế) | 73 |
| `aclCheckCore('patients', 'demo')` (hành chính) | 40 |
| `aclCheckCore('patients', 'appt')` (lịch hẹn) | 24 |
| `aclCheckCore('patients', 'rx')` (đơn thuốc) | 15 |

Giới hạn theo cơ sở là 1 tuỳ chọn toàn cục `restrict_user_facility` (`library/globals.inc.php:947-952`): *"Restrict non-authorized users to the Schedule Facilities set in User admin."*

| | Medplum | OpenEMR |
|---|---|---|
| Đơn vị quyền | Loại resource × hành động × **điều kiện trên dữ liệu** | Nhóm chức năng (`patients/med`, `patients/demo`…) × mức (xem/sửa) |
| Giới hạn **bệnh nhân nào** | Có (`criteria`, compartment) | Không ở mức ACL. Chỉ có giới hạn theo cơ sở, chủ yếu cho lịch |
| Nơi kiểm tra | 1 tầng (`Repository`) | Rải ở từng màn hình (hơn 150 lời gọi) |
| Thêm vai trò mới | Tạo `AccessPolicy` (dữ liệu) | Cấu hình nhóm phpGACL trong màn hình admin |

🧠 OpenEMR trả lời câu **"bạn được làm loại việc gì"** (RBAC theo chức năng). Medplum trả lời thêm câu **"trên những bản ghi nào"** (ABAC theo thuộc tính dữ liệu). Với 1 phòng khám, câu thứ nhất là đủ. Với chuỗi phòng khám hay SaaS nhiều khách hàng, câu thứ hai là bắt buộc.

## Sai lầm thường gặp

1. **Để membership không có `accessPolicy` và `access`.** Hệ thống giữ hành vi cũ: thêm luật `*`, tức là **toàn quyền** trong project (`accesspolicy.ts:234-238`, *"Preserve legacy behavior of null access policy"*). Luôn gán policy, hoặc cấu hình `Project.defaultAccessPolicies`.
2. **Gán tenant cho bệnh nhân mà quên `propagate=true`.** Patient thuộc chi nhánh, nhưng Observation cũ vẫn không có nhãn, nên bác sĩ chi nhánh thấy bệnh nhân mà không thấy kết quả.
3. **Viết `criteria` bằng chained search sâu** (VD `Observation?subject.organization.partof=…`). Mọi câu search của người đó kéo thêm `EXISTS` nhiều tầng. Nên chép nhãn bằng compartment.
4. **Dùng `hiddenFields` như cơ chế bảo mật duy nhất cho dữ liệu nhạy cảm.** Trường bị ẩn khi trả về, nhưng search theo trường đó vẫn có thể lộ sự tồn tại (VD search `Condition?code=HIV` trả về 1 kết quả). Dữ liệu nhạy cảm nên chặn ở mức **loại resource hoặc `criteria`**.
5. **Nhầm `Project` là chi nhánh.** Tạo 12 project cho 12 chi nhánh thì hội chẩn liên chi nhánh và bệnh nhân khám nhiều nơi trở nên rất khó (lối tắt A ở [`1c-G`](./1c-greenfield-phan-quyen.md#3-lối-tắt-nghe-hợp-lý-nhưng-sai)).

## Tự kiểm tra

1. BS Minh được thêm `readonlyFields: ['status']` cho `Observation`. Ông gửi `PUT` đổi `status` thành `amended`. Kết quả là lỗi hay ghi thành công nhưng `status` giữ nguyên? Dựa vào hàm nào?
2. Viết 1 `writeConstraint` để điều dưỡng không đổi được `Observation.status` từ `final` sang giá trị khác.
3. Bà Lan khám thêm ở Quận 3. Cần gọi gì để BS Hà thấy hồ sơ của bà, mà BS Minh vẫn thấy?
4. Vì sao Medplum không cho thành viên thường ghi `AccessPolicy`, kể cả khi policy của họ có luật `*`?
