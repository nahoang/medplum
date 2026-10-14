# 3a — Định danh & tiếp nhận bệnh nhân: `$match`, `$extract`, gộp hồ sơ

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `ebe3ee8` (bản clone mới, khác `4c1eb8d` của các file trước).
>
> Trả lời câu: *"Bệnh nhân điền phiếu ở quầy. Hệ thống biến phiếu thành hồ sơ thế nào, nhận ra người cũ thế nào, và khi trùng thì gộp thế nào?"* Thiết kế từ đầu: [`3a-greenfield-dinh-danh-benh-nhan.md`](./3a-greenfield-dinh-danh-benh-nhan.md).
>
> Đường dẫn Medplum tính từ `packages/server/src/` nếu không ghi khác.

## Mục lục

- [1. 3 việc tách rời nhau](#1-3-việc-tách-rời-nhau)
- [2. `$match`: thu ứng viên](#2-match-thu-ứng-viên)
- [3. `$match`: so 12 trường theo khung CMS](#3-match-so-12-trường-theo-khung-cms)
- [4. `$match`: chấm điểm, phân tầng, chặn cứng](#4-match-chấm-điểm-phân-tầng-chặn-cứng)
- [5. Chế độ "chỉ trả kết quả chắc chắn" và audit](#5-chế-độ-chỉ-trả-kết-quả-chắc-chắn-và-audit)
- [6. Kịch bản bà Lan chạy qua `$match`](#6-kịch-bản-bà-lan-chạy-qua-match)
- [7. Tiếng Việt: dấu và chữ Đ](#7-tiếng-việt-dấu-và-chữ-đ)
- [8. `$extract`: phiếu → hồ sơ](#8-extract-phiếu--hồ-sơ)
- [9. Gộp hồ sơ: không có trong server](#9-gộp-hồ-sơ-không-có-trong-server)
- [10. Đối chiếu OpenEMR](#10-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 3 việc tách rời nhau

| Việc | Ai làm | Có trong server Medplum? |
|---|---|---|
| **Tiếp nhận**: phiếu (`QuestionnaireResponse`) → `Patient`, `Coverage`, `AllergyIntolerance`… | `$extract` (sinh Bundle, **không lưu**) hoặc Bot tiếp nhận (tự lưu) | Có `$extract`. Bot là của khách hàng |
| **Tìm trùng**: Patient mới có giống ai đã có không | `Patient/$match` | Có |
| **Gộp**: nối 2 bản ghi, chuyển tham chiếu | Bot của khách hàng | **Không** |

🧠 Đây là đúng triết lý "headless" ([00](./00-tong-quan-medplum.md)): server cung cấp **phép đo** (match), còn **quyết định** (gộp hay không) để ứng dụng. Lý do: chi phí ghép nhầm khác nhau theo ứng dụng (📘 `packages/docs/docs/fhir-datastore/patient-deduplication/architecture-overview.md`, mục "Cost of false positives" và "Downstream application").

## 2. `$match`: thu ứng viên

✅ `fhir/operations/patientmatch.ts:75-94` (`patientMatchHandler`):
- Đầu vào phải là `Patient` (`:79-81`).
- Phải có ít nhất 1 trong: identifier, name, birthDate, telecom (`:82-86`, `:359-365`).

Thu ứng viên bằng **tối đa 3 nhóm search FHIR thường** (`gatherCandidates`, `:140-200`). Không so với cả bảng:

| Chiến lược | Search | ✅ |
|---|---|---|
| 1. Định danh | `identifier=system\|value` cho **từng** identifier | `:165-171` |
| 2. Liên lạc | `telecom=<giá trị>` cho từng phone/email | `:173-181` |
| 3. Tên + ngày sinh | `birthdate` trong ±1 ngày **và** (`family=` hoặc `given=`) | `:183-194`, khoảng ngày ở `:327-349` |

- Mỗi search lấy tối đa `CANDIDATE_SEARCH_COUNT = 100` (`:41`). Chạm trần thì đánh dấu `truncated` (`:154-156`).
- Gộp theo `id` (`:144`, `:159`).
- Search đi qua `repo.search`, nên **chịu AccessPolicy** của người gọi ([1c](./1c-access-control.md)). Ứng viên ngoài phạm vi quyền không bao giờ được thấy.

🧠 Hệ quả: `$match` chỉ tìm được ứng viên mà **search thường** tìm được. Search tên dùng `to_tsvector('simple', …) @@ to_tsquery('simple', 'lan:*')` (`fhir/sql.ts:160-171`, `:236-247`), **không bỏ dấu**. Bản ghi "Nguyen Thi Lan" không được tìm ra từ "Nguyễn Thị Lan" ở chiến lược 3. Chỉ lọt vào nếu chiến lược 1 hoặc 2 bắt được (cùng CCCD, cùng SĐT).

## 3. `$match`: so 12 trường theo khung CMS

📘 Medplum hiện thực **CMS Patient Matching framework**, bản `3.4.0` (`patientmatch.ts:46-47`). Đây là khung của cơ quan bảo hiểm y tế liên bang Mỹ (CMS), quy định **tổ hợp trường nào đủ để trả hồ sơ** cho bên hỏi.

✅ 12 trường được trích bằng FHIRPath rồi chuẩn hoá (`utils/cms-patient-match.ts:165-188`):

| Trường | FHIRPath | Chuẩn hoá | Cho phép "fuzzy"? |
|---|---|---|---|
| firstName | `name.given` | `foldString` | Có (trừ khi bật chặn song sinh) |
| lastName | `name.family` | `foldString` | Có |
| dob | `birthDate` | chỉ nhận ngày đủ `YYYY-MM-DD` (`:228-230`) | Lệch đúng ±1 ngày = fuzzy (`:353-368`) |
| streetLine | `address.line` | `foldString` | Có |
| zip | `address.postalCode` | 5 chữ số (`:208-210`) | Không |
| phone | `telecom(phone)` | chỉ giữ số, bỏ mã `1` của Mỹ (`:216-222`) | Không |
| email | `telecom(email)` | trim + chữ thường | Không |
| ssnLast4, itinLast4 | identifier SSN/ITIN | 4 ký tự cuối | Không |
| mbi, legalId | identifier Medicare (MBI, beneficiary id) | `foldString` | Không |
| **namespaceId** | **mọi identifier khác** (`system\|value`) | `foldString` | Không |

- **Fuzzy** = khoảng cách Damerau-Levenshtein ≤ 1 (thêm/bớt/thay/đảo 1 ký tự), chỉ áp với chuỗi ≥ 5 ký tự sau chuẩn hoá (`:306-330`, `:376-377`).
- `foldString` = NFD, bỏ dấu, chữ thường, **chỉ giữ `a-z0-9`** (`:260-262`).

✅ **Bảng tổ hợp** (`:95-152`): 1 chuỗi `if … else if`, khớp tổ hợp đầu tiên thì dừng. Ví dụ:

| Mã | Tổ hợp | Ghi chú |
|---|---|---|
| 01 | tên* + họ* + DOB* + số nhà* | `*` = cho phép fuzzy, nhưng **tối đa 1** trường fuzzy (`:79-83`) |
| 02 | tên + họ* + DOB* + SĐT | |
| 03 | tên* + họ* + DOB* + email | |
| 04–07 | tên/họ + DOB + 4 số cuối SSN/ITIN | |
| 11, 12 | tên + DOB + SĐT / email | |
| 22 | **namespaceId** khớp | 1 identifier cùng `system` và `value` là đủ |
| 29 | tên* + họ* + SĐT + ZIP | **Tự tắt khi 2 DOB đều có mà khác nhau** (`:141-149`), để không ghép cha mẹ với con cùng tên |
| 30 | họ* + DOB + SĐT | |

Quy tắc 23–28 (định danh bảo hiểm) **không** được đánh giá, vì khung CMS không quy định cách truyền mã bảo hiểm theo namespace của payer (`:52-53`).

## 4. `$match`: chấm điểm, phân tầng, chặn cứng

✅ `scoreCandidate` (`patientmatch.ts:210-225`):

| Điều kiện | `score` | `match-grade` |
|---|---|---|
| Khớp 1 tổ hợp | `1` | `certain` |
| Hậu tố thế hệ mâu thuẫn (Jr vs Sr, II vs III) | `0` | `certainly-not` |
| Còn lại | `min((exact + 0.5 × fuzzy) / 12, 0.9)` | `probable` nếu ≥ 0.65, `possible` nếu ≥ 0.2, ngược lại `certainly-not` (`:42-43`, `:227-235`) |

- Trần `0.9` bảo đảm **chỉ tổ hợp mới cho ra `certain`**. Cộng điểm nhiều tới đâu cũng không vượt qua được.
- `certainly-not` bị lọc khỏi kết quả (`:118`).
- Kết quả sắp theo `score` giảm dần (`:127`).
- Mỗi entry có `search.mode = 'match'`, `search.score`, extension `match-grade`. Nếu `certain` thì thêm mã tổ hợp và `exact`/`fuzzy` (`:243-262`).

**Chặn song sinh** (`:111-117`): nếu **hơn 1** ứng viên có DOB khớp (kể cả ±1 ngày) thì chấm lại toàn bộ với `exactFirstName: true`. Song sinh khác tên 1 ký tự (VD "Lan"/"Lân" sau bỏ dấu đều là "lan", nhưng "Hoa"/"Hoà" cũng vậy) không còn được coi là fuzzy.

🧠 Đây là khác biệt cốt lõi với cách "cộng điểm" của OpenEMR (mục 10): **điểm chỉ dùng để xếp hàng cho người duyệt**, còn quyết định "chắc chắn" là logic rời rạc, giải thích được bằng 1 mã tổ hợp.

## 5. Chế độ "chỉ trả kết quả chắc chắn" và audit

✅ Tham số `onlyCertainMatches` (`patientmatch.ts:121-125`):
- Chỉ trả kết quả nếu có **đúng 1** ứng viên `certain` **và** không search nào bị cắt ở 100 (`truncated`).
- 0 hoặc ≥ 2 thì trả Bundle rỗng.

Kèm theo là 1 AuditEvent (`:89-91`, `:272-308`), `description` là JSON gồm:

| Trường | Giá trị |
|---|---|
| `outcome` | `released` / `no-match` / `ambiguous` / `gather-truncated` |
| `criteria` | Mã tổ hợp của từng ứng viên `certain` |
| `matchedIds` | id các ứng viên |
| `uniqueness` | `unique` / `ambiguous` / `unproven` / `none` |

`ambiguous` và `gather-truncated` ghi kết quả `MinorFailure` (`:301`) để nổi lên trên giám sát.

🧠 Ý nghĩa nghiệp vụ: chế độ này dành cho **trả hồ sơ cho bên ngoài** (bệnh viện khác, payer hỏi "bệnh nhân này có hồ sơ ở bạn không"). Nguyên tắc: không chắc là duy nhất thì **không trả gì**. Đây là đặc tính pháp lý (K3): trả nhầm hồ sơ là sự cố lộ dữ liệu.

## 6. Kịch bản bà Lan chạy qua `$match`

💭 Dữ liệu ở [3a-G Phần 1](./3a-greenfield-dinh-danh-benh-nhan.md#phần-1--bối-cảnh-giả-định-). Giả định CCCD lưu với cùng `system` `urn:vn:cccd` (💭, Việt Nam chưa có URI chuẩn chung trong FHIR). Gọi `$match` với bản ghi B (đầu vào), so với A, C, D đang có:

Lưu ý: A và D được lễ tân gõ SĐT là `0912 345 678` (có dấu cách), B gõ `0912345678`.

| Ứng viên | Có lọt vào ứng viên không | Nếu lọt vào thì so trường | Kết quả 🧠 (suy từ code) |
|---|---|---|---|
| A (Nguyễn Thị Lan, có dấu) | **Không.** Chiến lược 2 search `telecom=0912345678` là so token **nguyên văn**, không khớp `0912 345 678`. Chiến lược 3 search tên không bỏ dấu, "Nguyen" không khớp "Nguyễn" | tên, họ, DOB, số nhà đều `exact` sau `foldString`, SĐT cũng `exact` sau `normalizePhone` | **Không có trong kết quả**, dù nếu được so thì là `certain` tổ hợp 01 |
| C (NGUYEN THI LAN, DOB +1) | Có, nhờ chiến lược 3 (tên + DOB ±1) | tên, họ exact, DOB fuzzy. Không có SĐT, số nhà | Không tổ hợp nào. Điểm (2 + 0.5)/12 = 0.21 → **possible** |
| D (con gái, cùng tên) | Không (SĐT lệch định dạng, DOB ngoài ±1 ngày) | — | Không có trong kết quả |

Nếu A được lưu SĐT `0912345678` thì A lọt vào, ra **certain** (tổ hợp 01). D cũng lọt vào, nhưng quy tắc 29 bị chặn vì DOB khác, điểm 3/12 = 0.25 → **possible**.

Nếu đầu vào là **C** (có CCCD) thì A lọt vào nhờ chiến lược 1 và là **certain** theo tổ hợp 22 (cùng `urn:vn:cccd|001158012345`).

🧠 Bài học:
1. **Bước thu ứng viên yếu hơn bước chấm điểm.** Chấm điểm chuẩn hoá SĐT (`utils/cms-patient-match.ts:216-222`) và bỏ dấu tên, nhưng thu ứng viên dùng search FHIR thường, không chuẩn hoá gì (`patientmatch.ts:168`, `:180`, `:189-192`). Ứng viên không lọt vào thì chấm điểm tốt cũng vô ích. Cách chữa: **chuẩn hoá lúc lưu** (SĐT dạng E.164 `+84912345678`, thêm 1 `name` không dấu), hoặc gọi `$match` nhiều lần với các biến thể.
2. D lọt vào `possible` khi SĐT cùng định dạng. Hàng đợi người duyệt **sẽ** thấy mẹ và con. Đây là đúng thiết kế: `possible` là để người xem.
3. Tổ hợp 22: **1 identifier khớp là đủ chắc chắn**. Nếu ở Việt Nam hay có chuyện dùng CCCD/thẻ BHYT của người nhà để khám, phải cân nhắc không đưa mã đó vào `$match`, hoặc không dùng `onlyCertainMatches`.

## 7. Tiếng Việt: dấu và chữ Đ

✅ Kiểm thử thật bằng đúng logic `foldText` (`util/text.ts:4`, `:14-16`, regex `\p{Diacritic}`) + `foldString` (`utils/cms-patient-match.ts:260-262`):

| Đầu vào | Sau `foldString` |
|---|---|
| Nguyễn | `nguyen` |
| Thị Lan | `thilan` |
| Trần Văn Minh | `tranvanminh` |
| **Đặng** | **`ang`** |
| Dang | `dang` |
| **Đỗ** | **`o`** |

🧠 Nguyên nhân: Đ/đ (U+0110/U+0111) là chữ riêng, **không** phân rã NFD thành D + dấu. Sau khi bỏ dấu, `đ` vẫn còn, rồi bị bước "chỉ giữ `a-z0-9`" **xoá mất**. Hệ quả cho họ Đặng, Đỗ, Đinh, Đào, Đoàn:
- "Đặng" vs "Dang" → `ang` vs `dang`: khác nhau, chuỗi < 5 ký tự nên không xét fuzzy → `none`.
- "Đặng" vs "Đặng" vẫn khớp (cả 2 cùng thành `ang`), nên chỉ lỗi khi **1 bên gõ không dấu**. Đó lại đúng là trường hợp hay gặp nhất (dữ liệu HL7 v2, tên in hoa không dấu).

Cách sửa nếu dùng Medplum ở Việt Nam: thay `đ→d`, `Đ→D` trước khi gọi `$match`, hoặc chuẩn hoá ở Bot tiếp nhận. Phát hiện cùng loại đã ghi ở [1a-comparison phụ lục](./1a-comparison-fhir-datastore.md).

## 8. `$extract`: phiếu → hồ sơ

📘 Chuẩn **SDC** (Structured Data Capture IG) định nghĩa cách nhúng "khuôn trích xuất" vào `Questionnaire`. Medplum hiện thực kiểu **template-based extraction**:

✅ `fhir/operations/extract.ts`:
- Lấy `QuestionnaireResponse` theo id hoặc tham số, rồi tìm `Questionnaire` theo `url` (`:131-162`).
- Khuôn là các resource trong `Questionnaire.contained` (`:200-205`).
- Extension `templateExtract` trên Questionnaire hoặc trên 1 item chỉ khuôn nào được dùng. Đặt trên item lặp thì sinh nhiều resource, mỗi lần lặp 1 resource (`:442-462`, ngữ cảnh ở `:487-508`).
- `templateExtractValue` là biểu thức FHIRPath điền giá trị. `extractAllocateId` cấp UUID để các resource trong cùng Bundle tham chiếu nhau.
- Mỗi resource thành 1 entry `POST`, hoặc `PUT` nếu có `resourceId`. Có hỗ trợ `ifNoneExist` (tạo có điều kiện) (`:465-485`).
- Kết quả: **Bundle `transaction` chưa thực thi** (`:171`, `:215`).

🧠 `ifNoneExist` chính là chỗ chống trùng **rẻ nhất**: khuôn Patient có `ifNoneExist = "identifier=urn:vn:cccd|{CCCD}"` thì gửi Bundle 2 lần không tạo 2 bệnh nhân. Nhưng chỉ chống được trùng **có cùng mã**, không chống được A–B.

🧠 Ghi chú phụ: `processingOrder` (`:118-125`) khai báo khoá bằng **tên biến** (`allocIdExtension: 1`) thay vì giá trị URL (`[allocIdExtension]: 1`). JavaScript hiểu đó là khoá chuỗi `"allocIdExtension"`, nên `getOrder` (`:128`) luôn trả mặc định 5 và việc sắp thứ tự ở `:284` không có tác dụng. Chưa kiểm chứng có gây sai kết quả thật không.

### Bot tiếp nhận mẫu làm gì

✅ `examples/medplum-patient-intake-demo/src/bots/core/intake-form.ts` (Bot chạy khi có `QuestionnaireResponse` mới):

| Bước | Dòng |
|---|---|
| Dựng `Patient` từ câu trả lời: tên, ngày sinh, địa chỉ, SĐT, SSN, người liên hệ khẩn cấp, chủng tộc, ngôn ngữ | `:52-128` |
| **`createResource(patient)`**, luôn tạo mới | `:131` |
| Gắn `QuestionnaireResponse.subject` về bệnh nhân | `:135-136` |
| Tạo Observation (hoàn cảnh xã hội), AllergyIntolerance, MedicationStatement, Condition, FamilyMemberHistory, Immunization | `:140-227` |
| Tạo `Coverage` (bảo hiểm) cho từng nhà bảo hiểm khai | `:234` |
| Tạo **7** `Consent` (đồng ý điều trị, chia sẻ dữ liệu…) | `:246-318` |

🧠 Đối chứng nội bộ: Bot mẫu **không** gọi `$match`. Nếu dùng nguyên Bot này ở quầy, mỗi lần bà Lan điền phiếu sinh 1 Patient mới. Đây là chỗ demo bỏ qua bài toán của module này.

## 9. Gộp hồ sơ: không có trong server

✅ `examples/medplum-demo-bots/src/deduplication/` có 2 Bot:

| Bot | Làm gì |
|---|---|
| `find-matching-patients.ts` | So tên, họ, ngày sinh, ZIP. Bỏ qua cặp trong danh sách "không ghép". Tạo `RiskAssessment` (điểm) + `Task` cho người duyệt |
| `merge-matching-patients.ts` | Người duyệt điền phiếu → gộp |

Gộp làm 4 việc:
1. **Liên kết** (`linkPatientRecords`, `:123-132`): bản chính thêm `link {type: 'replaces'}`, bản phụ thêm `link {type: 'replaced-by'}` và `active = false`.
2. Gộp thông tin liên lạc.
3. **Viết lại tham chiếu** của dữ liệu lâm sàng từ bản phụ sang bản chính (`rewriteClinicalDataReferences`, `:254+`), dùng `Patient/$everything` có phân trang.
4. Xoá bản phụ **nếu người duyệt chọn**. Nếu không thì để `active = false` (`:97-104`).

Có hàm tách lại (`unlinkPatientRecords`, `:145-158`): gỡ link, bật lại `active`. Nhưng **không** hoàn tác bước 3. Tham chiếu đã viết lại thì không biết dòng nào vốn của bản phụ.

📘 Docs nêu 2 kiểu bản ghi chính (`patient-deduplication/merging.mdx`):
- **Independent master**: tạo Patient mới làm bản chính, các bản nguồn giữ nguyên. Tách lại dễ (gỡ liên kết, gộp lại).
- **Record promotion**: nâng 1 bản nguồn thành bản chính. Đơn giản hơn, hợp khi nguồn sống ngắn (VD di chuyển từ hệ thống cũ).

## 10. Đối chiếu OpenEMR

| | Medplum | OpenEMR ✅ |
|---|---|---|
| Tìm trùng | `$match` theo yêu cầu (đồng bộ) | Điểm lưu sẵn trong cột `patient_data.dupscore` (`sql/database.sql:8468`), tính lại theo lô (`DuplicatePatientService.php:161-199`) |
| Cách tính | Tổ hợp quy tắc CMS + điểm phụ | Cộng điểm: email 6, DOB 6, họ+tên 6, **giới 6**, soundex họ 2, SĐT 1 (`:93-112`). Ngưỡng 12 = "ít nhất 2 tín hiệu mạnh" (`:50`, `:87-90`) |
| "Không phải trùng" | Bot tự lưu danh sách | `dupscore = -1` (`:44`, `:145-152`) |
| Gộp | Liên kết `Patient.link`, giữ bản phụ (Bot) | Chuyển mọi dòng sang bệnh nhân đích, chuyển tài liệu, **xoá dòng định danh** của bản nguồn (`PatientMergeService.php` docblock dòng 4–9, `mergeAllTables` `:298`) |
| Tách lại | Có (gỡ link) | Không |
| Chạy thử | — | Có chế độ dry run (`$production = false`) |

🧠 Với kịch bản bà Lan, công thức OpenEMR cho A–D: DOB khác (0) + họ tên giống (6) + giới giống (6) + soundex (2) + SĐT (1) = **15 ≥ 12** → đề xuất gộp **mẹ với con**. Giới tính được 6 điểm, ngang ngày sinh, trong khi chỉ có 2 giá trị phổ biến. Đây là ví dụ cụ thể cho điểm mù "cộng điểm che mâu thuẫn" ở [3a-G V2](./3a-greenfield-dinh-danh-benh-nhan.md#v2--chuẩn-hoá--chấm-điểm).

## Sai lầm thường gặp

1. **Lưu SĐT theo định dạng tuỳ người gõ.** Bước thu ứng viên so nguyên văn, nên `0912 345 678` và `0912345678` không gặp nhau (mục 6).
2. **Gọi `$match` rồi coi `probable` là trùng.** Chỉ `certain` mới được tự động. `probable`/`possible` là hàng đợi người duyệt.
3. **Lưu CCCD/BHYT không có `system`.** Tổ hợp 22 so `system|value`. Thiếu `system` thì không bao giờ khớp.
4. **Dùng nguyên Bot tiếp nhận mẫu.** Luôn tạo mới (`intake-form.ts:131`).
5. **Gộp bằng cách viết lại tham chiếu rồi xoá bản phụ** ngay từ đầu. Mất khả năng tách lại.
6. **Không xử lý chữ Đ** trước khi so tên ở Việt Nam.
7. **Gọi `$match` bằng tài khoản có quyền hẹp.** Ứng viên ngoài AccessPolicy không được thấy, nên "không có trùng" có thể chỉ là "không thấy".

## Tự kiểm tra

1. Vì sao điểm tối đa ngoài tổ hợp là 0.9 chứ không phải 1?
2. Bệnh nhân song sinh "Lan" và "Lân", cùng DOB, cùng SĐT. `$match` xử lý thế nào? Nếu chỉ có 1 người trong DB thì sao?
3. Một payer hỏi "bà Lan có hồ sơ ở đây không" và có 2 ứng viên `certain`. Server trả gì, ghi audit gì?
4. Viết `ifNoneExist` cho khuôn Patient trong `$extract` để chống tạo trùng theo CCCD.
