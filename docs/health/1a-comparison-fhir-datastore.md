# 1a-C — So sánh 5 cách lưu dữ liệu y tế: vì sao Medplum chọn "bảng rộng theo resource + lookup table"

> **Loại file:** C (so sánh kỹ thuật, theo [`../templates/saleor-design-technique-comparison-product-attribute.md`](../templates/saleor-design-technique-comparison-product-attribute.md)).
> **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Đây là góc nhìn thứ 2 cho cùng module 1a:
> - [`1a-fhir-datastore.md`](./1a-fhir-datastore.md) trả lời **"Medplum làm gì"**. Đọc code, có `file:line`, là nguồn ✅ chính của file này.
> - **File này** đặt 5 kỹ thuật cạnh nhau như **ứng viên ngang hàng**, cùng 1 kịch bản dữ liệu, rồi hỏi "mỗi cái thua ở đâu, thắng ở đâu".
>
> **Khác bài Saleor ở 1 điểm quan trọng:** trong bài Saleor, 5/6 ứng viên là 💭 phản thực tế. Ở đây **4/5 ứng viên là hệ thống thật đang chạy**: OpenEMR (2 ứng viên), HAPI FHIR, Medplum. Chỉ ứng viên JSONB thuần là 💭. Vì vậy câu hỏi không còn là "vì sao cách kia sai" mà là **"mỗi cách đúng trong điều kiện nào"**.

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn)
- [Phần 1 — Bài toán: 7 đặc tính](#phần-1--bài-toán-7-đặc-tính)
- [Phần 2 — 5 ứng viên, cùng 1 kịch bản](#phần-2--5-ứng-viên-cùng-1-kịch-bản)
  - [2.1 Cột cố định — OpenEMR](#21-cột-cố-định--openemr-)
  - [2.2 EAV chuỗi — OpenEMR LBF và Medplum v1](#22-eav-chuỗi--openemr-lbf-và-medplum-v1-)
  - [2.3 JSONB thuần + GIN](#23-jsonb-thuần--gin-)
  - [2.4 Bảng index dùng chung theo kiểu — HAPI FHIR](#24-bảng-index-dùng-chung-theo-kiểu--hapi-fhir-)
  - [2.5 Bảng rộng theo resource + lookup table — Medplum](#25-bảng-rộng-theo-resource--lookup-table--medplum-)
- [Phần 3 — Bảng so sánh trực diện](#phần-3--bảng-so-sánh-trực-diện)
- [Phần 4 — Ứng viên nào bị loại, ứng viên nào chỉ khác hướng](#phần-4--ứng-viên-nào-bị-loại-ứng-viên-nào-chỉ-khác-hướng)
- [Phần 5 — 4 lối tắt nghe hợp lý nhưng sai](#phần-5--4-lối-tắt-nghe-hợp-lý-nhưng-sai)
- [Phần 6 — Thiết kế thật của Medplum tại từng ngã rẽ](#phần-6--thiết-kế-thật-của-medplum-tại-từng-ngã-rẽ)
- [Phần 7 — Ma trận đánh đổi, có số](#phần-7--ma-trận-đánh-đổi-có-số)
- [Phần 8 — Đối chứng nội bộ: Medplum dùng cách của HAPI ở đâu](#phần-8--đối-chứng-nội-bộ-medplum-dùng-cách-của-hapi-ở-đâu)
- [Phần 9 — Luyện phản biện](#phần-9--luyện-phản-biện)
- [Phần 10 — Checklist cho dự án MVP](#phần-10--checklist-cho-dự-án-mvp)
- [Phụ lục — Hệ quả riêng cho dự án ở Việt Nam](#phụ-lục--hệ-quả-riêng-cho-dự-án-ở-việt-nam)
- [Tự luyện tiếp theo](#tự-luyện-tiếp-theo)
- [Nguồn](#nguồn)

## Quy tắc nhãn nguồn

| Nhãn | Nghĩa | Dùng ở đâu |
|---|---|---|
| ✅ grounded | Code thật, có `file:line` | 2.1, 2.2, 2.4, 2.5, Phần 6, Phần 8 |
| 📘 spec | FHIR R4, Postgres docs | Phần 1 |
| 💭 minh hoạ | Ứng viên 2.3, giá trị dữ liệu, con số ước tính | 2.3, các bảng dữ liệu, Phần 7 |
| 🧠 suy luận | "Vì sao", không có tài liệu thiết kế gốc nào để trích | Phần 4, 5, 7, 9 |

Đường dẫn rút gọn: Medplum tính từ `packages/server/src/`. OpenEMR tính từ gốc repo (`/home/user/openemr/openemr`). HAPI tính từ `hapi-fhir-jpaserver-model/src/main/java/ca/uhn/fhir/jpa/model/entity/` (`/home/user/hapifhir/hapi-fhir`).

---

## Phần 1 — Bài toán: 7 đặc tính

Bài toán: **lưu hồ sơ y tế cho nhiều phòng khám, tìm kiếm nhanh và đúng, giữ được lịch sử, theo chuẩn FHIR.**

| # | Đặc tính | Vì sao đòi hỏi |
|---|---|---|
| P1 | Khoảng 150 loại tài liệu, lồng sâu, có `extension` tự do | Chuẩn FHIR quy định 📘. Không đoán trước hết được trường nào sẽ xuất hiện |
| P2 | Search đúng **kiểu** và đúng **ngữ nghĩa khoảng** | `date=ge2026-01-01` là so sánh khoảng. `value-quantity=gt140` là so sánh số |
| P3 | Mảng có cấu trúc search được từng phần tử | Nhiều tên, nhiều địa chỉ, nhiều định danh trên 1 bệnh nhân |
| P4 | Mã chuẩn `system\|code`, nhiều dạng câu hỏi | `code=8480-6`, `code=http://loinc.org\|8480-6`, `code=http://loinc.org\|` |
| P5 | Lịch sử bất biến | Pháp lý: truy vết ai sửa gì, khi nào |
| P6 | Tách tenant tuyệt đối | Phòng khám A không bao giờ thấy dữ liệu phòng khám B |
| P7 | Đổi cấu trúc index mà không động vào dữ liệu nghiệp vụ | Spec cập nhật, thêm search parameter tuỳ biến, sửa bug index |

### Giải thích vì sao từng đặc tính tồn tại

**P1: vì sao không đoán trước được.** Domain model do HL7 viết, không do đội dev viết (khác biệt K1 trong [`plan.md`](./plan.md#1-khác-biệt-cốt-lõi-khi-chuyển-từ-saleor-sang-medplum)). Đội dev không có quyền "chỉ làm 5 bảng cho đủ dùng" nếu muốn liên thông: 1 đối tác lab gửi `DiagnosticReport` có `extension` lạ thì hệ thống vẫn phải nhận và trả lại nguyên vẹn.

**P2: vì sao là đặc tính nguy hiểm nhất.** Giống P3 trong bài Saleor: sai kiểu **không báo lỗi**, chỉ trả sai kết quả. Ở y tế, "trả sai" nghĩa là bác sĩ lọc "các lần huyết áp > 140" mà sót mất 1 lần đo 150, vì `"150" < "90"` theo thứ tự chuỗi. Đây không còn là lỗi hiển thị như ecommerce, mà là **rủi ro an toàn người bệnh**.

**P5 và P6: vì sao không thương lượng được.** 2 đặc tính này đến từ luật (HIPAA ở Mỹ, quy định hồ sơ bệnh án điện tử ở Việt Nam), không đến từ sở thích kỹ thuật. Thiết kế thiếu P5/P6 không phải "thiết kế yếu hơn" mà là **không bán được** cho cơ sở y tế.

**P7: vì sao xuất hiện muộn nhưng đắt nhất.** Ngày đầu không ai thấy cần. Đến năm thứ 3, khi phải đổi kiểu cột trên bảng 100 triệu dòng, đặc tính này quyết định đó là 1 lần reindex chạy nền hay 1 đêm downtime kèm rủi ro mất dữ liệu. Bằng chứng: Medplum đã phải làm đúng việc này 2 lần (v18 và v105, xem [`1a`](./1a-fhir-datastore.md#6-con-đường-tiến-hoá-medplum-cũng-từng-lưu-số-dạng-chuỗi)).

### Hậu quả nếu bỏ qua, bằng số (💭 ước tính)

Một chuỗi 20 phòng khám, mỗi phòng 80 lượt khám/ngày, mỗi lượt khoảng 15 Observation (dấu hiệu sinh tồn, xét nghiệm):

- 20 × 80 × 15 × 300 ngày ≈ **7,2 triệu Observation/năm**. Sau 5 năm khoảng 36 triệu dòng.
- Nếu mỗi Observation bị sửa trung bình 1,3 lần (nhập lại, bổ sung kết quả), bảng lịch sử khoảng 47 triệu dòng.
- Với quy mô này, đổi kiểu 1 cột bằng `ALTER TABLE … USING <chuyển đổi>` trên bảng nguồn sự thật là thao tác khoá bảng nhiều giờ. Còn đổi trên cột chiếu rồi reindex thì chạy nền theo trang.

---

## Phần 2 — 5 ứng viên, cùng 1 kịch bản

> **Kịch bản cố định cho cả 5 mục.** Bà **Nguyễn Thị Lan** có 2 tên (khai sinh "Nguyễn Thị Lan", thường gọi "Lan Nguyễn"). Lần khám hôm nay có 2 kết quả:
> - huyết áp tâm thu **150 mmHg** (LOINC `8480-6`)
> - HbA1c **8.2 %** (LOINC `4548-4`)
>
> Mỗi mục trả lời đúng 1 câu: *"dữ liệu này nằm ở đâu, hình dạng ra sao?"*

### 2.1 Cột cố định — OpenEMR ✅

**Giới thiệu:** mỗi khái niệm nghiệp vụ là 1 cột. Đây là mặc định của thiết kế quan hệ chuẩn hoá.

**DDL thật** (rút gọn):

```sql
-- sql/database.sql:8350 — patient_data, 132 cột
CREATE TABLE `patient_data` (
  `id` bigint(20) NOT NULL auto_increment,
  `fname` varchar(255) NOT NULL default '',
  `mname` varchar(255) NOT NULL default '',
  `lname` varchar(255) NOT NULL default '',
  `DOB` date default NULL,
  ... 125 cột khác: guardian*, imm_reg_*, care_team_*, industry, county ...
);

-- sql/database.sql:2422 — form_vitals
CREATE TABLE `form_vitals` (
  `pid` bigint(20) default '0',
  `bps` varchar(40) default NULL,          -- huyết áp tâm thu: CHUỖI
  `bpd` varchar(40) default NULL,          -- huyết áp tâm trương: CHUỖI
  `weight` DECIMAL(12,6) default '0.00',   -- cân nặng: SỐ
  `pulse` DECIMAL(12,6) default '0.00',
  ...
);
```

**Dữ liệu:**

`patient_data`:

| pid | fname | mname | lname | DOB |
|---|---|---|---|---|
| 1 | Lan | Thị | Nguyễn | 1958-03-12 |

`form_vitals`:

| id | pid | bps | bpd | weight |
|---|---|---|---|---|
| 1 | 1 | `'150'` | `'95'` | 0.00 |

HbA1c **không nằm ở `form_vitals`** (không có cột). Nó đi đường khác: `procedure_order` → `procedure_report` → `procedure_result` (`:10395`, `:10493`, `:10519`).

**Vấn đề cụ thể:**
- **Tên thứ 2 "Lan Nguyễn" không có chỗ lưu.** Mỗi trường 1 cột.
- **Kiểu không nhất quán ngay trong 1 bảng:** `weight` là `DECIMAL`, còn `bps` là `varchar(40)`. Lọc "huyết áp > 140" bằng SQL phải `CAST`. Quên `CAST` thì so sánh chuỗi, `'150' > '140'` đúng nhưng `'90' > '140'` cũng đúng.
- **Luật nghiệp vụ trôi vào lớp code:** "`bps` và `bpd` đều bằng 0 nghĩa là không có dữ liệu" nằm ở lớp dịch FHIR (`src/Services/FHIR/Observation/FhirObservationVitalsService.php:880-884`), không nằm ở schema.
- **Xuất FHIR cần 1 lớp dịch cho mỗi resource:** 36 file `Fhir*Service.php` ngay trong `src/Services/FHIR/`, chưa kể các thư mục con theo từng resource.

**Sai lầm thường gặp:** chọn cột cố định không sai. Với 1 phòng khám, 1 quốc gia, quy trình ổn định, đây là cách **nhanh nhất để ra sản phẩm** và dễ viết báo cáo SQL nhất. Sai lầm là chọn nó cho sản phẩm cần **liên thông FHIR làm lõi**. Khi đó mỗi trường mới phải sửa ở 3 nơi: schema, form, lớp dịch FHIR.

### 2.2 EAV chuỗi — OpenEMR LBF và Medplum v1 ✅

**Giới thiệu:** thuộc tính thành **dòng dữ liệu**, giá trị là chuỗi. Nguồn gốc lịch sử chính là hệ thống bệnh án lâm sàng những năm 1990 (xem Phần 2.5 của bài Saleor).

**DDL thật, 2 biến thể:**

```sql
-- (a) OpenEMR — form tự cấu hình (Layout-Based Forms), sql/database.sql:10262
CREATE TABLE `lbf_data` (
  `form_id`     int(11)     NOT NULL,   -- 1 lần điền form
  `field_id`    varchar(31) NOT NULL,   -- trỏ layout_options.field_id
  `field_value` LONGTEXT,               -- MỌI giá trị là chuỗi
  PRIMARY KEY (`form_id`,`field_id`)
);
```

```sql
-- (b) Medplum phiên bản đầu, migrations/schema/v1.ts:2359-2391 (bảng "Observation")
"code" TEXT,             -- dòng 2364
"date" TEXT,             -- dòng 2365: ngày dạng chuỗi
"valueQuantity" TEXT,    -- dòng 2391: số dạng chuỗi
```

Biến thể (b) không phải EAV về hình dạng (vẫn là cột). Nhưng nó chia đúng **điểm yếu cốt lõi** của EAV chuỗi: mọi kiểu bị ép về `TEXT`.

**Dữ liệu** (biến thể a, phòng khám tự tạo form "Theo dõi tiểu đường"):

| form_id | field_id | field_value |
|---|---|---|
| 501 | `hba1c` | `'8.2'` |
| 501 | `bp_sys` | `'150'` |
| 502 | `hba1c` | `'10.1'` |
| 503 | `hba1c` | `'9'` |

**Vấn đề cụ thể, bug sort giống hệt bài Saleor:** `ORDER BY field_value` cho thứ tự `'10.1' < '8.2' < '9'`. Bệnh nhân có HbA1c cao nhất (10.1) đứng **đầu** danh sách "tăng dần". Không có exception nào. Test với 2 giá trị `'8.2'`, `'9'` sẽ không lộ lỗi.

**Bằng chứng Medplum đã trả giá cho đúng lỗi này** ✅ `migrations/schema/v18.ts:102-108`:

```sql
ALTER TABLE "Observation"
  ALTER COLUMN "date" TYPE TIMESTAMP WITH TIME ZONE USING NULL,
  ALTER COLUMN "valueQuantity" TYPE DOUBLE PRECISION USING NULL
```

**Sai lầm thường gặp:** chọn EAV vì "phòng khám nào cũng muốn form riêng". Đúng là OpenEMR cần LBF cho việc này, và LBF giải đúng nhu cầu **thu thập** dữ liệu linh hoạt. Nhưng dữ liệu đã vào EAV chuỗi thì gần như không dùng được cho **lâm sàng định lượng**: không vẽ biểu đồ, không cảnh báo ngưỡng, không thống kê.

### 2.3 JSONB thuần + GIN 💭

**Giới thiệu:** lưu nguyên resource vào 1 cột `JSONB`, đánh index GIN, search bằng toán tử JSON của Postgres. Đây là hướng của các dự án kiểu Fhirbase. Mô tả dưới đây là 💭 minh hoạ, không trích code dự án nào.

```sql
-- 💭
CREATE TABLE observation (
  id         uuid PRIMARY KEY,
  project_id uuid NOT NULL,
  resource   jsonb NOT NULL
);
CREATE INDEX ON observation USING gin (resource jsonb_path_ops);
```

**Dữ liệu:**

| id | project_id | resource |
|---|---|---|
| `b71a…` | `p-anbinh` | `{"resourceType":"Observation","code":{"coding":[{"system":"http://loinc.org","code":"8480-6"}]},"valueQuantity":{"value":150,"unit":"mmHg"},…}` |
| `c22f…` | `p-anbinh` | `{…"code":{"coding":[{"code":"4548-4",…}]},"valueQuantity":{"value":8.2,"unit":"%"},…}` |

**Điểm mạnh thật:** P1 thoả hoàn toàn. Thêm resource mới, thêm extension không cần DDL. `resource @> '{"code":{"coding":[{"code":"8480-6"}]}}'` dùng được GIN.

**Vấn đề cụ thể:**
- **P2 vỡ một nửa:** `(resource->'valueQuantity'->>'value')::numeric > 140` chạy đúng kiểu nhưng **GIN không giúp được phép so sánh khoảng**. Muốn nhanh phải tạo thêm expression index B-tree cho từng đường dẫn. Đến đó thì thực chất đang tự tạo "cột chiếu", chỉ là giấu trong index.
- **Ngữ nghĩa khoảng của FHIR không có sẵn:** `effectiveDateTime = "2026-10"` (cả tháng) và `effectivePeriod` phải được quy về khoảng trước khi so sánh. JSONB không biết điều đó.
- **P3:** "tên nào trong mảng `name` chứa *Lan*" cần `jsonb_array_elements` hoặc `jsonb_path_query`, khó dùng index.
- **P7:** đổi cách index = đổi expression index, phải build lại index trên toàn bảng.

**Sai lầm thường gặp:** "Postgres hỗ trợ JSONB tốt, cần gì phức tạp." Đúng cho prototype và cho metadata tự do. Sai khi search parameter phải chạy nhanh ở hàng chục triệu dòng với ngữ nghĩa khoảng.

### 2.4 Bảng index dùng chung theo kiểu — HAPI FHIR ✅

**Giới thiệu:** bản gốc lưu dạng văn bản (có thể nén). Mọi giá trị search được tách ra thành **dòng** trong vài bảng index, **mỗi bảng 1 kiểu dữ liệu**, **dùng chung cho mọi loại resource**. Về bản chất đây là **EAV có phân kiểu**: mỗi kiểu 1 bảng giá trị với cột đúng kiểu.

**Bảng thật:**

| Bảng | Nguồn | Vai trò |
|---|---|---|
| `HFJ_RESOURCE` | `ResourceTable.java:104-123` | 1 dòng / resource: id, loại, version hiện tại, cờ `SP_*_PRESENT`. **Không chứa nội dung** |
| `HFJ_RES_VER` | `ResourceHistoryTable.java:61-84` | **Mọi phiên bản**, nội dung ở `RES_TEXT` (LOB) / `RES_TEXT_VC` (`:135-140`). Mã hoá `RES_ENCODING`: `JSON`, `JSONC` (nén), `DEL`, `ESR` (lưu ngoài) (`ResourceEncodingEnum.java:25-52`) |
| `HFJ_SPIDX_STRING` | `ResourceIndexedSearchParamString.java:81` | `SP_VALUE_EXACT` + `SP_VALUE_NORMALIZED` (`:114-117`) |
| `HFJ_SPIDX_TOKEN` | `ResourceIndexedSearchParamToken.java:82` | `SP_SYSTEM`, `SP_VALUE`, các hash `HASH_SYS`, `HASH_SYS_AND_VALUE`, `HASH_VALUE` (`:85-112`) |
| `HFJ_SPIDX_DATE` | `ResourceIndexedSearchParamDate.java:86` | `SP_VALUE_LOW`, `SP_VALUE_HIGH` (`:91-99`): **khoảng**, giống ý tưởng range của Medplum |
| `HFJ_SPIDX_QUANTITY`, `_QUANTITY_NRML`, `_NUMBER`, `_URI`, `_COORDS` | các file cùng tên | `SP_VALUE` kiểu `Double` cho quantity (`ResourceIndexedSearchParamQuantity.java:87`) |
| `HFJ_RES_LINK` | `ResourceLink.java:54-69` | Reference: `SRC_RESOURCE_ID`, `SRC_PATH`, `TARGET_RESOURCE_ID` |

**Chìa khoá của thiết kế: `HASH_IDENTITY`** (`BaseResourceIndexedSearchParam.java:73-78`, comment gốc: *"Composite of resourceType, paramName, and partition info"*). Mỗi dòng index mang 1 số băm của bộ (tenant, loại resource, tên tham số). Mọi index đều bắt đầu bằng cột này, VD `IDX_SP_TOKEN_HASH_V2 (HASH_IDENTITY, SP_SYSTEM, SP_VALUE, RES_ID, PARTITION_ID)` (`ResourceIndexedSearchParamToken.java:68`). Nhờ vậy 1 bảng chứa token của mọi loại resource mà search vẫn thu hẹp được ngay.

**Dữ liệu** (💭 giá trị, ✅ tên bảng/cột):

`HFJ_SPIDX_TOKEN` (chỉ các dòng của 2 Observation, rút gọn):

| RES_ID | RES_TYPE | SP_NAME | SP_SYSTEM | SP_VALUE | HASH_IDENTITY |
|---|---|---|---|---|---|
| 9001 | Observation | code | http://loinc.org | 8480-6 | h(Obs,code) |
| 9001 | Observation | status | … | final | h(Obs,status) |
| 9001 | Observation | category | …observation-category | vital-signs | h(Obs,category) |
| 9002 | Observation | code | http://loinc.org | 4548-4 | h(Obs,code) |
| 9002 | Observation | status | … | final | h(Obs,status) |

`HFJ_SPIDX_QUANTITY`:

| RES_ID | SP_NAME | SP_VALUE | SP_UNITS |
|---|---|---|---|
| 9001 | value-quantity | 150.0 | mm[Hg] |
| 9002 | value-quantity | 8.2 | % |

`HFJ_SPIDX_STRING` cho bà Lan (2 tên → nhiều dòng):

| RES_ID | SP_NAME | SP_VALUE_EXACT | SP_VALUE_NORMALIZED |
|---|---|---|---|
| 77 | name | Nguyễn Thị Lan | NGUYEN THI LAN |
| 77 | name | Lan Nguyễn | LAN NGUYEN |
| 77 | family | Nguyễn | NGUYEN |

`SP_VALUE_NORMALIZED` bỏ dấu và viết hoa: `hapi-fhir-base/src/main/java/ca/uhn/fhir/util/StringUtil.java:42-70` (*"strip out diacritical marks from latin script and also convert to upper case"*).

**Vấn đề cụ thể:**
- **Số dòng index lớn:** 1 Observation sinh khoảng 5–15 dòng index rải trên nhiều bảng (code, status, category, subject, date, value…). Bảng `HFJ_SPIDX_TOKEN` của cả hệ thống thành bảng lớn nhất DB.
- **Mỗi điều kiện search là 1 JOIN/EXISTS:** `code=8480-6&value-quantity=gt140&date=ge2026-01-01` cần đi qua 3 bảng index khác nhau rồi giao kết quả.
- **Ghi chậm hơn:** 1 lần cập nhật phải so sánh và cập nhật toàn bộ dòng index cũ/mới.

**Điểm mạnh thật:**
- **Thêm search parameter mới không cần DDL.** Chỉ thêm dòng với `SP_NAME` mới, vì bảng index không có cột theo tên tham số. Đây là P7 ở mức tốt nhất trong 5 ứng viên.
- Bỏ dấu tiếng Việt khi search có sẵn (xem [Phụ lục](#phụ-lục--hệ-quả-riêng-cho-dự-án-ở-việt-nam)).

**Sai lầm thường gặp:** đánh giá HAPI là "chậm" chỉ vì nhiều JOIN. Thực tế các index đều được tối ưu với `HASH_IDENTITY` đứng đầu. Chi phí thật nằm ở **dung lượng và ghi**, không hẳn ở đọc.

### 2.5 Bảng rộng theo resource + lookup table — Medplum ✅

Chi tiết đầy đủ ở [`1a-fhir-datastore.md`](./1a-fhir-datastore.md). Tóm tắt để đặt cạnh 4 ứng viên trên:
- Mỗi loại resource có 1 bảng, gồm `content TEXT` (nguồn sự thật) và **1 hoặc vài cột cho mỗi search parameter**, sinh tự động (`migrations/migrate.ts:247-367`).
- Kiểu cột chọn theo kiểu tham số: range, token băm `UUID[]`, cột thường (`fhir/searchparameter.ts:116-183`).
- Riêng mảng có cấu trúc (tên, địa chỉ) và reference thì tách ra lookup table (`fhir/searchparameter.ts:205-211`).

**Dữ liệu** (rút gọn từ [`1a` mục 4](./1a-fhir-datastore.md#4-dữ-liệu-thật-trông-thế-nào)):

`"Observation"`, **1 dòng cho mỗi Observation**, mọi điều kiện search nằm trên cùng dòng:

| id | status | __code | valueQuantity | __valueQuantity | __date | subject | compartments |
|---|---|---|---|---|---|---|---|
| `b71a…` | final | `{h(loinc\|8480-6), h(\|8480-6), h(loinc\|)}` | 150 | `[150,150]` | `[2026-10-14…]` | `Patient/8f2c…` | `{8f2c…}` |
| `c22f…` | final | `{h(loinc\|4548-4), …}` | 8.2 | `[8.2,8.2]` | `[2026-10-14…]` | `Patient/8f2c…` | `{8f2c…}` |

`"HumanName"`: 2 dòng cho bà Lan (giống HAPI ở đúng chỗ này, xem Phần 8).

**Vấn đề cụ thể:**
- **Thêm search parameter = thêm cột = DDL**, kèm reindex (`Repository.VERSION`, `fhir/repo.ts:261-292`).
- **Bảng rất rộng:** bảng `Observation` có hàng chục cột. Phần lớn `NULL` với từng dòng cụ thể. Đây chính là đánh đổi "nhiều cột NULL" giống bài Saleor.
- **Token băm không đọc được bằng mắt.**

**Bảng tổng kết cả 5 mục, cùng câu hỏi "huyết áp 150 nằm ở đâu, kiểu gì":**

| Ứng viên | Huyết áp 150 nằm ở đâu | Kiểu | Lọc `> 140` có đúng & nhanh? |
|---|---|---|---|
| 2.1 Cột cố định (OpenEMR) | `form_vitals.bps` | `varchar(40)` | Đúng nếu nhớ `CAST`, không dùng index |
| 2.2 EAV chuỗi (LBF / Medplum v1) | `lbf_data.field_value` / `Observation.valueQuantity` | `LONGTEXT` / `TEXT` | **Sai** nếu so sánh chuỗi |
| 2.3 JSONB | Trong `resource->'valueQuantity'->'value'` | số JSON | Đúng, nhưng GIN không giúp so sánh khoảng |
| 2.4 HAPI | `HFJ_SPIDX_QUANTITY.SP_VALUE` (1 dòng riêng) | `Double` | Đúng, nhanh, qua 1 bảng index |
| 2.5 Medplum | `Observation.valueQuantity` + `__valueQuantity` (cùng dòng) | `DOUBLE PRECISION` + `NUMRANGE` | Đúng, nhanh, không JOIN |

---

## Phần 3 — Bảng so sánh trực diện

| Kỹ thuật | P1 nhiều loại, extension | P2 đúng kiểu, khoảng | P3 mảng có cấu trúc | P4 token | P5 lịch sử | P6 tenant | P7 đổi index không động dữ liệu | Đọc DB bằng mắt / BI |
|---|---|---|---|---|---|---|---|---|
| 2.1 Cột cố định | ✗ mỗi trường 1 cột | ◐ tuỳ cột | ✗ | ✗ phải thêm cột mã | ✗ tự làm (`log`) | ✗ tách DB | ✗ dữ liệu = cột | ✅ tốt nhất |
| 2.2 EAV chuỗi | ✅ | ✗ | ◐ | ✗ | ✗ | ◐ | ✗ | ◐ |
| 2.3 JSONB | ✅ | ◐ phải tự thêm expression index | ◐ | ◐ | tự làm | ✅ thêm cột | ◐ build lại index | ◐ |
| 2.4 HAPI | ✅ | ✅ | ✅ | ✅ | ✅ `HFJ_RES_VER` | ✅ `PARTITION_ID` | ✅ **không cần DDL** | ✗ phải JOIN nhiều bảng |
| 2.5 Medplum | ✅ | ✅ | ✅ lookup table | ✅ băm | ✅ `_History` | ✅ `projectId` | ✅ cần DDL + reindex | ✗ token băm |

Ký hiệu: ✅ thoả · ◐ thoả một phần · ✗ không thoả.

## Phần 4 — Ứng viên nào bị loại, ứng viên nào chỉ khác hướng

| Ứng viên | Kết luận cho bài toán "nền tảng FHIR nhiều tenant" | Đặc tính quyết định |
|---|---|---|
| 2.1 Cột cố định | **Loại** | P1, P3, P7. Đúng cho EHR 1 tổ chức, sai cho nền tảng FHIR |
| 2.2 EAV chuỗi | **Loại** | P2. Chính Medplum đã tự loại nó ở v18 |
| 2.3 JSONB thuần | **Loại cho search, giữ cho lưu trữ** | P2 ở quy mô lớn. Cả HAPI lẫn Medplum đều giữ ý tưởng "bản gốc nguyên khối" nhưng không search trên nó |
| 2.4 HAPI | **Không loại, khác hướng** | Thoả cả 7 đặc tính. Khác Medplum ở đánh đổi **dòng ↔ cột** |
| 2.5 Medplum | Thiết kế được chọn | Thoả cả 7 đặc tính |

🧠 Điểm khác cốt lõi với bài Saleor: ở bài Saleor, hybrid thắng tuyệt đối. Ở đây **có 2 người thắng** (HAPI và Medplum), và lựa chọn giữa 2 bên phụ thuộc vào **đánh đổi vận hành**, không phải đúng/sai. Xem Phần 7.

## Phần 5 — 4 lối tắt nghe hợp lý nhưng sai

Tự đoán câu in nghiêng trước khi đọc phần "Vì sao sai". 3 lối tắt đầu là dừng quá sớm, lối tắt D là leo quá xa.

**Lối tắt A — *"Lưu FHIR vào JSONB là xong, Postgres lo hết."***

*Câu hỏi nào sẽ lộ ra chưa đủ?*

**Vì sao sai:** hỏi "lấy các Observation có `effectiveDateTime` trong tháng 10/2026, kể cả bản ghi chỉ ghi `2026-10`" là thấy ngay. JSONB lưu đúng chuỗi `"2026-10"`, nhưng không biết chuỗi đó nghĩa là **cả tháng**. Phải tự viết hàm quy về khoảng, rồi tự tạo index trên hàm đó. Nghĩa là tự xây lại `range-column` của Medplum hoặc `SP_VALUE_LOW/HIGH` của HAPI.

**Lối tắt B — *"Làm bảng cột cố định cho 10 resource cần dùng trước, khi nào cần thêm thì thêm."***

**Vì sao sai:** dữ liệu từ đối tác (lab, bệnh viện khác) đến dưới dạng FHIR đầy đủ, có trường và extension mà 10 bảng không có chỗ chứa. Hoặc là **vứt dữ liệu** (sai pháp lý, mất khả năng trả lại nguyên vẹn), hoặc là thêm 1 cột JSON "phần còn lại". Đến lúc đó thì đã thành ứng viên 2.5 làm dở, vì nguồn sự thật bị chia đôi giữa cột và JSON.

**Lối tắt C — *"Token cứ lưu `system|code` dạng chuỗi trong cột TEXT[], cần gì băm."***

**Vì sao sai:** câu hỏi FHIR có 4 dạng (`code`, `system|code`, `system|`, `|code`). Lưu 1 chuỗi `system|code` thì 3 dạng còn lại phải dùng `LIKE`, không dùng index hiệu quả được. Medplum giải bằng cách sinh **nhiều token cho mỗi Coding**, mỗi dạng câu hỏi 1 token (`fhir/token-column.ts:84-111`). Băm UUID chỉ là bước làm gọn kích thước. Ý chính là **sinh trước mọi dạng câu hỏi**.

**Lối tắt D (leo quá xa) — *"Vậy MVP của mình cũng nên tự xây kiểu Medplum/HAPI cho chắc."***

**Vì sao sai:** cả 2 thiết kế tồn tại để phục vụ **khoảng 150 loại resource × nhiều tenant × nhiều năm**. MVP 1 phòng khám, 5–10 loại dữ liệu, chưa liên thông thì không có bằng chứng nào đòi hỏi mức đó. Lựa chọn hợp lý là **dùng** Medplum/HAPI làm backend, hoặc cột cố định nếu không cần FHIR. **Tự xây** lại datastore FHIR gần như không bao giờ đúng ở giai đoạn MVP.

## Phần 6 — Thiết kế thật của Medplum tại từng ngã rẽ

### 6.1 Tránh cột cố định → bảng và cột **sinh tự động từ định nghĩa SearchParameter**

✅ `migrations/migrate.ts:323-367`. Không ai viết tay `ALTER TABLE "Observation" ADD COLUMN …`. Generator đọc toàn bộ SearchParameter rồi sinh migration. Các file `migrations/schema/v*.ts` mở đầu bằng *"Generated by @medplum/generator"* (bản cũ) hoặc *"This is a generated file. Do not edit manually."* (bản mới, VD `v122.ts:3-6`). Medplum giữ được tốc độ đọc của cột cố định, mà không phải tự duy trì 150 bảng bằng tay.

### 6.2 Tránh EAV chuỗi → kiểu cột theo kiểu tham số

✅ `searchparameter.ts:127-136` (range), `migrate.ts:371-422` (bảng ánh xạ kiểu). Bài học v1 → v18 là lý do tồn tại của ngã rẽ này.

### 6.3 Tránh JSONB-làm-chỗ-search → `content` là TEXT và **không bao giờ bị query vào trong**

✅ `migrate.ts:252` (`content TEXT NOT NULL`). Mọi search đi qua cột chiếu. 🧠 Chọn `TEXT` thay vì `JSONB` khớp với nguyên tắc này: không cần toán tử JSON thì không cần trả chi phí chuẩn hoá JSONB khi ghi. Lý do chính thức vẫn là câu hỏi mở.

### 6.4 Khác HAPI → **1 dòng cho mỗi resource**, không phải 1 dòng cho mỗi giá trị

✅ So sánh trực tiếp:

| | HAPI | Medplum |
|---|---|---|
| Index token | `HFJ_SPIDX_TOKEN`: 1 dòng / (resource, tham số, giá trị) | Cột `__code UUID[]` trên chính dòng resource |
| Search 3 điều kiện | 3 bảng index, giao kết quả | 3 điều kiện `WHERE` trên 1 bảng |
| Thêm tham số | Không DDL | DDL + reindex |
| Index Postgres | B-tree với `HASH_IDENTITY` đứng đầu | GIN trên mảng, B-tree trên cột thường, GiST/range |

🧠 Medplum đổi **sự linh hoạt khi thêm tham số** (P7 phần "không DDL") lấy **truy vấn ít JOIN hơn**. Đổi được vì Medplum đã tự động hoá phần DDL (6.1) và reindex (`__version`). Cái giá còn lại chỉ là thời gian chạy migration.

### 6.5 Giống HAPI → **khoảng** cho date/number/quantity

✅ HAPI có `SP_VALUE_LOW/HIGH` (`ResourceIndexedSearchParamDate.java:91-99`) từ lâu. Medplum đi tới cùng ý tưởng qua `TSTZRANGE/NUMRANGE` ở version 15 (05/2026) và v105, và vẫn đang trong giai đoạn chuyển đổi (`repo.ts:2103`, `supportsRangeSearch`). 🧠 2 đội độc lập đi tới cùng 1 lời giải, đây là tín hiệu tốt cho thấy đó là đặc tính thật của bài toán (P2), không phải sở thích.

## Phần 7 — Ma trận đánh đổi, có số

|  | HAPI (dòng) | Medplum (cột) |
|---|---|---|
| Số dòng index cho 1 Observation | 💭 khoảng 8–15 dòng rải trên 3–5 bảng | 1 dòng (+ vài dòng `_References`) |
| Search nhiều điều kiện | Nhiều JOIN/EXISTS | Nhiều điều kiện trên 1 dòng |
| Thêm search parameter tuỳ biến | Ghi thêm dòng, không DDL | DDL tự sinh + reindex nền |
| Dung lượng index | Lớn: lặp `RES_ID`, `HASH_IDENTITY`, `PARTITION_ID` mỗi dòng | Nhỏ hơn mỗi resource, nhưng bảng rộng nhiều NULL |
| Chi phí NULL | Không có | 📘 Postgres lưu NULL bằng 1 bit trong null bitmap mỗi dòng → gần như 0 |
| Đọc 1 resource | `HFJ_RESOURCE` → `HFJ_RES_VER` (có thể giải nén `JSONC`) | Đọc thẳng `content` |
| Đa CSDL | Chạy trên nhiều DB (JPA/Hibernate) | Chỉ Postgres (dùng `UUID[]`, `TSTZRANGE`, GIN) |

💭 Ước tính với 36 triệu Observation (Phần 1):
- **HAPI:** khoảng 36M × 10 ≈ **360 triệu dòng** trong các bảng `HFJ_SPIDX_*`. Mỗi dòng mang ít nhất `RES_ID` (8 byte), `HASH_IDENTITY` (8 byte), `PARTITION_ID`, giá trị và các hash. Riêng phần lặp đã cỡ vài GB.
- **Medplum:** 36 triệu dòng trong bảng `Observation`. Chi phí thêm là các cột rộng và GIN trên mảng.

🧠 Đây không phải bên nào "rẻ hơn" tuyệt đối. Dòng cuối của bảng là khác biệt chiến lược: **HAPI chọn chạy được trên nhiều loại DB**, trả giá bằng mô hình EAV có phân kiểu (thứ chạy được ở mọi DB). **Medplum chọn gắn chặt Postgres** để dùng mảng, range, GIN, đổi lại được mô hình 1 dòng / resource. Một quyết định hạ tầng kéo theo cả kiến trúc lưu trữ.

## Phần 8 — Đối chứng nội bộ: Medplum dùng cách của HAPI ở đâu

Medplum **không** dùng "1 dòng / resource" cho mọi thứ. Có đúng 1 nhóm dữ liệu dùng mô hình "1 dòng / giá trị" giống HAPI: **lookup tables** (`fhir/searchparameter.ts:205-211`).

| Lookup table | Vì sao phải tách thành dòng |
|---|---|
| `HumanName` | Cần `LIKE '%Lan%'` trên **từng** tên, không phải trên cả mảng (`lookups/humanname.ts:26-31`, `lookups/lookuptable.ts:176-177`) |
| `Address` | Như trên, cho từng phần địa chỉ |
| `<Type>_References` | JOIN 2 chiều cho chained search (`fhir/search.ts:1689`) |

🧠 Quy tắc rút ra: Medplum chỉ tách thành dòng khi **phép so sánh cần làm trên từng phần tử với toán tử không phải "bằng"** (chứa chuỗi, JOIN). Với "bằng" trên mảng (token), mảng băm + GIN trên cùng dòng là đủ. Đây là bản "ShippingMethod" của module này: chọn kỹ thuật theo **toán tử cần hỗ trợ**, không theo sở thích.

## Phần 9 — Luyện phản biện

Tự viết câu trả lời trước khi mở gợi ý.

**Phản đối 1 — CTO của khách (quen MySQL/SQL Server):**
> *"Sao chọn nền tảng chỉ chạy Postgres? HAPI chạy được trên SQL Server mà team mình đang có."*

<details><summary>Gợi ý</summary>

Thừa nhận đúng: đây là đánh đổi có thật, dòng cuối Phần 7. Hỏi ngược: *"Team có định tự vận hành DB không, hay dùng dịch vụ hosted?"* Nếu dùng Medplum hosted thì loại DB không còn là chi phí của team. Nếu bắt buộc tự vận hành trên SQL Server sẵn có, HAPI là lựa chọn hợp lý hơn. Đó không phải thua kỹ thuật mà là khớp hạ tầng sẵn có, đúng tinh thần "không có ứng viên sai, chỉ có điều kiện khác" ở Phần 4.
</details>

**Phản đối 2 — Data analyst:**
> *"Tôi cần viết SQL báo cáo doanh thu, chỉ số chất lượng mỗi tuần. Token băm UUID thì tôi đọc kiểu gì?"*

<details><summary>Gợi ý</summary>

Không né: cả Medplum lẫn HAPI đều **không thiết kế cho SQL báo cáo trực tiếp**. Đề xuất đúng hướng: tách kho báo cáo. Dùng Bulk `$export` (NDJSON) hoặc SQL-on-FHIR (`packages/core/src/sql-on-fhir/`) đổ sang 1 kho phân tích có cột nghiệp vụ dễ đọc. Đây là mô hình chuẩn trong y tế: **CSDL giao dịch (OLTP) theo FHIR, kho phân tích (OLAP) theo nhu cầu báo cáo**, giống ý tưởng OMOP CDM.
</details>

**Phản đối 3 — Dev senior:**
> *"Bảng `Observation` hàng chục cột mà mỗi dòng dùng vài cột, vi phạm chuẩn hoá. HAPI sạch hơn."*

<details><summary>Gợi ý</summary>

Giống phản đối 3 của bài Saleor. NULL trên Postgres gần như miễn phí (null bitmap). "Sạch" theo chuẩn hoá phải đổi bằng 360 triệu dòng index và nhiều JOIN mỗi lần search (Phần 7). Chuẩn hoá là phương tiện, không phải mục tiêu. Và Medplum vẫn chuẩn hoá đúng chỗ cần (lookup tables, Phần 8).
</details>

## Phần 10 — Checklist cho dự án MVP

Đọc từ trên xuống, dừng ở dòng đầu tiên trả lời "Không":

| Câu hỏi | Không → | Có → |
|---|---|---|
| Sản phẩm có cần **trao đổi dữ liệu FHIR** với bên ngoài (lab, bảo hiểm, cơ quan quản lý) trong 12–18 tháng tới? | **Cột cố định** (2.1) là đủ và nhanh nhất. Thêm lớp xuất FHIR sau, như OpenEMR | (xuống) |
| Có đội đủ người vận hành 1 FHIR server riêng không? | **Dùng dịch vụ hosted** (Medplum cloud hoặc tương đương), không tự cài | (xuống) |
| Hạ tầng bắt buộc không phải Postgres? | **HAPI FHIR** | **Medplum** tự host |
| Cần form tự cấu hình cho từng phòng khám? | (bỏ qua) | Dùng `Questionnaire` + `QuestionnaireResponse` → `$extract` thành resource có kiểu. **Không** dùng EAV chuỗi (2.2) cho dữ liệu cần định lượng |
| Cần báo cáo SQL hằng tuần? | (bỏ qua) | Thiết kế kho phân tích riêng ngay từ đầu (Phản đối 2) |
| Trước mỗi bước "leo" ở trên: có **bằng chứng nghiệp vụ cụ thể** (hợp đồng, quy định, đối tác) hay chỉ đoán? | Dừng ở bước hiện tại | Leo đúng 1 bước |

## Phụ lục — Hệ quả riêng cho dự án ở Việt Nam

**Phát hiện: tìm tên không dấu.** Người Việt thường gõ "Nguyen Thi Lan" khi tìm "Nguyễn Thị Lan".

| Hệ thống | Hành vi | Nguồn |
|---|---|---|
| HAPI | Có cột `SP_VALUE_NORMALIZED` bỏ dấu + viết hoa. Tìm `name=nguyen` khớp "Nguyễn". **Nhưng** chỉ bỏ ký tự U+0300–036F sau NFD, mà `Đ`/`đ` không tách dấu được, nên "Đặng" thành "ĐANG": tìm `dang` **không** khớp | ✅ `StringUtil.java:42-70`, `ResourceIndexedSearchParamString.java:117` |
| Medplum, search tên (`HumanName`) | So khớp bằng `LIKE`/`ILIKE` (`lookups/lookuptable.ts:138,176-177`; `humanname.ts:60`), **không thấy bước bỏ dấu** | ✅ code. 🧠 Kết luận "không khớp" là suy luận từ code, **cần test thật** |
| Medplum, terminology `$expand` & ghép bệnh nhân | **Có** bỏ dấu: hàm SQL `medplum_unaccent` (`fhir/sql.ts:1394-1397`), `foldText` (`util/text.ts`, dùng ở `operations/expand.ts:19`, `operations/utils/cms-patient-match.ts:5`) | ✅ |

🧠 Hệ quả: nếu dùng Medplum cho thị trường Việt Nam, màn hình tìm bệnh nhân theo tên nhiều khả năng cần bổ sung, ví dụ thêm 1 search parameter tuỳ biến lưu tên không dấu, hoặc chuẩn hoá ở client. **Việc đầu tiên nên làm**: chạy thử `Patient?name=nguyen` trên 1 bản Medplum có bệnh nhân "Nguyễn".

## Tự luyện tiếp theo

1. Không nhìn Phần 3, tự điền lại 5 ô của cột P7 và giải thích bằng 1 câu mỗi ô.
2. Viết câu SQL HAPI (💭) cho `Observation?code=8480-6&value-quantity=gt140`, đếm số bảng phải chạm. Rồi so với SQL Medplum ở [`1a` mục 4](./1a-fhir-datastore.md#4-dữ-liệu-thật-trông-thế-nào).
3. Áp checklist Phần 10 cho dự án MVP thật của bạn. Ghi lại dừng ở dòng nào và bằng chứng gì.

## Nguồn

- [`1a-fhir-datastore.md`](./1a-fhir-datastore.md): nguồn ✅ cho mọi chi tiết Medplum.
- Medplum `16cb130`:
  - `packages/server/src/migrations/migrate.ts`, `migrations/schema/v1.ts`, `v18.ts`, `v105.ts`
  - `packages/server/src/fhir/searchparameter.ts`, `token-column.ts`, `lookups/*`, `sql.ts`
  - `packages/server/src/util/text.ts`
- OpenEMR `4c1eb8d`:
  - `sql/database.sql` (`patient_data`, `form_vitals`, `lbf_data`, `procedure_*`)
  - `src/Services/FHIR/Observation/FhirObservationVitalsService.php`
- HAPI FHIR `6d818ea`:
  - `hapi-fhir-jpaserver-model/.../entity/ResourceTable.java`, `ResourceHistoryTable.java`, `ResourceEncodingEnum.java`
  - `hapi-fhir-jpaserver-model/.../entity/BaseResourceIndexedSearchParam.java`, `ResourceIndexedSearchParam{String,Token,Date,Quantity}.java`, `ResourceLink.java`
  - `hapi-fhir-base/src/main/java/ca/uhn/fhir/util/StringUtil.java`
