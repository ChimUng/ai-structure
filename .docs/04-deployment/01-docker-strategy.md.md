# Docker Strategy

## Dev local — `docker-compose.yml`
Chỉ chạy hạ tầng phụ trợ khi dev máy local: Kafka, Zookeeper, MongoDB, Kafka-UI (port 8090 để xem topic/message trực quan). 4 service Spring Boot chạy trực tiếp qua `mvn spring-boot:run`, KHÔNG chạy trong container ở môi trường dev (build container mỗi lần sửa code chậm hơn nhiều so với hot-reload local).

## Production — Dockerfile multi-stage (đã có sẵn từng service)
Mỗi service có `Dockerfile` riêng theo pattern 2 stage giống nhau:
\`\`\`dockerfile
# Stage 1: Build
FROM maven:3.9.9-eclipse-temurin-<version> AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:<version>-jre
WORKDIR /app
COPY --from=builder /app/target/<service>-0.0.1-SNAPSHOT.jar app.jar
EXPOSE <port>
ENTRYPOINT ["java", "-jar", "app.jar"]
\`\`\`

## Bảng Dockerfile hiện có
| Service | Base image builder | Base image runtime | Port |
|---|---|---|---|
| auth-service | maven:3.9.9-eclipse-temurin-21 | eclipse-temurin:21-jdk | 8085 |
| user-service | maven:3.9.9-eclipse-temurin-21 | eclipse-temurin:21-jdk | 8081 |
| anime-catalog-service | maven:3.9.9-eclipse-temurin-21 | eclipse-temurin:21-jre | 8080 |
| animeepisode (episode cũ) | maven:3.9.9-eclipse-temurin-17-alpine | eclipse-temurin:17-jre-alpine | 8084 |
| ai-search-service | maven:3.9.9-eclipse-temurin-21 | eclipse-temurin:21-jdk | 8080 (trùng port catalog — PHẢI đổi trước khi deploy chung) |
| api-gateway | (chưa có Dockerfile — cần tạo khi deploy ECS) | | 4004 |

## Lưu ý bất nhất cần xử lý trước khi lên production
- Java version LỆCH giữa các service: `auth/user/catalog` dùng JDK 21 runtime image (nặng, có compiler — production nên đổi sang `-jre` để nhẹ hơn), `animeepisode` dùng JDK 17 alpine. Cần thống nhất version khi viết Dockerfile cho `episode-service`/`ai-search-service` mới.
- `ai-search-service` Dockerfile khai `EXPOSE 8080` — TRÙNG với `anime-catalog-service`. Không ảnh hưởng gì trong container riêng biệt (mỗi container 1 network namespace), nhưng cần map port khác nhau rõ ràng ở ECS Task Definition để tránh nhầm lẫn khi đọc log/debug.
- `api-gateway` chưa có Dockerfile — cần bổ sung theo đúng pattern bảng trên trước khi đưa lên ECS.