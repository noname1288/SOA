### 📑 Hệ thống “Tạo & Giao việc” **

## 1. Tổng quan

Hệ thống **Tạo & Giao việc** hỗ trợ trưởng nhóm của một team khởi tạo task, gán một hoặc nhiều người thực hiện, đồng thời tự động gửi thông báo (email / in‑app) tới người liên quan.
Thiết kế theo kiến trúc **microservice** độc lập, mỗi service chịu một **Single Responsibility**, giao tiếp qua REST (synchronous) và nội mạng Docker.

### 1.1 Mục tiêu chính

| # | Mục tiêu                  | Lý do                                                        |
| - | ------------------------- | ------------------------------------------------------------ |
| 1 | **Phân quyền rõ ràng**    | Chỉ trưởng nhóm trong team mới có quyền tạo/giao việc.             |
| 2 | **Quản lý vòng đời task** | Lưu trữ đầy đủ metadata (tên, mô tả, deadline, assignee). |
| 3 | **Hỗ trợ đa‑assignee**    | Một task có thể chỉ định nhiều thành viên.                   |
| 4 | **Thông báo tức thời**    | Đảm bảo assignee nhận notification ngay sau khi tạo.         |

---

## 2. Các thành phần hệ thống

| Service                                           | Cổng (mặc định) | Trách nhiệm chính                                            | DB riêng | Ghi chú                  |
| ------------------------------------------------- | --------------- | ------------------------------------------------------------ | -------- | ------------------------ |
| **API Gateway** (`gateway`)                       | **8888**        | Điểm vào duy nhất; kiểm tra JWT; định tuyến; rate‑limit, log | Không    | Spring Cloud Gateway 4.x |
| **User Service** (`user-service`)                 | 8081            | Xác thực, đăng ký, refresh token; trả info user/team‑members | ✅        | Spring Boot 3 + MySQL    |
| **Team Service** (`team-service`)                 | 8082            | CRUD team; quản lý vai trò; API `isLeader(teamId, userId)`    | ✅        | MySQL                    |
| **Task Service** (`task-service`)                 | 8083            | CRUD task; xác thực trưởng nhóm; lưu assignee; gọi Notification    | ✅        | MySQL                    |
| **Notification Service** (`notification-service`) | 8084            | Nhận yêu cầu gửi thông báo; gửi email / in‑app; log          | ✅        | MySQL                    |

---

## 3. Kiến trúc triển khai

```
Client (Web/SPA) ──► API‑Gateway (8888)
                        │
                        ├─► User‑Service (8081)  ──────┐  (auth, info)
                        │                              ▼
                        ├─► Team‑Service (8082)  ◄─ Task‑Service (8083)
                        │                                   │
                        └─► Notification‑Service (8084) ◄───┘
```

---

## 4. Giao tiếp giữa các service

### REST API
- Tất cả các service đều cung cấp REST API
- API Gateway đóng vai trò trung gian cho tất cả các request
- Các service giao tiếp trực tiếp với nhau trong mạng nội bộ

### Mạng nội bộ
- Các service chạy trong Docker network
- Service names trong Docker Compose:
  - `gateway-service`
  - `user-service`
  - `team-service`
  - `task-service`
  - `notification-service`

---

## 5. Luồng dữ liệu chính

1. Người dùng gửi yêu cầu qua API Gateway
2. Task Service nhận yêu cầu
3. Task Service kiểm tra người dùng đó có phải leader thông qua Team Service
4. Nếu hợp lệ, yêu cầu được lưu và gửi thông báo cho các thành viên được gán nhiệm vụ
5. Notification Service gửi thông báo qua Email

---

## 6. Roadmap mở rộng

| Phase | Tính năng                                  | Mục đích                |
| ----- | ------------------------------------------ | ----------------------- |
| 1     | Hoàn thiện CRUD Task + Email Notification  | MVP                     |
| 2     | Thêm in‑app notification (WebSocket / SSE) | Real‑time               |
| 3     | Tách **Audit Service** + Kafka             | Event sourcing, tracing |
| 4     | Module báo cáo (task progress, burn‑down)  | BI / Analytics          |

---

## 7. Sơ đồ kiến trúc (tham khảo)

Xem sơ đồ chi tiết tại `docs/assets/task-system-architecture.png` 

---
