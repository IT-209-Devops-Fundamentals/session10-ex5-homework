# Bài tập 5: Tự động hóa điều phối đa dịch vụ với Docker Compose & File .env

- **Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals
- **Học viên:** Trần Đức Anh
- **Mã sinh viên / User:** `stephen-devops`
- **Đường dẫn nộp bài:** `homework/session_10/ex5/`

---

## 1. Mục tiêu

- Sử dụng Docker Compose CLI v2 để điều phối tự động cụm 2 dịch vụ: Web Backend (`backend`) và PostgreSQL Database (`database`).
- Tách biệt các biến môi trường nhạy cảm (tên DB, user, password, port) ra tệp `.env`, đồng thời tạo tệp mẫu `.env.example` và dùng `.gitignore` để tránh lộ thông tin bảo mật lên GitHub.
- Khai báo đầy đủ các chỉ thị cấp root: `services`, `volumes`, `networks` và quan hệ phụ thuộc `depends_on`.

---

## 2. Cấu hình các tệp tin

### 2.1. Tệp `docker-compose.yml`
```yaml
services:
  database:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend-net

  backend:
    image: hw-backend:v1
    ports:
      - "${PORT}:3000"
    environment:
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

networks:
  backend-net:
```

### 2.2. Tệp biến môi trường mẫu `.env.example`
```env
PORT=8080
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password
```

### 2.3. Tệp `.gitignore`
Chặn không cho đẩy tệp `.env` chứa mật khẩu thật lên Git:
```gitignore
.env
```

---

## 3. Các bước thực hiện và kết quả kiểm tra

### Bước 1: Khởi chạy cụm dịch vụ với `docker compose up -d`
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ docker compose up -d
[+] Running 4/4
 ✔ Network ex5_backend-net      Created                                                    0.1s 
 ✔ Volume "ex5_pgdata"          Created                                                    0.1s 
 ✔ Container ex5-database-1     Started                                                    0.4s 
 ✔ Container ex5-backend-1      Started                                                    0.5s 
```
> Nhận xét: Docker Compose tự động tạo mạng `ex5_backend-net`, volume `ex5_pgdata` và khởi động container `database` trước rồi mới khởi động `backend` theo đúng chỉ thị `depends_on`.

### Bước 2: Kiểm tra trạng thái các dịch vụ bằng `docker compose ps`
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ docker compose ps
NAME                IMAGE                COMMAND                  SERVICE    CREATED         STATUS         PORTS
ex5-backend-1       hw-backend:v1        "java -jar app.jar"      backend    8 seconds ago   Up 7 seconds   0.0.0.0:8080->3000/tcp
ex5-database-1      postgres:15-alpine   "docker-entrypoint.s…"   database   8 seconds ago   Up 7 seconds   5432/tcp
```
> Nhận xét: Cả 2 service đều đang ở trạng thái chạy ổn định (Up), service `backend` ánh xạ đúng port `8080` ra ngoài máy host từ biến môi trường `${PORT}`.

### Bước 3: Xem nhật ký hoạt động bằng `docker compose logs`
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ docker compose logs
ex5-database-1  | PostgreSQL init process complete; ready for start up.
ex5-database-1  | 2026-10-13 11:18:30.120 UTC [1] LOG:  database system is ready to accept connections
ex5-backend-1   | Product REST API Server started on port 3000
```

### Bước 4: Kiểm tra phản hồi ứng dụng qua lệnh `curl`
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ curl -i http://localhost:8080/
HTTP/1.1 200 OK
Date: Tue, 13 Oct 2026 11:18:35 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 524

{
  "status": "success",
  "message": "Hello from Product REST API (Spring Boot / Java Backend)!",
  "total_products": 4,
  "data": [
    {"id": 1, "name": "Laptop Dell XPS 15", "price": 1850.0, "category": "Electronics", "stock": 25},
    {"id": 2, "name": "Ban phim co Keychron K2", "price": 85.0, "category": "Accessories", "stock": 60},
    {"id": 3, "name": "Chuot khong day Logitech MX Master 3S", "price": 99.0, "category": "Accessories", "stock": 40},
    {"id": 4, "name": "Man hinh LG UltraFine 4K 27 inch", "price": 450.0, "category": "Displays", "stock": 15}
  ]
}
```

### Bước 5: Hạ dịch vụ và giải phóng tài nguyên bằng `docker compose down`
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ docker compose down
[+] Running 3/3
 ✔ Container ex5-backend-1   Removed                                                       0.3s 
 ✔ Container ex5-database-1  Removed                                                       0.2s 
 ✔ Network ex5_backend-net   Removed                                                       0.1s 
```

Xác nhận dịch vụ đã được dọn dẹp sạch sẽ:
```bash
stephen-devops@0310263186:~/homework/session_10/ex5$ docker compose ps
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
```

---

## 4. Kết luận

- Docker Compose giúp tự động hóa khởi tạo cụm dịch vụ đa container một cách nhanh chóng, đồng bộ và nhất quán.
- Tách biến môi trường ra tệp `.env` giúp quản lý cấu hình tập trung và nâng cao bảo mật mã nguồn khi làm việc nhóm.
- Quy trình kiểm tra với `up -d`, `ps`, `logs` và `down` hoạt động chính xác 100% theo yêu cầu đề bài.
