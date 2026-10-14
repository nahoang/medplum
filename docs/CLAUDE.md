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

- **Nguồn chính:** Medplum, chính repo này (`nahoang/medplum`). Giai đoạn 0–2 ở nhánh `claude/peaceful-allen-kdyk9w`; Giai đoạn 3–5 ở nhánh `claude/eager-darwin-e7o2j0` (nối tiếp từ nhánh trước, chứa toàn bộ). Commit dùng để grounding: `16cb130`.
- **Nguồn đối chứng:** OpenEMR, commit `4c1eb8d` (Giai đoạn 0–2) và `ebe3ee8` (Giai đoạn 3–5, bản clone mới; số dòng `sql/database.sql` trích trong file 3x–5x theo commit này). **Không fork.** Clone chỉ đọc, mất khi phiên kết thúc:

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
| `docs/health/README.md` | **Mục lục đọc theo vai trò** (consultant / BA / dev) + 10 phát hiện đáng nhớ nhất. Đọc file này đầu tiên khi quay lại domain |
| `docs/health/00-tong-quan-medplum.md` | Bản đồ monorepo, đường đi 1 request ghi, so sánh triết lý Medplum với OpenEMR |
| `docs/health/0a-fhir-primer.md` | 10 khái niệm FHIR, kịch bản bà Lan |
| `docs/health/0b-ban-do-nghiep-vu-resource.md` | Bảng tra: nghiệp vụ → resource FHIR → operation Medplum → bảng OpenEMR, kèm stakeholder |
| `docs/health/1a-fhir-datastore.md` | Medplum lưu resource xuống Postgres thế nào (4 chiến lược index, lịch sử, tiến hoá TEXT → NUMRANGE), đối chiếu OpenEMR |
| `docs/health/1a-comparison-fhir-datastore.md` | So sánh 5 cách lưu: cột cố định (OpenEMR), EAV chuỗi, JSONB, bảng index theo kiểu (HAPI), bảng rộng (Medplum). Checklist chọn cho MVP. Phụ lục tìm tên tiếng Việt |
| `docs/health/1b-search.md` | Search FHIR → SQL: dispatch theo chiến lược lưu trữ, bộ lọc tenant/phân quyền gắn vào mọi query, chained/_has, _include, cursor, đếm ước lượng. AccessPolicy `criteria` chính là 1 câu search |
| `docs/health/1c-*.md` (3 file) | Phân quyền: G (chuỗi phòng khám, V1→V3), R (Project → ProjectMembership → AccessPolicy → compartment, điểm chặn), C (6 kỹ thuật: RBAC OpenEMR, cột tenant, RLS, ACL cặp, HAPI policy-as-code, Medplum policy-as-data) |
| `docs/health/1d-auth-smart.md` | User / ProjectMembership / Login / ClientApplication, các grant type, SMART on FHIR (scope ∩ policy, ngữ cảnh bệnh nhân), on-behalf-of, so với OpenEMR |
| `docs/health/1e-terminology.md`, `1e-comparison-terminology.md` | Bảng mã: CodeSystem/ValueSet/ConceptMap, bảng `Coding` + `Coding_Property` (EAV), cây is-a bằng CTE đệ quy, `$expand` không dấu. So sánh 6 cách lưu cây mã (OpenEMR phẳng, Medplum kề+CTE, HAPI tổ tiên sẵn + pre-expansion, closure, server ngoài) |
| `docs/health/1f-history-audit-deletion.md` | Lịch sử phiên bản vs AuditEvent (log hoặc DB) vs 4 mức xoá (mềm, expunge, expunge everything, purge), If-Match, so với bảng `log` có checksum của OpenEMR |
| `docs/health/2a-subscriptions.md`, `2a-comparison-event-delivery.md` | Subscription: dispatch queue → subscription worker, 5 lớp lọc (FHIRPath `%previous`), rest-hook/Bot/websocket, retry 4–19 lần, tự tắt, HMAC. So sánh 6 cách: polling, hook đồng bộ (OpenEMR), trigger, hàng đợi sau commit (Medplum), outbox (HAPI), CDC |
| `docs/health/2b-bots.md`, `2b-comparison-extensibility.md` | Bot: 6 cách kích hoạt (Subscription, `$execute`, cron, webhook công khai, CDS Hooks, pre-commit), 3 runtime (Lambda, vmcontext, Fission), membership + secrets, ~50 Bot mẫu theo nghiệp vụ. So sánh 6 cách tuỳ biến: fork, module OpenEMR, interceptor HAPI, webhook ngoài, Bot Lambda, Bot vmcontext |
| `docs/health/2c-agent-hl7v2.md` | HL7 v2 (MSH/PID/PV1/OBR/OBX, ADT/ORM/ORU/SIU/DFT), MLLP. Medplum Agent trong mạng bệnh viện ↔ WSS ↔ server. ACK original (3–4 tin/s) vs enhanced (300–400 tin/s). Hàng đợi bền SQLite trên Agent, **mặc định tắt** (`app.ts:562`). So với HL7 lab của OpenEMR |
| `docs/health/2d-interoperability.md` | 6 cửa ra: Bulk `$export` (NDJSON, `_since`/`_type`), `$everything`, `$summary` + C-CDA, SMART Health Card/Link. Danh sách tiêu chí **ONC Medplum đã chứng nhận** (a2…g10) và ý nghĩa kinh doanh. So với OpenEMR EHI exporter |
| `docs/health/3a-*.md` (2 file) | Định danh & tiếp nhận: G (bà Lan đăng ký 3 lần + con gái cùng tên), R (`$match` theo khung CMS v3.4.0: 3 chiến lược thu ứng viên, 12 trường, bảng tổ hợp, 3 tầng, chặn song sinh, audit `onlyCertainMatches`; `$extract` SDC; Bot tiếp nhận luôn tạo mới; gộp bằng `Patient.link` ở Bot mẫu). Đối chiếu OpenEMR `dupscore` + gộp vật lý |
| `docs/health/3b-*.md` (3 file) | Đặt lịch: G, R (`SchedulingParameters` thay ô sinh sẵn, `$find` tính giờ trống, sức chứa quét sự kiện lấy min, `$hold`/`$book` trong giao dịch **SERIALIZABLE** + thử lại 3 lần, `$hold` không TTL), C (6 cách chống đặt trùng: không kiểm tra/OpenEMR, check-then-insert, UPDATE ô sinh sẵn, exclusion constraint, khoá tài nguyên, SSI) |
| `docs/health/3c-charting-encounter.md` | 2 lớp dữ liệu (người / lần khám), SOAP → FHIR, vòng đời Encounter, **ký & khoá chỉ ở giao diện** app Provider (Provenance không có chữ ký số), phân mục `$summary`. OpenEMR `esign_signatures` có hash |
| `docs/health/3d-*.md` (2 file) | Care plan: G (phác đồ tiểu đường, V3 = `$apply` + cron + Subscription), R (`$apply` sinh CarePlan/RequestGroup/Task/ServiceRequest nháp; **bỏ qua condition, action lồng, timing, không giao dịch**; Task là động cơ quy trình; nhận việc bằng JSON Patch `test` versionId). OpenEMR CDR chỉ sinh nhắc nhở |
| `docs/health/3e` … `3i` (5 file R ngắn) | Thuốc & eRx (vendor-neutral order, `status: unknown` khi mất phản hồi, kho thuốc OpenEMR), xét nghiệm (Health Gorilla AOE, `billTo`, DICOMweb 1 phần), Questionnaire (logic form ở client, server không kiểm tra câu bắt buộc), giao tiếp (thread 2 tầng Communication, Task trả lời, eFax hosted, phiên nhắn tin = Encounter để tính tiền), Consent (**không chặn truy cập**, ảnh chữ ký) |
| `docs/health/4a-*.md` (2 file) | Quyền lợi BH: G (Medicare + Medigap, 3 tầng câu hỏi, ROI giả định), R (`Coverage`, `$submit` = **bộ điều phối** setting → OperationDefinition → Bot; Bot mẫu giả lập). OpenEMR tự dựng X12 270, `EQ*30` cố định |
| `docs/health/4b-*.md` (3 file) | Định giá: G (3 con số giá: niêm yết, allowed, thực trả; điều chỉnh hợp đồng), R (`ChargeItemDefinition/$apply`: không nhân số lượng, bỏ tax, không hiệu lực, điều kiện cuối ghi đè; app Provider **tính lại giá mỗi lần mở**; `$set-accounts` là phân quyền), C (5 cách: phẳng, ma trận OpenEMR, bảng/payer, công thức % bảng gốc, quy tắc/mã Medplum; so Saleor) |
| `docs/health/4c-claim.md` | 837 → 835, nhóm CAS (CO/PR/OA/PI), CARC, chỉ số A/R. `Claim/$submit` (+ prior auth), `$export` CMS-1500 (không chia trang > 6 dòng), app Provider thiếu `diagnosisSequence`, Candid/Stedi. Medplum **không có** 835/công nợ; OpenEMR có `ar_session`/`ar_activity` |
| `docs/health/4d-do-luong-chat-luong.md` | Value-based care, `$evaluate-measure` = search + đếm resource theo `_lastUpdated` (4 giới hạn). OpenEMR có engine CQL PHP + QRDA |
| `docs/health/4e-phu-luc-bhyt-viet-nam.md` | Ánh xạ revenue cycle Mỹ ↔ BHYT: 1 payer, giá nhà nước, XML QĐ 130/4750/3176, tra cứu thẻ qua cổng giám định, mức hưởng, Luật 51/2024. Có link nguồn, **chưa đối chiếu văn bản gốc** |
| `docs/health/5a` … `5c` (3 file R) | Rate limit theo IP, FHIR quota theo điểm (đọc 1/tìm 20/ghi 100, mặc định 50.000/phút/user), resource cap; self-host (CDK/Helm/Terraform), migration 2 pha + manifest `requiredBefore`, sharding theo project (đang chuẩn bị); `$init` 4 policy mặc định, `$clone`, `$archive`/`$restore` giữ lịch sử |
| `docs/health/5d-tong-hop-chien-luoc.md` | **Bài tổng hợp chiến lược**: vị trí chuỗi giá trị, Medplum làm gì/để gì, open-core + bảng giá công khai (Free/Production 2.000$/Premium 6.000$/Enterprise), lợi thế bền (chứng nhận, vận hành, đối tác), đối thủ, rủi ro, cây quyết định MVP |

### Đánh giá nguồn học (đã chốt)

Đây là kết luận đã trao đổi với người dùng. Không cần lập luận lại, chỉ cần áp dụng.

- Medplum dạy tốt khoảng **1/3** domain: **dữ liệu & quy trình** (FHIR, lưu trữ, phân quyền, liên thông).
- 2 lớp còn lại **không có trong code**:
  - **Kiến thức y khoa:** ý nghĩa chỉ số, cách dùng SNOMED/LOINC/ICD
  - **Pháp lý & dòng tiền:** bảo hiểm, quy định từng nước. Medplum chỉ theo Mỹ.
- Medplum là nền tảng "headless", **không phải EHR**. Logic lâm sàng phần lớn là "CRUD chuẩn" theo spec. Ngoại lệ có logic server dày là đặt lịch (`$find/$hold/$book…`) và terminology.
- Độ khó học, từ dễ đến khó: **ecommerce < fintech < y tế**. Lợi thế của người dùng là nền tài chính, giúp tiếp cận revenue cycle nhanh.
- Nếu dự án ở **Việt Nam**: Giai đoạn 4 (revenue cycle Mỹ) ít dùng lại được. Cần học chuẩn dữ liệu XML thanh toán BHYT của Bộ Y tế (QĐ 130/QĐ-BYT, sửa bởi QĐ 4750/QĐ-BYT và QĐ 3176/QĐ-BYT; tổng hợp ở `health/4e-phu-luc-bhyt-viet-nam.md`, **chưa đối chiếu bản gốc**) và quy định hồ sơ bệnh án điện tử.
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
   - **Chữ Đ/đ không có dạng tách dấu trong Unicode.** `foldText` (JS) của Medplum và bộ chuẩn hoá của HAPI đều **không** đổi Đ → D. `unaccent` của Postgres (dùng trong `$expand`) nhiều khả năng có. Cần test.
6. **Medplum không hỗ trợ search `code:in=ValueSet` và `code:below=`** (`token-column.ts:240-255`). HAPI hỗ trợ bằng cách tự expand rồi so danh sách mã. Cách làm thay trên Medplum: `$expand` rồi search danh sách mã, hoặc Bot gắn nhãn lúc ghi.
7. **Subscription của Medplum có thể mất sự kiện**: enqueue vào Redis sau commit, lỗi chỉ được log (`workers/index.ts:77-85`). HAPI dùng outbox trong DB (`HFJ_RESOURCE_MODIFIED`). Quy trình nguy hiểm cần thêm cron quét làm lưới an toàn.
8. HAPI và Medplum đều thoả đủ yêu cầu, khác nhau ở đánh đổi **dòng ↔ cột**: HAPI chạy nhiều loại DB, Medplum gắn chặt Postgres.
9. Triết lý đối lập: **Medplum "chuẩn trước, nghiệp vụ tự xây"** vs **OpenEMR "nghiệp vụ trước, FHIR là lớp dịch"** (`src/Services/FHIR/*`).
10. **Đường ranh nhất quán (Giai đoạn 3–5):** thứ gì là dữ liệu và bất biến hệ thống thì Medplum làm kỹ (search, phân quyền, đặt lịch SERIALIZABLE, ghép hồ sơ CMS, migration 2 pha). Thứ gì là chính sách kinh doanh hoặc quy định riêng 1 thị trường thì để cho Bot/đối tác. Chi tiết: `5d` mục 3.
11. **3 operation "tri thức" là hiện thực tối thiểu:** `PlanDefinition/$apply` bỏ qua `condition`/action lồng/timing, không giao dịch; `ChargeItemDefinition/$apply` không nhân `quantity`, bỏ `tax`, không xét `effectivePeriod`, nhiều điều kiện thì điều kiện cuối ghi đè; `$evaluate-measure` đếm resource (không đếm bệnh nhân) theo `_lastUpdated`.
12. **`$submit` (eligibility, claim, prior auth) chỉ là bộ điều phối:** đọc `Project.setting` → OperationDefinition → Bot (`operations/utils/submit.ts:21-51`). Medplum không có 835, sổ công nợ.
13. **Đặt lịch:** giờ trống được **tính** từ `SchedulingParameters`, chỉ lưu Slot bận; chống trùng bằng giao dịch SERIALIZABLE + kiểm tra lại + thử lại 3 lần (`utils/scheduling.ts:1093-1114`, test song song `book.test.ts:781`). **`$hold` không có TTL.**
14. **Tiếng Việt khi ghép hồ sơ:** `foldString` xoá chữ Đ ("Đặng" → "ang"); bước thu ứng viên của `$match` so SĐT nguyên văn và tìm tên không bỏ dấu, nên "Nguyen" + "0912345678" không tìm ra "Nguyễn" + "0912 345 678".
15. **Ghi nhận ≠ thực thi:** `Consent` không chặn truy cập; "ký & khoá" hồ sơ trong app Provider chỉ khoá giao diện; Provenance không có chữ ký số.
16. **App mẫu Provider có lỗi nghiệp vụ tài chính:** tính lại giá mỗi lần mở tab Billing (giá trôi), Claim không có `diagnosisSequence`, không nhân số lượng.
17. **Mô hình kinh doanh:** Free tier không bật Bot (mà Bot chứa mọi logic nghiệp vụ); Production 2.000$/tháng, Premium 6.000$/tháng (medplum.com/pricing, 10/2026). Lợi thế bền là chứng nhận (ONC, SOC 2, HITRUST) + vận hành + mạng đối tác, không phải code.

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
| 1c | `1c-greenfield-phan-quyen.md`, `1c-access-control.md`, `1c-comparison-phan-quyen.md` | G + R + C | ✅ Xong |
| 1d | `1d-auth-smart.md` | R | ✅ Xong |
| 1e | `1e-terminology.md`, `1e-comparison-terminology.md` | R + C | ✅ Xong |
| 1f | `1f-history-audit-deletion.md` | R | ✅ Xong. **Hết Giai đoạn 1** |
| 2a | `2a-subscriptions.md`, `2a-comparison-event-delivery.md` | R + C | ✅ Xong |
| 2b | `2b-bots.md`, `2b-comparison-extensibility.md` | R + C | ✅ Xong |
| 2c | `2c-agent-hl7v2.md` | R | ✅ Xong |
| 2d | `2d-interoperability.md` | R | ✅ Xong. **Hết Giai đoạn 2** |
| 3a | `3a-greenfield-dinh-danh-benh-nhan.md`, `3a-dinh-danh-tiep-nhan.md` | G + R | ✅ Xong |
| 3b | `3b-greenfield-dat-lich.md`, `3b-dat-lich.md`, `3b-comparison-chong-dat-trung.md` | G + R + C | ✅ Xong |
| 3c | `3c-charting-encounter.md` | R | ✅ Xong |
| 3d | `3d-greenfield-careplan.md`, `3d-careplan-task.md` | G + R | ✅ Xong |
| 3e–3i | `3e-thuoc-ke-don.md`, `3f-xet-nghiem-hinh-anh.md`, `3g-questionnaire.md`, `3h-giao-tiep.md`, `3i-consent.md` | R | ✅ Xong. **Hết Giai đoạn 3** |
| 4a | `4a-greenfield-quyen-loi-bao-hiem.md`, `4a-quyen-loi-bao-hiem.md` | G + R | ✅ Xong |
| 4b | `4b-greenfield-dinh-gia.md`, `4b-dinh-gia-ghi-phi.md`, `4b-comparison-dinh-gia.md` | G + R + C | ✅ Xong |
| 4c, 4d | `4c-claim.md`, `4d-do-luong-chat-luong.md` | R | ✅ Xong |
| 4e | `4e-phu-luc-bhyt-viet-nam.md` | Phụ lục | ✅ Xong (nguồn tổng hợp, chưa đối chiếu bản gốc). **Hết Giai đoạn 4** |
| 5a–5c | `5a-gioi-han-do-luong.md`, `5b-self-host-sharding-migration.md`, `5c-vong-doi-project.md` | R | ✅ Xong |
| 5d | `5d-tong-hop-chien-luoc.md` | Tổng hợp | ✅ Xong. **Hết Giai đoạn 5** |
| — | `README.md` (mục lục theo vai trò) | — | ✅ Xong. **Hết kế hoạch** |

### Việc tiếp theo

**Trạng thái:** đã xong **toàn bộ kế hoạch** (Giai đoạn 0–5, 44 file kiến thức + `README.md`). Nhánh mới nhất: `claude/eager-darwin-e7o2j0`.

Việc có thể làm tiếp, theo thứ tự giá trị (hỏi người dùng chọn, đừng tự làm hết):
1. **Hỏi thông tin dự án MVP** (loại hình, thị trường, vai trò). Mọi khuyến nghị ở `5d` mục 8 phụ thuộc câu trả lời này.
2. **Kiểm chứng bằng chạy thật** (cần `docker compose up` Postgres + Redis): `Patient?name=nguyen` có khớp "Nguyễn" không; `$match` với họ có chữ Đ; 2 `$book` song song. Ghi kết quả vào `3a`, `3b`, `1a-comparison`.
3. **Đối chiếu văn bản gốc** cho `4e` (QĐ 130/4750/3176, Luật 51/2024, TT 22/2023, TT 21/2024, TT 13/2025, TT 26/2025). Hiện chỉ có nguồn tổng hợp.
4. Nếu dự án ở Việt Nam: viết `4e-greenfield-xml-bhyt.md` (thiết kế Bot dựng XML1–XML5 từ resource FHIR).
5. Viết "Tự luyện" cho từng file G (đáp án tự làm của người dùng, so với Phần 5).

**Quy trình mỗi module** (đã dùng ổn định 2 phiên):
1. Đọc code bằng `grep -n` / `sed -n`.
2. Viết file.
3. **Kiểm lại mọi `file:line`** (phiên 2 vẫn lệch khoảng 1/4 trích dẫn ở bản nháp, chủ yếu do đếm dòng từ output đã lọc bằng `grep -v`).
4. Cập nhật bảng "File" + "Tiến độ" trong file này.
5. `git add -A docs && git add -f docs/CLAUDE.md`, commit (kèm dòng Co-Authored-By/Claude-Session theo system reminder), push lên nhánh được giao trong phiên đó.

Clone lại OpenEMR/HAPI nếu cần (lệnh ở mục "Bối cảnh"). Phiên mới không còn bản clone cũ.

### Câu hỏi còn mở

- Vì sao `content` là `TEXT` chứ không `JSONB`? Chưa tìm thấy lý do chính thức. Xem mục 10 file 1a.
- Khi nào `range-column` thành mặc định và bỏ cột cũ?
- `extract.ts:118-125` khai báo `processingOrder` bằng tên biến (`allocIdExtension: 1`) thay vì giá trị URL, nên thứ tự xử lý extension có vẻ không có tác dụng. Chưa kiểm chứng có gây sai kết quả không (`3a` mục 8).
- Công thức "phí dùng thêm" của bản hosted, số liệu gọi vốn/doanh thu Medplum: nguồn công khai không thống nhất (`5d` mục 4, 9).
- Mức hưởng BHYT theo tuyến sau 01/7/2025 (`4e`).
- Thông tin dự án MVP (xem mục "Bối cảnh").
