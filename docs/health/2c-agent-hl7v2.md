# 2c — Agent và HL7 v2: nói chuyện với hệ thống bệnh viện cũ

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Bệnh viện, phòng xét nghiệm, máy xét nghiệm vẫn gửi dữ liệu theo chuẩn cũ qua mạng nội bộ. Làm sao đưa dữ liệu đó lên 1 nền tảng cloud, không mất tin nào?"*

## Mục lục

- [1. Vì sao HL7 v2 vẫn sống](#1-vì-sao-hl7-v2-vẫn-sống)
- [2. Giải phẫu 1 tin nhắn HL7 v2](#2-giải-phẫu-1-tin-nhắn-hl7-v2)
- [3. Kiến trúc Medplum Agent](#3-kiến-trúc-medplum-agent)
- [4. ACK: xác nhận đã nhận, và vì sao nó quyết định tốc độ](#4-ack-xác-nhận-đã-nhận-và-vì-sao-nó-quyết-định-tốc-độ)
- [5. Không mất tin: hàng đợi bền trên máy Agent](#5-không-mất-tin-hàng-đợi-bền-trên-máy-agent)
- [6. Bot chuyển HL7 sang FHIR](#6-bot-chuyển-hl7-sang-fhir)
- [7. Đối chiếu OpenEMR](#7-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vì sao HL7 v2 vẫn sống

📘 HL7 v2 ra đời từ cuối thập niên 1980. Hệ thống thông tin bệnh viện (HIS), phòng xét nghiệm (LIS), chẩn đoán hình ảnh (RIS/PACS) và máy xét nghiệm trao đổi bằng tin nhắn văn bản dạng `|` qua giao thức **MLLP** trên TCP, thường **không mã hoá**, trong mạng nội bộ.

🧠 Vì sao chưa bị FHIR thay:
- Hàng chục năm hệ thống đã cài đặt, chạy ổn định. Thay là chi phí lớn, không mang lại doanh thu.
- Máy xét nghiệm, máy theo dõi chỉ nói HL7 v2, ASTM, DICOM.
- Luồng sự kiện (nhập viện, chuyển khoa, ra viện, kết quả xét nghiệm) đã chuẩn hoá quanh HL7 v2 từ lâu.

Hệ quả cho sản phẩm cloud: **muốn lấy dữ liệu từ bệnh viện thì phải có 1 cầu nối đặt trong mạng bệnh viện**. Đó là Medplum Agent.

📘 Docs Medplum (`packages/docs/docs/agent/index.md`): *"runs inside your firewall and connects to devices over low level protocols such as HL7/MLLP, ASTM, and DICOM. These network protocols are commonly unencrypted, and therefore require an adapter to a secure transport."*

## 2. Giải phẫu 1 tin nhắn HL7 v2

💭 Tin nhắn ADT (nhập viện/cập nhật thông tin bệnh nhân), rút gọn:

```
MSH|^~\&|HIS|BV_DAKHOA|MEDPLUM|ANBINH|20261014080000||ADT^A08|MSG000123|P|2.5
PID|1||AB-000123^^^BV_DAKHOA^MR||NGUYEN^THI LAN||19580312|F|||12 Tran Hung Dao^^Ha Noi^^100000
PV1|1|O|KHAM-NOI^^^BV_DAKHOA||||77^TRAN^VAN MINH
```

| Đoạn | Ý nghĩa | Trường đáng chú ý |
|---|---|---|
| `MSH` | Đầu tin: ai gửi, gửi ai, loại tin | `MSH-9` loại tin (`ADT^A08`), `MSH-10` mã tin (dùng chống trùng), `MSH-12` phiên bản |
| `PID` | Bệnh nhân | `PID-3` mã bệnh nhân, `PID-5` họ tên (`HỌ^TÊN`), `PID-7` ngày sinh, `PID-11` địa chỉ |
| `PV1` | Lượt khám/nhập viện | Khoa, bác sĩ |
| `OBR`, `OBX` | (trong tin kết quả `ORU`) chỉ định và từng kết quả | `OBX-3` mã xét nghiệm, `OBX-5` giá trị, `OBX-6` đơn vị |

📘 Các loại tin hay gặp:

| Loại | Sự kiện |
|---|---|
| `ADT^A01/A04/A08/A03` | Nhập viện / đăng ký / cập nhật / ra viện |
| `ORM^O01`, `OML` | Chỉ định (xét nghiệm, chẩn đoán hình ảnh) |
| `ORU^R01` | Kết quả |
| `SIU` | Lịch hẹn |
| `DFT^P03` | Ghi phí (tài chính) |

✅ Medplum có bộ phân tích HL7 v2 trong `packages/core/src/hl7.ts`: `Hl7Context` (`:50`), `Hl7Message` (`:95`), `Hl7Segment` (`:452`), `Hl7Field` (`:705`). Truy cập kiểu `msg.getSegment('PID').getField(5).getComponent(1)`.

## 3. Kiến trúc Medplum Agent

```
 Mạng bệnh viện                          │  Cloud
                                         │
 HIS/LIS ──MLLP──▶ ┌──────────────┐      │      ┌──────────────┐
 Máy XN ──ASTM───▶ │ Medplum Agent│══WSS════════▶│ Medplum      │──▶ Bot "hl7→FHIR" ──▶ Patient, Observation…
 PACS ───DICOM──▶  │ (Node, cài   │◀═════════════│ server       │
                   │  tại chỗ)    │  ACK/push    └──────────────┘
                   └──────────────┘      │
```

📘 Cấu hình gồm 4 resource (`packages/docs/docs/agent/index.md`, mục Installation):

| Resource | Vai trò |
|---|---|
| `Endpoint` | Địa chỉ Agent lắng nghe, VD `address: mllp://0.0.0.0:56000`, `connectionType: hl7v2-mllp` |
| `Bot` | Nhận tin, chuyển sang FHIR, trả ACK |
| `Agent` | Gắn kênh: `channel[].endpoint` → `channel[].target` (Bot). Kèm `setting[]` |
| `ClientApplication` | Danh tính để Agent đăng nhập lên server |

✅ Agent nói chuyện với server qua **WebSocket** (`packages/agent/src/app.ts:353`, đổi `https:` thành `wss:`). Các loại tin trao đổi gồm:
- `agent:connect`, `agent:heartbeat`
- `agent:transmit:request` / `:response` (chuyển tin)
- `push` (server đẩy tin xuống thiết bị)
- `agent:reloadconfig`, `agent:upgrade`, `agent:logs`, `agent:stats`

(`app.ts:410-504`)

✅ Operation quản trị Agent từ server (`packages/server/src/fhir/routes.ts:311-331`): `$status`, `$bulk-status`, `$reload-config`, `$upgrade`, `$fetch-logs`, `$stats`. Thêm `$push` để gửi tin **xuống** thiết bị (`fhir/operations/agentpush.ts`).

🧠 Hướng kết nối quan trọng: Agent **chủ động mở** kết nối ra ngoài (wss). Bệnh viện không phải mở cổng cho Internet đi vào. Bộ phận IT bệnh viện thường chỉ chấp nhận cách này.

## 4. ACK: xác nhận đã nhận, và vì sao nó quyết định tốc độ

📘 Docs `packages/docs/docs/agent/acknowledgement-modes.md`:

| Chế độ | Cách chạy | Thông lượng (theo docs) |
|---|---|---|
| **Original** | Bên gửi gửi tin, **chờ** bên nhận xử lý xong mới nhận ACK `AA`/`AE`/`AR`, rồi mới gửi tin tiếp | khoảng **3–4 tin/giây** |
| **Enhanced** (fast ACK) | Bên nhận lưu an toàn rồi trả ngay `CA` (commit accept). Xử lý sau, gửi ACK ứng dụng sau | khoảng **300–400 tin/giây** |

🧠 Ví dụ quy mô: bệnh viện 500 giường có thể sinh hàng chục nghìn tin ADT/ORU mỗi ngày, dồn vào giờ cao điểm. 3 tin/giây là tắc nghẽn ngay. Nhưng `CA` là **lời hứa** "tôi đã lưu an toàn". Nếu chưa thật sự lưu, đó là lời hứa sai. Xem mục 5.

## 5. Không mất tin: hàng đợi bền trên máy Agent

✅ Tài liệu thiết kế `packages/agent/DURABLE_QUEUE_ARCHITECTURE.md` (trạng thái ghi trong file: *"Draft implementation plan"*) nêu thẳng vấn đề cũ (mục 1):
- Agent giữ tin trong **bộ nhớ** (`webSocketQueue`, `hl7Queue`).
- Ở chế độ enhanced, thư viện đã trả `CA` ngay khi nhận (`packages/hl7/src/connection.ts:127-131`).
- Agent chết hoặc khởi động lại là **mất tin mà bên gửi tưởng đã an toàn**.

Giải pháp: hàng đợi **SQLite** (`node:sqlite`) trên máy Agent, mã ở `packages/agent/src/queue/` (`durable-queue.ts`, `dispatcher.ts`, `worker.ts`, `retention.ts`, `schema.ts`):
- Lưu tin **trước** khi trả `CA`.
- Xử lý tuần tự theo kênh, hoặc song song theo khoá logic (VD `MSH-4`).
- Chống trùng theo `(channel, MSH-10)`.
- Thử lại: `none` / `normal` / `guaranteed` (mặc định `guaranteed`, thử mãi tới khi server trả lời).
- Lưu tin đã xử lý 7 ngày, tin lỗi 90 ngày.

Các tham số nằm ở `DURABLE_QUEUE_ARCHITECTURE.md` mục 7.

**Trạng thái tại commit này** ✅ `packages/agent/src/app.ts:562`: hàng đợi bền bật bằng `Agent.setting` tên `durableQueue`, **mặc định `false`**. Bản thiết kế ghi dự kiến chuyển mặc định sang `true` sau khi triển khai rộng.

🧠 So với [2a](./2a-subscriptions.md#6-thử-lại-tự-tắt-bảo-mật): server Medplum chấp nhận khe nhỏ có thể mất sự kiện nội bộ, nhưng ở **biên** với bệnh viện thì đội Medplum đang làm hàng đợi bền. Lý do: ở biên, bên kia là hệ thống ngoài **đã nhận lời hứa `CA`** và sẽ không gửi lại.

## 6. Bot chuyển HL7 sang FHIR

✅ Ví dụ `examples/medplum-demo-bots/src/hl7-bot.ts`:
- Đọc `MSH-9` lấy loại tin (`:10-11`). Chỉ xử lý `ADT^A28`, `A08`, `A30` (`:13-18`).
- Lấy tên từ `PID-5` (`:23-24`), mã bệnh nhân từ `PID-3` (`:27`), địa chỉ từ `PID-11` (`:30-33`).
- Tạo hoặc cập nhật `Patient`, trả ACK.

Các Bot mẫu khác: `adt-transfer-listener/` (ADT qua SFTP), `dft-transfer-listener/` (ghi phí), `lab-integration/`, `hl7-integration/`.

🧠 Bot HL7→FHIR chính là nơi chứa **kiến thức tích hợp**: mỗi bệnh viện điền HL7 v2 hơi khác nhau (trường nào chứa mã BHYT, định dạng tên, mã khoa). Logic ánh xạ thường dùng `ConceptMap` ([1e](./1e-terminology.md)) để dịch mã nội bộ của bệnh viện sang mã chuẩn.

## 7. Đối chiếu OpenEMR

✅ OpenEMR có HL7 v2 cho **xét nghiệm**:
- `interface/orders/gen_hl7_order.inc.php`: sinh tin chỉ định (ORM) gửi lab.
- `interface/orders/receive_hl7_results.inc.php`: đọc tin kết quả (ORU) đổ vào `procedure_report`/`procedure_result`.
- Trao đổi thường qua file/SFTP với lab.

| | Medplum | OpenEMR |
|---|---|---|
| Vị trí | Agent riêng trong mạng bệnh viện + server cloud | Chạy trên chính máy chủ OpenEMR |
| Giao thức | MLLP, ASTM, DICOM, qua WSS lên cloud | Chủ yếu file/SFTP cho lab |
| Ánh xạ HL7 → dữ liệu | Bot do khách viết | Code PHP cứng cho luồng xét nghiệm |
| Phạm vi | Mọi loại tin | Tập trung chỉ định/kết quả xét nghiệm |

## Sai lầm thường gặp

1. **Bật enhanced ACK mà không bật hàng đợi bền.** Thông lượng tăng, nhưng Agent chết là mất tin bên gửi tưởng đã lưu (mục 5).
2. **Bot HL7 không chống trùng theo `MSH-10`.** Bên gửi gửi lại khi không nhận ACK thì tạo `Patient`/`Observation` trùng.
3. **Ánh xạ cứng mã khoa, mã xét nghiệm trong Bot.** Đổi mã ở bệnh viện là phải sửa code. Dùng `ConceptMap`.
4. **Mở cổng MLLP của bệnh viện ra Internet** thay vì đặt Agent bên trong. MLLP không mã hoá.
5. **Coi HL7 v2 là 1 chuẩn cố định.** Mỗi nhà cung cấp HIS dùng trường tuỳ chọn khác nhau. Luôn xin **tài liệu đặc tả giao diện** của từng bệnh viện.

## Tự kiểm tra

1. Trong tin `ORU^R01`, kết quả HbA1c nằm ở đoạn nào, trường nào chứa mã LOINC, trường nào chứa giá trị?
2. Bệnh viện gửi 20.000 tin ADT/ngày, dồn vào 4 giờ cao điểm. Original mode (khoảng 3,5 tin/giây) có kịp không?
3. Vì sao Agent chủ động kết nối ra cloud thay vì cloud kết nối vào Agent?
