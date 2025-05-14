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

## 4. Luồng dữ liệu chính

### 4.1 Tạo task

1. **Client → Gateway**: `POST /api/task` kèm JWT.
2. **Gateway → User‑Service**: Check token (introspect) → **200 OK** hoặc **401**.
3. **Gateway → Task‑Service**: Chuyển tiếp request.
4. **Task‑Service → Team‑Service**: `GET /teams/is-leader?userId=…?teamId=…`

   * Nếu `false` → **403 Forbidden**.
5. **Task‑Service**:

   * Lưu bản ghi **tasks**.
   * Lưu **task\_assignments** (n\:many).
6. **Task‑Service → Notification‑Service**:

   ```
   {
     "title": "New Task: Design Dashboard",
     "receiverIds": [3,5,8],
     "content": "...",
     "channel": ["EMAIL","IN_APP"]
   }
   ```
7. **Notification‑Service** gửi email → SMTP / (tùy) FCM, ghi log.
8. **Task‑Service → Gateway → Client**: Trả **201 Created** cùng payload task.

### 4.2 Xem task

*Luồng tương tự, bỏ bước kiểm tra trưởng nhóm nếu chỉ view.*

### 4.3 Xóa task

*Luồng tương tự, bỏ bước kiểm tra trưởng nhóm nếu chỉ view.*

---

## 5. Mô hình dữ liệu tối thiểu

| Bảng               | Trường chính                                                                               | Ghi chú              |
| ------------------ | ------------------------------------------------------------------------------------------ | -------------------- |
| `users`            | `user_id PK`, `email`, `display_name`, `password_hash`, `status`                           | User Service         |
| `teams`            | `team_id PK`, `name`, `created_by`                                                         | Team Service         |
| `team_members`     | `team_id FK`, `user_id FK`, `role ENUM('ADMIN','MEMBER')`                                  |                      |
| `tasks`            | `task_id PK`, `team_id FK`, `title`, `description`, `due_date`, `creator_id`, `created_at` | Task Service         |
| `task_assignments` | `task_assign_id PK`, `task_id FK`, `user_id FK`, `status ENUM('PENDING','DONE')`           |                      |
| `notifications`    | `notify_id PK`, `user_id FK`, `title`, `content`, `channel`, `sent_at`, `status`           | Notification Service |


---

## 6. Bảo mật & Middleware

| Lớp                 | Mô tả                                                                         | Công cụ            |
| ------------------- | ----------------------------------------------------------------------------- | ------------------ |
| **Auth**            | JWT (HS256); lưu public‑key ở Gateway; refresh‑token qua User‑Service         | Spring Security 6  |
| **Rate limit**      | Giới hạn 100 req/10 phút / IP (configurable)                                  | Bucket4j @ Gateway |
| **API Logging**     | Sleuth / Micrometer → Grafana                                                 | Zipkin, Prometheus |
| **Circuit‑Breaker** | Tùy chọn khi thêm Kafka hoặc nhiều backend                                    | Resilience4j       |

---

## 7. DevOps & CI/CD (gợi ý)

1. **Gradle 8.13** multi‑project; mỗi service tự build Docker image:

   ```
   ./gradlew :task-service:jibDockerBuild
   ```
2. **Docker Compose** cho local dev (`infra/docker-compose.yml`).
3. **GitHub Actions**: on push → build & test → publish image → deploy (Railway / Render / k8s).

---

## 8. Roadmap mở rộng

| Phase | Tính năng                                  | Mục đích                |
| ----- | ------------------------------------------ | ----------------------- |
| 1     | Hoàn thiện CRUD Task + Email Notification  | MVP                     |
| 2     | Thêm in‑app notification (WebSocket / SSE) | Real‑time               |
| 3     | Tách **Audit Service** + Kafka             | Event sourcing, tracing |
| 4     | Module báo cáo (task progress, burn‑down)  | BI / Analytics          |

---

## 9. Sơ đồ kiến trúc (tham khảo)

Xem sơ đồ chi tiết tại `docs/assets/task-system-architecture.png` 

```markdown
![Task System – High‑Level Diagram](assets/task-system-architecture.png)
```

---
