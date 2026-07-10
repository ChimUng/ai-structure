# AWS EC2 — Kafka Host

## Mục đích
EC2 dùng DUY NHẤT để tự host Kafka + Zookeeper (không dùng MSK managed).

## Cấu hình cần xác nhận/điền khi setup
- [ ] Instance type: ___ (điền khi chốt, VD t3.medium)
- [ ] Docker Compose file dùng để chạy Kafka+Zookeeper trên EC2 này — có thể tái sử dụng đúng service `kafka`/`zookeeper` trong `docker-compose.yml` gốc của repo, KHÔNG cần file riêng.
- [ ] Kafka `advertised.listeners` phải trỏ về địa chỉ mà ECS Fargate task có thể reach tới (Private IP nếu cùng VPC, hoặc Public IP + security group mở port 9092 nếu khác VPC) — đây là lỗi phổ biến nhất khi Kafka chạy Docker: mặc định advertise `localhost`, ECS sẽ không connect được.
- [ ] Security Group: mở port 9092 (Kafka) CHỈ cho phép từ Security Group của ECS Fargate service (`user-service`, `anime-catalog-service`) — không mở ra Internet.
- [ ] `KAFKA_BOOTSTRAP_SERVERS` ở ECS Task Definition của `user-service`/`anime-catalog-service` trỏ về địa chỉ EC2 này thay vì `localhost:9092`.

## Rủi ro đã biết
- EC2 chạy Kafka là single point of failure (không có replica/cluster) — chấp nhận được ở giai đoạn hiện tại, nhưng cần ghi nhận nếu traffic tăng thì đây là điểm cần nâng cấp lên MSK hoặc cluster nhiều node trước tiên.
- Không có backup/snapshot tự động cho Kafka data trên EC2 — nếu instance chết, message chưa consume có thể mất.