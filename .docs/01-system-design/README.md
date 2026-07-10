# System Design — AnimeFlix Backend

## 1. Service Map

| Service | Port | Database | Vai trò |
|---|---|---|---|
| api-gateway | 4004 | - | Entry point duy nhất, route + validate API key |
| auth-service | 8085 | animeflix_auth (Mongo) + Redis | JWT, session, developer key + rate limit |
| anime-catalog-service | 8080 | animeflix_catalog (Mongo) + Redis | Nguồn dữ liệu anime (sync từ AniList), publish sự kiện tập mới |
| user-service | 8081 | animeflix_user (Mongo) + Redis | Hành vi người dùng: history, favorite, notification, recommendation |
| episode-service | TBD | TBD | (chưa code) — sẽ tách domain episode/streaming khỏi catalog & user |
| ai-search-service | TBD | TBD | (chưa code) — semantic search dựa trên vector embedding |

Hướng phụ thuộc hiện tại (ai gọi ai qua WebClient):
\`\`\`
api-gateway ──> auth-service (validate X-API-KEY)
user-service ──> anime-catalog-service (lấy basic info anime cho favorite/history)
user-service ──> auth-service (lấy memberSince cho UserStats)
Client ──> api-gateway ──> anime-catalog-service / auth-service (route trực tiếp)
\`\`\`

> Lưu ý: `user-service` hiện KHÔNG đi qua `api-gateway` để gọi 2 service kia — nó gọi thẳng qua WebClient nội bộ (xem `docs/02-backend/user-service/`).

## 2. Giao tiếp Inter-Service (REST)

Quy ước bắt buộc khi 1 service gọi service khác:
- Mỗi service đích có 1 `@Bean WebClient` riêng, đặt tên rõ (`animeCatalogWebClient`, `authServiceWebClient`), khai báo ở `config/WebClientConfig.java`.
- Luôn có `.timeout(Duration.ofSeconds(5))`.
- Luôn có `.onErrorResume(...)` trả về giá trị an toàn (`Mono.empty()`, danh sách rỗng) — KHÔNG để lỗi tầng dưới làm sập response tầng trên.
- Header nội bộ dùng để truyền danh tính: `X-User-Id` (do Gateway/Filter set sau khi validate, service con đọc qua `SecurityContextUtil`).
- Hiện TẤT CẢ lời gọi liên-service đều thiếu Circuit Breaker thật sự — xem `.claude/skills/microservice-resilience/SKILL.md` để biết kế hoạch bổ sung Resilience4j.

## 3. API Gateway Routing

Chi tiết đầy đủ: `docs/02-backend/api-gateway/`. Tóm tắt:
- Route `/api/auth/**` → `auth-service`, bỏ qua `AuthenticationFilter` (auth-service tự lo xác thực nội bộ của nó).
- Route `/anime/**` → `anime-catalog-service`, bắt buộc qua `AuthenticationFilter` (check `X-API-KEY` → gọi `auth-service` validate).
- `user-service` KHÔNG có route nào qua Gateway hiện tại (client gọi thẳng port 8081) — cần bổ sung route khi lên production.

## 4. Kiến trúc hướng sự kiện (Kafka)

- Topic duy nhất hiện có: `anime.episode.new`.
- Producer: `anime-catalog-service` (`EpisodeEventPublisher`), key = `animeId` để đảm bảo thứ tự theo từng anime.
- Consumer: `user-service` (`EpisodeEventConsumer`), group `user-service-notifications`, manual ack, luôn ack dù thành công hay lỗi (tránh infinite retry loop — chấp nhận mất 1 notification hiếm khi lỗi thay vì block partition).
- Schema event: `NewEpisodeEvent` (duplicate định nghĩa ở cả 2 service — `com.animeflix.animecatalogservice.DTO.kafka` và `com.animeflix.userservice.dto.kafka` — do khác nhau package tránh phụ thuộc chéo module).
- Khi thêm `episode-service`: TÁI SỬ DỤNG topic này, không tạo topic mới trùng chức năng (xem `PROJECT_REQUIREMENTS.md` mục episode-service).

## 5. Câu hỏi mở / chưa chốt
- Chưa có API Gateway route cho `user-service` — cần bổ sung trước khi deploy production.
- Chưa có Circuit Breaker cho lời gọi liên-service — kế hoạch ở skill `microservice-resilience`.
- `episode-service` sẽ đứng ở vị trí nào trong sơ đồ dependency (client gọi thẳng hay qua Gateway) — cần quyết định khi bắt đầu code.