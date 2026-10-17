# Software Requirements Specification (SRS)

## Hệ thống Backend CinemaPTIT

**Phiên bản:** 1.0  
**Ngày:** 23/09/2026  
**Phạm vi:** Backend REST API và các dịch vụ nền của hệ thống đặt vé xem phim

## 1. Mục đích và phạm vi

CinemaPTIT Backend cung cấp các dịch vụ cho ứng dụng đặt vé xem phim trực tuyến và hoạt động tại rạp. Backend chịu trách nhiệm xác thực người dùng, quản lý dữ liệu phim/rạp/suất chiếu, giữ ghế theo thời gian thực, tạo và thanh toán booking, phát hành vé điện tử, check-in, quản trị vận hành, báo cáo và hỗ trợ khách hàng bằng AI.

Tài liệu này đặc tả hành vi và các yêu cầu ở mức sản phẩm của backend hiện tại. Giao diện React, quy trình triển khai frontend và thiết kế chi tiết cơ sở dữ liệu không nằm trong phạm vi chính của tài liệu này.

## 2. Bối cảnh và kiến trúc hệ thống

### 2.1. Tác nhân và hệ thống ngoài

| Tác nhân/hệ thống | Trách nhiệm |
|---|---|
| Khách hàng (`USER`) | Duyệt phim, chọn suất/ghế, đặt vé, thanh toán, hủy vé, xem vé và lịch sử booking |
| Nhân viên (`STAFF`) | Bán vé tại quầy, mở/đóng ca, xem booking và check-in vé |
| Quản lý (`MANAGER`) | Quản lý hoạt động chi nhánh, nhân sự, kho, khuyến mãi và báo cáo được phân quyền |
| Quản trị viên (`ADMIN`) | Quản trị toàn hệ thống và cấu hình hệ thống |
| Frontend React | Gọi REST API, kết nối WebSocket và hiển thị kết quả cho người dùng |
| MySQL | Lưu trữ dữ liệu nghiệp vụ lâu dài |
| Redis | Giữ ghế, token/refresh token, OTP và lịch sử hội thoại AI |
| VNPay Sandbox | Tạo giao dịch, callback kết quả, truy vấn và hoàn tiền |
| Google OAuth2 | Đăng nhập bằng tài khoản Google |
| Gemini AI | Xử lý hội thoại và tra cứu dữ liệu qua các tool nghiệp vụ |
| Gmail SMTP | Gửi email xác nhận booking và OTP |

### 2.2. Kiến trúc logic

Backend được tổ chức theo các lớp:

- **Controller:** tiếp nhận HTTP request và trả response JSON.
- **Service:** thực hiện nghiệp vụ, kiểm tra quyền và điều phối giao dịch.
- **Repository/JPA:** truy cập MySQL.
- **Entity/DTO/Mapper:** mô hình dữ liệu và hợp đồng trao đổi API.
- **Config:** Security, JWT, Redis, WebSocket, mail, AI và OAuth2.
- **Event/Job:** xử lý bất đồng bộ và dọn booking hết hạn.

### 2.3. Thông số triển khai mặc định

- Base URL: `http://localhost:8081/api`.
- Java 21, Spring Boot 3.4, Maven.
- MySQL: database `cinema`.
- Redis: `localhost:6379`.
- WebSocket SockJS: `/api/ws` với broker `/topic` và application prefix `/app`.
- Mọi bí mật cấu hình phải được cấp qua biến môi trường hoặc file `.env`, không hard-code trong mã nguồn.

## 3. Quy tắc nghiệp vụ

### 3.1. Người dùng và quyền

- Tài khoản có trạng thái `ACTIVE` hoặc `BLOCKED`; tài khoản bị khóa không được xác thực như tài khoản hoạt động.
- Mật khẩu được lưu bằng BCrypt.
- Access token JWT có thời hạn 30 phút; refresh token có thời hạn 30 ngày.
- Logout phải thu hồi access token bằng `jti` trong Redis. Refresh phải kiểm tra token cũ còn hợp lệ và cấp cặp token mới.
- JWT dùng chữ ký HMAC-SHA512 và claim `role` để tạo quyền `ROLE_USER`, `ROLE_STAFF`, `ROLE_MANAGER`, `ROLE_ADMIN`.

### 3.2. Ghế, vé và suất chiếu

- Mỗi cặp `showtime_id` và `seat_id` chỉ có một ticket.
- Khi tạo suất chiếu, hệ thống tự tạo ticket cho toàn bộ ghế đang hoạt động trong phòng.
- Ticket có thể ở các trạng thái `AVAILABLE`, `HOLDING`, `BOOKED`, `CHECKED_IN` hoặc `EXPIRED`.
- Ghế được giữ trong Redis với khóa `seat_hold:{showtimeId}:{seatId}` và TTL 6 phút.
- Booking chờ thanh toán hết hạn phải được hủy, ticket trả về `AVAILABLE` và Redis lock được giải phóng.
- Mỗi ticket đã thanh toán phải có QR code duy nhất gắn với booking, ticket và ghế.

### 3.3. Thanh toán và hoàn tiền

- Booking online bắt đầu ở trạng thái `PENDING`; chỉ chuyển sang `COMPLETED` khi VNPay trả kết quả thành công.
- VNPay callback phải được kiểm tra chữ ký và mã kết quả trước khi cập nhật booking.
- Xử lý callback phải idempotent: booking đã `COMPLETED` hoặc `CANCELLED` không được cập nhật lặp.
- Booking tiền mặt tại quầy được đánh dấu đã thanh toán ngay sau khi tạo.
- Booking thẻ tại quầy giữ nguyên luồng thanh toán VNPay.
- Yêu cầu hoàn tiền và quyết định hoàn tiền chỉ dành cho vai trò được cấp quyền và phải được lưu vết.

## 4. Yêu cầu chức năng

### 4.1. Xác thực, tài khoản và phân quyền

| ID | Yêu cầu |
|---|---|
| FR-AUTH-01 | Hệ thống phải cho phép đăng ký tài khoản bằng thông tin cá nhân hợp lệ và kiểm tra dữ liệu trùng lặp. |
| FR-AUTH-02 | Hệ thống phải cho phép đăng nhập bằng username/email và mật khẩu, trả access token cùng refresh token. |
| FR-AUTH-03 | Hệ thống phải hỗ trợ đăng nhập OAuth2 bằng Google và điều hướng về frontend theo cấu hình. |
| FR-AUTH-04 | Hệ thống phải hỗ trợ logout, refresh token, đổi mật khẩu, quên mật khẩu và đặt lại mật khẩu bằng OTP email. |
| FR-AUTH-05 | Hệ thống phải cho phép người dùng xem/cập nhật thông tin cá nhân; admin/manager có thể quản lý người dùng theo quyền. |
| FR-AUTH-06 | Hệ thống phải từ chối request thiếu token, token hết hạn, token bị thu hồi hoặc không đủ role bằng lỗi HTTP phù hợp. |

### 4.2. Danh mục phim và rạp

| ID | Yêu cầu |
|---|---|
| FR-CAT-01 | Hệ thống phải cung cấp danh sách, chi tiết, phim đang chiếu và phim sắp chiếu. |
| FR-CAT-02 | Admin phải quản lý được phim, đạo diễn, thể loại và đánh giá phim. |
| FR-CAT-03 | Hệ thống phải cung cấp danh sách chi nhánh, phòng chiếu, ghế và sơ đồ ghế. |
| FR-CAT-04 | Admin/manager phải quản lý được phòng, ghế, trạng thái phòng và giá theo loại ghế. |
| FR-CAT-05 | Hệ thống phải cho phép tra cứu suất chiếu theo ngày, phim, phòng và chi nhánh. |
| FR-CAT-06 | Admin/manager phải tạo, cập nhật, xóa hoặc hủy suất chiếu theo quy tắc lịch chiếu. |

### 4.3. Đặt vé và giữ ghế

| ID | Yêu cầu |
|---|---|
| FR-BOOK-01 | Khách hàng đã xác thực phải tạo booking bằng suất chiếu, danh sách ghế, món ăn tùy chọn và thông tin khuyến mãi nếu có. |
| FR-BOOK-02 | Hệ thống phải kiểm tra ghế tồn tại, đang hoạt động, chưa được đặt và không bị người khác giữ. |
| FR-BOOK-03 | Hệ thống phải giữ ghế nguyên tử trong Redis trong 6 phút và phát trạng thái ghế qua WebSocket. |
| FR-BOOK-04 | Hệ thống phải cho phép khách hàng xem danh sách booking, chi tiết booking, hủy booking hợp lệ và gửi yêu cầu hoàn tiền. |
| FR-BOOK-05 | Staff phải tạo được booking tại quầy bằng CASH; customer account có thể được tìm hoặc tạo theo email/số điện thoại. |
| FR-BOOK-06 | Hệ thống phải hỗ trợ thanh toán CARD tại quầy thông qua luồng VNPay. |
| FR-BOOK-07 | Hệ thống phải áp dụng và xác thực mã khuyến mãi theo thời hạn, điều kiện và giá trị đơn hàng. |
| FR-BOOK-08 | Hệ thống phải gửi sự kiện thanh toán thành công để tạo email xác nhận và cập nhật trạng thái ticket. |

### 4.4. Thanh toán VNPay

| ID | Yêu cầu |
|---|---|
| FR-PAY-01 | Hệ thống phải tạo URL thanh toán VNPay có tham số giao dịch và chữ ký HMAC-SHA512 hợp lệ. |
| FR-PAY-02 | Hệ thống phải xử lý return URL và IPN từ VNPay, xác minh chữ ký, mã phản hồi và mã giao dịch. |
| FR-PAY-03 | Khi thanh toán thành công, hệ thống phải chuyển booking sang `COMPLETED`, payment sang `PAID`, ticket sang `BOOKED` và giải phóng seat lock. |
| FR-PAY-04 | Khi thanh toán thất bại hoặc hết hạn, hệ thống phải hủy booking và trả ticket về trạng thái có thể bán. |
| FR-PAY-05 | Hệ thống phải hỗ trợ truy vấn giao dịch và yêu cầu hoàn tiền theo quyền được cấp. |

### 4.5. Vé điện tử và check-in

| ID | Yêu cầu |
|---|---|
| FR-TICKET-01 | Hệ thống phải cung cấp ticket theo showtime, chi tiết ticket và trạng thái ghế/vé. |
| FR-TICKET-02 | Hệ thống phải tạo QR code cho ticket đã thanh toán. |
| FR-TICKET-03 | Staff/manager/admin phải check-in bằng chuỗi QR hoặc ảnh QR. |
| FR-TICKET-04 | Chỉ ticket ở trạng thái `BOOKED` mới được check-in; check-in thành công phải lưu thời điểm và chuyển sang `CHECKED_IN`. |
| FR-TICKET-05 | Hệ thống phải từ chối QR không hợp lệ, ticket không tồn tại, ticket đã dùng hoặc ticket chưa thanh toán. |

### 4.6. Vận hành, đồ ăn và thông báo

| ID | Yêu cầu |
|---|---|
| FR-OPS-01 | Staff phải mở ca, đóng ca, xem ca hiện tại và lịch sử ca của mình. |
| FR-OPS-02 | Manager phải quản lý lịch làm việc và xem thông tin nhân viên thuộc phạm vi được phân công. |
| FR-OPS-03 | Admin/manager phải quản lý món ăn, tồn kho, điều chỉnh nhập/xuất và lịch sử giao dịch kho. |
| FR-OPS-04 | Người dùng đã đăng nhập phải xem thông báo và đánh dấu thông báo đã đọc. |
| FR-OPS-05 | Hệ thống phải ghi audit log cho các hoạt động quản trị và vận hành cần truy vết. |
| FR-OPS-06 | Admin phải xem và cập nhật các system setting được cho phép. |

### 4.7. Báo cáo và AI chatbot

| ID | Yêu cầu |
|---|---|
| FR-REPORT-01 | Admin/manager phải xem dashboard, báo cáo doanh thu và báo cáo vận hành theo bộ lọc thời gian/phạm vi. |
| FR-REPORT-02 | Hệ thống phải cho phép xuất báo cáo doanh thu và vận hành dưới dạng XLSX. |
| FR-AI-01 | Hệ thống phải cung cấp endpoint chatbot công khai để trả lời câu hỏi về phim, suất chiếu, rạp, giá vé và menu. |
| FR-AI-02 | Chatbot phải lưu lịch sử hội thoại theo `sessionId` trong Redis. |
| FR-AI-03 | Chatbot phải tạo được liên kết đặt vé dựa trên thông tin người dùng cung cấp và không được tự xác nhận booking thay người dùng. |

## 5. Hợp đồng API mức cao

Tất cả endpoint REST nằm dưới prefix `/api`. Request/response sử dụng JSON trừ endpoint upload ảnh QR và endpoint xuất file XLSX. API phải trả lỗi theo một cấu trúc thống nhất có mã lỗi và thông báo; lỗi xác thực trả `401`, lỗi thiếu quyền trả `403`, dữ liệu không tồn tại trả `404`, dữ liệu không hợp lệ hoặc xung đột nghiệp vụ trả mã HTTP phù hợp.

| Nhóm | Prefix chính | Quyền mặc định |
|---|---|---|
| Auth | `/sign-up`, `/auth/**`, `/oauth2/**` | Public hoặc tài khoản hiện tại tùy endpoint |
| Catalog | `/movie/**`, `/genre/**`, `/director/**`, `/branch/**`, `/room/**` | GET public; thay đổi cần quyền |
| Showtime/seat/ticket | `/showtime/**`, `/seat/**`, `/ticket/**` | Tra cứu public; quản trị/check-in cần quyền |
| Booking | `/booking/**` | USER đã đăng nhập; staff/manager/admin theo nghiệp vụ |
| Payment | `/v1/vnpay/**` | Tạo giao dịch cần xác thực; callback public có xác minh chữ ký |
| Operations | `/staff/**`, `/food/**`, `/promotion/**`, `/notifications/**` | Theo role |
| Admin/report | `/admin/**` | ADMIN hoặc MANAGER theo endpoint |
| Chat/cleanup | `/chat`, `/v1/cleanup/**` | Public, phải có kiểm soát lạm dụng khi triển khai thật |

## 6. WebSocket và đồng bộ thời gian thực

- Client kết nối SockJS tại `/api/ws`.
- Client subscribe `/topic/showtime/{showtimeId}/seats`.
- Backend broadcast khi ghế chuyển giữa `AVAILABLE`, `HOLDING` và `BOOKED`.
- Trạng thái broadcast phải phản ánh kết quả lock/transaction thực tế; không được thông báo ghế đã đặt nếu giao dịch cập nhật thất bại.

## 7. Yêu cầu phi chức năng

| ID | Yêu cầu |
|---|---|
| NFR-01 | Bảo mật: mật khẩu phải BCrypt, JWT phải ký HS512, bí mật DB/VNPay/mail/AI phải lấy từ biến môi trường. |
| NFR-02 | Phân quyền: kiểm tra ở cả URL level và method level; mọi API thay đổi dữ liệu phải yêu cầu xác thực phù hợp. |
| NFR-03 | Nhất quán: thao tác đặt vé, thanh toán, lock ghế và giải phóng ghế phải có xử lý giao dịch và idempotency phù hợp. |
| NFR-04 | Tin cậy: lỗi tích hợp VNPay, mail, Redis hoặc AI phải trả lỗi có kiểm soát và không làm lộ secret/stack trace cho client. |
| NFR-05 | Hiệu năng: truy vấn danh sách phải hỗ trợ phân trang/lọc khi API cung cấp; lock ghế phải phản hồi đủ nhanh cho trải nghiệm chọn ghế. |
| NFR-06 | Khả năng bảo trì: giữ tách biệt Controller-Service-Repository, trao đổi qua DTO và xử lý exception tập trung. |
| NFR-07 | Khả năng quan sát: các thay đổi quản trị, giao dịch và lỗi quan trọng phải có log/audit phù hợp. |
| NFR-08 | Tương thích: API phải phục vụ frontend React qua CORS cho các origin cấu hình của môi trường phát triển. |

## 8. Mô hình dữ liệu nghiệp vụ chính

Các thực thể cốt lõi gồm:

- `User`, `Branch`, `Room`, `Seat`.
- `Movie`, `Director`, `Genre`, `Showtime`.
- `Ticket`, `Booking`, `BookingFood`, `Food`.
- `Promotion`, `SeatTypePrice`, `Notification`, `AuditLog`.
- Các thực thể hỗ trợ staff schedule, inventory, payment/refund và system setting.

Quan hệ quan trọng: một branch có nhiều room; một room có nhiều seat và showtime; một movie có nhiều showtime; một showtime có nhiều ticket; một booking thuộc về user và showtime, chứa nhiều ticket và món ăn.

## 9. Luồng nghiệp vụ chính

### 9.1. Khách hàng mua vé online

1. Client lấy danh sách phim, chi nhánh, suất chiếu và trạng thái ghế.
2. Client gửi yêu cầu tạo booking với các ghế đã chọn.
3. Backend kiểm tra và lock từng ghế trong Redis trong 6 phút.
4. Backend tạo booking `PENDING`, ticket `HOLDING` và trả booking ID.
5. Client yêu cầu URL VNPay và điều hướng người dùng sang cổng thanh toán.
6. Backend nhận return/IPN, xác minh chữ ký và cập nhật trạng thái một lần.
7. Khi thành công, hệ thống tạo QR, broadcast ghế `BOOKED`, xóa lock và gửi email xác nhận.

### 9.2. Nhân viên bán vé tại quầy

1. Staff mở ca và chọn phim, suất chiếu, ghế, món ăn.
2. Backend kiểm tra ghế không bị giữ/đặt.
3. Với CASH, booking hoàn tất và QR được tạo ngay.
4. Với CARD, booking chờ VNPay xử lý như thanh toán online.
5. Backend broadcast trạng thái ghế và gửi xác nhận sau khi hoàn tất.

### 9.3. Check-in

1. Staff/manager/admin gửi QR dạng text hoặc ảnh.
2. Backend giải mã/tìm ticket và kiểm tra trạng thái `BOOKED`.
3. Nếu hợp lệ, backend lưu `checkedInAt` và chuyển ticket sang `CHECKED_IN`.

## 10. Xử lý lỗi và bảo vệ dữ liệu

- Lỗi nghiệp vụ phải sử dụng mã lỗi nội bộ thống nhất và được chuyển thành response API ổn định.
- Không trả mật khẩu, refresh token, secret key, thông tin kết nối hoặc stack trace cho client.
- Callback thanh toán phải chống replay bằng kiểm tra mã giao dịch, chữ ký và trạng thái hiện tại của booking.
- Các thao tác cập nhật dữ liệu phải kiểm tra ownership hoặc phạm vi chi nhánh khi role là `STAFF`/`MANAGER`.
- Endpoint cleanup phải chỉ xử lý booking hết hạn và phải giải phóng ticket/lock theo cùng một quy tắc nghiệp vụ.

## 11. Môi trường và tiêu chí chấp nhận tổng quát

### 11.1. Phụ thuộc bắt buộc

- Java 21 và Maven Wrapper.
- MySQL database `cinema`.
- Redis trên cổng `6379`.
- Các biến môi trường cho JWT, mail, Google OAuth2, Gemini và VNPay.

### 11.2. Tiêu chí chấp nhận

- Ứng dụng khởi động thành công khi các phụ thuộc bắt buộc và biến môi trường hợp lệ.
- Người dùng có thể đăng ký/đăng nhập, duyệt suất chiếu, lock ghế, tạo booking và hoàn tất thanh toán sandbox.
- Ghế đã bị người khác giữ không thể bị booking đồng thời.
- Booking hết hạn tự giải phóng ghế.
- Ticket hợp lệ check-in được một lần; ticket đã check-in hoặc chưa thanh toán bị từ chối.
- Endpoint yêu cầu quyền phải từ chối đúng role không hợp lệ.
- Các callback VNPay lặp lại không tạo booking hoặc email trùng.
- Báo cáo, thông báo, audit log và xuất XLSX hoạt động theo quyền được cấp.
