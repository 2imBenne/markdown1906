# TÀI LIỆU LUỒNG KIẾN TRÚC BACKEND (UMC CARE)

## 1. Tổng Quan Kiến Trúc

Dự án UMC Care sử dụng kiến trúc MVC (Model-View-Controller) trên nền tảng Golang, tương tác với MySQL thông qua GORM và định tuyến bằng thư viện `net/http` thuần. Luồng xử lý chung bao gồm Middleware kiểm tra JWT và phân quyền (RBAC), Controller xử lý logic nghiệp vụ, và Model tương tác với cơ sở dữ liệu.

## 2. Danh Sách Các Tính Năng

1. Xác Thực & Phân Quyền (Auth & RBAC)
2. Đặt Lịch Khám Bệnh (Booking)
3. Thanh Toán & Hóa Đơn (Payment)
4. Phân Hệ Khám Bệnh (Doctor Portal)
5. Phân Hệ Cận Lâm Sàng (Lab Technician)
6. Quản Lý Tiêm Chủng (Vaccination)
7. Tác Vụ Ngầm (Background Workers)
8. Phân Hệ Admin & Quản Trị Hệ Thống (Admin Portal)
9. Quản Lý Lịch Làm Việc & Ca Trực (Doctor Schedule)

---

## 3. Chi Tiết Luồng Từng Tính Năng

### 3.1 Xác Thực & Phân Quyền (Auth & RBAC)

- **API Endpoints:** `POST /api/login`, `POST /api/register`, `POST /api/send-otp`
- **Mô tả:** Người dùng xác thực bằng số điện thoại và mật khẩu, nhận về Access Token (15 phút) và Refresh Token (7 ngày). Hệ thống áp dụng kiểm tra Token Version để vô hiệu hóa token cũ khi bị giáng quyền hoặc đổi mật khẩu. Các quyền được kiểm tra qua RBAC với 5 vai trò (Admin bypass mọi quyền).
- **Các file liên quan:** `controllers/auth_controller.go`, `middlewares/auth_middleware.go`, `utils/jwt.go`.

```mermaid
sequenceDiagram
    autonumber
    actor User as Client
    participant API as Auth Controller
    participant OTP as OTP Cache (In-memory)
    participant DB as MySQL

    %% Luồng đăng ký & OTP
    User->>API: POST /api/send-otp (Số điện thoại)
    API->>OTP: Tạo & lưu mã 6 số (TTL 5 phút)
    API-->>User: SMS chứa OTP

    User->>API: POST /api/register (Thông tin + OTP)
    API->>OTP: Xác thực OTP
    API->>DB: Hash Password (bcrypt) & Lưu User
    DB-->>API: OK

    %% Luồng đăng nhập
    User->>API: POST /api/login (SĐT + Password)
    API->>DB: Kiểm tra Password & Token Version
    DB-->>API: Valid
    API->>API: Tạo Access Token & Refresh Token
    API-->>User: Trả về Token qua HttpOnly Cookies/JSON
```

---

### 3.2 Đặt Lịch Khám Bệnh (Booking Appointment)

- **API Endpoints:** `POST /api/appointments`
- **Mô tả:** Bệnh nhân chọn ngày và TimeSlot của bác sĩ. Hệ thống sử dụng khóa bi quan (FOR UPDATE) để chặn trùng lịch, thiết lập trạng thái "pending" và giữ chỗ trong đúng 1 phút.
- **Các file liên quan:** `controllers/appointment_controller.go`, `models/appointment.go`.

```mermaid
sequenceDiagram
    autonumber
    actor Patient as Bệnh nhân
    participant API as Appointment Controller
    participant DB as MySQL Database
    participant PayOS as PayOS Gateway
    participant Worker as Reminder Worker

    %% Phase 1: Tạo lịch & Giữ chỗ
    Patient->>API: Đặt lịch (Bác sĩ, Ngày, TimeSlot)
    API->>DB: Truy vấn TimeSlot + Pessimistic Lock (FOR UPDATE)

    alt Slot đã được đặt
        DB-->>API: Lỗi trùng lịch
        API-->>Patient: Báo lỗi "Khung giờ không khả dụng"
    else Slot khả dụng
        API->>DB: Tạo Appointment (Status="pending", ExpiresAt=Now+1m)
        DB-->>API: Lưu thành công
        API->>PayOS: Khởi tạo dữ liệu thanh toán
        PayOS-->>API: Trả về Payment QR/URL
        API-->>Patient: Trả kết quả + Yêu cầu thanh toán
    end

    %% Phase 2: Hủy tự động nếu không thanh toán
    loop Chạy ngầm mỗi 30s
        Worker->>DB: checkExpiredPendingAppointments()
        DB-->>Worker: Trả về các lịch "pending" quá 1 phút
        Worker->>DB: Cập nhật Status="cancelled", hoàn trả lại TimeSlot
    end
```

---

### 3.3 Luồng Thanh Toán (Payment)

- **API Endpoints:** `POST /api/payments`, `POST /api/payments/clinical`, `POST /api/payments/vaccination`
- **Mô tả:** Tích hợp với PayOS để sinh mã VietQR thanh toán cho các dịch vụ: khám bệnh, cận lâm sàng, thuốc và tiêm chủng. Hỗ trợ Webhook xử lý giao dịch tự động và báo cáo thời gian thực qua WebSocket.
- **Các file liên quan:** `controllers/payment_controller.go`, `utils/payos.go`, `utils/websocket.go`.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant API as Payment Controller
    participant PayOS
    participant WS as WebSocket Hub
    participant DB

    User->>API: Yêu cầu thanh toán (Phí khám / CLS / Thuốc)
    API->>DB: Lưu Payment record
    API->>PayOS: Khởi tạo dữ liệu giao dịch
    PayOS-->>API: Link thanh toán & QRCode
    API-->>User: Trả về QRCode

    User->>PayOS: Quét mã chuyển khoản
    PayOS->>API: Gửi Webhook xác nhận (Success)
    API->>DB: Update Payment = Paid + Update Status nghiệp vụ
    API->>WS: Push thông báo Real-time
    WS-->>User: Hiện thông báo "Thanh toán thành công"
```

---

### 3.4 Phân Hệ Khám Bệnh (Doctor Portal)

- **API Endpoints:** `GET /api/doctor/appointments`, `POST /api/appointment-services/bulk`, `POST /api/prescriptions`
- **Mô tả:** Bác sĩ nhận hàng đợi, cập nhật trạng thái "processing", chỉ định xét nghiệm (CLS), và kết thúc ca khám bằng việc kê đơn thuốc.
- **Các file liên quan:** `controllers/doctor_controller.go`, `models/record.go`.

```mermaid
sequenceDiagram
    autonumber
    actor Doctor
    participant API as Doctor Controller
    participant DB

    Doctor->>API: GET /api/doctor/appointments
    API->>DB: Truy vấn hàng đợi theo lịch
    DB-->>API: Danh sách bệnh nhân (paid)
    API-->>Doctor: Hiển thị

    Doctor->>API: Cập nhật ca khám = "processing"
    API->>DB: Save status

    alt Yêu cầu xét nghiệm
        Doctor->>API: Chỉ định dịch vụ Cận Lâm Sàng
        API->>DB: Tạo AppointmentService (Chờ thanh toán)
        API-->>Doctor: Phản hồi thành công
    end

    Doctor->>API: POST /api/prescriptions (Kê đơn)
    API->>DB: Lưu đơn thuốc + Cập nhật Appointment = "completed"
    DB-->>API: Hoàn tất
    API-->>Doctor: Kết thúc ca khám
```

---

### 3.5 Phân Hệ Cận Lâm Sàng (Lab Technician)

- **API Endpoints:** `GET /api/lab/pending-services`, `PUT /api/lab/services/complete`
- **Mô tả:** Kỹ thuật viên (KTV) tiếp nhận dịch vụ y tế đã thanh toán, thực hiện xét nghiệm, nhập kết quả, tự động tạo hồ sơ bệnh án `LabResult` và gửi thông báo.
- **Các file liên quan:** `controllers/lab_controller.go`, `models/record.go`.

```mermaid
sequenceDiagram
    autonumber
    actor LabTech as Kỹ Thuật Viên
    participant API as Lab Controller
    participant DB
    participant Notify as Notification Module

    LabTech->>API: Lấy danh sách dịch vụ chờ
    API->>DB: Query dịch vụ "paid" / tiêm chủng "confirmed"
    DB-->>API: Trả về danh sách
    API-->>LabTech: Hiển thị giao diện

    LabTech->>API: Nhập kết quả & Hoàn thành
    API->>DB: Cập nhật Service = "completed"
    API->>DB: Sinh bản ghi LabResult vào DB
    API->>Notify: Tạo Notification gửi bệnh nhân
    API-->>LabTech: Lưu thành công
```

---

### 3.6 Quản Lý Tiêm Chủng (Vaccination)

- **API Endpoints:** `POST /api/vaccinations/*`
- **Mô tả:** Bệnh nhân chọn vắc-xin, hệ thống trừ tồn kho, thanh toán thành công chuyển sang "confirmed". KTV sau đó thực hiện tiêm và cập nhật thành "completed".
- **Các file liên quan:** `controllers/vaccination_controller.go`, `models/vaccination.go`.

```mermaid
sequenceDiagram
    autonumber
    actor Patient
    participant API as Vaccine Controller
    participant DB
    actor LabTech as KTV

    Patient->>API: Đăng ký Vắc-xin & chọn ngày
    API->>DB: Trừ trực tiếp tồn kho Vaccine
    API->>DB: Tạo bản ghi Registration (Pending)
    API-->>Patient: Chuyển hướng thanh toán

    Note over Patient, API: Patient thanh toán thành công qua PayOS
    API->>DB: Cập nhật Registration = "confirmed"

    LabTech->>API: Cập nhật đã tiêm xong
    API->>DB: Đổi Registration = "completed"
    API-->>LabTech: OK
```

---

### 3.7 Hệ Thống Tác Vụ Ngầm (Background Workers)

- **Mô tả:** Chạy tự động chu kỳ 30 giây một lần với 3 nhiệm vụ chính: dọn dẹp lịch hết hạn, nhắc nhở trước 5 phút khi ca khám bắt đầu, và nhắc tái khám.
- **Các file liên quan:** `utils/reminder_worker.go`.

```mermaid
sequenceDiagram
    autonumber
    participant Worker as reminder_worker.go
    participant DB
    participant Notification

    loop Chu kỳ 30 Giây
        %% Hủy lịch
        Worker->>DB: checkExpiredPendingAppointments()
        DB-->>Worker: Hủy slot "pending" > 1 phút & Hoàn trả slot

        %% Nhắc lịch sắp diễn ra
        Worker->>DB: checkUpcomingAppointments()
        DB-->>Worker: Lấy lịch cách hiện tại <= 5 phút
        Worker->>Notification: Gửi push notification cho bệnh nhân

        %% Nhắc tái khám
        Worker->>DB: checkRevisitReminders()
        DB-->>Worker: Quét toa thuốc có revisit_date = ngày mai
        Worker->>Notification: Báo bệnh nhân đặt lại lịch
    end
```

---

### 3.8 Phân Hệ Admin & Quản Trị Hệ Thống (Admin Portal)

- **API Endpoints:** `/api/admin/users`, `/api/admin/staff/permissions`, `/api/admin/staff/promote`
- **Mô tả:** Admin có quyền cao nhất (RoleID=1), có thể gán quyền chi tiết cho Staff hoặc khóa tài khoản. Khi khóa, hệ thống áp dụng kỹ thuật cộng thêm 9999 vào `token_version` để ngay lập tức vô hiệu hóa tất cả JWT cũ của user.
- **Các file liên quan:** `controllers/user_controller.go`, `controllers/permission_controller.go`, `middlewares/admin_middleware.go`.

```mermaid
sequenceDiagram
    autonumber
    actor Admin
    participant API as Admin Controller
    participant DB as MySQL

    %% Luồng cấp quyền cho Staff
    Admin->>API: Gán quyền cho Staff (VD: "appointment:manage")
    API->>DB: Kiểm tra Role Admin (RoleID == 1)
    API->>DB: Insert vào bảng trung gian user_permissions
    DB-->>API: Thành công
    API-->>Admin: Thông báo đã gán quyền

    %% Luồng Block/Khóa tài khoản
    Admin->>API: Yêu cầu Block User (Truyền UserID)
    API->>DB: Tìm User theo ID
    API->>DB: Update token_version = token_version + 9999
    API->>DB: Update is_blocked = true
    DB-->>API: Cập nhật thành công
    API-->>Admin: Đã khóa User

    Note over API, DB: Mọi request tiếp theo của User này sẽ bị chặn ở<br>Auth Middleware do Token Version không khớp
```

---

### 3.9 Quản Lý Lịch Làm Việc & Ca Trực (Doctor Schedule)

- **API Endpoints:** `/api/doctor/schedules/request`, `/api/schedules`
- **Mô tả:** Hệ thống quản lý lịch làm việc của Bác sĩ. Bác sĩ đăng ký ca trực, hệ thống tự động sinh ra các `TimeSlot` tương ứng (30 phút/slot) và gắn với Phòng khám. Bệnh nhân sẽ query các slot này để đặt lịch.
- **Các file liên quan:** `controllers/schedule_controller.go`, `models/doctor.go`.

```mermaid
sequenceDiagram
    autonumber
    actor Doctor as Bác sĩ
    participant API as Schedule Controller
    participant DB as MySQL
    actor Patient as Bệnh nhân

    %% Luồng đăng ký lịch
    Doctor->>API: POST /api/doctor/schedules/request (Ngày, Ca làm)
    API->>DB: Tạo DoctorSchedule mới
    API->>DB: Auto-generate các TimeSlot (Mỗi slot 30 phút)
    API->>DB: Gán lịch vào một Room (Phòng khám) cụ thể
    DB-->>API: Hoàn tất
    API-->>Doctor: Đã tạo lịch làm việc

    %% Bệnh nhân lấy lịch trống
    Patient->>API: GET /api/timeslots?doctor_id=...&date=...
    API->>DB: Truy vấn các TimeSlot chưa bị ai đặt (status trống/hủy)
    DB-->>API: Danh sách Slot khả dụng
    API-->>Patient: Hiển thị lên UI để Booking
```
