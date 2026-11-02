# Bài 2 : Sự cố phiên làm việc trên hệ thống vận tải
## Phần 1: Phân tích
### 1. So sánh Browser Cache và Cookies
Browser Cache và Cookies đều lưu dữ liệu trên trình duyệt nhưng có mục đích và dung lượng khác nhau.
|Tiêu chí	|Browser Cache	|Cookies|
|---|---|---|
|Mục đích	|Lưu tài nguyên website|	Lưu thông tin trạng thái, định danh|
|Dung lượng	|Có thể từ MB đến GB	|Thường giới hạn khoảng 4 KB mỗi Cookie|
|Dữ liệu|	CSS, JavaScript, hình ảnh|	Session ID, Token, tùy chọn người dùng|
|Thời gian lưu|	Theo chính sách Cache|	Session hoặc Persistent|
|Gửi lên Server|	Không tự động gửi trong mỗi HTTP request|	Tự động gửi nếu phù hợp phạm vi Cookie|

**Phân tích sự đánh đổi:**
- Browser Cache có dung lượng lớn, phù hợp lưu trữ tài nguyên tĩnh để tăng tốc độ tải website.
- Cookies có dung lượng nhỏ, không phù hợp lưu trữ giỏ hàng lớn hoặc hình ảnh, nhưng phù hợp lưu thông tin định danh và trạng thái phiên làm việc.
- Không nên lưu toàn bộ dữ liệu giỏ hàng vào Cookie. Nên lưu giỏ hàng trên Server hoặc cơ sở dữ liệu, sau đó liên kết với tài khoản shipper.
### 2. Phân biệt Session Cookie và Persistent Cookie
|Session Cookie|	Persistent Cookie|
|---|---|
|Không có Max-Age hoặc Expires|	Có Max-Age hoặc Expires|
|Thường kết thúc khi phiên trình duyệt kết thúc|	Có thể tồn tại sau khi đóng trình duyệt|
|Phù hợp phiên làm việc tạm thời	|Phù hợp chức năng ghi nhớ đăng nhập|

**Kết luận:** Hệ thống vận tải nên sử dụng Persistent Cookie để duy trì trạng thái đăng nhập theo thời hạn được cấu hình.
## Phần 2: Triển khai
### 1. Cấu trúc Cookie cần thiết
Hệ thống cần sử dụng Persistent Cookie kết hợp các cờ bảo mật để duy trì trạng thái đăng nhập và bảo vệ Token.

Các thuộc tính cần thiết:
|Thuộc tính	|Chức năng|
|---|---|
|`Token`|	Chứa mã định danh phiên đăng nhập|
|`Max-Age`|	Quy định thời gian tồn tại của Cookie|
|`HttpOnly`|	Ngăn JavaScript phía Client đọc Cookie|
|`Secure`|	Chỉ gửi Cookie qua kết nối HTTPS|
|`SameSite`|	Hạn chế gửi Cookie trong các yêu cầu liên trang|
|`Path`|	Xác định phạm vi đường dẫn Cookie có hiệu lực|

### 2. Ví dụ cấu hình Persistent Cookie
Máy chủ thiết lập Cookie thông qua HTTP Response Header:
```http
Set-Cookie: session_token=<RANDOM_TOKEN>; Max-Age=604800; Path=/; HttpOnly; Secure; SameSite=Lax
```
**Giải thích:**
- `session_token`: Tên Cookie chứa Token định danh.
- `Max-Age=604800`: Cookie tồn tại trong 7 ngày.
- `Path=/`: Cookie có hiệu lực trên toàn bộ website.
- `HttpOnly`: JavaScript không thể truy cập Token thông qua document.cookie.
- `Secure`: Chỉ gửi Cookie qua HTTPS.
- `SameSite=Lax`: Hạn chế gửi Cookie trong một số yêu cầu liên trang, giúp giảm nguy cơ CSRF.

**Cấu hình đề xuất**
 Persistent Cookie có thời hạn 7 ngày và được bảo vệ bằng HttpOnly, Secure, SameSite.
### 3. Cơ chế hoạt động
1. Shipper đăng nhập thành công.
2. Server tạo Token ngẫu nhiên và gửi Cookie về trình duyệt.
3. Trình duyệt lưu Cookie theo thời hạn `Max-Age`.
4. Khi shipper đóng và mở lại trình duyệt, Cookie vẫn còn hiệu lực.
5. Khi shipper truy cập hệ thống, trình duyệt tự động gửi Cookie phù hợp lên Server.
6. Server kiểm tra Token và xác thực người dùng.
7. Nếu Token hợp lệ, hệ thống khôi phục phiên đăng nhập và dữ liệu giỏ hàng.
## Phần 3: Kiểm thử
### 1. Kiểm tra Cookie bằng F12
Thực hiện các bước:
1. Mở website vận tải.
2. Nhấn `F12` để mở Developer Tools.
3. Chọn tab Application.
4. Tìm mục Storage → Cookies.
5. Chọn tên miền website.
6. Kiểm tra các thuộc tính `Expires/Max-Age`, `HttpOnly` , `Secure` và `SameSite`.

Kết quả mong đợi:
- Cookie có thời hạn 7 ngày.
- Cột HttpOnly được đánh dấu.
- Cột Secure được đánh dấu.
- SameSite được thiết lập là Lax.

Không chia sẻ giá trị Token thật khi chụp ảnh nộp bài.
### 2. Xử lý lỗi giao diện cũ bằng Hard Reload
**Tình huống:** Shipper hoặc nhân viên kỹ thuật đã thay đổi CSS nhưng trình duyệt vẫn hiển thị màu sắc giao diện cũ.

Nguyên nhân: Trình duyệt có thể đang sử dụng file CSS được lưu trong Browser Cache.

Cách xử lý:
|Hệ điều hành|	Phím tắt|
|---|---|
|Windows	|`Ctrl + Shift + R`|
|Windows|	`Ctrl + F5`|
|macOS	|`Command + Shift + R`|

Các bước thực hiện:
1. Mở website cần kiểm tra.
2. Nhấn tổ hợp `Ctrl + Shift + R`.
3. Chờ trình duyệt tải lại tài nguyên.
4. Kiểm tra giao diện đã cập nhật CSS mới hay chưa.
Hard Reload yêu cầu tải lại tài nguyên và bỏ qua Cache thông thường, không cần xóa toàn bộ lịch sử duyệt web.
