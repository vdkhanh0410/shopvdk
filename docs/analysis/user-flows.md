# SƠ ĐỒ LUỒNG NGƯỜI DÙNG VÀ HOẠT ĐỘNG HỆ THỐNG — SHOPVDK (USER FLOWS & SYSTEM WORKFLOWS)

> **Mã tài liệu:** ANA-02  
> **Trạng thái:** Bản thảo phê duyệt (Draft)  
> **Giai đoạn:** Phase 02 — System Analysis & Modeling  
> **Dự án:** ShopVDK - Sports E-commerce Platform  

---

## 1. Tổng Quan Các Luồng Nghiệp Vụ Cốt Lõi

Hệ thống ShopVDK tập trung vào 5 luồng hoạt động chính:

1. **Flow 1: Khám phá & Lọc Động Sản Phẩm Cầu Lông (Catalog & Dynamic Facet Search)**
2. **Flow 2: Đặt Hàng Khách Vãng Lai (Guest Checkout Flow — Không Cần Đăng Nhập)**
3. **Flow 3: Tra Cứu Đơn Hàng Công Khai (Public Order Lookup bằng Mã Đơn + SĐT)**
4. **Flow 4: Vòng Đời Vận Hành Đơn Hàng & Cơ Chế Hoàn Kho (Admin Fulfillment & Restocking)**
5. **Flow 5: Xác Thực & Đăng Ký / Đăng Nhập Thành Viên (Authentication with JWT)**

---

## 2. Chi Tiết Các Sơ Đồ Luồng (Workflow Diagrams)

### 2.1 FLOW 1: Khám Phá & Lọc Động Sản Phẩm Cầu Lông

Luồng người dùng tương tác với bộ lọc động 2 lớp thông qua cơ chế truy vấn **PostgreSQL `JSONB`**:

```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng
    participant FE as React Frontend
    participant BE as Spring Boot API
    participant DB as PostgreSQL 17 (JSONB)

    User->>FE: Truy cập danh mục (vd: /categories/vot-cau-long)
    FE->>BE: GET /api/categories/vot-cau-long/filter-schema
    BE-->>FE: Trả về cấu hình bộ lọc Vợt (Trọng lượng U, Độ cứng, Điểm cân bằng...)
    FE->>User: Hiển thị Sidebar với các checkbox tương ứng
    
    User->>FE: Tích chọn: Hãng Yonex, Bản 4U, Điểm cân bằng Nặng đầu
    FE->>BE: GET /api/products?category=vot-cau-long&brand=yonex&weight=4U&balance=head-heavy&page=0&size=12
    
    Note over BE,DB: Sử dụng toán tử JSONB containment (@>)<br/>kết hợp chỉ mục GIN jsonb_path_ops
    BE->>DB: SELECT * FROM products p JOIN product_variants pv ON ...<br/>WHERE p.specifications @> '{"balance": "head-heavy"}'<br/>AND pv.attributes @> '{"weight": "4U"}'
    DB-->>BE: Danh sách sản phẩm & SKU thỏa mãn
    
    BE-->>FE: HTTP 200 OK (Page data: items, totalPages, totalElements)
    FE->>User: Hiển thị lưới sản phẩm với thời gian phản hồi < 200ms
```

---

### 2.2 FLOW 2: Đặt Hàng Khách Vãng Lai (Guest Checkout Flow) — [TRỌNG TÂM CỐT LÕI]

Luồng xử lý đơn hàng không yêu cầu đăng nhập, tuân thủ nguyên tắc **Fail-Fast** (kiểm tra toàn diện dữ liệu trước khi chạm vào Database) và giao dịch nguyên tử `@Transactional`:

```mermaid
sequenceDiagram
    autonumber
    actor Guest as Khách vãng lai
    participant FE as React Frontend
    participant Cart as Local Storage (Guest Cart)
    participant Ctrl as Spring Boot Controller
    participant Svc as Order Service (@Transactional)
    participant DB as PostgreSQL 17

    Guest->>FE: Bấm "Tiến hành Đặt hàng" từ trang Giỏ hàng
    FE->>Cart: Đọc danh sách SKU và số lượng đặt
    FE->>Guest: Hiển thị Form Đặt hàng (/checkout)
    
    Guest->>FE: Nhập: Tên, SĐT, Email, Địa chỉ giao hàng, Ghi chú<br/>+ Chọn phương thức: COD hoặc Chuyển khoản QR
    Guest->>FE: Nhấn nút "Xác nhận Đặt hàng"
    
    FE->>FE: Client-side Validation sơ bộ (UX feedback nhanh)
    FE->>Ctrl: POST /api/orders/guest-checkout<br/>(Body: recipientInfo, items[skuId, qty], paymentMethod)
    
    Note over Ctrl: BƯỚC BẢO VỆ DATABASE (FAIL-FAST):<br/>Jakarta Bean Validation (@Valid trên DTO)
    alt Payload không hợp lệ (SĐT sai định dạng, thiếu Tên/Địa chỉ, Giỏ hàng rỗng, số lượng <= 0)
        Ctrl-->>FE: HTTP 400 Bad Request (Danh sách lỗi chi tiết theo từng trường)
        Note over Ctrl,DB: DỪNG NGAY LẬP TỨC: Không mở Transaction, không chiếm Connection Pool, không khóa dòng DB!
        FE->>Guest: Báo lỗi đỏ tại các ô nhập liệu tương ứng
    else Payload hợp lệ 100%
        Ctrl->>Svc: Gọi createGuestOrder(validRequest)
        Note over Svc,DB: BẮT ĐẦU MỞ GIAO DỊCH CƠ SỞ DỮ LIỆU (@Transactional)
        
        Svc->>DB: 1. SELECT id, stock_quantity FROM product_variants WHERE id IN (...) FOR UPDATE
        
        alt SKU hết hàng hoàn toàn (stock_quantity == 0)
            DB-->>Svc: stock_quantity == 0
            Note over Svc: Ném OutOfStockException -> ROLLBACK
            Svc-->>Ctrl: OutOfStockException
            Ctrl-->>FE: HTTP 409 Conflict ("Sản phẩm [Tên - SKU] đã hết hàng")
            FE->>Guest: Báo lỗi hết hàng, gợi ý khách chọn biến thể khác
        else SKU không đủ số lượng (0 < stock_quantity < requested_qty)
            DB-->>Svc: 0 < stock_quantity < requested_qty
            Note over Svc: Ném InsufficientStockException -> ROLLBACK
            Svc-->>Ctrl: InsufficientStockException(availableStock)
            Ctrl-->>FE: HTTP 409 Conflict ("Sản phẩm chỉ còn lại [X] chiếc, không đủ số lượng yêu cầu")
            FE->>Guest: Báo lỗi và gợi ý khách giảm số lượng xuống [X] chiếc
        else Tất cả SKU đủ tồn kho (stock_quantity >= requested_qty)
            Svc->>DB: 2. UPDATE product_variants SET stock_quantity = stock_quantity - :qty
            Svc->>DB: 3. INSERT INTO orders (order_code, user_id=NULL, recipient_name, recipient_phone,<br/>recipient_email, shipping_address, payment_method, status='PENDING', total_amount...)
            Svc->>DB: 4. INSERT INTO order_items (order_id, variant_id, price, quantity...)
            Note over Svc,DB: COMMIT Giao dịch thành công & Giải phóng khóa FOR UPDATE
            
            Svc-->>Ctrl: OrderResponseDTO (orderCode, totalAmount, paymentInfo)
            Ctrl-->>FE: HTTP 201 Created
            FE->>Cart: Xóa sạch giỏ hàng tạm (Cart.clear())
            FE->>Guest: Chuyển hướng sang trang /orders/success/:orderCode<br/>(Hiển thị Mã đơn, tóm tắt đơn và mã QR chuyển khoản nếu có)
        end
    end
```

---

### 2.3 FLOW 3: Tra Cứu Đơn Hàng Công Khai (Public Order Lookup)

Cho phép khách vãng lai kiểm tra đơn hàng thời gian thực mà **không cần bất kỳ Token đăng nhập nào**:

```mermaid
flowchart TD
    Start([Khách truy cập /orders/track]) --> Input[Nhập Mã đơn hàng & Số điện thoại]
    Input --> ValidateFE{Kiểm tra định dạng tại Client}
    ValidateFE -- Thiếu thông tin --> ShowFEError[Báo lỗi: Vui lòng nhập đủ Mã đơn và SĐT]
    ValidateFE -- Hợp lệ --> SendAPI[Gửi POST /api/orders/track<br/>Body: orderCode, phone]
    
    SendAPI --> QueryDB[(Query Database:<br/>WHERE order_code = :code<br/>AND recipient_phone = :phone)]
    
    QueryDB --> CheckExist{Có bản ghi khớp đồng thời?}
    CheckExist -- Không khớp --> Error404[Trả về HTTP 404:<br/>Không tìm thấy đơn hàng]
    Error404 --> ShowUIError[Hiển thị: Thông tin đơn hàng không chính xác]
    
    CheckExist -- Khớp dữ liệu --> ReturnData[Trả về HTTP 200 OK:<br/>Chi tiết đơn hàng & Lịch sử trạng thái]
    ReturnData --> RenderTimeline[Hiển thị Timeline đơn hàng:<br/>Chờ xác nhận -> Đã duyệt -> Đang giao -> Hoàn tất]
```

---

### 2.4 FLOW 4: Vận Hành Đơn Hàng Admin & Cơ Chế Hoàn Kho (Admin Fulfillment & Restocking)

Quy trình quản lý trạng thái đơn hàng được thiết kế **tách bạch hoàn toàn giữa hành vi nghiệp vụ ngoài đời (External SOP)** và **lệnh chuyển trạng thái trên hệ thống phần mềm (System Transitions)**:

```mermaid
stateDiagram-v2
    [*] --> PENDING: Khách đặt hàng thành công (Tồn kho đã bị trừ)
    
    note right of PENDING
        Trạng thái PENDING: Đơn chờ xử lý.
        Admin tự thẩm định thông tin ngoại tuyến (gọi điện, kiểm tra Zalo/SMS...).
        Hệ thống không can thiệp vào hành vi vật lý, chỉ chờ lệnh từ Admin.
    end note

    PENDING --> CONFIRMED: Action CONFIRM_ORDER (Admin nhấp "Xác nhận đơn")
    PENDING --> CANCELLED: Action CANCEL_ORDER (Admin nhấp "Hủy đơn" kèm lý do)
    
    CONFIRMED --> SHIPPING: Action SHIP_ORDER (Admin nhấp "Bàn giao vận chuyển")
    CONFIRMED --> CANCELLED: Action CANCEL_ORDER (Khách báo hủy trước khi xuất kho)
    
    SHIPPING --> COMPLETED: Action COMPLETE_ORDER (Giao thành công & Thu tiền COD)
    SHIPPING --> CANCELLED: Action CANCEL_ORDER (Giao thất bại / Khách từ chối nhận)
    
    state CANCELLED {
        [*] --> RestockAction
        RestockAction: Giao dịch @Transactional tự động cộng hoàn lại<br/>stock_quantity cho từng biến thể SKU trong đơn
    }
    
    COMPLETED --> [*]: Đơn hàng kết thúc thành công (Doanh thu ghi nhận)
    CANCELLED --> [*]: Tồn kho đã được hoàn lại 100%
```

---

### 2.5 FLOW 5: Xác Thực Khách Hàng & Quản Trị Viên (Authentication with JWT)

Thiết kế tối ưu theo thói quen E-commerce tại Việt Nam: Người dùng đăng nhập linh hoạt bằng **Số điện thoại HOẶC Email**, backend xử lý truy vấn kép trên chỉ mục:

```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng / Admin
    participant FE as React Frontend
    participant BE as Spring Boot Security
    participant DB as PostgreSQL 17

    User->>FE: Nhập [Số điện thoại hoặc Email] + Mật khẩu tại trang /login
    FE->>BE: POST /api/auth/login<br/>Body: { "identifier": "0987654321", "password": "..." }
    
    Note over BE,DB: Truy vấn linh hoạt bằng 1 trong 2 thông tin định danh<br/>(Được tối ưu bằng B-Tree Index trên cả cột phone và email)
    BE->>DB: SELECT * FROM users WHERE (email = :identifier OR phone = :identifier) AND is_active = true
    
    alt Không tìm thấy tài khoản
        DB-->>BE: Null
        BE-->>FE: HTTP 401 Unauthorized ("Tài khoản hoặc mật khẩu không chính xác")
        FE->>User: Báo lỗi đăng nhập
    else Tìm thấy tài khoản
        DB-->>BE: User entity (chứa password_hash BCrypt và roles)
        BE->>BE: PasswordEncoder.matches(rawPassword, encodedPassword)
        
        alt Mật khẩu không trùng khớp
            BE-->>FE: HTTP 401 Unauthorized ("Tài khoản hoặc mật khẩu không chính xác")
            FE->>User: Báo lỗi đăng nhập
        else Mật khẩu chính xác
            BE->>BE: JwtProvider.generateToken(user.getId(), user.getEmail(), user.getRoles())
            BE-->>FE: HTTP 200 OK (accessToken, tokenType: "Bearer", userProfile)
            FE->>FE: Lưu token vào bộ nhớ an toàn (Secure client storage)
            
            alt Vai trò là ROLE_ADMIN
                FE->>User: Chuyển hướng sang trang Quản trị /admin/dashboard
            else Vai trò là ROLE_CUSTOMER
                FE->>User: Chuyển hướng sang trang /account/profile hoặc tiếp tục mua sắm
            end
        end
    end
```

