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

**Bảng: User**

| Column     | Data Type    | Description               |
|------------|--------------|---------------------------|
| user_id    | integer(10)  | ID người dùng (Primary Key) |
| fullname   | varchar(255) | Họ và tên                 |
| email      | varchar(255) | Email                     |
| username   | varchar(255) | Tên người dùng            |
| password   | varchar(255) | Mật khẩu                  |

### 4.2 team‑service

**Bảng: Team**

| Column | Data Type    | Description               |
|--------|--------------|---------------------------|
| id     | INT          | ID nhóm (Primary Key)     |
| name   | VARCHAR(255) | Tên nhóm                  |

**Bảng: Team_Membership**

| Column        | Data Type      | Description                                               |
|---------------|----------------|-----------------------------------------------------------|
| membership_id | INT            | ID của Membership                                         |
| team_id       | VARCHAR(255)   | ID nhóm (Foreign Key tới bảng teams)                     |
| user_id       | VARCHAR(255)   | ID người dùng (References User Service)                   |
| role          | VARCHAR(50)    | Vai trò của thành viên (ví dụ: "Admin", "Member")        |

### 4.3 task‑service

**Bảng: Task**

| Column      | Data Type      | Description                                           |
|-------------|----------------|-------------------------------------------------------|
| id          | INT            | ID công việc (Primary Key)                            |
| title       | VARCHAR(255)   | Tiêu đề công việc                                     |
| description | TEXT           | Mô tả chi tiết công việc                              |
| due_date    | DATETIME       | Thời gian hết hạn công việc                           |
| created_by  | VARCHAR(255)   | ID của người tạo công việc (References User Service)  |
| created_at  | DATETIME       | Thời gian tạo công việc                               |
| team_id     | VARCHAR(255)   | ID team                                               |

**Bảng: Task_Assignee**

| Column            | Data Type      | Description                                                |
|-------------------|----------------|------------------------------------------------------------|
| task_assignee_id  | INT            | ID của task_assignees                                      |
| task_id           | VARCHAR(255)   | ID công việc (Foreign Key tới bảng tasks)                  |
| user_id           | VARCHAR(255)   | ID người thực hiện công việc (References User Service)     |

---

## ✅ Tổng kết

Hệ thống Tạo và Giao Việc dựa trên kiến trúc **Service-Oriented Architecture (SOA)** sẽ sử dụng các microservices chuyên biệt như:

- `Task Service`
- `User Service`
- `Team Service`
- `Notification Service`

...để xử lý các yêu cầu từ người dùng một cách hiệu quả và có thể mở rộng.

Các dịch vụ sẽ giao tiếp với nhau qua **REST API và HTTP**.  
**Bảo mật** được đảm bảo qua các cơ chế xác thực và phân quyền người dùng.  
Hệ thống sẽ được **triển khai bằng Docker** để quản lý dễ dàng hơn.

---
