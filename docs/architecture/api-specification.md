# THIẾT KẾ ĐẶC TẢ GIAO DIỆN LẬP TRÌNH — SHOPVDK (RESTFUL API SPECIFICATION)

> **Mã tài liệu:** DES-02  
> **Trạng thái:** Bản thảo phê duyệt (Draft)  
> **Giai đoạn:** Phase 03 — System Design & Architecture  
> **Dự án:** ShopVDK - Sports E-commerce Platform  
> **Phiên bản API:** `v1` (Base URL: `/api/v1`)  
> **Chuẩn dữ liệu:** JSON (`application/json; charset=UTF-8`)  

---

## 1. Nguyên Tắc Thiết Kế API & Chuẩn Dữ Liệu Chung

Hợp đồng API (API Contract) này đóng vai trò là "giao kèo kỹ thuật" bất biến giữa:
- **Backend (Spring Boot 3.x - Do Project Owner triển khai):** Cung cấp các RESTful Endpoints, kiểm soát toàn vẹn dữ liệu, xác thực JWT và xử lý Business Rules.
- **Frontend (React + TypeScript - Do AI triển khai):** Tiêu thụ (consume) API, hiển thị giao diện, điều phối trạng thái và quản lý giỏ hàng/thanh toán.

### 1.1 Quy ước Nền tảng (Foundational Conventions)
1. **URI Path Naming:** Sử dụng chữ thường, danh từ số nhiều, phân tách từ bằng dấu gạch ngang (`kebab-case`), ví dụ: `/api/v1/products`, `/api/v1/product-variants`.
2. **JSON Property Naming:** Sử dụng chuẩn `camelCase` cho toàn bộ Request Body và Response Data (ví dụ: `orderCode`, `stockQuantity`, `recipientPhone`).
3. **HTTP Verbs:** Tuân thủ ngữ nghĩa chuẩn RESTful:
   - `GET`: Truy vấn tài nguyên (Idempotent & Safe, không làm thay đổi trạng thái hệ thống).
   - `POST`: Tạo mới tài nguyên hoặc kích hoạt tác vụ nghiệp vụ quan trọng (đặt hàng, đăng nhập).
   - `PUT`: Cập nhật toàn diện tài nguyên (Idempotent).
   - `PATCH`: Cập nhật từng phần (Partial Update) hoặc thay đổi trạng thái (State Transition).
   - `DELETE`: Xóa mềm (`is_active = false`) hoặc xóa cứng tài nguyên.
4. **Timezone & Định dạng Ngày giờ:** Chuẩn ISO-8601 UTC với offset, ví dụ: `2026-11-14T21:45:00.000Z` hoặc `2026-11-14T21:45:00+07:00`.
5. **Tiền tệ & Số học (Monetary Values):** Giá tiền trả về kiểu số thực tế `BigDecimal` (được JSON serialize dạng `number` trong dải an toàn, đơn vị: VNĐ).

---

## 2. Cấu Trúc Khung Phản Hồi Chuẩn (Standard Response Envelopes)

Nhằm giúp Frontend xử lý đồng bộ việc bóc tách dữ liệu và bắt lỗi (Error Handling), toàn bộ API phản hồi từ Backend đều được bọc trong các lớp cấu trúc cố định sau:

### 2.1 Cấu trúc Phản hồi Thành công Đơn lẻ (`ApiResponse<T>`)

Sử dụng cho các thao tác tạo mới, lấy chi tiết, cập nhật hoặc xóa đơn lẻ:

```json
{
  "success": true,
  "message": "Thao tác thành công",
  "data": { ... },
  "timestamp": "2026-11-14T21:45:00.000Z"
}
```

### 2.2 Cấu trúc Phản hồi Phân trang (`PageResponse<T>`)

Sử dụng cho tất cả các danh sách có phân trang và bộ lọc (Catalog Products, Orders, Admin Tables):

```json
{
  "success": true,
  "message": "Lấy danh sách dữ liệu thành công",
  "data": {
    "items": [ ... ],
    "page": 0,
    "size": 12,
    "totalElements": 48,
    "totalPages": 4,
    "isFirst": true,
    "isLast": false,
    "hasNext": true,
    "hasPrevious": false
  },
  "timestamp": "2026-11-14T21:45:00.000Z"
}
```

### 2.3 Cấu trúc Phản hồi Lỗi Chuẩn (`ApiErrorResponse`)

Áp dụng chuẩn RFC 7807 (Problem Details) với thông điệp người dùng thân thiện, chi tiết kiểm tra hợp lệ dữ liệu (Bean Validation errors) và mã lỗi nghiệp vụ độc lập:

```json
{
  "success": false,
  "status": 400,
  "errorCode": "INVALID_PAYLOAD",
  "message": "Dữ liệu gửi lên không hợp lệ, vui lòng kiểm tra lại",
  "path": "/api/v1/orders/guest-checkout",
  "timestamp": "2026-11-14T21:45:00.000Z",
  "errors": [
    {
      "field": "recipientPhone",
      "rejectedValue": "12345",
      "reason": "Số điện thoại người nhận không hợp lệ (yêu cầu 10 chữ số chuẩn định dạng Việt Nam)"
    },
    {
      "field": "items[0].quantity",
      "rejectedValue": 0,
      "reason": "Số lượng đặt mua tối thiểu phải từ 1"
    }
  ]
}
```

---

## 3. Xác Thực & Phân Quyền (Authentication & Security)

### 3.1 Cơ Chế Token JWT
- **Loại Token:** Bearer Token JWT (JSON Web Token), thuật toán ký `HMAC-SHA256` hoặc `HMAC-SHA512`.
- **Header truyền tải:**
  ```http
  Authorization: Bearer <access_token>
  ```
- **Thời hạn hiệu lực:** Access Token có thời hạn mặc định là `24 giờ` (cho MVP).
- **Phân loại vai trò người dùng (Roles):**
  - `ROLE_CUSTOMER`: Người dùng thành viên đã đăng ký tài khoản.
  - `ROLE_ADMIN`: Quản trị viên hệ thống có toàn quyền quản lý kho, sản phẩm và đơn hàng.

### 3.2 Ma Trận Phân Quyền Endpoints (Endpoint Security Matrix)

| Nhóm Endpoint | Đường Dẫn | Phương Thức | Quyền Truy Cập (Access) | Ghi Chú |
| :--- | :--- | :--- | :--- | :--- |
| **Auth** | `/api/v1/auth/register` | `POST` | `Public` (Bất kỳ ai) | Đăng ký thành viên mới |
| **Auth** | `/api/v1/auth/login` | `POST` | `Public` (Bất kỳ ai) | Đăng nhập bằng Email/SĐT |
| **Auth** | `/api/v1/auth/me` | `GET` | `Authenticated` | Lấy thông tin tài khoản hiện tại |
| **Catalog** | `/api/v1/categories/**` | `GET` | `Public` | Xem danh mục sản phẩm |
| **Catalog** | `/api/v1/brands/**` | `GET` | `Public` | Xem thương hiệu |
| **Catalog** | `/api/v1/products/**` | `GET` | `Public` | Xem và lọc sản phẩm, SKU |
| **Order** | `/api/v1/orders/guest-checkout` | `POST` | `Public` | Đặt hàng không cần đăng nhập |
| **Order** | `/api/v1/orders/checkout` | `POST` | `ROLE_CUSTOMER` | Đặt hàng cho thành viên |
| **Order** | `/api/v1/orders/track` | `GET` | `Public` | Tra cứu đơn hàng bằng SĐT + Mã đơn |
| **Customer** | `/api/v1/customer/orders/**` | `GET` | `ROLE_CUSTOMER` | Lịch sử mua hàng thành viên |
| **Admin** | `/api/v1/admin/**` | `ALL` | `ROLE_ADMIN` | Toàn quyền quản trị hệ thống |

---

## 4. Đặc Tả Chi Tiết Từng Nhóm API

---

### Module 1: Xác Thực Tài Khoản (Authentication — `/api/v1/auth`)

#### 1.1 Đăng ký tài khoản khách hàng (`POST /api/v1/auth/register`)
- **Mô tả:** Đăng ký tài khoản thành viên mới với mật khẩu mã hóa BCrypt.
- **Quyền hạn:** `Public`.
- **Request Body:**
  ```json
  {
    "email": "customer@gmail.com",
    "phone": "0987654321",
    "password": "Password123@",
    "fullName": "Nguyễn Văn Khách"
  }
  ```
- **Validation Rules:**
  - `email`: `@NotBlank`, `@Email`, tối đa 150 ký tự.
  - `phone`: `@NotBlank`, Regex `^(0[3|5|7|8|9])[0-9]{8}$` (10 chữ số theo chuẩn nhà mạng Việt Nam).
  - `password`: `@NotBlank`, tối thiểu 8 ký tự, bao gồm ít nhất 1 chữ hoa, 1 chữ thường và 1 chữ số.
  - `fullName`: `@NotBlank`, tối đa 100 ký tự.
- **Response `201 Created`:**
  ```json
  {
    "success": true,
    "message": "Đăng ký tài khoản thành công",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "tokenType": "Bearer",
      "expiresIn": 86400,
      "user": {
        "id": 10,
        "email": "customer@gmail.com",
        "phone": "0987654321",
        "fullName": "Nguyễn Văn Khách",
        "role": "ROLE_CUSTOMER",
        "createdAt": "2026-11-14T21:45:00.000Z"
      }
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```
- **Lỗi thường gặp:**
  - `400 Bad Request`: `INVALID_PAYLOAD` (Dữ liệu không đúng định dạng).
  - `409 Conflict`: `DUPLICATE_RESOURCE` ("Email đã tồn tại" hoặc "Số điện thoại đã tồn tại").

---

#### 1.2 Đăng nhập hệ thống linh hoạt (`POST /api/v1/auth/login`)
- **Mô tả:** Đăng nhập linh hoạt bằng **Email HOẶC Số điện thoại** kèm mật khẩu.
- **Quyền hạn:** `Public`.
- **Request Body:**
  ```json
  {
    "identifier": "0987654321",
    "password": "Password123@"
  }
  ```
- **Validation Rules:**
  - `identifier`: `@NotBlank` (người dùng có thể nhập Email hoặc Số điện thoại).
  - `password`: `@NotBlank`.
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Đăng nhập thành công",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "tokenType": "Bearer",
      "expiresIn": 86400,
      "user": {
        "id": 1,
        "email": "admin@shopvdk.vn",
        "phone": "0901234567",
        "fullName": "Quản Trị Viên",
        "role": "ROLE_ADMIN",
        "createdAt": "2026-11-14T21:00:00.000Z"
      }
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```
- **Lỗi thường gặp:**
  - `401 Unauthorized`: `BAD_CREDENTIALS` ("Tài khoản hoặc mật khẩu không chính xác").
  - `403 Forbidden`: `ACCOUNT_LOCKED` ("Tài khoản đã bị tạm khóa, vui lòng liên hệ quản trị viên").

---

#### 1.3 Lấy thông tin tài khoản hiện tại (`GET /api/v1/auth/me`)
- **Mô tả:** Lấy profile của người dùng đang đăng nhập dựa trên JWT Bearer Token.
- **Quyền hạn:** `Authenticated` (`ROLE_CUSTOMER` hoặc `ROLE_ADMIN`).
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Lấy thông tin người dùng thành công",
    "data": {
      "id": 10,
      "email": "customer@gmail.com",
      "phone": "0987654321",
      "fullName": "Nguyễn Văn Khách",
      "role": "ROLE_CUSTOMER",
      "createdAt": "2026-11-14T21:45:00.000Z"
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

---

### Module 2: Danh Mục & Thương Hiệu (Catalog — Public)

#### 2.1 Danh sách danh mục đa cấp (`GET /api/v1/categories`)
- **Mô tả:** Lấy danh sách cây phân cấp danh mục hoặc danh sách phẳng các danh mục đang hoạt động.
- **Query Params:**
  - `tree` (boolean, optional, default: `true`): `true` trả về dạng cây cha - con (Parent -> Children); `false` trả về danh sách phẳng.
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Lấy danh mục thành công",
    "data": [
      {
        "id": 1,
        "name": "Vợt Cầu Lông",
        "slug": "vot-cau-long",
        "displayOrder": 1,
        "parentId": null,
        "children": [
          {
            "id": 7,
            "name": "Vợt Cầu Lông Yonex",
            "slug": "vot-cau-long-yonex",
            "displayOrder": 1,
            "parentId": 1,
            "children": []
          },
          {
            "id": 8,
            "name": "Vợt Cầu Lông Victor",
            "slug": "vot-cau-long-victor",
            "displayOrder": 2,
            "parentId": 1,
            "children": []
          }
        ]
      },
      {
        "id": 2,
        "name": "Giày Cầu Lông",
        "slug": "giay-cau-long",
        "displayOrder": 2,
        "parentId": null,
        "children": []
      }
    ],
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

---

#### 2.2 Danh sách thương hiệu (`GET /api/v1/brands`)
- **Mô tả:** Lấy danh sách tất cả thương hiệu đang hoạt động (`is_active = true`).
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Lấy danh sách thương hiệu thành công",
    "data": [
      {
        "id": 1,
        "name": "Yonex",
        "slug": "yonex",
        "logoUrl": "https://cdn.shopvdk.vn/brands/yonex.png"
      },
      {
        "id": 2,
        "name": "Victor",
        "slug": "victor",
        "logoUrl": "https://cdn.shopvdk.vn/brands/victor.png"
      },
      {
        "id": 3,
        "name": "Lining",
        "slug": "lining",
        "logoUrl": "https://cdn.shopvdk.vn/brands/lining.png"
      }
    ],
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

---

### Module 3: Sản Phẩm & Biến Thể SKU (Products & Variants — Public)

#### 3.1 Danh sách sản phẩm với bộ lọc chuyên sâu (`GET /api/v1/products`)
- **Mô tả:** Tìm kiếm, phân loại, sắp xếp và lọc động theo cấu hình thông số kỹ thuật thể thao (JSONB).
- **Query Params:**
  - `keyword` (string, optional): Từ khóa tìm kiếm theo tên sản phẩm.
  - `categorySlug` (string, optional): Lọc theo slug danh mục (ví dụ: `vot-cau-long`).
  - `brandSlugs` (list string, optional, định dạng phân tách dấu phẩy): Lọc theo thương hiệu (ví dụ: `yonex,victor`).
  - `minPrice` / `maxPrice` (numeric, optional): Khoảng giá mong muốn (ví dụ: `minPrice=1000000&maxPrice=5000000`).
  - **Bộ lọc thông số kỹ thuật động (Dynamic Specification Filters - Mapping vào cột `specifications JSONB`):**
    - `specs_balance_point`: (ví dụ: `head_heavy`, `even_balance`, `head_light`).
    - `specs_stiffness`: (ví dụ: `stiff`, `medium`, `flexible`).
    - `specs_play_style`: (ví dụ: `tan_cong`, `cong_thu_toan_dien`, `phan_tat_thu_ve`).
  - `sortBy` (string, optional, default: `newest`):
    - `newest`: Sản phẩm mới nhất.
    - `price_asc`: Giá tăng dần.
    - `price_desc`: Giá giảm dần.
    - `name_asc`: Tên từ A - Z.
  - `page` (int, optional, default: `0`): Chỉ số trang (bắt đầu từ 0).
  - `size` (int, optional, default: `12`): Số lượng sản phẩm mỗi trang.

- **Response `200 OK` (`PageResponse<ProductSummaryResponse>`):**
  ```json
  {
    "success": true,
    "message": "Lấy danh sách sản phẩm thành công",
    "data": {
      "items": [
        {
          "id": 101,
          "name": "Vợt Cầu Lông Yonex Astrox 88D Pro",
          "slug": "vot-cau-long-yonex-astrox-88d-pro",
          "basePrice": 4200000,
          "minPrice": 4200000,
          "maxPrice": 4350000,
          "thumbnailUrl": "https://cdn.shopvdk.vn/products/astrox-88d-pro-main.jpg",
          "brand": {
            "id": 1,
            "name": "Yonex",
            "slug": "yonex"
          },
          "category": {
            "id": 1,
            "name": "Vợt Cầu Lông",
            "slug": "vot-cau-long"
          },
          "inStock": true,
          "totalStockQuantity": 25,
          "specifications": {
            "balance_point": "head_heavy",
            "stiffness": "stiff",
            "play_style": "tan_cong"
          },
          "createdAt": "2026-11-14T20:00:00.000Z"
        }
      ],
      "page": 0,
      "size": 12,
      "totalElements": 1,
      "totalPages": 1,
      "isFirst": true,
      "isLast": true,
      "hasNext": false,
      "hasPrevious": false
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

---

#### 3.2 Chi tiết sản phẩm & Biến thể SKU (`GET /api/v1/products/{slug}`)
- **Mô tả:** Lấy đầy đủ thông số kỹ thuật, mô tả và danh sách biến thể SKU còn hàng để người dùng chọn trên UI (chọn Size, Màu sắc, Trọng lượng).
- **Path Param:** `slug` (ví dụ: `vot-cau-long-yonex-astrox-88d-pro`).
- **Response `200 OK` (`ApiResponse<ProductDetailResponse>`):**
  ```json
  {
    "success": true,
    "message": "Lấy chi tiết sản phẩm thành công",
    "data": {
      "id": 101,
      "name": "Vợt Cầu Lông Yonex Astrox 88D Pro",
      "slug": "vot-cau-long-yonex-astrox-88d-pro",
      "basePrice": 4200000,
      "description": "Vợt cầu lông đỉnh cao thiên công dành cho người chơi trình độ khá giỏi...",
      "brand": {
        "id": 1,
        "name": "Yonex",
        "slug": "yonex",
        "logoUrl": "https://cdn.shopvdk.vn/brands/yonex.png"
      },
      "category": {
        "id": 1,
        "name": "Vợt Cầu Lông",
        "slug": "vot-cau-long"
      },
      "specifications": {
        "balance_point": "head_heavy",
        "stiffness": "stiff",
        "max_tension_lbs": 28,
        "weight_specs": ["3U", "4U"],
        "grip_specs": ["G5"],
        "swing_weight": "heavy",
        "play_style": "tan_cong"
      },
      "variants": [
        {
          "id": 501,
          "skuCode": "YONEX-AX88D-PRO-4UG5",
          "price": 4200000,
          "stockQuantity": 15,
          "attributes": {
            "weight": "4U",
            "grip": "G5"
          },
          "isActive": true
        },
        {
          "id": 502,
          "skuCode": "YONEX-AX88D-PRO-3UG5",
          "price": 4350000,
          "stockQuantity": 10,
          "attributes": {
            "weight": "3U",
            "grip": "G5"
          },
          "isActive": true
        }
      ],
      "createdAt": "2026-11-14T20:00:00.000Z"
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```
- **Lỗi thường gặp:**
  - `404 Not Found`: `PRODUCT_NOT_FOUND` ("Không tìm thấy sản phẩm tương ứng với đường dẫn yêu cầu").

---

### Module 4: Đặt Hàng & Tra Cứu Đơn Hàng (Orders — Public & Customer)

#### 4.1 Đặt hàng ẩn danh — Không cần tài khoản (`POST /api/v1/orders/guest-checkout`)
- **Mô tả:** Cho phép khách vãng lai đặt hàng trực tiếp mà không cần đăng ký tài khoản. Hệ thống thực hiện kiểm tra Fail-Fast trước khi khóa dòng dữ liệu (`SELECT ... FOR UPDATE`) để trừ tồn kho.
- **Quyền hạn:** `Public`.
- **Request Body:**
  ```json
  {
    "recipientName": "Trần Thị Khách",
    "recipientPhone": "0912345678",
    "recipientEmail": "tranthikhach@gmail.com",
    "shippingAddress": "Số 123 Đường Cầu Giấy, Phường Quan Hoa, Quận Cầu Giấy, Hà Nội",
    "orderNotes": "Giao giờ hành chính, vui lòng gọi trước 15 phút",
    "paymentMethod": "COD",
    "items": [
      {
        "variantId": 501,
        "quantity": 1
      }
    ]
  }
  ```
- **Validation Rules (Jakarta Bean Validation Fail-Fast):**
  - `recipientName`: `@NotBlank`, tối đa 100 ký tự.
  - `recipientPhone`: `@NotBlank`, Regex chuẩn 10 chữ số Việt Nam `^(0[3|5|7|8|9])[0-9]{8}$`.
  - `recipientEmail`: `@Email`, tối đa 150 ký tự (Optional, có thể null hoặc chuỗi rỗng).
  - `shippingAddress`: `@NotBlank`, tối đa 500 ký tự.
  - `paymentMethod`: `@NotBlank`, bắt buộc thuộc danh sách cho phép (hiện tại: `'COD'`).
  - `items`: `@NotEmpty`, danh sách ít nhất 1 mặt hàng.
  - `items[].variantId`: `@NotNull`, `@Positive`.
  - `items[].quantity`: `@NotNull`, `@Min(1)`, `@Max(100)`.

- **Response `201 Created` (`ApiResponse<OrderCreationResponse>`):**
  ```json
  {
    "success": true,
    "message": "Đặt hàng thành công",
    "data": {
      "orderCode": "VDK-20261114-A7B2C",
      "recipientName": "Trần Thị Khách",
      "recipientPhone": "0912345678",
      "shippingAddress": "Số 123 Đường Cầu Giấy, Phường Quan Hoa, Quận Cầu Giấy, Hà Nội",
      "totalAmount": 4200000,
      "paymentMethod": "COD",
      "orderStatus": "PENDING",
      "createdAt": "2026-11-14T21:45:00.000Z"
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

- **Xử lý Ngoại lệ Tồn kho Riêng biệt (Phù hợp Business Rule BR-INV-02):**
  - Khi SKU hết hàng hoàn toàn (`stock_quantity == 0`):
    ```json
    {
      "success": false,
      "status": 409,
      "errorCode": "OUT_OF_STOCK",
      "message": "Sản phẩm 'Vợt Cầu Lông Yonex Astrox 88D Pro (4U/G5)' đã hết hàng",
      "path": "/api/v1/orders/guest-checkout",
      "timestamp": "2026-11-14T21:45:00.000Z",
      "errors": []
    }
    ```
  - Khi SKU còn tồn kho nhưng không đủ số lượng mua (`0 < stock_quantity < requested_quantity`):
    ```json
    {
      "success": false,
      "status": 409,
      "errorCode": "INSUFFICIENT_STOCK",
      "message": "Sản phẩm 'Vợt Cầu Lông Yonex Astrox 88D Pro (4U/G5)' chỉ còn lại 2 chiếc, không đủ số lượng bạn yêu cầu",
      "path": "/api/v1/orders/guest-checkout",
      "timestamp": "2026-11-14T21:45:00.000Z",
      "errors": []
    }
    ```

---

#### 4.2 Đặt hàng cho khách hàng thành viên (`POST /api/v1/orders/checkout`)
- **Mô tả:** Giống như logic đặt hàng của Guest nhưng liên kết trực tiếp `user_id` với người dùng đang xác thực bằng token.
- **Quyền hạn:** `ROLE_CUSTOMER`.
- **Request Body & Validation:** Tương tự `4.1` (thông tin người nhận có thể tự động điền từ profile của User trên FE).
- **Response `201 Created`:** Cấu trúc tương tự `4.1`.

---

#### 4.3 Tra cứu đơn hàng công khai (`GET /api/v1/orders/track`)
- **Mô tả:** Khách vãng lai và người mua hàng tra cứu tiến độ xử lý đơn hàng tại trang `/orders/track` mà không cần tài khoản hay token đăng nhập.
- **Quyền hạn:** `Public`.
- **Query Params:**
  - `orderCode` (string, required): Mã đơn hàng (ví dụ: `VDK-20261114-A7B2C`).
  - `phone` (string, required): Số điện thoại nhận hàng (dùng làm lớp bảo mật kép 2 yếu tố để xác minh quyền tra cứu).
- **Response `200 OK` (`ApiResponse<OrderDetailResponse>`):**
  ```json
  {
    "success": true,
    "message": "Tra cứu đơn hàng thành công",
    "data": {
      "orderCode": "VDK-20261114-A7B2C",
      "orderStatus": "PENDING",
      "paymentMethod": "COD",
      "recipientName": "Trần Thị Khách",
      "recipientPhone": "0912345678",
      "shippingAddress": "Số 123 Đường Cầu Giấy, Phường Quan Hoa, Quận Cầu Giấy, Hà Nội",
      "orderNotes": "Giao giờ hành chính, vui lòng gọi trước 15 phút",
      "totalAmount": 4200000,
      "createdAt": "2026-11-14T21:45:00.000Z",
      "items": [
        {
          "id": 12,
          "productName": "Vợt Cầu Lông Yonex Astrox 88D Pro",
          "skuCode": "YONEX-AX88D-PRO-4UG5",
          "variantAttributes": {
            "weight": "4U",
            "grip": "G5"
          },
          "priceAtPurchase": 4200000,
          "quantity": 1,
          "subtotal": 4200000
        }
      ]
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```
- **Lỗi thường gặp:**
  - `404 Not Found`: `ORDER_NOT_FOUND` ("Không tìm thấy đơn hàng tương ứng với mã đơn và số điện thoại đã cung cấp").

---

#### 4.4 Lịch sử đơn hàng của thành viên (`GET /api/v1/customer/orders`)
- **Mô tả:** Lấy danh sách các đơn hàng thuộc về chính tài khoản khách hàng đang đăng nhập.
- **Quyền hạn:** `ROLE_CUSTOMER`.
- **Query Params:** `page` (int, default: 0), `size` (int, default: 10).
- **Response `200 OK` (`PageResponse<OrderSummaryResponse>`):** Danh sách đơn hàng tóm tắt phân trang.

---

### Module 5: Quản Trị Hệ Thống (Admin Operations — `/api/v1/admin/*`)

*Tất cả các API trong mục này đều bắt buộc quyền `ROLE_ADMIN`.*

#### 5.1 Thống kê nhanh Bảng điều khiển (`GET /api/v1/admin/dashboard/stats`)
- **Mô tả:** Lấy các chỉ số KPI vận hành chính: doanh thu hôm nay, số đơn chờ duyệt, số sản phẩm sắp hết hàng (`stock_quantity < 5`).
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Lấy số liệu thống kê thành công",
    "data": {
      "todayRevenue": 25800000,
      "pendingOrdersCount": 8,
      "shippingOrdersCount": 14,
      "completedOrdersCount": 120,
      "lowStockVariantsCount": 3
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```

---

#### 5.2 Quản trị Đơn hàng — Danh sách (`GET /api/v1/admin/orders`)
- **Mô tả:** Tìm kiếm và lọc danh sách đơn hàng toàn hệ thống phục vụ vận hành đóng gói và giao hàng.
- **Query Params:**
  - `status` (string, optional): `PENDING`, `CONFIRMED`, `SHIPPING`, `DELIVERED`, `CANCELLED`.
  - `keyword` (string, optional): Tìm kiếm theo mã đơn (`orderCode`), tên khách, hoặc số điện thoại.
  - `fromDate`, `toDate` (string ISO, optional): Lọc theo ngày đặt hàng.
  - `page` (int, default: 0), `size` (int, default: 20).
- **Response `200 OK` (`PageResponse<AdminOrderSummaryResponse>`):** Trả về danh sách đơn hàng có đủ thông tin trạng thái và ngày tạo.

---

#### 5.3 Quản trị Đơn hàng — Chuyển trạng thái Vận hành (`PATCH /api/v1/admin/orders/{id}/status`)
- **Mô tả:** Thực hiện chuyển đổi trạng thái của đơn hàng theo đúng State Machine quy định tại `BR-ORD-01`.
  - Nếu chuyển sang `CANCELLED`: Hệ thống tự động kích hoạt tiến trình hoàn trả tồn kho nguyên tử (`Atomic Restocking`) cho các SKU trong đơn.
- **Path Param:** `id` (ID số của đơn hàng).
- **Request Body:**
  ```json
  {
    "newStatus": "CONFIRMED",
    "note": "Đã gọi điện xác nhận địa chỉ nhận hàng với khách"
  }
  ```
- **Validation Rules:**
  - `newStatus`: `@NotBlank`, giá trị hợp lệ: `CONFIRMED`, `SHIPPING`, `DELIVERED`, `CANCELLED`.
  - `note`: Tùy chọn, tối đa 255 ký tự ghi chú nội bộ của Admin.
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "message": "Cập nhật trạng thái đơn hàng thành công",
    "data": {
      "orderId": 15,
      "orderCode": "VDK-20261114-A7B2C",
      "previousStatus": "PENDING",
      "currentStatus": "CONFIRMED",
      "updatedAt": "2026-11-14T21:45:00.000Z"
    },
    "timestamp": "2026-11-14T21:45:00.000Z"
  }
  ```
- **Lỗi thường gặp:**
  - `400 Bad Request`: `INVALID_STATE_TRANSITION` ("Không thể chuyển trạng thái từ DELIVERED sang CANCELLED").

---

#### 5.4 Quản trị Sản phẩm & Biến thể SKU

##### A. Tạo mới sản phẩm (`POST /api/v1/admin/products`)
- **Request Body:**
  ```json
  {
    "name": "Giày Cầu Lông Yonex 65Z3 Trắng",
    "slug": "giay-cau-long-yonex-65z3-trang",
    "brandId": 1,
    "categoryId": 2,
    "basePrice": 2750000,
    "description": "Dòng giày thi đấu chuyên nghiệp bảo vệ khớp gối...",
    "specifications": {
      "shoe_type": "all_around",
      "upper_material": "Synthetic Leather",
      "midsole": "Power Cushion+"
    },
    "variants": [
      {
        "skuCode": "YONEX-65Z3-W-40",
        "price": 2750000,
        "stockQuantity": 8,
        "attributes": {
          "size": 40,
          "color": "Trắng"
        }
      },
      {
        "skuCode": "YONEX-65Z3-W-41",
        "price": 2750000,
        "stockQuantity": 12,
        "attributes": {
          "size": 41,
          "color": "Trắng"
        }
      }
    ]
  }
  ```
- **Response `201 Created`:** Trả về đối tượng sản phẩm và biến thể SKU vừa tạo.

##### B. Cập nhật tồn kho hoặc giá biến thể SKU (`PUT /api/v1/admin/variants/{variantId}`)
- **Request Body:**
  ```json
  {
    "price": 2800000,
    "stockQuantity": 20,
    "isActive": true
  }
  ```
- **Response `200 OK`:** Cập nhật thông tin SKU thành công.

---

## 5. Từ Điển Mã Lỗi Nghiệp Vụ (Error Code Dictionary)

Bảng tổng hợp toàn bộ các mã lỗi chuẩn (`errorCode`) mà Backend trả về để Frontend bóc tách và hiển thị UI thông báo tương ứng:

| Mã Lỗi (`errorCode`) | Mã HTTP | Ý Nghĩa Nghiệp Vụ | Hành Động Phía Frontend (UI Action) |
| :--- | :--- | :--- | :--- |
| `INVALID_PAYLOAD` | `400` | Dữ liệu đầu vào sai định dạng hoặc vi phạm Bean Validation | Hiển thị thông báo đỏ ngay dưới từng trường Input tương ứng |
| `INVALID_STATE_TRANSITION` | `400` | Cố gắng chuyển trạng thái đơn hàng vi phạm State Machine | Hiển thị Toast cảnh báo hành động không hợp lệ |
| `UNAUTHORIZED` | `401` | Chưa truyền Bearer Token hoặc Token đã hết hạn | Điều hướng sang trang Login hoặc kích hoạt Popup đăng nhập |
| `BAD_CREDENTIALS` | `401` | Sai tài khoản (Email/SĐT) hoặc mật khẩu | Hiển thị thông báo "Tài khoản hoặc mật khẩu không chính xác" |
| `ACCOUNT_LOCKED` | `403` | Tài khoản đang ở trạng thái `is_active = false` | Hiển thị thông báo tài khoản bị khóa |
| `FORBIDDEN` | `403` | Không đủ quyền hạn truy cập (ví dụ Customer truy cập Admin) | Điều hướng đến trang 403 Forbidden |
| `RESOURCE_NOT_FOUND` | `404` | Không tìm thấy tài nguyên theo ID hoặc Slug | Hiển thị màn hình 404 Empty/Not Found |
| `OUT_OF_STOCK` | `409` | Sản phẩm đã hết hàng (`stock_quantity == 0`) | Hiển thị thông báo hết hàng, vô hiệu hóa nút thanh toán của SKU đó |
| `INSUFFICIENT_STOCK` | `409` | Tồn kho không đủ đáp ứng số lượng đặt | Hiển thị thông báo số lượng còn lại chính xác, tự động gán max quantity |
| `DUPLICATE_RESOURCE` | `409` | Trùng lặp Unique Constraint (Email, SĐT, Slug, SKU) | Báo đỏ trường bị trùng lặp |
| `INTERNAL_SERVER_ERROR` | `500` | Lỗi ngoại lệ chưa được bắt tại tầng Server | Hiển thị Toast lỗi hệ thống chung ("Đã có sự cố xảy ra, vui lòng thử lại") |

---

## 6. Hướng Dẫn Tích Hợp Frontend (Frontend Integration Guidelines)

1. **Bộ định tuyến API tập trung:** Tất cả các cuộc gọi HTTP từ React đều thông qua một instance `Axios` duy nhất tại `@/services/apiClient.ts` được cấu hình sẵn Base URL và Interceptor tự động gán `Authorization: Bearer <token>`.
2. **Quản lý Giỏ hàng Guest Cart:**
   - Giỏ hàng của khách vãng lai và thành viên đều được lưu trữ trước mắt tại `LocalStorage` với cấu trúc `CartItem { variantId, productSlug, productName, skuCode, price, quantity, attributes, thumbnailUrl }`.
   - Khi ấn nút "Đặt hàng", toàn bộ mảng này được gửi lên endpoint `/api/v1/orders/guest-checkout` hoặc `/api/v1/orders/checkout`.
3. **Fail-Fast Validation 2 lớp:**
   - Lớp 1 (FE): Sử dụng thư viện `react-hook-form` + `zod` để chặn lỗi ngay từ Client trước khi gửi Request.
   - Lớp 2 (BE): Sử dụng Spring Boot Jakarta Bean Validation (`@Valid`) để bảo vệ dữ liệu ở mức tối cao.
