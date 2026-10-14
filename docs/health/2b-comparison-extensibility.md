# 2b-C — So sánh 6 cách cho khách hàng tuỳ biến logic nghiệp vụ

> **Loại file:** C. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d` · HAPI FHIR `6d818ea`.
>
> Nguồn ✅: [`2b-bots.md`](./2b-bots.md). Câu hỏi: *"Nền tảng phục vụ nhiều phòng khám, mỗi nơi 1 quy trình. Cho họ tuỳ biến bằng cách nào mà không phá nhau, không phá nền tảng?"*

## Mục lục

- [Phần 1 — Bài toán](#phần-1--bài-toán)
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

**Kịch bản cố định:** chuỗi An Bình muốn khi bệnh nhân điền xong phiếu tiếp nhận thì tự tạo hồ sơ và nhắn Zalo xác nhận. Phòng khám Mắt Sáng (cùng nền tảng) muốn mỗi kết quả đo nhãn áp > 21 mmHg thì tạo việc cho bác sĩ.

---

## Phần 1 — Bài toán

| # | Đặc tính | Vì sao |
|---|---|---|
| X1 | Tenant tự triển khai, không chờ nhà cung cấp nền tảng | Nhịp nghiệp vụ của phòng khám khác nhịp release của nền tảng |
| X2 | Cách ly: logic của A lỗi không ảnh hưởng B và nền tảng | Nhiều tenant chung server |
| X3 | Bị giới hạn bởi phân quyền | Logic của A không đọc được dữ liệu B |
| X4 | Truy cập được dữ liệu FHIR và dịch vụ ngoài (SMS, thanh toán) | Hầu hết quy trình là "đọc FHIR + gọi đối tác + ghi FHIR" |
| X5 | Chặn được dữ liệu sai **trước** khi lưu (khi cần) | An toàn: không cho lưu đơn thuốc chống chỉ định |
| X6 | Quan sát được: log, audit, đo thời gian | Quy trình y tế phải truy vết được |

## Phần 2 — 6 ứng viên

### 2.1 Fork và sửa mã nguồn 💭 (thực tế phổ biến với OpenEMR)

Mỗi khách hàng 1 bản sửa riêng của nền tảng.
- X4, X5 tuyệt đối (làm gì cũng được).
- **X1, X2 vỡ**: mỗi lần nâng cấp nền tảng phải merge lại. Không chung server được.

### 2.2 Module/plugin trong tiến trình — OpenEMR ✅

`interface/modules/custom_modules/` (VD `oe-module-comlink-telehealth`, `oe-module-dorn`), nghe sự kiện `src/Events/*`.
- X4, X5 tốt (cùng tiến trình, gọi thẳng service).
- X2 vỡ: module lỗi làm sập ứng dụng. X3: module toàn quyền.
- X1: cài module là việc của người vận hành server.

### 2.3 Interceptor trong tiến trình — HAPI ✅

`@Hook(Pointcut.STORAGE_PRECOMMIT_RESOURCE_CREATED)` (`hapi-fhir-base/.../interceptor/api/Pointcut.java:1847`), khoảng 110 điểm móc.
- X5 tốt nhất: móc được vào mọi giai đoạn (trước lưu, trong transaction, sau lưu, trước trả response).
- X1, X2, X3 vỡ: JAR Java, restart server, toàn quyền.

### 2.4 Webhook ra ngoài, khách tự host logic 💭

Nền tảng chỉ bắn sự kiện. Khách tự dựng server nhận (giống Subscription rest-hook tới URL ngoài).
- X1, X2, X3 tốt (logic ở hạ tầng của khách).
- X5 vỡ: webhook chạy **sau** khi lưu. X6 phân tán. Khách phải tự vận hành server, xác thực, lưu bí mật.

### 2.5 Hàm serverless do nền tảng host — Medplum Bot trên Lambda ✅

`Bot.runtimeVersion = awslambda` (`bots/execute.ts:47-54`). Khách gửi mã qua API `$deploy`. Nền tảng chạy trong Lambda riêng, cấp `accessToken` theo membership của Bot, cấp `secrets`.
- X1: `$deploy` qua API.
- X2: Lambda tách tiến trình.
- X3: Bot có AccessPolicy.
- X4: `MedplumClient` + `secrets`.
- X6: AuditEvent + metric (`:71-79`).
- X5: có qua **pre-commit** (`fhir/precommit.ts:128-142`), nhưng phải bật 2 khoá.

### 2.6 Sandbox trong tiến trình — Medplum `vmcontext` ✅

`vm.runInNewContext` (`bots/vmcontext.ts:158`), timeout 10 giây (`:48`).
- X1, X3, X4, X6 như 2.5.
- X2 **một phần**: Node `vm` không phải ranh giới bảo mật mạnh, chung bộ nhớ và CPU với server.

## Phần 3 — Bảng so sánh

| Ứng viên | X1 tự triển khai | X2 cách ly | X3 phân quyền | X4 FHIR + ngoài | X5 chặn trước lưu | X6 quan sát |
|---|---|---|---|---|---|---|
| 2.1 Fork | ✗ | ✗ | ✗ | ✅ | ✅ | ◐ |
| 2.2 Module (OpenEMR) | ✗ | ✗ | ✗ | ✅ | ✅ | ◐ |
| 2.3 Interceptor (HAPI) | ✗ | ✗ | ✗ | ✅ | ✅ **mạnh nhất** | ◐ |
| 2.4 Webhook ngoài | ✅ | ✅ | ✅ | ◐ khách tự lo | ✗ | ✗ |
| 2.5 Bot Lambda (Medplum) | ✅ | ✅ | ✅ | ✅ | ◐ pre-commit | ✅ |
| 2.6 Bot vmcontext (Medplum) | ✅ | ◐ | ✅ | ✅ | ◐ | ✅ |

## Phần 4 — Khi nào đúng

| Ứng viên | Đúng khi |
|---|---|
| 2.1 | 1 tổ chức tự vận hành, có đội dev riêng, chấp nhận chi phí merge |
| 2.2 | EHR 1 tổ chức (OpenEMR), module do đối tác uy tín viết |
| 2.3 | Tổ chức tự vận hành FHIR server, cần can thiệp sâu (bảo mật, validate đặc thù) |
| 2.4 | Khách đã có hạ tầng và đội backend riêng |
| 2.5 | **SaaS nhiều tenant** cho khách tự tuỳ biến |
| 2.6 | Tự host Medplum, không có Lambda/K8s, Bot nhẹ và tin cậy được |

## Phần 5 — Lối tắt sai

**A — *"Cho khách viết SQL trigger, nhanh nhất."*** Trigger chạy trong DB chung, thấy mọi tenant (vỡ X3), lỗi là hỏng giao dịch ghi (vỡ X2). Và không hiểu FHIR ([1a](./1a-fhir-datastore.md): dữ liệu là JSON trong `content`).

**B — *"`vmcontext` là sandbox nên chạy code khách thoải mái."*** Module `vm` của Node **không** được thiết kế làm ranh giới bảo mật cho mã không tin cậy. Bản hosted dùng Lambda là có lý do.

**C — *"Mọi kiểm tra đều đưa vào pre-commit cho chắc."*** Mỗi pre-commit là 1 lần chạy Bot **chặn** đường ghi. 5 pre-commit × 300 ms = 1,5 giây cho mỗi lần lưu, và đối tác ngoài sập thì không lưu được gì.

**D (leo quá xa) — *"MVP cần hệ thống workflow engine riêng (BPMN, Temporal) cho quy trình."*** Với vài quy trình đầu, Subscription + Bot + `Task` (module 3d) là đủ. Workflow engine riêng chỉ đáng khi có quy trình nhiều bước, chạy nhiều ngày, cần hiển thị trạng thái.

## Phần 6 — Medplum tại từng ngã rẽ

| Ngã rẽ | Medplum | ✅ |
|---|---|---|
| Tránh 2.2/2.3 (trong tiến trình, toàn quyền) | Bot là **resource**, có membership + AccessPolicy | `bots/execute.ts:41-45` |
| Tránh 2.4 (khách tự host) | Nền tảng host, cấp token + bí mật | `bots/utils.ts:344-356` |
| Giữ X5 | Pre-commit là Subscription đặc biệt, gọi đồng bộ | `fhir/precommit.ts:27-31`, `repo.ts:1009` |
| Nhiều runtime | Lambda / vmcontext / Fission chọn theo `runtimeVersion` | `bots/execute.ts:47-60` |
| Tắt theo project | Feature `bots` | `bots/utils.ts:160-164` |

## Phần 7 — Đánh đổi bằng số

💭 Ước tính độ trễ thêm vào **đường ghi** cho mỗi lần lưu `MedicationRequest`:

| Cách kiểm tra dị ứng thuốc | Trễ thêm khi lưu | Khi Bot lỗi |
|---|---|---|
| Pre-commit Bot trên Lambda (khởi động lạnh) | khoảng vài trăm ms đến hơn 1 giây | **Không lưu được** |
| Pre-commit Bot `vmcontext` | khoảng chục đến vài trăm ms | Không lưu được |
| Subscription → Bot tạo `Task` cảnh báo sau khi lưu | 0 | Đơn vẫn lưu, cảnh báo đến muộn hoặc mất |
| CDS Hooks khi bác sĩ **đang** kê (trước khi bấm lưu) | 0 lúc lưu, gợi ý trên màn hình | Không có gợi ý, vẫn lưu được |

🧠 Với kiểm tra **an toàn người bệnh**, CDS Hooks (cảnh báo khi kê) + pre-commit (chặn cuối) là cặp hợp lý: bác sĩ thấy cảnh báo sớm, dữ liệu sai vẫn không lọt. Với thứ **không nguy hiểm** (nhắn tin, đồng bộ), dùng Subscription.

## Phần 8 — Đối chứng nội bộ

Có 1 kiểm tra **không** giao cho Bot mà viết cứng trong server: không cho xoá resource mà `ProjectMembership` đang trỏ tới (`fhir/precommit.ts:48-58`, `:177+`).

🧠 Lý do: đây là **toàn vẹn của chính nền tảng** (xoá `AccessPolicy` đang dùng thì người dùng không đăng nhập được, xem lỗi trong `accesspolicy.ts:273-283`). Nghiệp vụ của khách thì giao cho Bot. Bất biến của nền tảng thì code cứng. Đó là ranh giới đúng.

## Phần 9 — Phản biện

**1 — CTO khách hàng:** *"Chúng tôi không muốn mã nghiệp vụ nằm trên hạ tầng của nhà cung cấp."*

<details><summary>Gợi ý</summary>

Hợp lệ (rủi ro phụ thuộc, dữ liệu đi qua). Có 2 lựa chọn: (a) tự host Medplum, Bot chạy trên Fission/K8s của khách; (b) dùng Subscription rest-hook tới server của khách (2.4) cho mọi thứ không cần chặn trước lưu. Cái mất đi ở (b) là X5 và việc nền tảng lo bí mật, token.
</details>

**2 — Đội bảo mật:** *"Bot `publicWebhook` là cửa vào không xác thực."*

<details><summary>Gợi ý</summary>

Đúng. Giảm rủi ro bằng: Bot đó có AccessPolicy hẹp nhất có thể, xác minh chữ ký của bên gọi trong mã Bot, ghi AuditEvent `always`. Không bật `publicWebhook` cho Bot có quyền đọc dữ liệu lâm sàng rộng.
</details>

**3 — Dev:** *"Debug Bot trên Lambda khó hơn debug code trong server."*

<details><summary>Gợi ý</summary>

Đúng. Bù lại bằng: viết Bot như hàm thuần nhận `BotEvent` để unit test với `MockClient` (các Bot mẫu đều có `.test.ts` đi kèm, VD `examples/medplum-demo-bots/src/hello-patient.test.ts`), và bật `auditEventTrigger = always` khi mới triển khai.
</details>

## Phần 10 — Checklist

- [ ] Logic này có phải **chặn** dữ liệu sai trước khi lưu không? Có thì pre-commit (và cân nhắc thêm CDS Hooks). Không thì Subscription.
- [ ] Bot có AccessPolicy tối thiểu?
- [ ] Bí mật nằm trong `Project.secret`, không trong mã?
- [ ] Bot idempotent (an toàn khi bị gọi lại)?
- [ ] Runtime phù hợp: Lambda cho mã khách hàng, `vmcontext` chỉ cho mã tin cậy và nhẹ?
- [ ] Có test với dữ liệu mẫu (bà Lan) trước khi bật Subscription thật?
