# ShopVDK — Sports E-commerce Platform

Nền tảng thương mại điện tử chuyên kinh doanh dụng cụ và trang phục thể thao (phiên bản MVP tập trung chuyên sâu vào **Cầu Lông**, thiết kế kiến trúc mở sẵn sàng mở rộng sang Tennis, Pickleball), được xây dựng theo mô hình **Modular Monolith** nhằm rèn luyện tư duy và kỹ năng Software Engineering thực chiến.

---

## 🛠 Công nghệ Sử dụng (Tech Stack)

- **Frontend**: ReactJS, TypeScript, Tailwind CSS, React Router, TanStack Query, Axios
- **Backend**: Java 21 LTS, Spring Boot 3.x (`jakarta.*` namespace), Maven Wrapper, Spring Data JPA, Spring Security, JWT
- **Database**: PostgreSQL 17 (Dockerized)
- **DevOps**: Docker, Docker Compose, Git, GitHub Actions

---

## 📁 Cấu trúc Thư mục Dự kiến (Monorepo Layout)

```text
shopvdk/
├── backend/               # Mã nguồn Backend (Spring Boot 3.x, Java 21)
├── frontend/              # Mã nguồn Frontend (React, TypeScript, Tailwind CSS)
├── docs/                  # Tài liệu phân tích và kiến trúc hệ thống
│   ├── requirements/      # Yêu cầu nghiệp vụ (Phase 01)
│   ├── analysis/          # Phân tích ca sử dụng & luồng dữ liệu (Phase 02)
│   ├── architecture/      # Thiết kế Database, API & kiến trúc (Phase 03)
│   └── development/       # Quy chuẩn code & tiến độ phát triển
├── .env.example           # File mẫu biến môi trường (copy thành .env)
├── docker-compose.yml     # Quản lý dịch vụ hạ tầng (PostgreSQL 17)
├── GEMINI.md              # Quy tắc làm việc và phân chia vai trò cho AI
├── PROMPT_SETUP.md        # Hướng dẫn chi tiết thiết lập dự án
└── .gitignore             # Quy tắc loại trừ file cho Monorepo
```

---

## 🚀 Khởi động Môi trường Phát triển (Local Dev Setup)

### 1. Cấu hình biến môi trường
Tạo file `.env` từ file mẫu:
```bash
cp .env.example .env
```
*(File `.env` đã được cấu hình trong `.gitignore`, tuyệt đối không commit thông tin nhạy cảm vào Git)*

### 2. Khởi động Cơ sở dữ liệu (PostgreSQL)

Yêu cầu máy đã cài đặt và khởi động Docker Desktop:

```bash
docker-compose up -d
```

Kiểm tra container đang chạy:

```bash
docker ps --filter "name=shopvdk-postgres"
```

Thông tin kết nối cục bộ được quản lý thông qua file `.env`. Mặc định cổng kết nối là `5432` và database là `shopvdk`.

---

## 📖 Quy trình Phát triển (SDLC)

Dự án tuân thủ nghiêm ngặt 8 giai đoạn của quy trình kỹ thuật phần mềm:
1. **Phase 01**: Discovery & Requirement Analysis
2. **Phase 02**: System Analysis & Modeling
3. **Phase 03**: System Design & Architecture
4. **Phase 04**: Backend Development (Project Owner sở hữu & tự tay lập trình CRUD)
5. **Phase 05**: Frontend Development (AI triển khai theo chuẩn API Contract)
6. **Phase 06**: Testing & Quality Assurance
7. **Phase 07**: Deployment & DevOps
8. **Phase 08**: Optimization & Engineering Growth

