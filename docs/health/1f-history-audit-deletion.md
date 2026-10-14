# 1f — Lịch sử, nhật ký truy cập và xoá dữ liệu

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Thanh tra hỏi 'ai đã sửa kết quả xét nghiệm này, sửa gì, ai đã xem hồ sơ bà Lan tuần trước'. Bệnh nhân yêu cầu xoá dữ liệu. Hệ thống trả lời bằng gì?"*
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. 3 câu hỏi pháp lý khác nhau](#1-3-câu-hỏi-pháp-lý-khác-nhau)
- [2. Lịch sử phiên bản: "ai đã sửa gì"](#2-lịch-sử-phiên-bản-ai-đã-sửa-gì)
- [3. AuditEvent: "ai đã xem / làm gì"](#3-auditevent-ai-đã-xem--làm-gì)
- [4. 4 mức xoá](#4-4-mức-xoá)
- [5. Chống ghi đè: If-Match](#5-chống-ghi-đè-if-match)
- [6. Đối chiếu OpenEMR](#6-đối-chiếu-openemr)
- [7. Liên hệ chuẩn tuân thủ](#7-liên-hệ-chuẩn-tuân-thủ)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 3 câu hỏi pháp lý khác nhau

🧠 Hay bị gộp làm 1, nhưng là 3 cơ chế riêng:

| Câu hỏi | Cơ chế | Lưu ở đâu |
|---|---|---|
| **Dữ liệu đã thay đổi thế nào?** (ai sửa, trước/sau ra sao) | Lịch sử phiên bản | Bảng `<Type>_History` |
| **Ai đã làm gì với hệ thống?** (kể cả chỉ **xem**, tìm kiếm, đăng nhập) | Nhật ký truy cập | `AuditEvent`: log hoặc bảng `AuditEvent` |
| **Làm sao xoá thật khi luật yêu cầu?** | Xoá có kiểm soát | `$expunge`, purge |

Lịch sử phiên bản **không** ghi lại việc xem. Muốn biết "ai đã xem hồ sơ bà Lan" thì phải có AuditEvent.

## 2. Lịch sử phiên bản: "ai đã sửa gì"

✅ Đã thấy ở [1a mục 5](./1a-fhir-datastore.md#5-ghi-sửa-xoá-lịch-sử-không-bao-giờ-mất). Mỗi lần ghi tạo 1 dòng `_History` với `versionId` mới (`fhir/repo.ts:2035-2051`). `meta` của mỗi phiên bản mang:
- `meta.author`: ai ghi
- `meta.onBehalfOf`: ghi thay ai
- `meta.lastUpdated`: lúc nào

(`repo.ts:1040-1060`)

**API đọc lịch sử:**

| Endpoint 📘 | Hàm | Ghi chú |
|---|---|---|
| `GET Observation/bp-150/_history` | `readHistory` (`repo.ts:786`) | Kiểm tra quyền `history` trên từng phiên bản (`:797`) |
| `GET Observation/bp-150/_history/{vid}` | `readVersion` (`repo.ts:894`) | Kiểm tra quyền `vread` (`:905`) |

💭 Ví dụ lịch sử 1 kết quả bị sửa:

| versionId | lastUpdated | author | status | value |
|---|---|---|---|---|
| v1 | 08:00 | ĐD Hoa | preliminary | 150 |
| v2 | 08:30 | BS Minh | final | 150 |
| v3 | 10:15 | BS Minh | amended | 145 |

🧠 Đọc nghiệp vụ: v3 là **đính chính** sau khi đã chốt. 📘 FHIR có trạng thái `amended`/`corrected` riêng. Hệ thống tốt nên **chặn sửa thẳng bản `final`** trừ khi đổi sang `amended`. Ở Medplum, điều đó làm được bằng `writeConstraint` ([1c mục 4](./1c-access-control.md#4-tầng-3--accesspolicy-chính-sách-là-dữ-liệu)).

## 3. AuditEvent: "ai đã xem / làm gì"

✅ Mọi thao tác qua `Repository` gọi `logEvent` (`fhir/repo.ts:2477-2569`):

1. Tăng bộ đếm metric theo loại thao tác (`:2501-2507`).
2. **Bỏ qua** thao tác **đọc** của tài khoản hệ thống và thao tác trên chính `AuditEvent` (`:2509-2512`). Không thì hệ thống tự ghi log việc tự đọc log.
3. Dựng `AuditEvent` gồm: loại thao tác (`read`, `vread`, `search`, `create`, `update`, `delete`, `history`…, `util/auditevent.ts:108-131`), project, người làm, IP, kết quả, **câu search đã chạy**, resource bị tác động, thời gian xử lý (`repo.ts:2522-2537`).
4. Ghi ra **2 nơi tuỳ cấu hình** (`config/types.ts:78-79`):

| Cấu hình | Hành vi | Code |
|---|---|---|
| `logAuditEvents` | Ghi JSON ra log (stdout → CloudWatch, Datadog…) | `util/auditevent.ts:284-289` |
| `saveAuditEvents` | Lưu thành resource `AuditEvent` trong DB, **chỉ** khi thao tác có resource cụ thể. Lưu **không chờ** (không làm chậm request) | `repo.ts:2540-2568` |

**Ẩn tên trong log** ✅ `util/auditevent.ts:298-310` (`applyOptionalRedaction`): bật `redactAuditEvents` (toàn server hoặc theo project) thì bỏ `display` của reference. 🧠 Bản thân log truy cập cũng là dữ liệu nhạy cảm: "BS Minh xem hồ sơ Nguyễn Thị Lan" đã tiết lộ bà Lan là bệnh nhân ở đây.

**AuditEvent cho đăng nhập, bot, subscription** ✅ cũng có: `LoginEvent`, `LogoutEvent` (`util/auditevent.ts:101-102`), `createBotAuditEvent` (`:321`), `createSubscriptionAuditEvent` (`:410`).

🧠 Thiết kế 2 đầu ra phản ánh đánh đổi thật:
- Lưu **mọi** lần đọc vào DB thì bảng `AuditEvent` thành bảng lớn nhất hệ thống.
- Chỉ ghi ra log thì truy vấn "ai đã xem bà Lan" phải làm ở hệ thống log, không làm qua FHIR API.

## 4. 4 mức xoá

| Mức | Cách gọi | Ai được | Dữ liệu còn lại | Code |
|---|---|---|---|---|
| **Xoá mềm** | `DELETE Observation/x` | Ai có quyền `delete` | Dòng chính: `deleted=true`, `content=''`. `_History` còn đủ, thêm 1 dòng "đã xoá" | `repo.ts:1471`, `repository/row-builder.ts:105-125` |
| **Expunge 1 resource** | `POST Observation/x/$expunge` | **Admin project** hoặc super admin | Xoá thật dòng chính, `_History`, lookup table | `fhir/operations/expunge.ts:30-34`, `repo.ts:1664-1700` |
| **Expunge kèm mọi thứ liên quan** | `$expunge?everything=true` (VD trên `Patient`) | Admin project, chạy **bất đồng bộ** (`AsyncJob`) | Xoá resource và mọi thứ trong compartment của nó, kể cả `Binary` đính kèm | `expunge.ts:40-55`, `:118-120` |
| **Purge theo thời gian** | Endpoint admin, `before=<ngày>` | **Chỉ super admin** | Xoá mọi phiên bản cũ hơn mốc, cho cả loại resource | `repo.ts:1772-1789`, gọi từ `admin/super.ts:325` |

🧠 Ý nghĩa nghiệp vụ của từng mức:
- **Xoá mềm** là xoá "theo nghiệp vụ": nhập nhầm, huỷ chỉ định. Bằng chứng vẫn còn.
- **Expunge** là xoá "theo pháp lý": dữ liệu nhập nhầm vào hồ sơ người khác (phải gỡ hẳn), yêu cầu xoá dữ liệu cá nhân hợp lệ.
- **Purge** là xoá "theo chính sách lưu trữ": VD dữ liệu thử nghiệm, log quá hạn lưu.

Quyền tăng dần theo mức độ không thể hoàn tác. Đúng nguyên tắc.

## 5. Chống ghi đè: If-Match

✅ `PUT` và `DELETE` có header `If-Match: W/"<versionId>"`. Nếu phiên bản hiện tại khác, trả `412 Precondition Failed` (`repo.ts:1026` cho update; `:1487-1501` và `:1535-1539` cho delete). Commit gần đây *"Honor If-Match on FHIR delete (#10895)"* vừa bổ sung cho delete.

🧠 Tình huống y tế: ĐD Hoa và BS Minh cùng mở hồ sơ bà Lan. Hoa lưu chỉ số mới, Minh lưu chẩn đoán từ bản cũ hơn. Không có If-Match thì bản của Minh **ghi đè mất** chỉ số của Hoa mà không ai biết. Có If-Match thì Minh nhận lỗi và phải tải lại.

## 6. Đối chiếu OpenEMR

✅ `sql/database.sql`:

| Bảng | Cột chính | Vai trò |
|---|---|---|
| `log` (`:7772`) | `date`, `event`, `category`, `user`, `patient_id`, `success`, `comments`, **`checksum`** | Nhật ký thao tác. `checksum` để phát hiện log bị sửa |
| `log_comment_encrypt` (`:12586`) | — | Mã hoá nội dung log |
| `extended_log` (`:12440`) | — | Sự kiện mở rộng (VD in, xuất hồ sơ) |
| `api_log` (`:92`) | — | Nhật ký gọi API |

Lịch sử thay đổi dữ liệu: **không có cơ chế chung**. Phần lớn bảng nghiệp vụ chỉ có cột `date`/`last_updated`, ghi đè trực tiếp. Bản cũ (nếu cần) phải lấy từ `log.comments`. Xoá: phần lớn là cờ `activity=0` hoặc `deleted=1` theo từng bảng.

| | Medplum | OpenEMR |
|---|---|---|
| Lịch sử phiên bản | **Tự động, mọi resource** (`_History`) | Không có chung. Một số qua log |
| Nhật ký truy cập | `AuditEvent` chuẩn FHIR, ra log hoặc DB | Bảng `log` riêng |
| Chống sửa log | 🧠 Dựa vào hạ tầng log bên ngoài (không thấy checksum trong code) | Có cột `checksum` và bảng mã hoá |
| Xoá thật | `$expunge`, purge, có phân quyền | Không có cơ chế chuẩn |

🧠 Mỗi bên mạnh 1 chỗ. Medplum mạnh ở **lịch sử dữ liệu** (có sẵn vì kiến trúc "bản gốc + phiên bản"). OpenEMR mạnh ở **chống giả mạo log**, vì nó là EHR phải đạt chứng nhận ONC với log nằm ngay trong DB của mình.

## 7. Liên hệ chuẩn tuân thủ

📘 Docs Medplum `packages/docs/docs/compliance/` có tài liệu cho HIPAA, SOC2, ONC, 21 CFR Part 11, HITRUST, ISO 9001… Ví dụ `cfr11.md:28-30` ánh xạ yêu cầu "Audit Trails" của 21 CFR Part 11 (hồ sơ điện tử trong thử nghiệm lâm sàng/dược) sang "audit logs and edit history for all data and data access", và tham chiếu tiêu chí ONC `(d)(2) Auditable Events and Tamper-Resistance`.

🧠 Với dự án Việt Nam: quy định về hồ sơ bệnh án điện tử và bảo vệ dữ liệu cá nhân yêu cầu lưu vết và bảo mật. **Cần đọc văn bản hiện hành** để biết cụ thể thời hạn lưu trữ, yêu cầu ký số, mức độ truy vết. Các cơ chế trong file này là "nguyên liệu". Việc ánh xạ sang từng điều khoản là việc của pháp chế.

## Sai lầm thường gặp

1. **Nghĩ có `_History` là đủ audit.** `_History` không ghi việc **xem**. Phải bật `logAuditEvents` hoặc `saveAuditEvents`.
2. **Bật `saveAuditEvents` mà không tính dung lượng.** Mỗi thao tác 1 resource. Cần kế hoạch purge theo chính sách lưu trữ hoặc đẩy sang kho log.
3. **Dùng `DELETE` cho yêu cầu xoá dữ liệu cá nhân.** Xoá mềm vẫn giữ toàn bộ lịch sử. Phải `$expunge`, và cân nhắc `everything=true` cho bệnh nhân.
4. **Cho mọi user quyền `delete`.** Xoá mềm vẫn làm dữ liệu biến mất khỏi màn hình làm việc. Nên hạn chế bằng AccessPolicy, ưu tiên đổi `status` (VD `entered-in-error`) thay vì xoá.
5. **Không dùng If-Match trong app nhiều người cùng sửa.** Mất dữ liệu âm thầm.

## Tự kiểm tra

1. Thanh tra hỏi: "Ai đã xem hồ sơ bà Lan từ 1/10 đến 7/10?" Cần cấu hình gì từ trước để trả lời được, và truy vấn ở đâu?
2. Kết quả HbA1c bị nhập nhầm vào hồ sơ bà Lan (thực ra của bệnh nhân khác). Nên `DELETE`, đổi `status = entered-in-error`, hay `$expunge`? Mỗi lựa chọn để lại dấu vết gì?
3. Vì sao `$expunge?everything=true` chạy bất đồng bộ còn `$expunge` 1 resource thì không?
