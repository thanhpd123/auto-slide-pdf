# Auto Slide — quy tắc cho Claude

## Dự án là gì
Trang web (giao diện **tiếng Việt**) giúp người **không biết gì về AI hay prompt**, trước tiên là bác sĩ, làm slide đẹp. Người dùng điền vài thứ, tải file lên, bấm một nút và nhận `.pptx`. Phần mềm tự ghép prompt và tự chạy AI.

- Thiết kế đầy đủ: `docs/superpowers/specs/2026-11-12-auto-slide-design.md`. **Đọc file này trước khi làm bất cứ việc gì.**
- Bộ mẫu riêng của chủ dự án (slide cũ, template, kết quả làm tay, prompt gốc) nằm **ngoài repo** ở `D:\File\ppt`. Đây là tài liệu độc quyền của bác sĩ, chỉ được đọc trên máy để thử nghiệm (xem quy tắc 8).
- Quy trình làm đẹp slide: skill `restyling-pptx-with-template` (trong `.claude/skills/`).

## Đã chốt với chủ dự án
- Hai chế độ: **B "Làm đẹp slide cũ" làm trước**, A "Làm slide mới" (chủ đề, đối tượng nghe, thời lượng, tài liệu tham khảo) làm sau.
- Ngôn ngữ slide do người dùng chọn. Giao diện web bằng tiếng Việt.
- Bán cho bất kỳ ai. **Người dùng tự trả tiền AI bằng key của mình**, phần mềm hướng dẫn đăng ký, dẫn link thẳng tới trang thanh toán và trang lấy key. Mặc định có bản miễn phí dùng Gemini API free của chủ dự án, model **Gemma 4 31B**.
- Template: kết hợp (1) thư viện tự làm, (2) nguồn có giấy phép cho phép thương mại và phân phối lại, (3) dẫn link để người dùng tự tải. **Không crawl** template từ các trang cấm phân phối lại.
- Chủ dự án muốn xem mockup giao diện **sau**, đừng tự đề xuất lại lúc này.

## Quy tắc bắt buộc
1. **Không bao giờ để AI viết lại, bịa hoặc sửa câu chữ, số liệu y khoa hay trích dẫn của người dùng.** Ở chế độ B, AI chỉ trả về JSON (mã khối, layout, chỗ chia). Code mới là bên chép chữ gốc. Luôn có bước code so khớp chữ cũ với chữ mới.
2. **Code làm việc máy móc, AI làm việc cần suy nghĩ.** Code lo đọc, đổ, đo tràn, render, crop. AI lo hiểu vai trò khối chữ, chọn layout, chọn chỗ chia, gợi ý ảnh. Mọi tính năng phải chạy được với model nhỏ miễn phí (Gemma), không chỉ với Claude.
3. Không thu nhỏ chữ để nhét vừa khung. Nếu tràn thì chia slide.
4. Ảnh và template phải rõ giấy phép, kèm `credits.txt`.
5. Không lưu tài liệu người dùng lâu dài. Không lưu key AI trên server.
6. Chữ người dùng nhìn thấy phải là tiếng Việt dễ hiểu, không dùng thuật ngữ kỹ thuật trần như "prompt", "token", "API" mà không giải thích.
7. Không sửa các file gốc trong `D:\File\ppt`.
8. **Tuyệt đối không đưa bất cứ thứ gì của bác sĩ lên git hay GitHub**: slide, template của hội nghị, ảnh, kết quả, prompt có tên file, và cả **tên bài, tên hội nghị, tên bệnh viện, chữ hoặc số liệu trích từ slide** trong code, tài liệu, test, commit message. Test và ví dụ chỉ dùng dữ liệu tự tạo. Muốn thử bằng bộ mẫu thật thì đọc trực tiếp từ `D:\File\ppt` hoặc biến môi trường, không chép vào repo.

## Cách làm việc với chủ dự án
- Trả lời bằng **tiếng Việt**, ngắn gọn, ít thuật ngữ.
- Chủ dự án hay gõ tiếng Việt không dấu, nên hiểu theo ngữ cảnh.
- Commit thì được. **Hỏi trước khi push** lên GitHub (`origin` = https://github.com/thanhpd123/auto-slide-pdf.git).
