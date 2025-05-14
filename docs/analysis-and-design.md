Dưới đây là **phiên bản hoàn thiện** của `analysis-and-design.md` – đã gộp nội dung cũ với bản kiến trúc chuẩn bạn mong muốn, thống nhất thuật ngữ **Leader** (trưởng nhóm) và “Tạo & Giao việc”.
Bạn chỉ cần chép nguyên khối Markdown này vào dự án (ví dụ `docs/analysis-and-design.md`).

---

````markdown
# 📑 Hệ thống “Tạo & Giao Việc” – Phân tích & Thiết kế

> Phiên bản 1.0 · Cập nhật 14/05/2025

---

## 1 🎯 Problem Statement

Trưởng nhóm (Leader) cần:

1. **Tạo công việc (Task)** với tiêu đề, mô tả, deadline …
2. **Giao một hoặc nhiều thành viên** trong team thực hiện.
3. Hệ thống phải lưu trữ task và **gửi thông báo tức thời** (email / in‑app) cho assignee.

> Thành viên **không phải Leader** của team **không được** tạo/giao việc.

---

## 2 🧩 Danh sách Microservice

| Service                     | Cổng | Trách nhiệm chính                                                  | Công nghệ                         |
|-----------------------------|------|--------------------------------------------------------------------|----------------------------------|
| **gateway**                 | 8888 | Điểm vào duy nhất; xác thực JWT; định tuyến, log, rate‑limit       | Spring Cloud Gateway 4           |
| **user‑service**           | 8081 | Đăng ký, đăng nhập, refresh token; trả danh sách thành viên        | Spring Boot 3 · Spring Security  |
| **team‑service**           | 8082 | CRUD team; kiểm tra \`isLeader(teamId,userId)\`                     | Spring Boot 3 · Spring Data JPA  |
| **task‑service**           | 8083 | CRUD task; lưu assignee; gọi notification                           | Spring Boot 3 · Spring Data JPA  |
| **notification‑service**   | 8084 | Nhận yêu cầu gửi mail / in‑app; lưu lịch sử                         | Spring Boot 3 · Jakarta Mail     |

Mỗi service có **database riêng** (MySQL) theo mô hình *Database per Service*.

---

## 3 🔄 Giao tiếp Service

* Đồng bộ **REST + JSON** trong nội mạng Docker `soa-net`.  
* JWT do *user‑service* sinh, *gateway* kiểm tra.  
* Sau giai đoạn 2 có thể thêm Kafka topic `task.created` cho email bất đồng bộ.

---

## 4 🗄️ Thiết kế dữ liệu

### 4.1 user‑service

| Trường | Kiểu | Ghi chú |
|--------|------|--------|
| user_id (PK) | BIGINT AUTO | |
| email | VARCHAR UNIQUE | |
| display_name | VARCHAR | |
| password | VARCHAR (Bcrypt) | |
| created_at | DATETIME | |

### 4.2 team‑service

| Trường | Kiểu |
|--------|------|
| team_id (PK) | BIGINT AUTO |
| name | VARCHAR |
| created_by | BIGINT (FK users) |

**team_membership**

| Trường | Kiểu | |
|--------|------|--|
| id (PK) | BIGINT | |
| team_id | BIGINT | |
| user_id | BIGINT | |
| role | ENUM('LEADER','MEMBER') | |

### 4.3 task‑service

| Trường | Kiểu |
|--------|------|
| task_id (PK) | BIGINT |
| team_id | BIGINT |
| title | VARCHAR(255) |
| description | TEXT |
| due_date | DATETIME |
| creator_id | BIGINT |
| created_at | DATETIME |

**task_assignment**

| Trường | Kiểu | |
|--------|------|--|
| id (PK) | BIGINT |
| task_id | BIGINT |
| user_id | BIGINT |
| status | ENUM('PENDING','DONE') |

### 4.4 notification‑service

| Trường | Kiểu | |
|--------|------|--|
| notify_id (PK) | BIGINT |
| user_id | BIGINT |
| title | VARCHAR |
| content | TEXT |
| channel | ENUM('EMAIL','IN_APP') |
| sent_at | DATETIME |
| status | ENUM('SUCCESS','FAIL','RETRY') |

---

## 5 🖼️ ER Diagram (toàn hệ thống)

```mermaid
erDiagram
    USERS ||--o{ TEAM_MEMBERSHIP : thuộc
    TEAMS ||--o{ TEAM_MEMBERSHIP : chứa
    TEAMS ||--o{ TASKS : sở_hữu
    TASKS ||--o{ TASK_ASSIGNMENT : phân_công
    USERS ||--o{ TASK_ASSIGNMENT : thực_hiện
    USERS ||--o{ NOTIFICATIONS : nhận
````

---

## 6 🚦 Luồng dữ liệu chính

1. **Client → Gateway** `POST /api/tasks` kèm JWT.
2. **Gateway** xác thực token qua *user‑service*.
3. **Gateway → task‑service** chuyển tiếp yêu cầu.
4. **task‑service → team‑service**: hỏi `isLeader`.
      *Không phải leader → 403.*
5. **task‑service** lưu `tasks` + `task_assignment`.
6. **task‑service → notification‑service** gửi payload notify.
7. **notification‑service** gửi email, log.
8. Phản hồi **201 Created** trả về client.

---

## 7 🔐 Bảo mật

* JWT HS256/RS256; public‑key cache 15 phút tại gateway.
* Rate‑limit 100 req / 10 phút / IP (Bucket4j).

---

## 8 🐳 Docker & Triển khai

### 8.1 Dockerfile multi‑stage (dùng cho mọi service)

```dockerfile
# ----- build -----
FROM gradle:8.13-jdk21 AS builder
WORKDIR /home/app
COPY . .
RUN --mount=type=cache,target=/home/gradle/.gradle \
    gradle clean bootJar

# ----- runtime -----
FROM eclipse-temurin:21-jre
WORKDIR /opt/app
COPY --from=builder /home/app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/opt/app/app.jar"]
```

### 8.2 docker‑compose (dev)

```yaml
version: "3.9"
services:
  gateway:
    build: ./services/gateway
    ports: ["8888:8888"]
    depends_on: [user-service, team-service, task-service]
    networks: [soa-net]

  user-service:
    build: ./services/user-service
    networks: [soa-net]

  team-service:
    build: ./services/team-service
    networks: [soa-net]

  task-service:
    build: ./services/task-service
    networks: [soa-net]

  notification-service:
    build: ./services/notification-service
    networks: [soa-net]

networks:
  soa-net:
```

> Biến môi trường (DB URL, JWT secret, SMTP…) đặt trong `.env`.

---

## 9 🚀 Roadmap mở rộng

| Phase | Tính năng                     | Kết quả mong đợi        |
| ----- | ----------------------------- | ----------------------- |
| 1     | CRUD Task + Email notify      | MVP hoàn chỉnh          |
| 2     | In‑app notify (WebSocket/SSE) | Real‑time UX            |
| 3     | Thêm Kafka + Audit Service    | Event sourcing, tracing |
| 4     | Báo cáo tiến độ / Burn‑down   | BI & Analytics          |

---

## ✅ Tóm tắt

Kiến trúc microservices tách biệt:

* **gateway** bảo mật & định tuyến
* **user / team / task / notification** mỗi service một trách nhiệm
* Giao tiếp REST + JWT, chạy trong Docker
* Dễ mở rộng, dễ triển khai CI/CD

> Cần bổ sung Swagger OpenAPI, Testcontainers hay dashboard Prometheus? — Hãy cho mình biết!

```
```
