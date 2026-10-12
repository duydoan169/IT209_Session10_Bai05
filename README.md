# BÀI 5: TỰ ĐỘNG HÓA ĐIỀU PHỐI ĐA DỊCH VỤ VỚI DOCKER COMPOSE & FILE .ENV

---

## 1. Mục tiêu & Bối cảnh

Trong bài tập trước (BT4), việc quản lý cụm dịch vụ phân tán bằng các lệnh Docker CLI thủ công (`docker network create`, `docker volume create`, `docker run...`) đòi hỏi nhiều thao tác, dễ sai sót cấu hình và khó duy trì trong môi trường nhóm hoặc quy trình CI/CD.

Bài tập này nâng cấp toàn bộ hạ tầng sang chuẩn **Infrastructure as Code (IaC)** bằng **Docker Compose CLI v2 (`docker compose`)**:
- Khai báo toàn bộ dịch vụ (`backend`, `database`), mạng (`networks`) và ổ đĩa (`volumes`) trong một tệp duy nhất: `docker-compose.yml`.
- Tách biệt hoàn toàn thông tin cấu hình và dữ liệu nhạy cảm (Credentials, Ports, Database Names) sang tệp `.env`.
- Cung cấp tệp mẫu `.env.example` và cấu hình `.gitignore` nhằm tuân thủ nguyên tắc **Bảo mật mặc định (Security by Default)**, ngăn chặn việc commit bí mật lên kho mã nguồn Git.

---

## 2. Cấu trúc thư mục

```
BT5/
├── docker-compose.yml  # Bản thiết kế hạ tầng đa container theo chuẩn IaC
├── .env                # Biến môi trường thực tế (chứa credentials, port - không commit Git)
├── .env.example        # Tệp mẫu hướng dẫn cấu hình biến môi trường
├── .gitignore          # Cấu hình bỏ qua tệp .env và node_modules
├── Dockerfile          # Multi-stage Dockerfile đóng gói ứng dụng Backend
├── .dockerignore       # Bỏ qua node_modules, log, file thừa khi build
├── package.json        # Định nghĩa dependencies (Express)
├── server.js           # Mã nguồn Web Backend API
└── README.md           # Tài liệu hướng dẫn, giải thích chỉ thị và nhật ký kiểm thử
```

---

## 3. Quản lý biến môi trường bảo mật

### 3.1. Tệp cấu hình thực tế (`.env`)
```env
# Application Port Settings
HOST_PORT=8083
PORT=3000

# PostgreSQL Database Configuration
DB_NAME=appdb
DB_USER=postgres
DB_PASSWORD=SuperSecretPass123!
```

### 3.2. Tệp mẫu cấu hình (`.env.example`)
Dùng để hướng dẫn các thành viên trong nhóm thiết lập môi trường mà không làm lộ mật khẩu thực tế:
```env
# Application Port Settings
HOST_PORT=8083
PORT=3000

# PostgreSQL Database Configuration (Template - Không lưu thông tin thực tế)
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_secure_password
```

### 3.3. Tệp `.gitignore`
Đảm bảo tệp `.env` không bao giờ bị đưa lên Git repository:
```gitignore
# Environment variables containing sensitive credentials
.env

# Dependencies
node_modules/

# Logs
*.log
npm-debug.log*
```

---

## 4. Phân tích chi tiết `docker-compose.yml`

```yaml
services:
  database:
    image: postgres:15-alpine
    container_name: compose-database
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend-net

  backend:
    build: .
    container_name: compose-backend
    restart: unless-stopped
    ports:
      - "${HOST_PORT}:${PORT}"
    environment:
      PORT: ${PORT}
      DB_HOST: database
      DB_PORT: 5432
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      - database
    networks:
      - backend-net

volumes:
  pgdata:
    driver: local

networks:
  backend-net:
    driver: bridge
```

### Giải thích các chỉ thị quan trọng:
1. **`services`:** Khai báo các dịch vụ thành phần trong hệ thống:
   - **`database`:** Sử dụng image `postgres:15-alpine`. Nhận các thông số nhạy cảm (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`) nội suy tự động từ file `.env` qua cú pháp `${VAR_NAME}`.
   - **`backend`:** Tự động build image từ `Dockerfile` tại thư mục hiện tại (`build: .`), ánh xạ port từ host vào container (`${HOST_PORT}:${PORT}`).
2. **`depends_on`:**
   - Thiết lập mối quan hệ phụ thuộc: Dịch vụ `backend` chỉ được khởi tạo và chạy sau khi dịch vụ `database` đã được tạo và kích hoạt.
3. **`networks` (cấp root & service):**
   - Khởi tạo custom bridge network `backend-net`. Cả 2 service đều tham gia vào network này, cho phép backend kết nối tới database thông qua tên service `database` nhờ Embedded DNS của Docker.
4. **`volumes` (cấp root & service):**
   - Khai báo Named Volume `pgdata` ở cấp độ root và mount vào `/var/lib/postgresql/data` của PostgreSQL, đảm bảo tính bền vững của dữ liệu.

---

## 5. Quy trình thực thi & Kiểm thử bằng Docker Compose CLI v2

### 5.1. Khởi chạy toàn bộ hệ thống (`docker compose up -d`)
```bash
docker compose up -d --build
```
**Nhật ký thực thi thực tế:**
```text
 Network bt5_backend-net Created 
 Volume bt5_pgdata Created 
 Container compose-database Creating 
 Container compose-database Created 
 Container compose-backend Creating 
 Container compose-backend Created 
 Container compose-database Starting 
 Container compose-database Started 
 Container compose-backend Starting 
 Container compose-backend Started 
```
> **Đánh giá:** Chỉ với một lệnh duy nhất, Docker Compose v2 đã tự động: build image, tạo network `bt5_backend-net`, tạo volume `bt5_pgdata`, khởi động database trước và sau đó khởi động backend theo đúng thứ tự phụ thuộc.

---

### 5.2. Kiểm tra trạng thái các dịch vụ (`docker compose ps`)
```bash
docker compose ps
```
**Kết quả thực tế:**
```text
NAME               IMAGE                COMMAND                  SERVICE    CREATED         STATUS         PORTS
compose-backend    bt5-backend          "docker-entrypoint.s…"   backend    8 seconds ago   Up 6 seconds   0.0.0.0:8083->3000/tcp, [::]:8083->3000/tcp
compose-database   postgres:15-alpine   "docker-entrypoint.s…"   database   8 seconds ago   Up 7 seconds   5432/tcp
```
Cả hai container đều ở trạng thái **Up** và vận hành trơn tru.

---

### 5.3. Kiểm tra Logs hoạt động (`docker compose logs`)
```bash
docker compose logs
```
**Trích xuất Log thực tế:**
- **Backend:**
  ```text
  compose-backend   | Backend server listening on port 3000
  ```
- **PostgreSQL Database:**
  ```text
  compose-database  | PostgreSQL init process complete; ready for start up.
  compose-database  | LOG: starting PostgreSQL 15.19 on x86_64-pc-linux-musl...
  compose-database  | LOG: listening on IPv4 address "0.0.0.0", port 5432
  compose-database  | LOG: database system is ready to accept connections
  ```

---

### 5.4. Kiểm tra Endpoint API qua `curl`
```bash
curl -i http://localhost:8083/
```
*(Trên Windows PowerShell: `curl.exe -i -4 http://localhost:8083/`)*

**Kết quả HTTP Response thực tế:**
```http
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 255
ETag: W/"ff-r5P35pWv2yP3k4zsA7uKeFbZ4xg"
Date: Mon, 12 Oct 2026 01:31:24 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{
  "status": "success",
  "service": "backend-api",
  "message": "Hello from Multi-service Docker Compose Architecture!",
  "database_config": {
    "host": "database",
    "port": "5432",
    "name": "appdb",
    "user": "postgres"
  },
  "uptime": 5.893839046,
  "timestamp": "2026-10-12T01:31:24.868Z"
}
```
API phản hồi chuẩn `HTTP 200 OK`, đọc chính xác các biến môi trường cấu hình kết nối Database từ `.env`.

---

### 5.5. Dừng và dọn dẹp hệ thống (`docker compose down`)
Khi cần dừng môi trường:
```bash
docker compose down
```
Lệnh này sẽ dừng an toàn và giải phóng container cùng network đã tạo. Khi muốn xóa cả volume lưu trữ:
```bash
docker compose down -v
```

---

## 6. Kết luận & Lợi ích thực tiễn
1. **Infrastructure as Code (IaC):** Toàn bộ kiến trúc hạ tầng được phiên bản hóa (version-controlled) trong Git cùng mã nguồn ứng dụng, giúp việc setup môi trường mới cho developer mới chỉ tốn đúng 1 câu lệnh (`docker compose up -d`).
2. **Bảo mật tuyệt đối:** Các thông tin nhạy cảm được quản lý tập trung trong `.env`, không bị hardcode trong source code hay Dockerfile.
3. **Thao tác chuẩn Docker CLI v2:** Tận dụng tối đa cú pháp hiện đại `docker compose` tích hợp sẵn trong Docker Engine.
