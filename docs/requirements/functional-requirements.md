# YÊU CẦU CHỨC NĂNG CHI TIẾT — SHOPVDK (FUNCTIONAL REQUIREMENTS)

> **Mã tài liệu:** REQ-03  
> **Trạng thái:** Bản thảo phê duyệt (Draft)  
> **Giai đoạn:** Phase 01 — Discovery & Requirement Analysis  
> **Dự án:** ShopVDK - Sports E-commerce Platform  

---

## 1. Tổng quan Kiến trúc Chức năng

Hệ thống ShopVDK được cấu thành từ 6 phân hệ nghiệp vụ chính:

```mermaid
flowchart TD
    subgraph Storefront [Phân hệ Khách hàng]
        AUTH[1. Xác thực & Tài khoản]
        CATALOG[2. Danh mục & Sản phẩm]
        SEARCH[3. Tìm kiếm & Bộ lọc thể thao]
        CART[4. Quản lý Giỏ hàng]
        CHECKOUT[5. Đặt hàng: Guest & Member Checkout]
        TRACK[6. Tra cứu đơn hàng bằng Mã + SĐT]
    end
    
    subgraph BackOffice [Phân hệ Quản trị]
        ADMIN_CATALOG[Quản lý Sản phẩm, Hãng & Ma trận SKU]
        ADMIN_ORDER[Quản lý Đơn hàng & Vận hành]
        ADMIN_DASH[Dashboard Thống kê]
    end
    
    CATALOG --> CART --> CHECKOUT --> TRACK
    CHECKOUT --> ADMIN_ORDER
    ADMIN_CATALOG --> CATALOG
```

---

## 2. Chi tiết Yêu cầu Chức năng (Functional Modules)

### MODULE 1: XÁC THỰC VÀ PHÂN QUYỀN (FR-AUTH)

- **FR-AUTH-01: Đăng ký tài khoản Khách hàng (Register)**
  - Người dùng có thể đăng ký bằng Email, Mật khẩu, Họ tên, Số điện thoại.
  - Mật khẩu phải được băm bảo mật (BCrypt) trước khi lưu trữ.
  - Phân quyền mặc định là `ROLE_CUSTOMER`.
- **FR-AUTH-02: Đăng nhập & Cấp Token (Login)**
  - Cho phép người dùng đăng nhập linh hoạt bằng **Số điện thoại HOẶC Email**, kèm Mật khẩu.
  - Backend thực hiện truy vấn kép trên chỉ mục `email` và `phone` để xác thực danh tính.
  - Trả về JSON Web Token (JWT) cho các yêu cầu cần xác thực.
- **FR-AUTH-03: Đăng nhập Quản trị (Admin Login)**
  - Cổng đăng nhập bảo mật riêng cho nhân sự có quyền `ROLE_ADMIN`.
- **FR-AUTH-04: Nguyên tắc Tự do Mua sắm (CRITICAL)**
  - **Hệ thống TUYỆT ĐỐI KHÔNG ép buộc người dùng phải đăng nhập mới được mua hàng.** Khách vãng lai có quyền xem hàng, chọn biến thể, thêm giỏ và tiến hành Đặt hàng (Guest Checkout) hoàn toàn bình thường.

---

### MODULE 2: DANH MỤC, THƯƠNG HIỆU & SẢN PHẨM (FR-CATALOG)

- **FR-CATALOG-01: Quản lý Danh mục (MVP Scope: Chuyên sâu Cầu Lông)**
  - **Phạm vi MVP:** Tập trung duy nhất vào hệ sinh thái sản phẩm Cầu lông (Badminton):
    - Cây danh mục cấp 1: *Cầu lông*
    - Nhóm sản phẩm cấp 2: *Vợt cầu lông, Giày cầu lông, Quần áo cầu lông, Túi & Balo cầu lông, Cước căng vợt, Phụ kiện cầu lông*.
  - **Khả năng mở rộng (Post-MVP):** Cấu trúc bảng `categories` hỗ trợ phân cấp cha - con (`parent_id`) không giới hạn, sẵn sàng mở rộng sang *Tennis*, *Pickleball* sau khi hoàn thành MVP mà không cần thay đổi cấu trúc bảng hay code logic.
- **FR-CATALOG-02: Quản lý Thương hiệu (Brands)**
  - Danh sách các hãng cầu lông chính hãng: Yonex, Victor, Lining, Mizuno, Apacs, Kumpoo, Kawasaki, VS (Venson)...
- **FR-CATALOG-03: Quản lý Ma trận Biến thể Sản phẩm (Product Variants & SKUs)**
  - Một sản phẩm cha (Product) có nhiều biến thể con (Product Variants / SKUs):
    - **Vợt cầu lông:** Biến thể theo *Trọng lượng / Chu vi cán* (`3U/G5`, `4U/G5`, `4U/G6`).
    - **Giày cầu lông:** Biến thể theo *Size giày EU* (36, 37, 38, 39, 40, 41, 42, 43, 44, 45).
    - **Quần áo cầu lông:** Biến thể theo *Size* (S, M, L, XL, XXL) và *Màu sắc*.
    - **Cước & Phụ kiện:** Biến thể theo *Màu sắc cước/quấn cán* hoặc *Quy cách đóng gói* (Vỉ đơn / Cuộn lớn).
- **FR-CATALOG-04: Quản lý Tồn kho Biến thể (SKU Inventory)**
  - Mỗi SKU biến thể có: Mã SKU riêng biệt, Giá bán riêng (nếu có chênh lệch), Số lượng tồn kho (`quantity`).
  - Khi một biến thể hết hàng, hệ thống hiển thị "Tạm hết hàng" cho riêng biến thể đó, các biến thể khác vẫn bán bình thường.
- **FR-CATALOG-05: Thông số Kỹ thuật Thể thao (Sports Technical Specs)**
  - Bảng thông số chi tiết hỗ trợ người chơi thể thao ra quyết định:
    - Vợt: Điểm cân bằng (Nặng đầu / Cân bằng / Nhẹ đầu), Độ cứng thân vợt (Dẻo / Trung bình / Cứng), Mức căng tối đa (`lbs`).

---

### MODULE 3: TÌM KIẾM VÀ BỘ LỌC (FR-SEARCH)

> [!IMPORTANT]
> **Tích hợp Nghiệp vụ Thực tế:**
> Bộ lọc dưới đây được trích xuất trực tiếp từ tài liệu nghiệp vụ ShopVNB và chuẩn hóa 100% cho hệ sinh thái sản phẩm **Cầu Lông** của dự án ShopVDK.
> - Tổ chức theo mô hình **2 lớp**: (1) Bộ lọc chung xuất hiện ở mọi trang danh mục; (2) Bộ lọc chuyên biệt cho từng loại danh mục sản phẩm cầu lông.
> - **Đã loại bỏ:** Tiêu chí "Chi nhánh" (vì ShopVDK quản lý tồn kho tại kho trung tâm, không phân mạng lưới chi nhánh như ShopVNB).

- **FR-SEARCH-01: Tìm kiếm theo từ khóa (Text Search)**
  - Tìm kiếm theo tên sản phẩm, thương hiệu hoặc mã sản phẩm.
  - Xử lý trạng thái rỗng (Empty State) khi không tìm thấy kết quả phù hợp.

- **FR-SEARCH-02: Bộ lọc Động theo Danh mục (Category-specific Dynamic Filtering)**
  - **Nguyên tắc nghiệp vụ:** Khi khách hàng duyệt vào một Danh mục cụ thể, Filter Sidebar sẽ tự động điều chỉnh hiển thị đúng các tiêu chí của danh mục đó (không hiện thuộc tính của danh mục khác).

---

#### A. BỘ LỌC CHUNG (Áp dụng cho MỌI danh mục)

| STT | Nhóm lọc | Mô tả | Giá trị lọc | Vị trí lưu trữ DB |
|---|---|---|---|---|
| 1 | **Thương hiệu** | Lọc theo nhãn hiệu nhà sản xuất | Yonex · Victor · Lining · Mizuno · Apacs · Kumpoo · Kawasaki · VS · Taro · Beyono... | `products.brand_id` |
| 2 | **Mức giá** | Phân khúc ngân sách định sẵn | Dưới 500.000đ · 500.000đ – 1 triệu · 1 – 2 triệu · 2 – 3 triệu · Trên 3 triệu | `product_variants.price` |
| 3 | **Tình trạng hàng** | Trạng thái tồn kho | Còn hàng trong kho · Tất cả | `product_variants.stock_quantity > 0` |

---

#### B. BỘ LỌC VỢT CẦU LÔNG (`/categories/vot-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Trọng lượng (Weight)** | Phân nhóm U/F/2F theo lực cổ tay | **2U** (90-94g) · **3U** (85-89g) · **4U** (80-84g) · **5U** (75-79g) · **F** (70-74g) · **2F** (<70g) | `product_variants.attributes.weight` *(SKU)* |
| 2 | **Điểm cân bằng** | Xác định độ đầm đầu vợt | **Nhẹ đầu** (Thủ/phản tạt) · **Cân bằng** (Công thủ toàn diện) · **Hơi nặng đầu** · **Nặng đầu** (Tấn công/đập cầu) | `products.specifications.balance` |
| 3 | **Độ cứng đũa** | Mức độ uốn thân & trợ lực | **Dẻo** (Trợ lực tốt) · **Trung bình** · **Cứng** · **Siêu cứng** (Kiểm soát cao) | `products.specifications.stiffness` |
| 4 | **Phong cách chơi** | Lối đánh của người chơi | **Tấn công** · **Công thủ toàn diện** · **Phòng thủ / Phản tạt** | `products.specifications.play_style` |
| 5 | **Nội dung chơi** | Thể thức thi đấu phù hợp | **Đánh đơn** · **Đánh đôi** · **Cả hai** | `products.specifications.game_type` |
| 6 | **Trình độ chơi** | Trình độ kinh nghiệm | **Mới chơi** · **Trung bình (Phong trào)** · **Khá / Tốt** · **Chuyên nghiệp** | `products.specifications.player_level` |

---

#### C. BỘ LỌC GIÀY CẦU LÔNG (`/categories/giay-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Size giày (EU)** | Kích cỡ chân người mua | **36 · 37 · 38 · 39 · 40 · 40.5 · 41 · 42 · 42.5 · 43 · 44 · 44.5 · 45** | `product_variants.attributes.size` *(SKU)* |
| 2 | **Đối tượng** | Nhóm người dùng phù hợp | **Nam** · **Nữ** · **Trẻ em** · **Unisex** | `products.specifications.target_audience` |
| 3 | **Form giày** | Độ rộng bàn chân | **Slim** (Chân thon) · **Unisex** (Tiêu chuẩn) · **Wide** (Chân bè ngang 3E/4E) | `products.specifications.shoe_fit` |
| 4 | **Điểm nổi bật** | Đặc tính công nghệ giày | **Bám sân tốt** · **Chống lật cổ chân** · **Đệm êm giảm chấn (Cushion)** · **Thoáng khí** · **Siêu nhẹ** | `products.specifications.highlights` |
| 5 | **Thể loại chơi** | Mặt sân phù hợp | **Trong nhà (Thảm PVC/gỗ)** · **Ngoài trời (Bê tông/xi măng)** · **Cả hai** | `products.specifications.surface_type` |
| 6 | **Phân khúc** | Cấp độ dòng sản phẩm | **Tầm thấp (Giá rẻ)** · **Trung cấp (Phong trào)** · **Cao cấp (Chuyên nghiệp)** | `products.specifications.tier` |

---

#### D. BỘ LỌC TÚI VỢT CẦU LÔNG (`/categories/tui-vot-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Số ngăn lớn** | Sức chứa theo số ngăn | **1 ngăn lớn** · **2 ngăn lớn** (3-6 cây) · **3 ngăn lớn** (6-9 cây) | `products.specifications.compartment_count` |
| 2 | **Thể loại túi** | Phân loại kiểu dáng túi | **Túi chữ nhật (Bao kiếm)** · **Túi du lịch xách tay** · **Túi vuông thời trang** | `products.specifications.bag_type` |
| 3 | **Điểm nổi bật** | Tính năng tiện ích đặc thù | **Ngăn cách nhiệt (Thermal)** · **Ngăn để giày riêng** · **Vải chống thấm nước** | `products.specifications.highlights` |
| 4 | **Đối tượng** | Nhóm người dùng | **Nam** · **Nữ** · **Unisex** | `products.specifications.target_audience` |

---

#### E. BỘ LỌC BALO CẦU LÔNG (`/categories/balo-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Thể loại balo** | Kiểu dáng thiết kế | **Balo đứng truyền thống** · **Balo vuông thể thao** · **Balo thời trang đa năng** | `products.specifications.backpack_type` |
| 2 | **Điểm nổi bật** | Tiện ích chuyên dụng | **Ngăn cài vợt riêng** · **Ngăn để giày riêng bên dưới** · **Đệm lưng quai đeo trợ lực** · **Chống thấm nước** | `products.specifications.highlights` |
| 3 | **Đối tượng** | Nhóm người dùng | **Nam** · **Nữ** · **Trẻ em** · **Unisex** | `products.specifications.target_audience` |

---

#### F. BỘ LỌC ÁO, QUẦN VÀ VÁY CẦU LÔNG (`/categories/quan-ao-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Size trang phục** | Kích cỡ theo vóc dáng | **XS · S · M · L · XL · XXL · XXXL** | `product_variants.attributes.size` *(SKU)* |
| 2 | **Đối tượng** | Giới tính và độ tuổi | **Nam** · **Nữ** · **Trẻ em** · **Unisex** | `products.specifications.target_audience` |
| 3 | **Thể loại** | Loại trang phục thi đấu/tập | **Áo thun cộc tay** · **Áo có cổ (Polo)** · **Quần soóc/đùi** · **Váy / Quần váy nữ** · **Áo khoác gió** | `products.specifications.apparel_type` |
| 4 | **Màu sắc** | Màu sắc chủ đạo | **Đen · Trắng · Xanh dương · Đỏ · Vàng · Cam · Xanh lá · Hồng · Tím · Xám** | `product_variants.attributes.color` *(SKU)* |

---

#### G. BỘ LỌC CƯỚC CẦU LÔNG (`/categories/cuoc-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Đường kính dây (Gauge)** | Độ dày sợi cước đan vợt | **Siêu mỏng** (0.61 – 0.65mm) · **Trung bình** (0.66 – 0.68mm) · **Dày / Bền** (0.69 – 0.70mm) | `products.specifications.gauge_mm` |
| 2 | **Đặc tính nổi trội** | Trải nghiệm khi đánh cầu | **Trợ lực / Nảy cầu** · **Âm nổ to vang** · **Độ bền cao** · **Cảm giác kiểm soát** · **Tạo xoáy** | `products.specifications.features` |
| 3 | **Màu dây / Quy cách** | Tùy chọn đóng gói | Màu sắc (Trắng, Vàng, Cam, Đen...) · Quy cách (Vỉ đơn 10m, Cuộn lớn 200m) | `product_variants.attributes` *(SKU)* |

---

#### H. BỘ LỌC PHỤ KIỆN CẦU LÔNG (`/categories/phu-kien-cau-long`)

| STT | Nhóm lọc | Mô tả nghiệp vụ | Giá trị chi tiết | Thuộc tính JSONB |
|---|---|---|---|---|
| 1 | **Thể loại phụ kiện** | Phân loại trang bị bổ trợ | **Quấn cán cao su (Overgrip)** · **Quấn cán vải (Towel grip)** · **Quấn lót cốt cán (Cushion wrap)** · **Vớ / Tất thể thao** · **Băng chặn mồ hôi cổ tay/trán** · **Móc chìa khóa** | `products.specifications.accessory_type` |

- **FR-SEARCH-03: Sắp xếp (Sorting)**
  - Giá tăng dần, Giá giảm dần, Mới nhất, Bán chạy nhất.

---

### MODULE 4: GIỎ HÀNG (FR-CART)

- **FR-CART-01: Giỏ hàng Khách vãng lai (Guest Cart)**
  - Lưu trữ phía Client (Local Storage) để khách chưa đăng nhập vẫn lưu được món hàng khi tải lại trang hoặc tắt trình duyệt.
- **FR-CART-02: Thao tác Giỏ hàng**
  - Thêm sản phẩm kèm biến thể (SKU) đã chọn.
  - Tăng / giảm số lượng (không cho phép chọn vượt quá tồn kho thực tế).
  - Xóa sản phẩm khỏi giỏ.
  - Tự động tính toán tổng tiền tạm tính.

---

### MODULE 5: ĐẶT HÀNG & MUA HÀNG KHÔNG CẦN ĐĂNG NHẬP (FR-ORDER) — [TRỌNG TÂM]

- **FR-ORDER-01: Luồng Mua Hàng Khách Vãng Lai (Guest Checkout)**
  - Khách hàng không cần đăng nhập hay tạo tài khoản.
  - Biểu mẫu đặt hàng chỉ yêu cầu nhập các trường thông tin giao hàng trực tiếp:
    1. **Họ và tên người nhận** (`recipient_name` - Bắt buộc)
    2. **Số điện thoại nhận hàng** (`recipient_phone` - Bắt buộc, kiểm tra định dạng SĐT Việt Nam 10 chữ số)
    3. **Địa chỉ email** (`recipient_email` - Bắt buộc để nhận thông báo đơn hàng)
    4. **Địa chỉ giao hàng** (`shipping_address` - Bắt buộc: Tỉnh/Thành phố, Quận/Huyện, Phường/Xã, Số nhà/Tên đường)
    5. **Ghi chú đơn hàng** (`order_notes` - Tùy chọn)
- **FR-ORDER-02: Luồng Mua Hàng Thành Viên (Member Checkout)**
  - Khách đã đăng nhập hệ thống sẽ tự động điền sẵn Họ tên, SĐT và Địa chỉ mặc định từ hồ sơ cá nhân.
- **FR-ORDER-03: Tạo Mã Đơn Hàng Duy Nhất (Order Code)**
  - Hệ thống tự sinh mã đơn hàng thân thiện cho con người, ví dụ: `VDK-20261112-8921` (không dùng số ID tuần tự lộ dữ liệu).
- **FR-ORDER-04: Phương thức Thanh toán (Payment Methods)**
  - **COD (Cash on Delivery):** Thanh toán tiền mặt khi nhận hàng (phổ biến nhất).
  - **Chuyển khoản ngân hàng trực tiếp (Bank Transfer):** Hiển thị thông tin tài khoản ngân hàng, mã QR và Cú pháp chuyển khoản chứa `Order Code`.
- **FR-ORDER-05: Trừ Kho Thời Gian Thực (Inventory Reservation)**
  - Ngay khi đặt hàng thành công, số lượng tồn kho của các biến thể (SKUs) tương ứng trong đơn hàng sẽ lập tức bị trừ tương ứng.
- **FR-ORDER-06: Vòng đời & Trạng thái Đơn hàng (Order State Machine)**

```mermaid
stateDiagram-v2
    [*] --> PENDING: Khách đặt hàng thành công
    PENDING --> CONFIRMED: Admin gọi điện / kiểm tra xác nhận
    CONFIRMED --> SHIPPING: Đã đóng gói & bàn giao vận chuyển
    SHIPPING --> COMPLETED: Giao hàng thành công & Thu tiền COD
    PENDING --> CANCELLED: Khách yêu cầu hủy / Không liên lạc được
    CONFIRMED --> CANCELLED: Hủy trước khi xuất kho
    CANCELLED --> [*]: Hoàn trả tồn kho tự động
    COMPLETED --> [*]
```

- **FR-ORDER-07: Cơ chế Hoàn trả Tồn kho khi Hủy Đơn (Stock Release)**
  - Khi đơn hàng chuyển sang trạng thái `CANCELLED`, hệ thống tự động cộng hoàn lại số lượng tồn kho cho các SKU tương ứng.

---

### MODULE 6: TRA CỨU ĐƠN HÀNG KHÔNG CẦN TOKEN (FR-TRACKING)

- **FR-TRACKING-01: Tra cứu đơn hàng công khai (Public Order Lookup)**
  - Khách vãng lai truy cập trang `/orders/track`.
  - Nhập **Mã đơn hàng** (`orderCode`) + **Số điện thoại nhận hàng** (`recipientPhone`).
  - Hệ thống kiểm tra cặp điều kiện: nếu khớp chính xác, trả về chi tiết đơn hàng (Danh sách món, tổng tiền, trạng thái đơn hiện tại).
  - **Không yêu cầu bất kỳ Authentication Token nào**, đảm bảo tính bảo mật vừa đủ mà tiện lợi tối đa cho khách vãng lai.
- **FR-TRACKING-02: Tra cứu cho Khách hàng thành viên**
  - Khách hàng đã đăng nhập có thể xem toàn bộ lịch sử đơn hàng của mình tại trang `/account/orders`.

---

### MODULE 7: QUẢN TRỊ CỬA HÀNG (FR-ADMIN)

- **FR-ADMIN-01: Bảng số liệu Tổng quan (Dashboard)**
  - Tổng số đơn chờ duyệt, tổng doanh thu trong ngày/tháng, sản phẩm sắp hết hàng.
- **FR-ADMIN-02: Quản lý Đơn hàng**
  - Xem danh sách đơn, lọc theo trạng thái (`PENDING`, `CONFIRMED`, `SHIPPING`, `COMPLETED`, `CANCELLED`).
  - Xem chi tiết từng đơn hàng, thay đổi trạng thái đơn, ghi chú lý do nếu hủy đơn.
- **FR-ADMIN-03: Quản lý Danh mục & Sản phẩm**
  - Thêm/sửa danh mục, thêm sản phẩm mới kèm danh sách biến thể SKU và số lượng kho ban đầu.

---

## 3. Ràng Buộc Thiết Kế Cơ Sở Dữ Liệu (Database Design Constraints)

### 3.1 Bảng `orders` (Quản lý Đơn hàng & Guest Checkout)
> [!IMPORTANT]
> **Yêu cầu kỹ thuật bắt buộc cho Bảng `orders` ở Phase 03:**
> 1. **Cột `user_id` PHẢI là `NULLABLE`:**
>    - Khi đơn hàng do Khách vãng lai đặt (Guest): `user_id = NULL`.
>    - Khi đơn hàng do Thành viên đặt (Member): `user_id = <ID của user>`.
> 2. **Lưu trữ trực tiếp thông tin giao hàng dạng Snapshot trên bảng `orders`:**
>    - `recipient_name VARCHAR(100) NOT NULL`
>    - `recipient_phone VARCHAR(20) NOT NULL`
>    - `recipient_email VARCHAR(150) NOT NULL`
>    - `shipping_address TEXT NOT NULL`
>    - `order_notes TEXT NULL`
>    - `order_code VARCHAR(50) UNIQUE NOT NULL`
>    - `order_status VARCHAR(30) NOT NULL`
>    - `payment_method VARCHAR(30) NOT NULL`
>    - `total_amount NUMERIC(15, 2) NOT NULL`
> 3. **Lý do kỹ thuật (Data Snapshot):** Không được phụ thuộc vào bảng `users` để lấy địa chỉ người nhận, vì nếu sau này user đổi địa chỉ nhà, các đơn hàng trong quá khứ không được phép bị thay đổi thông tin lịch sử giao nhận.

---

### 3.2 Bảng `products` & `product_variants` (Xử lý Thuộc tính Động bằng PostgreSQL JSONB)
> [!TIP]
> **Chiến lược Kiến trúc Database cho Thuộc tính Thể thao Động ở Phase 03:**
> 1. **Vấn đề của mô hình truyền thống (Anti-patterns):**
>    - Nếu tạo riêng từng cột (`stiffness`, `weight_grip`, `shoe_size`, `string_gauge`...) trên bảng `products`, bảng sẽ bị loãng (Sparse Columns), hàng loạt cột bị `NULL` vô ích (ví dụ: sản phẩm Áo quần thì cột `max_tension_lbs` bị null).
>    - Nếu dùng mô hình **EAV (Entity-Attribute-Value)** với các bảng `attributes` và `attribute_values`, câu lệnh SQL sẽ cực kỳ nặng nề, phải `JOIN` từ 5-10 bảng chỉ để lấy thông số hiển thị hoặc lọc sản phẩm, làm giảm hiệu năng nghiêm trọng.
> 2. **Giải pháp Tối ưu: Ứng dụng kiểu dữ liệu `JSONB` của PostgreSQL 17:**
>    - Bảng `products`: Bổ sung cột **`specifications JSONB NOT NULL DEFAULT '{}'`** để lưu trữ thông số kỹ thuật động theo từng loại danh mục.
>      - *Ví dụ Vợt:* `{"balance": "Head-heavy", "stiffness": "Stiff", "max_tension": 30}`
>      - *Ví dụ Giày:* `{"surface": "Indoor / Carpet", "technology": "Power Cushion"}`
>    - Bảng `product_variants`: Bổ sung cột **`attributes JSONB NOT NULL DEFAULT '{}'`** để định nghĩa biến thể cụ thể của SKU.
>      - *Ví dụ SKU Vợt:* `{"weight": "4U", "grip": "G5"}`
>      - *Ví dụ SKU Giày:* `{"size": "41"}`
>      - *Ví dụ SKU Áo:* `{"size": "L", "color": "Navy Blue"}`
> 3. **Chiến lược Đánh Chỉ mục Tối ưu (Indexing Strategy):**
>    - Đánh **GIN Index (`jsonb_path_ops`)** trên cột `specifications` và `attributes`.
>    - Giúp câu lệnh truy vấn lọc động (sử dụng toán tử `@>` của PostgreSQL) đạt tốc độ thực thi tương đương với các cột quan hệ thông thường mà vẫn giữ cho Database Schema cực kỳ tinh gọn, linh hoạt và không cần chạy migrations mỗi khi có môn thể thao mới.
> 4. **Bảo đảm tính Mở rộng Đa môn Thể thao (Zero-Code Backend Extension):**
>    - Dù MVP chỉ chạy Cầu lông, sau này khi thêm môn Tennis (với các thuộc tính `head_size`, `unstrung_weight`) hay Pickleball (`core_thickness`), lập trình viên chỉ cần tạo thêm Category và đẩy dữ liệu JSON tương ứng vào database.
>    - **Tuyệt đối không cần viết thêm bảng mới, không cần sửa Entity Java, không cần migration DDL.** Backend Spring Boot hoàn toàn không phải sửa đổi code!

