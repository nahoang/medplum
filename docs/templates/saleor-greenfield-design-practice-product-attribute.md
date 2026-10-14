# Thiết kế từ đầu (greenfield) vs. Saleor thật — luyện tập trên Product & Attribute

> **Khác hướng đọc của mọi file khác trong `knowledge/` cho tới giờ** — các file trước (kể cả [`ecommerce-setup-1b-product-attribute.md`](./ecommerce-setup-1b-product-attribute.md), nguồn "Saleor thật" dùng xuyên suốt file này) đọc code **đã hoàn thiện** rồi rút bài học. File này đảo ngược hướng: giả định **chưa có 1 dòng code nào**, mô phỏng đúng trình tự 1 dự án greenfield thật sẽ đi qua — discovery (BA) → database design khái niệm (cầu nối BA-dev) → thiết kế đầu tiên (dev) — RỒI mới mở 1b ra đối chiếu. Mục đích không phải học lại EAV (đã học đủ ở 1b) mà học **con đường quyết định** dẫn tới nó, và quan trọng không kém: học cách nhận ra khi nào KHÔNG nên đi hết con đường đó.
>
> Tái dùng nguyên phương pháp discovery (Stakeholder map → Bảng business rule) đã luyện ở [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md), áp cho đúng 1 tình huống mới: **không có hệ thống sẵn để gap-analysis** — chỉ có 1 buổi kickoff và 1 trang giấy trắng.
>
> Muốn xem V1/V2/V3 ở Phần 4 dưới dạng bảng so sánh trực diện với 2 kỹ thuật chưa xuất hiện ở đây (STI thuần, CTI), cùng ma trận điểm mạnh/yếu lý giải vì sao đích đến là **hybrid** chứ không thuần 1 kỹ thuật — xem [`saleor-design-technique-comparison-product-attribute.md`](./saleor-design-technique-comparison-product-attribute.md).

---

## Mục lục

> File dài (8 Phần, khá nhiều mục con) — mục lục này để nhảy nhanh, không cần cuộn. Link theo chuẩn anchor phổ biến (GitHub/VS Code); nếu trình đọc markdown bạn dùng slugify khác đi (VD 1 số bản Obsidian), có thể cần cuộn tay thay vì nhảy chính xác — tên mục vẫn đúng để tự tìm bằng Ctrl+F.

- [Quy tắc nhãn nguồn trong file này](#quy-tắc-nhãn-nguồn-trong-file-này)
- [Phần 1 — Bối cảnh giả định](#phần-1-bối-cảnh-giả-định-buổi-kickoff-greenfield)
- [Phần 2 — BA view](#phần-2-ba-view-từ-1-câu-nói-tới-yêu-cầu-chức-năng)
  - [Bước 1 — Stakeholder map](#bước-1-stakeholder-map)
  - [Bước 2 — Bảng business rule](#bước-2-bảng-business-rule)
  - [Bước 3 — 7 câu hỏi discovery](#bước-3-7-câu-hỏi-discovery-quyết-định-hướng-thiết-kế)
  - [Artifact đầu ra Phần 2](#artifact-đầu-ra-phần-2-giả-định-thực-tế-thường-gặp)
  - [Sai lầm thường gặp khi làm BA view](#sai-lầm-thường-gặp-khi-làm-ba-view-minh-hoạ-ngay-trên-đúng-bối-cảnh-phần-1-2)
- [Phần 3 — Database design từ góc nhìn BA](#phần-3-database-design-từ-góc-nhìn-ba-xác-định-thực-thể-và-quan-hệ-trước-khi-chọn-công-nghệ)
  - [Bước 1 — Rút thực thể](#bước-1-rút-thực-thể-từ-đúng-ngôn-ngữ-đã-dùng-ở-phần-2)
  - [Kiểm tra lại — thực thể có "thật" không?](#kiểm-tra-lại-thực-thể-này-có-thật-không)
  - [Thuộc tính sơ bộ mỗi thực thể](#thuộc-tính-sơ-bộ-mỗi-thực-thể-bắt-đầu-lộ-ranh-giới-của-ai)
  - [Bước 2 — Ma trận quan hệ](#bước-2-ma-trận-quan-hệ-áp-đúng-hỏi-2-chiều-bằng-ngôn-ngữ-nghiệp-vụ-chưa-đụng-tới-fkcode)
  - [Sơ đồ khái niệm](#sơ-đồ-khái-niệm-chỉ-thực-thể-loại-quan-hệ-chưa-có-cộtkiểu-dữ-liệu-nào)
  - [Mỗi thực thể phục vụ đúng ai](#mỗi-thực-thể-phục-vụ-đúng-ai-nối-ngược-lại-stakeholder-map-phần-2)
  - [Cầu nối sang Phần 4](#cầu-nối-sang-phần-4-ánh-xạ-sang-tên-gọi-kỹ-thuật)
  - [Sai lầm thường gặp khi làm database design](#sai-lầm-thường-gặp-khi-làm-database-design-khác-hẳn-2-near-miss-đã-có-ở-kiểm-tra-lại-thực-thể)
- [Phần 4 — Developer view](#phần-4-developer-view-claude-tự-thiết-kế-khi-chỉ-có-artifact-phần-2-và-phần-3)
  - [Từ mô hình khái niệm sang Django ORM](#từ-mô-hình-khái-niệm-phần-3-sang-django-orm-dịch-từng-dòng)
  - [V1 — cột cứng trên Product](#v1-cột-cứng-trên-product)
  - [V2 — 1 cột JSONB](#v2-phản-xạ-tiếp-theo-1-cột-jsonb)
  - [Điểm mù của V2](#điểm-mù-của-v2-chỉ-lộ-ra-nếu-hỏi-tiếp-q4-q7)
  - [V3 — EAV cổ điển](#v3-nếu-bị-đẩy-tiếp-eav-cổ-điển-2-tầng-1-cột-value)
  - [Đào sâu — thiết kế đúng quan hệ N-N](#đào-sâu-thiết-kế-đúng-quan-hệ-n-n-kỹ-năng-dùng-lại-được-không-riêng-saleor)
  - [Thứ tự xây dựng thực tế](#thứ-tự-xây-dựng-thực-tế-3-tầng-không-ra-đời-cùng-lúc)
  - [3 lối tắt "nghe hợp lý nhưng sai"](#3-lối-tắt-nghe-hợp-lý-nhưng-sai-luyện-phân-biệt-không-chỉ-đi-theo-1-đường-thẳng)
  - [Data flow](#data-flow-mô-phỏng-dữ-liệu-chảy-qua-đúng-thứ-tự-implement)
  - [Sai lầm thường gặp khi viết code cho schema đã đúng](#sai-lầm-thường-gặp-khi-viết-code-cho-schema-đã-đúng-khác-hẳn-3-lối-tắt)
- [Phần 5 — Đối chiếu thực tế](#phần-5-đối-chiếu-thực-tế-2-hệ-thống-thật-2-lựa-chọn-khác-hẳn-nhau)
- [Phần 6 — Bảng so sánh tổng hợp 5 hướng](#phần-6-bảng-so-sánh-tổng-hợp-5-hướng)
- [Phần 7 — Đánh giá](#phần-7-đánh-giá-vì-sao-có-khoảng-cách-và-khi-nào-không-nên-nhảy-thẳng-lên-saleor-thật)
  - [Luyện phản biện](#luyện-phản-biện-bảo-vệ-quyết-định-trước-3-kiểu-phản-đối-thường-gặp)
- [Phần 8 — Checklist tái sử dụng cho module khác](#phần-8-checklist-tái-sử-dụng-cho-module-khác)
- [Tự luyện tiếp theo](#tự-luyện-tiếp-theo)
- [Liên hệ / Nguồn](#liên-hệ-nguồn)

---

## Quy tắc nhãn nguồn trong file này

Phần lớn nội dung Phần 2-4 là **tự mô phỏng** (không có dự án/khách hàng thật đứng sau) — đọc nhầm 1 dòng 💭 thành ✅ trong file này nghĩa là tưởng nhầm "Claude tự đoán" thành "sự thật đã kiểm chứng":

| Nhãn | Nghĩa | Dùng ở đâu trong file |
|---|---|---|
| ✅ grounded | Trích từ code Saleor/WooCommerce thật, đã có nghiên cứu + `file:line` xác nhận sẵn ở file khác trong repo này ([1b](./ecommerce-setup-1b-product-attribute.md) cho Saleor, [solutions/ecommerce-domain-solutions.md](../solutions/ecommerce-domain-solutions.md) + [woocommerce-ecommerce-domain-knowledge.md](./woocommerce-ecommerce-domain-knowledge.md) cho WooCommerce) | Phần 5, Phần 6 (cột "thật") |
| 💭 mô phỏng | Claude tự đóng vai — suy luận từ pattern phổ biến trong ngành (đã thấy qua huấn luyện), KHÔNG phải quan sát từ 1 dự án/khách hàng thật, không có `file:line` vì chưa có code nào tồn tại | Phần 1, Phần 3, Phần 4 (mô hình khái niệm + toàn bộ V1/V2/V3) |
| 🧠 suy luận | Lập luận phân tích (vì sao có khoảng cách, khi nào nên/không nên) — là lý lẽ, không phải sự kiện trích dẫn được | Phần 7 |

---

## Phần 1 — Bối cảnh giả định: buổi kickoff greenfield 💭

> **Founder nói trong buổi kickoff đầu tiên với đội kỹ thuật:** *"Tụi mình vừa gọi vốn xong, đang xây lại toàn bộ hệ thống bán hàng từ đầu — trước giờ dùng Shopify nhưng phí % doanh thu ăn vào biên lợi nhuận quá nhiều khi scale. Giai đoạn đầu tụi mình chỉ bán quần áo, nhưng chắc chắn 6 tháng nữa mở thêm ngành phụ kiện, và nếu gọi vốn vòng sau ổn thì có thể mở cả đồ điện tử. Phần 'sản phẩm có nhiều biến thể/thuộc tính' này là lõi của cả hệ thống, nên team tự thiết kế, không đi thuê platform có sẵn nữa."*

Khác các tình huống đã luyện ở [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md) (khách đã CÓ Saleor, hỏi gap so với hệ thống sẵn có), tình huống này **không có hệ thống nào để đối chiếu** — chỉ có 1 câu nói và 1 trang giấy trắng. Đây đúng là bối cảnh 1 dev/BA gặp thật khi bắt đầu 1 dự án greenfield.

---

## Phần 2 — BA view: từ 1 câu nói tới yêu cầu chức năng

### Bước 1 — Stakeholder map

(áp đúng 5 câu hỏi đã luyện ở file phương pháp — ai nói, ai thao tác thật, ai nhận kết quả, có bên thứ 3 không, ai phát hiện nếu sai)

| Vai trò | Quan tâm gì | Vì sao dễ bị bỏ sót |
|---|---|---|
| Founder (người nói câu trên) | Tốc độ ra mắt, không phụ thuộc phí % của Shopify, mở ngành hàng mới nhanh | Là người duy nhất được nghe trực tiếp, nhưng KHÔNG phải người nhập sản phẩm hàng ngày |
| Nhân viên vận hành catalog | Người thao tác thật — nhập sản phẩm mới, gán thuộc tính, sửa tên hiển thị | Câu nói không hề nhắc tới — "team tự thiết kế" ngụ ý đây là việc của dev, nhưng người DÙNG hệ thống hàng ngày là vận hành, không phải dev |
| Khách hàng cuối (storefront) | Lọc/tìm sản phẩm theo Size, Color, khoảng giá... | Founder nói về "quản lý sản phẩm" nhưng không nhắc trải nghiệm lọc/tìm — dễ bị thiết kế chỉ tối ưu cho màn hình admin, quên storefront |
| Đội dev (bạn) | Phải tự quyết định độ phức tạp schema — không có ai review kiến trúc catalog trước | Founder giao toàn quyền ("team tự thiết kế") nghe như tự do, nhưng thực ra là rủi ro: không có gap matrix để đối chiếu như khi đã có sẵn 1 hệ thống tham khảo |

#### Cách tìm ra 4 vai trò trên — áp trực tiếp 5 câu hỏi của phương pháp gốc

Không phải tự nhiên liệt kê được 4 vai trò ở trên — đây là kết quả của việc áp đúng 5 câu hỏi đã luyện ở [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md), theo đúng thứ tự:

1. *Ai nói câu này?* → **Founder** — luôn là câu dễ nhất, ghi trước.
2. *Ai sẽ thao tác thật hàng ngày với thứ vừa yêu cầu?* → **Nhân viên vận hành catalog** — founder nói "team tự thiết kế" nhưng người NHẬP sản phẩm/gán thuộc tính mỗi ngày là vận hành, không phải founder hay dev.
3. *Ai nhận kết quả đầu ra?* → **Khách hàng cuối** — "kết quả" ở đây là trải nghiệm lọc/tìm sản phẩm trên storefront, thứ founder không hề nhắc tới trong câu nói gốc.
4. *Có bên thứ 3 nào ngoài công ty khách không?* → **Chưa trả lời được ở giai đoạn này** — chưa biết sẽ dùng nhà cung cấp hosting/CDN/thanh toán nào, vì founder mới nói "team tự thiết kế", chưa tới bước chọn hạ tầng. Đây là câu hỏi cần **hỏi lại** khi dự án đi xa hơn, không phải bỏ qua vì "không áp dụng" — 2 việc khác nhau.
5. *Ai là người phát hiện ra nếu làm sai?* → **Đội dev (bạn)** — nếu schema thiết kế sai (VD không mở rộng được ngành hàng mới), chính dev là người đầu tiên thấy khó khi cố thêm tính năng, trước khi vận hành hay khách hàng nhận ra.

**Điểm đáng chú ý:** câu hỏi 4 không ra được câu trả lời cụ thể — đây KHÔNG phải dấu hiệu làm sai bước, mà là tín hiệu thật: ở giai đoạn kickoff đầu tiên, 1 số câu hỏi trong 5 câu chưa có đủ thông tin để trả lời. Ghi nhận "chưa biết, cần hỏi lại" là hành vi đúng — khác hẳn việc bỏ trống không ghi gì, hoặc đoán bừa cho đủ 5 dòng.

### Bước 2 — Bảng business rule

| # | Loại | Rule | Câu hỏi cần hỏi thêm |
|---|---|---|---|
| 1 | Tường minh | Bán quần áo trước, phụ kiện sau 6 tháng, có thể điện tử sau nữa | Phụ kiện/điện tử có chia sẻ thuộc tính nào với quần áo không, hay hoàn toàn khác? |
| 2 | Tường minh | "Team tự thiết kế" — không thuê platform có sẵn | Có ràng buộc stack không (Django? Postgres?) hay hoàn toàn tự do? |
| 3 | Ẩn | Mở ngành hàng mới không nói rõ ai thực hiện — nhưng mốc "6 tháng nữa" (không phải "sau khi dev rảnh") ngụ ý kỳ vọng KHÔNG phụ thuộc lịch dev | Khi mở phụ kiện, ai là người thêm — dev sửa code hay vận hành tự thêm qua màn hình admin? |
| 4 | Ẩn | Founder không nhắc gì tới lọc/tìm sản phẩm trên storefront | Khách hàng có cần lọc theo Size/Color/khoảng giá không, hay chỉ xem theo danh mục? |
| 5 | Ẩn | Không nhắc tới việc đổi tên hiển thị giá trị thuộc tính (VD "Đỏ" → "Đỏ đô") | Giá trị thuộc tính có dùng lại nhiều lần không — đổi 1 chỗ có cần tự động cập nhật mọi sản phẩm đang dùng không? |
| 6 | Ẩn | Không nhắc gì tới đa ngôn ngữ dù vừa gọi vốn (dấu hiệu có thể mở rộng thị trường) | Có kế hoạch bán ngoài Việt Nam trong 1-2 năm tới không? |

#### Cách tìm ra rule ẩn — áp trực tiếp kỹ thuật "để làm được X phải giả định gì"

2 ví dụ áp kỹ thuật đã luyện ở file phương pháp — với mỗi rule tường minh, tự hỏi *"để làm được đúng cái này, hệ thống phải giả định điều gì mà founder chưa nói ra?"*:

- **Từ rule #1** ("bán quần áo trước, phụ kiện sau 6 tháng") → để mốc "6 tháng" này thực hiện được đúng như kỳ vọng (nhanh, không phải 1 dự án lớn), hệ thống phải giả định: **thêm ngành hàng mới KHÔNG cần sửa code**. Đây chính là rule ẩn #3 trong bảng trên — không phải tự nhiên viết ra, mà suy trực tiếp từ việc hỏi "để làm được rule #1, cái gì phải đúng trước?"
- **Từ Phần 1** ("sản phẩm có nhiều biến thể/thuộc tính") → để câu này có nghĩa thực tế (không chỉ là 1 cụm từ mô tả chung chung), hệ thống phải giả định: **giá trị thuộc tính (VD "M") là thứ TỒN TẠI ĐỘC LẬP, dùng lại được giữa nhiều biến thể** — chứ không phải mỗi biến thể tự gõ tay 1 chuỗi text riêng. Đây chính là rule ẩn #5.

**Dấu hiệu nhận biết đã tìm đúng rule ẩn:** rule ẩn luôn trả lời được câu "tại sao KHÔNG làm ngược lại cũng vẫn đúng nghĩa đen của lời founder" — VD founder không hề cấm "mỗi biến thể tự gõ tay Size", câu nói gốc vẫn đúng nghĩa đen dù làm vậy — nhưng làm vậy sẽ KHÔNG thoả được kỳ vọng ẩn (dùng lại được, đổi tên 1 chỗ). Rule ẩn chính là khoảng cách giữa "đúng nghĩa đen" và "đúng kỳ vọng thật".

**Hậu quả cụ thể nếu bỏ qua bước tìm rule ẩn:** nếu chỉ đọc rule tường minh rồi code luôn, thiết kế nhiều khả năng dừng ở mức thoả đúng nghĩa đen (VD field `size`/`color` cứng trên `Product`, đúng "sản phẩm có thuộc tính" theo nghĩa đen) nhưng vỡ trận ngay khi rule ẩn #3 phát huy tác dụng (mở ngành phụ kiện, cần sửa code) — đúng kịch bản V1 sẽ thấy ở Phần 4.

### Bước 3 — 7 câu hỏi discovery quyết định hướng thiết kế

Khác phương pháp gốc (kết thúc bằng Gap matrix, vì có sẵn hệ thống để đối chiếu), tình huống greenfield không có gì để đối chiếu — nên BA cần 1 bộ câu hỏi **dự đoán trước** độ phức tạp schema cần thiết. Mỗi câu ứng đúng 1 điểm rẽ nhánh thiết kế sẽ gặp lại ở Phần 3 (mức khái niệm) và Phần 4 (mức code):

| # | Câu hỏi discovery | Điểm rẽ nhánh thiết kế nó quyết định |
|---|---|---|
| Q1 | Ngành hàng mới có chia sẻ chung thuộc tính với ngành cũ không? | Có cần tách theo "khuôn mẫu sản phẩm" hay 1 bộ cột chung là đủ |
| Q2 | Ai thêm ngành hàng mới — dev hay vận hành? | Tập thuộc tính thuộc về **schema** (đổi cần deploy) hay thuộc về **dữ liệu** (đổi ngay) |
| Q3 | Khách có cần lọc/tìm theo thuộc tính không (Size=M, giá trong khoảng X-Y)? | Có cần index riêng theo đúng kiểu dữ liệu (số, ngày, boolean) không |
| Q4 | Giá trị thuộc tính có dùng lại nhiều lần không? Đổi tên hiển thị có cần khớp mọi nơi đang dùng không? | Có cần 1 bảng danh mục giá trị dùng chung (tách khỏi bảng gán) không |
| Q5 | Có kế hoạch đa ngôn ngữ không? | Có cần bảng dịch riêng cho từng giá trị không |
| Q6 | Thuộc tính có bao giờ trỏ sang 1 sản phẩm/trang khác trong hệ thống không (VD "sản phẩm đi kèm")? | Có cần kiểu giá trị REFERENCE (FK thật, CASCADE khi xoá) không |
| Q7 | Có thuộc tính nào áp dụng cho CẢ sản phẩm (dùng chung mọi biến thể) VÀ thuộc tính áp dụng RIÊNG từng biến thể không? | Có cần tách 2 đường gán riêng (product-level vs variant-level) không |

### Artifact đầu ra Phần 2 — giả định thực tế thường gặp

Buổi discovery đầu tiên hiếm khi hỏi hết cả 7 câu ngay. Giả định thực tế: **Q1-Q3 được hỏi** (founder trả lời được ngay — "không chia sẻ gì, quần áo cần Size/Color, điện tử cần RAM"; "vận hành tự thêm, không đợi dev"; "có, khách cần lọc Size/giá"), còn **Q4-Q7 được hỏi dần qua các buổi sau** — không phải "không bao giờ hỏi", chỉ là chưa hỏi hết ngay buổi đầu.

**Phần 3 và Phần 4 dùng 2 lát cắt thời gian khác nhau của đúng quá trình discovery này — cố tình, không phải mâu thuẫn:** Phần 3 giả định discovery đã **hoàn tất** (cả 7 câu đều có câu trả lời, qua nhiều buổi cộng lại) để xây mô hình khái niệm đầy đủ — đây là ĐÍCH ĐẾN. Phần 4 (V1/V2/V3) cố tình **phát lại từng chặng** của đúng quá trình đó, dừng lại đúng chỗ thông tin dừng ở TỪNG buổi (Q1-Q3 trước, Q4/Q6 sau, Q7 sau nữa) — để thấy hậu quả cụ thể nếu code sớm hơn thời điểm Phần 3 hoàn tất, thay vì đợi đủ thông tin rồi mới viết dòng đầu tiên.

### Sai lầm thường gặp khi làm BA view — minh hoạ ngay trên đúng bối cảnh Phần 1-2

Mỗi sai lầm dưới đây là 1 **phiên bản sai cụ thể** của đúng 3 bước đã làm ở trên — không phải nguyên tắc chung chung, mà cho thấy nếu làm ẩu thì bảng trông thế nào, và hậu quả cụ thể là gì (thường trỏ thẳng tới V1/V2 sai lầm ở Phần 4).

**Sai lầm 1 — chỉ ghi vai trò founder nhắc tới, bỏ sót vai trò ẩn:**

Phiên bản sai — Stakeholder map rút gọn còn 2 dòng, đúng bằng cách chỉ chép lại ai "có mặt" trong câu nói gốc:

| Vai trò | Quan tâm gì |
|---|---|
| Founder | Ra mắt nhanh, không phụ thuộc Shopify |
| Khách hàng | Mua được hàng |

So với bảng thật đã làm (4 dòng) — thiếu đúng 2 vai trò mà câu hỏi 2 và câu hỏi 5 của phương pháp gốc chủ động dò ra: Nhân viên vận hành catalog, Đội dev. Hậu quả cụ thể: thiết kế sẽ chỉ tối ưu cho "quản lý sản phẩm" ở mức khái niệm (đúng lời founder), quên hẳn màn hình NHẬP LIỆU mà vận hành cần dùng hàng ngày — tới lúc launch mới phát hiện thiếu, phải làm bổ sung. **Dấu hiệu nhận biết** (theo đúng [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md)): map chỉ có 1-2 dòng, toàn "khách hàng"/"founder" chung chung, không có vai trò nào KHÔNG được nhắc tên trong câu nói gốc.

**Sai lầm 2 — copy nguyên rule tường minh, không tìm rule ẩn:**

Phiên bản sai — bảng business rule chỉ 2 dòng, toàn tường minh, không có dòng "Ẩn" nào:

| # | Rule |
|---|---|
| 1 | Bán quần áo trước, phụ kiện sau 6 tháng |
| 2 | Team tự thiết kế |

So với bảng thật (6 dòng, 4/6 là "Ẩn") — thiếu đúng rule ẩn #3 (thêm ngành hàng không cần deploy), rule QUYẾT ĐỊNH độ phức tạp schema. Hậu quả cụ thể: nếu chỉ đọc rule 1 theo nghĩa đen ("bán quần áo, sau bán phụ kiện" — đơn thuần là lộ trình sản phẩm), rất dễ code y hệt V1 (Phần 4) — field cứng `size`/`color` trên `Product` — vì bản thân rule tường minh KHÔNG hề cấm điều đó, chỉ rule ẩn mới cấm. **Dấu hiệu nhận biết:** bảng chỉ có cột "Tường minh", không có dòng "Ẩn" nào.

**Sai lầm 3 — nhảy thẳng vào code trước khi làm xong Bước 1-2:**

Nếu vừa nghe xong câu nói của founder (Phần 1) mà mở editor viết Django model ngay — bỏ qua hẳn Stakeholder map và Bảng business rule — gần như chắc chắn sẽ tái hiện đúng V1 (Phần 4): chưa có Q1 (rút ra từ chính rule ẩn #3) để loại field cứng, chưa có bất kỳ tín hiệu nào để biết "quần áo" và "phụ kiện" sẽ KHÔNG chia sẻ thuộc tính. Đây chính là "Bước 0" đã nhấn mạnh ở [`docs/domain/tasks/CLAUDE.md`](../tasks/CLAUDE.md): đừng vội mở editor.

**Sai lầm 4 — coi discovery là 1 buổi họp duy nhất:**

2 biến thể của lỗi này, sai theo 2 hướng ngược nhau:
- Cố ép hỏi hết cả 7 câu Q1-Q7 ngay trong buổi kickoff đầu tiên — founder mới nói xong 1 câu ngắn, chưa sẵn sàng trả lời chi tiết về REFERENCE/đa ngôn ngữ/product-variant split — dễ khiến buổi họp quá tải, founder trả lời qua loa cho xong (dữ liệu discovery thu được sẽ SAI, không chỉ là "chưa thu được").
- Chỉ hỏi đúng 1 buổi (Q1-Q3), rồi coi như đã xong BA, không quay lại hỏi Q4-Q7 ở buổi sau — khác hẳn khung "2 lát cắt thời gian" vừa nêu ở trên: Q4-Q7 phải được hỏi dần, không phải bị bỏ quên vĩnh viễn.

**Dấu hiệu nhận biết:** sau buổi kickoff đầu tiên, không có kế hoạch cụ thể nào cho buổi discovery TIẾP THEO (ai hỏi, hỏi gì, khi nào) — đó là dấu hiệu Q4-Q7 sẽ bị quên thật, không phải "để dành hỏi sau" như dự định ban đầu.

**Sai lầm 5 — biến bảng business rule thành tài liệu nội bộ, không thực sự quay lại hỏi khách:**

Cột "Câu hỏi cần hỏi thêm" (Bước 2) dễ bị hiểu nhầm là XONG việc ngay khi VIẾT RA — trong khi giá trị thật của nó chỉ có khi thực sự đem hỏi lại founder. Viết ra 6 câu hỏi hay mà không ai hỏi founder câu nào trong số đó thì bảng business rule chỉ là tài liệu tự trấn an nội bộ, không thay đổi được lượng thông tin discovery thật đang có — khác hẳn 1 gap matrix (đã học ở phương pháp gốc) vốn CHỈ có giá trị sau khi đối chiếu với sự thật, bảng business rule cũng chỉ có giá trị sau khi đối chiếu lại với khách, không phải ngay khi viết xong.

---

## Phần 3 — Database design từ góc nhìn BA: xác định thực thể và quan hệ TRƯỚC khi chọn công nghệ 💭

> Bước hay bị nhảy cóc nhất trong thực tế: đi thẳng từ "yêu cầu nghiệp vụ" (Phần 2) sang "viết Django model" (Phần 4) — bỏ qua đúng bước đứng giữa, nơi 1 BA (hoặc data analyst) xác định **thực thể nào tồn tại và quan hệ giữa chúng ra sao**, hoàn toàn bằng ngôn ngữ nghiệp vụ, CHƯA cần biết sẽ dùng Django/Postgres hay công nghệ nào khác. Đây chính là 3 tầng kinh điển của thiết kế database: **mô hình khái niệm** (entity + quan hệ, ngôn ngữ nghiệp vụ — mục này) → **mô hình logic** (thêm thuộc tính/khoá) → **mô hình vật lý** (kiểu dữ liệu/index/constraint cụ thể — cả 2 tầng sau nằm ở Phần 4). Bỏ qua tầng khái niệm dễ khiến quyết định công nghệ (JSONB? EAV?) lấn át câu hỏi lẽ ra phải trả lời trước: **thực thể này có thật không, và quan hệ với thực thể kia là loại gì?**

### Bước 1 — Rút thực thể từ đúng ngôn ngữ đã dùng ở Phần 2

Không thêm khái niệm mới — chỉ gọi tên lại đúng danh từ đã xuất hiện trong bối cảnh (Phần 1) và bảng business rule (Phần 2), giữ nguyên ngôn ngữ nghiệp vụ (chưa dùng tên bảng/model nào):

| Thực thể (ngôn ngữ nghiệp vụ) | Xuất hiện ở đâu trong Phần 1-2 |
|---|---|
| Ngành hàng / khuôn sản phẩm | Rule #1 — "bán quần áo trước, phụ kiện sau 6 tháng" ngụ ý mỗi ngành 1 nhóm sản phẩm riêng |
| Sản phẩm | Hiển nhiên từ bối cảnh — "hệ thống quản lý sản phẩm" |
| Biến thể bán được | Phần 1 — "sản phẩm có nhiều biến thể/thuộc tính" |
| Thuộc tính | Phần 1 + Rule #4 (lọc theo Size/Color) |
| Giá trị thuộc tính | Rule #5 — "đổi tên hiển thị giá trị thuộc tính" ngụ ý giá trị là 1 thứ tồn tại độc lập, không chỉ là chuỗi gõ tay mỗi lần |

### Kiểm tra lại — thực thể này có "thật" không?

Không phải danh từ nào xuất hiện trong câu nói cũng là 1 thực thể riêng — cần 1 bộ tiêu chí, không chỉ cảm tính:

1. Có **định danh/vòng đời riêng** không? (tồn tại độc lập, tạo/sửa/xoá được riêng nó, không hoàn toàn phụ thuộc 1 thực thể khác)
2. Có **thuộc tính mô tả CHÍNH nó** không? (không phải chỉ là 1 giá trị/thuộc tính của thực thể khác)
3. Có **tham gia quan hệ độc lập** với ít nhất 1 thực thể khác không?

Cả 5 thực thể ở Bước 1 đều pass cả 3 câu — nhưng đáng xem qua 2 trường hợp KHÔNG pass, vì đây chính là bẫy dễ mắc nhất khi mới học rút thực thể:

- **"Size = M" (1 giá trị thuộc tính CỤ THỂ)** — KHÔNG phải 1 thực thể riêng khỏi "Giá trị thuộc tính". Nó không có vòng đời tách biệt (câu 1 fail) — nó là **1 dòng dữ liệu** (instance) của thực thể "Giá trị thuộc tính", không phải 1 loại thực thể mới.
- **"Ngành phụ kiện"** — KHÔNG phải thực thể riêng khỏi "Ngành hàng". Cùng lý do — nó là 1 instance cụ thể của "Ngành hàng", giống "Ngành quần áo" cũng là 1 instance khác của đúng thực thể đó.

**Vì sao 2 near-miss này đáng dừng lại xem kỹ:** nhầm lẫn "instance cụ thể" với "1 loại thực thể riêng" chính xác là gốc rễ dẫn tới V1 (Phần 4) — nếu coi "Ngành phụ kiện" là 1 thực thể khác hẳn "Ngành quần áo" (thay vì cùng 1 thực thể "Ngành hàng", chỉ khác dòng dữ liệu), phản xạ tự nhiên tiếp theo là tạo bảng/cột riêng cho từng ngành — đúng sai lầm đã thấy ở V1. Bài học: lỗi thiết kế database nghiêm trọng nhất thường không nằm ở tầng code, mà đã nằm sẵn ở tầng NHẬN THỨC KHÁI NIỆM — nhầm giữa "loại" (type) và "thực thể riêng lẻ" (instance).

### Thuộc tính sơ bộ mỗi thực thể — bắt đầu lộ ranh giới "của ai"

Chưa phải mô hình logic đầy đủ (đó là việc của Phần 4) — chỉ đủ để xác nhận ranh giới "thuộc tính này của thực thể nào", tránh nhét nhầm dữ liệu của thực thể B vào thực thể A:

| Thực thể | Thuộc tính sơ bộ (chỉ mô tả CHÍNH nó) | KHÔNG bao gồm — và tại sao |
|---|---|---|
| Ngành hàng | Tên ngành | KHÔNG có "Size/Color" — Ngành hàng chỉ QUY ĐỊNH dùng thuộc tính nào (quan hệ N-N sẽ xác định ở Bước 2 ngay dưới), không tự mang giá trị |
| Sản phẩm | Tên, mô tả, ảnh | KHÔNG có Size/Color trực tiếp — đi qua Biến thể + Giá trị thuộc tính, không phải cột của Sản phẩm |
| Biến thể | SKU, giá, tồn kho | KHÔNG có chữ "M" ngay trên nó — chỉ có LIÊN KẾT tới Giá trị thuộc tính, giữ đúng nguyên tắc dùng lại giá trị (rule ẩn #5 ở Phần 2) |
| Thuộc tính | Tên ("Size"), kiểu dữ liệu | KHÔNG có "M"/"L" ngay trên nó — đó thuộc về Giá trị thuộc tính, thực thể riêng |
| Giá trị thuộc tính | Tên hiển thị ("M"), mã nội bộ | KHÔNG có "đang dùng cho sản phẩm nào" — đó là QUAN HỆ (xác định ở Bước 2 ngay dưới), không phải thuộc tính của chính nó |

Quy luật rút ra: mỗi khi phân vân "thuộc tính X có nên nằm trên thực thể A không", tự hỏi *"X có đúng với MỌI dòng dữ liệu của A, hay chỉ đúng với 1 vài dòng/1 quan hệ cụ thể?"* — đúng với mọi dòng → thuộc tính thật của A; chỉ đúng 1 số trường hợp → dấu hiệu đó là dữ liệu của 1 thực thể/quan hệ khác, không phải thuộc tính của A. Đây chính xác là câu hỏi đã loại "Size" khỏi `Product` ngay từ V1 (Phần 4) — không phải MỌI sản phẩm đều có Size (sách thì không), nên Size không phải thuộc tính thật của Sản phẩm.

### Bước 2 — Ma trận quan hệ: áp đúng "hỏi 2 chiều" bằng ngôn ngữ nghiệp vụ, chưa đụng tới FK/code

Tái dùng nguyên phương pháp đã học ở [`saleor-database-relationship-cardinality.md`](./saleor-database-relationship-cardinality.md) — nhưng khác lúc dùng lại ở Phần 4 (nơi câu hỏi gắn với tên field/model cụ thể), ở đây câu hỏi **chỉ dùng danh từ nghiệp vụ**, vì mục đích là xác định quan hệ TRƯỚC khi biết sẽ code bằng gì:

| Thực thể A | Câu hỏi 2 chiều | Thực thể B | Kết luận | Rule nào gợi ý |
|---|---|---|---|---|
| Ngành hàng | "1 ngành hàng có nhiều sản phẩm không?" (có) / "1 sản phẩm thuộc nhiều ngành hàng cùng lúc không?" (không) | Sản phẩm | 1-N | Hiển nhiên từ bối cảnh |
| Sản phẩm | "1 sản phẩm có nhiều biến thể không?" (có — Đỏ-M, Đỏ-L) / "1 biến thể thuộc nhiều sản phẩm không?" (không) | Biến thể | 1-N | Phần 1 |
| Ngành hàng | "1 ngành hàng cần nhiều thuộc tính không?" (có — Áo thun cần Size+Color+Chất liệu) / "1 thuộc tính dùng cho nhiều ngành hàng không?" (có — Size còn dùng cho Giày sau này) | Thuộc tính | **N-N** | Q1 |
| Thuộc tính | "1 thuộc tính có nhiều giá trị hợp lệ không?" (có — Size có M/L/XL) / "1 giá trị thuộc về nhiều thuộc tính không?" (không — "M" chỉ thuộc Size) | Giá trị thuộc tính | 1-N | Rule #5 |
| Biến thể | "1 biến thể có nhiều giá trị thuộc tính không?" (có — vừa Size=M vừa Color=Đỏ) / "1 giá trị gắn cho nhiều biến thể không?" (có — nhiều biến thể khác đều Size=M) | Giá trị thuộc tính | **N-N** | Rule #5 + Q4 |
| Sản phẩm | "1 sản phẩm có giá trị thuộc tính dùng CHUNG cho mọi biến thể của nó không?" (có — Chất liệu áp dụng cả sản phẩm) / "1 giá trị đó gắn cho nhiều sản phẩm không?" (có — nhiều sản phẩm khác cũng Cotton) | Giá trị thuộc tính | **N-N** (nhánh riêng, không trùng dòng trên) | Q7 |

**3 dòng N-N tô đậm ở trên (Ngành hàng↔Thuộc tính, Biến thể↔Giá trị thuộc tính, Sản phẩm↔Giá trị thuộc tính) chính là 3 quan hệ N-N sẽ thấy lại — với đầy đủ cơ chế code — ở mục "Đào sâu thiết kế đúng quan hệ N-N" của Phần 4.** Điểm cần thấy ở TẦNG NÀY: cả 3 đều xác định được chỉ bằng câu hỏi nghiệp vụ, không cần biết trước sẽ dùng Django hay bất kỳ ORM nào — quyết định "N-N hay không" thuộc về BẢN CHẤT NGHIỆP VỤ, quyết định "code thế nào" (Phần 4) mới là kỹ thuật.

### Sơ đồ khái niệm — chỉ thực thể + loại quan hệ, chưa có cột/kiểu dữ liệu nào

| Thực thể A | Quan hệ | Thực thể B | Loại |
|---|---|---|---|
| Ngành hàng | có nhiều | Sản phẩm | 1-N |
| Sản phẩm | có nhiều | Biến thể | 1-N |
| Ngành hàng | quy định dùng | Thuộc tính | N-N |
| Thuộc tính | có nhiều | Giá trị thuộc tính | 1-N |
| Biến thể | được gán | Giá trị thuộc tính | N-N |
| Sản phẩm | được gán (dùng chung mọi biến thể) | Giá trị thuộc tính | N-N |

Cố tình CHƯA vẽ cột/kiểu dữ liệu/bảng trung gian nào ở tầng này — đó là việc của mô hình logic/vật lý (Phần 4). Tầng khái niệm chỉ cần trả lời đúng 2 câu: "thực thể nào tồn tại" và "quan hệ giữa chúng là loại gì" — sai ở tầng này (VD nhầm N-N thành 1-N) sẽ kéo sai toàn bộ quyết định code phía sau, trong khi sửa ở TẦNG NÀY chỉ cần sửa 1 dòng trong bảng trên, không cần sửa migration đã chạy.

### Mỗi thực thể phục vụ đúng ai — nối ngược lại Stakeholder map (Phần 2)

Thiết kế dữ liệu không phải bài tập trừu tượng — mỗi thực thể ở trên tồn tại để phục vụ đúng 1 (hoặc vài) vai trò đã xác định ở Stakeholder map:

| Thực thể | Stakeholder quan tâm nhất (Phần 2) | Vì sao |
|---|---|---|
| Ngành hàng | Founder, Nhân viên vận hành | Founder quyết định MỞ ngành nào; vận hành là người thao tác thật khi mở |
| Sản phẩm | Nhân viên vận hành, Khách hàng cuối | Vận hành nhập liệu; khách hàng nhìn thấy trên catalog |
| Biến thể | Nhân viên vận hành, Khách hàng cuối | Vận hành nhập SKU/giá/tồn kho; khách hàng chọn mua đúng phiên bản |
| Thuộc tính | Đội dev, Nhân viên vận hành | Dev thiết kế cơ chế; vận hành khai báo thuộc tính mới khi cần (rule ẩn #3) |
| Giá trị thuộc tính | Nhân viên vận hành, Khách hàng cuối | Vận hành nhập/sửa giá trị; khách hàng lọc theo giá trị (rule #4 ở Phần 2) |

**Điểm đáng chú ý:** không thực thể nào chỉ phục vụ đúng 1 vai trò — mọi thực thể đều chạm ít nhất 2 stakeholder khác nhau. Đây là bằng chứng ngược cho lý do Stakeholder map (Bước 1, Phần 2) quan trọng: bỏ qua bước đó (chỉ thiết kế theo đúng lời founder) rất dễ tối ưu data model cho ĐÚNG 1 vai trò (thường là vai trò của người nói, hoặc của dev) mà quên tối ưu cho vai trò còn lại — VD thiết kế đủ cho vận hành nhập liệu nhưng quên khách hàng cần LỌC theo giá trị thuộc tính (đòi hỏi index/quan hệ khác hẳn so với chỉ cần LƯU giá trị).

### Cầu nối sang Phần 4 — ánh xạ sang tên gọi kỹ thuật

| Thực thể (ngôn ngữ BA) | Tên kỹ thuật dùng từ Phần 4 trở đi |
|---|---|
| Ngành hàng / khuôn sản phẩm | `ProductType` |
| Sản phẩm | `Product` |
| Biến thể | `ProductVariant` |
| Thuộc tính | `Attribute` |
| Giá trị thuộc tính | `AttributeValue` |

3 dòng N-N ở mục "Ma trận quan hệ" chính là 3 điểm Phần 4 phải quyết định **cách hiện thực cụ thể** — đây là lúc câu hỏi đổi từ "quan hệ này LÀ GÌ" (đã trả lời xong ở Phần này) sang "quan hệ này NÊN CODE THẾ NÀO" (JSONB? bảng trung gian? cột nào kiểu gì?) — đúng ranh giới giữa BA và developer.

### Sai lầm thường gặp khi làm database design — khác hẳn 2 near-miss đã có ở "Kiểm tra lại thực thể"

Mục "Kiểm tra lại — thực thể này có thật không?" ở trên đã chỉ ra 2 trường hợp **thừa thực thể** ("Size=M", "Ngành phụ kiện" bị coi nhầm là loại riêng). 4 sai lầm dưới đây khác hẳn — về **quan hệ** thay vì thực thể, và về **thiếu/gộp nhầm** thay vì thừa. Mỗi sai lầm trỏ thẳng tới đúng 1 hậu quả cụ thể đã/sẽ thấy ở Phần 4, để thấy rõ: phần lớn "lỗi thiết kế" nhìn như lỗi code thực ra đã có mầm mống từ tầng khái niệm này.

**Sai lầm 1 — nhầm chiều quan hệ 1-N vì đổi nghĩa thực thể giữa chừng:**

Phiên bản sai — chỉ hỏi 1 chiều rồi kết luận vội: *"1 Sản phẩm có thể có nhiều Ngành hàng không? Có thể, nếu vừa là hàng khuyến mãi vừa là áo thun"* → kết luận sai thành N-N. Lỗi nằm ở chỗ đã lặng lẽ ĐỔI NGHĨA "Ngành hàng" giữa chừng — Bước 1 định nghĩa nó là khuôn quyết định BỘ THUỘC TÍNH nào active (schema-defining), không phải nhãn marketing/khuyến mãi (1 khái niệm khác, chưa hề xuất hiện ở Phần 1). Nếu founder thật sự cần khái niệm "danh mục khuyến mãi", đó là **1 thực thể mới cần thêm**, không phải lý do sửa quan hệ Ngành hàng↔Sản phẩm đã đúng thành N-N.

Hậu quả cụ thể: kết luận sai N-N ở đây làm `Product` mất hẳn 1 `product_type_id` rõ ràng để biết BỘ THUỘC TÍNH nào active — sập luôn nền tảng cho toàn bộ cơ chế EAV sẽ xây ở Phần 4. **Dấu hiệu nhận biết:** kết luận cardinality mà không tự hỏi lại "2 chiều câu hỏi có đang dùng ĐÚNG 1 nghĩa duy nhất của thực thể không, hay đã lặng lẽ đổi nghĩa?"

**Sai lầm 2 — bỏ sót hẳn 1 thực thể vì tưởng chỉ là 1 thuộc tính:**

Phiên bản sai — bỏ qua việc tách "Giá trị thuộc tính" thành thực thể riêng ở Bước 1, nhét thẳng vào Biến thể:

| Thực thể | Thuộc tính sơ bộ |
|---|---|
| Biến thể | SKU, giá, tồn kho, **size (string)** |

Hậu quả cụ thể: đây CHÍNH XÁC là gốc rễ dẫn tới V1 (field cứng) và V2 (JSONB) ở Phần 4 — 2 thiết kế đó không phải "lỗi code", mà là hệ quả trực tiếp của việc chưa rút đúng thực thể ngay từ tầng này. Rút đúng "Giá trị thuộc tính" thành 1 dòng riêng (đã làm ở Bước 1) tự động loại V1/V2 từ trước khi viết dòng code đầu tiên. **Dấu hiệu nhận biết:** danh sách thực thể có ít hơn 5 dòng, hoặc "giá trị" xuất hiện dưới dạng tên 1 field (`size`, `color`) thay vì 1 dòng thực thể độc lập.

**Sai lầm 3 — kết luận cardinality theo cảm giác/tên gọi, bỏ qua hỏi thật chiều thứ 2:**

Phiên bản sai — thấy "Ngành hàng CÓ Thuộc tính" nghe xuôi tai như quan hệ cha-con đơn giản, kết luận luôn 1-N (mỗi Ngành hàng tự định nghĩa Attribute riêng của mình) mà không tự hỏi chiều 2: *"1 Thuộc tính có dùng cho nhiều Ngành hàng không?"* (có — Size dùng cả Áo lẫn Giày).

Hậu quả cụ thể: kết luận sai 1-N ở đây dẫn tới việc 2 Ngành hàng khác nhau cùng cần "Size" sẽ tạo ra 2 Attribute "Size" trùng lặp hoàn toàn thay vì dùng chung 1 — mất đúng lợi ích dedup mà rule ẩn #5 (Phần 2) đã đặt ra làm mục tiêu. **Dấu hiệu nhận biết:** Ma trận quan hệ (Bước 2) có dòng nào chỉ ghi kết quả 1 chiều câu hỏi, chiều còn lại bỏ trống hoặc suy luận ngầm không viết ra.

**Sai lầm 4 — gộp 2 quan hệ N-N độc lập thành 1 vì chung 1 đầu:**

Phiên bản sai — thấy cả "Biến thể↔Giá trị thuộc tính" và "Sản phẩm↔Giá trị thuộc tính" đều có chung 1 đầu (Giá trị thuộc tính), kết luận nhầm đây là 1 quan hệ duy nhất (viết gộp "Sản phẩm/Biến thể ↔ Giá trị thuộc tính") thay vì 2 dòng tách biệt như Sơ đồ khái niệm đã làm.

Hậu quả cụ thể: đây chính là gốc rễ CONCEPTUAL dẫn tới Lối tắt B ở Phần 4 (gộp `AssignedVariantAttributeValue` và `AssignedProductAttributeValue` vào 1 bảng, dùng cột `variant` nullable — lỗi `NULL ≠ NULL` phá `unique_together`). Lỗi đó không phải lỗi code thuần tuý — nó bắt nguồn từ việc coi 2 quan hệ N-N độc lập là 1 quan hệ duy nhất ngay từ tầng khái niệm này. **Dấu hiệu nhận biết:** Sơ đồ khái niệm có ít hơn 6 dòng, hoặc 2 dòng N-N liên quan Giá trị thuộc tính bị viết gộp chung 1 dòng.

---

## Phần 4 — Developer view: Claude tự thiết kế khi chỉ có artifact Phần 2 và Phần 3 💭

> Toàn bộ phần này là **mô phỏng**, không phải quan sát từ 1 dự án thật — viết ra phản xạ thiết kế **nhiều khả năng nhất** dựa trên pattern phổ biến trong ngành, ứng với đúng lượng thông tin đã có (Q1-Q3, và mô hình khái niệm đã chốt ở Phần 3), không phải "đáp án đúng" đã biết trước rồi diễn lại cho có kịch bản.

### Từ mô hình khái niệm (Phần 3) sang Django ORM — dịch từng dòng

Trước khi vào V1/V2/V3 (nơi độ phức tạp thay đổi theo lượng thông tin discovery CÓ TẠI TỪNG THỜI ĐIỂM — xem lại khung "2 lát cắt thời gian" ở cuối Phần 2), mục này làm việc dev thật sự làm ĐẦU TIÊN khi Phần 3 đã hoàn tất: dịch thẳng từng dòng trong "Sơ đồ khái niệm" sang Django ORM. Không có quyết định thiết kế mới nào ở đây — chỉ là ánh xạ cơ giới từ **loại quan hệ** (đã xác định ở Phần 3) sang **construct ORM tương ứng**.

#### Bảng dịch nhanh — loại quan hệ nào dùng construct ORM nào

| Loại quan hệ (Phần 3) | Construct Django ORM | Đặt FK ở bên nào |
|---|---|---|
| 1-N | `ForeignKey` | Đặt trên bên "N", trỏ về bên "1" |
| N-N, KHÔNG mang dữ liệu riêng của cặp | `ManyToManyField` (Django tự tạo bảng trung gian ẩn, không thấy được, không thêm cột được) | Đặt trên 1 trong 2 bên (tuỳ chiều hay query nhiều hơn) |
| N-N, CÓ mang dữ liệu riêng của cặp | `ManyToManyField(..., through=...)` hoặc model trung gian tường minh với 2 `ForeignKey` | Bảng trung gian độc lập, không đặt field trên bảng nào trong 2 bên |

#### Dịch 3 quan hệ 1-N — đơn giản, ít khi cần bàn thêm

3 quan hệ 1-N đã xác định ở Phần 3 (Ngành hàng→Sản phẩm, Sản phẩm→Biến thể, Thuộc tính→Giá trị thuộc tính) dịch thẳng thành `ForeignKey`, không có quyết định nào cần cân nhắc thêm — kết hợp luôn với "Thuộc tính sơ bộ mỗi thực thể" đã lập ở Phần 3 để có field:

```python
class ProductType(models.Model):         # Ngành hàng
    name = models.CharField(max_length=250)

class Product(models.Model):              # Sản phẩm — bên "N", FK trỏ về ProductType
    name = models.CharField(max_length=250)
    description = models.TextField(blank=True)
    product_type = models.ForeignKey(ProductType, on_delete=models.CASCADE)

class ProductVariant(models.Model):       # Biến thể — bên "N", FK trỏ về Product
    sku = models.CharField(max_length=100, unique=True)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)

class Attribute(models.Model):            # Thuộc tính
    slug = models.SlugField(unique=True)
    name = models.CharField(max_length=100)

class AttributeValue(models.Model):       # Giá trị thuộc tính — bên "N", FK trỏ về Attribute
    slug = models.SlugField()
    name = models.CharField(max_length=100)
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
```

**Vì sao nhóm này "ít khi cần bàn thêm":** khác 3 quan hệ N-N ngay dưới, 1-N không có quyết định kiểu "có cần bảng trung gian không, có cần field riêng của cặp không" — chỉ cần xác định đúng bên nào là "1", bên nào là "N" (đã làm ở Bước 2, Phần 3), rồi đặt `ForeignKey` đúng chiều. Rủi ro thật duy nhất: đặt FK NHẦM CHIỀU (VD lỡ đặt `ProductType` trỏ về `Product`) — hỏi lại đúng câu 2 chiều đã dùng ở Phần 3 sẽ tự lộ ra ngay: "1 Product có nhiều ProductType không?" (không) → FK không thể nằm ở `ProductType`.

#### Dịch 3 quan hệ N-N — đây mới là nơi cần quyết định thêm

3 quan hệ N-N đã xác định ở Phần 3 (Ngành hàng↔Thuộc tính, Biến thể↔Giá trị thuộc tính, Sản phẩm↔Giá trị thuộc tính) — mỗi quan hệ cần trả lời đúng câu hỏi ở bảng dịch nhanh: *"quan hệ này có mang dữ liệu riêng của cặp không?"* trước khi biết dùng `ManyToManyField` phẳng hay cần bảng trung gian tường minh:

| Quan hệ N-N (Phần 3) | Có dữ liệu riêng của cặp? | Construct ORM | Chi tiết |
|---|---|---|---|
| Ngành hàng ↔ Thuộc tính | Có — `variant_selection` (Attribute này có tạo ra Variant không, chỉ đúng cho 1 cặp cụ thể) | `through=` (bảng `AttributeVariant`) | Trace đầy đủ ở mục "Đào sâu — thiết kế đúng quan hệ N-N" ngay dưới |
| Biến thể ↔ Giá trị thuộc tính | Có — cần biết giá trị gán cho ĐÚNG Variant nào, qua đúng "khung" nào (`AssignedVariantAttribute`), không chỉ 1 tầng | Bảng trung gian tường minh, 2 tầng nối tiếp (không phải 1 `through=` đơn) | Chi tiết ở mục "N-N xếp chồng N-N" (Bước 4 của mục N-N) |
| Sản phẩm ↔ Giá trị thuộc tính | Không có field nghiệp vụ đặc thù (không có `variant_selection` tương đương) | `through=` (bảng `AssignedProductAttributeValue`) — vẫn tường minh dù đơn giản hơn | 🧠 suy luận: nhiều khả năng vẫn cần `sort_order` (cùng lý do đã học ở Bước 3, mục N-N — thứ tự thuộc tính hiển thị trên trang sản phẩm), theo đúng pattern nhất quán các bảng gán khác của Saleor; chưa tự đọc lại đủ code `product.py` để xác nhận 100% |

Cả 3 dòng trên được code đầy đủ ngay trong 2 mục tiếp theo — bảng này chỉ để thấy TRƯỚC khi đọc chi tiết: cả 3 quan hệ N-N của Phần 3 đều rơi vào nhánh "cần bảng trung gian tường minh", **không nhánh nào dùng được `ManyToManyField` phẳng** — đúng lý do sẽ giải thích kỹ ở Bước 1-2 của mục N-N ngay dưới.

### V1 — cột cứng trên Product

Chỉ riêng Q1 đã loại ngay lựa chọn đơn giản nhất, trước khi cần tới Q2/Q3:

```python
# V1 — 💭 loại ngay từ Q1: "phụ kiện/điện tử KHÔNG chia sẻ thuộc tính với quần áo"
class Product(models.Model):
    name = models.CharField(max_length=250)
    size = models.CharField(choices=[("S", "S"), ("M", "M"), ("L", "L")], null=True)
    color = models.CharField(max_length=50, null=True)
    # mở "điện tử" 6 tháng sau → cần ram, storage... size/color để trống vô nghĩa
```

Đây đúng dạng "Sai lầm thường gặp" mà chính [1b](./ecommerce-setup-1b-product-attribute.md) đã ghi nhận — nhưng 1b cũng nói rõ nó **đúng** ở quy mô 1 ngành hàng cố định mãi mãi. Q1 ("6 tháng nữa mở phụ kiện, KHÔNG chia sẻ thuộc tính") loại lựa chọn này ngay từ discovery, không cần đợi tới lúc bảng `Product` phình cột mới nhận ra.

### V2 — phản xạ tiếp theo: 1 cột JSONB

Với đúng Q1-Q3 đã trả lời (chưa có Q4-Q7), lựa chọn **nhiều khả năng nhất** tiếp theo:

```python
# V2 — 💭 thoả Q1 (mỗi category tự do định nghĩa key riêng), Q2 (vận hành tự thêm key, không deploy),
# Q3 (Postgres JSONB lọc được bằng @> / GIN index — "có vẻ" đủ dùng)
class Product(models.Model):
    name = models.CharField(max_length=250)
    category = models.CharField(max_length=100)   # "ao-thun" | "phu-kien" | "dien-tu"
    attributes = models.JSONField(default=dict)    # {"size": "M", "color": "Red"} hoặc {"ram": "8GB"}
```

**Vì sao đây là phản xạ nhiều khả năng nhất, không phải lựa chọn ngẫu nhiên:** thoả đúng 3 câu đã biết (Q1/Q2/Q3 ở trên), và là pattern xuất hiện dày đặc nhất cho đúng loại câu hỏi "flexible product attributes trên Postgres" — không phải Claude "sáng tạo" ra, mà là lựa chọn phổ biến nhất được củng cố qua tần suất xuất hiện trong tài liệu/mã nguồn đã thấy qua huấn luyện. Đáng chú ý: đây **chính xác** là "Cách A" mà [1b](./ecommerce-setup-1b-product-attribute.md) tự đặt tên và so sánh trực tiếp với thiết kế thật của Saleor (mục "So sánh 2 cách hiện thực 'thuộc tính linh hoạt'") — bản thân tài liệu nguồn đã xác nhận đây đúng là hướng người mới/phổ biến hay chọn, không phải suy diễn của file này.

### Điểm mù của V2 — chỉ lộ ra nếu hỏi tiếp Q4-Q7

Không phải lỗi thiết kế — là **thiếu thông tin discovery**. 4 câu chưa hỏi, 4 điểm mù tương ứng (nội dung kỹ thuật lấy nguyên từ so sánh đã grounded ở 1b, không suy diễn lại ở đây):

| Câu chưa hỏi | Điểm mù của V2 nếu câu trả lời là "có" |
|---|---|
| Q4 — giá trị dùng lại, đổi tên cần khớp mọi nơi? | "Đỏ" là chuỗi lặp lại y hệt trong JSON của MỌI sản phẩm màu đỏ — đổi tên hiển thị phải `UPDATE` toàn bộ document chứa nó, không có 1 dòng canonical để sửa 1 lần |
| Q3 (mở rộng) — lọc theo **khoảng** (giá, cân nặng), không chỉ khớp đúng giá trị? | GIN index trên jsonb lọc khớp giá trị tốt, nhưng so sánh khoảng (`>`, `<`) trên field lồng trong jsonb phải ép kiểu, không có B-Tree riêng cho từng field |
| Q6 — thuộc tính có trỏ sang entity khác? | Kiểu REFERENCE không có FK thật trong jsonb — xoá entity bị tham chiếu không tự CASCADE, phải tự dọn bằng job riêng |
| Q5 — đa ngôn ngữ? | Không có 1 dòng canonical để gắn bảng dịch — dịch "Đỏ" sang tiếng Anh phải tự nghĩ cơ chế riêng, không tái dùng được pattern dịch chuẩn (🧠 suy ra trực tiếp từ hệ quả "không có dòng canonical" ở dòng trên, 1b không viết riêng góc dịch thuật cho Cách A) |

*(3 dòng đầu ✅ trích thẳng từ mục so sánh Cách A/Cách B ở [1b](./ecommerce-setup-1b-product-attribute.md); dòng cuối 🧠 tự suy ra.)*

**Nếu buổi discovery dừng ở Q1-Q3, V2 là thiết kế hợp lý — không phải sai.** Đây không phải "thiết kế kém", mà là hệ quả trực tiếp của lượng thông tin discovery có tại thời điểm đó — đúng nguyên tắc đã nêu ở phương pháp gốc: **rule ẩn** (ở đây là Q4-Q7) mới là rule quyết định gap thật, không phải rule tường minh.

### V3 — nếu bị đẩy tiếp: EAV "cổ điển" 2 tầng, 1 cột value

Giả sử buổi discovery thứ 2 hỏi tiếp Q4 (có) và Q6 (có), nhưng Q7 (product-level vs variant-level) vẫn chưa hỏi. Phản xạ tiếp theo **nhiều khả năng nhất** — không phải nhảy thẳng lên đúng kiến trúc Saleor:

```python
# V3 — 💭 thoả Q4 (bảng giá trị dùng chung, dedup được) nhưng CHƯA tách product-level/variant-level (Q7),
# và nhiều khả năng dùng 1 cột value chung thay vì nhiều cột kiểu riêng — chưa có tín hiệu discovery nào
# cụ thể đẩy xa tới mức đó
class Attribute(models.Model):
    slug = models.SlugField(unique=True)          # "size", "ram"
    name = models.CharField(max_length=100)

class AttributeValue(models.Model):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    value = models.TextField()                     # "M" | "8GB" | "true" | "2026-01-01" — MỌI kiểu ép về chuỗi

class ProductAttributeValue(models.Model):          # chưa tách Product-level vs Variant-level
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    value = models.ForeignKey(AttributeValue, on_delete=models.CASCADE)
```

Đây là đúng dạng "EAV cổ điển 1 cột `value TEXT`" — và điều đáng chú ý: **chính [1b](./ecommerce-setup-1b-product-attribute.md) đã tự gọi tên đúng pattern này** khi giải thích vì sao Saleor KHÔNG làm vậy ("Không phải EAV cổ điển '1 cột `value TEXT`' — ép mọi kiểu về chuỗi, tự parse lại khi đọc"). Nói cách khác: tài liệu nguồn về Saleor tự xác nhận đây là hướng người mới hay đi tới nếu chỉ dừng ở "cần dùng lại giá trị" (Q4) mà chưa có tín hiệu cụ thể nào đẩy xa hơn.

**Vì sao V3 chưa tự nhảy tới ~11 cột kiểu riêng của Saleor:** không có câu nào trong 7 câu discovery đã liệt kê trực tiếp hỏi "mỗi kiểu thuộc tính có cần lưu đúng kiểu dữ liệu gốc (số, ngày, boolean), hay ép về chuỗi là đủ" — đây là chi tiết kỹ thuật (đánh đổi giữa schema đơn giản và đúng kiểu dữ liệu/index) mà 1 buổi discovery với founder khó tự lộ ra; thường chỉ dev tự nhận ra sau khi **đã vận hành** đủ lâu để thấy giá trị số bị lưu dạng chuỗi gây sai khi sort (`"10" < "9"` theo thứ tự chuỗi).

### Đào sâu — thiết kế đúng quan hệ N-N (kỹ năng dùng lại được, không riêng Saleor)

Tách riêng khỏi V1/V2/V3 vì đây không phải "1 bước trong ladder độ phức tạp" mà là **1 kỹ năng ngang** áp dụng được cho bất kỳ quan hệ N-N nào gặp sau này — tái dùng nguyên phương pháp 2 chiều đã dùng ở mức khái niệm tại Phần 3, giờ áp lại ở mức schema/code, đào sâu thêm phần "khi nào cần bảng trung gian tường minh" mà [`saleor-database-relationship-cardinality.md`](./saleor-database-relationship-cardinality.md) chưa đi sâu.

**Bước 1 — xác nhận đây thật sự là N-N, không phải 1-N, bằng câu hỏi 2 chiều:**

- Chiều 1: *"1 `Attribute` (VD Size) có thể gán cho nhiều `ProductType` không?"* → có (Size dùng chung cho cả Áo thun, Giày...).
- Chiều 2: *"1 `ProductType` (VD Áo thun) có thể có nhiều `Attribute` không?"* → có (Áo thun cần cả Size, Color, Chất liệu).

Cả 2 chiều đều "có" → N-N thật — đúng kết luận đã chốt ở Phần 3, giờ chỉ đổi từ ngôn ngữ nghiệp vụ ("Ngành hàng ↔ Thuộc tính") sang tên field cụ thể. (Nếu chỉ 1 chiều "có" — VD 1 `ProductVariant` có nhiều `Stock` nhưng 1 `Stock` chỉ thuộc đúng 1 `ProductVariant` — đó là 1-N, không phải N-N. Nhầm lẫn 2 loại này là lỗi thiết kế phổ biến nhất khi mới học quan hệ dữ liệu.)

**Bước 2 — thử phương án ngây thơ nhất: `ManyToManyField` phẳng:**

```python
# 💭 lựa chọn đầu tiên hầu như ai cũng nghĩ tới — Django cho sẵn field này, tự tạo 1 bảng
# trung gian ẩn (chỉ 2 cột FK, không tự thấy được, không thêm được cột nào khác)
class Attribute(models.Model):
    slug = models.SlugField(unique=True)
    product_types = models.ManyToManyField(ProductType)
```

Đủ dùng nếu quan hệ N-N **không mang thêm dữ liệu nào của riêng nó** — chỉ cần biết "có liên kết hay không". Nhưng đề bài ở đây đòi hỏi hơn thế: cần biết "Attribute này được dùng để **tạo ra Variant** hay chỉ hiển thị thông tin" (VD Size quyết định có bao nhiêu SKU khác nhau, Chất liệu thì không) — đây là dữ liệu KHÔNG thuộc về `Attribute` (dùng chung cho mọi ProductType nó gán vào), cũng KHÔNG thuộc về `ProductType` (dùng chung cho mọi Attribute nó có) — nó chỉ đúng cho **chính cặp quan hệ** (Attribute=Size, ProductType=Áo thun) → `variant_selection=True`, trong khi (Attribute=Chất liệu, ProductType=Áo thun) → `variant_selection=False`.

**Dấu hiệu quan trọng nhất để nhận ra khi nào `ManyToManyField` phẳng KHÔNG đủ:** hễ có bất kỳ thuộc tính nào chỉ đúng với **1 cặp cụ thể** (không đúng cho toàn bộ 1 trong 2 phía) — plain M2M hết dùng được, bắt buộc sang Bước 3.

**Bước 3 — nâng cấp: bảng trung gian tường minh (`through=`), và 3 quyết định phải chốt ngay lúc này:**

```python
# ✅ khuôn thật của Saleor — attribute/models/product_variant.py:55-78 (đã grounded ở 1b)
class AttributeVariant(SortableModel):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    product_type = models.ForeignKey(ProductType, on_delete=models.CASCADE)
    variant_selection = models.BooleanField(default=False)   # dữ liệu CỦA RIÊNG cặp quan hệ

    class Meta:
        unique_together = ("attribute", "product_type")      # unique(attribute, product_type) — 1b, sơ đồ ER
```

| # | Câu hỏi | Nếu "có" |
|---|---|---|
| 1 | Quan hệ có mang dữ liệu riêng của **cặp** (không thuộc về 1 trong 2 phía)? | Bắt buộc bảng trung gian tường minh (`through=`) — plain `ManyToManyField` hết dùng được |
| 2 | Thứ tự hiển thị có phải dữ liệu nghiệp vụ thật? | Kế thừa `SortableModel`/thêm cột `sort_order` — Saleor làm vậy vì thứ tự Attribute hiện trên form nhập sản phẩm là lựa chọn của nhân viên vận hành (muốn Size hiện trước Color), dễ bị quên vì "thứ tự" không nằm trong mô tả nghiệp vụ ban đầu |
| 3 | Có rủi ro gán trùng khi 2 request chạm cùng lúc? | `unique_together`/`UniqueConstraint` trên đúng cặp FK — thiếu dòng này, chỉ code tầng ứng dụng tự canh không trùng, dễ có 2 dòng trùng lặp thật trong DB nếu 2 request race nhau (đúng nguyên tắc "Check-Then-Act" đã nêu ở `CLAUDE.md`) |

**Bước 4 — phát hiện dễ bỏ sót nhất: đây là N-N xảy ra NHIỀU LẦN LIÊN TIẾP, không phải 1 lần:**

Nhìn lại đúng 3 tầng đã học ở [1b](./ecommerce-setup-1b-product-attribute.md) qua lăng kính N-N sẽ thấy: bảng trung gian của quan hệ N-N tầng 1 lại trở thành **1 phía** của quan hệ N-N tầng tiếp theo — không phải 3 tầng độc lập nằm cạnh nhau, mà xâu chuỗi:

```
Attribute ──< N-N #1: (attribute, product_type) >── ProductType
                       (bảng trung gian: AttributeVariant — CHƯA có Variant/giá trị nào)
                                │
                                │  chính AttributeVariant lại là 1 PHÍA của N-N tiếp theo
                                ▼
AttributeVariant ──< N-N #2: (assignment, variant) >── ProductVariant
                       (bảng trung gian: AssignedVariantAttribute — CHƯA có giá trị nào)
                                │
                                │  chính AssignedVariantAttribute lại là 1 PHÍA của N-N tiếp theo
                                ▼
AssignedVariantAttribute ──< N-N #3: (assignment, value) >── AttributeValue
                       (bảng trung gian: AssignedVariantAttributeValue — giá trị THẬT)
```

Đây là lý do thiết kế "trông phức tạp" nếu chỉ lướt qua — thực ra là **3 quyết định N-N độc lập**, mỗi quyết định áp đúng Bước 1-3 ở trên **riêng biệt**, không phải 1 quyết định khổng lồ duy nhất. Nhận ra "đây là N-N xếp chồng N-N" chính là kỹ năng tách 1 schema nhiều bảng thành từng quan hệ đơn giản để đọc — không tự nhiên mà 1b cần tới 4 bảng chỉ cho đúng 1 đường Variant. (Đúng 3 dòng N-N đã xác định bằng ngôn ngữ nghiệp vụ ở Phần 3 — Ngành hàng↔Thuộc tính, Biến thể↔Giá trị thuộc tính, Sản phẩm↔Giá trị thuộc tính — chính là N-N #1 và gộp #2+#3 ở đây, giờ nhìn lại bằng tên bảng cụ thể.)

**Checklist tái dùng khi tự thiết kế 1 quan hệ N-N mới (không riêng Product/Attribute):**

| Câu hỏi | Nếu "có" → |
|---|---|
| Cả 2 chiều đều "1 bên có nhiều bên kia" (Bước 1)? | Xác nhận N-N thật, không phải 1-N bị hiểu nhầm |
| Quan hệ mang dữ liệu riêng của CẶP? | Bắt buộc `through=`, không dùng `ManyToManyField` phẳng |
| Thứ tự hiển thị là dữ liệu nghiệp vụ thật? | Thêm `SortableModel`/cột `sort_order` |
| Có rủi ro gán trùng khi nhiều request cùng lúc? | `unique_together`/`UniqueConstraint` trên đúng cặp FK — không tin code tầng ứng dụng tự canh |
| Bảng trung gian tự nó lại cần N-N với 1 entity thứ 3? | Dấu hiệu "N-N xếp chồng" (Bước 4) — tách thành nhiều quan hệ độc lập, đừng cố nhét vào 1 bảng duy nhất |

### Thứ tự xây dựng thực tế — 3 tầng không ra đời cùng lúc

Nếu thật sự cầm task này đi code, thứ tự hợp lý nhất **không phải** viết đủ 3 tầng model rồi migrate 1 lần — mỗi tầng phụ thuộc tầng trước đã có dữ liệu thật để test, và tách PR đúng ranh giới N-N ở trên giúp review đúng chỗ rủi ro nhất trước khi có dữ liệu phụ thuộc vào nó:

1. **PR 1 — Tầng định nghĩa** (`Attribute`/`AttributeValue`) + màn hình admin CRUD riêng. Chạy độc lập được, test được ngay (tạo "Size", thêm giá trị "S"/"M"/"L") mà chưa cần đụng tới `Product`/`ProductType` nào.
2. **PR 2 — Tầng gán** (`AttributeVariant`/`AttributeProduct`, N-N #1 ở Bước 4) + màn hình "chọn Attribute cho ProductType". Phụ thuộc PR 1 đã có Attribute thật để chọn. Đây đúng là lúc áp Bước 1-3 của mục N-N phía trên — chốt `unique_together`, `variant_selection`, `sort_order` **ngay tại PR này**, vì thêm `unique_together` sau khi DB đã có bản ghi trùng thật sẽ tốn công dọn dữ liệu trước khi migrate được.
3. **PR 3 — Tầng giá trị thật** (`AssignedVariantAttributeValue`, N-N #2 và #3 ở Bước 4) + màn hình nhập sản phẩm. Phụ thuộc PR 2 đã có "quy tắc" để biết Variant nào được hỏi Attribute nào.
4. **PR 4 — GraphQL mutation/query lộ ra ngoài**, chỉ sau khi cả 3 tầng đã có dữ liệu thật để viết test integration đúng nghĩa, không mock rỗng.

**Vì sao thứ tự này quan trọng hơn vẻ ngoài thủ tục của nó:** PR 2 là nơi rủi ro cao nhất trong cả 4 PR (đúng bảng trung gian N-N, đúng constraint — sai ở đây thì PR 3/4 xây trên nền sai). Tách riêng PR 2 khỏi PR 3 nghĩa là reviewer tập trung đúng phần rủi ro cao nhất (schema quan hệ) TRƯỚC KHI có dữ liệu thật phụ thuộc vào nó, thay vì lẫn trong 1 PR khổng lồ khiến phần quan trọng nhất bị chìm giữa nhiều thay đổi khác.

### 3 lối tắt "nghe hợp lý nhưng sai" — luyện phân biệt, không chỉ đi theo 1 đường thẳng 💭

V1→V2→V3 ở trên là 1 đường thẳng — mỗi bước là phản xạ hợp lý nhất xét đúng lượng thông tin có tại thời điểm đó. Nhưng phân biệt đúng/sai thật sự khó không nằm ở việc đi đúng 1 đường thẳng — nó nằm ở chỗ **ngay cả khi đã có đủ thông tin (Q1-Q7 đều "có"), vẫn còn nhiều lối tắt nghe rất hợp lý nhưng sai**. 3 lối tắt dưới đây mỗi cái tempting vì 1 lý do kỹ thuật thật, và sai vì 1 lý do kỹ thuật thật khác — đọc code, tự đoán câu hỏi in nghiêng trước khi đọc phần "Vì sao sai", đây là bài luyện phân biệt, không phải danh sách antipattern để học thuộc lòng.

**Lối tắt A — dùng `GenericForeignKey` của Django thay vì tách cột theo entity:**

```python
# 💭 tempting vì đây là tính năng CHÍNH THỐNG của Django (django.contrib.contenttypes), không
# phải hàng tự chế — "gắn 1 AttributeValue vào BẤT KỲ model nào" nghe như giải quyết gọn cả bài
# toán Product-level vs Variant-level vs Category... trong 1 bảng duy nhất
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

class AssignedAttributeValue(models.Model):
    value = models.ForeignKey(AttributeValue, on_delete=models.CASCADE)
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey("content_type", "object_id")
```

*Tự đoán trước khi đọc tiếp: field nào ở trên KHÔNG phải FK thật theo đúng nghĩa ràng buộc DB?*

**Vì sao sai:** `object_id` chỉ là 1 số nguyên trần — Postgres không biết nó "trỏ tới" bảng nào, nên không tạo được FK constraint thật, không `CASCADE` ở tầng DB (Django tự giả lập CASCADE ở tầng ORM; bất kỳ thao tác nào đi vòng qua ORM — bulk SQL, tool khác, migration dữ liệu thô — đều có thể để lại dòng mồ côi vĩnh viễn). Muốn "lấy tên Product đang gắn value này" cũng không viết được 1 câu `JOIN` đơn giản, vì `content_type` có thể trỏ sang nhiều bảng khác nhau — phải tự rẽ nhánh theo `content_type_id` trong code, không đẩy được xuống SQL. Bằng chứng cụ thể nhất: Saleor **có đúng bài toán này thật** ở kiểu `REFERENCE` (1 AttributeValue trỏ sang Product, Variant, Collection, Category, hay Page) — và chọn **5 cột FK riêng biệt, luôn NULL trừ đúng 1 cột** (`reference_product`/`reference_variant`/`reference_collection`/`reference_category`/`reference_page`, đã thấy ở [1b](./ecommerce-setup-1b-product-attribute.md)) thay vì 1 `GenericForeignKey` duy nhất — trả giá bằng nhiều cột NULL hơn, đổi lại giữ FK thật + CASCADE thật + JOIN đơn giản cho từng loại tham chiếu.

**Lối tắt B — gộp Product-level và Variant-level vào 1 bảng, dùng cột `variant` nullable làm cờ phân biệt:**

```python
# 💭 tempting vì "đỡ phải tách 2 bảng AssignedProductAttributeValue/AssignedVariantAttributeValue,
# đỡ phải nhớ lúc nào JOIN bảng nào" — 1 bảng, check variant IS NULL hay NOT là biết ngay
class AssignedAttributeValue(models.Model):
    value = models.ForeignKey(AttributeValue, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    variant = models.ForeignKey(ProductVariant, on_delete=models.CASCADE, null=True, blank=True)
    # variant NULL → giá trị áp dụng cho cả Product; có variant → chỉ áp dụng riêng variant đó

    class Meta:
        unique_together = ("value", "product", "variant")
```

*Tự đoán trước khi đọc tiếp: `unique_together` ở trên có thật sự chặn được gán trùng trong MỌI trường hợp không?*

**Vì sao sai — lỗi rất dễ bỏ sót:** trong Postgres (và chuẩn SQL nói chung), `NULL` không bằng chính nó trong ràng buộc unique — 2 dòng cùng `(value=10, product=500, variant=NULL)` **không** bị coi là trùng, vì `NULL ≠ NULL` theo ngữ nghĩa SQL. Nghĩa là `unique_together` ở trên chặn được gán trùng ở cấp Variant (vì `variant` có giá trị cụ thể để so sánh), nhưng **không chặn được gán trùng ở cấp Product** (vì `variant=NULL` không kích hoạt ràng buộc) — vẫn có thể lỡ gán "Chất liệu=Cotton" cho cùng 1 Product 2 lần mà DB không báo lỗi gì. Lỗi này không lộ ra khi test tay vài lần đầu (dữ liệu ít, không ai cố tình gán trùng) — chỉ lộ ra khi có đủ thao tác đồng thời thật. Ngoài lỗi này, gộp 2 tầng vào 1 bảng còn xoá mất đúng ranh giới "định nghĩa" (tầng 2 — Attribute nào ĐƯỢC PHÉP là Product-level hay Variant-level) và "dữ liệu thật" (tầng 3) mà [1b](./ecommerce-setup-1b-product-attribute.md) đã giải thích rõ vì sao phải tách riêng — với thiết kế này, muốn biết "ProductType Áo thun có Attribute nào là Product-level" phải LỌC dữ liệu đã gán thay vì đọc thẳng 1 bảng quy tắc.

**Lối tắt C — chuẩn hoá triệt để: tách hẳn 1 bảng riêng cho mỗi kiểu dữ liệu (`input_type`):**

```python
# 💭 tempting vì "đúng chuẩn 3NF, không cột nào NULL vô nghĩa" — nghe như sửa đúng điểm yếu
# đã nêu ở V3 (Saleor có ~11 cột, hầu hết luôn NULL trên mỗi dòng)
class AttributeValueText(models.Model):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    value = models.CharField(max_length=255)

class AttributeValueNumeric(models.Model):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    value = models.FloatField()

class AttributeValueReference(models.Model):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, null=True, on_delete=models.CASCADE)
    variant = models.ForeignKey(ProductVariant, null=True, on_delete=models.CASCADE)
# ...lặp lại cho mỗi input_type còn lại (boolean, date_time, rich_text...), rồi mỗi bảng value
# còn cần thêm 1 bảng "gán" (assignment) RIÊNG của chính nó
```

*Tự đoán trước khi đọc tiếp: muốn hiển thị "toàn bộ thuộc tính của 1 Variant" lên 1 màn hình admin, câu query sẽ trông thế nào?*

**Vì sao sai — đúng nguyên tắc chung nhưng sai bài toán cụ thể:** không cột nào NULL vô nghĩa nữa, nhưng đổi lại: (1) số bảng nhân theo số `input_type` × 2 (Product-level + Variant-level) — Saleor có ~11 `input_type` nên đây là 22+ bảng chỉ để lưu giá trị thuộc tính; (2) thêm 1 `input_type` mới lại phải thêm bảng + migration mới — quay lại đúng vấn đề Q2 đã chặn ngay từ V1 ("thêm thứ mới không được cần deploy"), chỉ khác là xảy ra ở tầng kiểu dữ liệu thay vì tầng ngành hàng; (3) câu hỏi tự đoán ở trên không có câu trả lời gọn — phải gọi riêng biệt (hoặc `UNION`) qua toàn bộ 11+ bảng, tệ hơn hẳn cách "1 bảng nhiều cột NULL" của Saleor (chỉ 1 câu query). Đây là bằng chứng cụ thể cho nguyên tắc sẽ nhắc lại ở Phần 7: **chuẩn hoá là phương tiện, không phải mục đích** — tiêu chí đúng là khớp với cách dữ liệu THẬT được đọc/ghi, không phải "không cột nào NULL" xét riêng lẻ.

### Data flow — mô phỏng dữ liệu chảy qua đúng thứ tự implement

Khác mục trên (phân biệt đúng/sai), mục này bám theo **1 cách làm duy nhất** (Saleor thật) và trace xem ở MỖI bước implement — đúng thứ tự PR 1→2→3 đã lên ở mục "Thứ tự xây dựng thực tế" — bảng nào có dữ liệu gì, và tại sao bước sau luôn phải tham chiếu ngược lại bước trước. Đây chính là câu trả lời cho "tại sao quan hệ database có hình dạng này" — không phải lý thuyết, mà nhìn thấy được ngay trên trình tự dữ liệu phát sinh. Dùng lại đúng bộ dữ liệu mẫu ở [1b](./ecommerce-setup-1b-product-attribute.md) (Product "Áo thun form rộng - Đỏ", 2 Variant TSHIRT-M-RED/TSHIRT-L-RED, cùng id) để 2 file khớp nhau hoàn toàn — không phải dữ liệu mới, mà chính bộ dữ liệu đó, nhìn theo góc "được tạo ra theo trình tự nào".

**Giả định có sẵn trước khi bắt đầu trace:** `ProductType` id=100 ("Áo thun") đã tồn tại — input từ 1 luồng Setup khác (module ProductType), không phải output của luồng Product/Attribute đang trace ở đây.

#### Bảng chú giải — 10 bảng sẽ xuất hiện, giải thích bằng câu hỏi thường trước khi vào chi tiết

Trước khi trace từng bước, đọc qua bảng này 1 lượt — mỗi dòng trả lời "bảng này tồn tại để trả lời câu hỏi gì", tách khỏi cơ chế FK/id sẽ thấy ở các bước sau. Cột "Tầng" dùng lại đúng cách chia 3 tầng đã học ở [1b](./ecommerce-setup-1b-product-attribute.md) (Định nghĩa chung → Gán → Giá trị thật), cộng 2 bảng nằm NGOÀI phạm vi 3 tầng đó (entity thật — `Product`/`ProductVariant`/`ProductType` vốn đã có sẵn, không riêng của Attribute):

| Bảng | Tầng | Trả lời đúng câu hỏi gì | Ý nghĩa 1 dòng dữ liệu |
|---|---|---|---|
| `ProductType` | Entity có sẵn (ngoài trace) | "Đây là NGÀNH HÀNG/khuôn mẫu gì?" | "Áo thun" — khuôn quyết định 1 Product thuộc ngành nào, và Attribute nào áp dụng cho ngành đó |
| `Product` | Entity thật | "Đây là SẢN PHẨM cụ thể nào?" | "Áo thun form rộng - Đỏ" — 1 sản phẩm khách nhìn thấy trên catalog, thuộc đúng 1 ProductType |
| `ProductVariant` | Entity thật | "Đây là PHIÊN BẢN bán được cụ thể nào?" | "TSHIRT-M-RED" — đơn vị thật có giá/tồn kho/SKU riêng, thuộc đúng 1 Product |
| `Attribute` | 1 — Định nghĩa | "Thuộc tính này TÊN gì, kiểu dữ liệu gì?" | 1 khái niệm dùng chung toàn hệ thống — "Size" chỉ tồn tại đúng 1 dòng dù dùng cho bao nhiêu ngành hàng |
| `AttributeValue` | 1 — Định nghĩa | "Thuộc tính này có những GIÁ TRỊ hợp lệ nào?" | 1 giá trị cụ thể thuộc về 1 Attribute — "M" là 1 dòng, dùng lại được cho mọi Variant có Size=M, không lặp |
| `AttributeVariant` | 2 — Gán (quy tắc) | "ProductType này CÓ CẦN hỏi thuộc tính này không?" | 1 quy tắc — "Áo thun cần hỏi Size" — chưa chứa giá trị cụ thể nào |
| `AttributeProduct` | 2 — Gán (quy tắc) | Tương tự, cho nhánh Product-level | 1 quy tắc — "Áo thun cần hỏi Chất liệu" — áp dụng chung cho cả Product, không tách theo Variant |
| `AssignedVariantAttribute` | Giữa tầng 2 và 3 | "Variant cụ thể này có đang được hỏi thuộc tính này không?" | 1 "khung" nối 1 Variant thật với 1 quy tắc đã khai báo — TSHIRT-M-RED cần trả lời Size, nhưng chưa nói trả lời gì |
| `AssignedVariantAttributeValue` | 3 — Giá trị thật | "Variant cụ thể này ĐÃ CHỌN giá trị gì cho thuộc tính đó?" | Giá trị thật đã điền — TSHIRT-M-RED → Size = M |
| `AssignedProductAttributeValue` | 3 — Giá trị thật | "Product này đã chọn giá trị gì cho thuộc tính cấp Product?" | Giá trị thật ở cấp Product, dùng chung mọi Variant của nó — Áo thun form rộng → Chất liệu = Cotton |

**2 điểm dễ nhầm khi mới đọc bảng này:** (1) `AttributeVariant`/`AttributeProduct` (tầng 2) và `AssignedVariantAttributeValue`/`AssignedProductAttributeValue` (tầng 3) nghe tên rất giống nhau nhưng KHÁC hẳn về ý nghĩa — tầng 2 là quy tắc áp dụng cho cả 1 ProductType (VD toàn bộ Áo thun), tầng 3 là giá trị áp dụng cho đúng 1 Product/Variant cụ thể (VD chỉ TSHIRT-M-RED); (2) `AssignedVariantAttribute` (không có "Value" ở cuối tên) không chứa giá trị nào cả — nó chỉ là "khung" trung gian, dễ tưởng nhầm đã có dữ liệu thật vì tên khá giống `AssignedVariantAttributeValue`.

#### Các bảng liên kết với nhau qua đâu

Bảng chú giải ở trên giải thích từng bảng RIÊNG LẺ — bảng dưới đây nối chúng lại, liệt kê đúng 8 liên kết FK đủ để phủ hết cả 10 bảng, để thấy "liên quan như thế nào" tách khỏi phần dữ liệu mẫu cụ thể sẽ trace ở các bước sau:

| Bảng giữ FK | Trỏ tới | Qua cột | Ý nghĩa liên kết |
|---|---|---|---|
| `Product` | `ProductType` | `product_type_id` | Sản phẩm nào thuộc ngành hàng nào |
| `ProductVariant` | `Product` | `product_id` | Phiên bản bán nào thuộc sản phẩm nào |
| `AttributeValue` | `Attribute` | `attribute_id` | Giá trị nào thuộc thuộc tính nào |
| `AttributeVariant` | `Attribute` + `ProductType` | `attribute_id` + `product_type_id` | Quy tắc Variant-level: thuộc tính nào áp cho ngành hàng nào — **đi qua đúng bảng này**, Attribute ↔ ProductType mới thật sự là quan hệ N-N |
| `AttributeProduct` | `Attribute` + `ProductType` | `attribute_id` + `product_type_id` | Tương tự, cho nhánh Product-level |
| `AssignedVariantAttribute` | `AttributeVariant` + `ProductVariant` | `assignment_id` + `variant_id` | Variant thật nào đang áp dụng quy tắc nào |
| `AssignedVariantAttributeValue` | `AssignedVariantAttribute` + `AttributeValue` | `assignment_id` + `value_id` | "Khung" đó đã chọn đúng giá trị nào |
| `AssignedProductAttributeValue` | `Product` + `AttributeValue` | `product_id` + `value_id` | Product nào đã chọn giá trị nào (cấp Product, không qua "khung" trung gian nào) |

Nhìn cột "Trỏ tới" sẽ thấy 1 quy luật: **mọi bảng đều trỏ ngược về 1 bảng "gốc" hơn nó** (bảng gán trỏ về bảng định nghĩa, bảng giá trị trỏ về bảng gán) — không có FK nào đi vòng ngược lại. Đây chính là cấu trúc sẽ thấy lại rõ hơn ở phần trace từng bước ngay dưới.

#### Vì sao cần tới 10 bảng — thử gộp từng cặp để thấy lý do

10 bảng nghe có vẻ nhiều — cách rõ nhất để thấy vì sao KHÔNG gộp bớt được là tự hỏi "sao không gộp 2 bảng X, Y thành 1?" cho từng cặp liền kề, rồi xem cụ thể cái gì hỏng nếu gộp:

1. **Gộp `Attribute` + `AttributeValue`?** → `Attribute` là 1 khái niệm ("Size"), `AttributeValue` là NHIỀU giá trị của khái niệm đó ("M", "L", "XL"...) — quan hệ 1-N thật. Gộp lại nghĩa là chữ "Size" phải lặp lại trên mỗi dòng giá trị — đổi tên hiển thị "Size" phải sửa N dòng thay vì 1.
2. **Gộp `AttributeVariant` thẳng vào `Attribute` (thêm cột `product_type_id` lên `Attribute`)?** → 1 Attribute gán được cho NHIỀU ProductType (Size còn dùng cho "Giày") — thêm 1 cột `product_type_id` chỉ cho phép Attribute đó thuộc đúng 1 ProductType, sai bản chất N-N đã xác nhận ở mục "Đào sâu — thiết kế đúng quan hệ N-N" phía trên.
3. **Gộp `AttributeVariant` (quy tắc, dùng chung) và `AssignedVariantAttribute` (khung riêng từng Variant)?** → `AttributeVariant` id=5 ("Size @ Áo thun") dùng CHUNG cho MỌI Variant thuộc ProductType đó; `AssignedVariantAttribute` có 1 dòng RIÊNG cho từng Variant (id=800 cho Variant 900, id=801 cho Variant 901, cả 2 cùng trỏ về `assignment_id=5`). Gộp lại nghĩa là mỗi Variant mới phải chép lại toàn bộ quy tắc thay vì trỏ về 1 dòng dùng chung — cùng họ lỗi dedup như mục 1.
4. **Gộp `AssignedVariantAttribute` (khung) và `AssignedVariantAttributeValue` (giá trị)?** → theo đúng lý do [1b](./ecommerce-setup-1b-product-attribute.md) đã nêu, tầng "khung" tồn tại riêng vì `AttributeVariant` còn mang ý nghĩa "attribute này có dùng để TẠO RA Variant hay không" (`variant_selection`), cần nhóm theo (Variant × AttributeVariant) TRƯỚC khi biết giá trị cụ thể. (🧠 thêm, suy từ hình dạng schema chứ 1b không nói thẳng: cột `assignment_id` trên `AssignedVariantAttributeValue` là FK thường chứ không phải `OneToOne`, nên 1 "khung" về lý thuyết chứa được nhiều dòng giá trị — khớp với `input_type=MULTISELECT` — nhưng đây là suy luận từ cấu trúc, không phải câu 1b khẳng định trực tiếp.)
5. **Gộp `AssignedVariantAttributeValue` và `AssignedProductAttributeValue` thành 1 bảng chung?** → đây chính là Lối tắt B đã phân tích ở trên: cần 1 cột `variant` nullable để phân biệt 2 cấp, nhưng `NULL ≠ NULL` phá vỡ đúng ràng buộc `unique_together` cần có để chặn gán trùng.

**Nhận xét sau cả 5 câu hỏi:** không cặp nào gộp được mà không mất đúng 1 tính chất cụ thể (dedup, đúng bản chất N-N, nhóm đúng theo variant_selection, hoặc ràng buộc unique còn hoạt động). 10 bảng không phải "thiết kế cho nhiều" — đó là số bảng nhỏ nhất còn giữ đủ 4 tính chất đó cùng lúc. Đây là câu trả lời trực tiếp cho "tại sao phải tạo nhiều bảng như vậy".

#### Bước 1 (ứng PR 1 — tầng định nghĩa): tạo Attribute + AttributeValue, CHƯA gắn vào đâu cả

Dev tạo "Size" và 2 giá trị "M"/"L" — thao tác độc lập, chưa cần biết ProductType nào sẽ dùng nó:

| Bảng | Dòng mới |
|---|---|
| `Attribute` | id=2, slug=size, name=Size, input_type=DROPDOWN |
| `AttributeValue` | id=20, attribute_id=**2**, slug=m, name=M |
| `AttributeValue` | id=21, attribute_id=**2**, slug=l, name=L |

*Thử trả lời "Áo thun form rộng - Đỏ có Size gì?" ngay tại bước này* → **chưa trả lời được** — chưa có dòng nào nối "Size" với ProductType "Áo thun" hay với bất kỳ Product/Variant nào. `AttributeValue.attribute_id` là FK duy nhất tồn tại lúc này, trỏ ngược về `Attribute` vừa tạo — chưa có FK nào trỏ TỚI đây từ hướng ProductType/Product. Đây là lý do bước 2 phải tồn tại.

#### Bước 2 (ứng PR 2 — tầng gán, quan hệ N-N): gán Size vào ProductType "Áo thun"

| Bảng | Dòng mới | FK trỏ ngược về |
|---|---|---|
| `AttributeVariant` | id=5, attribute_id=**2**, product_type_id=**100**, variant_selection=True | `Attribute` id=2 (bước 1) + `ProductType` id=100 (có sẵn từ trước) |

Đây đúng là quan hệ N-N đã học ở mục "Đào sâu" phía trên — không thể là 1 field đơn (VD `Attribute.product_type_id`) vì 1 Attribute gán được cho NHIỀU ProductType (Size còn dùng cho "Giày" sau này) và 1 ProductType có NHIỀU Attribute — bắt buộc bảng trung gian riêng, giữ đúng cặp + field riêng của cặp (`variant_selection`).

*Thử lại câu hỏi trên* → giờ trả lời được "ProductType Áo thun CÓ CẦN hỏi Size không" (có) — nhưng vẫn **chưa trả lời được** "Variant CỤ THỂ nào có Size gì", vì chưa có `Product`/`ProductVariant` nào tồn tại, và chưa có dòng nào gắn 1 GIÁ TRỊ cụ thể (M hay L) vào bất kỳ thứ gì. Bước 2 chỉ tạo ra QUY TẮC ("Áo thun cần Size"), chưa tạo dữ liệu THẬT — lý do bước 3 phải tồn tại.

#### Bước 3 (ứng PR 3 — tầng giá trị thật): tạo Product/Variant thật, gán giá trị

Dev tạo Product "Áo thun form rộng - Đỏ" + 2 Variant, rồi nhập Size cho từng Variant trên màn hình sản phẩm:

| Bảng | Dòng mới | FK trỏ ngược về |
|---|---|---|
| `Product` | id=500, product_type_id=**100** | `ProductType` id=100 |
| `ProductVariant` | id=900 (TSHIRT-M-RED), product_id=**500** | `Product` id=500 |
| `ProductVariant` | id=901 (TSHIRT-L-RED), product_id=**500** | `Product` id=500 |
| `AssignedVariantAttribute` | id=800, variant_id=**900**, assignment_id=**5** | `ProductVariant` id=900 + `AttributeVariant` id=5 (**bước 2**) |
| `AssignedVariantAttribute` | id=801, variant_id=**901**, assignment_id=**5** | `ProductVariant` id=901 + `AttributeVariant` id=5 (**bước 2**) |
| `AssignedVariantAttributeValue` | id=950, assignment_id=**800**, value_id=**20** | `AssignedVariantAttribute` id=800 (dòng ngay trên) + `AttributeValue` id=20 "M" (**bước 1**) |
| `AssignedVariantAttributeValue` | id=951, assignment_id=**801**, value_id=**21** | `AssignedVariantAttribute` id=801 + `AttributeValue` id=21 "L" (**bước 1**) |

*Giờ mới trả lời được* "TSHIRT-M-RED có Size gì" — tra `AssignedVariantAttributeValue` có `assignment_id=800` → `value_id=20` → `AttributeValue` id=20 → "M". Đúng cách trace đã làm ở [1b](./ecommerce-setup-1b-product-attribute.md), nhưng lần này thấy rõ **từng dòng được tạo ra lúc nào**, không chỉ đọc dữ liệu đã có sẵn từ trước.

#### So sánh — đường Product-level (Chất liệu) ngắn hơn đúng 1 bước

Cùng logic trên, gán "Chất liệu = Cotton" cho CẢ Product (dùng chung cho 2 Variant, không tách riêng) chỉ cần 4 dòng thay vì 7:

| Bảng | Dòng mới | FK trỏ ngược về |
|---|---|---|
| `Attribute` | id=1, slug=chat-lieu | (độc lập, như bước 1 ở trên) |
| `AttributeValue` | id=10, attribute_id=**1**, slug=cotton | `Attribute` id=1 |
| `AttributeProduct` | id=6, attribute_id=**1**, product_type_id=**100** | `Attribute` id=1 + `ProductType` id=100 (ứng bước 2 ở trên) |
| `AssignedProductAttributeValue` | id=700, value_id=**10**, product_id=**500** | `AttributeValue` id=10 + `Product` id=500 (ứng bước 3, nhưng KHÔNG cần tầng `AssignedVariantAttribute` trung gian) |

*Vì sao thiếu hẳn 1 tầng so với đường Size:* Product-level không có khái niệm "chọn Variant nào cần hỏi" — mọi Variant của Product đó dùng chung 1 giá trị — nên `AssignedProductAttributeValue` gắn thẳng `product_id` + `value_id`, không cần bảng "khung" trung gian như `AssignedVariantAttribute` (bảng đó chỉ tồn tại để nhóm theo từng Variant riêng biệt, đúng nhu cầu chỉ đường Variant-level mới có).

#### Điểm mấu chốt — vì sao các quan hệ có đúng hình dạng này

Nhìn lại toàn bộ 3 bước: **mọi dòng dữ liệu mới luôn cần FK trỏ NGƯỢC về 1 dòng đã tồn tại từ bước trước** — không dòng nào trỏ "tới tương lai" (không thể tạo `AssignedVariantAttributeValue` trước khi có `AssignedVariantAttribute`; không thể có `AssignedVariantAttribute` trước khi có `AttributeVariant`; không thể có `AttributeVariant` trước khi có `Attribute`). Thứ tự PR 1→2→3 ở mục trước **không phải lựa chọn tuỳ ý cho gọn** — nó phản ánh đúng thứ tự phụ thuộc dữ liệu thật, và thứ tự đó lại phản ánh đúng **thứ tự nghiệp vụ thật ngoài đời**: nhân viên phải định nghĩa "Size là gì" trước (bước 1), rồi mới quyết định "Áo thun có cần hỏi Size không" (bước 2), rồi mới nhập "Variant này Size mấy" khi có sản phẩm thật để bán (bước 3) — không ai làm ngược lại được. Schema không áp đặt trình tự này — nó chỉ đang **mô hình hoá đúng** trình tự vốn đã tồn tại trong nghiệp vụ. Đây là câu trả lời đầy đủ cho "tại sao mối quan hệ database ra sao": không phải vì Saleor "thích" nhiều tầng, mà vì mỗi tầng ứng đúng 1 THỜI ĐIỂM khác nhau trong vòng đời dữ liệu, và FK giữa các tầng chỉ đang ghi lại đúng chuỗi phụ thuộc đó.

### Sai lầm thường gặp khi viết code cho schema ĐÃ ĐÚNG — khác hẳn "3 lối tắt"

"3 lối tắt" ở trên là sai **hình dạng schema**. 3 sai lầm dưới đây giả định schema đã đúng 100% (đúng như Saleor thật) — nhưng code TRUY VẤN/GHI vào đúng schema đó vẫn có thể sai. Đây là lớp lỗi khác hẳn, dễ bị bỏ qua vì "schema đúng rồi thì chắc ổn" — không đúng.

**Sai lầm 1 — N+1 query khi đọc thuộc tính của nhiều sản phẩm cùng lúc:**

```python
# 💭 phản xạ tự nhiên đầu tiên — đúng logic, sai hiệu năng
for product in Product.objects.filter(product_type_id=100):
    for av in product.assignedproductattributevalue_set.all():   # 1 query MỚI mỗi lần lặp
        print(av.value.name)
```

Với 1000 sản phẩm cùng ProductType, đoạn trên bắn ra **1001 query** — và càng nặng hơn khi nhớ lại "Bảng chú giải" ở trên: đọc đủ thuộc tính của 1 Variant cần JOIN qua tới 3-4 bảng, nên N+1 ở đây nhân thẳng theo độ sâu chuỗi FK vừa trace.

```python
# ✅ đúng — prefetch_related đi đúng chuỗi FK đã trace ở "Data flow"
products = Product.objects.filter(product_type_id=100).prefetch_related(
    "assignedproductattributevalue_set__value"
)
```

**Hậu quả cụ thể:** trang danh sách 1000 sản phẩm chậm hẳn (1000+ round-trip DB thay vì 2), nặng hơn nữa ở đường Variant-level (4 tầng JOIN). Đúng loại vấn đề mà pattern DataLoader (đã học ở `saleor-platform-engineering-patterns.md` §2, "chống N+1 GraphQL") giải quyết ở tầng resolver — không riêng ORM thuần. **Dấu hiệu nhận biết:** vòng lặp Python gọi tiếp 1 quan hệ khác BÊN TRONG vòng lặp ngoài — không liên quan schema đúng hay sai, N+1 luôn có thể xảy ra nếu quên `select_related`/`prefetch_related`.

**Sai lầm 2 — ghi qua nhiều bảng không bọc transaction:**

```python
# 💭 tempting vì "viết tuần tự cho dễ đọc"
assignment = AssignedVariantAttribute.objects.create(variant=variant, assignment=attr_variant)
AssignedVariantAttributeValue.objects.create(assignment=assignment, value=size_m)   # nếu dòng này lỗi...
```

Nếu dòng thứ 2 lỗi (validation, mất kết nối DB) SAU KHI dòng 1 đã commit — kết quả để lại đúng "khung mồ côi": `AssignedVariantAttribute` tồn tại nhưng KHÔNG có `AssignedVariantAttributeValue` nào đi kèm — trạng thái dở dang, khác hẳn trạng thái hợp lệ "chưa gán giá trị" đã thấy ở Bước 2 (Data flow), vì lần này là do code lỗi giữa chừng, không phải đang ở bước trung gian bình thường.

```python
# ✅ đúng — đúng nguyên tắc "Database Transactions" ở CLAUDE.md gốc dự án
from saleor.core.tracing import traced_atomic_transaction

with traced_atomic_transaction():
    assignment = AssignedVariantAttribute.objects.create(variant=variant, assignment=attr_variant)
    AssignedVariantAttributeValue.objects.create(assignment=assignment, value=size_m)
```

**Hậu quả cụ thể:** không bọc transaction nghĩa là 2 `.create()` có thể để lại 1 trong 2 thành công — sinh dữ liệu dở dang không tự dọn được, phải viết script riêng để tìm và xoá "khung mồ côi" sau này. **Dấu hiệu nhận biết:** ≥2 lệnh ghi liên tiếp trên các bảng CÓ QUAN HỆ PHỤ THUỘC (bảng sau cần bảng trước tồn tại) mà không nằm trong cùng 1 khối transaction.

**Sai lầm 3 — race condition dù schema đã đúng, vì code tầng ứng dụng tự kiểm tra trước khi ghi:**

```python
# 💭 tempting vì "đọc dễ hiểu, kiểm tra trước cho chắc"
existing = AssignedVariantAttributeValue.objects.filter(assignment=assignment, value=size_m).first()
if not existing:
    AssignedVariantAttributeValue.objects.create(assignment=assignment, value=size_m)
    # 2 request cùng lúc → cả 2 đều đọc thấy "not existing" TRƯỚC khi request nào kịp ghi → cả 2 cùng insert
```

Ngay cả khi schema ĐÚNG (đủ `unique_together`, không phải Lối tắt B) — đoạn trên vẫn có thể **crash bằng `IntegrityError` không được xử lý** nếu 2 request chạm cùng lúc — đúng kiểu race condition "Check-Then-Act" đã nêu ở `CLAUDE.md`.

```python
# ✅ đúng — để chính DB (unique constraint) là trọng tài cuối cùng, không tự kiểm tra trước
AssignedVariantAttributeValue.objects.get_or_create(assignment=assignment, value=size_m)
```

**Hậu quả cụ thể:** schema đúng KHÔNG tự động bảo vệ khỏi lỗi ở tầng ứng dụng — `unique_together` chỉ đảm bảo không có 2 dòng trùng trong DB, không đảm bảo code gọi nó luôn xử lý đẹp trường hợp trùng (nếu không bắt đúng cách, request thứ 2 trả lỗi 500 cho người dùng thay vì âm thầm coi như đã tồn tại). **Dấu hiệu nhận biết:** pattern "check rồi mới create" (`if not X.objects.filter(...).exists(): X.objects.create(...)`) xuất hiện ở bất kỳ đâu ghi dữ liệu có khả năng bị gọi đồng thời.

**Điểm chung của cả 3:** khác "3 lối tắt" (sai từ lúc THIẾT KẾ bảng), cả 3 sai lầm này chỉ lộ ra ở tầng **cách gọi ORM** — đúng bằng chứng cho việc "đọc code thấy hợp lý, schema đúng rồi" vẫn chưa đủ để tin tưởng 1 đoạn ghi/đọc dữ liệu — phải tự hỏi thêm: có N+1 không, có transaction bọc đủ không, có đang tự kiểm tra trước khi ghi thay vì để DB làm trọng tài không.

---

## Phần 5 — Đối chiếu thực tế: 2 hệ thống thật, 2 lựa chọn khác hẳn nhau ✅

Trước khi so sánh với Saleor, đáng nhìn qua 1 hệ thống thật KHÁC đã giải đúng bài toán này bằng lựa chọn thứ 3 hoàn toàn — chứng minh không có 1 "đáp án đúng duy nhất", chỉ có đánh đổi phù hợp với hạ tầng sẵn có.

**WooCommerce thật — không chọn EAV quan hệ, không chọn JSONB, mà tái dùng hạ tầng WordPress có sẵn:** mỗi biến thể là 1 WordPress post riêng (custom post type `product_variation`); thuộc tính dùng chung nhiều sản phẩm (VD "Color") là 1 WordPress **taxonomy** thật tiền tố `pa_` (`wc_attribute_taxonomy_name()`); thuộc tính riêng 1 sản phẩm lưu dạng serialized meta — khác hẳn cả V2 (JSONB) lẫn Saleor (bảng EAV riêng). ✅ [`ecommerce-domain-solutions.md` — Vấn đề 6](../solutions/ecommerce-domain-solutions.md#vấn-đề-6-sản-phẩm-có-biến-thể-sizemàu--mỗi-ngành-hàng-cần-bộ-thuộc-tính-lọc-khác-nhau), [`woocommerce-ecommerce-domain-knowledge.md`](./woocommerce-ecommerce-domain-knowledge.md).

**Không phải "WooCommerce thiết kế tệ hơn Saleor"** — bối cảnh hạ tầng khác hẳn: WooCommerce là 1 plugin cài lên WordPress đã có sẵn CPT + taxonomy vận hành tốt hàng triệu site — tận dụng thẳng hạ tầng đó rẻ hơn tự xây EAV riêng, đổi lại bị khoá chặt vào WordPress, không portable sang stack khác. Saleor không có "WordPress" nào để tận dụng — tự thiết kế EAV riêng là lựa chọn hợp lý cho đúng bối cảnh của nó (backend độc lập, headless, không gắn CMS nào).

**Saleor thật — EAV 3 tầng, nhiều cột kiểu riêng theo `input_type`** (đã học đầy đủ ở [1b](./ecommerce-setup-1b-product-attribute.md), tóm tắt đúng phần cần đối chiếu, không lặp lại toàn bộ ở đây):

```
Tầng 1 Attribute (định nghĩa chung)
   → Tầng 2 AttributeVariant / AttributeProduct (gán vào ProductType, tách 2 đường
     product-level và variant-level)
   → Tầng 3 AssignedVariantAttributeValue / AssignedProductAttributeValue (giá trị thật)
   → AttributeValue (~11 cột kiểu riêng theo input_type: slug/value, numeric, boolean,
     date_time, rich_text (JSONB thật), reference_product/variant/collection/category/page...)
```

---

## Phần 6 — Bảng so sánh tổng hợp 5 hướng

| Tiêu chí | V1 — cột cứng 💭 | V2 — JSONB 💭 | V3 — EAV cổ điển 💭 | WooCommerce thật ✅ | Saleor thật ✅ |
|---|---|---|---|---|---|
| Thêm ngành hàng mới có cần deploy? | Có (thêm cột) | Không | Không | Không (taxonomy term qua admin) | Không |
| Lọc theo khoảng (giá, cân nặng) có index đúng kiểu? | Có (cột thật) | Không (ép kiểu trong jsonb) | Không (mọi giá trị là chuỗi) | Hạn chế (term là chuỗi; custom attribute qua meta khó lọc khoảng) | Có (`numeric`/`date_time` là cột kiểu riêng, B-Tree riêng theo 1b) |
| Đổi tên hiển thị 1 giá trị dùng chung — sửa mấy chỗ? | 1 (nếu là cột `choices`) | N (mọi document đang lặp lại chuỗi đó) | 1 (đã tách bảng `AttributeValue`) | 1 (sửa tên taxonomy term) | 1 |
| Thuộc tính kiểu REFERENCE có FK thật/CASCADE? | Không áp dụng | Không (không có FK trong jsonb) | Không (V3 chưa có cột `reference_*` riêng) | Có 1 phần (trỏ ID post-to-post, không có ràng buộc CASCADE chặt như FK quan hệ) | Có (`reference_product`/`reference_variant`... là FK thật) |
| Phân biệt product-level vs variant-level? | Không | Không (chung 1 JSON trên Product) | Không (V3 chưa tách) | Có 1 phần (attribute khai ở product cha, variation kế thừa/override) | Có (2 đường song song, tách bảng nối riêng) |
| Đa ngôn ngữ | Tự thêm field `_en`/`_vi` | Tự thiết kế riêng, không pattern chuẩn | Dễ thêm (đã có dòng canonical để trỏ) | Có (plugin WPML/Polylang dùng chung cơ chế dịch post/taxonomy của WordPress) | Có (`AttributeValueTranslation`, 1 dòng canonical) |
| Chi phí xây ban đầu | Thấp nhất | Thấp | Trung bình | Thấp (tận dụng hạ tầng có sẵn) | Cao nhất (tự thiết kế toàn bộ layer) |
| Đúng khi nào | 1 ngành hàng, mãi mãi | 2-3 ngành hàng, chưa cần lọc khoảng/dịch/REFERENCE | Cần dedup+dịch nhưng chưa cần đúng kiểu dữ liệu/tách product-variant | Đã có sẵn WordPress, chấp nhận khoá hạ tầng | Catalog đa ngành hàng thật, cần đủ tín hiệu Q4-Q7 |

*Ghi chú về mức độ grounded: 2 cột "thật" đánh dấu ✅ ở phần data model (đã có nghiên cứu + trích dẫn code trong repo này qua 1b/solutions/woocommerce). Riêng hành vi UI admin cụ thể của WooCommerce (VD tốc độ thêm taxonomy term không cần deploy) dựa trên kiến thức nền tảng chung về WordPress, chưa tự grep lại riêng khi viết file này — mức độ chắc chắn thấp hơn phần data model.*

---

## Phần 7 — Đánh giá: vì sao có khoảng cách, và khi nào KHÔNG nên nhảy thẳng lên Saleor thật 🧠

Khoảng cách giữa V1→V2→V3 và Saleor thật **không đến từ việc thiết kế greenfield "kém hơn"** — đến từ 2 nguồn khác hẳn:

**1. Bất đối xứng thông tin theo thời gian.** Thiết kế của Saleor là kết quả của nhiều năm vận hành thật qua hàng nghìn merchant — những điểm mù ở Phần 4 (rename lặp lại, GIN index không đúng kiểu, CASCADE không cảnh báo) không lộ ra trong 1 buổi discovery, chúng lộ ra sau hàng trăm incident thật. [1b](./ecommerce-setup-1b-product-attribute.md) tự minh hoạ đúng loại incident này: gỡ 1 Attribute khỏi ProductType (nhìn như sửa cấu hình an toàn) thực chất CASCADE xoá vĩnh viễn giá trị đã chọn trên MỌI Variant đang dùng ProductType đó — không soft-delete, không cảnh báo riêng ở tầng API (cùng nội dung được nhắc lại ở "Phát hiện 8" trong [`ecommerce-phase1-setup-deep-dive.md`](./ecommerce-phase1-setup-deep-dive.md)). Không buổi kickoff nào, kể cả với BA giỏi nhất, tái tạo được lượng thông tin đó trước khi viết dòng code đầu tiên — nó chỉ xuất hiện sau khi hệ thống đã chạy đủ lâu để có sự cố thật.

**2. Saleor khấu hao chi phí phức tạp trên quy mô lớn hơn nhiều so với 1 dự án đơn lẻ.** Đây chính là điều mục "Đánh đổi thật" (1b) của [`ecommerce-phase1-setup-deep-dive.md`](./ecommerce-phase1-setup-deep-dive.md) đã nói rõ: EAV *"đáng đầu tư khi catalog thật sự đa ngành hàng... không cần khi shop chỉ bán đúng 1 ngành hàng cố định mãi mãi — field cố định đơn giản hơn, nhanh hơn, dễ debug hơn."* Saleor trả giá đó 1 lần, dùng lại cho **mọi** merchant dùng platform; 1 dự án greenfield đơn lẻ trả đúng giá đó cho **đúng 1 khách hàng** — tỷ lệ chi phí/lợi ích hoàn toàn khác.

**Quy tắc rút ra — map 7 câu hỏi discovery sang mức độ phức tạp nên chọn** (không phải "luôn chọn Saleor thật vì nó đúng nhất"):

| Câu trả lời discovery thật | Mức độ nên chọn |
|---|---|
| Q1 = "không" (mãi mãi 1 ngành hàng) | V1 — cột cứng; chính [1b](./ecommerce-setup-1b-product-attribute.md) xác nhận đây là lựa chọn **đúng** ở quy mô đó, không phải thoả hiệp tạm thời |
| Q1 = "có", nhưng Q4/Q6 = "không" | V2 — JSONB đủ dùng, chưa cần trả giá EAV quan hệ |
| Q4 = "có" nhưng Q6/Q7 = "không" | V3 — EAV cổ điển đủ dùng |
| Q4, Q6, Q7 đều "có" | Đáng trả giá đầy đủ kiểu Saleor thật |

**Bài học chính không phải "học thuộc mô hình EAV"** — đó chỉ là 1 fact tra cứu được, đã có sẵn ở [1b](./ecommerce-setup-1b-product-attribute.md). Kỹ năng thật sự đáng luyện là **biết hỏi đúng 7 câu discovery đủ sớm để biết mình đang ở dòng nào trong bảng trên — trước khi viết dòng code đầu tiên**, không phải sau khi bảng `Product` đã phình cột hay JSON đã lặp lại hàng nghìn document.

**Một quan sát về chính Claude (💭 tự đánh giá, không phải fact khách quan kiểm chứng được):** nếu không có áp lực discovery cụ thể như ở Phần 2, xu hướng nhiều khả năng nhất là neo vào lựa chọn thoả **đúng yêu cầu đã nêu ở mức tối thiểu** — V1/V2/V3 ở Phần 4 đều dừng lại đúng chỗ thông tin dừng, không tự "nhảy" xa hơn. Đây thực ra là bản năng đúng (tránh over-engineering khi chưa có tín hiệu cần), miễn là người phụ trách discovery chủ động hỏi tiếp khi cần — thay vì trông đợi 1 thiết kế tự động "biết" tới đáp án đầy đủ mà không có tín hiệu nào yêu cầu điều đó.

### Luyện phản biện — bảo vệ quyết định trước 3 kiểu phản đối thường gặp

Biết "khi nào nên chọn gì" (bảng ở trên) là điều kiện cần nhưng chưa đủ — judgment thật sự bị test lúc **có người phản đối** quyết định của bạn, không phải lúc tự đọc kết luận 1 mình. 3 câu phản đối dưới đây, mỗi câu ứng đúng 1 lối tắt đã phân tích ở Phần 4 (Lối tắt A/B/C) — tự viết câu trả lời của bạn trước, chỉ mở phần gợi ý sau khi đã viết xong; nếu mở ngay, bài luyện coi như không có tác dụng.

**Phản đối 1 — Founder (áp lực tốc độ):**
> *"Sao phải phức tạp vậy, JSONB nhanh hơn nhiều mà? Tụi mình đang gấp ra mắt, không có thời gian xây 4 tầng bảng."*

<details><summary>Gợi ý câu trả lời (chỉ mở sau khi đã tự viết)</summary>

Không phản bác thẳng "JSONB chậm hơn" — founder đúng ở chỗ JSONB xây nhanh hơn thật, phủ nhận thẳng sẽ mất uy tín ngay từ câu đầu. Thay vào đó quay lại đúng 3 câu discovery ĐÃ trả lời "có" (Q4/Q6/Q7 — dùng lại giá trị + dịch, thuộc tính tham chiếu entity khác, tách product/variant-level) và hỏi ngược: *"3 yêu cầu này mình đã thống nhất từ đầu, JSONB không đáp ứng đúng cả 3 — mình bỏ bớt yêu cầu nào, hay chấp nhận trả giá thời gian?"* Đồng thời trấn an bằng thứ tự xây dựng đã có ở Phần 4 (4 PR tách nhỏ) — không phải "xây xong cả 4 tầng mới chạy được", PR 1 (Attribute/AttributeValue) đã demo được ngay, không cần đợi đủ cả 3 tầng mới có gì để xem.
</details>

**Phản đối 2 — Dev junior trong team (đề xuất Lối tắt A):**
> *"Sao không dùng ContentType/GenericForeignKey của Django luôn? Có sẵn trong framework, code ít hơn, lại tổng quát — gắn được cho Product, Variant, hay Category nào cũng được."*

<details><summary>Gợi ý câu trả lời (chỉ mở sau khi đã tự viết)</summary>

Thừa nhận đúng điểm mạnh (ít code hơn, tổng quát hơn thật) trước, rồi hỏi cụ thể: *"Field REFERENCE của mình có cần khi xoá Product bị tham chiếu thì tự động dọn sạch giá trị attribute đang trỏ tới nó không?"* — câu trả lời thường là có (tránh dữ liệu rác tích tụ), và đó là lúc chỉ ra `GenericForeignKey` không `CASCADE` được ở tầng DB, chỉ giả lập ở tầng ORM. Đưa bằng chứng cụ thể thay vì lý thuyết suông: chính Saleor có đúng bài toán "trỏ sang nhiều loại entity" ở kiểu REFERENCE và chọn 5 cột FK riêng thay vì 1 GenericForeignKey — không phải vì không biết tới tính năng này, mà vì đã đánh đổi có chủ đích.
</details>

**Phản đối 3 — Tech lead (đề xuất Lối tắt C):**
> *"Bảng AttributeValue có tới 11 cột mà mỗi dòng chỉ dùng đúng 1 cột, còn lại NULL hết — vi phạm chuẩn hoá cơ bản. Sao không tách mỗi kiểu dữ liệu ra 1 bảng riêng cho sạch?"*

<details><summary>Gợi ý câu trả lời (chỉ mở sau khi đã tự viết)</summary>

Không né tránh — xác nhận đúng đây LÀ đánh đổi có thật (11 cột, đa số NULL, không phải 3NF thuần tuý). Nhưng hỏi ngược lại đúng câu hỏi tiêu chí: *"Màn hình admin cần hiển thị toàn bộ thuộc tính của 1 Variant bằng mấy câu query — 1 câu với thiết kế hiện tại, hay phải gọi qua 11+ bảng nếu tách riêng?"* và *"Thêm 1 kiểu dữ liệu mới có cần deploy schema mới không?"* — có, nhưng tần suất thêm `input_type` mới thấp hơn nhiều so với tần suất thêm ngành hàng mới (bài toán đó đã giải ở tầng Attribute/ProductType, không phải tầng AttributeValue). Chuẩn hoá là phương tiện phục vụ đúng bài toán đọc/ghi, không phải mục tiêu tự thân — đổi 1 chút "sạch" theo lý thuyết lấy 1 câu query đơn giản hơn nhiều là đánh đổi hợp lý ở đúng bài toán này.
</details>

---

## Phần 8 — Checklist tái sử dụng cho module khác

7 câu hỏi discovery ở Phần 2 và bảng ánh xạ ở Phần 7 không riêng cho Product/Attribute — cùng khuôn "V1 (cứng) → V2 (lối tắt phổ biến) → V3 (nửa vời) → thật" nhiều khả năng lặp lại được cho các sub-function khác của Setup đã học ở [`ecommerce-phase1-setup-deep-dive.md`](./ecommerce-phase1-setup-deep-dive.md) — 1e (Shipping, nhiều trục nhất) hay 1f (Voucher/Promotion, 2 lifecycle tách biệt) đều là ứng viên tốt để lặp lại đúng bài tập này ở phiên sau (chưa làm trong file này).

## Tự luyện tiếp theo

Áp đúng trình tự ở trên (bối cảnh giả định → discovery → tự thiết kế V1/V2/V3 → đối chiếu) cho 1 trong 2 module:

- ~~**1c Pricing**~~ ✅ Đã làm — xem [`saleor-greenfield-design-practice-pricing.md`](./saleor-greenfield-design-practice-pricing.md) (2026-08-01, cùng công ty/nhân vật tiếp nối mạch này).
- **1g Warehouse/Stock** — bối cảnh gợi ý: *"Nhiều khách cùng mua 1 sản phẩm chỉ còn đúng 1 cái, cùng lúc — làm sao chỉ đúng 1 người giữ được hàng?"* Tự hỏi: "còn hàng" là 1 cột số nguyên hay 1 giá trị suy ra tại thời điểm hỏi?

Không mở [1c](./ecommerce-setup-1c-pricing.md)/[1g](./ecommerce-setup-1g-warehouse-stock.md) trước khi tự làm xong V1-V3 — đúng nguyên tắc thứ tự đã nêu ở [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md): làm xong phần tự thiết kế trước, mở tài liệu đối chiếu sau, để gap thật không bị che bởi việc "tìm đúng cái mình đang cố tìm".

## Liên hệ / Nguồn

- [`ecommerce-setup-1b-product-attribute.md`](./ecommerce-setup-1b-product-attribute.md) — nguồn "Saleor thật" chính dùng trong file này, mọi trích dẫn ✅ về Saleor đều từ đây trừ khi ghi chú khác.
- [`saleor-ba-ambiguous-to-artifact-practice.md`](./saleor-ba-ambiguous-to-artifact-practice.md) — phương pháp discovery (Stakeholder map/Bảng business rule) tái dùng nguyên ở Phần 2.
- [`saleor-database-relationship-cardinality.md`](./saleor-database-relationship-cardinality.md) — phương pháp câu hỏi 2 chiều xác định N-N tái dùng 2 lần: ở Phần 3 (mức khái niệm, ngôn ngữ nghiệp vụ) và đào sâu thêm ở mục "Đào sâu — thiết kế đúng quan hệ N-N" (Phần 4, mức schema/code, phần "khi nào cần bảng trung gian tường minh" mà file gốc chưa đi sâu).
- [`ecommerce-phase1-setup-deep-dive.md`](./ecommerce-phase1-setup-deep-dive.md) — mục "Đánh đổi thật" (1b) và "Phát hiện 8" (CASCADE) dùng ở Phần 7.
- [`ecommerce-domain-solutions.md` — Vấn đề 6](../solutions/ecommerce-domain-solutions.md#vấn-đề-6-sản-phẩm-có-biến-thể-sizemàu--mỗi-ngành-hàng-cần-bộ-thuộc-tính-lọc-khác-nhau) + [`woocommerce-ecommerce-domain-knowledge.md`](./woocommerce-ecommerce-domain-knowledge.md) — nguồn WooCommerce thật dùng ở Phần 5.
