# Thiết kế API Backend CinemaPTIT

**Phiên bản:** 1.0  
**Ngày:** 23/09/2026  
**Base URL mặc định:** `http://localhost:8081/api`

## 1. Quy ước chung

### 1.1. HTTP và định dạng dữ liệu

- REST API sử dụng HTTP/JSON.
- Header bắt buộc với API bảo vệ:

```http
Authorization: Bearer <access-token>
Content-Type: application/json
```

- Các endpoint upload ảnh QR dùng `multipart/form-data`.
- Các endpoint export trả file XLSX với `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.
- ID dùng kiểu số nguyên dài trong các URL, ví dụ `/movie/{id}`.
- Thời gian nên truyền theo ISO-8601, ví dụ `2026-09-23T19:30:00`.
- Giá tiền dùng số thập phân, không truyền ký hiệu tiền tệ trong JSON.

### 1.2. Response thành công

Backend trả DTO tương ứng với từng nghiệp vụ. Với response có wrapper, cấu trúc khuyến nghị:

```json
{
  "code": 200,
  "message": "Success",
  "result": {}
}
```

Danh sách có thể trả trực tiếp mảng hoặc wrapper chứa `result`; client phải dựa vào DTO thực tế của endpoint thay vì giả định mọi response có cùng hình dạng.

### 1.3. Response lỗi

```json
{
  "code": "TICKET_NOT_AVAILABLE",
  "message": "The selected seat is no longer available",
  "timestamp": "2026-09-23T19:30:00Z"
}
```

Mã HTTP chính:

| HTTP | Ý nghĩa |
|---|---|
| `400` | Request sai định dạng hoặc vi phạm quy tắc nghiệp vụ |
| `401` | Chưa xác thực, token hết hạn hoặc token bị thu hồi |
| `403` | Đã xác thực nhưng không đủ role |
| `404` | Không tìm thấy tài nguyên |
| `409` | Xung đột, thường gặp khi ghế đã bị giữ/đặt |
| `500` | Lỗi hệ thống không dự kiến |

Không trả password, secret key, stack trace hoặc thông tin kết nối trong response lỗi.

## 2. Xác thực và phân quyền

### 2.1. Role

| Role | Phạm vi |
|---|---|
| `USER` | Đặt vé, xem thông tin cá nhân, booking và notification |
| `STAFF` | Bán vé tại quầy, mở/đóng ca, check-in và nghiệp vụ được phân công |
| `MANAGER` | Quản lý chi nhánh, nhân viên, kho, booking và báo cáo |
| `ADMIN` | Toàn quyền quản trị hệ thống |

### 2.2. Endpoint auth

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/sign-up` | Public | Đăng ký tài khoản |
| `POST` | `/auth/login` | Public | Đăng nhập, trả access/refresh token |
| `POST` | `/auth/logout` | Authenticated | Thu hồi access token hiện tại |
| `POST` | `/auth/refresh` | Public + refresh token | Cấp cặp token mới |
| `POST` | `/auth/change-password` | Authenticated | Đổi mật khẩu |
| `POST` | `/auth/forgot-password` | Public | Gửi OTP qua email |
| `POST` | `/auth/reset-password` | Public | Đặt lại mật khẩu bằng OTP |
| `GET` | `/oauth2/authorization/google` | Public | Bắt đầu Google OAuth2 |

Ví dụ đăng nhập:

```json
{
  "username": "customer@example.com",
  "password": "secret"
}
```

```json
{
  "result": {
    "accessToken": "<jwt>",
    "refreshToken": "<refresh-token>"
  }
}
```

## 3. API danh mục công khai

### 3.1. Phim, đạo diễn và thể loại

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/movie` | Public | Danh sách phim, hỗ trợ bộ lọc/phân trang theo DTO |
| `GET` | `/movie/upcoming` | Public | Phim sắp chiếu |
| `GET` | `/movie/now-showing` | Public | Phim đang chiếu |
| `GET` | `/movie/{id}` | Public | Chi tiết phim |
| `POST` | `/movie` | `ADMIN` | Tạo phim |
| `PUT` | `/movie/{id}` | `ADMIN` | Cập nhật phim |
| `DELETE` | `/movie/{id}` | `ADMIN` | Xóa/ngừng phim |
| `POST` | `/movie/{id}/rating` | Authenticated | Đánh giá phim |
| `GET` | `/movie/{id}/rating/my` | Authenticated | Lấy đánh giá của người dùng hiện tại |
| `GET` | `/movie/{id}/ratings` | Public | Danh sách đánh giá phim |
| `GET` | `/director` | Public | Danh sách đạo diễn |
| `GET` | `/director/{id}` | Public | Chi tiết đạo diễn |
| `POST` | `/director` | `ADMIN` | Tạo đạo diễn |
| `PUT` | `/director/{id}` | `ADMIN` | Cập nhật đạo diễn |
| `DELETE` | `/director/{id}` | `ADMIN` | Xóa đạo diễn |
| `GET` | `/genre` | Public | Danh sách thể loại |
| `GET` | `/genre/{id}` | Public | Chi tiết thể loại |
| `POST` | `/genre` | `ADMIN` | Tạo thể loại |
| `PUT` | `/genre/{id}` | `ADMIN` | Cập nhật thể loại |
| `DELETE` | `/genre/{id}` | `ADMIN` | Xóa thể loại |

### 3.2. Chi nhánh, phòng, ghế và suất chiếu

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/branch` | Public | Danh sách chi nhánh |
| `GET` | `/branch/{id}` | Public | Chi tiết chi nhánh |
| `POST` | `/branch` | `ADMIN` | Tạo chi nhánh |
| `PUT` | `/branch/{id}` | `ADMIN/MANAGER` | Cập nhật chi nhánh |
| `DELETE` | `/branch/{id}` | `ADMIN` | Xóa chi nhánh |
| `GET` | `/room` | Public | Danh sách phòng |
| `GET` | `/room/{id}` | Public | Chi tiết phòng |
| `POST` | `/room` | `ADMIN/MANAGER` | Tạo phòng |
| `PUT` | `/room/{id}` | `ADMIN/MANAGER` | Cập nhật phòng |
| `DELETE` | `/room/{id}` | `ADMIN/MANAGER` | Xóa phòng |
| `GET` | `/seat/{id}` | Public | Chi tiết ghế |
| `GET` | `/seat/room/{roomId}` | Public | Sơ đồ ghế của phòng |
| `POST` | `/seat` | `ADMIN/MANAGER` | Tạo ghế |
| `PUT` | `/seat/{id}` | `ADMIN/MANAGER` | Cập nhật ghế |
| `DELETE` | `/seat/{id}` | `ADMIN/MANAGER` | Xóa ghế |
| `PUT` | `/seat/room/{roomId}/layout` | `ADMIN/MANAGER` | Cập nhật layout phòng |
| `GET` | `/showtime` | Public | Danh sách suất chiếu |
| `GET` | `/showtime/today` | Public | Suất chiếu trong ngày |
| `GET` | `/showtime/upcoming` | Public | Suất chiếu sắp tới |
| `GET` | `/showtime/{id}` | Public | Chi tiết suất chiếu |
| `GET` | `/showtime/room/{roomId}` | Public | Suất chiếu theo phòng |
| `GET` | `/showtime/movie/{movieId}` | Public | Suất chiếu theo phim |
| `GET` | `/showtime/branch/{branchId}` | Public | Suất chiếu theo chi nhánh |
| `POST` | `/showtime` | `ADMIN/MANAGER` | Tạo suất chiếu và ticket tương ứng |
| `PUT` | `/showtime/{id}` | `ADMIN/MANAGER` | Cập nhật suất chiếu |
| `DELETE` | `/showtime/{id}` | `ADMIN/MANAGER` | Xóa/hủy suất chiếu |
| `GET` | `/ticket/showtime/{showtimeId}` | Public | Trạng thái ticket/ghế của suất chiếu |
| `GET` | `/seat-type-price` | Public | Giá theo loại ghế |
| `POST` | `/seat-type-price` | `ADMIN` | Tạo giá loại ghế |
| `PUT` | `/seat-type-price/{seatType}` | `ADMIN` | Cập nhật giá loại ghế |

## 4. API booking và ticket

### 4.1. Booking

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/booking` | Authenticated | Tạo booking online và giữ ghế |
| `POST` | `/booking/staff` | `ADMIN/STAFF` | Tạo booking tại quầy |
| `GET` | `/booking` | `ADMIN/STAFF/MANAGER` | Tra cứu booking quản trị |
| `GET` | `/booking/{id}` | Authenticated | Chi tiết booking |
| `GET` | `/booking/my-bookings/list` | Authenticated | Booking của người dùng hiện tại |
| `GET` | `/booking/my-bookings/{id}` | Authenticated | Chi tiết booking của người dùng |
| `PUT` | `/booking/{id}` | `ADMIN/STAFF` | Cập nhật booking theo nghiệp vụ |
| `DELETE` | `/booking/{id}` | Authenticated | Hủy booking hợp lệ |
| `PATCH` | `/booking/{id}/apply-promotion` | Authenticated | Áp dụng mã khuyến mãi |
| `GET` | `/booking/refunds` | `ADMIN/MANAGER` | Danh sách yêu cầu hoàn tiền |
| `POST` | `/booking/{id}/refund-request` | Authenticated | Gửi yêu cầu hoàn tiền |
| `POST` | `/booking/{id}/refund-process` | `ADMIN/MANAGER` | Xử lý nghiệp vụ hoàn tiền |
| `POST` | `/booking/{id}/refund-decision` | `ADMIN/MANAGER` | Chấp nhận/từ chối hoàn tiền |

Ví dụ tạo booking online:

```json
{
  "showtimeId": 12,
  "seatIds": [101, 102],
  "foods": [
    {
      "foodId": 3,
      "quantity": 2
    }
  ],
  "paymentMethod": "VNPAY"
}
```

Quy tắc xử lý:

1. Backend kiểm tra tất cả seat thuộc showtime và đang `AVAILABLE`.
2. Redis tạo lock `seat_hold:{showtimeId}:{seatId}` với TTL 6 phút.
3. Booking tạo ở trạng thái `PENDING`, ticket ở trạng thái `HOLDING`.
4. Nếu một ghế thất bại, toàn bộ yêu cầu phải được từ chối và lock đã tạo phải được giải phóng.

### 4.2. Ticket và check-in

| Method | Path | Auth | Body/Content-Type | Mô tả |
|---|---|---|---|---|
| `GET` | `/ticket` | `ADMIN/MANAGER` | Query params | Tra cứu ticket |
| `GET` | `/ticket/{id}` | Authenticated | - | Chi tiết ticket |
| `POST` | `/ticket/check-in` | `ADMIN/STAFF/MANAGER` | JSON | Check-in bằng mã QR text |
| `POST` | `/ticket/check-in/qr-image` | `ADMIN/STAFF/MANAGER` | Multipart | Check-in bằng ảnh QR |
| `PUT` | `/ticket/{id}` | `ADMIN` | JSON | Cập nhật ticket |
| `DELETE` | `/ticket/{id}` | `ADMIN` | - | Xóa ticket |

Ví dụ check-in bằng text:

```json
{
  "qrCode": "CINEMAHUB|BOOKING=BK-20260923-AB12CD|TICKET=501|SEAT=A1"
}
```

Chỉ ticket `BOOKED` được chuyển sang `CHECKED_IN`; ticket đã check-in hoặc chưa thanh toán phải bị từ chối.

## 5. API thanh toán VNPay

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/v1/vnpay/payment-url` | Authenticated | Tạo URL thanh toán |
| `POST` | `/v1/vnpay/querydr` | Authenticated | Truy vấn giao dịch |
| `POST` | `/v1/vnpay/refund` | Theo quyền | Gửi yêu cầu hoàn tiền |
| `GET` | `/v1/vnpay/return` | Public callback | Người dùng được redirect về sau thanh toán |
| `GET` | `/v1/vnpay/ipn` | Public callback | VNPay gọi server-to-server |

Callback phải:

- Kiểm tra chữ ký VNPay trước khi đọc kết quả.
- Kiểm tra `responseCode` và `transactionStatus`.
- Tìm booking bằng mã tham chiếu giao dịch.
- Không cập nhật lặp booking đã `COMPLETED` hoặc `CANCELLED`.
- Thành công: booking `COMPLETED`, payment `PAID`, ticket `BOOKED`, tạo QR và xóa seat lock.
- Thất bại: booking `CANCELLED`, ticket `AVAILABLE`, xóa seat lock.

## 6. API món ăn, khuyến mãi và thông báo

### 6.1. Food và inventory

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/food` | Public | Danh sách món đang bán |
| `GET` | `/food/all` | `ADMIN/MANAGER` | Tất cả món |
| `GET` | `/food/{id}` | Public | Chi tiết món |
| `POST` | `/food` | `ADMIN/MANAGER` | Tạo món |
| `PUT` | `/food/{id}` | `ADMIN/MANAGER` | Cập nhật món |
| `DELETE` | `/food/{id}` | `ADMIN/MANAGER` | Xóa món |
| `POST` | `/food/{id}/stock-adjustments` | `ADMIN/MANAGER` | Điều chỉnh tồn kho |
| `GET` | `/food/stock-transactions` | `ADMIN/MANAGER` | Lịch sử nhập/xuất |

### 6.2. Promotion và notification

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/promotion` | `ADMIN/MANAGER` | Tạo khuyến mãi |
| `GET` | `/promotion` | `ADMIN/MANAGER` | Danh sách khuyến mãi |
| `GET` | `/promotion/available` | Authenticated | Khuyến mãi có thể dùng |
| `PUT` | `/promotion/{id}` | `ADMIN/MANAGER` | Cập nhật khuyến mãi |
| `DELETE` | `/promotion/{id}` | `ADMIN/MANAGER` | Xóa khuyến mãi |
| `POST` | `/promotion/validate` | Authenticated | Kiểm tra mã khuyến mãi |
| `GET` | `/notifications` | Authenticated | Danh sách thông báo |
| `GET` | `/notifications/unread-count` | Authenticated | Số thông báo chưa đọc |
| `POST` | `/notifications/{id}/read` | Authenticated | Đánh dấu đã đọc |

## 7. API staff, user và quản trị

### 7.1. Staff và user

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/staff/dashboard` | `ADMIN/STAFF` | Dashboard staff |
| `GET` | `/staff/shift/active` | `ADMIN/STAFF` | Ca đang hoạt động |
| `GET` | `/staff/shift/history` | `ADMIN/STAFF` | Lịch sử ca |
| `GET` | `/staff/schedules/my` | `ADMIN/STAFF` | Lịch làm việc cá nhân |
| `POST` | `/staff/shift/open` | `ADMIN/STAFF` | Mở ca |
| `POST` | `/staff/shift/close` | `ADMIN/STAFF` | Đóng ca |
| `GET` | `/staff/manager/staff/{staffId}/detail` | `ADMIN/MANAGER` | Chi tiết nhân viên |
| `POST` | `/staff/manager/schedules` | `ADMIN/MANAGER` | Tạo lịch làm việc |
| `PUT` | `/staff/manager/schedules/{scheduleId}` | `ADMIN/MANAGER` | Cập nhật lịch |
| `DELETE` | `/staff/manager/schedules/{scheduleId}` | `ADMIN/MANAGER` | Xóa lịch |
| `POST` | `/user` hoặc `/sign-up` | Public | Tạo tài khoản người dùng |
| `GET` | `/my-info` | Authenticated | Thông tin cá nhân |
| `PUT` | `/my-info` | Authenticated | Cập nhật cá nhân |
| `GET` | `/users` | `ADMIN/MANAGER` | Danh sách người dùng |
| `GET` | `/users/{id}` | Authenticated | Chi tiết người dùng |
| `PUT` | `/users/{id}` | `ADMIN/MANAGER` | Cập nhật người dùng |
| `DELETE` | `/users/{id}` | `ADMIN/MANAGER` | Xóa người dùng |
| `PUT` | `/users/{id}/status` | `ADMIN/MANAGER` | Khóa/mở tài khoản |

### 7.2. Reports, audit và settings

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/admin/analytics/dashboard` | `ADMIN/MANAGER` | Dashboard tổng quan |
| `GET` | `/admin/reports/revenue` | `ADMIN/MANAGER` | Báo cáo doanh thu |
| `GET` | `/admin/operations/daily-report` | `ADMIN/MANAGER` | Báo cáo vận hành ngày |
| `GET` | `/admin/reports/export/revenue.xlsx` | `ADMIN/MANAGER` | Export doanh thu XLSX |
| `GET` | `/admin/reports/export/operations.xlsx` | `ADMIN/MANAGER` | Export vận hành XLSX |
| `GET` | `/admin/audit-logs` | `ADMIN/MANAGER` | Tra cứu audit log |
| `GET` | `/admin/settings` | `ADMIN` | Danh sách cấu hình |
| `PUT` | `/admin/settings/{settingKey}` | `ADMIN` | Cập nhật cấu hình |

## 8. Chatbot và cleanup

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/chat` | Public | Gửi tin nhắn cho AI chatbot |
| `DELETE` | `/v1/cleanup/database` | Public/internal | Dọn dữ liệu booking hết hạn |

Ví dụ chat:

```json
{
  "sessionId": "session-abc-123",
  "message": "Hôm nay có phim nào ở chi nhánh trung tâm?"
}
```

Chatbot có thể tra cứu phim, suất chiếu, giá ghế, menu và tạo link đặt vé. Lịch sử được lưu theo `sessionId` trong Redis.

## 9. WebSocket API

### 9.1. Kết nối

- Endpoint SockJS: `/api/ws`.
- Application prefix: `/app`.
- Broker prefix: `/topic`.
- Không yêu cầu JWT ở bước handshake theo cấu hình hiện tại; quyền nghiệp vụ vẫn phải được kiểm tra ở REST API.

### 9.2. Theo dõi ghế

Client subscribe:

```text
/topic/showtime/{showtimeId}/seats
```

Payload server broadcast:

```json
{
  "seatId": 101,
  "status": "HOLDING"
}
```

Giá trị `status`: `AVAILABLE`, `HOLDING`, `BOOKED`.

## 10. Luồng tích hợp mẫu

### 10.1. Đặt vé và thanh toán

```text
GET /showtime/{id}
GET /ticket/showtime/{showtimeId}
POST /booking
POST /v1/vnpay/payment-url
GET /v1/vnpay/return
GET /booking/my-bookings/{id}
```

### 10.2. Bán vé tiền mặt tại quầy

```text
POST /staff/shift/open
POST /booking/staff
GET /booking/{id}
POST /ticket/check-in
POST /staff/shift/close
```

## 11. Yêu cầu bảo mật và vận hành API

- Chỉ cho phép CORS từ các origin đã cấu hình của frontend.
- Không ghi access token, refresh token, mật khẩu hoặc secret vào log.
- Kiểm tra quyền ở cả URL level và method level.
- Request tạo booking phải chống trùng ghế và xử lý rollback khi lock một phần thất bại.
- VNPay return/IPN phải xác minh chữ ký và hỗ trợ idempotency.
- Các endpoint cleanup, report và management nên được giới hạn mạng hoặc scheduler khi triển khai production dù hiện tại có route public trong cấu hình.
- API production phải chạy HTTPS và đặt giới hạn request cho login, OTP, chat và callback.
