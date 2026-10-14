# 5d — Tổng hợp chiến lược: Medplum đứng ở đâu, kiếm tiền bằng gì, dùng cho MVP thế nào

> **Loại file:** bài phân tích, không theo template. Tổng hợp từ 0a–5c.
>
> Trả lời câu: *"Sau khi đọc hết code, 1 người làm chiến lược nên hiểu Medplum là sản phẩm gì, lợi thế nằm ở đâu, rủi ro là gì, và có nên dựng MVP y tế trên nó không?"*
>
> **Quy tắc nguồn:** nhận định về giá, gọi vốn, đối thủ đều có link. Không có link thì ghi 🧠 và coi là giả thuyết. Tra cứu 10/2026.

## Mục lục

- [1. Kết luận trong 5 dòng](#1-kết-luận-trong-5-dòng)
- [2. Vị trí trong chuỗi giá trị](#2-vị-trí-trong-chuỗi-giá-trị)
- [3. Medplum làm gì, để gì cho người khác: tổng kết 30 module](#3-medplum-làm-gì-để-gì-cho-người-khác-tổng-kết-30-module)
- [4. Mô hình kinh doanh open-core](#4-mô-hình-kinh-doanh-open-core)
- [5. Lợi thế cạnh tranh và độ bền](#5-lợi-thế-cạnh-tranh-và-độ-bền)
- [6. Đối thủ và thay thế](#6-đối-thủ-và-thay-thế)
- [7. Rủi ro](#7-rủi-ro)
- [8. Áp dụng cho MVP: cây quyết định](#8-áp-dụng-cho-mvp-cây-quyết-định)
- [9. Câu hỏi còn mở](#9-câu-hỏi-còn-mở)

Nhãn: ✅ code (dẫn qua file module) · 📘 nguồn công khai có link · 🧠 suy luận.

---

## 1. Kết luận trong 5 dòng

1. Medplum là **hạ tầng dữ liệu y tế** (FHIR server + phân quyền + tự động hoá), không phải phần mềm phòng khám. Nó bán cho **đội kỹ sư** xây sản phẩm y tế số, không bán cho bác sĩ.
2. Lõi kỹ thuật **rất kỹ** (lưu trữ, search, phân quyền, đặt lịch chống trùng, ghép hồ sơ theo CMS). Lớp nghiệp vụ **cố ý mỏng** (`$apply`, `$evaluate-measure`, `$submit` đều là hiện thực tối thiểu hoặc chỉ là bộ điều phối).
3. Kiếm tiền theo mô hình **open-core + hosted**: mã nguồn mở Apache 2.0, thu tiền từ vận hành, tuân thủ, tích hợp đối tác.
4. Lợi thế bền nhất không nằm ở code mà ở **chứng nhận và vận hành** (ONC, SOC 2, HITRUST, HIPAA BAA) và **mạng tích hợp đối tác** Mỹ.
5. Cho MVP ở **Việt Nam**: dùng được phần lõi, nhưng mọi thứ làm nên giá trị ở Mỹ (đối tác, chứng nhận, revenue cycle) **không mang theo được**. Phải tự xây lớp BHYT, đơn thuốc quốc gia, ký số, và tự vận hành vì yêu cầu dữ liệu trong nước.

## 2. Vị trí trong chuỗi giá trị

🧠 Chuỗi giá trị phần mềm y tế, từ dưới lên:

```
[6] Dịch vụ y tế (phòng khám, telehealth, chăm sóc mạn tính)        ← khách hàng của khách hàng Medplum
[5] Ứng dụng: app bệnh nhân, màn hình bác sĩ, lịch, billing UI      ← khách hàng Medplum tự xây (có component React mẫu)
[4] Logic nghiệp vụ: phác đồ, định giá, claim, ghép hồ sơ           ← Bot của khách hàng + đối tác
[3] Tích hợp: lab, eRx, clearinghouse, fax, HL7 v2 bệnh viện        ← Medplum có package/Bot mẫu, đối tác làm phần chính
[2] Nền tảng dữ liệu: FHIR API, phân quyền, subscription, bot runtime ← MEDPLUM
[1] Hạ tầng: Postgres, Redis, cloud, chứng nhận bảo mật               ← MEDPLUM (bản hosted)
```

| Loại sản phẩm | Lớp phủ | Ví dụ |
|---|---|---|
| EHR truyền thống | 1 → 6 trọn gói, đóng | Epic, Cerner/Oracle Health |
| EHR mã nguồn mở | 1 → 5, cho phòng khám dùng ngay | OpenEMR (✅ có cả billing X12, CQL, kho thuốc) |
| FHIR server thuần | 2 | HAPI FHIR |
| **Headless EHR / developer platform** | **1 → 3** | **Medplum**, Oystehr, Canvas (có thêm UI) |
| Phần mềm chuyên khâu | 3–4 cho 1 khâu | Candid (billing), DoseSpot (eRx), Health Gorilla (lab) |

🧠 Medplum chọn đứng ở **lớp 2** và đẩy giá trị lên lớp 1 (hosted, chứng nhận), để lớp 3–4 cho đối tác. Đây là định vị "AWS của y tế số" hơn là "Shopify của phòng khám".

## 3. Medplum làm gì, để gì cho người khác: tổng kết 30 module

| Mức | Module | Bằng chứng |
|---|---|---|
| **Làm kỹ, tốt hơn đối chứng** | Lưu trữ + search ([1a](./1a-fhir-datastore.md), [1b](./1b-search.md)) | Bảng rộng + lookup, tiến hoá TEXT → RANGE nhờ reindex |
| | Phân quyền policy-as-data ([1c](./1c-access-control.md)) | Compartment + AccessPolicy, tenant-aware |
| | Đặt lịch chống trùng ([3b](./3b-dat-lich.md)) | SERIALIZABLE + test song song thật |
| | Ghép hồ sơ ([3a](./3a-dinh-danh-tiep-nhan.md)) | Khung CMS v3.4.0, audit từng quyết định |
| | Migration không dừng dịch vụ ([5b](./5b-self-host-sharding-migration.md)) | 2 pha + manifest |
| | Di chuyển tenant ([5c](./5c-vong-doi-project.md)) | `$archive`/`$restore` giữ lịch sử |
| **Làm tối thiểu, cố ý** | Phác đồ `$apply` ([3d](./3d-careplan-task.md)) | Bỏ qua condition, timing, lồng nhau, không giao dịch |
| | Định giá `$apply` ([4b](./4b-dinh-gia-ghi-phi.md)) | Không nhân số lượng, không hiệu lực, điều kiện cuối ghi đè |
| | Đo chất lượng ([4d](./4d-do-luong-chat-luong.md)) | Đếm resource, lọc theo `_lastUpdated` |
| | DICOMweb ([3f](./3f-xet-nghiem-hinh-anh.md)) | Chỉ 1 phần chuẩn |
| **Chỉ điều phối, đối tác làm** | Kiểm tra quyền lợi, claim, prior auth ([4a](./4a-quyen-loi-bao-hiem.md), [4c](./4c-claim.md)) | `$submit` → setting → OperationDefinition → Bot |
| | eRx, lab, fax ([3e](./3e-thuoc-ke-don.md), [3f](./3f-xet-nghiem-hinh-anh.md), [3h](./3h-giao-tiep.md)) | Package + Bot mẫu, fax là dịch vụ hosted |
| **Ghi nhận, không thực thi** | Consent ([3i](./3i-consent.md)), ký hồ sơ ([3c](./3c-charting-encounter.md)) | Consent không chặn truy cập; "khoá" chỉ ở giao diện app mẫu |
| **Không có** | Sổ công nợ, đọc 835 ([4c](./4c-claim.md)), kho thuốc ([3e](./3e-thuoc-ke-don.md)), engine CQL ([4d](./4d-do-luong-chat-luong.md)) | OpenEMR có cả 3 |

🧠 Đường ranh rất nhất quán: **thứ gì là dữ liệu và bất biến hệ thống → Medplum làm kỹ. Thứ gì là chính sách kinh doanh hoặc quy định riêng 1 thị trường → để cho khách hàng/đối tác.** Mẫu này hiện ra ở mọi giai đoạn: `$hold` không có TTL vì TTL là chính sách; `$submit` không gửi X12 vì chọn clearinghouse là quyết định kinh doanh; `ChargeItemDefinition` tối giản vì hợp đồng payer là sản phẩm của người khác.

## 4. Mô hình kinh doanh open-core

📘 Bảng giá công khai ([medplum.com/pricing](https://www.medplum.com/pricing), số liệu đối chiếu thêm ở [healthcarediscovery.ai](https://healthcarediscovery.ai/companies/medplum/)):

| Gói | Giá | Đặc điểm nổi bật |
|---|---|---|
| Free (hosted) | 0 | Có BAA, MSA chuẩn. **Không bật Bot và email** |
| Production (hosted) | 2.000$/tháng | Dùng thật cho bệnh nhân. **Bật Bot**. AWS Marketplace ghi 24.000$/12 tháng + phí dùng thêm ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-gfbi35l2l7mma)) |
| Premium (hosted) | 6.000$/tháng | Thêm tích hợp chẩn đoán, lab, thuốc. Hợp với nhắn tin, tích hợp nhiều |
| Enterprise (hosted) | Liên hệ | Quy trình, tích hợp, dữ liệu phức tạp |
| Community (self-host) | 0 | Tự vận hành |
| Enterprise (self-host) | Liên hệ | |

🧠 Đọc bảng giá theo những gì đã học trong code:
1. **Cổng thu tiền đầu tiên là Bot.** Free không có Bot, mà Bot là nơi chứa **mọi** logic nghiệp vụ ([2b](./2b-bots.md)): tiếp nhận, `$submit` eligibility/claim ([4a](./4a-quyen-loi-bao-hiem.md#4-submit-bộ-điều-phối-không-phải-bộ-gửi)), cron phác đồ ([3d](./3d-careplan-task.md)). Không có Bot thì Medplum chỉ là kho FHIR. Đây là chỗ đặt "paywall" rất chính xác.
2. **Cổng thứ 2 là tích hợp đối tác** (lab, eRx). Đúng những package `health-gorilla-*`, `dosespot-*` trong repo: mã mở, nhưng kết nối thật cần hợp đồng và cấu hình phía Medplum.
3. **Đơn vị đo tải** ([5a](./5a-gioi-han-do-luong.md)): request, điểm FHIR (đọc 1, tìm 20, ghi 100), số resource. Phí "dùng thêm" trên AWS Marketplace nhiều khả năng đo bằng các đơn vị này (🧠, chưa thấy tài liệu công khai mô tả công thức).
4. **Self-host miễn phí nhưng docs chủ động thuyết phục không nên** ([5b](./5b-self-host-sharding-migration.md#2-medplum-tự-nói-gì-về-self-host)). Mã mở là kênh **phân phối và tạo niềm tin**, bản hosted là kênh **thu tiền**.

📘 Gọi vốn: Medplum là công ty Y Combinator (khoá S22). Nguồn công khai về các vòng gọi vốn **không thống nhất** (Dealroom ghi seed 125.000$, có trang ghi tổng 550.000$, không tìm thấy thông cáo vòng lớn) ([Dealroom](https://app.dealroom.co/companies/medplum), [Caplight](https://www.caplight.com/company/medplum), [Extruct](https://www.extruct.ai/hub/medplum-com-funding/)). 🧠 Không dùng số gọi vốn trong phân tích cho tới khi có nguồn gốc.

## 5. Lợi thế cạnh tranh và độ bền

| Lợi thế | Độ bền 🧠 | Vì sao |
|---|---|---|
| Code lõi | **Thấp** | Apache 2.0, ai cũng fork được |
| Chất lượng kỹ thuật lõi (search, phân quyền, lịch) | Trung bình | Tốn nhiều năm để làm lại, nhưng không phải bí mật |
| **Chứng nhận & tuân thủ** | **Cao** | 📘 ONC (danh sách tiêu chí ở [2d](./2d-interoperability.md)), SOC 2 Type II, HITRUST, HIPAA (`packages/docs/docs/compliance/`). Fork không kế thừa được chứng nhận. Khách hàng hosted "thừa hưởng" chứng nhận |
| **Vận hành hosted** | Cao | Docs tự nhận: "We built the system… We operate one well-understood fleet" |
| **Mạng đối tác tích hợp** (Health Gorilla, DoseSpot, ScriptSure, Candid, Stedi, eFax, Opkit…) | Trung bình–cao | Mỗi tích hợp là 1 hợp đồng + kiểm thử + hỗ trợ |
| Hệ sinh thái nhà phát triển (TypeScript, React component, ~25 ví dụ) | Trung bình | Chi phí chuyển đổi cho đội đã viết hàng trăm Bot |
| **Chi phí chuyển đổi của khách** | Trung bình | Dữ liệu dễ mang đi (`$export`), **logic** (Bot, AccessPolicy) khó mang đi ([5c mục 5](./5c-vong-doi-project.md#5-ý-nghĩa-kinh-doanh-chi-phí-chuyển-đổi)) |

🧠 Đòn bẩy của open-core ở đây: **mã mở hạ chi phí bán hàng** (kỹ sư tự thử, tự đánh giá, tự thuyết phục sếp), **chứng nhận + vận hành giữ biên lợi nhuận**. Rủi ro kinh điển: 1 bên lớn (cloud provider, consultancy) host bản mở và cạnh tranh giá. Rào cản chống lại là chứng nhận gắn với **đơn vị vận hành**, không gắn với code.

## 6. Đối thủ và thay thế

📘 Các bài so sánh công khai hiện có phần lớn do **công ty dịch vụ phần mềm** viết (có động cơ riêng), nên chỉ dùng để liệt kê, không dùng để kết luận ([Mindbowser: Headless EHR comparison](https://www.mindbowser.com/headless-ehr-comparison/), [Mindbowser: Medplum vs Healthie vs custom](https://www.mindbowser.com/ehr-for-digital-health-startups-medplum-healthie-or-custom/), [CB Insights: Canvas alternatives](https://www.cbinsights.com/company/canvas-medical/alternatives-competitors)):

| Thay thế | Định vị (theo nguồn trên) | So với Medplum 🧠 |
|---|---|---|
| **Canvas Medical** | EHR chạy được ngay + lớp tuỳ biến (Python SDK) | Nhiều nghiệp vụ hơn, ít "headless" hơn. Hợp khi muốn có UI bác sĩ ngay |
| **Healthie** | Wellness, đa chuyên khoa, GraphQL API, giá theo số provider | Hợp dịch vụ dinh dưỡng, tâm lý. Ít linh hoạt mô hình dữ liệu |
| **Oystehr** | Headless EHR | Cạnh tranh trực tiếp nhất |
| **OpenEMR** | EHR mã nguồn mở trọn gói (✅ đã đọc code) | Mạnh billing Mỹ, CQL, kho thuốc. Yếu kiến trúc đa tenant, API-first |
| **HAPI FHIR** (tự host) | FHIR server Java | Chỉ lớp 2, không có Bot runtime, lịch, ghép hồ sơ ([1a-C](./1a-comparison-fhir-datastore.md)) |
| **Tự xây** | — | Toàn quyền, tốn nhiều năm cho lõi mà Medplum đã có |

## 7. Rủi ro

| Rủi ro | Cho ai | Dấu hiệu trong code/docs |
|---|---|---|
| **API beta thay đổi** | Người dùng | Đặt lịch gắn `@experimental - Scheduling Beta API` ([3b](./3b-dat-lich.md)), có `beta-changelog.md` |
| **Lớp nghiệp vụ mỏng hơn docs gợi ý** | Người dùng | Docs mô tả cách khai báo `condition` trong PlanDefinition, server không đánh giá ([3d](./3d-careplan-task.md#3-apply-không-làm-gì)) |
| **Phụ thuộc Postgres** | Người dùng self-host | Kiến trúc gắn chặt Postgres ([CLAUDE.md phát hiện 8](../CLAUDE.md#phát-hiện-quan-trọng-đến-nay)) |
| **Mất sự kiện subscription** | Người dùng | Enqueue sau commit, lỗi chỉ log ([2a](./2a-subscriptions.md)) |
| **Gắn với thị trường Mỹ** | Medplum, người dùng ngoài Mỹ | Mã định danh CMS, CPT, X12, CMS-1500 trong core; tên "Đặng" thành "ang" khi ghép hồ sơ ([3a](./3a-dinh-danh-tiep-nhan.md#7-tiếng-việt-dấu-và-chữ-đ)) |
| **Đối tác là điểm đơn** | Người dùng | eFax cần đội Medplum bật ([3h](./3h-giao-tiep.md)); tích hợp đối tác theo gói giá |
| **Mô hình open-core bị cạnh tranh giá** | Medplum | 🧠 Như trên |

## 8. Áp dụng cho MVP: cây quyết định

Dự án MVP của người dùng **chưa xác định** (loại hình, thị trường, vai trò). Cây dưới đây dùng khi đã có câu trả lời.

```
Thị trường?
├── Mỹ
│   ├── Cần UI bác sĩ ngay, ít kỹ sư ──────────────▶ Canvas / Healthie (mua, không xây)
│   ├── Sản phẩm số khác biệt (telehealth, chăm sóc từ xa, chuyên khoa hẹp)
│   │   └── Có đội TypeScript ────────────────────▶ Medplum hosted (Production → Premium)
│   │                                                + Candid/Stedi cho billing, DoseSpot cho eRx
│   └── Phòng khám truyền thống cần billing đầy đủ ──▶ OpenEMR hoặc EHR thương mại
└── Việt Nam
    ├── Có BHYT ────▶ Lõi phải khớp QĐ 130/4750 + giám định ([4e](./4e-phu-luc-bhyt-viet-nam.md))
    │   ├── Thời gian ra thị trường quan trọng nhất ──▶ HIS trong nước sẵn có, tích hợp thêm
    │   └── Muốn kiến trúc chuẩn FHIR, nhiều tenant ──▶ Medplum self-host trong nước
    │                                                  + tự xây: Bot XML BHYT, đơn thuốc quốc gia,
    │                                                    ký số, chuẩn hoá tên tiếng Việt, kho thuốc
    └── Không BHYT (dịch vụ, thẩm mỹ, nha, dinh dưỡng, app bệnh nhân)
        └── Medplum self-host là phù hợp nhất: lõi lịch, phân quyền, Bot dùng được ngay
```

🧠 Ước lượng phần **phải tự xây** nếu chọn Medplum cho phòng khám có BHYT ở Việt Nam (theo các file đã đọc):

| Hạng mục | Module tham chiếu | Mức |
|---|---|---|
| Vận hành self-host (Postgres, Redis, nâng cấp, trực) | [5b](./5b-self-host-sharding-migration.md) | Liên tục |
| Bot dựng và gửi XML BHYT, xử lý xuất toán | [4e](./4e-phu-luc-bhyt-viet-nam.md), [4c](./4c-claim.md) | Lớn nhất |
| Danh mục mã Việt Nam (DVKT, thuốc, vật tư) | [1e](./1e-terminology.md) | Lớn, cập nhật liên tục |
| Bảng giá BHYT + giá dịch vụ, nhân số lượng, hiệu lực | [4b](./4b-dinh-gia-ghi-phi.md) | Trung bình |
| Ghép hồ sơ tiếng Việt (Đ, SĐT), CCCD | [3a](./3a-dinh-danh-tiep-nhan.md) | Nhỏ–trung bình |
| Ký số bệnh án, khoá hồ sơ thật | [3c](./3c-charting-encounter.md), [3i](./3i-consent.md) | Trung bình |
| Đơn thuốc điện tử quốc gia, kho thuốc | [3e](./3e-thuoc-ke-don.md) | Trung bình |

## 9. Câu hỏi còn mở

1. **Thông tin dự án MVP**: loại hình, thị trường, vai trò của người dùng (đã ghi từ phiên trước, vẫn chưa có).
2. Công thức tính "phí dùng thêm" trên bản hosted: chưa thấy tài liệu công khai.
3. Số vòng gọi vốn, doanh thu, số khách hàng: nguồn công khai không thống nhất. Cần nguồn gốc (thông cáo, hồ sơ) trước khi dùng.
4. Mức hưởng BHYT sau Luật 51/2024/QH15 (hiệu lực 01/7/2025) và cấu trúc cột XML mới nhất: cần văn bản gốc ([4e](./4e-phu-luc-bhyt-viet-nam.md#3-văn-bản-cần-đọc)).
5. Nên thử thật: `Patient?name=nguyen` có khớp "Nguyễn" không, và `$match` với họ có chữ Đ ([3a](./3a-dinh-danh-tiep-nhan.md#7-tiếng-việt-dấu-và-chữ-đ)). Cần chạy server với Postgres + Redis.
