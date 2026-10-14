# 4d — Đo lường chất lượng: `Measure/$evaluate-measure`

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8`.
>
> Trả lời câu: *"Payer trả thêm tiền nếu ≥ 80% bệnh nhân tiểu đường của An Bình có HbA1c < 9%. Hệ thống tính tỉ lệ đó thế nào, và có tin được con số không?"*

## Mục lục

- [1. Góc nhìn kinh doanh: trả theo giá trị](#1-góc-nhìn-kinh-doanh-trả-theo-giá-trị)
- [2. Measure và MeasureReport](#2-measure-và-measurereport)
- [3. `$evaluate-measure` của Medplum: đếm bằng search](#3-evaluate-measure-của-medplum-đếm-bằng-search)
- [4. 4 giới hạn khiến con số không dùng được cho hợp đồng](#4-4-giới-hạn-khiến-con-số-không-dùng-được-cho-hợp-đồng)
- [5. Đối chiếu OpenEMR: engine CQL đầy đủ](#5-đối-chiếu-openemr-engine-cql-đầy-đủ)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Góc nhìn kinh doanh: trả theo giá trị

📘/🧠 Mỹ đang chuyển dần từ **trả theo dịch vụ** (fee-for-service: làm nhiều thu nhiều) sang **trả theo giá trị** (value-based care): 1 phần tiền phụ thuộc **chỉ số chất lượng**.

| Mô hình | Tiền phụ thuộc gì | Vai trò của đo lường |
|---|---|---|
| Fee-for-service | Số dịch vụ | Báo cáo, không ảnh hưởng tiền |
| Thưởng chất lượng (pay-for-performance) | Dịch vụ + thưởng nếu đạt chỉ số | Quyết định **khoản thưởng** |
| Chia sẻ tiết kiệm (shared savings, ACO) | Chi phí thấp hơn mốc **và** đạt chất lượng | Điều kiện để được chia |
| Trả theo đầu người (capitation) | Số bệnh nhân quản lý | Phạt nếu chất lượng thấp |

📘 Chỉ số chuẩn ở Mỹ: **HEDIS** (do NCQA quản lý, payer dùng), **eCQM** (CMS dùng, viết bằng CQL). VD "HbA1c Poor Control (> 9%)" cho bệnh nhân tiểu đường 18–75 tuổi.

🧠 Với người làm tài chính: đo lường chất lượng là **điều khoản hợp đồng có tiền**. Con số phải tái lập được, kiểm toán được, và đúng định nghĩa của payer tới từng điều kiện loại trừ.

## 2. Measure và MeasureReport

| Resource | Vai trò | Trường chính |
|---|---|---|
| `Measure` | **Định nghĩa** chỉ số | `group[].population[]`: `initial-population`, `denominator`, `denominator-exclusion`, `numerator`, `numerator-exclusion`… mỗi cái có `criteria` (biểu thức) |
| `MeasureReport` | **Kết quả** cho 1 khoảng thời gian | `type`: `individual` (1 bệnh nhân), `subject-list` (danh sách), `summary` (đếm); `group[].population[].count`, `measureScore` |

📘 FHIR mặc định `criteria` viết bằng **CQL** (Clinical Quality Language). Lớp quyết định: HL7 + chương trình đo lường (CMS, NCQA).

## 3. `$evaluate-measure` của Medplum: đếm bằng search

✅ `packages/server/src/fhir/operations/evaluatemeasure.ts`:

| Bước | ✅ |
|---|---|
| Đầu vào: `periodStart`, `periodEnd` | `:20-23` |
| Tạo `MeasureReport` `status: complete`, **`type: summary`** | `:66-76` |
| Với mỗi `group`, mỗi `population`: nếu có `criteria.expression` thì đếm | `:78-80`, `:101-103`, `:123-125` |
| **Đếm** = `parseSearchRequest(criteria)` (criteria là **chuỗi search FHIR**, không phải CQL) + `total: accurate` | `:141-142` |
| Thêm điều kiện **`_lastUpdated`** trong khoảng thời gian | `:144-156` |
| Lưu MeasureReport | `:82` |

💭 Measure cho bài tiểu đường:

| population | criteria (search FHIR) |
|---|---|
| denominator | `Condition?code=http://hl7.org/fhir/sid/icd-10-cm\|E11.9` |
| numerator | `Observation?code=http://loinc.org\|4548-4&value-quantity=lt9` |

## 4. 4 giới hạn khiến con số không dùng được cho hợp đồng

🧠 Suy ra trực tiếp từ code ở mục 3:

| # | Giới hạn | Hệ quả với kịch bản 💭 |
|---|---|---|
| 1 | **Đếm resource, không đếm bệnh nhân** | Bà Lan có 4 Observation HbA1c < 9 trong năm → tử số +4. Tỉ lệ có thể > 100% |
| 2 | **Lọc theo `_lastUpdated`** (lúc bản ghi được sửa), không theo ngày lâm sàng | Sửa lỗi chính tả 1 Condition năm 2023 hôm nay → nó vào mẫu số năm nay. Ngược lại kết quả xét nghiệm tháng 12 nhập muộn sang tháng 1 thì rơi khỏi kỳ |
| 3 | **Tử số không phải tập con của mẫu số** | 2 câu search độc lập. HbA1c của bệnh nhân **không** tiểu đường cũng vào tử số |
| 4 | **Không có loại trừ, không có CQL**, chỉ `summary` | Không áp được "18–75 tuổi", "loại bệnh nhân chăm sóc cuối đời". Không ra được danh sách bệnh nhân cần can thiệp |

🧠 Đây là hiện thực **tối thiểu** thứ 3 trong Giai đoạn 3–4 (cùng `$apply` của PlanDefinition [3d](./3d-careplan-task.md#3-apply-không-làm-gì) và ChargeItemDefinition [4b](./4b-dinh-gia-ghi-phi.md#4-những-gì-apply-không-làm)). Mẫu chung: Medplum hiện thực **giao diện chuẩn** của các operation "tri thức lâm sàng" ở mức đơn giản nhất, phần logic thật để cho ứng dụng hoặc công cụ chuyên dụng.

Cách làm đúng trên Medplum:
- Bulk `$export` ([2d](./2d-interoperability.md)) dữ liệu ra kho phân tích, chạy CQL/SQL ở đó.
- Hoặc Bot cron tính theo từng bệnh nhân (đúng định nghĩa), ghi `MeasureReport` `type: individual` + `subject-list`, rồi tạo `Task` cho bệnh nhân chưa đạt ([3d](./3d-careplan-task.md)). Đây là cách biến chỉ số thành việc làm hằng ngày.

## 5. Đối chiếu OpenEMR: engine CQL đầy đủ

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Ngôn ngữ tiêu chí | Chuỗi search FHIR | **CQL** (biên dịch sang ELM), mô hình dữ liệu **QDM** |
| Engine | Search + đếm | Engine CQL viết bằng PHP: `src/Cqm/Cql/` (`Elm`, `Engine`, `Types`, `Ucum`…), `PhpCqmCalculation.php` (tính từ payload trước đây gửi cho dịch vụ `cqm-execution`) |
| Đơn vị đếm | Resource | Bệnh nhân (QDM patient) |
| Báo cáo nộp CMS | Không | **QRDA** (`src/Cqm/QrdaControllers/QrdaReportController.php`) |
| Chỉ số "Meaningful Use" | — | `src/ClinicalDecisionRules/AMC/` |

🧠 OpenEMR phải có engine CQL đầy đủ vì **chứng nhận ONC** cho EHR yêu cầu tính và nộp eCQM. Medplum là nền tảng, chứng nhận các tiêu chí khác ([2d](./2d-interoperability.md) liệt kê), không làm phần này.

## Sai lầm thường gặp

1. **Dùng `$evaluate-measure` để báo cáo cho payer.** Đếm resource, lọc theo ngày sửa: không khớp định nghĩa HEDIS/eCQM.
2. **Lọc kỳ theo `_lastUpdated`.** Phải theo ngày lâm sàng (`Observation.effective`, `Condition.onset`), xem `caredate` ở [3c](./3c-charting-encounter.md#5-tóm-tắt-bệnh-nhân-resource-nào-vào-mục-nào).
3. **Tính tử số độc lập với mẫu số.**
4. **Chỉ có báo cáo tổng.** Không có danh sách bệnh nhân chưa đạt thì không ai can thiệp được trước cuối kỳ.
