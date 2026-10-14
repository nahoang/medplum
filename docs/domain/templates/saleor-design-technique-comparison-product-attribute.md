# Product & Attribute — So sánh kỹ thuật thiết kế database & lý do Saleor chọn EAV+STI hybrid

> **Góc nhìn thứ 3** bên cạnh 2 file đã có cho cùng entity Product/Attribute — mỗi file trả lời 1 câu hỏi khác nhau, đọc cả 3 mới trọn bức tranh:
> - [`ecommerce-setup-1b-product-attribute.md`](./ecommerce-setup-1b-product-attribute.md) — Saleor **làm gì** (đọc code đã hoàn thiện, đúng thiết kế thật)
> - [`saleor-greenfield-design-practice-product-attribute.md`](./saleor-greenfield-design-practice-product-attribute.md) — đi **đúng trình tự** 1 dự án greenfield sẽ trải qua (discovery → conceptual → logical), dừng ở V1/V2/V3 là các bước trên đường đi, không phải kỹ thuật cạnh tranh
> - **File này** — coi 6 kỹ thuật đã biết là **ứng viên ngang hàng**, viết đủ code cho từng ứng viên (kể cả 2 cái chưa xuất hiện ở file greenfield: STI thuần, CTI), so sánh trực diện, rồi hỏi "tại sao ứng viên EAV+STI hybrid thắng, 5 ứng viên còn lại thua ở đâu" — không đi theo trình tự thời gian discovery nữa, mà theo cấu trúc luyện tập của file greenfield (code cho từng phương án → sai lầm thường gặp → lối tắt tự đoán → luyện phản biện → checklist)
> - [`saleor-eav-hybrid-deep-dive.md`](./saleor-eav-hybrid-deep-dive.md) — **góc nhìn thứ 4**, đào sâu riêng đúng 1 cặp (EAV thuần vs hybrid) đã so sánh ở đây: cơ chế Postgres thật đằng sau bug sort, đối chứng Magento EAV/WordPress `wp_postmeta`, và migration path nếu 1 hệ thống cần nâng cấp từ EAV thuần lên hybrid

---

## Mục lục

- [Quy tắc nhãn nguồn](#quy-tắc-nhãn-nguồn-trong-file-này)
- [Phần 1 — Bài toán cần giải](#phần-1-bài-toán-cần-giải-5-đặc-tính-cụ-thể)
- [Phần 2 — 6 ứng viên, đủ code từng kỹ thuật](#phần-2-6-ứng-viên-đủ-code-từng-kỹ-thuật)
  - [2.1 Fixed columns](#21-fixed-columns)
  - [2.2 STI thuần](#22-sti-thuần-single-table-inheritance)
  - [2.3 CTI](#23-cti-class-table-inheritance)
  - [2.4 JSONB (Cách A)](#24-jsonb-cách-a)
  - [2.5 EAV thuần](#25-eav-thuần)
  - [2.6 EAV+STI hybrid — Saleor thật](#26-eavsti-hybrid--saleor-thật)
- [Phần 3 — Bảng so sánh trực diện](#phần-3-bảng-so-sánh-trực-diện)
- [Phần 4 — Vì sao 5 ứng viên còn lại bị loại](#phần-4-vì-sao-5-ứng-viên-còn-lại-bị-loại--áp-đúng-p1-p5)
- [Phần 5 — 4 lối tắt nghe hợp lý nhưng sai](#phần-5-4-lối-tắt-nghe-hợp-lý-nhưng-sai-tự-đoán-trước-khi-đọc)
- [Phần 6 — Thiết kế thật của Saleor — 5 ngã rẽ](#phần-6-thiết-kế-thật-của-saleor--tại-mỗi-ngã-rẽ-vì-sao-không-đi-theo-5-kỹ-thuật-kia)
  - [6.1 Tránh Fixed columns](#61-ngã-rẽ-tránh-fixed-columns-21--mã-hoá-ai-dùng-thuộc-tính-gì-vào-dữ-liệu-không-vào-schema)
  - [6.2 Tránh STI thuần](#62-ngã-rẽ-tránh-sti-thuần-22--dùng-đúng-discriminator-sti-nhưng-chỉ-ở-nơi-số-loại-thật-sự-cố-định)
  - [6.3 Tránh CTI](#63-ngã-rẽ-tránh-cti-23--1-bảng-giá-trị-duy-nhất-không-tách-bảng-theo-input_type)
  - [6.4 Tránh JSONB toàn cục](#64-ngã-rẽ-tránh-jsonb-toàn-cục-24--jsonb-chỉ-dùng-cho-đúng-1-trong-12-input_type)
  - [6.5 Tránh EAV thuần](#65-ngã-rẽ-tránh-eav-thuần-25--không-collapse-12-kiểu-về-1-cột-value)
- [Phần 7 — Ma trận điểm mạnh/yếu](#phần-7-ma-trận-điểm-mạnhyếu--vì-sao-phải-ghép-không-dùng-thuần-1-cái)
- [Phần 8 — Đối chứng ShippingMethod](#phần-8-đối-chứng-trong-chính-saleor--khi-nào-không-cần-ghép)
- [Phần 9 — Luyện phản biện](#phần-9-luyện-phản-biện--bảo-vệ-quyết-định-eavsti-hybrid-trước-3-kiểu-phản-đối-thường-gặp)
- [Phần 10 — Checklist tái sử dụng](#phần-10-checklist-tái-sử-dụng)
- [Tự luyện tiếp theo](#tự-luyện-tiếp-theo)
- [Liên hệ / Nguồn](#liên-hệ-nguồn)

---

## Quy tắc nhãn nguồn trong file này

| Nhãn | Nghĩa | Dùng ở đâu trong file |
|---|---|---|
| ✅ grounded | Đọc trực tiếp từ code Saleor thật, có `file:line` | Phần 2.6, Phần 6, Phần 8 |
| 💭 minh hoạ | Code phản-thực-tế (counterfactual) — Saleor không làm vậy, viết ra để so sánh, không tồn tại trong repo | Phần 2.1-2.5, Phần 5 |
| 🧠 suy luận | Lập luận "vì sao Saleor không chọn X" — suy luận từ đối chiếu code thật với đặc tính bài toán, không phải lý do Saleor tự công bố (không có tài liệu thiết kế gốc nào của Saleor để trích) | Phần 4, Phần 7, Phần 9 |

---

## Phần 1 — Bài toán cần giải (5 đặc tính cụ thể)

Bán nhiều ngành hàng (áo, sách, điện tử) cần bộ thuộc tính lọc khác nhau cho mỗi ngành, nhưng mở ngành hàng mới không được phép cần migration + deploy. 5 đặc tính cụ thể bài toán này đòi hỏi — mỗi đặc tính sẽ loại bớt ứng viên ở Phần 4:

| # | Đặc tính | Vì sao đòi hỏi |
|---|---|---|
| P1 | Tập thuộc tính đổi được không cần deploy | Vận hành tự mở ngành hàng mới, không đợi lịch dev |
| P2 | Giá trị dùng lại được, dedup (đổi tên "L"→"L (Rộng)" chỉ 1 chỗ) | Tránh chép lặp chuỗi qua hàng nghìn dòng |
| P3 | Giữ đúng kiểu dữ liệu gốc (số, ngày, boolean), lọc theo khoảng có index riêng | So sánh `>`/`<` trên giá/cân nặng phải nhanh, có B-Tree |
| P4 | Đôi khi trỏ FK thật sang entity khác (kiểu REFERENCE) | Xoá entity bị tham chiếu phải tự CASCADE |
| P5 | Cả Product-level (dùng chung mọi Variant) lẫn Variant-level (riêng từng Variant) | Chất liệu áp dụng cả Product, Size áp dụng riêng từng Variant |

### Giải thích chi tiết từng đặc tính — vì sao nó tồn tại, không chỉ nó LÀ gì

**P1 — Tập thuộc tính đổi được không cần deploy.** Gốc rễ là câu hỏi **ai được quyền quyết định**: mở ngành hàng mới là 1 quyết định KINH DOANH (nắm cơ hội thị trường, mùa vụ, xu hướng tiêu dùng) — nếu quyết định đó phải xếp hàng chờ đúng 1 nguồn lực khan hiếm (lịch đội dev, chu kỳ release), tốc độ phản ứng thị trường của cả công ty bị khoá vào tốc độ kỹ thuật, ngược đúng thứ tự ưu tiên lẽ ra phải có (kỹ thuật phục vụ tốc độ kinh doanh, không phải ngược lại). Nếu bỏ P1 → quay lại đúng bảng Fixed columns (2.1), timeline t=0→t=+5 ngày đã mô phỏng: mỗi ngành hàng mới trễ đúng bằng thời gian đợi migration.

**P2 — Giá trị dùng lại được, dedup.** 2 lý do độc lập, không chỉ 1: (a) đổi tên hiển thị (rebranding, sửa lỗi chính tả, dịch thuật) phải khớp MỌI nơi đang dùng cùng lúc — không cách nào đảm bảo "sửa 1 chỗ, tự khớp 10.000 chỗ khác" nếu giá trị bị chép lặp thay vì tham chiếu; (b) báo cáo/thống kê ("có bao nhiêu sản phẩm Size=M") cần MỌI dòng ghi nhận cùng trỏ về đúng 1 khái niệm canonical — nếu 1 số dòng ghi `"M"`, số khác ghi `"m "` (dư khoảng trắng, sai hoa/thường) vì mỗi lần nhập là gõ tay lại từ đầu, phép đếm gộp sẽ sai mà không ai phát hiện được lỗi ở đâu. Nếu bỏ P2 → đúng con số 10.000 dòng cần UPDATE đã tính ở [2.4](#24-jsonb-cách-a).

**P3 — Giữ đúng kiểu dữ liệu gốc, lọc theo khoảng.** Khách hàng lọc/sort theo **nghĩa thật** của dữ liệu (số, ngày), không theo cách máy tính lưu trữ nó bên trong. "Lọc giá từ 200k-500k" là phép so sánh SỐ, "sắp xếp RAM tăng dần" là phép sort SỐ — nếu hệ thống lưu dạng chuỗi, 2 thao tác này cho kết quả SAI mà không ném ra lỗi nào để báo hiệu (đã chứng minh bằng bảng dữ liệu thật ở [2.5](#25-eav-thuần)). Đây là đặc tính nguy hiểm nhất trong 5 cái, vì hậu quả không gây crash — chỉ âm thầm sai, dễ tồn tại rất lâu trước khi ai nhận ra.

**P4 — Đôi khi trỏ FK thật sang entity khác.** Một số thuộc tính về bản chất KHÔNG phải giá trị mô tả đơn thuần (như "Đỏ", "M") mà là 1 **mối quan hệ** tới entity khác trong chính hệ thống (VD "Sản phẩm đi kèm" trỏ sang 1 `Product` khác, "Trang hướng dẫn sử dụng" trỏ sang 1 `Page`). Khi entity bị tham chiếu đó bị xoá, hệ thống phải biết để tự dọn theo (`CASCADE`) hoặc chặn xoá lại — nếu tham chiếu chỉ là 1 chuỗi ID nằm trong text/document tự do, DB không biết đây là 1 quan hệ thật, không tự dọn được, để lại **tham chiếu treo** (dangling reference) âm thầm, chỉ lộ ra khi có người bấm vào và gặp trang lỗi.

**P5 — Cả Product-level lẫn Variant-level.** Không phải mọi thuộc tính đều có cùng "phạm vi áp dụng" — 1 số mô tả bản chất **sản phẩm** (dùng chung mọi biến thể, VD Chất liệu — áo Đỏ size M hay L đều cùng 100% cotton), 1 số mô tả riêng **từng phiên bản bán được** (khác nhau giữa các Variant, VD Size). Ép chung về 1 phạm vi sẽ sai theo 1 trong 2 hướng: gắn Chất liệu theo Variant → lặp lại y hệt "Cotton" trên mọi Variant của cùng 1 Product (dư thừa, sửa "Cotton"→"Cotton 100%" phải sửa N chỗ thay vì 1 — vi phạm ngược lại P2); hoặc gắn Size theo Product → không còn cách nào biểu diễn Variant nào là M, Variant nào là L, vì cả Product chỉ có đúng 1 giá trị Size chung.

**Hậu quả cụ thể nếu bỏ qua cả 5 đặc tính, tính bằng số — 💭 minh hoạ, không phải số liệu Saleor công bố:** giả sử sàn có 50 ngành hàng đang hoạt động (thời trang, điện tử, sách, thực phẩm, nội thất...), mỗi ngành trung bình cần 6-8 thuộc tính riêng, giao nhau giữa các ngành gần như 0 (RAM không liên quan Size, Hạn dùng không liên quan Tác giả). Nếu hard-code theo Fixed columns: tổng số cột cần trên bảng `Product` ước tính 300-400 cột, mỗi dòng cụ thể chỉ dùng đúng 6-8 cột trong số đó — tức **hơn 95% giá trị 1 dòng bất kỳ là `NULL` vô nghĩa**. Đây không phải cảm giác định tính "nhiều cột trông không đẹp" — là con số cụ thể cho thấy vì sao 5 đặc tính trên không phải yêu cầu thừa thãi, mà là ranh giới thật giữa thiết kế sống được và thiết kế sập ở đúng quy mô sàn thương mại đa ngành hàng.

---

## Phần 2 — 6 ứng viên, đủ code từng kỹ thuật

> **Cùng 1 kịch bản dữ liệu xuyên suốt cả 6 mục** — 2 sản phẩm: "Áo thun đỏ" (Size=M, Color=Đỏ) và "Tai nghe Bluetooth" (RAM=8GB) — để 6 bảng có thể đặt cạnh nhau, so sánh ngang hàng đúng 1 câu hỏi: **"cùng 2 sản phẩm này, dưới kỹ thuật này, dữ liệu nằm ở đâu, hình dạng ra sao?"** Đây là ảnh chụp TĨNH, khác 2 phần dùng data table còn lại trong file: Phần 5 cho data tại đúng khoảnh khắc 1 LỐI TẮT THẤT BẠI (mỗi mục 1 tình huống riêng, không dùng chung 1 kịch bản), Phần 6 cho data THẬT của Saleor tại từng QUYẾT ĐỊNH cụ thể (đối chiếu với đường không đi). Phần 2 không kể chuyện gì xảy ra SAU — chỉ trả lời "trông thế nào ngay lúc này".

### 2.1 Fixed columns

**Giới thiệu:** không có tên riêng trong tài liệu pattern — đây là điểm MẶC ĐỊNH của thiết kế quan hệ chuẩn (3NF), không phải 1 kỹ thuật cần học riêng. Mỗi thuộc tính nghiệp vụ ánh xạ thẳng thành 1 cột cố định trên bảng. Là baseline để so sánh — mọi kỹ thuật khác trong Phần 2 đều tồn tại vì Fixed columns không đủ ở 1 điều kiện cụ thể nào đó.

```python
# 💭 V1 — cột cứng trên Product, KHÔNG có discriminator nào cả (giả định ngầm: chỉ bán 1 ngành hàng mãi mãi)
class Product(models.Model):
    name = models.CharField(max_length=250)
    size = models.CharField(choices=[("S", "S"), ("M", "M"), ("L", "L")], null=True)
    color = models.CharField(max_length=50, null=True)
    # mở "điện tử" 6 tháng sau → cần ram, storage... size/color để trống vô nghĩa
```

**Dữ liệu — bảng `Product` chỉ có cột `size`/`color`, KHÔNG có `ram`:**

| id | name | size | color |
|---|---|---|---|
| 1 | Áo thun đỏ | M | Đỏ |
| 2 | Tai nghe Bluetooth | `NULL` | `NULL` |

Dòng `id=2` lưu được TÊN sản phẩm, nhưng RAM=8GB **không có chỗ nào trên bảng để lưu** — không phải `NULL` do chưa nhập, mà `NULL` do không tồn tại cột nào cho giá trị đó. Đây là ảnh chụp tĩnh, đúng gốc rễ của "Vấn đề kỹ thuật cụ thể" mô tả ngay dưới.

**Vấn đề kỹ thuật cụ thể — mô phỏng theo mốc thời gian (💭):** shop đang bán quần áo, `Product` có 6 cột (`name`, `size`, `color`, `material`, `fit`, `season`). Founder quyết định mở ngành "Điện tử" — đây là chuỗi sự kiện thật sẽ xảy ra, không phải giả định trừu tượng:
- **t=0** — Vận hành muốn nhập sản phẩm điện tử đầu tiên (VD "Tai nghe Bluetooth"), mở màn hình nhập liệu → không có ô nào để nhập `ram`/`bluetooth_version`, vì cột đó chưa tồn tại.
- **t=+1 ngày** — Vận hành báo dev, dev viết migration `ALTER TABLE product ADD COLUMN ram VARCHAR(20) NULL, ADD COLUMN bluetooth_version VARCHAR(10) NULL`.
- **t=+3 ngày** — Migration được review, gộp vào release kế tiếp (không release riêng vì rủi ro).
- **t=+5 ngày** — Deploy xong, vận hành mới nhập được sản phẩm điện tử đầu tiên — **5 ngày trễ so với ý định ban đầu**, đúng bằng thời gian phụ thuộc lịch dev mà bài toán gốc (P1) yêu cầu KHÔNG được có.
- Sau khi có Điện tử: mọi dòng `Product` thuộc ngành Quần áo giờ có thêm 2 cột `ram`/`bluetooth_version` luôn `NULL` — và ngược lại, dòng Điện tử có `size`/`color`/`material`/`fit`/`season` luôn `NULL`.

**Sai lầm thường gặp khi dùng Fixed columns:** áp đúng ở quy mô 1 ngành hàng cố định (query `WHERE size = 'M'` nhanh, có index thẳng, dễ validate bằng `CHECK` constraint) — sai lầm không nằm ở việc CHỌN kỹ thuật này, mà ở việc chọn nó **khi đã biết trước** sẽ mở ngành hàng thứ 2 không chia sẻ thuộc tính (đúng P1). Dấu hiệu nhận biết: schema không có bất kỳ cột `type`/`category` nào để phân biệt — ngay cả khái niệm "ngành hàng" cũng chưa tồn tại trong model.

Có 1 biến thể sai lầm còn phổ biến hơn: đội dev **có** nhận ra sẽ mở nhiều ngành hàng, nhưng chọn cách "phòng thủ" bằng cách **thêm sẵn cột cho mọi ngành hàng dự đoán trước**, để khỏi phải chờ migration mỗi lần mở ngành mới. Nghe hợp lý (né được timeline t=0→t=+5 ngày đã mô phỏng ở trên), nhưng chỉ dời chi phí sang chỗ khác, không xoá được nó — vì mỗi ngành hàng mới vẫn cần đúng 1 lần "đoán trước + thêm cột" đó, chỉ là làm sớm hơn thay vì làm khi cần.

**Minh hoạ bằng bảng dữ liệu — số cột và tỉ lệ `NULL` phình theo từng đợt "thêm cột trước cho chắc" (💭, số liệu ăn khớp với ước tính 300-400 cột / >95% NULL đã nêu ở cuối Phần 1, viết lại dạng bảng theo mốc thời gian để thấy tốc độ phình):**

| Thời điểm | Ngành hàng đã mở | Tổng số cột thuộc tính trên `Product` | Số cột `NULL` trên 1 dòng "Áo thun" bất kỳ |
|---|---|---|---|
| Tháng 1 | Quần áo | 4 (`size`, `color`, `material`, `fit`) | 0/4 (0%) |
| Tháng 4 | + Điện tử | 8 (+ `ram`, `storage`, `bluetooth_version`, `warranty_months`) | 4/8 (50%) |
| Tháng 9 | + Sách | 11 (+ `author`, `publisher`, `isbn`) | 7/11 (64%) |
| Tháng 12 | + Nội thất | 14 (+ `chat_lieu`, `kich_thuoc`, `xuat_xu`) | 10/14 (71%) |
| Năm 3 (50 ngành, theo giả định Phần 1) | + 46 ngành khác | 300-400 | > 380/400 (> 95%) |

Đọc theo cột cuối: dòng "Áo thun" ở Tháng 1 (0% `NULL`) và dòng "Áo thun" ở Năm 3 (>95% `NULL`) là **cùng 1 dòng dữ liệu, không đổi nội dung** — chỉ khác số cột vô nghĩa mọc thêm xung quanh nó vì các ngành hàng KHÁC được thêm vào. Đây chính là lý do "thêm cột trước cho chắc" không phải giải pháp, chỉ là hoãn: mỗi ngành hàng mới vẫn cần y hệt 1 migration (chỉ khác là dev tự dự đoán và làm sớm thay vì vận hành yêu cầu và làm muộn) — P1 (đổi được không cần deploy) vẫn KHÔNG được giải, trong khi cái giá "cột NULL lãng phí" (vốn là đánh đổi chấp nhận được ở 2.6, xem Phần 7) giờ lại xảy ra ở đúng nơi không đáng — bảng `Product` cốt lõi, không phải bảng giá trị phụ trợ.

### 2.2 STI thuần (Single Table Inheritance)

**Giới thiệu:** nguồn — Martin Fowler, *Patterns of Enterprise Application Architecture* (2002), chương "Object-Relational Structural Patterns". Ý tưởng: khi có nhiều LOẠI CON của 1 class cha (VD `ClothingProduct`/`ElectronicsProduct` cùng là `Product`), gộp hết vào **đúng 1 bảng**, thêm 1 cột discriminator (VD `product_type`) để phân biệt loại, các cột riêng của từng loại con cùng tồn tại trên bảng đó — phần lớn `NULL` với dòng không thuộc loại đó. Rails ActiveRecord biến STI thành tính năng có sẵn (quy ước cột `type`), khiến tên gọi này phổ biến trong giới Rails/Django dù ý tưởng gốc đến từ sách Fowler.

```python
# 💭 khác V1 ở đúng 1 điểm: có discriminator column (product_type) — nhưng cấu trúc cột vẫn cố định
class Product(models.Model):
    name = models.CharField(max_length=250)
    product_type = models.CharField(       # discriminator — phân biệt Ý NGHĨA của các cột bên dưới
        choices=[("ao-thun", "Áo thun"), ("dien-tu", "Điện tử"), ("sach", "Sách")]
    )
    size = models.CharField(null=True, blank=True)       # chỉ có nghĩa khi product_type="ao-thun"
    color = models.CharField(null=True, blank=True)
    ram = models.CharField(null=True, blank=True)          # chỉ có nghĩa khi product_type="dien-tu"
    storage = models.CharField(null=True, blank=True)
    author = models.CharField(null=True, blank=True)       # chỉ có nghĩa khi product_type="sach"
    publisher = models.CharField(null=True, blank=True)
```

**Dữ liệu — cùng 2 sản phẩm, giờ có cột `product_type` + cột `ram` (giả sử dev đã đoán trước, thêm sẵn):**

| id | name | product_type | size | color | ram |
|---|---|---|---|---|---|
| 1 | Áo thun đỏ | `ao-thun` | M | Đỏ | `NULL` |
| 2 | Tai nghe Bluetooth | `dien-tu` | `NULL` | `NULL` | 8GB |

Khác 2.1: dòng `id=2` GIỜ lưu được RAM (vì cột đã tồn tại sẵn) — nhưng đổi lại dòng `id=1` có thêm 1 cột `ram` luôn `NULL` vô nghĩa, và dòng `id=2` có `size`/`color` luôn `NULL` vô nghĩa. Không mất dữ liệu như 2.1, nhưng số cột `NULL` mỗi dòng tăng theo số ngành hàng đã "đoán trước" — và vẫn phải ĐOÁN TRƯỚC (dev thêm cột `ram` trước khi biết chắc sẽ cần) mới có bảng này, khác thiết kế thật ở 2.6 (không cần đoán trước).

**Vấn đề kỹ thuật cụ thể (💭):** lặp lại đúng timeline t=0→t=+5 ngày ở mục 2.1 — chỉ khác 1 chi tiết: lúc dev viết migration, có thêm bước "cập nhật `choices=` của `product_type`" (dễ quên, vì `choices` chỉ validate ở tầng Django, không phải DB — thêm giá trị `"noi-that"` mà quên update `choices` vẫn lưu được vào DB, chỉ form admin không hiện lựa chọn mới cho tới khi deploy code có `choices` cập nhật). Tức là STI thuần thêm đúng 1 lớp rủi ro mới (đồng bộ `choices` với cột thật) mà Fixed columns không có, trong khi bài toán cốt lõi (P1 — cần deploy) vẫn y hệt không được giải.

**Sai lầm thường gặp khi dùng STI thuần — nhầm "có discriminator" với "đã linh hoạt":** thêm cột `product_type` tạo cảm giác hệ thống đã "biết phân loại", nhưng mở ngành hàng thứ 4 (VD "Nội thất") vẫn cần thêm cột `chat_lieu_go`, `kich_thuoc` — **vẫn migration + deploy y hệt Fixed columns**. `product_type` chỉ giải bài toán "phân biệt Ý NGHĨA của cột đã tồn tại sẵn", không giải bài toán "thêm THUỘC TÍNH MỚI không cần deploy" (P1) — 2 bài toán khác nhau, dễ nhầm làm 1 vì nhìn qua đã thấy "có vẻ linh hoạt hơn V1".

Đây đúng là sai lầm phổ biến nhất trong cả 5 ứng viên bị loại — đủ phổ biến để file này dành hẳn 1 mục luyện tập riêng, kèm bảng dữ liệu trace tay từng bước "lách" thất bại ra sao (dòng `product_type="noi-that"` tạo được nhưng toàn bộ cột thuộc tính rỗng): xem [Phần 5, Lối tắt A](#phần-5-4-lối-tắt-nghe-hợp-lý-nhưng-sai-tự-đoán-trước-khi-đọc), không lặp lại bảng ở đây.

### 2.3 CTI (Class Table Inheritance)

**Giới thiệu:** cùng nguồn Fowler, cùng chương với STI — nhưng đánh đổi theo hướng ngược lại: mỗi loại con có **1 bảng riêng**, chỉ chứa cột đặc thù của loại đó, nối với bảng cha bằng khoá ngoại 1-1 (quan hệ "is-a" tường minh ở tầng schema, không chỉ ở tầng code). Django gọi đây là "multi-table inheritance". Đổi lấy việc không còn cột `NULL` nào, nhưng trả giá bằng JOIN mỗi khi cần dữ liệu trộn nhiều loại con cùng lúc — đúng đánh đổi sẽ thấy rõ ở phần dữ liệu ngay dưới. (Fowler còn 1 pattern thứ 3 cùng họ — Concrete Table Inheritance, không đưa vào Phần 2 vì còn tệ hơn CTI cho đúng bài toán này, xem lý do ở phần trao đổi trước.)

```python
# 💭 Django multi-table inheritance — mỗi loại con 1 bảng riêng, FK ẩn (product_ptr) về bảng cha
class Product(models.Model):
    name = models.CharField(max_length=250)
    product_type = models.CharField(choices=[...])   # vẫn cần giữ để query nhanh không cần JOIN mọi bảng con

class ClothingProduct(Product):        # Django tự tạo product_ptr = OneToOneField(Product, parent_link=True)
    size = models.CharField(max_length=10)
    color = models.CharField(max_length=50)

class ElectronicsProduct(Product):
    ram = models.CharField(max_length=20)
    storage = models.CharField(max_length=20)
```

**Dữ liệu — cùng 2 sản phẩm, giờ nằm trên 3 bảng vật lý riêng biệt:**

`Product` (bảng gốc):

| id | name | product_type |
|---|---|---|
| 1 | Áo thun đỏ | `ao-thun` |
| 2 | Tai nghe Bluetooth | `dien-tu` |

`ClothingProduct`:

| product_ptr_id | size | color |
|---|---|---|
| 1 | M | Đỏ |

`ElectronicsProduct`:

| product_ptr_id | ram |
|---|---|
| 2 | 8GB |

Khác 2.2: không còn cột `NULL` nào (mỗi bảng chỉ chứa đúng cột của loại đó) — nhưng đổi lại "Tai nghe Bluetooth" giờ **nằm rải trên 2 bảng** (`Product` id=2 + `ElectronicsProduct` product_ptr_id=2), không phải 1 dòng duy nhất như 2.1/2.2. Muốn có đủ thông tin hiển thị 1 sản phẩm phải JOIN đúng cặp bảng — đây là ảnh chụp tĩnh cho đúng chi phí sẽ mô tả ngay dưới.

**Vấn đề kỹ thuật cụ thể — query storefront thật trông thế nào (💭):** storefront cần trang "Tất cả sản phẩm", phân trang 20 sản phẩm/trang, hiển thị đủ thuộc tính chính mỗi sản phẩm ngay trên card (không phải click vào mới thấy). Với CTI, không viết được 1 câu `Product.objects.all()[:20]` là đủ — vì bảng `Product` gốc không có `size`/`ram`, phải:
1. Query `Product.objects.all().order_by("-created_at")[:20]` lấy 20 dòng gốc, biết `product_type` mỗi dòng
2. Với mỗi dòng, dựa `product_type` để biết JOIN bảng `ClothingProduct` hay `ElectronicsProduct` — 20 dòng có thể thuộc N ngành khác nhau → **N query JOIN riêng biệt**, hoặc phải viết `UNION ALL` thủ công giữa các bảng con rồi tự map lại kiểu dữ liệu ở code (Django ORM không tự làm việc này qua MTI 1 cách hiệu quả cho trang phân trang trộn nhiều loại)
3. Thêm 1 ngành hàng mới = thêm 1 điểm cần sửa trong đúng đoạn dispatch này — không chỉ thêm bảng, còn phải nhớ sửa code liệt kê catalog ở bước 2

So sánh: EAV+STI hybrid (2.6) trả lời đúng câu này bằng 1 `prefetch_related` duy nhất bất kể có bao nhiêu loại ProductType, không cần dispatch theo loại ở tầng code.

**Sai lầm thường gặp khi dùng CTI — chọn vì "đúng chuẩn OOP" mà quên chi phí query trộn loại:** `ClothingProduct(size="M")` và `ElectronicsProduct(ram="8GB")` là 2 bảng vật lý riêng biệt. Mở ngành hàng mới vẫn là thêm 1 bảng = 1 migration, không giải được P1. Dấu hiệu nhận biết: mọi tính năng "browse catalog chung" đều cần vòng lặp/dispatch theo `product_type` trước khi biết JOIN bảng con nào.

Bảng dữ liệu trace tay đầy đủ (3 bảng vật lý tách biệt cho 3 ngành hàng + đếm số query thật cho 1 trang catalog 20 sản phẩm trộn ngành) đã có ở [Phần 5, Lối tắt B](#phần-5-4-lối-tắt-nghe-hợp-lý-nhưng-sai-tự-đoán-trước-khi-đọc) — không lặp lại ở đây, chỉ nhắc lại kết luận: số query CTI **tăng tuyến tính theo số ngành hàng xuất hiện trên cùng 1 trang**, đúng nghịch lý "càng nhiều ngành hàng càng chậm" dù đa dạng ngành hàng chính là mục tiêu ban đầu của bài toán.

### 2.4 JSONB (Cách A)

**Giới thiệu:** không phải pattern OOP như 2 mục trên — đây là **tính năng của chính database engine** (Postgres, phổ biến từ ~2012; các RDBMS lớn khác cũng có kiểu tương đương: MySQL `JSON`, SQL Server `JSON`), thuộc trào lưu ảnh hưởng từ NoSQL. Ý tưởng: lưu 1 cột có cấu trúc **tự do** (document), không khai báo cột/kiểu trước — đổi "ràng buộc lúc GHI" (schema-on-write, kiểu quan hệ truyền thống mà 2.1-2.3 đều thuộc về) lấy "ràng buộc lúc ĐỌC" (schema-on-read).

```python
# 💭 thoả P1 (mỗi category tự do định nghĩa key riêng, không migration) nhưng thiếu P2/P3/P4
class Product(models.Model):
    name = models.CharField(max_length=250)
    category = models.CharField(max_length=100)
    attributes = models.JSONField(default=dict)   # {"size": "M", "color": "Red"} hoặc {"ram": "8GB"}
```

**Dữ liệu — cùng 2 sản phẩm, giờ gộp lại đúng 1 bảng, không cột nào `NULL`:**

| id | name | category | attributes (JSONB) |
|---|---|---|---|
| 1 | Áo thun đỏ | `ao-thun` | `{"size": "M", "color": "Đỏ"}` |
| 2 | Tai nghe Bluetooth | `dien-tu` | `{"ram": "8GB"}` |

Nhìn thoáng qua đây là bảng "đẹp nhất" trong 3 bảng đã thấy (2.1-2.3) — không `NULL`, không tách bảng. Nhưng chuỗi `"Đỏ"` ở dòng `id=1` là **bản copy chép tay trong chính document này**, không trỏ tới đâu cả — nếu có sản phẩm thứ 2 cũng màu Đỏ, `"Đỏ"` sẽ xuất hiện y hệt LẦN NỮA trong document của dòng đó, không dùng lại được dòng đã có. So sánh trực tiếp với 2.5 ngay dưới để thấy khác biệt.

**Vấn đề kỹ thuật cụ thể — con số cho P2 (💭):** giả sử shop có 10.000 sản phẩm màu Đỏ (chuỗi `"Red"` xuất hiện trong 10.000 document JSONB khác nhau, mỗi document 1 bản copy độc lập của chuỗi này). Marketing quyết định đổi cách hiển thị từ "Red" sang "Đỏ đô" cho thân thiện hơn với khách Việt:
- Với JSONB: phải chạy `UPDATE product SET attributes = jsonb_set(attributes, '{color}', '"Đỏ đô"') WHERE attributes->>'color' = 'Red'` — quét và ghi lại **10.000 dòng**, tốn 1 lượt full table scan + rewrite (Postgres không update in-place cho JSONB, mỗi UPDATE tạo dòng mới ở tầng MVCC), có thể khoá bảng nếu chạy giữa giờ cao điểm.
- Với EAV+STI hybrid (2.6): chỉ `UPDATE attribute_value SET name = 'Đỏ đô' WHERE id = <id của giá trị Red>` — **đúng 1 dòng**, vì 10.000 sản phẩm chỉ tham chiếu (FK) tới cùng 1 dòng `AttributeValue`, không copy chuỗi.

**Sai lầm thường gặp khi dùng JSONB cho bài toán này:** tempting vì đúng 3 chữ "linh hoạt, Postgres hỗ trợ tốt, không cần tách bảng" — nhưng 3 lỗ hổng cụ thể (đã grounded so sánh trực tiếp với thiết kế thật ở [1b, mục "So sánh 2 cách hiện thực"](./ecommerce-setup-1b-product-attribute.md)):
- P2 vỡ: đúng ví dụ số liệu ở trên — 1 dòng cập nhật (EAV) so với 10.000 dòng (JSONB)
- P3 vỡ: so sánh khoảng (`>`, `<`) trên field lồng trong jsonb phải ép kiểu, không có B-Tree riêng cho từng field
- P4 vỡ: kiểu REFERENCE không có FK thật trong jsonb — xoá entity bị tham chiếu không tự CASCADE

### 2.5 EAV thuần (Entity-Attribute-Value)

**Giới thiệu:** nguồn gốc từ hệ thống dữ liệu lâm sàng/y tế thập niên 1990 (hồ sơ bệnh án có hàng nghìn loại chỉ số khả dĩ, mỗi bệnh nhân chỉ dùng đúng vài chục) — nơi bài toán "thuộc tính cực kỳ thưa, không đoán trước hết được" xuất hiện sớm và rõ nhất. Ý tưởng cốt lõi: thay vì thuộc tính là **CỘT** (như 2.1-2.3), biến nó thành **DÒNG DỮ LIỆU** trong 1 bảng riêng — 1 dòng trả lời "Entity nào - Attribute nào - Value nào". Bill Karwin, *SQL Antipatterns* (2010), xếp hẳn 1 chương cho EAV dưới tên "antipattern" — tín hiệu rằng đây là kỹ thuật đúng chỗ hẹp (dữ liệu cực thưa) nhưng rất dễ bị lạm dụng ở nơi không thực sự cần tới mức đó.

```python
# 💭 thoả P1, P2 (bảng giá trị dùng chung, dedup được) nhưng CHƯA giữ kiểu gốc (P3) — 1 cột value chung mọi kiểu
class Attribute(models.Model):
    slug = models.SlugField(unique=True)          # "size", "ram"
    name = models.CharField(max_length=100)

class AttributeValue(models.Model):
    attribute = models.ForeignKey(Attribute, on_delete=models.CASCADE)
    value = models.TextField()                     # "M" | "8GB" | "true" | "2026-01-01" — MỌI kiểu ép về chuỗi

class ProductAttributeValue(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    value = models.ForeignKey(AttributeValue, on_delete=models.CASCADE)
```

**Dữ liệu — cùng 2 sản phẩm, giờ tách thành 3 bảng nhỏ, mỗi giá trị chỉ tồn tại đúng 1 dòng:**

`Attribute`:

| id | slug |
|---|---|
| 1 | `size` |
| 2 | `color` |
| 3 | `ram` |

`AttributeValue` (cột `value` là `TEXT` chung cho mọi kiểu):

| id | attribute_id | value |
|---|---|---|
| 10 | 1 | `"M"` |
| 11 | 2 | `"Đỏ"` |
| 12 | 3 | `"8GB"` |

`ProductAttributeValue` (bảng nối):

| id | product_id | value_id |
|---|---|---|
| 1 | 1 (Áo thun đỏ) | 10 (M) |
| 2 | 1 (Áo thun đỏ) | 11 (Đỏ) |
| 3 | 2 (Tai nghe Bluetooth) | 12 (8GB) |

Khác 2.4: `"Đỏ"` giờ là **1 dòng canonical duy nhất** (`id=11`) — nếu có sản phẩm thứ 2 cũng màu Đỏ, chỉ cần thêm 1 dòng `ProductAttributeValue` mới trỏ `value_id=11`, không tạo lại chuỗi "Đỏ" lần nữa (đúng ưu điểm dedup P2 đã nói ở Phần 3). Nhưng nhìn cột `value` của `AttributeValue`: `"M"`, `"Đỏ"`, `"8GB"` đều là `TEXT` như nhau — không có cách nào phân biệt "8GB" cần so sánh theo SỐ, còn "M"/"Đỏ" chỉ cần so khớp chuỗi. Đây là ảnh chụp tĩnh cho đúng bug sẽ mô tả ngay dưới.

**Vấn đề kỹ thuật cụ thể — bug sort thật trông thế nào (💭):** shop bán RAM điện tử với 3 giá trị: "4GB", "8GB", "16GB" — cột `value` lưu dạng `TEXT`, giả sử tách riêng phần số để sort (`"4"`, `"8"`, `"16"`). Khách hàng bấm "sắp xếp theo dung lượng tăng dần" trên storefront:
- Sort đúng (theo số): 4 < 8 < 16
- Sort thật xảy ra (`ORDER BY value` trên cột TEXT, so sánh từng ký tự): `"16"` < `"4"` < `"8"` — vì ký tự đầu `'1'` < `'4'` < `'8'` theo bảng mã ASCII, Postgres không biết đây là số
- Khách hàng thấy "RAM 16GB" xếp ĐẦU danh sách "tăng dần", ngay trước "RAM 4GB" — sai hiển nhiên với bất kỳ ai nhìn vào, nhưng **không có exception nào được ném ra**, hệ thống chạy "bình thường", chỉ kết quả sai

Đây chính là lý do lỗi này khó bị test phát hiện: test với đúng 2 giá trị ("4GB", "8GB") không lộ ra vấn đề (`"4" < "8"` đúng cả 2 cách so sánh) — chỉ lộ khi có ≥ 3 giá trị và ít nhất 1 cặp có độ dài chữ số khác nhau (VD "9" và "10").

**Sai lầm thường gặp khi dùng EAV thuần:** nhìn bảng so sánh (Phần 3) thấy cột "Linh hoạt" và "Dedup" đều ✅ cao nhất trong các ứng viên không phải hybrid, dễ kết luận vội "vậy dùng EAV thuần luôn cho an toàn". Sai lầm nằm ở validate kiểu — không phải lỗi thiết kế cẩu thả, mà là **hệ quả tất yếu của việc "thuần"** (chỉ 1 cột `value`).

### 2.6 EAV+STI hybrid — Saleor thật

**Giới thiệu:** khác 5 mục trên — **không phải pattern có tên trong sách nào cả**. Đây là kết quả GHÉP đúng 2 ý tưởng đã giới thiệu ở 2.2 (STI) và 2.5 (EAV), áp mỗi ý tưởng cho đúng 1 tầng khác nhau của cùng bài toán: EAV giải "ai dùng thuộc tính gì" (tầng gán), STI-like giải "giá trị đó kiểu gì" (tầng lưu giá trị, không phải tầng `Product` như STI gốc). Không tìm được kiểu ghép này bằng cách đọc sách — chỉ tìm được bằng cách đọc 1 hệ thống thật đã giải đúng bài toán này, đúng như đã trao đổi ở câu hỏi trước ("nghiên cứu hệ thống thật học được nhiều hơn sách").

```python
# ✅ Tầng gán — EAV, attribute/models/base.py, attribute/models/product_variant.py
class Attribute(ModelWithMetadata, ModelWithExternalReference):
    input_type = models.CharField(choices=AttributeInputType.CHOICES, ...)   # discriminator

# ✅ Tầng giá trị — STI-like, attribute/models/base.py:365-421
class AttributeValue(ModelWithExternalReference):
    slug = models.SlugField(max_length=255, allow_unicode=True)
    value = models.CharField(max_length=255, blank=True, default="")   # DROPDOWN/MULTISELECT — mã màu hex
    rich_text = SanitizedJSONField(blank=True, null=True, ...)          # RICH_TEXT — JSONB thật
    plain_text = models.TextField(blank=True, null=True)                # PLAIN_TEXT
    boolean = models.BooleanField(blank=True, null=True)                # BOOLEAN
    date_time = models.DateTimeField(blank=True, null=True)             # DATE/DATE_TIME
    numeric = models.FloatField(null=True, blank=True)                  # NUMERIC — có index B-Tree riêng
    reference_product = models.ForeignKey(Product, null=True, ...)      # REFERENCE — FK thật, CASCADE thật
    reference_variant = models.ForeignKey(ProductVariant, null=True, ...)
    reference_collection = models.ForeignKey(Collection, null=True, ...)
    reference_category = models.ForeignKey(Category, null=True, ...)
    reference_page = models.ForeignKey(Page, null=True, ...)
```

1 dòng `AttributeValue` chỉ đúng 1 trong ~11 cột được điền — khớp `input_type` của `Attribute` cha, mọi cột còn lại luôn `NULL`. Đây đúng hình dạng STI kinh điển (1 bảng, discriminator, cột dùng chung, phần lớn `NULL` với từng dòng cụ thể) — chỉ khác STI "bình thường" ở chỗ discriminator (`input_type`) nằm ở bảng **cha** (`Attribute`) chứ không nằm ngay trên chính bảng này. Chi tiết đầy đủ (sơ đồ ER, ví dụ dữ liệu thật dạng bảng, cơ chế `on_delete=CASCADE`) đã có ở [1b](./ecommerce-setup-1b-product-attribute.md), không lặp lại ở đây.

**Dữ liệu — cùng 2 sản phẩm, hoàn tất bức tranh 6 bảng để so sánh ngang hàng:**

`Attribute` (giờ có thêm `input_type` — quyết định cột nào của `AttributeValue` sẽ được dùng):

| id | slug | input_type |
|---|---|---|
| 1 | `size` | DROPDOWN |
| 2 | `color` | DROPDOWN |
| 3 | `ram` | NUMERIC |

`AttributeValue` (giờ có nhiều cột kiểu riêng, không chỉ 1 `value` chung):

| id | attribute_id | value | numeric |
|---|---|---|---|
| 10 | 1 | `"M"` | `NULL` |
| 11 | 2 | `"Đỏ"` | `NULL` |
| 12 | 3 | `""` | `8` |

So với 2.5: dòng `id=10`/`id=11` vẫn dedup được y hệt EAV thuần (thừa hưởng ưu điểm P2) — khác biệt duy nhất nằm ở dòng `id=12`: RAM=8GB giờ có cột `numeric=8` riêng, không còn nằm trong cột `value` dạng chuỗi `"8GB"` nữa. Đúng 1 cột thêm vào là đủ khoảng cách giữa 2.5 và 2.6 — không phải 2 kiến trúc khác hẳn nhau, mà 2.6 = 2.5 + đúng phần "giữ kiểu gốc" mà 2.5 còn thiếu (khớp lại với Phần 6, mục 6.5).

**Bảng tổng kết cả 6 mục — cùng câu hỏi "Đỏ" và "8GB" nằm ở đâu:**

| Kỹ thuật | "Đỏ" nằm ở đâu | "8GB" nằm ở đâu, kiểu gì |
|---|---|---|
| 2.1 Fixed columns | Cột `color` trên `Product` | Không có chỗ lưu |
| 2.2 STI thuần | Cột `color` trên `Product` (thêm `NULL` ở dòng khác ngành) | Cột `ram` (`TEXT`), phải đoán trước khi thêm cột |
| 2.3 CTI | Cột `color` trên bảng con `ClothingProduct` | Cột `ram` (`TEXT`) trên bảng con `ElectronicsProduct` |
| 2.4 JSONB | Chuỗi lồng trong `attributes` JSONB, copy riêng mỗi document | Chuỗi lồng trong JSONB, không so sánh số được |
| 2.5 EAV thuần | 1 dòng canonical trong `AttributeValue.value` (`TEXT`) | 1 dòng canonical, nhưng vẫn `TEXT` — sort sai |
| 2.6 Hybrid (thật) | 1 dòng canonical trong `AttributeValue.value` | 1 dòng canonical, cột `numeric` riêng — sort đúng |

**Cùng 2 vấn đề kỹ thuật cụ thể ở JSONB (2.4) và EAV thuần (2.5) — hybrid giải ra sao:**
- Đổi tên "Red" → "Đỏ đô" (P2): `UPDATE attribute_value SET name = 'Đỏ đô' WHERE id = <id>` — 1 dòng, giống EAV thuần (thừa hưởng đúng ưu điểm dedup của EAV, không mất khi ghép thêm STI).
- Sort RAM 4GB/8GB/16GB đúng thứ tự số (P3): giá trị RAM có `input_type=NUMERIC` → lưu ở cột `numeric = FloatField()`, `ORDER BY numeric` so sánh đúng kiểu số ngay trong Postgres, không có bug `"16" < "4"` đã thấy ở 2.5 — vì không còn ép về TEXT nữa, đây chính là điểm STI-like sửa được mà EAV thuần không sửa được.

---

## Phần 3 — Bảng so sánh trực diện

| Kỹ thuật | Query speed | Validate kiểu | Linh hoạt (P1) | Dedup (P2) | Kiểu gốc + lọc khoảng (P3) | Khi nào đúng |
|---|---|---|---|---|---|---|
| **Fixed columns** (2.1) | Nhanh nhất | Chặt nhất (CHECK constraint) | ✗ — đổi cần migration | — (không cần, giá trị là scalar) | ✅ | Đúng 1 ngành hàng cố định mãi mãi |
| **STI thuần** (2.2) | Nhanh | Trung bình | ✗ — thêm loại mới vẫn cần thêm cột = migration | — | ✅ | Ít loại con, biết trước, tổng thuộc tính không quá lớn |
| **CTI** (2.3) | Nhanh nếu biết type trước, chậm khi cần liệt kê trộn mọi loại | Chặt | ✗ — thêm loại mới = thêm bảng = migration | — | ✅ | Biết trước các loại, mỗi loại thuộc tính khác biệt LỚN, hiếm khi query trộn |
| **JSONB** (2.4) | Khá nhanh (GIN index) | Yếu-trung bình | ✅ | ✗ — "Đỏ" lặp lại trong document mọi SP đỏ | ✗ — so sánh khoảng phải ép kiểu, không B-Tree riêng | Cần cân bằng linh hoạt + hiệu năng, không cần dedup/lọc khoảng nghiêm ngặt |
| **EAV thuần** (2.5) | Chậm nhất (nhiều JOIN) | **Yếu** — ép mọi kiểu về chuỗi | ✅ | ✅ | ✗ — `"10" < "9"` theo thứ tự chuỗi | Filter/search phức tạp cần đẩy ra search engine riêng bù (Elasticsearch...) |
| **EAV+STI hybrid** (2.6, Saleor thật) | Trung bình (3-4 JOIN, nhưng mỗi JOIN có index đúng chỗ) | Chặt (mỗi kiểu 1 cột riêng, `FloatField`/`DateTimeField`/FK thật) | ✅ | ✅ | ✅ | Đủ cả P1-P5 cùng lúc — đúng bài toán Saleor |

---

## Phần 4 — Vì sao 5 ứng viên còn lại bị loại — áp đúng P1-P5

| Ứng viên | Đặc tính nào loại nó | Ghi chú |
|---|---|---|
| Fixed columns (2.1) | P1 — mở ngành "Điện tử" cần thêm `ram`/`storage`, Size/Color để trống vô nghĩa với ngành đó | — |
| STI thuần (2.2) | P1 — dù có discriminator, thêm 1 *loại thuộc tính mới* vẫn là thêm 1 cột mới trên bảng chung → migration | Về bản chất là Fixed columns mở rộng, chỉ khác cách đặt tên |
| CTI (2.3) | P1 (mở ngành = thêm bảng con = migration) + hiệu năng khi liệt kê trộn (storefront cần hiển thị TẤT CẢ thuộc tính của 1 sản phẩm bất kể ngành, không biết trước sẽ JOIN bảng con nào) | — |
| JSONB (2.4) | P2 (không dedup) + P3 (không B-Tree riêng cho numeric) + P4 (REFERENCE không có FK thật, không tự CASCADE) | ✅ đã grounded chi tiết ở [1b](./ecommerce-setup-1b-product-attribute.md) |
| EAV thuần (2.5) | P3 — mọi kiểu ép về TEXT, sort số sai (`"10" < "9"`), không tận dụng được index kiểu riêng | 🧠 Saleor tự gọi tên đúng pattern này khi giải thích lý do KHÔNG làm vậy (xem 1b) |

(Fixed columns và STI thuần thực ra chung 1 lý do bị loại — cả 2 đều mã hoá tập thuộc tính vào **cột của schema**, khác EAV+STI hybrid mã hoá vào **dòng dữ liệu**. Sự khác biệt CTI với 2 cái trên chỉ là mức độ tách bảng, không đổi bản chất "vẫn migration khi mở ngành mới".)

---

## Phần 5 — 4 lối tắt "nghe hợp lý nhưng sai" (tự đoán trước khi đọc)

Đọc code ở Phần 2 dễ tưởng bảng so sánh Phần 3 đã đủ để quyết định — thực tế phân biệt đúng/sai còn khó hơn thế, vì cả 4 lối tắt dưới đây đều nghe **rất hợp lý** lúc mới cân nhắc. 3 lối tắt đầu (A/B/C) đều là dừng lại QUÁ SỚM — chưa leo thang đủ dù bằng chứng đã có; lối tắt cuối (D) đi ngược chiều — leo thang QUÁ XA khi không có bằng chứng. Tự đoán câu in nghiêng trước khi đọc "Vì sao sai".

**Lối tắt A — "STI thuần đã có cột `product_type` phân loại rõ ràng, vậy coi như đủ linh hoạt cho P1 chưa?"**

*Tự đoán trước: câu hỏi cụ thể nào sẽ lộ ra chưa đủ?*

**Vì sao sai:** hỏi thẳng "mở ngành hàng thứ 4 (VD Nội thất) có cần sửa code không?" — có, vẫn cần thêm cột `chat_lieu_go` mới migrate được. `product_type` chỉ giải bài toán "phân biệt Ý NGHĨA của cột đã tồn tại sẵn" (VD cột `size` chỉ áp dụng khi `product_type="ao-thun"`), hoàn toàn khác bài toán "thêm THUỘC TÍNH MỚI mà không cần deploy" (P1) — 2 bài toán nghe gần nhau nhưng độc lập.

**Ví dụ minh hoạ cụ thể — 💭 thử "lách" mà không migrate, xem thất bại ở đâu:** vận hành nghĩ "cứ tạo sản phẩm Nội thất trước, tính sau":

```python
# 💭 tạo được dòng Product — KHÔNG báo lỗi, trông như đã "linh hoạt"
Product.objects.create(name="Ghế gỗ sồi", product_type="noi-that")
# Nhưng lưu giá trị "Chất liệu: Gỗ sồi" thì KHÔNG có cột nào để lưu — chat_lieu_go chưa tồn tại
# 2 lựa chọn đều tệ ngay lúc này, không phải sau này:
# (a) nhét tạm vào cột size (vốn dành cho quần áo) → dữ liệu sai nghĩa hoàn toàn, size="Gỗ sồi" vô lý
# (b) bỏ trống, hứa "cập nhật sau khi có cột" → sản phẩm lên sàn thiếu thông tin, khách không lọc được
```

**Dữ liệu thật trông thế nào — bảng `Product` dưới thiết kế STI thuần (2.2), sau khi đã cố "lách" thêm dòng Nội thất:**

| id | name | product_type | size | color | ram | storage |
|---|---|---|---|---|---|---|
| 1 | Áo thun đỏ | `ao-thun` | M | Đỏ | `NULL` | `NULL` |
| 2 | Tai nghe Bluetooth | `dien-tu` | `NULL` | `NULL` | `NULL` | 32GB |
| 3 | Ghế gỗ sồi | `noi-that` | `NULL` | `NULL` | `NULL` | `NULL` |

Nhìn thẳng vào dòng `id=3`: **toàn bộ 4 cột thuộc tính đều `NULL`** — không phải vì Ghế gỗ sồi "không có thuộc tính gì", mà vì đúng thuộc tính nó cần (`chat_lieu`) chưa từng có cột nào trên bảng để lưu. So với dòng `id=1`/`id=2` (vẫn có 2/4 cột `NULL` do bản chất STI, nhưng ít nhất còn ĐÚNG cột để lưu giá trị thật của loại đó), dòng `id=3` là bằng chứng cụ thể nhất cho thấy P1 chưa được giải — thêm 1 dòng dữ liệu không đồng nghĩa lưu được dữ liệu đó.

**Cùng dữ liệu đó dưới thiết kế thật (2.6) — không có dòng nào rỗng vô cớ:**

`AttributeVariant`/`AttributeProduct` (bảng gán, dữ liệu chứ không phải cột):

| id | attribute_id (→ tên) | product_type_id (→ tên) |
|---|---|---|
| 5 | 1 (Size) | 100 (Áo thun) |
| 6 | 2 (RAM) | 101 (Điện tử) |
| 7 | 3 (Chất liệu gỗ) | 102 (Nội thất) |

Thêm dòng `id=7` (gán "Chất liệu gỗ" cho "Nội thất") là **1 INSERT**, không đụng tới cấu trúc cột nào của `Product` — khác hẳn việc phải thêm cột `chat_lieu_go` như bảng STI thuần ở trên.

**Dấu hiệu nhận biết đã mắc lối tắt A:** nghe câu "đã có cột phân loại `type`/`category`" và dừng phân tích ngay tại đó, không tiếp tục hỏi "thêm 1 THUỘC TÍNH MỚI (không phải thêm 1 GIÁ TRỊ của thuộc tính cũ) có cần sửa schema không?"

**Lối tắt B — "CTI đúng chuẩn OOP hơn (is-a relationship rõ ràng), nên chắc tốt hơn STI"**

*Tự đoán trước: "đúng chuẩn OOP" là ưu điểm ở tầng nào — code hay database?*

**Vì sao sai:** đúng chuẩn OOP ở tầng **code** (class hierarchy sạch, `ClothingProduct` rõ ràng "is-a" `Product`) không đồng nghĩa đúng ở tầng **database** — bài toán thật ở đây (storefront liệt kê catalog trộn mọi ngành hàng) cần query "trộn mọi loại" THƯỜNG XUYÊN HƠN cần "tách riêng theo loại", nên chi phí UNION/dispatch của CTI trả giá đắt hơn lợi ích OOP mang lại. Đúng chuẩn kiến trúc code không tự động đúng chuẩn kiến trúc dữ liệu — 2 tiêu chí đánh giá khác nhau.

**Ví dụ minh hoạ cụ thể — đếm số query thật cho 1 trang catalog 20 sản phẩm, trộn 3 ngành hàng (💭):**

```python
# CTI (2.3) — trang "Tất cả sản phẩm", 20 dòng trộn Áo/Điện tử/Sách
base_rows = Product.objects.all()[:20]                                  # query 1
clothing_ids = [p.id for p in base_rows if p.product_type == "ao-thun"]
electronics_ids = [p.id for p in base_rows if p.product_type == "dien-tu"]
book_ids = [p.id for p in base_rows if p.product_type == "sach"]
clothing = ClothingProduct.objects.filter(product_ptr_id__in=clothing_ids)      # query 2
electronics = ElectronicsProduct.objects.filter(product_ptr_id__in=electronics_ids)  # query 3
books = BookProduct.objects.filter(product_ptr_id__in=book_ids)                 # query 4
# → 4 query cho ĐÚNG 1 trang, và con số này TĂNG THEO SỐ NGÀNH HÀNG xuất hiện trên trang đó

# EAV+STI hybrid (2.6) — cùng đúng 1 trang, bất kể trộn bao nhiêu ngành hàng
products = Product.objects.prefetch_related("attributevalues")[:20]     # query 1 + 1 prefetch = 2 query cố định
```

**Hậu quả cụ thể:** số query của CTI **tăng tuyến tính theo số ngành hàng khác nhau xuất hiện trên 1 trang** — trang càng đa dạng ngành hàng (đúng mục tiêu ban đầu của bài toán: bán nhiều ngành hàng) càng chậm; trong khi hybrid giữ nguyên 2 query cố định dù trộn bao nhiêu ngành. Đây là nghịch lý của CTI trong đúng bài toán này: kỹ thuật "sạch" ở tầng code lại **phạt nặng nhất đúng lúc bài toán thành công** (càng nhiều ngành hàng càng chậm).

**Cụ thể hoá bằng số — số query thật tại từng quy mô trang (💭, nối tiếp đúng đoạn code đếm query ở trên):**

| Trang catalog | Số ngành hàng khác nhau xuất hiện trong 20 sản phẩm | Số query CTI (1 gốc + N bảng con) | Số query Hybrid |
|---|---|---|---|
| "Áo thun" (trang riêng 1 ngành) | 1 | 2 | 2 |
| "Tất cả sản phẩm" (Áo + Điện tử + Sách) | 3 | 4 | 2 |
| + mở thêm Nội thất, Mỹ phẩm | 5 | 6 | 2 |
| "Khuyến mãi toàn sàn" (trộn hầu hết ngành đang có) | 10 | 11 | 2 |

Cột CTI tăng đúng 1 đơn vị mỗi khi có thêm 1 ngành hàng xuất hiện trên trang — không có trần trên, vì phụ thuộc trực tiếp số `product_type` phân biệt trong đúng 20 dòng đang phân trang, không phụ thuộc tổng số ngành hàng của cả sàn. Quy ra độ trễ (🧠, ước tính chung dựa trên round-trip DB điển hình, không phải benchmark đo thật trên Saleor): mỗi query JOIN thêm tốn khoảng 5-15ms tuỳ pool connection và tải server tại thời điểm đó — trang "Khuyến mãi toàn sàn" (11 query) có thể cộng dồn thêm 50-150ms so với hybrid (2 query cố định) chỉ để tải đúng 1 trang, đủ để ăn vào ngân sách thời gian phản hồi (time-to-first-byte) mà 1 trang thương mại điện tử thường nhắm tới (thường dưới 200-300ms cho phần backend).

**Dữ liệu thật trông thế nào — 3 bảng vật lý tách biệt, không bảng nào chứa đủ để hiển thị 1 trang catalog:**

`Product` (bảng gốc, CTI):

| id | name | product_type |
|---|---|---|
| 1 | Áo thun đỏ | `ao-thun` |
| 2 | Tai nghe Bluetooth | `dien-tu` |
| 3 | Sách Dế Mèn | `sach` |

`ClothingProduct` (bảng con riêng, `product_ptr_id` là FK ẩn về `Product`):

| product_ptr_id | size | color |
|---|---|---|
| 1 | M | Đỏ |

`ElectronicsProduct` (bảng con riêng):

| product_ptr_id | ram | storage |
|---|---|---|
| 2 | — | 32GB |

`BookProduct` (bảng con riêng):

| product_ptr_id | author | publisher |
|---|---|---|
| 3 | Tô Hoài | Kim Đồng |

**Trace tay — muốn hiển thị "Tai nghe Bluetooth có storage bao nhiêu" ngay trên card catalog:** tra `Product` id=2 → biết `product_type="dien-tu"` → PHẢI biết trước quy tắc "dien-tu → tra bảng `ElectronicsProduct`" mới JOIN đúng chỗ → tra `ElectronicsProduct` `product_ptr_id=2` → ra `storage=32GB`. So với hybrid (2.6): tra thẳng `AssignedVariantAttributeValue` `WHERE variant_id=2` → ra ngay giá trị, không cần biết trước "loại nào tra bảng nào" — đúng khác biệt cốt lõi giữa "phải biết cấu trúc trước khi query" (CTI) và "cấu trúc nằm trong dữ liệu, query không cần biết trước" (EAV).

**Dấu hiệu nhận biết đã mắc lối tắt B:** đánh giá 1 lựa chọn database chỉ bằng tiêu chí "code có sạch/dễ đọc không" mà chưa đếm thử số query thật cho đúng pattern truy vấn phổ biến nhất của bài toán (ở đây là "liệt kê trộn", không phải "lấy 1 sản phẩm biết trước loại").

**Lối tắt C — "Bảng Phần 3 cho thấy EAV thuần Linh hoạt=✅ Dedup=✅ cao nhất trong các ứng viên không phải hybrid, vậy chọn nó là an toàn nhất khi chưa chắc chắn?"**

*Tự đoán trước: "an toàn nhất" theo tiêu chí nào — có tính luôn cả P3 (kiểu dữ liệu) không?*

**Vì sao sai:** "cao nhất" chỉ đúng trên đúng 2 cột (P1, P2) — đọc cả hàng sẽ thấy cột "Kiểu gốc + lọc khoảng" (P3) là ✗, đánh đổi mất hẳn kiểu dữ liệu gốc là mất thật, ảnh hưởng trực tiếp trải nghiệm khách hàng — 1 đánh đổi dễ bị bỏ sót nếu chỉ nhìn lướt 2 cột đầu thay vì đọc hết cả hàng.

**Dữ liệu thật trông thế nào — bảng `AttributeValue` dưới thiết kế EAV thuần (2.5, chỉ 1 cột `value` kiểu TEXT):**

| id | attribute_id (→ RAM) | value (TEXT) |
|---|---|---|
| 10 | 5 | `"4GB"` |
| 11 | 5 | `"8GB"` |
| 12 | 5 | `"16GB"` |

Chạy đúng câu khách hàng bấm "RAM tăng dần" — `SELECT * FROM attribute_value WHERE attribute_id=5 ORDER BY value ASC`:

| Thứ tự trả về | id | value | Vì sao đứng vị trí này |
|---|---|---|---|
| **1** | 12 | `"16GB"` | Ký tự đầu `'1'` nhỏ hơn `'4'` và `'8'` trong bảng mã ASCII — Postgres so sánh từng ký tự, không biết đây là số |
| 2 | 10 | `"4GB"` | Ký tự đầu `'4'` |
| 3 | 11 | `"8GB"` | Ký tự đầu `'8'` |

Khách hàng bấm "RAM tăng dần" nhưng thấy "16GB" hiện đầu tiên — sai hiển nhiên với mắt thường, nhưng hệ thống không có exception, không log lỗi nào cả, vì về mặt kỹ thuật `ORDER BY value` chạy đúng như được yêu cầu — chỉ là yêu cầu đó (sort chuỗi) không khớp ý định thật (sort số).

**Cùng 3 dòng dữ liệu đó dưới thiết kế thật (2.6) — thêm đúng 1 cột là đủ sửa:**

| id | attribute_id | value | **numeric** |
|---|---|---|---|
| 10 | 5 | `"4GB"` | **4** |
| 11 | 5 | `"8GB"` | **8** |
| 12 | 5 | `"16GB"` | **16** |

`ORDER BY numeric ASC` → trả về đúng thứ tự 10 (4) → 11 (8) → 12 (16). Cột `value` (TEXT) vẫn giữ nguyên để HIỂN THỊ ("4GB" thân thiện hơn số trần "4"), cột `numeric` (FLOAT) dùng riêng để SO SÁNH/SORT — đây chính là lý do phải tách 2 cột thay vì ép chung 1 cột như EAV thuần.

**Hậu quả cụ thể:** lỗi loại này thường sống sót qua QA vì bộ test dữ liệu nhỏ (2-3 giá trị) ít khi có đủ cặp giá trị lộ ra sai thứ tự (`"4" < "8"` đúng cả 2 cách so sánh, chỉ `"16"` mới lộ vấn đề) — nghĩa là **test pass không đồng nghĩa thiết kế đúng**, chỉ đồng nghĩa test chưa đủ dữ liệu để lộ ra bug đã có sẵn từ lúc chọn kỹ thuật.

**Cụ thể hoá bằng bảng — đúng những bộ dữ liệu nào làm lộ bug, bộ nào không (💭, vẫn trên Attribute RAM):**

| Bộ giá trị test | So sánh chuỗi (thực tế chạy) | So sánh số (ý định thật) | 2 kết quả có khớp nhau không | QA có bắt được bug không |
|---|---|---|---|---|
| `"4GB"`, `"8GB"` | 4 < 8 | 4 < 8 | Khớp | Không — PASS dù thiết kế sai |
| `"2GB"`, `"4GB"`, `"8GB"` | 2 < 4 < 8 | 2 < 4 < 8 | Khớp | Không — vẫn PASS |
| `"4GB"`, `"8GB"`, `"16GB"` | `"16" < "4" < "8"` | 4 < 8 < 16 | **Lệch** | **Có — FAIL, lộ đúng bug** |
| `"4GB"`, `"9GB"`, `"10GB"` | `"10" < "4" < "9"` | 4 < 9 < 10 | **Lệch** | **Có — FAIL, lộ đúng bug** |

Điều kiện để bug lộ ra không phải "có nhiều giá trị test" một cách chung chung, mà cụ thể là: **phải có ít nhất 1 cặp giá trị mà số chữ số khác nhau** (VD "8" 1 chữ số vs "16" 2 chữ số) — vì so sánh chuỗi chỉ đúng tình cờ khi mọi giá trị cùng độ dài chữ số. Nhiều bộ RAM ngoài đời thực (4/8/16/32GB) đều có cặp lệch độ dài như vậy, nên bug gần như chắc chắn xảy ra ở dữ liệu sản xuất thật — chỉ là bộ test do QA tự chọn (thường ưu tiên giá trị "đẹp", ít khi cố ý trộn 1 và 2 chữ số) tình cờ không rơi vào đúng trường hợp lộ bug. Hậu quả trong thực tế vận hành: bug tồn tại từ ngày đầu deploy, chỉ được phát hiện khi có khách hàng cụ thể report "sao RAM 16GB lại đứng đầu danh sách tăng dần" — có thể là vài tuần đến vài tháng sau, tuỳ catalog có đủ đa dạng RAM để rơi đúng trường hợp lộ bug hay không.

**Dấu hiệu nhận biết đã mắc lối tắt C:** đọc bảng so sánh chỉ theo CỘT (so 1 tiêu chí giữa các kỹ thuật) thay vì theo HÀNG (xem 1 kỹ thuật có yếu ở tiêu chí nào khác không) — "cao nhất" ở 2/5 tiêu chí không có nghĩa "tốt nhất" tổng thể nếu tiêu chí còn lại là điều kiện bắt buộc (P3 ở đây không phải tiêu chí phụ, mà trực tiếp ảnh hưởng trải nghiệm khách hàng).

**Lối tắt D — "Phần 7 cho thấy hybrid mạnh hơn ở CẢ P1, P2, P3 so với mọi ứng viên khác — vậy nên luôn dùng hybrid cho bất kỳ bài toán 'loại con có thuộc tính khác nhau' nào, kể cả `ShippingMethod`?"**

*Tự đoán trước: "mạnh hơn ở mọi tiêu chí đã liệt kê" có đồng nghĩa "nên dùng cho mọi bài toán cùng hình dạng" không — còn tiêu chí nào chưa liệt kê?*

**Vì sao sai:** Phần 7 chỉ so sánh 3 tiêu chí (P1-P3) — **không có tiêu chí "chi phí thêm vào đường hot-path"**. Tính phí ship chạy ở MỌI lần checkout (đường hot-path, không phải màn hình quản trị ít khi mở như catalog admin). Áp hybrid vào `ShippingMethod` — 💭 minh hoạ số liệu:

```python
# 💭 nếu áp EAV+STI hybrid vào ShippingMethod thay vì STI thuần đang dùng thật (đã grounded ở Phần 8)
# Mỗi lần tính phí ship (checkout), thay vì đọc thẳng 2 cột trên ShippingMethod:
method.minimum_order_weight, method.maximum_order_weight     # 0 JOIN thêm — cột ngay trên bảng

# ...sẽ phải làm thêm:
ShippingMethodAttributeValue.objects.filter(shipping_method=method).select_related("value", "value__attribute")
# → cộng thêm 2-3 JOIN vào ĐÚNG đường chạy mỗi đơn hàng, chỉ để đổi lấy khả năng "linh hoạt thêm loại
# shipping method mới không cần deploy" — khả năng không ai cần, vì PRICE_BASED/WEIGHT_BASED chưa từng
# đổi qua nhiều năm phát triển Saleor
```

**Hậu quả cụ thể:** trả giá bằng latency thật ở đường chạy nhiều nhất trong toàn hệ thống (mọi checkout), để mua 1 khả năng linh hoạt không có bằng chứng nghiệp vụ nào yêu cầu — đúng định nghĩa over-engineering đã nêu ở nguyên tắc "chỉ leo thang khi có bằng chứng cụ thể".

**Cụ thể hoá bằng số — quy đổi 2-3 JOIN thừa thành chi phí cộng dồn mỗi ngày (🧠 minh hoạ, không phải benchmark đo thật, chỉ để thấy bậc độ lớn):**

| Chỉ số | STI thuần (thật, Phần 8) | Hybrid áp nhầm (💭) |
|---|---|---|
| Số JOIN để biết "method này tính phí kiểu gì" | 0 — đọc thẳng cột `type` trên `ShippingMethod` | 2-3 — qua `AssignedShippingMethodAttributeValue` → `AttributeValue` → lấy `value` |
| Độ trễ thêm mỗi lần đọc (ước tính round-trip DB điển hình) | 0ms | ~3-5ms/JOIN × 2-3 JOIN ≈ 6-15ms |
| Số lượt checkout/ngày (sàn quy mô vừa, giả định minh hoạ) | — | 50.000 lượt/ngày |
| Tổng thời gian cộng dồn thêm mỗi ngày chỉ cho đúng bước đọc `type` | — | 50.000 × ~10ms ≈ 500.000ms ≈ **8-9 phút CPU/IO dồn lại mỗi ngày** |

Con số "8-9 phút/ngày" tự nó không lớn — điểm cần thấy là nó bị **cộng vào đúng nơi tần suất chạy cao nhất trong cả hệ thống** (mọi đơn hàng, không phải màn hình quản trị mở vài lần/ngày), và đổi lại chỉ để mua đúng 1 khả năng ("thêm loại phí ship thứ 3 không cần deploy") mà theo Phần 8, `PRICE_BASED`/`WEIGHT_BASED` **chưa từng đổi qua nhiều năm phát triển Saleor** — tức trả phí hằng ngày, thật, để mua 1 quyền chọn gần như chắc chắn không bao giờ dùng tới. Đây đúng cấu trúc chung của over-engineering: chi phí là chắc chắn và lặp lại (mỗi checkout), lợi ích là giả định và chưa từng xảy ra.

**Dữ liệu thật trông thế nào — cùng 1 thông tin ("method này WEIGHT_BASED hay PRICE_BASED"), 2 cách lưu:**

`ShippingMethod` dưới STI thuần (2.2, đúng thiết kế thật của Saleor — 1 cột, đọc thẳng):

| id | name | type | minimum_order_weight | maximum_order_weight |
|---|---|---|---|---|
| 1 | Giao tiêu chuẩn | `WEIGHT_BASED` | 0kg | 5kg |
| 2 | Giao nhanh | `PRICE_BASED` | `NULL` | `NULL` |

Đọc "method id=1 là loại gì" — 1 lần đọc cột `type`, xong.

`ShippingMethod` dưới hybrid áp nhầm (💭, phản-thực-tế) — `type` không còn là cột, mà là 1 Attribute như mọi thuộc tính khác:

| Bảng | Dữ liệu |
|---|---|
| `ShippingMethod` | `id=1, name="Giao tiêu chuẩn"` — KHÔNG còn cột `type` |
| `AssignedShippingMethodAttributeValue` | `shipping_method_id=1, value_id=50` |
| `AttributeValue` | `id=50, attribute_id=9 (Loại phí ship), value="WEIGHT_BASED"` |

Đọc "method id=1 là loại gì" giờ cần **3 bước JOIN** thay vì 1 lần đọc cột — và bước này chạy ở MỖI đơn hàng, không phải màn hình quản trị ít khi mở. Đây là con số cụ thể hoá cho "2-3 JOIN thừa" đã nói ở trên: không phải ước lượng mơ hồ, mà đúng 3 bảng phải đi qua để lấy lại 1 thông tin trước đó chỉ cần đọc 1 cột.

**Dấu hiệu nhận biết đã mắc lối tắt D:** thấy 1 kỹ thuật "thắng" trong 1 bảng so sánh rồi áp nó cho MỌI bài toán có vẻ giống hình dạng, mà không tự hỏi lại "bài toán mới này có đúng đặc tính (P1-P5) đã dùng để đánh giá không, hay chỉ giống bề ngoài (đều là 'loại con có thuộc tính khác nhau')?" — đúng sai lầm ngược chiều với lối tắt A/B/C (những cái đó dừng lại quá sớm, cái này leo thang quá xa).

---

## Phần 6 — Thiết kế thật của Saleor — tại mỗi ngã rẽ, vì sao KHÔNG đi theo 5 kỹ thuật kia

Đọc lướt code sẽ tưởng Saleor "chỉ dùng EAV" — thực ra thiết kế thật là kết quả của **5 ngã rẽ tách biệt**, mỗi ngã rẽ đúng 1 điểm mà 1 trong 5 ứng viên ở Phần 2 có thể đã được chọn, nhưng Saleor rẽ hướng khác. Đọc theo đúng thứ tự dưới đây để thấy: không phải "Saleor tự nhiên có thiết kế phức tạp", mà là 5 quyết định giới hạn phạm vi liên tiếp, mỗi quyết định có bằng chứng code cụ thể.

### 6.1 Ngã rẽ tránh Fixed columns (2.1) — mã hoá "ai dùng thuộc tính gì" vào DỮ LIỆU, không vào SCHEMA

Điểm quyết định: `size`/`color` có thể đã nằm thẳng trên `Product` như 2.1. Saleor rẽ khác — tách `Attribute` thành bảng độc lập, và **quan hệ "ProductType nào dùng Attribute nào" cũng là 1 DÒNG DỮ LIỆU** (`AttributeVariant`/`AttributeProduct`), không phải 1 CỘT trong schema.

✅ Mở ngành hàng "Nội thất" trong Saleor thật cần đúng 2 lệnh, KHÔNG 1 dòng `ALTER TABLE`:
```python
chat_lieu_go = Attribute.objects.create(slug="chat-lieu-go", name="Chất liệu gỗ", input_type=AttributeInputType.DROPDOWN)
AttributeProduct.objects.create(attribute=chat_lieu_go, product_type=noi_that)
```
So với timeline t=0→t=+5 ngày đã mô phỏng ở 2.1 (chờ dev viết + deploy migration), 2 dòng trên chạy được ngay lúc vận hành bấm "Lưu" trên UI admin — không có bước "chờ dev" nào trong luồng này.

**Dữ liệu thật trông thế nào — trace đủ 3 bảng, từ định nghĩa tới sản phẩm thật:**

`Attribute` (định nghĩa chung, không đụng cột `Product` nào):

| id | slug | name | input_type |
|---|---|---|---|
| 3 | `chat-lieu-go` | Chất liệu gỗ | DROPDOWN |

`AttributeProduct` (bảng gán — chính dòng dữ liệu này "dạy" hệ thống biết Nội thất cần Attribute nào, không phải cột schema nào):

| id | attribute_id | product_type_id (→ Nội thất) |
|---|---|---|
| 12 | 3 | 102 |

`Product` (bảng gốc, KHÔNG có cột `chat_lieu_go` nào cả — và không cần):

| id | name | product_type_id |
|---|---|---|
| 50 | Ghế gỗ sồi | 102 |

`AssignedProductAttributeValue` (giá trị thật, nối `Product` id=50 với đúng giá trị "Gỗ sồi"):

| id | value_id (→ AttributeValue "Gỗ sồi") | product_id |
|---|---|---|
| 80 | 30 | 50 |

**Trace tay:** "Ghế gỗ sồi (id=50) làm bằng chất liệu gì" → tra `AssignedProductAttributeValue` có `product_id=50` → `value_id=30` → tra `AttributeValue` id=30 → `name="Gỗ sồi"`. 4 bảng, 0 cột nào trong số đó là cột đặc thù riêng cho Nội thất trên `Product` — khác hẳn bảng STI thuần ở Phần 5 (Lối tắt A) nơi `Product` phải tự mang cột `chat_lieu_go` mới lưu được giá trị này.

### 6.2 Ngã rẽ tránh STI thuần (2.2) — dùng ĐÚNG discriminator STI, nhưng chỉ ở nơi số loại THẬT SỰ cố định

Đây là ngã rẽ tinh vi nhất, dễ đọc nhầm thành "Saleor không dùng STI" — thực ra Saleor **vẫn dùng STI** (discriminator `input_type` + nhiều cột dùng chung trên `AttributeValue`, đã thấy ở 2.6), chỉ khác 1 điều: áp đúng chỗ số loại **thật sự cố định, do DEV kiểm soát** — ✅ `AttributeInputType.CHOICES` (`attribute/__init__.py:4-29`) liệt kê đúng 12 loại: `DROPDOWN`/`MULTISELECT`/`FILE`/`REFERENCE`/`SINGLE_REFERENCE`/`NUMERIC`/`RICH_TEXT`/`PLAIN_TEXT`/`SWATCH`/`BOOLEAN`/`DATE`/`DATE_TIME` — khác hẳn "ngành hàng", nơi số lượng KHÔNG cố định và do VẬN HÀNH kiểm soát (đúng đặc tính P1).

Nói cách khác: Saleor không tránh STI vì STI "luôn sai" — mà vì áp đúng bài học đã rút ở [Phần 8](#phần-8-đối-chứng-trong-chính-saleor--khi-nào-không-cần-ghép) (`ShippingMethod`, cũng STI thuần, cũng đúng vì chỉ 2 loại cố định): **STI đúng khi số loại bị chặn trên và hiếm khi đổi; sai khi số loại do người vận hành tự mở rộng liên tục.** `input_type` thoả điều kiện đầu (12 loại, chỉ dev mới thêm loại thứ 13 khi cần hỗ trợ kiểu dữ liệu mới — hiếm khi xảy ra), "ngành hàng" thoả điều kiện sau (P1) — nên 2 khái niệm **giống nhau về hình dạng bài toán** ("nhiều loại con") lại cần 2 kỹ thuật khác nhau ngay trong cùng 1 entity `Attribute`/`AttributeValue`.

**Dữ liệu thật đặt cạnh nhau — 2 bảng "danh mục loại con" trong cùng hệ thống, 1 cố định 1 không:**

Danh mục `input_type` — cố định, KHÔNG có bảng DB nào cả, chỉ tồn tại dạng hằng số trong code (`attribute/__init__.py:4-29`), vì không ai cần thêm dòng mới qua UI:

| Giá trị | Ai thêm được | Tần suất đổi |
|---|---|---|
| DROPDOWN, MULTISELECT, FILE, REFERENCE, SINGLE_REFERENCE, NUMERIC, RICH_TEXT, PLAIN_TEXT, SWATCH, BOOLEAN, DATE, DATE_TIME | Chỉ dev (sửa code + release) | Nhiều năm chưa đổi |

Danh mục "ngành hàng" (`ProductType`) — bảng DB thật, vận hành tự thêm dòng bất cứ lúc nào:

| id | name | Ai thêm |
|---|---|---|
| 100 | Áo thun | Vận hành, qua UI admin |
| 101 | Điện tử | Vận hành, qua UI admin |
| 102 | Nội thất | Vận hành, qua UI admin (vừa thêm ở 6.1) |
| 103 | ... (bất kỳ lúc nào) | Vận hành |

Đặt 2 bảng cạnh nhau lộ rõ lý do kỹ thuật: `input_type` không cần là 1 bảng DB (không ai INSERT dòng mới vào đó qua ứng dụng) — nên STI (cột cứng gắn theo hằng số cố định) hoàn toàn an toàn; `ProductType` bắt buộc là 1 bảng DB có INSERT liên tục — nên cần EAV (cấu trúc nằm trong dữ liệu, không nằm trong schema).

### 6.3 Ngã rẽ tránh CTI (2.3) — 1 bảng giá trị duy nhất, không tách bảng theo `input_type`

Điểm quyết định: đã chọn STI cho tầng giá trị (6.2), vẫn còn 1 lựa chọn khác — tách `NumericAttributeValue`/`TextAttributeValue`/`DateAttributeValue`... thành bảng riêng theo từng `input_type` (đúng dạng CTI của 2.3, áp cho tầng giá trị thay vì tầng Product). Saleor không làm vậy — gộp chung 1 bảng `AttributeValue`.

✅ Lấy toàn bộ giá trị thuộc tính của 1 Variant, bất kể Size là DROPDOWN hay Cân nặng là NUMERIC, chỉ cần đúng 1 JOIN:
```python
AssignedVariantAttributeValue.objects.filter(variant=variant).select_related("value")
```
Nếu tách theo CTI (mỗi `input_type` 1 bảng), câu này phải viết lại thành N query khác nhau tuỳ input_type nào xuất hiện trên Variant đó — đúng chi phí "dispatch theo loại" đã thấy ở 2.3, giờ tránh được nhờ gộp bảng.

**Dữ liệu thật trông thế nào — 1 Variant có cả thuộc tính DROPDOWN lẫn NUMERIC, cùng nằm trong 1 bảng:**

`AttributeValue` (1 bảng duy nhất, dù `input_type` khác nhau — Size là DROPDOWN, RAM là NUMERIC):

| id | attribute_id (→ tên, input_type) | value | numeric |
|---|---|---|---|
| 20 | 2 (Size, DROPDOWN) | `"M"` | `NULL` |
| 21 | 6 (RAM, NUMERIC) | `""` | `8` |

`AssignedVariantAttributeValue` (gắn cả 2 giá trị trên vào ĐÚNG 1 Variant, VD "Tai nghe Bluetooth 8GB, size hộp M"):

| id | value_id | variant_id |
|---|---|---|
| 90 | 20 | 900 |
| 91 | 21 | 900 |

**Trace tay:** lấy hết thuộc tính của Variant id=900 — `SELECT * FROM assigned_variant_attribute_value av JOIN attribute_value v ON av.value_id = v.id WHERE av.variant_id = 900` → trả về CẢ 2 dòng (id=90 và id=91) trong ĐÚNG 1 câu query, dù 1 dòng là DROPDOWN và 1 dòng là NUMERIC. Nếu theo CTI (tách `NumericAttributeValue`/`TextAttributeValue` riêng), phải chạy 2 câu query khác nhau (1 cho mỗi bảng) rồi tự gộp kết quả ở code — đúng số lượng query tăng theo số `input_type` xuất hiện, y hệt vấn đề đã thấy ở CTI cho `Product` (Phần 5, Lối tắt B), chỉ khác đối tượng bị tách bảng (giá trị, không phải sản phẩm).

### 6.4 Ngã rẽ tránh JSONB toàn cục (2.4) — JSONB chỉ dùng cho ĐÚNG 1 trong 12 `input_type`

Điểm quyết định: Saleor rõ ràng biết và dùng JSONB tốt (`metadata`/`private_metadata` ở `core/models.py:81-98`) — nên đây không phải "không biết tới JSONB", mà là quyết định **giới hạn phạm vi** nó được dùng.

✅ Trong chính `AttributeValue` (`attribute/models/base.py:365-421`), chỉ đúng 1 cột là JSONB thật:
```python
rich_text = SanitizedJSONField(blank=True, null=True, sanitizer=clean_editorjs)   # chỉ khi input_type=RICH_TEXT
```
11 cột còn lại (`value`, `numeric`, `boolean`, `date_time`, `reference_*`...) đều KHÔNG phải JSONB, dù kỹ thuật hoàn toàn có thể gộp tất cả vào 1 cột `data = JSONField()` duy nhất (đúng thiết kế 2.4 áp vào tầng giá trị thay vì tầng Product). Ranh giới Saleor tự đặt ra: JSONB đúng cho **nội dung dạng tài liệu, không cố định trước, không cần lọc theo khoảng/dedup** (rich text — cây block EditorJS) — sai cho **giá trị cần lọc/sort/dedup** (số, ngày, boolean, reference).

**Dữ liệu thật trông thế nào — 3 dòng `AttributeValue`, mỗi dòng khác `input_type`, chỉ đúng 1 dòng dùng cột JSONB:**

| id | attribute_id (→ input_type) | value | numeric | rich_text (JSONB) |
|---|---|---|---|---|
| 30 | 1 (Size, DROPDOWN) | `"M"` | `NULL` | `NULL` |
| 31 | 6 (RAM, NUMERIC) | `""` | `8` | `NULL` |
| 32 | 9 (Mô tả chi tiết, RICH_TEXT) | `""` | `NULL` | `{"blocks":[{"type":"paragraph","data":{"text":"Vải cotton 100%..."}}]}` |

Nhìn theo cột `rich_text`: 2/3 dòng là `NULL`, đúng 1 dòng (`id=32`) có dữ liệu JSONB thật — và đúng dòng đó cũng là dòng duy nhất KHÔNG cần lọc theo khoảng hay dedup (không ai "lọc sản phẩm có mô tả chứa đúng cụm từ X" bằng index B-Tree, chỉ cần hiển thị nguyên khối). So với `Cách A` (JSONB toàn cục, 2.4) nếu áp cho cả 3 dòng trên, dòng `id=30`/`id=31` sẽ mất index kiểu riêng đã thấy ở 6.5 ngay dưới.

### 6.5 Ngã rẽ tránh EAV thuần (2.5) — không collapse 12 kiểu về 1 cột `value`

Điểm quyết định: đã có bảng `AttributeValue` riêng (đúng bước dedup của EAV) — bước tiếp theo tự nhiên nhất (và đúng nghĩa "thuần" của EAV) là gộp mọi giá trị vào 1 cột `value TEXT`. Saleor không dừng ở đó — tách tiếp thành nhiều cột kiểu riêng, mỗi cột có index đúng bản chất nó:

✅ `attribute/models/base.py:424-437`:
```python
class Meta:
    indexes = [
        GinIndex(name="attribute_search_gin", fields=["name", "slug"], opclasses=["gin_trgm_ops"] * 2),
        BTreeIndex(fields=["numeric"], name="attribute_value_numeric_idx"),
    ]
```
`GinIndex` (tìm kiếm text mờ theo tên/slug) và `BTreeIndex` (so sánh khoảng theo số) là 2 loại index **không thể cùng gắn hiệu quả trên 1 cột `value TEXT` chung** — đây là bằng chứng vật lý (không phải suy luận) rằng nếu Saleor chọn EAV thuần, bug sort RAM 4GB/8GB/16GB đã mô phỏng ở 2.5 sẽ xảy ra thật, vì không có cột `numeric` riêng để `BTreeIndex` gắn vào.

**Dữ liệu thật minh hoạ thêm 1 góc khác — không phải sort, mà là TÍNH TOÀN VẸN dữ liệu (data integrity) — Attribute "Có bảo hành" (BOOLEAN):**

Dưới EAV thuần (2.5, cột `value` TEXT tự do, không ràng buộc kiểu):

| id | attribute_id (→ Có bảo hành) | value (TEXT, nhập tự do) |
|---|---|---|
| 40 | 7 | `"true"` |
| 41 | 7 | `"0"` |
| 42 | 7 | `""` (bỏ trống) |
| 43 | 7 | `"Yes"` |

4 dòng trên **đều lưu được thành công**, không có lỗi nào — nhưng biểu diễn "có bảo hành" theo 4 cách khác nhau (`"true"`, `"0"`, rỗng, `"Yes"`). Query `WHERE value = 'true'` chỉ khớp đúng dòng `id=40`, bỏ sót `id=43` (`"Yes"` cũng có nghĩa "có" theo người nhập, nhưng khác chuỗi) — không phải lỗi hệ thống, mà lỗi **không có ràng buộc nào ở tầng DB ngăn dữ liệu rác vào cột tự do**.

Dưới thiết kế thật (2.6, cột `boolean = BooleanField(null=True)`):

| id | attribute_id | boolean |
|---|---|---|
| 40 | 7 | `True` |
| 41 | 7 | `False` |
| 42 | 7 | `NULL` |

Chỉ có đúng 3 giá trị hợp lệ (`True`/`False`/`NULL`) — Postgres từ chối thẳng bất kỳ giá trị nào khác ngay ở tầng cột (`"Yes"` không thể lưu vào cột kiểu `boolean`), không cần code tự validate. Đây là ví dụ thứ 2 (khác sort ở 2.5/Phần 5) cho thấy hệ quả của "ép mọi kiểu về TEXT" — không chỉ sai thứ tự sort, mà còn mất luôn ràng buộc kiểu dữ liệu mà chính kiểu đó vốn có sẵn (boolean chỉ có 2-3 trạng thái, TEXT thì vô hạn khả năng).

---

**Tổng kết 5 ngã rẽ:** không có ngã rẽ nào Saleor "phát minh" kỹ thuật mới — ở mỗi ngã rẽ, kỹ thuật được chọn đều đã có tên trong Phần 2 (EAV, STI, JSONB đều được dùng thật — chỉ khác chỗ dùng). Điều thật sự khác biệt là **phạm vi áp dụng của từng kỹ thuật được giới hạn đúng chỗ nó mạnh nhất**, thay vì chọn 1 kỹ thuật rồi áp cho toàn bộ bài toán. Đây là câu trả lời đầy đủ nhất cho câu hỏi "tại sao Saleor chọn thiết kế này" — không phải vì hybrid là 1 kỹ thuật thứ 6 tách biệt, mà vì đó là kết quả của 5 quyết định giới hạn phạm vi liên tiếp, mỗi quyết định đứng vững độc lập với 4 quyết định còn lại.

---

## Phần 7 — Ma trận điểm mạnh/yếu — vì sao phải ghép, không dùng thuần 1 cái

| | EAV thuần | STI thuần | EAV+STI hybrid (thật) |
|---|---|---|---|
| Dedup, đổi được không migration (P1, P2) | ✅ mạnh | ✗ yếu (loại mới = cột mới) | ✅ mạnh (thừa hưởng từ EAV) |
| Giữ kiểu gốc, lọc theo khoảng có index riêng (P3) | ✗ yếu (ép về TEXT) | ✅ mạnh (cột đúng kiểu) | ✅ mạnh (thừa hưởng từ STI) |
| Số cột NULL lãng phí | Không có (1 cột value) | Có, nhưng ít (chỉ N loại cố định) | Có — ~10/11 cột NULL mỗi dòng, đánh đổi lớn nhất |

Dòng cuối là cái giá phải trả — Saleor chấp nhận lãng phí không gian (nhiều cột `NULL`) để đổi lấy cả 2 điểm mạnh P1-P3 cùng lúc. Đây đúng nguyên tắc chung đã rút ra: **hệ thống production hiếm khi chọn thuần 1 kỹ thuật sách vở — ghép 2 kỹ thuật, mỗi cái bù đúng điểm yếu của cái kia**, chấp nhận đánh đổi cụ thể (ở đây là NULL columns) thay vì đánh đổi mù mờ.

**Cân đúng 2 vế của đánh đổi bằng số, không chỉ định tính (🧠, dựa trên cách Postgres lưu trữ NULL — kiến thức chung, không phải benchmark thật đo trên Saleor):**

| Vế | Chi phí | Có đo được không |
|---|---|---|
| ~10/11 cột `NULL` mỗi dòng `AttributeValue` | Postgres biểu diễn `NULL` qua 1 bit trong null bitmap ở header mỗi dòng (không lưu giá trị thật cho cột đó) — với 11 cột nullable, overhead ước tính dưới 2 byte/dòng, không phụ thuộc số cột NULL nhiều hay ít | Có — cố định, dự đoán được trước |
| Bug sort/filter sai nếu chọn EAV thuần thay vì hybrid (đã thấy ở 2.5) | Không có exception, không log lỗi — chỉ hiển thị sai cho khách hàng, có thể tồn tại nhiều tháng trước khi ai phát hiện | Không — chỉ phát hiện khi có người dùng report hoặc tình cờ để ý |

So sánh này giải thích rõ hơn vì sao "lãng phí NULL" nghe đáng ngại về mặt cảm tính nhưng thực ra là chi phí **rẻ và có thể đo trước**, trong khi cái Saleor tránh được (EAV thuần) là chi phí **đắt và không đo trước được** — 2 loại rủi ro không cùng hạng, nên đánh đổi ở đây không đối xứng như nhìn thoáng qua tưởng.

---

## Phần 8 — Đối chứng trong chính Saleor — khi nào KHÔNG cần ghép

Để thấy rõ ranh giới, đáng nhìn 1 chỗ khác trong Saleor dùng **STI thuần, không ghép EAV**: `ShippingMethod` (`shipping/models.py:235-257`).

```python
# ✅ shipping/models.py:235-257
class ShippingMethod(ModelWithMetadata):
    type = models.CharField(choices=ShippingMethodType.CHOICES)   # discriminator: PRICE_BASED / WEIGHT_BASED
    minimum_order_weight = MeasurementField(..., null=True, blank=True)   # chỉ có nghĩa khi type=WEIGHT_BASED
    maximum_order_weight = MeasurementField(..., null=True, blank=True)
```

Không có EAV nào đi kèm ở đây — vì `ShippingMethod` chỉ có **đúng 2 loại con cố định**, không ai tự thêm loại thứ 3 qua màn hình vận hành (P1 không áp dụng). STI thuần đủ dùng khi thiếu đúng đặc tính P1 (mở rộng không cần deploy) — đây là bằng chứng ngược: không phải Saleor "thích EAV" nói chung, mà EAV+STI hybrid chỉ được chọn đúng khi bài toán thật sự có P1.

**Sai lầm nếu áp nhầm EAV+STI hybrid vào `ShippingMethod`:** nghe hybrid "mạnh hơn" ở Phần 7 dễ tưởng nên dùng nó cho mọi bài toán "loại con khác thuộc tính". Áp nhầm vào đây sẽ tạo ra 2 bảng `ShippingMethodAttribute`/`AssignedShippingMethodAttributeValue` chỉ để phục vụ đúng 2 giá trị `PRICE_BASED`/`WEIGHT_BASED` không bao giờ đổi — thêm 3-4 JOIN cho mỗi lần tính phí ship (đường hot-path, chạy mỗi checkout) để đổi lấy 1 khả năng linh hoạt (P1) không ai cần dùng tới. Đúng dạng over-engineering: leo thang kỹ thuật khi không có bằng chứng cụ thể nào đòi hỏi — cùng nguyên tắc đã rút ra trước đó ("chỉ leo thang khi có rule nghiệp vụ cụ thể xác nhận, không phải vì đoán").

---

## Phần 9 — Luyện phản biện — bảo vệ quyết định EAV+STI hybrid trước 3 kiểu phản đối thường gặp

**Phản đối 1 — "CTI đúng chuẩn OOP hơn (is-a rõ ràng), sao không dùng?"**

Trả lời: đúng chuẩn OOP là tiêu chí đánh giá **code**, không phải tiêu chí đánh giá **database**. Bài toán thật ở đây là storefront cần liệt kê catalog trộn mọi ngành hàng (query "trộn loại" xảy ra THƯỜNG XUYÊN, không phải ngoại lệ) — CTI làm điều này đắt hơn hẳn (UNION/dispatch nhiều bảng con), trong khi is-a relationship sạch ở tầng class hierarchy không mang lại lợi ích tương xứng cho đúng pattern truy vấn này. Nếu bài toán đổi khác (VD hiếm khi cần liệt kê trộn, mỗi loại con được truy cập tách biệt gần như hoàn toàn — như module Payment/Invoice riêng biệt), CTI có thể là lựa chọn hợp lý hơn — không phải CTI luôn sai, chỉ sai cho đúng pattern truy vấn của bài toán Product/Attribute.

**Phản đối 2 — "JSONB đơn giản, Postgres hỗ trợ GIN index tốt, sao phải phức tạp hoá bằng EAV+STI?"**

Trả lời: JSONB thiếu đúng 3 đặc tính bài toán này cần — không dedup (P2), không B-Tree riêng theo kiểu cho lọc khoảng (P3), REFERENCE không có FK thật để tự CASCADE (P4). GIN index giải tốt bài toán "tìm document khớp 1 điều kiện" nhưng không giải bài toán "1 giá trị chỉ tồn tại đúng 1 dòng vật lý, đổi 1 chỗ cập nhật mọi nơi đang dùng". Nếu bài toán không cần dedup/lọc khoảng/REFERENCE (VD `metadata`/`private_metadata` tự do cho App bên thứ 3 gắn vào bất kỳ entity nào — ✅ `core/models.py:81-98`, `ModelWithMetadata`, 1 cột JSONB + 1 GIN index trên cả blob, không tách field con), JSONB lại là lựa chọn đúng — không phải JSONB luôn kém, chỉ kém cho đúng 3 đặc tính P2/P3/P4 của riêng bài toán attribute.

**Phản đối 3 — "10/11 cột NULL mỗi dòng nghe lãng phí quá, sao không EAV thuần cho gọn?"**

Trả lời: 2 loại chi phí này không cùng hạng — cột `NULL` trên Postgres tốn gần như không đáng kể (biểu diễn qua null bitmap ở header mỗi dòng, không lưu giá trị thật cho cột đó; đây là kiến thức chung của Postgres, không phải trích từ code Saleor 🧠), trong khi EAV thuần trả giá bằng **lỗi hiển thị/sắp xếp sai cho khách hàng thật** (sort số theo thứ tự chuỗi) — 1 bên là chi phí lưu trữ nhỏ, đo được, không ảnh hưởng người dùng; bên kia là lỗi chức năng, có thể không ai phát hiện ngay vì test dữ liệu nhỏ ít khi lộ sai thứ tự. So sánh "lãng phí không gian" với "đúng chức năng" là so sánh sai hạng mục.

---

## Phần 10 — Checklist tái sử dụng

Áp cho bất kỳ bài toán "thuộc tính/loại con linh hoạt" nào khác, không riêng Product/Attribute:

| Câu hỏi | "Không" → | "Có" → |
|---|---|---|
| Số loại con/thuộc tính có cố định, biết trước, ít khi đổi không? | (tiếp tục xuống) | STI thuần đủ — xem ví dụ `ShippingMethod` (Phần 8) |
| Vận hành cần tự thêm loại thuộc tính mới, không đợi deploy (P1)? | Fixed columns/STI thuần đủ | (tiếp tục xuống) |
| Giá trị có cần dùng lại/dedup giữa nhiều bản ghi (P2)? | JSONB đủ (không cần tách bảng catalog riêng) | (tiếp tục xuống) |
| Có cần giữ đúng kiểu gốc để lọc theo khoảng, có index riêng (P3)? | EAV thuần đủ (1 cột `value`) | **Cần thêm STI-like ở tầng giá trị — đúng thiết kế Saleor** |
| Trước khi leo thang bất kỳ bước nào ở trên: có bằng chứng nghiệp vụ CỤ THỂ buộc phải leo, hay chỉ đang đoán trước cho chắc? | Dừng lại ở bước hiện tại, đừng leo thêm (xem Phần 8, "sai lầm nếu áp nhầm hybrid") | Leo đúng 1 bước tiếp theo, không nhảy cóc |

Đọc bảng này theo hàng dọc chính là cách tự đi lại con đường V1→V2→V3→thiết kế thật đã luyện ở [`saleor-greenfield-design-practice-product-attribute.md`](./saleor-greenfield-design-practice-product-attribute.md) — chỉ khác là đi từ dưới lên (bắt đầu từ câu hỏi, không phải từ code) thay vì đi từ trên xuống (bắt đầu từ code, xem điểm mù).

---

## Tự luyện tiếp theo

- Áp đúng bảng Phần 10 cho module khác đã học (Pricing 1c, Voucher/Promotion 1f) — dự đoán trước module đó rơi vào hàng nào trong Phần 3, rồi đối chiếu lại với code thật đã grounded ở file tương ứng
- Tự viết lại Phần 9 (Luyện phản biện) bằng lời của mình, không nhìn lại câu trả lời — kiểm tra còn nhớ đúng 3 đặc tính P2/P3/P4 gắn với từng phản đối không
- Với 1 bài toán "loại con" bất kỳ đang gặp trong công việc thật (không phải Saleor) — áp checklist Phần 10, tự xác định đang ở hàng nào, và có bằng chứng cụ thể nào cho phép leo thang tiếp không

---

## Liên hệ / Nguồn

- [`ecommerce-setup-1b-product-attribute.md`](./ecommerce-setup-1b-product-attribute.md) — nguồn grounding chính cho toàn bộ cấu trúc EAV 3 tầng + 11 cột `AttributeValue`
- [`saleor-greenfield-design-practice-product-attribute.md`](./saleor-greenfield-design-practice-product-attribute.md) — trình tự discovery dẫn tới cùng kết luận, V1/V2/V3 tương ứng đúng 3 hàng đầu bảng so sánh ở file này
- [`saleor-database-relationship-cardinality.md`](./saleor-database-relationship-cardinality.md) — phương pháp xác định cardinality 1-N/N-N dùng trong toàn bộ 3 file
- [`saleor-eav-hybrid-deep-dive.md`](./saleor-eav-hybrid-deep-dive.md) — đào sâu thêm đúng cặp EAV thuần vs hybrid, có đối chứng Magento/WordPress
- `shipping/models.py:235-257` — đối chứng STI thuần, không ghép EAV
