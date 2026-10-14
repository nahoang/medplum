# CLAUDE.md — Điểm vào của thư mục `docs/` (đọc file này đầu tiên)

> File này để **phiên Claude sau nạp lại bối cảnh**, không phải hỏi lại từ đầu. Khi người dùng nói "đọc `docs/CLAUDE.md`", "làm tiếp phần health" hoặc tương tự, hãy đọc hết file này, rồi tới mục [Tiến độ](#tiến-độ) và [Việc tiếp theo](#việc-tiếp-theo) của domain đang làm.
>
> **Cập nhật file này ở cuối mỗi phiên**: tiến độ, quyết định mới, câu hỏi mới. File không được cập nhật thì phiên sau sẽ mất bối cảnh.
>
> File bị `.gitignore` của Medplum chặn (quy tắc `CLAUDE.md`, dòng 113). Sau khi sửa phải `git add -f docs/CLAUDE.md`.

## Người dùng & mục tiêu

- Consultant về chiến lược kinh doanh, tài chính, đầu tư. Giao tiếp **bằng tiếng Việt**.
- Cách học: chọn 1 hệ thống mã nguồn mở thật trong domain, đọc code để rút **domain knowledge** (nghiệp vụ, vì sao thiết kế như vậy), đối chiếu với hệ thống thứ 2, rồi dùng cho dự án MVP của mình.
- Đã học e-commerce trên Saleor (ở repo khác). Đang học **y tế số** trên Medplum, đối chiếu OpenEMR.

## Cấu trúc thư mục `docs/`

```
docs/
├── CLAUDE.md            ← file này: bối cảnh, quy ước chung, tiến độ từng domain
├── templates/           ← khuôn viết, dùng chung mọi domain
│   ├── README.md        ← khung rút gọn của template G và C
│   └── saleor-*.md      ← 2 bài mẫu gốc (giữ nguyên văn, link nội bộ bị gãy là bình thường)
└── health/              ← domain y tế
    ├── plan.md          ← kế hoạch tổng của domain
    ├── 00-*.md, 0a-*.md, 0b-*.md   ← nền tảng
    └── 1a-*.md, …       ← từng module theo mã trong plan.md
```

**Thêm domain mới** (VD fintech): tạo `docs/<domain>/plan.md`, đặt file kiến thức cùng thư mục theo mã module, thêm 1 mục `## Domain: <tên>` vào cuối file này. Không tạo thư mục ngoài `docs/`.

## Quy ước chung (mọi domain)

1. **Ngôn ngữ:** tiếng Việt, giữ thuật ngữ kỹ thuật tiếng Anh khi cần.
2. **Nhãn nguồn** cho mọi đoạn:
   - ✅ code thật có `file:line`
   - 📘 chuẩn, quy định, docs chính thức
   - 💭 mô phỏng / giá trị tự dựng
   - 🧠 suy luận

   Docs chính thức là 📘, không phải ✅.
3. **3 loại file**, thứ tự viết **G → R → C** cho module quan trọng:
   - **G** (`<mã>-greenfield-<chủ-đề>.md`): tự thiết kế trước, không mở code thật
   - **R** (`<mã>-<chủ-đề>.md`): đọc code thật
   - **C** (`<mã>-comparison-<chủ-đề>.md`): so sánh N ứng viên

   Module ít quan trọng chỉ cần R ngắn. Khung chi tiết: `docs/templates/README.md`.
4. **Commit hash** ghi ở đầu mỗi file R. Khi repo cập nhật, grep lại tên hàm trước khi tin số dòng cũ.
5. **Kiểm số dòng trước khi ghi.** Ở phiên trước, khoảng 1/3 số dòng nháp bị lệch 1–4 dòng. Luôn `grep -n`/`sed -n` xác nhận rồi mới viết `file:line`.
6. **Kịch bản xuyên suốt:** mỗi domain có 1 kịch bản dữ liệu dùng lại ở mọi file, để các bảng so sánh được với nhau.
7. **Link giữa các file** dùng đường dẫn tương đối, chỉ trỏ trong `docs/`.

---

## Domain: health (y tế số)

### Bối cảnh

- **Nguồn chính:** Medplum, chính repo này (`nahoang/medplum`), nhánh `claude/peaceful-allen-kdyk9w`. Commit dùng để grounding: `16cb130`.
- **Nguồn đối chứng:** OpenEMR, commit `4c1eb8d`. **Không fork.** Clone chỉ đọc, mất khi phiên kết thúc:

  ```bash
  GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 https://github.com/openemr/openemr /home/user/openemr/openemr
  ```

  Cần đúng commit cũ thì thử `git fetch --depth 1 origin 4c1eb8d`. Không được thì dùng HEAD mới và cập nhật hash trong file.
- **Nguồn đối chứng thứ 2:** HAPI FHIR, commit `6d818ea`. Clone chỉ đọc khi cần:

  ```bash
  GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 https://github.com/hapifhir/hapi-fhir /home/user/hapifhir/hapi-fhir
  ```
- **Kịch bản xuyên suốt:** bà **Nguyễn Thị Lan**, 68 tuổi, tiểu đường type 2, khám tại Phòng khám An Bình với BS Trần Văn Minh, huyết áp 150/95, chỉ định HbA1c.
- **Quy ước riêng:** phân biệt 3 lớp quyết định: HL7 spec / profile quốc gia / Medplum. Chỉ lớp thứ 3 mới viết "Medplum chọn…". Module chạm dữ liệu bệnh nhân phải có ít nhất 1 đặc tính pháp lý. Definition of Done: mục 6 của `docs/health/plan.md`.
- **Test:** `packages/core` chạy không cần DB. `packages/server` cần Postgres + Redis.
- **Chưa biết** (hỏi khi phù hợp, đừng tự đoán): dự án MVP thuộc loại nào (phòng khám, bệnh viện, bảo hiểm, app bệnh nhân), thị trường nào (Việt Nam hay Mỹ), người dùng đóng vai trò gì.

### File

| Đường dẫn | Nội dung |
|---|---|
| `docs/health/plan.md` | **Kế hoạch tổng**: 6 giai đoạn, khoảng 40 file, lịch 14 tuần, Definition of Done, bảng đối chứng OpenEMR theo module |
| `docs/health/00-tong-quan-medplum.md` | Bản đồ monorepo, đường đi 1 request ghi, so sánh triết lý Medplum với OpenEMR |
| `docs/health/0a-fhir-primer.md` | 10 khái niệm FHIR, kịch bản bà Lan |
| `docs/health/0b-ban-do-nghiep-vu-resource.md` | Bảng tra: nghiệp vụ → resource FHIR → operation Medplum → bảng OpenEMR, kèm stakeholder |
| `docs/health/1a-fhir-datastore.md` | Medplum lưu resource xuống Postgres thế nào (4 chiến lược index, lịch sử, tiến hoá TEXT → NUMRANGE), đối chiếu OpenEMR |
| `docs/health/1a-comparison-fhir-datastore.md` | So sánh 5 cách lưu: cột cố định (OpenEMR), EAV chuỗi, JSONB, bảng index theo kiểu (HAPI), bảng rộng (Medplum). Checklist chọn cho MVP. Phụ lục tìm tên tiếng Việt |
| `docs/health/1b-search.md` | Search FHIR → SQL: dispatch theo chiến lược lưu trữ, bộ lọc tenant/phân quyền gắn vào mọi query, chained/_has, _include, cursor, đếm ước lượng. AccessPolicy `criteria` chính là 1 câu search |

### Đánh giá nguồn học (đã chốt)

Đây là kết luận đã trao đổi với người dùng. Không cần lập luận lại, chỉ cần áp dụng.

- Medplum dạy tốt khoảng **1/3** domain: **dữ liệu & quy trình** (FHIR, lưu trữ, phân quyền, liên thông).
- 2 lớp còn lại **không có trong code**:
  - **Kiến thức y khoa:** ý nghĩa chỉ số, cách dùng SNOMED/LOINC/ICD
  - **Pháp lý & dòng tiền:** bảo hiểm, quy định từng nước. Medplum chỉ theo Mỹ.
- Medplum là nền tảng "headless", **không phải EHR**. Logic lâm sàng phần lớn là "CRUD chuẩn" theo spec. Ngoại lệ có logic server dày là đặt lịch (`$find/$hold/$book…`) và terminology.
- Độ khó học, từ dễ đến khó: **ecommerce < fintech < y tế**. Lợi thế của người dùng là nền tài chính, giúp tiếp cận revenue cycle nhanh.
- Nếu dự án ở **Việt Nam**: Giai đoạn 4 (revenue cycle Mỹ) ít dùng lại được. Cần học chuẩn dữ liệu XML thanh toán BHYT của Bộ Y tế (hiện được biết là QĐ 130/QĐ-BYT, **cần kiểm tra văn bản hiện hành**) và quy định hồ sơ bệnh án điện tử.
- Nên bổ sung: phỏng vấn 1–2 người làm thật (lễ tân, kế toán BHYT, điều dưỡng); dữ liệu giả lập Synthea.

**Dự án mã nguồn mở đã giới thiệu để đối chứng** (chưa clone, trừ OpenEMR):

| Nhóm | Dự án |
|---|---|
| FHIR server | **HAPI FHIR** (đối chứng cho 1a, bảng `HFJ_SPIDX_*`), Microsoft FHIR Server, Blaze, Fhirbase |
| EHR | **OpenEMR** (đã chọn), OpenMRS (EAV `obs`), Bahmni (gần bệnh viện Việt Nam), GNU Health, Ottehr |
| Cấp quốc gia | OpenHIE, OpenHIM, OpenCR/SanteMPI, DHIS2 |
| Công cụ | Synthea, OMOP CDM, Snowstorm, Mirth Connect (bản mới có thể đã đóng nguồn, cần kiểm tra) |

### Phát hiện quan trọng đến nay

Tóm tắt để phiên sau không phải đọc lại toàn bộ:

1. Medplum lưu mỗi resource thành **1 bản gốc JSON** (`content TEXT`) cộng **nhiều bản chiếu** phục vụ search. Có 4 chiến lược chiếu, chọn trong `packages/server/src/fhir/searchparameter.ts:116-183`:
   - `range-column`
   - `token-column` (băm UUID v5)
   - `lookup-table` (HumanName, Address, `_References`…)
   - `column`
2. Mọi lần ghi tạo 1 dòng `_History`. Xoá là xoá mềm. `$expunge` mới xoá thật.
3. **Medplum từng lưu `Observation.valueQuantity` và `date` dạng TEXT** (`packages/server/src/migrations/schema/v1.ts:2365,2391`). Sau đó đổi sang `DOUBLE PRECISION USING NULL` (v18), rồi `NUMRANGE`/`TSTZRANGE` (v105). Làm được vì cột search chỉ là bản chiếu, reindex lại từ `content` theo `Repository.VERSION` (hiện là 19, `packages/server/src/fhir/repo.ts:292`). Đây là bài học thiết kế chính của 1a.
4. OpenEMR có đủ 4 kỹ thuật lưu trữ trong 1 schema:
   - cột cố định: `patient_data` 132 cột
   - STI: `lists.type`
   - liên kết đa hình: `forms.formdir` + 38 bảng `form_*`
   - EAV: `lbf_data`

   Ngoài ra `form_vitals.bps` (huyết áp) là `varchar(40)`.
5. **Tìm tên không dấu (quan trọng cho Việt Nam):**
   - HAPI bỏ dấu khi index tên (`SP_VALUE_NORMALIZED`).
   - Medplum search `HumanName` bằng `LIKE`/`ILIKE`, không thấy bỏ dấu (`lookups/lookuptable.ts:138,176`). Nhiều khả năng `Patient?name=nguyen` không khớp "Nguyễn". **Cần test thật.**
   - Medplum có `medplum_unaccent` nhưng chỉ dùng cho `$expand` và patient match.
6. HAPI và Medplum đều thoả đủ yêu cầu, khác nhau ở đánh đổi **dòng ↔ cột**: HAPI chạy nhiều loại DB, Medplum gắn chặt Postgres.
7. Triết lý đối lập: **Medplum "chuẩn trước, nghiệp vụ tự xây"** vs **OpenEMR "nghiệp vụ trước, FHIR là lớp dịch"** (`src/Services/FHIR/*`).

### Tiến độ

| Mã | File | Loại | Trạng thái |
|---|---|---|---|
| — | `plan.md` + `docs/templates/` | — | ✅ Xong |
| 00 | `00-tong-quan-medplum.md` | R | ✅ Xong |
| 0a | `0a-fhir-primer.md` | R | ✅ Xong |
| 0b | `0b-ban-do-nghiep-vu-resource.md` | R | ✅ Xong |
| 1a | `1a-fhir-datastore.md` | R | ✅ Xong |
| 1a | `1a-comparison-fhir-datastore.md` | C | ✅ Xong |
| 1b | `1b-search.md` | R | ✅ Xong |
| 1c | Multi-tenancy & phân quyền | G + R + C | ⏳ Chưa |
| còn lại | Xem `docs/health/plan.md` mục 4 | | ⏳ Chưa |

### Việc tiếp theo

Theo thứ tự ưu tiên. Hỏi người dùng nếu họ muốn đổi.

1. **Hỏi người dùng 3 câu chưa biết** ở mục "Bối cảnh". Câu trả lời có thể cắt kế hoạch từ 14 xuống khoảng 6–8 tuần. VD dự án Việt Nam thì thay Giai đoạn 4 bằng chuẩn BHYT.
2. Viết module 1c theo G → R → C. Bối cảnh greenfield: *"Nhiều phòng khám chung 1 server, bác sĩ chỉ thấy bệnh nhân của mình"*. Code: `packages/server/src/fhir/accesspolicy.ts`, cột `compartments`, `repo.ts:1961`.

### Câu hỏi còn mở

- Vì sao `content` là `TEXT` chứ không `JSONB`? Chưa tìm thấy lý do chính thức. Xem mục 10 file 1a.
- Khi nào `range-column` thành mặc định và bỏ cột cũ?
- Thông tin dự án MVP (xem mục "Bối cảnh").
