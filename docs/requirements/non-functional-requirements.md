# YÊU CẦU PHI CHỨC NĂNG — SHOPVDK (NON-FUNCTIONAL REQUIREMENTS)

> **Mã tài liệu:** REQ-04  
> **Trạng thái:** Bản thảo phê duyệt (Draft)  
> **Giai đoạn:** Phase 01 — Discovery & Requirement Analysis  
> **Dự án:** ShopVDK - Sports E-commerce Platform  

---

## 1. Giới thiệu

Tài liệu này xác định các tiêu chuẩn chất lượng kỹ thuật, hiệu năng, bảo mật và khả năng vận hành của hệ thống ShopVDK. Các yêu cầu phi chức năng (NFR) là cơ sở ràng buộc để thiết kế kiến trúc hệ thống (Phase 03) và đánh giá chất lượng mã nguồn (Phase 06).

---

## 2. Tiêu chuẩn Chi tiết (NFR Specifications)

### 2.1 Hiệu năng và Khả năng Đáp ứng (Performance - NFR-PERF)

- **NFR-PERF-01: Thời gian phản hồi API (API Response Time)**
  - Đối với 95% các yêu cầu đọc (Read Operations: xem trang chủ, danh sách sản phẩm, chi tiết sản phẩm, lọc theo danh mục): Thời gian phản hồi của Backend phải **< 200ms**.
  - Đối với các yêu cầu ghi (Write Operations: đặt hàng, cập nhật giỏ hàng, cập nhật trạng thái đơn): Thời gian xử lý phải **< 500ms**.
- **NFR-PERF-02: Chiến lược Đánh Chỉ mục Cơ sở dữ liệu (Database Indexing)**
  - Bắt buộc đánh chỉ mục **B-Tree** cho: Khóa ngoại (`foreign keys`), các trường tìm kiếm chính xác (`order_code`, `slug`, `email`, `phone`).
  - Bắt buộc đánh chỉ mục **GIN Index (`jsonb_path_ops`)** trên các cột `JSONB` (`specifications`, `attributes`) để đảm bảo tốc độ lọc thuộc tính động tương đương với cột quan hệ truyền thống.
- **NFR-PERF-03: Phân trang Dữ liệu (Mandatory Pagination)**
  - Tất cả các API trả về danh sách (sản phẩm, đơn hàng, khách hàng) bắt buộc phải hỗ trợ phân trang (`page`, `size`, `sort`).
  - Giới hạn kích thước trang mặc định: `12` hoặc `16` sản phẩm/trang. Tuyệt đối không thực hiện truy vấn toàn bộ bảng (`SELECT *`) mà không giới hạn.

---

### 2.2 Bảo mật và An toàn Thông tin (Security - NFR-SEC)

- **NFR-SEC-01: Bảo mật Mật khẩu Người dùng**
  - Mật khẩu của tài khoản khách hàng và quản trị viên bắt buộc phải được mã hóa một chiều bằng thuật toán **BCrypt** (với độ phức tạp `strength >= 10`) trước khi lưu trữ vào PostgreSQL.
  - Tuyệt đối không lưu trữ, ghi log hoặc trả về mật khẩu dạng văn bản thô (Plaintext).
- **NFR-SEC-02: Cơ chế Xác thực & Phân quyền (JWT & RBAC)**
  - Sử dụng **JSON Web Token (JWT)** không trạng thái (Stateless Authentication) cho các API yêu cầu đăng nhập.
  - Phân quyền theo vai trò (Role-based Access Control):
    - `ROLE_CUSTOMER`: Chỉ có quyền truy cập thông tin cá nhân và đơn hàng của chính mình.
    - `ROLE_ADMIN`: Có quyền truy cập các API quản trị `/api/admin/**`.
  - Khóa bí mật (Secret Key) và thời hạn hết hạn (Expiration Time) của JWT phải được cấu hình qua biến môi trường (`.env`).
- **NFR-SEC-03: Kiểm tra Dữ liệu Đầu vào & Chống Tấn công Phổ biến**
  - **SQL Injection:** Toàn bộ truy vấn cơ sở dữ liệu phải thông qua Spring Data JPA / Hibernate với tham số hóa (Parameterized Queries).
  - **XSS & Dữ liệu không hợp lệ:** Sử dụng **Jakarta Bean Validation** (`@NotBlank`, `@Size`, `@Email`, `@Pattern`) trên toàn bộ Request DTOs.
  - **CORS (Cross-Origin Resource Sharing):** Cấu hình chặt chẽ chỉ cho phép tên miền/cổng của Frontend (ví dụ `http://localhost:5173` trong dev) gọi API Backend.
- **NFR-SEC-04: Nguyên tắc Zero-Secret Exposure**
  - Nghiêm cấm đưa mật khẩu database, JWT secret, khóa API vào mã nguồn hoặc lịch sử commit Git. Toàn bộ phải nạp qua file `.env`.
- **NFR-SEC-05: Bảo vệ API Tra cứu Đơn hàng Công khai**
  - Endpoint tra cứu đơn cho khách vãng lai `/api/orders/track` bắt buộc phải khớp chính xác đồng thời cả 2 thông tin: **Mã đơn hàng (`orderCode`)** VÀ **Số điện thoại nhận hàng (`recipientPhone`)** để ngăn chặn việc dò quét đơn hàng của người khác.

---

### 2.3 Độ tin cậy và Tính toàn vẹn Dữ liệu (Reliability & Data Integrity - NFR-REL)

- **NFR-REL-01: Quản lý Giao dịch Nghiêm ngặt (ACID Transactions)**
  - Toàn bộ luồng đặt hàng (Tạo đơn hàng + Lưu chi tiết món + Trừ tồn kho biến thể SKU) phải được bọc trong một giao dịch cơ sở dữ liệu duy nhất (`@Transactional`).
  - Nếu bất kỳ bước nào thất bại (ví dụ: một biến thể bị hết hàng trong lúc đặt), toàn bộ giao dịch phải tự động **Rollback 100%**, không để lại dữ liệu rác hay sai lệch tồn kho.
- **NFR-REL-02: Bảo toàn Dữ liệu Lịch sử (Data Snapshot)**
  - Thông tin người nhận (Tên, SĐT, Email, Địa chỉ giao hàng) tại thời điểm đặt hàng phải được lưu bản sao trực tiếp (Snapshot) vào bảng `orders`, không phụ thuộc vào bảng `users` để đảm bảo tính pháp lý và lịch sử đơn hàng không bị thay đổi.
- **NFR-REL-03: Chuẩn hóa Xử lý Lỗi (Centralized Exception Handling)**
  - Backend sử dụng `@RestControllerAdvice` để bắt và đóng gói tất cả các ngoại lệ thành định dạng JSON chuẩn (RFC 7807 Problem Details hoặc `ApiErrorResponse` chuẩn).
  - Không để lộ Stack Trace hoặc lỗi nội bộ của Database/Spring ra phía client.

---

### 2.4 Trải nghiệm Người dùng & Giao diện (Usability & UI/UX - NFR-USA)

- **NFR-USA-01: Thiết kế Tương thích Đa thiết bị (Responsive Design)**
  - Giao diện Frontend xây dựng bằng Tailwind CSS phải hiển thị tối ưu trên 3 kích thước màn hình chuẩn:
    - **Mobile:** Chiều rộng màn hình < 768px (Menu trượt, bộ lọc dạng Drawer/Modal).
    - **Tablet:** 768px - 1024px.
    - **Desktop:** > 1024px (Thanh lọc Sidebar cố định bên trái).
- **NFR-USA-02: Xử lý Đầy đủ 4 Trạng thái Giao diện (Mandatory UI States)**
  - Mọi trang và component tải dữ liệu bất đồng bộ bắt buộc phải có đủ 4 trạng thái:
    1. **Loading State:** Hiển thị khung xương (Skeleton Loading) hoặc biểu tượng chờ, không để màn hình trắng.
    2. **Empty State:** Hiển thị thông báo thân thiện kèm gợi ý hành động khi không có dữ liệu (ví dụ: Giỏ hàng trống -> Nút "Tiếp tục mua sắm", Tìm kiếm không thấy sản phẩm -> Nút "Xóa bộ lọc").
    3. **Error State:** Hiển thị thông báo lỗi rõ ràng, có nút "Thử lại" khi mất mạng hoặc API lỗi.
    4. **Success State:** Dữ liệu hiển thị trực quan, mượt mà.
- **NFR-USA-03: Tối ưu Tốc độ Đặt hàng Khách vãng lai (Frictionless Checkout)**
  - Luồng Guest Checkout tối đa chỉ gói gọn trong **1 bước (One-page Checkout)** hoặc tối đa 2 bước: điền thông tin giao nhận -> chọn thanh toán COD/Chuyển khoản -> Hoàn tất.

---

### 2.5 Khả năng Bảo trì và Chuẩn hóa Kỹ thuật (Maintainability - NFR-MAINT)

- **NFR-MAINT-01: Kiến trúc Mã nguồn Sạch (Clean Architecture / Modular Monolith)**
  - Mã nguồn Backend phân chia theo tính năng (Package-by-feature) hoặc tầng phân lớp chuẩn:
    - `module.catalog`: Quản lý danh mục, thương hiệu, sản phẩm, biến thể SKU.
    - `module.order`: Quản lý giỏ hàng, đơn hàng, trạng thái vận chuyển.
    - `module.user`: Quản lý tài khoản, xác thực JWT, địa chỉ.
    - `module.common`: Chứa cấu hình bảo mật, xử lý ngoại lệ, tiện ích dùng chung.
- **NFR-MAINT-02: Tài liệu hóa API Tự động (OpenAPI / Swagger)**
  - Tích hợp công cụ `springdoc-openapi` để tự động sinh tài liệu API tương tác tại đường dẫn `/swagger-ui.html`.
  - Toàn bộ Controller, Request Body, Response Schema phải có mô tả rõ ràng để Frontend dễ dàng tích hợp.
- **NFR-MAINT-03: Ghi nhận Nhật ký Hệ thống (Structured Logging)**
  - Sử dụng SLF4J / Logback để ghi log các sự kiện quan trọng: Khách tạo đơn mới, thay đổi trạng thái đơn, lỗi thanh toán/tồn kho.
  - Tuyệt đối không dùng `System.out.println()` trong mã nguồn production.

