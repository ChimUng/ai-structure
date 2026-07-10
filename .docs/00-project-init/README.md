# Project Init — AnimeFlix Backend

## Yêu cầu môi trường
- Java 17
- Maven 3.9+
- Docker + Docker Compose (chạy Kafka, Zookeeper, MongoDB, Kafka-UI)

## Setup lần đầu
\`\`\`bash
git clone <repo>
cd animeflix
docker-compose up -d          # Kafka (9092), Zookeeper, Mongo (27017), Kafka-UI (8090)
\`\`\`

## Chạy từng service (dev local)
\`\`\`bash
cd auth-service && mvn spring-boot:run             # port 8085
cd anime-catalog-service && mvn spring-boot:run    # port 8080
cd user-service && mvn spring-boot:run             # port 8081
cd api-gateway && mvn spring-boot:run              # port 4004
\`\`\`

## Biến môi trường tối thiểu cần có
Xem đầy đủ tại `docs/04-deployment/02-environment-variables.md`. Quan trọng nhất: `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `REDIS_URL` (auth-service), `KAFKA_BOOTSTRAP_SERVERS` (anime-catalog-service, user-service).

## Thứ tự khởi động khuyến nghị
1. Docker Compose (hạ tầng)
2. `auth-service` (các service khác gọi validate-key tới đây)
3. `anime-catalog-service`
4. `user-service`
5. `api-gateway` (cuối cùng, vì nó proxy tới các service trên)

## Đọc tiếp theo
- Quy tắc code: `CLAUDE.md` (root)
- Kiến trúc tổng thể: `docs/01-system-design/README.md`
- Chi tiết từng service: `docs/02-backend/<service-name>/`