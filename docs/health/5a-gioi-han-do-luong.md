# 5a — Giới hạn & đo lường sử dụng: rate limit, FHIR quota, resource cap

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"1 nền tảng nhiều tenant chặn 1 khách hàng dùng quá mức thế nào, và cách đo 'mức dùng' nói gì về mô hình kinh doanh?"*

## Mục lục

- [1. 3 lớp giới hạn](#1-3-lớp-giới-hạn)
- [2. Rate limit theo IP](#2-rate-limit-theo-ip)
- [3. FHIR quota: tính điểm theo chi phí](#3-fhir-quota-tính-điểm-theo-chi-phí)
- [4. Resource cap: giới hạn số resource](#4-resource-cap-giới-hạn-số-resource)
- [5. Ý nghĩa kinh doanh](#5-ý-nghĩa-kinh-doanh)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 docs · 💭 ví dụ · 🧠 suy luận. Đường dẫn từ `packages/server/src/`.

---

## 1. 3 lớp giới hạn

| Lớp | Đo cái gì | Theo ai | Cửa sổ | Bật khi | ✅ |
|---|---|---|---|---|---|
| **Rate limit** | Số **request** | **IP** | 1 phút | `rateLimitsEnabled` (server) | `ratelimit.ts` |
| **FHIR quota** | **Điểm** theo loại thao tác | **Membership** + **Project** | 1 phút | `rateLimitsEnabled` và project không tắt `enableFhirQuota` | `fhir/fhirquota.ts` |
| **Resource cap** | Tổng số **resource** đang có | **Project** | (đếm lại mỗi ngày) | Project có setting `resourceCap` và `enableResourceCap = true` | `fhir/resource-cap.ts` |

Tất cả dùng thư viện `rate-limiter-flexible` trên **Redis** riêng cho rate limit (`getRateLimitRedis`), nên giới hạn đúng trên **cả cụm** server, không phải từng máy (`ratelimit.ts:18-22` ghi lại lý do chuyển từ bộ nhớ trong sang Redis).

## 2. Rate limit theo IP

✅ `ratelimit.ts:33-62`, 4 nhóm:

| Nhóm | Đường dẫn | Mặc định code (/phút/IP) |
|---|---|---|
| `login` | `/auth/login`, `/auth/newuser`, `/auth/newproject` | 5 |
| `mfa` | `/auth/mfa/verify`, `/auth/mfa/login-enroll` | 10 |
| `auth` | `/auth/*`, `/oauth2/*` (trừ `/auth/me`) | 1.200 |
| `default` | còn lại | 60.000 |

- Khoá = IP (+ tên nhóm nếu không phải `default`) (`:128`).
- Vượt → `429` + header reset.
- Ghi đè được ở cấu hình server và `Project.systemSetting` (`serverConfigKey`, `systemSettingName`).

📘 Docs `packages/docs/docs/rate-limits.md` công bố bảng **Free tier / Paid tier**: default 6.000 vs 60.000 req/IP/phút, auth 160/phút. 🧠 Số docs khác số mặc định trong code (auth 1.200), nghĩa là bản hosted cấu hình riêng. Đây là ví dụ "docs mô tả dịch vụ, code mô tả phần mềm".

🧠 Mục đích chính của nhóm `login`/`mfa`: chống **dò mật khẩu**, là yêu cầu bảo mật (HIPAA Security Rule), không phải chuyện chi phí.

## 3. FHIR quota: tính điểm theo chi phí

✅ `fhir/fhirquota.ts:22-27`:

| Thao tác | Điểm |
|---|---|
| Read | 1 |
| History | 10 |
| Search | 20 |
| Write (create/update/delete/patch) | 100 |

- Hạn mức **mỗi người dùng** (`getUserFhirQuota`, `:36-46`): `UserConfiguration.option[fhirQuota]` → `Project.systemSetting[userFhirQuota]` → mặc định server **50.000 điểm/phút** (`config/utils.ts:61`).
- Hạn mức **cả project** (`getProjectFhirQuota`, `:51-54`): `totalFhirQuota`, mặc định = **10 ×** hạn mức người dùng.
- Mỗi request trừ điểm cả 2 khoá (user, project) (`:154-192`). Vượt → `429` kèm JSON chẩn đoán (`block`, `:230-250`).
- Batch được tính theo **từng thao tác bên trong** (📘 docs: 5 read + 1 search = 25 điểm).
- Project tắt được bằng `enableFhirQuota = false` (`:112`).
- Admin xem mức dùng qua `Project/$rate-limits` (chỉ project admin, `fhir/operations/project-rate-limits.ts:56-58`) và trang `/admin/rate-limits` của Medplum App.

💭 Bot đồng bộ HL7 v2 ([2c](./2c-agent-hl7v2.md)) nhận 300 tin/phút, mỗi tin 1 search + 3 write = 320 điểm → **96.000 điểm/phút**, vượt hạn mức mặc định 50.000. Phải cấp `UserConfiguration` riêng cho membership của Bot (docs hướng dẫn đúng trường hợp này).

🧠 Trọng số 1 : 20 : 100 phản ánh **chi phí thật trên Postgres**: ghi = 1 dòng resource + 1 dòng history + nhiều dòng lookup + index ([1a](./1a-fhir-datastore.md)), search = query có join ([1b](./1b-search.md)). Đây là đơn vị đo "tải" chứ không phải "request".

## 4. Resource cap: giới hạn số resource

✅ `fhir/resource-cap.ts`:
- Bật khi project có `resourceCap` (số) **và** `enableResourceCap = true` (`:44`; `context.ts:269-278`).
- Lần đầu đếm `COUNT(*)` mọi loại resource của project (trừ loại hệ thống/admin), lưu vào Redis, TTL 1 ngày (`:47-65`).
- Mỗi lần tạo resource trừ 1. Hết → lỗi `businessRule('resource-cap')` (`:78-97`). Tạo thất bại thì hoàn lại điểm (`:103-105`).

🧠 Đây là giới hạn **dung lượng lưu trữ**, phù hợp gói miễn phí / dùng thử (chặn 1 project thử nghiệm nạp hàng triệu bản ghi Synthea).

## 5. Ý nghĩa kinh doanh

🧠 3 lớp giới hạn là **3 đơn vị có thể định giá** của 1 nền tảng dữ liệu:

| Đơn vị | Giới hạn | Tương ứng chi phí hạ tầng |
|---|---|---|
| Request | Rate limit | Băng thông, CPU tầng web |
| Điểm FHIR (đọc/tìm/ghi có trọng số) | FHIR quota | CPU + IO Postgres |
| Số resource | Resource cap | Dung lượng Postgres + index |

Nối sang [5d](./5d-tong-hop-chien-luoc.md): bảng giá công khai của Medplum (nếu có) nên được đọc theo 3 đơn vị này. Khách hàng có tải **ghi** nhiều (đồng bộ HL7, import dữ liệu) tốn kém hơn hẳn khách hàng chủ yếu **đọc**, dù cùng số request.

## Sai lầm thường gặp

1. **Bot tích hợp dùng hạn mức mặc định.** Bị `429` lúc cao điểm, mất tin nhắn nếu không có hàng đợi bền ([2c](./2c-agent-hl7v2.md)).
2. **Tưởng rate limit theo người dùng.** Theo **IP**: 50 nhân viên sau 1 NAT của bệnh viện chia chung 1 hạn mức.
3. **Tính chi phí theo request.** 1 batch 100 write = 1 request nhưng 10.000 điểm.
4. **Tắt `enableFhirQuota` để "cho nhanh".** Mất bảo vệ: 1 vòng lặp lỗi trong Bot làm nghẽn cả project.
