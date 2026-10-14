# 2a — Subscription: báo cho hệ thống khác khi dữ liệu thay đổi

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Trả lời câu: *"Có kết quả xét nghiệm bất thường thì làm sao bác sĩ, app bệnh nhân, hệ thống báo cáo **biết ngay**, không phải chờ ai mở màn hình?"* So sánh kỹ thuật: [`2a-comparison-event-delivery.md`](./2a-comparison-event-delivery.md).
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. Vì sao y tế cần "đẩy" thay vì "kéo"](#1-vì-sao-y-tế-cần-đẩy-thay-vì-kéo)
- [2. Resource Subscription](#2-resource-subscription)
- [3. Đường đi của 1 sự kiện](#3-đường-đi-của-1-sự-kiện)
- [4. Điều kiện khớp: 4 lớp lọc](#4-điều-kiện-khớp-4-lớp-lọc)
- [5. Giao hàng: rest-hook, Bot, websocket](#5-giao-hàng-rest-hook-bot-websocket)
- [6. Thử lại, tự tắt, bảo mật](#6-thử-lại-tự-tắt-bảo-mật)
- [7. Đối chiếu OpenEMR và HAPI](#7-đối-chiếu-openemr-và-hapi)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vì sao y tế cần "đẩy" thay vì "kéo"

🧠 Ví dụ quy trình phụ thuộc sự kiện:

| Sự kiện | Ai cần biết | Trễ bao lâu thì có hại |
|---|---|---|
| Kết quả kali máu nguy hiểm | Bác sĩ trực | Vài phút |
| Bệnh nhân điền xong phiếu tiếp nhận | Lễ tân, bot tạo hồ sơ | Vài phút |
| Lịch hẹn bị huỷ | Hệ thống nhắn tin, slot trống cho người khác | Vài giờ |
| Claim được tạo | Hệ thống thanh toán | 1 ngày |

📘 FHIR có resource `Subscription`: khai báo "khi resource khớp điều kiện X thay đổi, gửi tới kênh Y". Docs Medplum (`packages/docs/docs/fhir-basics.md`, mục Subscriptions) gọi đây là cơ chế "giống webhook", thường ghép với **Bot** để tự động hoá quy trình (module 2b).

## 2. Resource Subscription

💭 Ví dụ: báo cho bot cảnh báo mỗi khi có kết quả kali được chốt hoặc sửa:

```json
{
  "resourceType": "Subscription",
  "status": "active",
  "reason": "Cảnh báo kali bất thường",
  "criteria": "Observation?code=http://loinc.org|2823-3",
  "channel": { "type": "rest-hook", "endpoint": "Bot/canh-bao-kali" },
  "extension": [
    { "url": "https://medplum.com/fhir/StructureDefinition/subscription-supported-interaction", "valueCode": "create" },
    { "url": "https://medplum.com/fhir/StructureDefinition/subscription-supported-interaction", "valueCode": "update" },
    { "url": "https://medplum.com/fhir/StructureDefinition/fhir-path-criteria-expression",
      "valueString": "%current.status = 'final' and %previous.status != 'final'" },
    { "url": "https://www.medplum.com/fhir/StructureDefinition/subscription-secret", "valueString": "…" },
    { "url": "https://medplum.com/fhir/StructureDefinition/subscription-max-attempts", "valueInteger": 10 }
  ]
}
```

Các extension do Medplum định nghĩa thêm (📘 docs `packages/docs/docs/subscriptions/subscription-extensions.md`). URL lấy từ code ở các mục dưới.

## 3. Đường đi của 1 sự kiện

✅

```
PUT Observation/k-1   (status: preliminary → final)
  │
  ├─ Repository.updateResourceImpl ... writeToDatabase (transaction)    repo.ts:1097
  │
  └─ SAU commit: addBackgroundJobs                                      repo.ts:1100-1103
        └─ addDispatchJobs → hàng đợi BullMQ "dispatch" (Redis)         workers/dispatch.ts:85-109
              │   lỗi ở bước này chỉ được GHI LOG                        workers/index.ts:77-85
              ▼
        execDispatchJob                                                 workers/dispatch.ts:115+
              ├─ đọc lại đúng phiên bản (versionId) và bản trước đó     :121-125
              ├─ addSubscriptionJobs   → hàng đợi "subscription"        workers/subscription.ts:329
              └─ addCronJobs …                                           (lịch chạy Bot)
                     │
                     ▼
        execSubscriptionJob                                             workers/subscription.ts:616
              ├─ rest-hook → sendRestHook (HTTP POST)                    :739
              └─ endpoint "Bot/…" → execBot                              :678, :886
```

🧠 2 điểm thiết kế:
1. **Mọi xử lý sự kiện chạy sau commit, ngoài request.** Người dùng không phải chờ webhook. 1 webhook chậm không làm chậm việc lưu kết quả xét nghiệm.
2. **Worker đọc lại theo `versionId`**, không dùng dữ liệu trong bộ nhớ của request. Đến lúc worker chạy, resource có thể đã đổi tiếp, nhưng mỗi sự kiện vẫn mang đúng phiên bản gây ra nó.

## 4. Điều kiện khớp: 4 lớp lọc

✅ `packages/core/src/subscriptions/index.ts:727-800` (`resourceMatchesSubscriptionCriteria`), theo thứ tự:

| Lớp | Kiểm tra | Code |
|---|---|---|
| 1 | Đúng loại resource trong `criteria` | `:745-751` |
| 2 | **FHIRPath** với `%current` và `%previous` (extension `fhir-path-criteria-expression`) | `:753-757`, hàm `isFhirCriteriaMet` `:832-856` |
| 3 | Đúng **loại thao tác** (`create`/`update`/`delete`, extension `subscription-supported-interaction`). Không khai báo thì nhận cả 3 | `:759-778` |
| 4 | Khớp câu search `criteria` (so khớp **trong bộ nhớ**, `matchesSearchRequest`) | `:780-782` |
| 5 | Nếu Subscription có `meta.accounts`, resource phải chung ít nhất 1 account (tenant) | `:784-797` |

🧠 Lớp 2 (`%previous`) là thứ biến Subscription từ "báo mọi lần ghi" thành "báo khi **chuyển trạng thái**". VD chỉ báo khi kết quả **vừa** chuyển sang `final`, không báo lại mỗi lần sửa ghi chú. Đây là chỗ phần lớn quy trình y tế cần.

Lớp 4 dùng bộ máy search **trong bộ nhớ**, không phải SQL ([1b mục 9](./1b-search.md#9-hai-bản-hiện-thực-của-cùng-1-ngôn-ngữ-search)). Modifier nào bộ máy này không hỗ trợ thì không dùng được trong `criteria`.

## 5. Giao hàng: rest-hook, Bot, websocket

| Kênh | Hành vi | Code |
|---|---|---|
| `rest-hook` tới URL ngoài | HTTP POST body là resource (xoá thì `{}` + header `X-Medplum-Deleted-Resource`). Timeout 120 giây | `workers/subscription.ts:739-760`, `:80`, `:847` |
| `rest-hook` tới `Bot/<id>` | Chạy Bot trong Medplum, không ra mạng | `:678`, `:886-935` |
| `websocket` | Đẩy qua kết nối websocket đang mở (app đang chạy). Lưu và tra Subscription trong Redis, bật bằng feature `websocket-subscriptions` | `:344-348`, `:573`, `ws/subscriptions.ts` |

**URL ngoài phải nằm trong danh sách cho phép** (`:418-421`, `isAllowedOutboundUrlForQueue`). 🧠 Chặn việc dùng Subscription để gọi vào mạng nội bộ của server (SSRF).

## 6. Thử lại, tự tắt, bảo mật

✅ `workers/subscription.ts`:

| Cơ chế | Chi tiết | Code |
|---|---|---|
| Số lần thử | Mặc định **4**, tối đa **19** (đổi bằng extension `subscription-max-attempts`) | `:87-93`, `:936-948` |
| Giãn cách | Exponential backoff có jitter, trần **8 giờ**. 19 lần ≈ 72–80 giờ | `:85`, `:178-183` |
| Lỗi phía server Medplum (chưa đọc được subscription/resource) | Đếm riêng, cho thử nhiều hơn | `:97-100`, `:666` |
| Tự tắt | Lỗi liên tiếp vượt ngưỡng trong cửa sổ thời gian thì đổi `status` thành `off`, ghi `error` | `:967-1018` |
| Bỏ qua bản cũ | Lần thử lại mà resource đã có phiên bản mới thì cân nhắc bỏ (`currentVersion.meta.versionId !== versionId`) | `:654` |
| Chữ ký | Có `subscription-secret` thì gửi header `X-Signature` = HMAC-SHA256 của body | `:855-862` |
| Nhật ký | Mỗi lần giao tạo AuditEvent `transmit` | `util/auditevent.ts:410` |

🧠 Điểm yếu cần biết, dựa trên luồng ở mục 3:
- Việc đưa vào hàng đợi chạy **sau** commit, và lỗi ở bước đó chỉ được log (`workers/index.ts:77-85`). Nếu Redis không nhận job đúng lúc đó, hoặc tiến trình chết giữa commit và enqueue, **sự kiện mất** dù dữ liệu đã lưu.
- Quy trình quan trọng (VD cảnh báo kết quả nguy hiểm) nên có **lưới an toàn**: 1 Bot chạy định kỳ (cron) quét `Observation?_lastUpdated=gt…&status=final` để bắt sự kiện bị sót. Xem so sánh ở [`2a-comparison`](./2a-comparison-event-delivery.md).
- Comment trong code (`:230`) ghi rõ: với rest-hook, việc kiểm tra AccessPolicy của người tạo Subscription **chưa thực sự chặn** thông báo. Chỉ websocket mới áp policy (`:308-309`).

## 7. Đối chiếu OpenEMR và HAPI

**OpenEMR** ✅: không có Subscription FHIR ra ngoài. Có **sự kiện trong tiến trình PHP** (Symfony EventDispatcher), thư mục `src/Events/`: `Patient/PatientCreatedEvent.php:21`, `Services/ServiceSaveEvent.php:18`, cùng các nhóm `Appointments`, `Billing`, `Encounter`, `Messaging`… Module PHP đăng ký lắng nghe và chạy **ngay trong request**. Webhook có sẵn chỉ là chiều **vào** cho cổng thanh toán (`src/PaymentProcessing/Rainforest/Webhooks/`).

**HAPI** ✅: có Subscription FHIR, và có bảng **`HFJ_RESOURCE_MODIFIED`** (`hapi-fhir-jpaserver-model/.../entity/ResourceModifiedEntity.java:43-45`, cột `SUMMARY_MESSAGE`, `CREATED_TIME`). Đây là bảng lưu thông điệp "resource đã thay đổi" **trong DB**, để giao lại nếu bước gửi vào kênh lỗi. Bộ lọc khớp nằm ở `hapi-fhir-jpaserver-subscription/.../SubscriptionMatcherInterceptor.java`.

| | Medplum | OpenEMR | HAPI |
|---|---|---|---|
| Cơ chế | Hàng đợi Redis (BullMQ), sau commit | Event PHP trong request | Subscription + bảng thông điệp trong DB |
| Ra ngoài hệ thống | rest-hook, websocket | Tự viết trong module | rest-hook, email, message (các lớp `SubscriptionDelivering{RestHook,Email,Message}Listener`), websocket |
| Điều kiện chuyển trạng thái | Có (`%previous` trong FHIRPath) | Tự viết | Có, qua SubscriptionTopic: `queryCriteria.previous` được so trên phiên bản trước (`hapi-fhir-jpaserver-subscription/.../topic/SubscriptionTriggerMatcher.java:89-111`) |
| Mất sự kiện khi lỗi giữa chừng | Có thể (mục 6) | Mất cùng request | Giảm nhờ lưu thông điệp trong DB |

## Sai lầm thường gặp

1. **`criteria` quá rộng** (`Observation`) rồi lọc trong Bot. Mỗi lần ghi Observation là 1 job. Đẩy điều kiện vào `criteria` và FHIRPath.
2. **Không dùng `%previous`**, nên bot gửi tin nhắn cho bệnh nhân **mỗi lần** bác sĩ sửa ghi chú, không chỉ lần đầu kết quả có.
3. **Endpoint ngoài không xác minh `X-Signature`.** Ai biết URL cũng giả được thông báo.
4. **Coi Subscription là cơ chế đảm bảo tuyệt đối** cho cảnh báo lâm sàng nguy hiểm. Cần lưới an toàn bằng quét định kỳ (mục 6).
5. **Không theo dõi Subscription bị tự tắt.** Sau chuỗi lỗi, `status = off` và **không báo ai**. Cần 1 Subscription/cron theo dõi `Subscription?status=off`.

## Tự kiểm tra

1. Viết FHIRPath để chỉ báo khi `Appointment.status` chuyển từ `booked` sang `cancelled`.
2. Endpoint của đối tác sập 2 ngày. Với cấu hình mặc định, sự kiện trong 2 ngày đó còn được giao không?
3. Vì sao worker đọc lại resource theo `versionId` thay vì nhận resource trong payload của job?
