# 3i — Consent: đồng ý điều trị, chia sẻ dữ liệu, chữ ký

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"Bà Lan ký đồng ý điều trị và đồng ý chia sẻ dữ liệu cho bảo hiểm. Điều đó được lưu thế nào, và hệ thống có tự chặn truy cập khi bà rút lại đồng ý không?"*

## Mục lục

- [1. Consent là bản ghi 1 thỏa thuận](#1-consent-là-bản-ghi-1-thỏa-thuận)
- [2. Đồng ý lại: thay thế, không sửa](#2-đồng-ý-lại-thay-thế-không-sửa)
- [3. Chữ ký tay số hoá](#3-chữ-ký-tay-số-hoá)
- [4. Consent không tự chặn truy cập](#4-consent-không-tự-chặn-truy-cập)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Consent là bản ghi 1 thỏa thuận

📘 `packages/docs/docs/consent/index.mdx`: **1 `Consent` = 1 thỏa thuận**. Bà Lan đồng ý điều trị nhưng từ chối thoả thuận thanh toán thì phải có 2 Consent với 2 trạng thái riêng.

| Trường | Hệ mã | Ví dụ 💭 |
|---|---|---|
| `scope` | consentscope: `patient-privacy`, `treatment`, `research`, `adr` | `treatment` |
| `category` | LOINC `59284-0` (Privacy Consent Document) hoặc v3-ActCode (`nopp`, `pay`, `med`) | `med` |
| `policyRule` / `policy` | Chính sách áp dụng: `hipaa-npp`, `hipaa-self-pay`, hoặc URL hợp đồng | URL điều khoản phòng khám An Bình |
| `patient`, `dateTime`, `performer` | Ai, khi nào, ai ghi nhận | |
| `status` | `active`, `rejected`, `inactive`, `entered-in-error` | `active` |

📘 Docs nhấn mạnh: tổ hợp mã nào nghĩa là "đồng ý điều trị" là **chính sách của tổ chức**, phải thống nhất trước khi viết logic, vì đổi sau phải chuyển đổi dữ liệu cũ.

✅ Bot tiếp nhận mẫu tạo **7 `Consent`** từ 1 phiếu (`examples/medplum-patient-intake-demo/src/bots/core/intake-form.ts:246-318`, [3a](./3a-dinh-danh-tiep-nhan.md#bot-tiếp-nhận-mẫu-làm-gì)).

## 2. Đồng ý lại: thay thế, không sửa

📘 Mỗi lần đồng ý lại (chính sách đổi, hằng năm) là **Consent mới**, Consent cũ chuyển `inactive`. Từ chối vẫn lưu (`rejected`).

🧠 Lý do (docs nói rõ): câu hỏi kiểm toán là *"vào ngày X, bệnh nhân đã đồng ý điều gì?"*. Sửa tại chỗ thì phải đào `_history` ([1f](./1f-history-audit-deletion.md)), thay thế thì search theo `date` là ra. Docs cũng chỉ ra công cụ History/Blame của Medplum App cho phép xem ai đổi trường nào.

## 3. Chữ ký tay số hoá

📘 Ký trong form ([3g](./3g-questionnaire.md)) → lưu vào `QuestionnaireResponse` dưới extension `questionnaireresponse-signature`, kiểu `Signature`:
- `type` = `ProofOfOrigin`
- `who` = người đăng nhập
- `when` = thời điểm ký
- **`data` = ảnh chữ ký PNG base64**

🧠 So với chữ ký trên `Provenance` của app Provider ([3c](./3c-charting-encounter.md#4-ký-và-khoá-hồ-sơ)), ở đây `data` **có** dữ liệu, nhưng là **ảnh nét ký**, không phải chữ ký số mật mã. Nó chứng minh "ai đã ký" (qua phiên đăng nhập), không chứng minh "nội dung chưa bị sửa". Với yêu cầu chữ ký số ở Việt Nam ([3c mục 4](./3c-charting-encounter.md#4-ký-và-khoá-hồ-sơ)) cần tích hợp thêm dịch vụ ký số.

## 4. Consent không tự chặn truy cập

✅ Grep `packages/server/src` (trừ test): `Consent` chỉ xuất hiện trong các file migration tạo bảng, cột. `fhir/accesspolicy.ts` không nhắc tới Consent.

🧠 Nghĩa là: bà Lan rút đồng ý chia sẻ dữ liệu cho bảo hiểm (`Consent.status = inactive`) thì tài khoản bảo hiểm **vẫn đọc được** nếu AccessPolicy của nó cho phép. Medplum tách 2 khái niệm:

| | Consent | AccessPolicy ([1c](./1c-access-control.md)) |
|---|---|---|
| Là gì | **Bằng chứng** bệnh nhân đã đồng ý/từ chối | **Cơ chế** chặn/cho đọc ghi |
| Ai đọc | Con người, kiểm toán, Bot | Server, ở mọi request |

Muốn Consent có hiệu lực kỹ thuật: Subscription trên `Consent` → Bot cập nhật AccessPolicy/ProjectMembership tương ứng (VD gỡ bệnh nhân khỏi phạm vi của tài khoản bảo hiểm). Đây lại là 1 điểm "headless": Medplum cho đủ mảnh, ứng dụng phải ráp.

📘 FHIR R4 có định nghĩa `Consent.provision` (cho phép/từ chối theo loại dữ liệu, theo người nhận, theo thời gian) đủ để máy đọc. Nhưng đánh giá nó tự động (consent enforcement) là phần nhiều server FHIR không làm sẵn.

## Sai lầm thường gặp

1. **Gộp nhiều thoả thuận vào 1 Consent.** Không ghi được "đồng ý A, từ chối B".
2. **Sửa Consent tại chỗ khi đồng ý lại.** Mất câu trả lời "ngày X đồng ý gì" ở dạng search được.
3. **Tưởng `status: inactive` sẽ chặn truy cập.** Không (mục 4).
4. **Coi ảnh chữ ký là chữ ký số.** Không chống sửa nội dung.
5. **Đổi tổ hợp mã sau khi đã có dữ liệu.** Phải chuyển đổi toàn bộ Consent cũ.
