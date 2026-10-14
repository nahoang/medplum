# 5b — Self-host vs hosted, sharding, migration CSDL

> **Loại file:** R (ngắn). **Commit:** Medplum `16cb130`.
>
> Trả lời câu: *"Chạy Medplum ở đâu, tốn gì để vận hành, và 1 nền tảng FHIR nhiều tenant nâng cấp schema, chia CSDL thế nào mà không dừng dịch vụ?"*

## Mục lục

- [1. 4 cách triển khai](#1-4-cách-triển-khai)
- [2. Medplum tự nói gì về self-host](#2-medplum-tự-nói-gì-về-self-host)
- [3. Migration 2 pha: trước và sau khi deploy](#3-migration-2-pha-trước-và-sau-khi-deploy)
- [4. Sharding: đang trong giai đoạn chuẩn bị](#4-sharding-đang-trong-giai-đoạn-chuẩn-bị)
- [5. Ý nghĩa kinh doanh](#5-ý-nghĩa-kinh-doanh)
- [Sai lầm thường gặp](#sai-lầm-thường-gặp)

Nhãn: ✅ code `file:line` · 📘 docs · 💭 ví dụ · 🧠 suy luận.

---

## 1. 4 cách triển khai

| Cách | Thư mục | Thành phần |
|---|---|---|
| **AWS (CDK)** | `packages/cdk/src/` | ✅ `backend.ts`: Aurora Postgres (`rds.DatabaseCluster`, có RDS Proxy), ElastiCache Redis, ECS **Fargate**, WAF, CloudTrail, GuardDuty (`cloudtrail.ts`, `guardduty.ts`). Frontend qua S3 + CloudFront (`frontend.ts`, `oai.ts`) |
| **Kubernetes (Helm)** | `charts/` | Chart cho server, cần Postgres + Redis bên ngoài |
| **Azure / GCP (Terraform)** | `terraform/azure`, `terraform/gcp` | Hạ tầng tương đương trên 2 cloud còn lại |
| **Máy đơn / Docker** | `docker-compose*.yml`, docs `install-on-ubuntu.md` | Phát triển, dùng thử |

📘 Docs `packages/docs/docs/self-hosting/` có hướng dẫn cho AWS, Azure, GCP, DigitalOcean, Kubernetes, Ubuntu, cùng các bài về kiểm thử tải, khôi phục thảm hoạ, rollback, hiệu năng index GIN.

🧠 Bộ thành phần tối thiểu luôn là: **Postgres** (dữ liệu, [1a](./1a-fhir-datastore.md)) + **Redis** (cache, hàng đợi BullMQ cho subscription/bot [2a](./2a-subscriptions.md), rate limit [5a](./5a-gioi-han-do-luong.md)) + **server Node** + **lưu file** (S3 hoặc tương đương, cho `Binary`) + **chạy Bot** (Lambda, hoặc `vmcontext` trong server, [2b](./2b-bots.md)).

## 2. Medplum tự nói gì về self-host

📘 `packages/docs/docs/self-hosting/considerations.md` ("Should you Self-Host?") nói thẳng chi phí:
- **Nâng cấp liên tục**: Medplum phát hành thường xuyên, mỗi bản là việc của đội self-host.
- **Sở hữu hạ tầng**: chọn cỡ, nâng cấp Postgres và Redis là việc liên tục.
- **Trực 24/7**: hệ thống y tế dùng cả đêm và cuối tuần.

Và nói khi nào **nên** self-host:
- Ràng buộc pháp lý/hợp đồng về **chủ quyền dữ liệu** (VD hệ thống liên bang Mỹ).
- Triển khai **biên**, kết nối kém, dữ liệu nhỏ.
- Học tập, không chạy thật.

Lợi ích hosted được nêu: vận hành được quản lý, **kế thừa tuân thủ** (chứng nhận ONC, Trust Center), log streaming (gói Premium/Enterprise).

🧠 Đây là trang **bán hàng** viết trong docs kỹ thuật, và đáng đọc như 1 tài liệu chiến lược: open-core kiếm tiền bằng cách làm cho **tự vận hành tốn hơn thuê**. Ở Việt Nam, yêu cầu dữ liệu y tế lưu trong nước khiến lý do "chủ quyền dữ liệu" gần như luôn đúng → self-host hoặc tìm đối tác hosting trong nước là bắt buộc, kéo theo toàn bộ chi phí ở trên.

## 3. Migration 2 pha: trước và sau khi deploy

✅ `packages/server/src/migrations/`:

| Pha | Thư mục | Số lượng | Chạy khi | Ví dụ |
|---|---|---|---|---|
| **Pre-deploy** (schema) | `schema/` | 122 file (`v1` … `v122`) | **Trước** khi server mới nhận traffic, đồng bộ | Thêm cột, thêm bảng |
| **Post-deploy** (data) | `data/` | 52 file + manifest | **Sau** khi server mới chạy, bất đồng bộ qua `AsyncJob`/BullMQ | Reindex, tạo index `CONCURRENTLY`, điền dữ liệu cột mới |

✅ Post-deploy có **manifest** `data/data-version-manifest.json`: mỗi migration ghi `serverVersion` (có từ bản nào) và **`requiredBefore`** (phải chạy xong trước khi nâng lên bản nào). Server từ chối khởi động nếu bỏ qua migration bắt buộc (docblock `migration-versions.ts:5-27`, trừ chế độ `FIRST_BOOT`).

✅ Kiểu post-deploy: `reindex`, `custom`, `dynamic`, chạy theo từng **shard** (`data/types.ts:4-11`).

✅ Migration schema được **sinh tự động** bằng cách so định nghĩa FHIR với CSDL hiện tại (`migrate-main.ts:47-67` ghi file `schema/vN.ts` và `data/vN.ts`). 76 file migration dùng `CONCURRENTLY` (tạo index không khoá bảng).

🧠 Vì sao 2 pha: bảng `Observation` của 1 nền tảng lớn có hàng trăm triệu dòng. Thêm cột rỗng thì nhanh (pre-deploy), nhưng **điền** cột đó (reindex từ `content`, [CLAUDE.md phát hiện 3](../CLAUDE.md#phát-hiện-quan-trọng-đến-nay)) mất hàng giờ. Làm đồng bộ thì dừng dịch vụ hàng giờ. Tách ra thì server mới chạy ngay, dữ liệu được điền dần. Đây là kỹ thuật "expand–migrate–contract" quen thuộc của hệ thống không được phép dừng.

## 4. Sharding: đang trong giai đoạn chuẩn bị

✅ `packages/server/src/fhir/sharding.ts`:
- Shard `global` là mặc định khi chưa bật sharding (`:12-15`).
- 2 ID tạm `placeholder` và `todo` để viết sẵn logic định tuyến trước khi sharding thật hoạt động. Cả 2 **gộp về `global`** (`:17-31`, `normalizeShardId`). Docblock gọi đây là "future-proofing phase".
- Đơn vị chia: **Project**. Project mới đặt vào shard có `isDefaultShard` (`getDefaultShardId`, `:92`).
- **Các loại resource luôn ở shard global** (`globalShardResourceTypes`, `:141-153`): `Project`, `ProjectMembership`, `User`, `ClientApplication`, `JsonWebKey`, `Login`, `DomainConfiguration`… vì phải đọc được **trước khi biết project** (luồng đăng nhập).
- 1 giao dịch chỉ được chạm 1 shard. Giao dịch trộn resource global và resource của project → lỗi định tuyến (`shardRoutingError`, `:277`). Thấy rõ trong `withTransaction` ([3b](./3b-dat-lich.md#vì-sao-serializable-đủ-chống-trùng): `repo.ts`, "Resolving up front is what binds the transaction to a shard").

🧠 Sharding theo **tenant** (project) là lựa chọn tự nhiên khi mọi dòng đã có `projectId` ([1c](./1c-access-control.md)): dữ liệu 1 phòng khám không bao giờ cần join với phòng khám khác. Cái khó nằm ở **dữ liệu dùng chung** (người dùng đăng nhập nhiều project, terminology dùng chung [1e](./1e-terminology.md)), nên mới có danh sách global.

## 5. Ý nghĩa kinh doanh

🧠
- **Migration 2 pha + manifest `requiredBefore`** là cơ chế cho phép **phát hành thường xuyên** mà khách self-host không bị gãy, nhưng buộc họ **không được bỏ qua phiên bản** quá xa. Đây là ràng buộc vận hành cần ghi vào hợp đồng nếu tư vấn khách self-host.
- **Sharding theo project** là tiền đề cho **khách hàng lớn có CSDL riêng** (cô lập hiệu năng, cô lập dữ liệu) trên cùng bản hosted. Đó là sản phẩm "Enterprise" điển hình.
- Hạ tầng tham chiếu là AWS (CDK là cách "chính chủ"). Azure/GCP qua Terraform. Ở Việt Nam cần tính thêm chi phí chuyển sang hạ tầng trong nước.

## Sai lầm thường gặp

1. **Nhảy nhiều phiên bản 1 lần khi self-host.** Vi phạm `requiredBefore`, server không khởi động.
2. **Coi post-deploy migration đã xong khi server đã chạy.** Reindex có thể còn chạy hàng giờ; search trên cột mới chưa đủ dữ liệu.
3. **Chạy Bot bằng `vmcontext` trên production self-host** vì không có Lambda. Bot lỗi làm chậm server chung ([2b](./2b-bots.md)).
4. **Ước tính chi phí self-host chỉ bằng tiền máy chủ.** Chi phí lớn nhất là người trực và nâng cấp (docs nói rõ).
