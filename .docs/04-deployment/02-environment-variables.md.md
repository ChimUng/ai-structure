# Environment Variables

## Trạng thái hiện tại — CẦN LƯU Ý
Toàn bộ biến môi trường (kể cả secret: JWT key, Redis URL có password, Mongo URI) đang được set qua `application.yml` với default value fallback dạng `${VAR_NAME:giá-trị-mặc-định}`. Nghĩa là NẾU không set ENV var khi deploy, hệ thống vẫn chạy được bằng giá trị hardcode sẵn trong file — điều này đồng nghĩa **secret thật đang nằm trong source code** (VD: `jwt.access-secret` ở `auth-service/application.yml`, Redis password ở URL Upstash).

Đây là rủi ro bảo mật thật sự (secret bị lộ nếu repo public hoặc log ra) — không phải lỗi thao tác, mà là điểm CẦN NÂNG CẤP khi có thời gian: chuyển sang AWS Secrets Manager / Parameter Store, xóa default value nhạy cảm khỏi `application.yml`, chỉ giữ default cho giá trị KHÔNG nhạy cảm (port, timeout...).

## Bảng biến môi trường theo service (giá trị thật lấy từ ENV khi deploy ECS, override default trong yml)

| Service | Biến | Nhạy cảm? | Ghi chú |
|---|---|---|---|
| auth-service | `JWT_ACCESS_SECRET` | 🔴 Có | Base64, >= 256 bit |
| auth-service | `JWT_REFRESH_SECRET` | 🔴 Có | Khác key access |
| auth-service | `REDIS_URL` | 🔴 Có (chứa password trong URL) | Upstash rediss:// |
| anime-catalog-service | `services.anime-catalog.url` tương đương | ⚪ Không | Internal URL |
| user-service | `KAFKA_BOOTSTRAP_SERVERS` | ⚪ Không | Trỏ vào EC2 host Kafka (xem file 03) |
| user-service | `ANIME_CATALOG_URL`, `AUTH_SERVICE_URL` | ⚪ Không | Internal service URL |
| ai-search-service | `GEMINI_API_KEY` | 🔴 Có | |
| ai-search-service | Mongo Atlas URI | 🔴 Có | Đang hardcode thẳng trong yml (không qua ENV var nào cả — mức rủi ro cao nhất trong toàn hệ thống) |
| api-gateway | `service.auth.url`, `service.anime.url` | ⚪ Không | |

## Việc cần làm khi deploy ECS Fargate
Ở bước 04 (ECS Task Definition), phần "Environment variables" của mỗi Task Definition PHẢI set đủ các biến 🔴 ở trên bằng giá trị thật (khác với default trong yml) — nếu quên set, service vẫn chạy được (do có fallback) nhưng đang dùng đúng secret hardcode trong code, KHÔNG phải secret production riêng.