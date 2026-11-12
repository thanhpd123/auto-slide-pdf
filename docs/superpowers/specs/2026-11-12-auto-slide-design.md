# Auto Slide — Thiết kế bản đầu tiên (chế độ "Làm đẹp slide cũ")

Ngày: 2026-11-12 · Repo: https://github.com/thanhpd123/auto-slide-pdf

## 1. Ý tưởng và lý do

AI (Claude) đã làm được slide đẹp, nhưng chỉ khi người dùng biết viết prompt tốt và biết cách đưa file cho AI. Chủ dự án đã tự làm việc này cho bác sĩ: lấy một file slide cũ có nội dung đúng nhưng xấu (khoảng 30 slide), một template hội nghị đẹp (11 layout), viết prompt 10 quy tắc, và nhận về bản mới khoảng 31 slide, mang giao diện mới, có ảnh minh hoạ kèm `credits.txt`. Bộ mẫu này là **tài liệu độc quyền của bác sĩ**, nằm ngoài repo (`D:\File\ppt`) và không bao giờ được đưa lên git.

Người dùng mục tiêu là bác sĩ, và về sau là bất kỳ ai cần thuyết trình. Họ biết rõ mình muốn nói gì và thường có tài liệu, nhưng **không biết gì về AI hay prompt**. Sản phẩm là một trang web gói toàn bộ kinh nghiệm đó lại: người dùng chỉ điền vài thứ, tải file lên, bấm một nút và nhận file.

## 2. Mục tiêu

**Bản đầu tiên (phạm vi tài liệu này): chế độ B, "Làm đẹp slide cũ".**

- Người dùng tải lên file `.pptx` cũ, chọn template, chọn ngôn ngữ, bấm "Tạo slide" và nhận về `.pptx` mới (PDF là tuỳ chọn).
- Chất lượng đầu ra ngang bản làm tay, tức là làm đúng 10 quy tắc của prompt gốc:
  1. Lấy template làm gốc, giữ nguyên theme, slide master, layout, font, màu.
  2. Xoá các slide mẫu của template.
  3. Với mỗi slide cũ, chọn layout phù hợp nhất (title, section, nội dung, 2 cột, chữ + ảnh, bảng, biểu đồ, closing...) rồi đổ nội dung vào placeholder.
  4. **Giữ nguyên chữ, thứ tự slide, hình ảnh, bảng và speaker notes. Không viết lại nội dung.**
  5. Slide nhiều chữ thì chia thành 2 slide, không thu nhỏ chữ và không để tràn.
  6. Không sửa file gốc.
  7. Render từng slide ra ảnh để kiểm tra tràn chữ và lệch khung, rồi sửa.
  8. Slide nào cần minh hoạ thì tìm ảnh có giấy phép tự do, kiểm tra ảnh đúng chủ đề, chèn vào khung ảnh của layout, crop cho vừa và không làm méo ảnh.
  9. Ghi danh sách nguồn ảnh: tên ảnh, link gốc, tác giả, giấy phép.
  10. Không chèn ảnh vào slide nhiều chữ hoặc slide có bảng/biểu đồ.
- Cuối cùng báo cho người dùng biết slide nào bị chia và slide nào đổi bố cục.
- Giao diện web hoàn toàn bằng **tiếng Việt**.

**Bản sau (không làm trong bản đầu, nhưng kiến trúc phải chừa chỗ):** chế độ A, "Làm slide mới". Người dùng nhập chủ đề, đối tượng nghe, thời lượng, tài liệu tham khảo (PDF, bài báo), và ngôn ngữ slide do người dùng chọn. AI sẽ viết dàn ý và nội dung **dựa trên tài liệu**.

## 3. Không làm (non-goals) trong bản đầu

- Chế độ A, "Làm slide mới".
- Tài khoản, đăng nhập, lịch sử.
- Thu tiền của người dùng. Người dùng tự trả tiền AI bằng key của mình (xem mục 5).
- Chỉnh sửa slide trên web. Người dùng tải `.pptx` về và tự sửa trong PowerPoint hoặc Google Slides.
- Hỗ trợ đầu vào khác `.pptx`, ví dụ `.key` hay Google Slides link. `.pdf` slide cũ thì để sau.

## 4. Điều tuyệt đối không được xảy ra (anti-goals)

- **AI tự sửa, bịa hoặc "làm đẹp" câu chữ, số liệu y khoa hay trích dẫn.** Ở chế độ B, chữ trên slide mới phải trùng khớp với chữ trên slide cũ. Phần mềm phải **tự kiểm chứng bằng code**: so tập chữ cũ với tập chữ mới (đã chuẩn hoá khoảng trắng), và báo lỗi nếu mất hoặc thêm chữ. Ngoại lệ duy nhất là những phần chữ được lặp lại khi chia slide, ví dụ tiêu đề có thêm "(tiếp)" hoặc "(2/2)".
- Chữ tràn khung, chữ bị thu nhỏ đến mức khó đọc, ảnh bị méo.
- Dùng ảnh hoặc template không rõ giấy phép.
- Giữ tài liệu của người dùng lâu dài trên server.
- Đưa slide, template, ảnh hay bất cứ chữ/số liệu nào của bác sĩ vào repo hoặc lên GitHub.
- Người dùng gặp thuật ngữ kỹ thuật ("prompt", "token", "API") mà không có lời giải thích bằng tiếng Việt dễ hiểu.

## 5. Ràng buộc đã chốt

### 5.1 Nền tảng
- **Trang web, mở bằng trình duyệt.** Giao diện tiếng Việt.
- Bán cho bất kỳ ai dùng.

### 5.2 AI và chi phí
- **Mặc định miễn phí:** dùng key Gemini API (free tier) của chủ dự án, chạy model **Gemma 4 31B**. Builder cần kiểm tra tên model chính xác trên Gemini API và giới hạn tốc độ của free tier.
- **Bản tốt hơn:** người dùng **tự mang key** (trước tiên là key Claude, sau này có thể thêm key khác). Phần mềm phải có **trang hướng dẫn từng bước bằng tiếng Việt** gồm: tạo tài khoản, nạp tiền (link thẳng đến trang thanh toán), tạo key (link thẳng đến trang lấy key), dán key vào phần mềm. Chỉ lưu key ở trình duyệt của người dùng, không lưu trên server. Key chỉ được gửi kèm từng yêu cầu tạo slide.
- Phải có giới hạn lượt dùng cho bản miễn phí để key chung của chủ dự án không bị dùng cạn hoặc bị lạm dụng.

### 5.3 Quyền riêng tư
- Gemini free tier có thể dùng dữ liệu gửi lên để cải thiện model của Google. Builder cần kiểm tra lại điều khoản hiện hành. Màn hình chọn AI phải **cảnh báo rõ** điều này, và khuyên không đưa thông tin nhận dạng bệnh nhân khi dùng bản miễn phí.
- File tải lên và file kết quả chỉ giữ tạm, rồi tự xoá sau một thời gian ngắn. Builder tự chọn thời gian, đề xuất ≤ 24 giờ.

### 5.4 Template
Kết hợp ba nguồn:
1. **Thư viện tự làm:** vài template "chuẩn hội nghị y khoa" do chủ dự án tự thiết kế và sở hữu. **Không dùng template của hội nghị trong bộ mẫu**, vì đó là tài liệu độc quyền.
2. **Template từ nguồn có giấy phép cho phép dùng thương mại và phân phối lại**, có ghi nguồn. **Không crawl** template từ các trang cấm phân phối lại.
3. **Dẫn link:** trang gợi ý các trang template đẹp, kèm hướng dẫn tải về rồi upload lên. Người dùng tự tải về, nên tuân thủ điều khoản của trang đó.
- Người dùng luôn có thể tải lên template `.pptx` hoặc `.potx` của riêng mình.

### 5.5 Ảnh minh hoạ
- Nguồn: Wikimedia Commons, như cách đã làm trong `images/credits.txt`. Chỉ lấy ảnh public domain hoặc CC BY / CC BY-SA. Ghi nguồn trong file `credits.txt` đi kèm và một dòng nguồn nhỏ dưới ảnh trên slide.
- Người dùng có thể tắt tính năng tự thêm ảnh.

## 6. Cách làm — phần quyết định của tôi (chủ dự án chưa duyệt từng điểm)

### 6.1 Nguyên tắc chính: code làm việc máy móc, AI chỉ làm việc cần suy nghĩ
Khi làm bằng tay, Claude tự viết code, tự render và tự sửa. Model nhỏ miễn phí như Gemma không làm được chuỗi việc dài như vậy một cách ổn định. Vì vậy:

| Việc | Ai làm |
|---|---|
| Đọc slide cũ: chữ, thứ tự đoạn, cấp bullet, ảnh, bảng, notes | Code |
| Đọc template: danh sách layout, placeholder, kích thước khung | Code |
| Xoá slide mẫu, tạo slide mới từ layout, đổ chữ, ảnh, bảng, notes | Code |
| Đo chữ có vừa khung không (tính theo font/cỡ chữ của template) | Code |
| Render ra ảnh / PDF (LibreOffice headless) | Code |
| Crop ảnh vừa khung, không méo | Code |
| Kiểm chứng "chữ cũ = chữ mới" | Code |
| Hiểu vai trò của từng khối chữ trên slide cũ (tiêu đề, nhãn section, bullet, chú thích nguồn, số liệu nổi bật, footer cần bỏ) | AI |
| Chọn layout cho từng slide | AI, kèm luật dự phòng bằng code |
| Quyết định chia slide ở chỗ nào cho hợp ý | AI, kèm luật dự phòng bằng code |
| Slide nào cần minh hoạ, tìm ảnh bằng từ khoá gì | AI |
| Xem ảnh tìm được có đúng chủ đề không | AI có khả năng nhìn ảnh, hoặc bỏ qua nếu model không hỗ trợ |

AI luôn trả về **JSON theo schema cố định** chứ không trả về code. Phần mềm kiểm tra JSON và gọi lại AI nếu sai. Câu chữ của slide **không bao giờ đi qua AI để được viết lại**. AI chỉ trả về *mã số* của các khối chữ và cách xếp chúng, còn code mới là bên chép chữ gốc vào slide mới. Đây là cách bảo đảm anti-goal số 1.

### 6.2 Luồng xử lý (chế độ B)
1. **Tách nội dung:** code đọc slide cũ thành danh sách khối, mỗi khối có mã số, loại (chữ / ảnh / bảng / nhóm hình), vị trí, cỡ chữ và nội dung.
2. **Phân tích:** AI nhận danh sách khối của từng slide (không có toàn bộ file) cùng danh sách layout của template. AI trả về: layout chọn, khối nào vào placeholder nào, khối nào bỏ (ví dụ footer "<tên hội nghị> · <số trang>" vì template đã có số trang), slide có cần chia không và chia sau khối nào, có cần ảnh không và từ khoá tìm ảnh.
3. **Dựng slide:** code tạo file mới từ template rồi đổ nội dung vào.
4. **Kiểm tra:** code đo tràn chữ. Slide nào tràn thì chia (theo gợi ý của AI hoặc luật dự phòng) rồi dựng lại. Render ra ảnh để làm ảnh xem trước và để kiểm tra lần cuối.
5. **Ảnh minh hoạ:** tìm trên Wikimedia Commons, lọc theo giấy phép, AI xem ảnh, code crop rồi chèn, và ghi `credits.txt`.
6. **Kiểm chứng và trả kết quả:** so chữ cũ với chữ mới. Trả về `.pptx`, PDF (tuỳ chọn), `credits.txt`, kèm báo cáo tiếng Việt về slide nào bị chia hoặc đổi bố cục, và cảnh báo nếu có.

### 6.3 Điểm khó đã biết
Slide cũ thường **không dùng placeholder** mà vẽ tay bằng nhiều text box và hình. Ví dụ hay gặp là biểu đồ cột vẽ bằng hình chữ nhật + freeform + nhãn %. Bản đầu sẽ xử lý như sau:
- Text box → tách thành khối chữ như bình thường.
- Ảnh, bảng thật → chép sang.
- **"Biểu đồ vẽ tay"** và các nhóm hình phức tạp mà không nhận ra được: **chép nguyên nhóm hình sang** và đặt vào vùng nội dung của layout (giữ tỉ lệ), không cố dựng lại thành biểu đồ thật. Báo cho người dùng biết những slide này để họ tự xem lại. Dựng lại thành biểu đồ thật để dành cho bản sau.

### 6.4 Công nghệ (đề xuất)
- **Backend:** Python (FastAPI) + `python-pptx`, đây là bộ đã chạy được trong quy trình làm tay. Render/PDF bằng **LibreOffice headless**.
- **Frontend:** trang web đơn giản do chính backend phục vụ, tiếng Việt, không cần framework nặng.
- **Chạy nền:** mỗi lần tạo slide mất vài phút, nên chạy dạng hàng đợi. Trang web hiện tiến độ theo từng bước ("Đang đọc slide cũ… Đang chọn bố cục slide 7/30…").
- **Triển khai:** Docker, chạy trên server có cài được LibreOffice. Vercel kiểu serverless không phù hợp với LibreOffice và với tác vụ chạy lâu.
- **Lớp AI:** viết một interface chung, có hai bản cài đặt là Gemini/Gemma và Claude, để sau này dễ thêm model khác.

## 7. Màn hình (bản đầu)
Chủ dự án muốn xem mockup sau, nên ở đây chỉ mô tả bằng chữ:
1. **Trang chủ:** giới thiệu ngắn và hai lựa chọn. "Làm đẹp slide cũ" bấm được. "Làm slide mới" ghi "Sắp có".
2. **Bước 1 – Tải slide cũ** (`.pptx`).
3. **Bước 2 – Chọn template:** thư viện có sẵn, tự tải lên, hoặc link "Tìm template đẹp ở đâu?".
4. **Bước 3 – Tuỳ chọn:** ngôn ngữ của các chữ phần mềm tự thêm như "(tiếp)" (chế độ B không dịch nội dung); bật/tắt ảnh minh hoạ; xuất thêm PDF.
5. **Bước 4 – Chọn AI:** "Miễn phí (Gemma)" kèm cảnh báo quyền riêng tư, hoặc "Dùng key Claude của tôi" kèm link "Hướng dẫn lấy key".
6. **Bấm "Tạo slide"** → trang tiến độ → trang kết quả gồm ảnh xem trước từng slide, nút tải về, báo cáo slide bị chia hoặc đổi bố cục, danh sách nguồn ảnh.

## 8. Thế nào là thành công
- Chạy trên máy với bộ mẫu riêng của chủ dự án (đọc trực tiếp từ `D:\File\ppt`, không chép vào repo) cho ra kết quả mà chủ dự án đánh giá ngang hoặc gần bằng bản làm tay, **bằng cả Gemma miễn phí lẫn Claude**.
- Test tự động trong repo chỉ dùng file .pptx tự tạo, không chứa nội dung của bác sĩ.
- Kiểm chứng chữ đạt 100%: không mất, không thêm, không đổi chữ.
- Không slide nào tràn chữ khi mở bằng PowerPoint.
- Một người không biết gì về AI dùng được từ đầu đến cuối mà không cần hỏi.

## 9. Để builder tự quyết
- Cấu trúc thư mục, thư viện cụ thể cho hàng đợi, lưu trữ tạm và giới hạn lượt dùng.
- Cách đo tràn chữ cụ thể: tính bằng font metrics, hoặc dựa vào ảnh render, hoặc kết hợp cả hai.
- Schema JSON chi tiết cho AI và nội dung prompt.
- Luật dự phòng chọn layout và chia slide khi AI trả lời sai.
- Thiết kế giao diện chi tiết, sẽ duyệt mockup với chủ dự án sau.
