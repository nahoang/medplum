# 2a-C — So sánh 6 cách báo sự kiện dữ liệu y tế

> **Loại file:** C. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Nguồn ✅: [`2a-subscriptions.md`](./2a-subscriptions.md). Câu hỏi: *"Kết quả kali nguy hiểm vừa được lưu. Cách nào để chắc chắn bác sĩ được báo, nhanh, không trùng, không làm chậm việc lưu?"*

## Mục lục

- [Phần 1 — Bài toán: 6 đặc tính](#phần-1--bài-toán-6-đặc-tính)
- [Phần 2 — 6 ứng viên](#phần-2--6-ứng-viên)
- [Phần 3 — Bảng so sánh](#phần-3--bảng-so-sánh)
- [Phần 4 — Khi nào đúng](#phần-4--khi-nào-đúng)
- [Phần 5 — Lối tắt sai](#phần-5--lối-tắt-sai)
- [Phần 6 — Medplum tại từng ngã rẽ](#phần-6--medplum-tại-từng-ngã-rẽ)
- [Phần 7 — Đánh đổi bằng số](#phần-7--đánh-đổi-bằng-số)
- [Phần 8 — Đối chứng nội bộ](#phần-8--đối-chứng-nội-bộ)
- [Phần 9 — Phản biện](#phần-9--phản-biện)
- [Phần 10 — Checklist](#phần-10--checklist)

Nhãn: ✅ code · 📘 chuẩn · 💭 phản thực tế/ước tính · 🧠 suy luận.

**Kịch bản cố định:** lab gửi `Observation` kali = 6,8 mmol/L (nguy hiểm) cho bà Lan lúc 02:00. BS trực phải biết trong 15 phút. Endpoint nhắn tin của đối tác đang chập chờn.

---

## Phần 1 — Bài toán: 6 đặc tính

| # | Đặc tính | Vì sao |
|---|---|---|
| E1 | **Không mất** sự kiện | Mất 1 cảnh báo kali là rủi ro tính mạng |
| E2 | **Không làm chậm / không làm hỏng** việc lưu | Lab gửi hàng nghìn kết quả/giờ. Webhook chậm không được chặn ghi |
| E3 | Lọc theo **chuyển trạng thái**, không theo mọi lần ghi | Chỉ báo khi kết quả vừa `final`, không báo lại khi sửa ghi chú |
| E4 | Thử lại khi bên nhận lỗi, có giới hạn | Đối tác sập vài giờ |
| E5 | Người dùng/vận hành tự khai báo, không deploy | Thêm quy trình mới mà không chờ dev |
| E6 | Tôn trọng tenant & phân quyền | Không gửi dữ liệu phòng khám A cho webhook của phòng khám B |

## Phần 2 — 6 ứng viên

### 2.1 Polling: bên nhận tự hỏi định kỳ 💭

`GET Observation?_lastUpdated=gt<lần trước>&code=…` mỗi 1 phút.

- E1 tốt nếu bên nhận nhớ mốc thời gian đúng (dùng cursor, [1b mục 8](./1b-search.md#8-phân-trang-và-đếm-tổng)). E2 tốt, server không làm gì thêm.
- Trễ = chu kỳ hỏi. Tốn tài nguyên khi không có gì mới. E3 phải tự so phiên bản.

### 2.2 Hook đồng bộ trong request — OpenEMR ✅

Sự kiện PHP (Symfony EventDispatcher) `src/Events/Patient/PatientCreatedEvent.php:21`, `src/Events/Services/ServiceSaveEvent.php:18`. Listener chạy **trong** request lưu dữ liệu.

- E3 tốt (có cả dữ liệu cũ lẫn mới trong tay).
- **E2 vỡ:** listener gọi HTTP chậm thì người dùng chờ. Listener ném lỗi thì có thể làm hỏng việc lưu.
- **E1:** request lỗi giữa chừng thì mất cả sự kiện. Không thử lại.
- E5: listener là code PHP, phải deploy.

### 2.3 Trigger DB / LISTEN-NOTIFY 💭

Trigger Postgres gửi `NOTIFY` khi bảng đổi. Worker `LISTEN`.

- E1: `NOTIFY` không lưu lại. Worker mất kết nối là mất sự kiện.
- E3, E6 khó: trigger chỉ thấy dòng SQL, không thấy ngữ nghĩa FHIR (mà [1a](./1a-fhir-datastore.md) cho thấy là JSON trong `content`).

### 2.4 Hàng đợi sau commit — Medplum ✅

Commit DB, rồi đưa job vào BullMQ/Redis (`fhir/repo.ts:1100-1103` → `workers/dispatch.ts:85-109`). Worker lọc (`packages/core/src/subscriptions/index.ts:727-800`), giao, thử lại với backoff (`workers/subscription.ts:85-93`, `:178-183`).

- E2 tốt: request không chờ.
- E3 tốt: FHIRPath `%previous`/`%current`.
- E4 tốt: tới 19 lần, trần 8 giờ, tự tắt khi lỗi kéo dài.
- E5 tốt: `Subscription` là resource.
- E6: kiểm tra account (tenant) có. AccessPolicy cho rest-hook **chưa chặn thật** (`workers/subscription.ts:230`, `:308-309`).
- **E1 có khe hở:** enqueue sau commit, lỗi enqueue chỉ log (`workers/index.ts:77-85`). Tiến trình chết đúng giữa commit và enqueue thì mất sự kiện.

### 2.5 Transactional outbox — HAPI ✅

Thông điệp "resource đã đổi" được **lưu trong DB** (`HFJ_RESOURCE_MODIFIED`, `ResourceModifiedEntity.java:43-45`), sau đó mới đẩy ra kênh. Gửi lỗi thì vẫn còn trong bảng để thử lại.

- **E1 tốt nhất** trong các cách tự làm: thông điệp và dữ liệu cùng sống trong DB.
- Thêm 1 lần ghi DB cho mỗi thay đổi. Cần job dọn bảng.
- E3: qua SubscriptionTopic, so `previous` (`SubscriptionTriggerMatcher.java:89-111`).

### 2.6 CDC (Change Data Capture) + message bus 💭📘

Đọc WAL của Postgres (VD Debezium) → Kafka → consumer.

- E1, E2 rất tốt: không đụng vào ứng dụng, không mất nếu WAL còn.
- E3, E5, E6 khó: dữ liệu ở mức dòng SQL, phải dựng lại ngữ nghĩa FHIR và phân quyền ở consumer. Hạ tầng nặng (Kafka, connector).

## Phần 3 — Bảng so sánh

| Ứng viên | E1 không mất | E2 không chặn ghi | E3 chuyển trạng thái | E4 thử lại | E5 tự khai báo | E6 tenant/quyền |
|---|---|---|---|---|---|---|
| 2.1 Polling | ✅ nếu dùng cursor | ✅ | ◐ tự so | — (bên nhận lo) | ✅ | ✅ (qua API có quyền) |
| 2.2 Hook đồng bộ (OpenEMR) | ✗ | ✗ | ✅ | ✗ | ✗ | ◐ |
| 2.3 Trigger / NOTIFY | ✗ | ✅ | ✗ | ✗ | ✗ | ✗ |
| 2.4 Hàng đợi sau commit (Medplum) | ◐ khe enqueue | ✅ | ✅ | ✅ | ✅ | ◐ |
| 2.5 Outbox (HAPI) | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ |
| 2.6 CDC | ✅ | ✅ | ✗ | ✅ | ✗ | ✗ |

## Phần 4 — Khi nào đúng

| Ứng viên | Đúng khi |
|---|---|
| 2.1 | Bên nhận là hệ thống đồng bộ theo lô (BI, kho dữ liệu), chấp nhận trễ vài phút |
| 2.2 | Logic nhỏ, nhanh, **phải** cùng giao dịch (VD cập nhật trường tính toán) |
| 2.3 | Gần như không bao giờ cho dữ liệu FHIR |
| 2.4 | Đa số quy trình tự động hoá: nhắn tin, tạo Task, gọi đối tác |
| 2.5 | Quy trình không được phép mất sự kiện, và tự vận hành server FHIR |
| 2.6 | Tổ chức lớn đã có nền tảng streaming, cần đổ dữ liệu sang nhiều hệ thống phân tích |

🧠 Thực tế tốt nhất là **ghép**: 2.4 cho tốc độ + 2.1 (quét định kỳ) làm lưới an toàn cho quy trình nguy hiểm. Đây là mô hình "push để nhanh, pull để chắc".

## Phần 5 — Lối tắt sai

**A — *"Gọi webhook ngay trong API lưu kết quả, cho đơn giản."*** Đây là 2.2. Đối tác chậm 30 giây thì lab bị timeout khi gửi kết quả, rồi gửi lại, rồi sinh kết quả trùng.

**B — *"Subscription có retry 19 lần rồi, chắc chắn không mất."*** Retry chỉ bảo vệ **sau** khi job đã vào hàng đợi. Khe giữa commit và enqueue (2.4) không được retry che.

**C — *"`criteria` = `Observation` cho chắc, lọc trong bot."*** Mỗi lần ghi bất kỳ Observation là 1 job + 1 lần chạy bot. Với lab gửi hàng nghìn kết quả/giờ, hàng đợi phình và cảnh báo thật bị trễ sau hàng nghìn job vô ích.

**D (leo quá xa) — *"Dựng Kafka + Debezium cho MVP."*** Chưa có bằng chứng về quy mô hay số hệ thống tiêu thụ. Subscription + quét định kỳ là đủ.

## Phần 6 — Medplum tại từng ngã rẽ

| Ngã rẽ | Medplum chọn | ✅ |
|---|---|---|
| Tránh 2.2 (chặn request) | Mọi xử lý sự kiện sau commit, ngoài request | `repo.ts:1100-1103` |
| Tránh 2.3 (mức SQL) | Sự kiện mang `resourceType`, `id`, `versionId`, `previousVersionId`, `interaction` | `workers/dispatch.ts:31-43`, `:85-101` |
| Tránh 2.6 (hạ tầng nặng) | Dùng Redis + BullMQ vốn đã có cho cache | `workers/dispatch.ts:45-70` |
| Khác 2.5 | **Không** lưu thông điệp trong DB, enqueue thẳng vào Redis | `workers/index.ts:72-86` |

🧠 Lý do khả dĩ cho ngã rẽ cuối: tránh 1 lần ghi DB thêm cho **mọi** thay đổi. Đây là cùng triết lý "tối ưu đường ghi thường xuyên" đã thấy ở 1a, 1c. Cái giá là khe E1.

## Phần 7 — Đánh đổi bằng số

💭 Với 20 phòng khám ở [1a-C Phần 1](./1a-comparison-fhir-datastore.md#hậu-quả-nếu-bỏ-qua-bằng-số--ước-tính): khoảng 7,2 triệu Observation/năm, khoảng 24.000/ngày làm việc.

| | Outbox (2.5) | Hàng đợi sau commit (2.4) |
|---|---|---|
| Ghi DB thêm mỗi thay đổi | +1 dòng (24.000/ngày) + job dọn | 0 |
| Xác suất mất sự kiện | Gần 0 | = xác suất (Redis lỗi **hoặc** tiến trình chết) trong khe vài mili-giây sau commit |
| Ước tính 💭 | — | Nếu 1 sự cố/tháng làm mất các job trong khoảng 1 giây: khoảng 0–1 sự kiện mất/tháng |

🧠 Mất 1 sự kiện/tháng là chấp nhận được cho tin nhắn nhắc lịch, **không** chấp nhận được cho cảnh báo kali. Do đó quyết định không nằm ở chọn 1 kỹ thuật cho cả hệ thống, mà là **thêm lưới an toàn cho đúng quy trình nguy hiểm**.

## Phần 8 — Đối chứng nội bộ

Ngay trong Medplum, **websocket subscription** đi đường khác hẳn rest-hook:
- Lưu trong Redis, không lưu DB (`workers/subscription.ts:344-348`).
- **Có** áp AccessPolicy (`:308-309`).
- Không retry (giao cho kết nối đang mở).

🧠 Hợp lý vì websocket phục vụ **giao diện đang mở** (VD danh sách bệnh nhân tự cập nhật). Mất 1 thông báo thì lần tải lại sau sẽ có dữ liệu đúng. Không cần đảm bảo giao như rest-hook.

## Phần 9 — Phản biện

**1 — Trưởng khoa xét nghiệm:** *"Hệ thống cảnh báo bằng webhook mà có thể mất thì tôi không ký duyệt."*

<details><summary>Gợi ý</summary>

Đồng ý là không được phụ thuộc 1 kênh. Đề xuất 2 lớp: Subscription cho tốc độ (giây), và Bot chạy cron 5 phút quét kết quả `final` nguy hiểm **chưa có Task xác nhận**, tạo cảnh báo bổ sung. Đo tỉ lệ lớp 2 bắt được sự kiện mà lớp 1 sót để chứng minh.
</details>

**2 — Dev:** *"Sao không dùng outbox như HAPI cho chắc?"*

<details><summary>Gợi ý</summary>

Nếu tự host Medplum và có quyền sửa server thì là lựa chọn hợp lý, nhưng phải duy trì bản sửa riêng. Lưới an toàn bằng cron đạt mục tiêu E1 cho quy trình nguy hiểm mà không phải fork nền tảng.
</details>

**3 — Đối tác nhận webhook:** *"Chúng tôi nhận trùng thông báo."*

<details><summary>Gợi ý</summary>

Retry nghĩa là "ít nhất 1 lần", không phải "đúng 1 lần". Bên nhận phải **idempotent**: dùng `resourceType/id` + `versionId` làm khoá chống trùng. Payload chứa đúng phiên bản (worker đọc theo `versionId`).
</details>

## Phần 10 — Checklist

- [ ] Quy trình này mất 1 sự kiện thì hậu quả gì? Nếu nghiêm trọng thì thêm quét định kỳ làm lưới an toàn.
- [ ] `criteria` đủ hẹp? Có dùng `%previous` để chỉ bắt chuyển trạng thái?
- [ ] Đã khai báo `subscription-supported-interaction` (không báo khi xoá nếu không cần)?
- [ ] Endpoint ngoài xác minh `X-Signature` và xử lý trùng (idempotent)?
- [ ] Có theo dõi Subscription bị tự tắt (`status=off`)?
- [ ] Dữ liệu gửi đi có vượt quá quyền của người tạo Subscription không (rest-hook chưa áp AccessPolicy)?
