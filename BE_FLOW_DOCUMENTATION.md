# TÀI LIỆU KIẾN TRÚC VÀ LUỒNG HỆ THỐNG BACKEND (UMC CARE)

## I. TỔNG QUAN KIẾN TRÚC

Dự án UMC Care (Hoàn Mỹ Care) sử dụng kiến trúc MVC (Model-View-Controller) trên nền tảng Golang, tương tác với MySQL thông qua GORM và định tuyến bằng thư viện `net/http` thuần. Hệ thống được bảo mật bằng JWT Token Versioning và phân quyền RBAC đa cấp độ (Admin, Staff, Doctor, LabTech, User).

---

## II. USE CASE HỆ THỐNG

### 1. Sơ Đồ Use Case Tổng Quát

Hệ thống phân chia quyền hạn theo 5 tác nhân (Actor) chính:

```mermaid
flowchart LR
    %% Định nghĩa Actors
    User((Bệnh Nhân))
    Doctor((Bác Sĩ))
    Lab((KTV CLS))
    Admin((Admin / Staff))

    %% Định nghĩa Use Cases
    subgraph Hệ Thống UMC Care
        UC1(Đăng ký / Đăng nhập)
        UC2(Quản lý Hồ sơ bệnh án)
        UC3(Đặt / Hủy lịch khám)
        UC4(Thanh toán phí dịch vụ)
        UC5(Khám bệnh & Kê đơn)
        UC6(Thực hiện Xét nghiệm CLS)
        UC7(Đăng ký Tiêm chủng)
        UC8(Quản lý Lịch làm việc)
        UC9(Quản trị Người dùng & Phân quyền)
    end

    %% Ràng buộc
    User --> UC1
    User --> UC2
    User --> UC3
    User --> UC4
    User --> UC7

    Doctor --> UC1
    Doctor --> UC5
    Doctor --> UC8

    Lab --> UC1
    Lab --> UC6

    Admin --> UC1
    Admin --> UC9
```

### 2. Kịch Bản Chi Tiết Các Use Case Cốt Lõi

_(Trình bày theo chuẩn khung đặc tả kịch bản Use Case)_

<br>
<div align="center"><i>Bảng 2.1 Kịch bản Đặt lịch khám bệnh</i></div>

<table border="1" style="border-collapse: collapse; width: 100%; margin-bottom: 20px;">
  <tr>
    <td style="width: 25%; padding: 8px;"><b>Tên use case</b></td>
    <td style="padding: 8px;"><b>Đặt lịch khám bệnh</b></td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tác nhân chính</b></td>
    <td style="padding: 8px;">Người dùng (Bệnh nhân)</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tiền điều kiện</b></td>
    <td style="padding: 8px;">Người dùng đã đăng nhập vào hệ thống.</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Hậu điều kiện</b></td>
    <td style="padding: 8px;">Lịch khám được tạo thành công, giữ chỗ trong 1 phút chờ thanh toán.</td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Kịch bản chính</b><br>
      1. Người dùng chọn Bác sĩ, Ngày khám mong muốn.<br>
      2. Hệ thống hiển thị các khung giờ (TimeSlot) còn trống.<br>
      3. Người dùng chọn 1 khung giờ và nhấn nút "Đặt lịch".<br>
      4. Hệ thống khóa khung giờ, tạo cuộc hẹn (Appointment) với trạng thái "pending".<br>
      5. Hệ thống hiển thị thông báo thành công và chuyển sang giao diện thanh toán phí khám.
    </td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Ngoại lệ</b><br>
      1. Khung giờ đã bị người khác đặt trước lúc gửi request: Hệ thống thông báo lỗi "Khung giờ không khả dụng".<br>
      2. Người dùng không thanh toán sau 1 phút: Worker hệ thống chạy ngầm tự động hủy lịch và trả lại khung giờ.
    </td>
  </tr>
</table>

<div align="center"><i>Bảng 2.2 Kịch bản Khám bệnh và Kê đơn</i></div>

<table border="1" style="border-collapse: collapse; width: 100%; margin-bottom: 20px;">
  <tr>
    <td style="width: 25%; padding: 8px;"><b>Tên use case</b></td>
    <td style="padding: 8px;"><b>Khám bệnh và Kê đơn</b></td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tác nhân chính</b></td>
    <td style="padding: 8px;">Bác sĩ</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tiền điều kiện</b></td>
    <td style="padding: 8px;">Bác sĩ đăng nhập vào Portal và bệnh nhân đã thanh toán phí khám.</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Hậu điều kiện</b></td>
    <td style="padding: 8px;">Trạng thái lịch khám chuyển thành "completed", hệ thống sinh đơn thuốc mới.</td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Kịch bản chính</b><br>
      1. Bác sĩ mở giao diện xem hàng đợi bệnh nhân trong ngày.<br>
      2. Bác sĩ gọi bệnh nhân vào khám (hệ thống cập nhật trạng thái "processing").<br>
      3. Bác sĩ nhập chẩn đoán và chọn các loại thuốc cần kê vào đơn.<br>
      4. Bác sĩ nhấn "Hoàn thành và Kê đơn".<br>
      5. Hệ thống lưu đơn thuốc, đẩy thông báo thanh toán đơn thuốc cho bệnh nhân và đánh dấu hoàn thành ca khám.
    </td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Ngoại lệ</b><br>
      1. Bác sĩ yêu cầu làm Cận lâm sàng (CLS) trước khi kê đơn: Hệ thống tạo Service yêu cầu thanh toán CLS, ca khám tạm dừng chờ kết quả từ KTV.
    </td>
  </tr>
</table>

<div align="center"><i>Bảng 2.3 Kịch bản Thanh toán dịch vụ (PayOS)</i></div>

<table border="1" style="border-collapse: collapse; width: 100%; margin-bottom: 20px;">
  <tr>
    <td style="width: 25%; padding: 8px;"><b>Tên use case</b></td>
    <td style="padding: 8px;"><b>Thanh toán dịch vụ (Khám, CLS, Thuốc)</b></td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tác nhân chính</b></td>
    <td style="padding: 8px;">Người dùng (Bệnh nhân)</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Tiền điều kiện</b></td>
    <td style="padding: 8px;">Người dùng có hóa đơn dịch vụ y tế đang ở trạng thái chờ thanh toán.</td>
  </tr>
  <tr>
    <td style="padding: 8px;"><b>Hậu điều kiện</b></td>
    <td style="padding: 8px;">Hóa đơn cập nhật trạng thái "paid", kích hoạt bước nghiệp vụ tiếp theo.</td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Kịch bản chính</b><br>
      1. Người dùng chọn thanh toán cho dịch vụ.<br>
      2. Hệ thống gọi API PayOS sinh mã VietQR tích hợp số tiền và nội dung chuyển khoản.<br>
      3. Người dùng mở app ngân hàng quét mã và chuyển tiền.<br>
      4. PayOS gửi Webhook xác nhận giao dịch thành công về Backend.<br>
      5. Backend cập nhật dữ liệu và gửi WebSocket báo trạng thái Real-time lên App.
    </td>
  </tr>
  <tr>
    <td colspan="2" style="padding: 8px;">
      <b>Ngoại lệ</b><br>
      1. Giao dịch bị hủy do hết hạn mã QR: Hệ thống hủy giao dịch thanh toán chờ.
    </td>
  </tr>
</table>

---

## III. DANH SÁCH CHI TIẾT CÁC LUỒNG TÍNH NĂNG (BACKEND FLOW)

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

## IV. CHI TIẾT SEQUENCE DIAGRAM TỪNG LUỒNG

### Luồng 1. Xác Thực & Phân Quyền (Auth & RBAC)

- **API Endpoints:** `POST /api/login`, `POST /api/register`, `POST /api/send-otp`
- **Mô tả:** Người dùng xác thực bằng số điện thoại và mật khẩu, nhận về Access Token (15 phút) và Refresh Token (7 ngày). Hệ thống áp dụng kiểm tra Token Version để vô hiệu hóa token cũ khi bị giáng quyền hoặc đổi mật khẩu.
- **Các file liên quan:** `auth_controller.go`, `auth_middleware.go`, `jwt.go`.

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

### Luồng 2. Đặt Lịch Khám Bệnh (Booking Appointment)

- **API Endpoints:** `POST /api/appointments`
- **Mô tả:** Bệnh nhân chọn ngày và TimeSlot của bác sĩ. Sử dụng khóa bi quan (FOR UPDATE) để chặn trùng lịch, thiết lập trạng thái "pending" và giữ chỗ trong đúng 1 phút.
- **Các file liên quan:** `appointment_controller.go`, `models/appointment.go`.

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

### Luồng 3. Thanh Toán (Payment)

- **API Endpoints:** `POST /api/payments`, `POST /api/payments/clinical`, `POST /api/payments/vaccination`
- **Mô tả:** Tích hợp PayOS sinh VietQR. Hỗ trợ Webhook xử lý giao dịch tự động và báo cáo thời gian thực qua WebSocket.
- **Các file liên quan:** `payment_controller.go`, `payos.go`, `websocket.go`.

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

### Luồng 4. Phân Hệ Khám Bệnh (Doctor Portal)

- **API Endpoints:** `GET /api/doctor/appointments`, `POST /api/appointment-services/bulk`, `POST /api/prescriptions`
- **Các file liên quan:** `doctor_controller.go`, `models/record.go`.

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

### Luồng 5. Phân Hệ Cận Lâm Sàng (Lab Technician)

- **API Endpoints:** `GET /api/lab/pending-services`, `PUT /api/lab/services/complete`
- **Các file liên quan:** `lab_controller.go`, `models/record.go`.

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

### Luồng 6. Quản Lý Tiêm Chủng (Vaccination)

- **API Endpoints:** `POST /api/vaccinations/*`
- **Các file liên quan:** `vaccination_controller.go`, `models/vaccination.go`.

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

### Luồng 7. Tác Vụ Ngầm (Background Workers)

- **Mô tả:** Chạy tự động 30s/lần với 3 nhiệm vụ: dọn lịch hủy, nhắc nhở trước 5 phút, và nhắc tái khám.
- **Các file liên quan:** `reminder_worker.go`.

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

### Luồng 8. Phân Hệ Admin & Quản Trị Hệ Thống

- **API Endpoints:** `/api/admin/users`, `/api/admin/staff/permissions`
- **Các file liên quan:** `user_controller.go`, `permission_controller.go`.

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
```

### Luồng 9. Quản Lý Lịch Làm Việc & Ca Trực (Doctor Schedule)

- **API Endpoints:** `/api/doctor/schedules/request`, `/api/schedules`
- **Các file liên quan:** `schedule_controller.go`.

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
