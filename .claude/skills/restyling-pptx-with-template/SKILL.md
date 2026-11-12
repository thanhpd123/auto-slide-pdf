---
name: restyling-pptx-with-template
description: Use when an existing .pptx deck with correct content but poor design must be moved onto a new, nicer .pptx/.potx template (keeping the original wording), or when building/testing the Auto Slide "Làm đẹp slide cũ" pipeline and you need the reference rules and quality bar.
---

# Restyling a PPTX onto a template

## Overview
Đổ nội dung một bộ slide cũ sang template mới **mà không đổi một chữ nào**. Chất lượng chuẩn là bản làm tay trong bộ mẫu riêng của chủ dự án (`D:\File\ppt`, ngoài repo, không được đưa lên git). Khi đọc/ghi .pptx, dùng kèm skill `pptx`.

**Nguyên tắc cốt lõi:** câu chữ đi từ file cũ sang file mới bằng code (copy), không bao giờ đi qua AI để "viết lại".

## Quy tắc (theo prompt gốc của chủ dự án)
1. Lấy template làm gốc, giữ nguyên theme, master, layout, font, màu. Xoá các slide mẫu của template.
2. Mỗi slide cũ dùng **layout phù hợp nhất** của template và đổ vào placeholder, không chồng text box lên.
3. Giữ nguyên chữ, thứ tự slide, ảnh, bảng, speaker notes.
4. Nhiều chữ thì **chia thành 2 slide** (tiêu đề thêm "(tiếp)" hoặc "(2/2)"), không thu nhỏ chữ, không để tràn.
5. Không sửa file gốc. Ghi kết quả ra file mới.
6. Render từng slide ra ảnh, tự xem để bắt lỗi tràn chữ hoặc lệch khung, sửa rồi render lại.
7. Ảnh minh hoạ chỉ thêm khi slide cần: ảnh có giấy phép tự do (Wikimedia Commons: PD, CC BY, CC BY-SA). Mở ảnh xem có đúng chủ đề không. Chèn vào khung ảnh của layout, **crop, không kéo méo**.
8. Ghi `images/credits.txt`: tên ảnh, slide dùng, link gốc, tác giả, giấy phép.
9. **Không chèn ảnh** vào slide nhiều chữ hoặc slide có bảng/biểu đồ.
10. Cuối cùng báo: slide nào bị chia, slide nào đổi bố cục, slide nào cần người xem lại.

## Chọn layout nhanh
| Slide cũ | Layout |
|---|---|
| Slide đầu (tên bài, người trình bày) | Title (key visual) |
| Mục lục / roadmap | Title only, hoặc Title and content |
| Nhãn chương ("01 · <TÊN CHƯƠNG>") đổi chương | Chèn slide Section trước chương đó nếu template có |
| Bullet thuần | Title and content |
| Hai nhóm ý đối nhau (chỉ định / chống chỉ định) | Two columns |
| Bullet + ảnh | Text and image |
| Số liệu, biểu đồ | Chart and findings |
| Bảng | Table |
| Kết thúc / cảm ơn | Closing |

## Lỗi hay gặp
- **Slide cũ không dùng placeholder**, chỉ có nhiều text box và biểu đồ vẽ tay bằng hình (ví dụ biểu đồ cột dựng bằng hình chữ nhật, freeform và nhãn %). Phải phân loại từng khối: tiêu đề, nhãn chương, bullet, nguồn trích dẫn, footer. Biểu đồ vẽ tay thì chép nguyên nhóm hình sang và giữ tỉ lệ, rồi báo người dùng.
- Footer cũ kiểu "<tên hội nghị> · <số trang>" thì bỏ, vì template đã có số trang. Đây là chữ duy nhất được phép bỏ, và phải ghi lại trong báo cáo.
- Dòng trích dẫn nguồn (tác giả, tạp chí, năm) **phải giữ**, đặt ở placeholder chú thích hoặc nguồn.
- Chỉ dùng `normAutofit`/shrink thì vẫn tính là tràn. Phải chia slide.
- Quên speaker notes khi chia slide: notes đi theo phần nội dung tương ứng.

## Kiểm tra trước khi báo xong
- So tập chữ (chuẩn hoá khoảng trắng) của file cũ với file mới: không mất, không thêm. Chỉ ngoại lệ là các phần "(tiếp)"/"(2/2)" và footer đã bỏ.
- Số ảnh và bảng gốc đều có mặt.
- Đã xem ảnh render của **mọi** slide, không chỉ vài slide.
