# API Contract và Auth Design

**Hệ thống:** CinemaPTIT Backend  
**Phiên bản:** 1.0  
**Ngày:** 23/09/2026  
**Base URL:** `http://localhost:8081/api`

## 1. Quy ước API

### 1.1. Request headers

Endpoint cần đăng nhập sử dụng:

```http
Authorization: Bearer <accessToken>
Content-Type: application/json
```

Endpoint upload dùng `multipart/form-data`. Endpoint export báo cáo trả file XLSX.

### 1.2. Response thành công

Các controller sử dụng wrapper `ApiResponse<T>`:

```json
{
  "code": 200,
  "message": "Success",
  "result": {}
}
```

Các field:

| Field | Kiểu | Bắt buộc | Mô tả |
|---|---|---:|---|
| `code` | `number` | Có | Mặc định là `200` |
| `message` | `string` | Không | Thông báo kết quả |
| `result` | `object/array/null` | Không | Dữ liệu trả về |

`result` được bỏ qua khi có giá trị `null` do cấu hình JSON `NON_NULL`.

### 1.3. Response lỗi

```json
{
  "code": 400,
  "error": "Bad Request",
  "message": "Password must be at least 6 characters",
  "timestamp": "2026-09-23T10:15:30.000+00:00",
  "path": "/api/auth/reset-password"
}
```

| Field | Kiểu | Mô tả |
|---|---|---|
| `code` | `number` | Mã HTTP hoặc mã lỗi tương ứng |
| `error` | `string` | Tên loại lỗi HTTP |
| `message` | `string` | Chi tiết lỗi cho client |
| `timestamp` | `date-time` | Thời điểm phát sinh lỗi |
| `path` | `string` | Đường dẫn API bị lỗi |

## 2. Token và vòng đời phiên đăng nhập

### 2.1. Token

- Access token là JWT ký bằng HMAC-SHA512.
- Access token có thời hạn khoảng 30 phút.
- Refresh token có thời hạn khoảng 30 ngày.
- JWT chứa subject là username, claim `role` và `jti`.
- Refresh token được lưu trong Redis để kiểm tra hiệu lực.
- Khi logout, `jti` của access token được đưa vào Redis blacklist đến khi token hết hạn.
- Client không được tự giải mã và tin tưởng claim quyền để thay thế kiểm tra ở backend.

### 2.2. Quy tắc client

1. Lưu access token và refresh token ở nơi bảo vệ phù hợp với ứng dụng.
2. Gửi access token trong header `Authorization` cho API bảo vệ.
3. Khi nhận `401`, gọi refresh một lần bằng refresh token.
4. Nếu refresh thất bại, xóa session cục bộ và đưa người dùng về màn hình đăng nhập.
5. Không retry vô hạn khi nhận `401` hoặc `403`.
6. Gọi logout khi người dùng chủ động đăng xuất.

## 3. Auth API Contract

### 3.1. Đăng ký

`POST /sign-up`  
**Auth:** Public  
**Content-Type:** `application/json`

Request:

```json
{
  "username": "nguyenvana",
  "fullName": "Nguyen Van A",
  "email": "vana@example.com",
  "phoneNumber": "0900000000",
  "password": "secret123",
  "gender": "MALE",
  "dob": "2000-01-15"
}
```

| Field | Kiểu | Bắt buộc | Quy tắc |
|---|---|---:|---|
| `username` | `string` | Không khai báo annotation bắt buộc | Username đăng nhập |
| `fullName` | `string` | Không | Họ tên |
| `email` | `string` | Không | Email người dùng |
| `phoneNumber` | `string` | Không | Số điện thoại |
| `password` | `string` | Không, nhưng nếu có phải hợp lệ | Tối thiểu 6 ký tự |
| `gender` | `string` | Không | Giá trị nghiệp vụ giới tính |
| `dob` | `date` | Không | Định dạng `yyyy-MM-dd` |
| `role` | `enum` | Không nên gửi từ client | Role tùy chọn khi admin tạo user |
| `branchId` | `number` | Không | Chi nhánh của nhân viên nếu có |

Response `200`:

```json
{
  "code": 200,
  "message": "Register successfully!",
  "result": {
    "id": 10,
    "fullName": "Nguyen Van A",
    "username": "nguyenvana",
    "email": "vana@example.com",
    "phoneNumber": "0900000000",
    "role": "USER",
    "branchId": null,
    "dob": "2000-01-15",
    "gender": "MALE",
    "status": "ACTIVE",
    "createdAt": "2026-09-23T10:15:30",
    "membershipTier": "BRONZE",
    "totalSpending": 0
  }
}
```

### 3.2. Đăng nhập

`POST /auth/login`  
**Auth:** Public  
**Content-Type:** `application/json`

Request:

```json
{
  "username": "nguyenvana",
  "password": "secret123"
}
```

Response `200`:

```json
{
  "code": 200,
  "message": "Login successfully",
  "result": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

Tài khoản `BLOCKED`, mật khẩu sai hoặc username không tồn tại phải bị từ chối.

### 3.3. Refresh token

`POST /auth/refresh`  
**Auth:** Không cần access token  
**Content-Type:** `application/json`

Request:

```json
{
  "refreshToken": "eyJ..."
}
```

Response `200` có cùng schema với login. Refresh token không hợp lệ, hết hạn hoặc không còn tồn tại trong Redis trả `401` hoặc lỗi nghiệp vụ tương ứng.

### 3.4. Logout

`POST /auth/logout`  
**Auth:** Authenticated  
**Content-Type:** Không bắt buộc

Request header:

```http
Authorization: Bearer <accessToken>
```

Response:

```json
{
  "code": 200,
  "message": "Logout successfully"
}
```

Backend blacklist access token hiện tại theo `jti`. Sau khi logout, token đó không được dùng tiếp dù thời gian JWT chưa hết hạn.

### 3.5. Đổi mật khẩu

`POST /auth/change-password`  
**Auth:** Authenticated  
**Content-Type:** `application/json`

Request:

```json
{
  "currentPassword": "old-secret",
  "newPassword": "new-secret123"
}
```

Response:

```json
{
  "code": 200,
  "message": "Change password successfully"
}
```

Backend phải kiểm tra mật khẩu hiện tại trước khi BCrypt mật khẩu mới.

### 3.6. Quên mật khẩu

`POST /auth/forgot-password`  
**Auth:** Public  
**Content-Type:** `application/json`

Request:

```json
{
  "email": "vana@example.com"
}
```

`email` bắt buộc, không được trống và phải đúng định dạng email. OTP được lưu trong Redis với thời hạn dự kiến 5 phút và gửi qua email.

Response:

```json
{
  "code": 200,
  "message": "OTP sent to email successfully"
}
```

### 3.7. Đặt lại mật khẩu

`POST /auth/reset-password`  
**Auth:** Public  
**Content-Type:** `application/json`

Request:

```json
{
  "email": "vana@example.com",
  "otp": "123456",
  "newPassword": "new-secret123"
}
```

| Field | Kiểu | Bắt buộc | Quy tắc |
|---|---|---:|---|
| `email` | `string` | Có | Email hợp lệ |
| `otp` | `string` | Có | Có thể gửi tên JSON `OTP` do hỗ trợ alias |
| `newPassword` | `string` | Có | Tối thiểu 6 ký tự |

Response:

```json
{
  "code": 200,
  "message": "Password reset successfully"
}
```

OTP sai hoặc hết hạn phải trả lỗi và không thay đổi mật khẩu.

### 3.8. Google OAuth2

Client mở:

```text
GET /api/oauth2/authorization/google
```

Sau khi Google xác thực thành công, backend kiểm tra email đã verified, tìm hoặc tạo user role `USER`, tạo access/refresh token và redirect về URL thành công cấu hình bởi `APP_OAUTH2_SUCCESS_URL`.

## 4. User API Contract

### 4.1. User response

```json
{
  "id": 10,
  "fullName": "Nguyen Van A",
  "username": "nguyenvana",
  "email": "vana@example.com",
  "phoneNumber": "0900000000",
  "role": "USER",
  "branchId": null,
  "dob": "2000-01-15",
  "gender": "MALE",
  "status": "ACTIVE",
  "createdAt": "2026-09-23T10:15:30",
  "membershipTier": "BRONZE",
  "totalSpending": 0
}
```

Không bao giờ trả field `password`.

### 4.2. Thông tin cá nhân

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/my-info` | Authenticated | Lấy user hiện tại |
| `PUT` | `/my-info` | Authenticated | Cập nhật user hiện tại |

### 4.3. Quản trị user

| Method | Path | Auth | Query/body |
|---|---|---|---|
| `POST` | `/users` hoặc `/user` | `ADMIN/MANAGER` | `CreateUserRequest` |
| `GET` | `/users` | Authenticated, nên giới hạn `ADMIN/MANAGER` | `page`, `size`, `username`, `email`, `phone` |
| `GET` | `/users/{id}` | Authenticated | Path `id` |
| `PUT` | `/users/{id}` | Authenticated, nên kiểm tra ownership/quyền | `UpdateUserRequest` |
| `DELETE` | `/users/{id}` | Authenticated, nên giới hạn quản trị | Path `id` |
| `PUT` | `/users/{id}/status` | `ADMIN/MANAGER` | Query `status=ACTIVE` hoặc `BLOCKED` |

Response danh sách user:

```json
{
  "code": 200,
  "result": {
    "data": [],
    "page": 1,
    "size": 10,
    "totalPages": 0,
    "totalElements": 0
  }
}
```

Tên field phân trang phải được đối chiếu với `PageResponse` khi frontend tích hợp.

## 5. Ma trận phân quyền

| Nhóm API | Public | USER | STAFF | MANAGER | ADMIN |
|---|---:|---:|---:|---:|---:|
| GET movie/showtime/genre/director/branch/room | Có | Có | Có | Có | Có |
| GET food | Có | Có | Có | Có | Có |
| Đăng ký/login/refresh/forgot/reset | Có | Có | Có | Có | Có |
| Chat | Có | Có | Có | Có | Có |
| Booking cá nhân | Không | Có | Có | Có | Có |
| Booking tại quầy | Không | Không | Có | Không theo controller hiện tại | Có |
| Check-in ticket | Không | Không | Có | Có | Có |
| Mở/đóng ca | Không | Không | Có | Có | Có |
| Quản lý food/inventory | Không | Không | Không | Có | Có |
| Quản lý promotion | Không | Không | Không | Có | Có |
| Quản lý phim | Không | Không | Không | Không | Có |
| Quản lý phòng/ghế | Không | Không | Không | Có | Có |
| Báo cáo/audit | Không | Không | Không | Có | Có |
| System settings | Không | Không | Không | Không | Có |
| Quản lý user | Không | Không | Không | Có giới hạn | Có |

## 6. Auth error contract

Client cần xử lý tối thiểu các tình huống sau:

| Tình huống | HTTP dự kiến | Hành động client |
|---|---:|---|
| Thiếu Authorization | `401` | Đưa về login hoặc thử refresh nếu có session |
| Access token hết hạn | `401` | Gọi refresh một lần |
| Token bị blacklist sau logout | `401` | Xóa session và yêu cầu login lại |
| Refresh token sai/hết hạn | `401` | Xóa cả hai token |
| Không đủ role | `403` | Hiển thị quyền bị từ chối, không retry |
| Sai mật khẩu hiện tại | `400` | Hiển thị lỗi tại form đổi mật khẩu |
| OTP sai/hết hạn | `400` | Yêu cầu nhập/gửi OTP mới |
| Email không tồn tại | `404` hoặc mã lỗi nghiệp vụ | Hiển thị thông báo phù hợp |
| Tài khoản bị khóa | `401/403` | Không cho truy cập hệ thống |

## 7. Checklist tích hợp frontend

- [ ] Tạo Axios interceptor gắn `Authorization: Bearer`.
- [ ] Chỉ có một request refresh đang chạy tại một thời điểm.
- [ ] Sau refresh thành công, retry request cũ đúng một lần.
- [ ] Sau refresh thất bại, xóa auth state và chuyển tới `/login`.
- [ ] Không lưu password hoặc OTP vào local storage.
- [ ] Ẩn chức năng trên giao diện theo role nhưng vẫn luôn dựa vào backend để bảo mật.
- [ ] Hiển thị `message` từ `ApiResponse`/`ErrorResponse`, không phụ thuộc cứng vào một ngôn ngữ.
- [ ] Với `POST /auth/logout`, gửi đúng access token trong header.
- [ ] Với reset password, gửi field `otp` hoặc `OTP`, không gửi tên khác.
