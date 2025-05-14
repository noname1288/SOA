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
