# 1c-C — So sánh 6 cách phân quyền dữ liệu y tế

> **Loại file:** C. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Bộ 3 file của module 1c:
> - [`1c-greenfield-phan-quyen.md`](./1c-greenfield-phan-quyen.md): con đường discovery → V1/V2/V3
> - [`1c-access-control.md`](./1c-access-control.md): Medplum làm gì (nguồn ✅)
> - **File này**: 6 kỹ thuật ngang hàng, cùng kịch bản, vì sao chọn cái nào. 3/6 ứng viên là hệ thống thật.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bài toán: 7 đặc tính](#phần-1--bài-toán-7-đặc-tính)
- [Phần 2 — 6 ứng viên](#phần-2--6-ứng-viên)
- [Phần 3 — Bảng so sánh trực diện](#phần-3--bảng-so-sánh-trực-diện)
- [Phần 4 — Vì sao loại](#phần-4--vì-sao-loại)
- [Phần 5 — 4 lối tắt nghe hợp lý nhưng sai](#phần-5--4-lối-tắt-nghe-hợp-lý-nhưng-sai)
- [Phần 6 — Medplum tại từng ngã rẽ](#phần-6--medplum-tại-từng-ngã-rẽ)
- [Phần 7 — Ma trận đánh đổi](#phần-7--ma-trận-đánh-đổi)
- [Phần 8 — Đối chứng nội bộ](#phần-8--đối-chứng-nội-bộ)
- [Phần 9 — Luyện phản biện](#phần-9--luyện-phản-biện)
- [Phần 10 — Checklist](#phần-10--checklist)

## Quy tắc nhãn nguồn

| Nhãn | Dùng ở đâu |
|---|---|
| ✅ code thật | 2.1 OpenEMR, 2.5 HAPI, 2.6 Medplum, Phần 6, 8 |
| 📘 docs/chuẩn | 2.3 (Postgres RLS), Phần 1 |
| 💭 phản thực tế | 2.2, 2.3, 2.4, số liệu ước tính |
| 🧠 suy luận | Phần 4, 5, 7, 9 |

**Kịch bản cố định:** chuỗi An Bình có 12 chi nhánh. BS Minh làm ở Cầu Giấy và Quận 3. BS Hà chỉ làm ở Quận 3. Bà Lan khám ở Cầu Giấy, có huyết áp 150. Lễ tân Hoa làm ở Cầu Giấy. App bảo hiểm XYZ cần xem `Claim`.

---

## Phần 1 — Bài toán: 7 đặc tính

| # | Đặc tính | Vì sao |
|---|---|---|
| A1 | Lọc theo **bản ghi** (bệnh nhân nào), không chỉ theo **chức năng** | Bác sĩ chi nhánh A không được thấy bệnh nhân chi nhánh B |
| A2 | 1 người nhiều phạm vi, 1 bệnh nhân nhiều phạm vi | Bác sĩ làm nhiều nơi, bệnh nhân khám nhiều nơi |
| A3 | Theo loại dữ liệu × hành động × trường | Lễ tân xem lịch, không xem chẩn đoán. Không ai sửa được kết quả đã `final` |
| A4 | Không thể "quên" áp dụng | 1 API quên điều kiện = lộ dữ liệu toàn chuỗi |
| A5 | Vận hành tự thêm vai trò/chi nhánh, không deploy | Mở chi nhánh 13 không chờ dev |
| A6 | Ứng dụng bên thứ 3 bị giới hạn bởi cả người uỷ quyền và phạm vi app | Chuẩn SMART on FHIR, quy định chia sẻ dữ liệu 📘 |
| A7 | Nhanh ở quy mô lớn | Phân quyền chạy trên **mọi** câu truy vấn |

**Hậu quả bằng số (💭):** 12 chi nhánh × 4.000 bệnh nhân = 48.000 bệnh nhân, 200 nhân viên. Lỗ hổng A4 ở **1** API danh sách bệnh nhân là 200 người thấy toàn bộ 48.000 hồ sơ, thay vì khoảng 4.000 mỗi người. Theo luật bảo vệ dữ liệu cá nhân, đó là sự cố lộ dữ liệu sức khoẻ của **44.000 người**.

---

## Phần 2 — 6 ứng viên

### 2.1 RBAC theo chức năng — OpenEMR phpGACL ✅

**Ý tưởng:** quyền = (nhóm người dùng) × (mục chức năng, VD `patients/med`) × (mức: xem/sửa). Code kiểm tra bằng lời gọi hàm tại từng màn hình.

```php
// src/Common/Acl/AclMain.php:170
public static function aclCheckCore($section, $value, $user = '', $return_value = ''): bool
// gọi ở từng màn hình, VD
if (!AclMain::aclCheckCore('patients', 'med')) { /* từ chối */ }
```

**Dữ liệu (rút gọn):**

| Nhóm | `patients/demo` | `patients/med` | `patients/appt` |
|---|---|---|---|
| Physicians | write | write | write |
| Front Office | write | — | write |

**Vấn đề với kịch bản:** BS Hà (Quận 3) có `patients/med` thì xem được **bà Lan ở Cầu Giấy**. ACL không có khái niệm "bệnh nhân nào". Tuỳ chọn `restrict_user_facility` (`library/globals.inc.php:947-952`) chỉ giới hạn lịch theo cơ sở. → **Vỡ A1.** Lời gọi rải ở hơn 150 nơi (73 nơi `patients/med`) → **rủi ro A4**.

**Khi nào đúng:** 1 phòng khám, mọi bác sĩ được xem mọi bệnh nhân. Đây là thực tế của phần lớn phòng khám nhỏ.

### 2.2 Cột tenant + `WHERE` trong code ứng dụng 💭

**Ý tưởng:** mỗi bảng có `clinic_id`, mỗi API tự thêm `WHERE clinic_id = :user_clinic`. Đây là V1 trong [`1c-G`](./1c-greenfield-phan-quyen.md#v1--cột-clinic_id--vai-trò-cố-định-chỉ-có-q1), mô hình SaaS phổ biến nhất.

| observation.id | patient_id | clinic_id | value |
|---|---|---|---|
| bp-150 | lan | 1 (Cầu Giấy) | 150 |

**Vấn đề:**
- `clinic_id` 1 giá trị → vỡ A2.
- Điều kiện ở từng API → A4 phụ thuộc kỷ luật code review.
- Lễ tân/bác sĩ phân biệt bằng `if role` trong code → A5 vỡ.

### 2.3 Postgres Row-Level Security 💭📘

**Ý tưởng:** chính sách nằm trong DB, DB tự thêm điều kiện cho mọi câu truy vấn.

```sql
-- 💭
ALTER TABLE observation ENABLE ROW LEVEL SECURITY;
CREATE POLICY clinic_isolation ON observation
  USING (clinic_ids && current_setting('app.user_clinics')::int[]);
```

**Điểm mạnh thật:** A4 tốt nhất. Kể cả câu SQL viết tay cũng bị lọc. A7 tốt nếu có index.

**Vấn đề:**
- Chính sách viết bằng SQL, sửa cần DBA và migration → A5 vỡ.
- Không biết ngữ nghĩa nghiệp vụ (bệnh nhân, compartment, reference), mọi thứ phải dịch sang cột.
- A3 phần "theo trường" (ẩn 1 trường) không làm được bằng RLS, cần thêm view hoặc column privilege.
- A6 phải tự dựng: truyền scope của app vào `current_setting`.

### 2.4 ACL theo cặp người × bệnh nhân 💭

**Ý tưởng:** bảng `access(user_id, patient_id, level)`.

| user | patient | level |
|---|---|---|
| minh | lan | write |
| hoa | lan | read-demo |

**Vấn đề:**
- 200 × 48.000 = tới 9,6 triệu dòng. Bệnh nhân mới phải sinh khoảng 17 dòng (số nhân viên của chi nhánh). Nhân viên mới phải sinh khoảng 4.000 dòng.
- Nhân sự thay đổi thì bảng lệch khỏi thực tế → vỡ A5 và rủi ro A1.

**Khi nào đúng:** quan hệ chăm sóc **cá nhân hoá thật sự**, VD 1 điều dưỡng theo dõi đúng 30 bệnh nhân. Ngay cả khi đó, Medplum vẫn mô hình hoá bằng `CareTeam` thay vì bảng cặp (xem Phần 8).

### 2.5 Policy-as-code — HAPI FHIR `AuthorizationInterceptor` ✅

**Ý tưởng:** lập trình viên viết 1 lớp Java, mỗi request tự dựng danh sách luật bằng `RuleBuilder`.

```java
// hapi-fhir-server/.../interceptor/auth/AuthorizationInterceptor.java:79, :264
public class AuthorizationInterceptor implements IRuleApplier {
  public List<IAuthRule> buildRuleList(RequestDetails theRequestDetails) {
    return new ArrayList<>();   // mặc định rỗng: dev PHẢI override
  }
}
// 💭 cách dùng điển hình
return new RuleBuilder()
  .allow().read().allResources().inCompartment("Patient", new IdType("Patient/lan")).andThen()
  .denyAll().build();
// RuleBuilder.java:535, :560 (inCompartment), :634 (inCompartmentWithFilter)
```

Kèm `SearchNarrowingInterceptor` (`.../auth/SearchNarrowingInterceptor.java:94`, `buildAuthorizedList` ở `:220`) để **viết lại câu search** của người dùng, thêm điều kiện compartment.

**Điểm mạnh:** A1, A3 tốt (theo compartment, loại resource, hành động). Linh hoạt tuyệt đối vì là code.

**Vấn đề:**
- Luật là code Java → thêm vai trò = deploy → **vỡ A5**.
- Kiểm tra quyền và thu hẹp search là **2 cơ chế riêng** dev phải giữ khớp nhau → rủi ro A4 ở mức cấu hình.

### 2.6 Policy-as-data + compartment — Medplum ✅

Tóm tắt [`1c-access-control.md`](./1c-access-control.md):
- `AccessPolicy` là resource (dữ liệu). Luật có `criteria` **viết bằng search FHIR**, có tham số `%organization`, `%patient`, `%profile`.
- Gán qua `ProjectMembership.access[]`.
- Nhãn phạm vi được chép vào `meta.compartment` của từng resource khi ghi (`$set-accounts` với `propagate`).
- **1 điểm ghép** cho search (`repo.ts:1932-1999`) và 1 điểm kiểm tra cho đọc/ghi (`core/access.ts:121`).

| Resource | compartments |
|---|---|
| `Observation/bp-150` | `{Organization/cau-giay, Patient/lan}` |

`ProjectMembership` BS Minh: 2 phần tử `access[]`, `%organization` = Cầu Giấy và Quận 3.

**Vấn đề:**
- Đổi chi nhánh của bệnh nhân phải chạy lại `$set-accounts` (nhãn chép sẵn, không tự cập nhật).
- `criteria` có 2 bộ máy đánh giá (SQL và trong bộ nhớ) → phải test cả 2.
- Membership không có policy = toàn quyền project (hành vi cũ, `accesspolicy.ts:234-238`).

---

## Phần 3 — Bảng so sánh trực diện

| Kỹ thuật | A1 theo bản ghi | A2 nhiều phạm vi | A3 loại × hành động × trường | A4 không thể quên | A5 không deploy | A6 bên thứ 3 | A7 nhanh |
|---|---|---|---|---|---|---|---|
| 2.1 RBAC chức năng (OpenEMR) | ✗ | — | ◐ không theo trường | ✗ rải code | ◐ cấu hình nhóm | ✗ | ✅ |
| 2.2 Cột + WHERE trong code | ✅ | ✗ | ✗ | ✗ | ✗ | ✗ | ✅ |
| 2.3 Postgres RLS | ✅ | ✅ nếu dùng mảng | ◐ | ✅ **tốt nhất** | ✗ | ◐ | ✅ |
| 2.4 ACL cặp | ✅ | ✅ | ◐ | ◐ | ✗ phình | ✗ | ◐ |
| 2.5 Policy-as-code (HAPI) | ✅ | ✅ | ✅ | ◐ 2 cơ chế | ✗ | ◐ | ✅ |
| 2.6 Policy-as-data (Medplum) | ✅ | ✅ | ✅ | ✅ 1 điểm ghép | ✅ | ✅ ∩ SMART | ✅ GIN |

## Phần 4 — Vì sao loại

| Ứng viên | Đặc tính loại nó (cho bài toán chuỗi phòng khám / SaaS) | Ghi chú |
|---|---|---|
| 2.1 | A1 | **Đúng** cho 1 phòng khám |
| 2.2 | A2, A4 | Đúng cho SaaS đơn giản, mỗi khách 1 tenant không giao nhau |
| 2.3 | A5 | 🧠 Là **lớp phòng thủ thứ 2** rất tốt, đặt dưới 2.6. Không phải thay thế |
| 2.4 | A5, quy mô | Đúng cho nhóm chăm sóc nhỏ, nhưng nên mô hình bằng `CareTeam` |
| 2.5 | A5 | Đúng khi đội dev kiểm soát toàn bộ luật và hiếm khi đổi |

## Phần 5 — 4 lối tắt nghe hợp lý nhưng sai

**A — *"Ẩn ở giao diện là đủ, user thường đâu biết gọi API."*** Sai từ gốc: app di động, app đối tác, và bất kỳ ai mở DevTools đều gọi API trực tiếp. Phân quyền phải ở **tầng dữ liệu** (Medplum: `Repository`), không ở tầng giao diện.

**B — *"Mỗi chi nhánh 1 `Project` cho chắc."*** Tách cứng thì hội chẩn và bệnh nhân khám nhiều nơi phải sao chép dữ liệu giữa project. `Project` dành cho **khác pháp nhân / khác hợp đồng dữ liệu**. Chi nhánh dùng compartment.

**C — *"Dùng `hiddenFields` để giấu chẩn đoán nhạy cảm."*** Trường bị ẩn khi trả về, nhưng search theo giá trị vẫn lộ **sự tồn tại**: `Condition?code=B20` (HIV) trả 1 kết quả là đã lộ. Dữ liệu nhạy cảm phải chặn bằng loại resource hoặc `criteria`, có thể kèm `meta.security` label.

**D (leo quá xa) — *"MVP 1 phòng khám cũng cần policy-as-data, compartment, SMART scopes."*** Không có bằng chứng A1/A2/A6 thì chi phí cấu hình và test policy lớn hơn lợi ích. RBAC 3 vai trò (bác sĩ, lễ tân, quản trị) là đủ. Nếu dùng Medplum, vẫn **nên** gán 3 policy đơn giản thay vì để trống (Phần 2.6, điểm cuối).

## Phần 6 — Medplum tại từng ngã rẽ

| Ngã rẽ | Medplum chọn | Bằng chứng ✅ |
|---|---|---|
| Tránh 2.1 (chỉ chức năng) | Luật có `criteria` trên dữ liệu | `core/access.ts:139-163` |
| Tránh 2.2 (rải code) | Ghép policy tại **1** hàm cho mọi search | `repo.ts:451-454` (gọi), `:1932-1999` (dựng) |
| Tránh 2.3 (SQL trong DB) | Policy viết bằng **ngôn ngữ search FHIR**, là resource sửa được qua API | `repo.ts:1974-1983` dùng lại `buildSearchExpression` |
| Tránh 2.4 (bảng cặp) | Nhãn ở **dữ liệu** (`compartments`), tham số ở **membership** (`%organization`). Số dòng = số tenant × số người, không phải số bệnh nhân × số người | `accesspolicy.ts:259-301` |
| Tránh 2.5 (code Java) | Policy là dữ liệu, admin project tự sửa trong giới hạn | `accesspolicy.ts:343-400` |
| Giống 2.5 | Compartment là đơn vị quyền chính, giống `inCompartment` của HAPI | `repo.ts:2061-2101` |

## Phần 7 — Ma trận đánh đổi

| | Medplum (nhãn chép lúc ghi) | Kiểu JOIN lúc đọc (V3 trong 1c-G) |
|---|---|---|
| Search theo chi nhánh | `compartments && ARRAY[...]` trên 1 bảng, có GIN | JOIN qua bệnh nhân → chi nhánh **mỗi** lần |
| Bệnh nhân đổi chi nhánh | Phải ghi lại nhãn trên **mọi** resource của bệnh nhân (`$set-accounts` + `propagate`, có chế độ async) | Đổi 1 dòng |
| Tần suất | 💭 Search: hàng nghìn lần/giờ. Đổi chi nhánh: vài lần/ngày | |

🧠 Đánh đổi bất đối xứng y như bài Saleor: tối ưu cho thao tác **xảy ra liên tục** (đọc), chấp nhận chi phí ở thao tác **hiếm** (đổi chi nhánh). Đây là lý do `$set-accounts` có chế độ `respond-async`: bệnh nhân lâu năm có hàng nghìn resource.

## Phần 8 — Đối chứng nội bộ

Chính Medplum có chỗ dùng gần kiểu ACL theo cặp (2.4): **CareTeam làm tenant** (📘 `packages/docs/docs/access/multi-tenant-access-policy.md`, mục CareTeam). Mỗi bệnh nhân 1 `CareTeam`, membership của điều dưỡng **tích luỹ 1 phần tử `access[]` cho mỗi bệnh nhân** được phân công.

🧠 Đây đúng là 2.4, nhưng đặt ở đúng nơi nó hợp lý: **nhóm chăm sóc nhỏ, phân công cá nhân**. Mỗi người chỉ có vài chục phần tử, không phải hàng nghìn. Bài học giống bài Saleor: kỹ thuật "đắt" chỉ dùng ở chỗ bài toán thật sự đòi hỏi.

## Phần 9 — Luyện phản biện

**1 — CISO:** *"Sao không dùng RLS của Postgres? DB tự chặn, an toàn hơn chặn ở ứng dụng."*

<details><summary>Gợi ý</summary>

Đồng ý RLS mạnh ở A4. Nhưng vận hành không tự sửa được (A5), và RLS không hiểu FHIR. Đề xuất **kết hợp**: policy-as-data là lớp chính, RLS theo `projectId` là lớp phòng thủ thứ 2 cho trường hợp có người truy cập DB trực tiếp (BI, script). Lưu ý đây là đề xuất 🧠, không phải tính năng có sẵn của Medplum.
</details>

**2 — Trưởng IT phòng khám:** *"Policy viết bằng câu search khó đọc với người không phải dev."*

<details><summary>Gợi ý</summary>

Đúng. Giải pháp không phải bỏ policy-as-data mà là **chuẩn hoá vài template** (bác sĩ chi nhánh, lễ tân, bảo hiểm) có tham số, rồi để người vận hành chỉ chọn template + chi nhánh. Đó chính là thiết kế `access[].policy + parameter`.
</details>

**3 — Dev:** *"Chép nhãn vào mọi resource là phi chuẩn hoá, sẽ lệch dữ liệu."*

<details><summary>Gợi ý</summary>

Có rủi ro lệch thật (Phần 7). Nhưng nhãn được tính lại **mỗi lần ghi** (`repo.ts:1021`, `:1060`), và có `$set-accounts` + `propagate` để đồng bộ hàng loạt. Đổi lại, phép kiểm tra chạy trên mọi câu search không cần JOIN. Câu hỏi đúng không phải "có phi chuẩn hoá không" mà là "lệch thì có cơ chế sửa không, và sửa ở thao tác hiếm hay thao tác thường xuyên".
</details>

## Phần 10 — Checklist

| Câu hỏi | Không → | Có → |
|---|---|---|
| Có cần giới hạn theo **bản ghi** (bệnh nhân/chi nhánh), không chỉ theo chức năng? | RBAC chức năng (2.1) | (xuống) |
| Người hoặc bệnh nhân có thuộc **nhiều** phạm vi? | Cột tenant (2.2), nhưng gom điều kiện về 1 lớp | (xuống) |
| Vận hành cần tự thêm vai trò/phạm vi? | Policy-as-code (2.5) chấp nhận được | Policy-as-data (2.6) |
| Có truy cập DB trực tiếp ngoài API (BI, script)? | (bỏ qua) | Thêm RLS theo tenant làm lớp thứ 2 |
| Có dữ liệu đặc biệt nhạy cảm? | (bỏ qua) | Chặn bằng loại resource / `criteria` / security label. **Không** chỉ dùng `hiddenFields` |
| Có app bên thứ 3? | (bỏ qua) | SMART scopes ∩ policy |
| Có tình huống cấp cứu? | (bỏ qua) | Quy trình break-glass có thời hạn và audit (📘 `packages/docs/docs/access/access-policies.md:380-404`) |
