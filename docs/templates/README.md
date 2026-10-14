# Templates — khuôn viết file học domain

2 file trong thư mục này là **bài mẫu gốc** viết cho Saleor (e-commerce, Django), giữ nguyên văn để làm khuôn. Khi viết file mới cho Medplum, **chép cấu trúc, không chép nội dung**. Các link nội bộ trong 2 file (VD `ecommerce-setup-1b-product-attribute.md`, `saleor-ba-ambiguous-to-artifact-practice.md`) trỏ tới repo kiến thức Saleor gốc, **không tồn tại trong repo này**. Thấy link gãy là chuyện bình thường.

| Template | File mẫu | Trả lời câu hỏi | Khi nào dùng |
|---|---|---|---|
| **G — Greenfield** | [`saleor-greenfield-design-practice-product-attribute.md`](./saleor-greenfield-design-practice-product-attribute.md) | "Nếu chưa có dòng code nào, mình sẽ đi tới thiết kế này bằng con đường nào?" | Module có bài toán nghiệp vụ rõ (đặt lịch, billing, phân quyền...), muốn luyện discovery → thiết kế |
| **C — So sánh kỹ thuật** | [`saleor-design-technique-comparison-product-attribute.md`](./saleor-design-technique-comparison-product-attribute.md) | "Có N cách giải ngang hàng, vì sao hệ thống thật chọn cách này?" | Module có quyết định kỹ thuật lớn, có nhiều ứng viên cạnh tranh (cách lưu trữ, cách index, cách mở rộng) |

Cả 2 template đều dựa vào 1 loại file thứ 3 làm nguồn grounding. Trong Saleor đó là file `1b` (đọc code đã hoàn thiện, có `file:line`). Bên Medplum gọi loại này là **R — Đọc code thật**. Kết quả học của từng domain nằm ở `docs/<domain>/` (hiện có [`docs/health/`](../health/)). Quy ước chung xem [`docs/CLAUDE.md`](../CLAUDE.md), quy ước riêng cho health xem [kế hoạch](../health/plan.md#3-quy-ước-chung-cho-mọi-file).

## Bộ khung rút gọn — Template G

1. Quy tắc nhãn nguồn
2. Phần 1 — Bối cảnh giả định (1 câu nói của founder/khách) 💭
3. Phần 2 — BA view: Stakeholder map (5 câu hỏi) → Bảng business rule (tường minh/ẩn) → N câu hỏi discovery → Sai lầm thường gặp
4. Phần 3 — Mô hình khái niệm: rút thực thể → kiểm tra "thực thể thật?" → thuộc tính sơ bộ → ma trận quan hệ (hỏi 2 chiều) → thực thể phục vụ ai → cầu nối sang tên kỹ thuật → Sai lầm thường gặp
5. Phần 4 — Developer view: dịch khái niệm sang code → V1/V2/V3 (dừng đúng chỗ thông tin dừng) → điểm mù → 3 lối tắt "nghe hợp lý nhưng sai" → data flow → sai lầm khi code schema đã đúng
6. Phần 5 — Đối chiếu thực tế ✅ (code thật, `file:line`)
7. Phần 6 — Bảng so sánh tổng hợp
8. Phần 7 — Đánh giá 🧠: vì sao có khoảng cách, khi nào KHÔNG nên nhảy thẳng lên thiết kế thật, kèm luyện phản biện 3 kiểu phản đối
9. Phần 8 — Checklist tái sử dụng · Tự luyện tiếp theo · Liên hệ/Nguồn

## Bộ khung rút gọn — Template C

1. Quy tắc nhãn nguồn
2. Phần 1 — Bài toán: P1…Pn (đặc tính cụ thể), giải thích **vì sao** mỗi đặc tính tồn tại, kèm hậu quả bằng số
3. Phần 2 — N ứng viên, mỗi ứng viên có: giới thiệu (nguồn gốc), code, **bảng dữ liệu cùng 1 kịch bản xuyên suốt**, vấn đề kỹ thuật cụ thể, sai lầm thường gặp
4. Phần 3 — Bảng so sánh trực diện
5. Phần 4 — Vì sao N−1 ứng viên bị loại (áp đúng P1…Pn)
6. Phần 5 — 4 lối tắt "nghe hợp lý nhưng sai" (3 cái dừng quá sớm, 1 cái leo quá xa)
7. Phần 6 — Thiết kế thật: tại mỗi ngã rẽ, vì sao không đi theo ứng viên kia ✅
8. Phần 7 — Ma trận điểm mạnh/yếu, vì sao phải ghép, kèm đánh đổi được đo bằng số
9. Phần 8 — Đối chứng ngay trong chính hệ thống: chỗ nào KHÔNG cần ghép
10. Phần 9 — Luyện phản biện · Phần 10 — Checklist · Tự luyện · Nguồn
