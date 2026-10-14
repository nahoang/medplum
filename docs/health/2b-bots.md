# 2b — Bot: logic nghiệp vụ tuỳ biến chạy trên nền tảng

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Trả lời câu: *"Mỗi phòng khám có quy trình riêng (nhắn tin nhắc lịch, tính BMI, chuyển phiếu tiếp nhận thành hồ sơ, gửi claim cho đối tác). Viết logic đó ở đâu mà không phải sửa code nền tảng?"* So sánh kỹ thuật: [`2b-comparison-extensibility.md`](./2b-comparison-extensibility.md).
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. Bot là gì](#1-bot-là-gì)
- [2. 6 cách kích hoạt 1 Bot](#2-6-cách-kích-hoạt-1-bot)
- [3. 3 môi trường chạy](#3-3-môi-trường-chạy)
- [4. Danh tính, quyền và bí mật của Bot](#4-danh-tính-quyền-và-bí-mật-của-bot)
- [5. Bot pre-commit: chặn hoặc sửa dữ liệu trước khi lưu](#5-bot-pre-commit-chặn-hoặc-sửa-dữ-liệu-trước-khi-lưu)
- [6. Bot mẫu cho thấy nghiệp vụ gì](#6-bot-mẫu-cho-thấy-nghiệp-vụ-gì)
- [7. Đối chiếu OpenEMR và HAPI](#7-đối-chiếu-openemr-và-hapi)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Bot là gì

📘 `Bot` là resource riêng của Medplum (`packages/definitions/src/fhir/r4/profiles-medplum.json`). Trường chính:

| Trường | Ý nghĩa |
|---|---|
| `sourceCode`, `executableCode` | Mã TypeScript gốc và bản đã build (lưu dạng `Attachment`) |
| `runtimeVersion` | `vmcontext`, `awslambda` hoặc `fission` |
| `timeout` | Thời gian chạy tối đa (giây) |
| `cron[x]` | Lịch chạy định kỳ |
| `system` | Bot hệ thống, đọc được bí mật cấp hệ thống |
| `runAsUser` | Chạy bằng quyền của người gọi thay vì quyền của Bot |
| `publicWebhook`, `rawBody` | Cho phép gọi không cần đăng nhập (webhook từ bên ngoài) |
| `cdsService` | Khai báo Bot là dịch vụ CDS Hooks (`hook`, `prefetch`…) |
| `auditEventTrigger`, `auditEventDestination` | Khi nào ghi AuditEvent (`always`, `on-error`…) và ghi vào log hay resource |

🧠 Bot là **hàm** nhận 1 sự kiện (`BotEvent`: `input`, `contentType`, `secrets`, `headers`…), có sẵn 1 `MedplumClient` đã xác thực để đọc/ghi FHIR. Giống "serverless function", nhưng danh tính và quyền nằm trong mô hình FHIR của Medplum.

Bot chỉ chạy được khi project bật feature `bots` (`bots/utils.ts:160-164`).

## 2. 6 cách kích hoạt 1 Bot

| Cách | Khi nào dùng | ✅ |
|---|---|---|
| **Subscription** `endpoint = Bot/<id>` | Dữ liệu thay đổi → chạy logic | `workers/subscription.ts:678`, `:886` |
| **`$execute`** gọi trực tiếp | App gọi 1 nghiệp vụ phức tạp như gọi API | `fhir/routes.ts:199-208` (`/Bot/:id/$execute`, `/Bot/$execute?identifier=`) |
| **Cron** (`Bot.cron`) | Việc định kỳ: nhắc tái khám, quét kết quả chưa duyệt | `workers/cron.ts`, được dispatch đưa vào lịch (`workers/dispatch.ts`, `addCronJobs`) |
| **Webhook công khai** (`publicWebhook`) | Bên ngoài gọi vào (Stripe, Twilio, đối tác lab) | `webhook/routes.ts:44-47` |
| **CDS Hooks** (`cdsService`) | Gợi ý lâm sàng khi bác sĩ mở hồ sơ, kê đơn | `cds/routes.ts:31-37` |
| **Pre-commit** | Chặn hoặc sửa dữ liệu **trước** khi lưu | `fhir/precommit.ts` (mục 5) |

Ngoài ra, **Agent** (module 2c) chuyển tin nhắn HL7 v2 từ bệnh viện vào Bot.

## 3. 3 môi trường chạy

✅ `bots/execute.ts:27-79` (`executeBot`) chọn theo `runtimeVersion`:

| Runtime | Chạy ở đâu | Ghi chú |
|---|---|---|
| `awslambda` | AWS Lambda, mỗi Bot 1 function | Mặc định của bản hosted. Hỗ trợ streaming (`:47-54`). Xoá Bot thì xoá luôn Lambda (`workers/dispatch.ts`) |
| `vmcontext` | Trong tiến trình Node của server, sandbox `vm.runInNewContext` | `bots/vmcontext.ts:158`. Timeout mặc định **10 giây** (`:48`, `:105-106`). Dùng khi tự host không có Lambda |
| `fission` | Fission (serverless trên Kubernetes) | Cho tự host trên K8s |

Sau mỗi lần chạy: đo thời gian (metric `medplum.bot.execute.time`, `:71-72`), ghi AuditEvent (`:74-79`, `createBotAuditEvent`).

🧠 `vmcontext` chạy chung tiến trình với server. Bot vòng lặp vô hạn hay ăn hết bộ nhớ có thể ảnh hưởng **mọi** người dùng. Đây là lý do bản hosted dùng Lambda để cách ly.

## 4. Danh tính, quyền và bí mật của Bot

✅ `bots/execute.ts:41-45`: trước khi chạy, server tạo:
- `accessToken` cho Bot (`getBotAccessToken`), mặc định chạy bằng **membership của Bot**, hoặc của người gọi nếu `runAsUser`.
- `secrets` (`getBotSecrets`, `bots/utils.ts:344-356`): gộp `Project.secret` của project chứa Bot và project đang chạy. Thêm `systemSecret` nếu `Bot.system = true`.

🧠 Hệ quả:
1. **Bot có `ProjectMembership` và `AccessPolicy` như người** ([1c](./1c-access-control.md)). Bot gửi tin nhắn chỉ cần đọc `Patient`, `Appointment`. Không nên cho đọc `Condition`.
2. **API key đối tác** (Twilio, Stripe, lab) để trong `Project.secret`, không viết trong mã Bot.
3. Mọi thao tác Bot ghi đều mang `meta.author` = Bot. Lịch sử phân biệt được "người sửa" và "máy sửa".

## 5. Bot pre-commit: chặn hoặc sửa dữ liệu trước khi lưu

✅ `fhir/precommit.ts`:
- Là 1 `Subscription` có extension `https://medplum.com/fhir/StructureDefinition/pre-commit-bot` = `true` (`:27-31`).
- Chỉ chạy khi **cả** cấu hình server `preCommitSubscriptionsEnabled` (`config/types.ts:149`) **và** setting cùng tên của project được bật (`:64-68`).
- Được gọi **đồng bộ** trong đường ghi, trước khi lưu (`fhir/repo.ts:1009`).
- Bot lỗi thì **từ chối** việc ghi với OperationOutcome do Bot trả (`precommit.ts:128-142`). Bot trả về resource đã sửa thì resource đó được lưu (`repo.ts:1011-1017`).

💭 Ví dụ nghiệp vụ:
- Từ chối `MedicationRequest` nếu bệnh nhân có `AllergyIntolerance` với hoạt chất đó.
- Tự gắn `Observation.interpretation = H` khi giá trị vượt ngưỡng.
- Chặn tạo `Patient` trùng số CCCD.

🧠 Pre-commit đổi **an toàn dữ liệu** lấy **độ trễ ghi**: mỗi lần ghi phải chờ Bot. Bot chậm hoặc lỗi mạng làm **mọi** thao tác ghi loại resource đó thất bại. Vì vậy có 2 khoá bật/tắt.

Kèm theo là 1 kiểm tra cứng không cần Bot: không cho xoá resource mà `ProjectMembership` đang trỏ tới, như `profile` hoặc `accessPolicy` (`precommit.ts:48-58`, `:177+`).

## 6. Bot mẫu cho thấy nghiệp vụ gì

✅ `examples/medplum-demo-bots/src/` có khoảng 50 Bot mẫu. Nhóm theo nghiệp vụ:

| Nghiệp vụ | Bot mẫu |
|---|---|
| Tiếp nhận | `patient-intake.ts`, `questionnaire-bots/`, `sick-note-questionnaire-bot/`, `form-data-upload.ts` |
| Lịch hẹn & nhắn tin | `appointment-bots/`, `auto-responder-bots/`, `twilio/`, `zoom-bots/`, `slack-bots/` |
| Lâm sàng | `bmi-calculation-bot.ts`, `finalize-report.ts`, `cds-hooks-via-proxy.ts` |
| Xét nghiệm & liên thông | `lab-integration/`, `hl7-integration/`, `hl7-bot.ts`, `adt-transfer-listener/`, `dft-transfer-listener/`, `ccda-converter/`, `epic/` |
| Tài chính | `billing-bots/`, `candid-health/`, `stripe-bots/`, `eligibility-check-opkit.ts` |
| Dữ liệu | `deduplication/`, `nppes-lookup.ts` (tra cứu mã định danh nhà cung cấp Mỹ), `textract-bot.ts` (OCR) |
| Hạ tầng | `sftp-upload.ts`, `http-proxy.ts`, `create-pdf.ts` |

🧠 Danh sách này là **bản đồ những thứ Medplum không làm sẵn**. Mọi tích hợp đối tác (thanh toán, nhắn tin, lab, EHR khác) đều đi qua Bot. Nghiệp vụ thật của 1 sản phẩm dựng trên Medplum phần lớn nằm ở **Bot của khách hàng**, không nằm ở server Medplum.

## 7. Đối chiếu OpenEMR và HAPI

| | Medplum | OpenEMR | HAPI |
|---|---|---|---|
| Cơ chế | Bot (TypeScript), chạy ngoài tiến trình (Lambda) hoặc sandbox | **Module PHP** trong tiến trình: `interface/modules/custom_modules/` (VD `oe-module-comlink-telehealth`, `oe-module-ehi-exporter`) + nghe Event (`src/Events/`) | **Interceptor Java** trong tiến trình: `@Hook` (`hapi-fhir-base/.../interceptor/api/Hook.java:37`) gắn vào khoảng 110 `Pointcut` (VD `STORAGE_PRESTORAGE_RESOURCE_CREATED`, `STORAGE_PRECOMMIT_RESOURCE_CREATED` ở `Pointcut.java:1644`, `:1847`) |
| Triển khai | Gửi mã qua API (`$deploy`), không restart server | Cài module, có thể cần restart/nâng cấp | Đóng gói JAR, restart server |
| Ai viết được | Dev của khách hàng | Dev PHP có quyền vào server | Dev Java vận hành server |
| Cách ly | Lambda: tách hẳn. `vmcontext`: sandbox, chung tiến trình | Không | Không |
| Phân quyền | Bot có membership + AccessPolicy | Toàn quyền như code lõi | Toàn quyền |

🧠 3 vị trí trên trục "**ai được tuỳ biến**":
- OpenEMR, HAPI: người **vận hành server**.
- Medplum: **khách hàng** của nền tảng, trong ranh giới quyền.

Đó là điều kiện cần để làm SaaS nhiều tenant: tenant A tuỳ biến mà không đụng tenant B.

## Sai lầm thường gặp

1. **Nhét API key đối tác vào mã Bot.** `sourceCode` là resource, ai có quyền đọc Bot là đọc được key. Dùng `Project.secret`.
2. **Cho Bot AccessPolicy `*`.** Bot bị lỗi logic hoặc bị lợi dụng qua webhook công khai thì phá toàn project.
3. **Bot `publicWebhook` không xác minh chữ ký** của bên gọi (Stripe, Twilio đều có chữ ký). Ai biết URL cũng gọi được.
4. **Đặt logic nặng vào pre-commit.** Gọi API ngoài trong pre-commit thì đối tác chậm = toàn hệ thống không ghi được.
5. **Bot không idempotent.** Subscription có retry ([2a](./2a-subscriptions.md#6-thử-lại-tự-tắt-bảo-mật)). Bot gửi SMS mà không kiểm tra "đã gửi chưa" sẽ gửi trùng.
6. **Dùng `vmcontext` cho Bot chạy lâu.** Mặc định 10 giây, và chiếm tài nguyên server chung.

## Tự kiểm tra

1. Bệnh nhân điền xong phiếu tiếp nhận (`QuestionnaireResponse`), cần tạo `Patient`, `Coverage`, `AllergyIntolerance`. Chọn cách kích hoạt nào, Bot cần quyền gì?
2. Muốn chặn kê đơn thuốc mà bệnh nhân dị ứng. Pre-commit hay Subscription? Đánh đổi là gì?
3. Vì sao bản hosted chạy Bot trên Lambda thay vì `vmcontext`?
