# AWS ECS Fargate — Application Services

## Phạm vi
4 service Spring Boot: `api-gateway`, `auth-service`, `anime-catalog-service`, `user-service` — mỗi service = 1 ECS Service riêng, chạy trên launch type Fargate (không tự quản EC2 worker node).

## Mỗi service cần (checklist khi tạo Task Definition)
- [ ] 1 Task Definition riêng, container image từ Dockerfile tương ứng (build + push lên ECR).
- [ ] CPU/Memory: điền khi chốt (Fargate yêu cầu tổ hợp CPU/Memory hợp lệ theo bảng AWS, VD 0.25 vCPU/0.5GB cho service nhẹ).
- [ ] Port mapping đúng port thật của service (8085, 8080, 8081, 4004).
- [ ] Environment variables: set đủ biến 🔴 nhạy cảm theo bảng ở file 02, KHÔNG dựa vào default trong yml cho môi trường production.
- [ ] Network mode: `awsvpc` (bắt buộc với Fargate).
- [ ] Security Group riêng cho ECS Service, cho phép:
  - Inbound từ ALB (nếu có Load Balancer đứng trước)
  - Outbound tới EC2 Kafka (port 9092)
  - Outbound tới MongoDB (Atlas hoặc self-host — cần xác nhận nơi host Mongo production, hiện chỉ thấy config `localhost:27017` trong `application.yml`, chưa rõ production trỏ đâu)
  - Outbound tới Redis (Upstash — qua Internet, không cần Security Group riêng vì Upstash là managed ngoài VPC)

## Routing giữa các service trên ECS
- `api-gateway` cần biết địa chỉ nội bộ của `auth-service`/`anime-catalog-service` — dùng ECS Service Discovery (AWS Cloud Map) để có DNS nội bộ ổn định (VD `auth-service.animeflix.local`) thay vì hardcode IP (Fargate task IP đổi mỗi lần restart).
- `user-service` gọi `anime-catalog-service`/`auth-service` cũng theo cùng cơ chế Service Discovery.
- **Cần xác nhận**: hiện `user-service` KHÔNG có route qua `api-gateway` (xem `docs/01-system-design/README.md`) — khi lên production, client sẽ gọi thẳng ECS Service của `user-service` qua ALB riêng, hay sẽ bổ sung route ở gateway? Quyết định này ảnh hưởng tới việc có cần expose `user-service` ra ALB public hay không.

## Load Balancer
- 1 Application Load Balancer đứng trước `api-gateway` (public-facing) — đây là entry point duy nhất theo thiết kế `01-system-design`.
- Target Group trỏ vào ECS Service của `api-gateway`, health check path: cần thêm actuator health endpoint nếu chưa có (hiện Dockerfile của `animeepisode` có `HEALTHCHECK` gọi `/actuator/health`, các service khác CHƯA thấy cấu hình tương tự — cần bổ sung `spring-boot-starter-actuator` + expose `/actuator/health` cho cả 4 service trước khi gắn ALB health check).

## Việc còn thiếu (đã biết) trước khi go-live thật
- [ ] Actuator health check cho `auth-service`, `anime-catalog-service`, `user-service`, `api-gateway`.
- [ ] Dockerfile cho `api-gateway`.
- [ ] Chuẩn hóa secret qua Secrets Manager thay vì hardcode default trong yml.
- [ ] Xác nhận nơi host MongoDB production (Atlas hay self-host).
- [ ] Quyết định route `user-service` qua Gateway hay ALB riêng.