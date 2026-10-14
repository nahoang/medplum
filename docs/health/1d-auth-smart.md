# 1d — Xác thực, OAuth2 và SMART on FHIR

> **Loại file:** R. **Commit:** Medplum `16cb130` · OpenEMR `4c1eb8d`.
>
> Trả lời câu: *"Ai đang gọi API? Họ chứng minh danh tính thế nào? Ứng dụng bên thứ 3 được uỷ quyền tới đâu?"* Module [1c](./1c-access-control.md) trả lời "được làm gì". Module này trả lời câu đứng trước nó: "là ai".
>
> Đường dẫn Medplum tính từ `packages/server/src/`.

## Mục lục

- [1. Vì sao xác thực trong y tế khác các ngành khác](#1-vì-sao-xác-thực-trong-y-tế-khác-các-ngành-khác)
- [2. 4 khái niệm: User, ProjectMembership, Login, ClientApplication](#2-4-khái-niệm-user-projectmembership-login-clientapplication)
- [3. Các luồng đăng nhập (grant type)](#3-các-luồng-đăng-nhập-grant-type)
- [4. SMART on FHIR: chuẩn để app bên thứ 3 cắm vào](#4-smart-on-fhir-chuẩn-để-app-bên-thứ-3-cắm-vào)
- [5. Mỗi request được nhận diện thế nào](#5-mỗi-request-được-nhận-diện-thế-nào)
- [6. Login là nhật ký phiên, không chỉ là token](#6-login-là-nhật-ký-phiên-không-chỉ-là-token)
- [7. Đối chiếu OpenEMR](#7-đối-chiếu-openemr)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)
- [Tự kiểm tra](#tự-kiểm-tra)

Nhãn: ✅ code `file:line` · 📘 chuẩn/docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. Vì sao xác thực trong y tế khác các ngành khác

🧠 Có 3 đặc thù:

1. **Nhiều loại "người dùng" không phải nhân viên.** Bệnh nhân (cổng bệnh nhân), người nhà được uỷ quyền, app của bên thứ 3 (bảo hiểm, app theo dõi sức khoẻ), máy móc (thiết bị, hệ thống lab gửi kết quả tự động).
2. **Chuẩn hoá bắt buộc.** 📘 Ở Mỹ, quy định về API cho bệnh nhân truy cập dữ liệu (ONC Cures Act) dẫn tới yêu cầu hỗ trợ **SMART on FHIR**, tức OAuth2 + quy ước scope và ngữ cảnh bệnh nhân. 1 app viết đúng chuẩn chạy được trên mọi EHR hỗ trợ SMART.
3. **"Đang xem trong ngữ cảnh nào" là thông tin bảo mật.** App mở từ màn hình hồ sơ bà Lan chỉ nên thấy bà Lan, không phải toàn bộ bệnh nhân bác sĩ có quyền xem.

## 2. 4 khái niệm: User, ProjectMembership, Login, ClientApplication

📘 Định nghĩa trong `packages/definitions/src/fhir/r4/profiles-medplum.json`.

| Khái niệm | Là gì | Trường đáng chú ý |
|---|---|---|
| `User` | Tài khoản đăng nhập (1 email) | `email`, `passwordHash`, `mfaRequired`, `mfaEnrolled`, `mfaMethod` (`totp` hoặc email), `project` (nếu user chỉ tồn tại trong 1 project) |
| `ProjectMembership` | User X là ai trong project P | `profile` (Practitioner/Patient/…), `accessPolicy`, `access[]`, `admin` (xem [1c](./1c-access-control.md#3-tầng-2--projectmembership-ai-vai-trò-gì-trong-project-nào)) |
| `Login` | **1 phiên đăng nhập** | `user`, `membership`, `client`, `scope`, `authMethod`, `code`, `codeChallenge`, `refreshSecret`, `granted`, `revoked`, `mfaVerified`, `launch`, `remoteAddress`, `userAgent`, `expiresAt` |
| `ClientApplication` | 1 ứng dụng (web, mobile, backend, app bên thứ 3) | `secret`, `retiringSecret`, `redirectUris`, `jwksUri`, `launchUri`, `pkceOptional`, `accessTokenLifetime`, `refreshTokenLifetime`, `allowedOrigin`, `grantType`, `defaultScope`, `certificateTrustStore` |

🧠 Tách `User` khỏi `ProjectMembership` cho phép 1 người là bác sĩ ở project A và bệnh nhân ở project B, cùng 1 email. Tách `ClientApplication` thành resource riêng cho phép **app cũng có membership và AccessPolicy** như người. Bot, tích hợp lab, app bảo hiểm đều được phân quyền theo đúng cơ chế ở 1c.

## 3. Các luồng đăng nhập (grant type)

✅ `oauth/token.ts:86-110` dispatch theo `grant_type`:

| Grant type | Dùng cho | Ví dụ nghiệp vụ 💭 |
|---|---|---|
| `authorization_code` (+ PKCE) | Web/mobile app có người dùng | Bác sĩ đăng nhập app khám bệnh. Bệnh nhân đăng nhập cổng bệnh nhân |
| `client_credentials` | Máy với máy, không có người | Hệ thống lab gửi kết quả. Job đồng bộ dữ liệu. Bot |
| `refresh_token` | Gia hạn phiên | App mobile giữ đăng nhập 2 tuần |
| `urn:ietf:params:oauth:grant-type:token-exchange` | Đổi token của IdP bên ngoài lấy token Medplum (`token.ts:502-503`) | Bệnh viện đã có SSO (Azure AD, Okta). Nhân viên không cần mật khẩu Medplum riêng |
| Pre-authorized code | Mã dùng 1 lần do hệ thống cấp trước (`auth/preauthorize.ts:19-47`, có TTL tối đa) | Gửi link cho bệnh nhân mở hồ sơ mà không cần tạo mật khẩu |

**Các đường đăng nhập phía người dùng** ✅ `auth/routes.ts:28-47`:
- `/auth/login`: mật khẩu
- `/auth/google`: Google
- `/auth/external`: IdP ngoài
- `/auth/mfa/*`: MFA (TOTP hoặc mã gửi email, `auth/mfa.ts:89-302`)
- `/auth/newpatient`: bệnh nhân tự đăng ký
- `/auth/newproject`: tạo project mới
- `/auth/scope`: chọn scope
- `/auth/revoke`: thu hồi phiên

**Endpoint OAuth chuẩn** ✅ `oauth/routes.ts:14-22`: `/oauth2/authorize`, `/token`, `/userinfo`, `/logout`, `/introspect`, `/register` (đăng ký client động).

**Thời hạn mặc định** ✅ `oauth/keys.ts:107-108`: access token **1 giờ**, refresh token **2 tuần**. Ghi đè được theo từng `ClientApplication` (`oauth/token.ts:1006`).

**Khai báo khả năng** ✅ `wellknown.ts:34-49` (`/.well-known/openid-configuration`):
- `code_challenge_methods_supported: ['S256', 'plain']`
- `token_endpoint_auth_methods_supported`: `client_secret_basic`, `client_secret_post`, `private_key_jwt`

## 4. SMART on FHIR: chuẩn để app bên thứ 3 cắm vào

📘 SMART App Launch quy định:
- **Cách app được mở**:
  - *EHR launch*: mở từ trong EHR, nhận sẵn ngữ cảnh bệnh nhân.
  - *Standalone launch*: app tự mở, người dùng chọn bệnh nhân.
- **Scope** dạng `patient/Observation.rs`, `user/Claim.rs`, `system/*.cruds`:
  - `patient/` = chỉ dữ liệu của bệnh nhân trong ngữ cảnh.
  - `user/` = theo quyền của người dùng.
  - `system/` = cho máy với máy.
- **Ngữ cảnh** trả kèm token: `patient`, `encounter`, `fhirContext`.

✅ Medplum khai báo tại `/.well-known/smart-configuration` (`fhir/smart.ts:58`, `wellknown.ts:88`). Danh sách capability (`fhir/smart.ts:87-104`):
- `launch-ehr`, `launch-standalone`
- `client-public`
- `context-ehr-patient`, `context-ehr-encounter`, `context-standalone-patient`, `context-banner`, `context-style`
- `permission-v1`, `permission-v2`, `permission-offline`, `permission-patient`, `permission-user`
- `sso-openid-connect`

Ngữ cảnh lưu trong resource `SmartAppLaunch` (`patient`, `encounter`, `fhirContext`) và gắn vào `Login.launch`. Operation `/ClientApplication/:id/$smart-launch` (`fhir/routes.ts:424`) tạo phiên launch.

**Scope ∩ AccessPolicy** ✅ `fhir/smart.ts:196-211`. Scope `patient/` mà không có ngữ cảnh bệnh nhân thì **từ chối** (`:204-207`). Còn lại, chính sách cuối = giao của AccessPolicy người dùng và scope (`intersectSmartScopes`).

🧠 Hệ quả nghiệp vụ: bác sĩ mở 1 app tính liều insulin từ hồ sơ bà Lan với scope `patient/Observation.rs`. App chỉ đọc được Observation **của bà Lan**, dù bác sĩ có quyền xem 4.000 bệnh nhân.

## 5. Mỗi request được nhận diện thế nào

✅ `oauth/middleware.ts`:
- `Authorization: Bearer <JWT>` → tìm `Login` từ token (`:70`, `getLoginForAccessToken`).
- `Authorization: Basic <clientId:secret>` → client credentials trực tiếp (`:74-75`, `getLoginForBasicAuth`).

Kết quả là `AuthState` gồm `login`, `membership`, `project`, `profile`, có thể kèm `onBehalfOf` và `onBehalfOfMembership` (`:31-32`).

**Thay mặt người khác** ✅ header `x-medplum-on-behalf-of` (`oauth/utils.ts:1182-1191`). 1 client (VD backend của app) gọi API **thay mặt** 1 user cụ thể. Quyền tính theo membership của user đó (`fhir/accesspolicy.ts`, `getRepoForLogin`). `meta.onBehalfOf` được ghi vào resource ([0a](./0a-fhir-primer.md#phần-riêng-của-medplum-meta-mở-rộng-và-resource-tự-định-nghĩa)).

🧠 Ví dụ: trợ lý y khoa nhập liệu thay bác sĩ. `meta.author` là trợ lý, `meta.onBehalfOf` là bác sĩ. Đây đúng là cách hồ sơ giấy vẫn làm ("BS Minh chỉ định, ĐD Hoa ghi").

## 6. Login là nhật ký phiên, không chỉ là token

✅ Mỗi lần đăng nhập tạo 1 resource `Login`. Các trường cho thấy nó đóng 3 vai trò:

| Vai trò | Trường |
|---|---|
| Trạng thái luồng OAuth | `code`, `codeChallenge`, `codeChallengeMethod`, `nonce`, `granted` |
| Quản lý phiên | `refreshSecret`, `previousRefreshSecret`, `refreshSecretRotatedAt` (xoay vòng refresh token), `expiresAt`, `revoked` |
| Bằng chứng kiểm toán | `authMethod`, `authTime`, `mfaVerified`, `remoteAddress`, `userAgent`, `basedOn` |

🧠 Vì `Login` là resource, "thu hồi mọi phiên của 1 nhân viên nghỉ việc" là 1 câu search và 1 loạt cập nhật `revoked=true`. Câu hỏi của thanh tra "người này đăng nhập từ đâu, có MFA không" cũng trả lời được bằng dữ liệu có sẵn.

## 7. Đối chiếu OpenEMR

✅ OpenEMR cũng hỗ trợ OAuth2 và SMART:
- Thư mục `oauth2/` (`authorize.php`)
- `src/RestControllers/SMART/`: `SMARTAuthorizationController.php`, `SMARTConfigurationController.php`, `ScopePermissionParser.php`, `PatientContextSearchController.php`, `ScopeConsentResolver.php`

| | Medplum | OpenEMR |
|---|---|---|
| Đăng nhập nhân viên | OAuth2/OIDC, cùng 1 hệ thống cho mọi client | Màn hình đăng nhập PHP truyền thống (session cookie) cho app chính. OAuth2 cho API và app SMART |
| App bên thứ 3 | `ClientApplication` = resource, có membership + AccessPolicy | Client OAuth đăng ký riêng, quyền theo scope |
| SSO doanh nghiệp | Token exchange, IdP ngoài | Tuỳ cấu hình |
| Lý do có SMART | Thiết kế từ đầu theo hướng API | 📘 Đáp ứng chứng nhận ONC cho EHR |

🧠 Ở OpenEMR, SMART là **cửa phụ** cho app ngoài, còn giao diện chính vẫn chạy session PHP. Ở Medplum, **mọi** client (kể cả app admin của chính Medplum) đều đi qua OAuth. Đây là khác biệt "nền tảng API-first" và "ứng dụng có thêm API".

## Sai lầm thường gặp

1. **Dùng `client_credentials` cho app có người dùng.** Mọi thao tác ghi nhận là "app", không biết bác sĩ nào làm. Mất khả năng kiểm toán. Phải dùng `authorization_code`, hoặc `x-medplum-on-behalf-of` nếu backend gọi thay.
2. **Nhúng `secret` của ClientApplication vào app web/mobile.** App chạy trên thiết bị người dùng là *public client*: dùng PKCE, không dùng secret.
3. **Xin scope `user/*.cruds` cho app bên thứ 3 "cho chắc".** Phạm vi thực tế = giao với quyền của người dùng, nhưng vẫn vi phạm nguyên tắc tối thiểu và làm người dùng e ngại khi đồng ý.
4. **Không bật MFA cho tài khoản quản trị.** `User.mfaRequired` có sẵn. Tài khoản admin project có thể sửa `AccessPolicy` của mọi người ([1c mục 6](./1c-access-control.md#6-lúc-đăng-nhập-ghép-thành-1-chính-sách-cuối-cùng)).
5. **Để access token sống quá lâu.** Mặc định 1 giờ là hợp lý. Kéo dài lên nhiều ngày làm việc thu hồi phiên khi mất thiết bị kém hiệu quả.

## Tự kiểm tra

1. Hệ thống lab gửi kết quả xét nghiệm tự động vào Medplum. Dùng grant type nào? `ClientApplication` này nên có AccessPolicy gì?
2. Bà Lan dùng app theo dõi đường huyết của bên thứ 3, kết nối qua SMART standalone launch với scope `patient/Observation.cru`. App có ghi được Observation cho bệnh nhân khác không? Vì sao?
3. Một nhân viên nghỉ việc. Liệt kê 3 thao tác trên `User`, `ProjectMembership`, `Login` để cắt hoàn toàn truy cập.
