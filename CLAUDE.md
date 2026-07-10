# CLAUDE.md — AnimeFlix Backend (Microservices)

## 1. Project Overview
AnimeFlix là hệ thống backend microservices xem anime gồm 4 service: `api-gateway`, `auth-service`, `anime-catalog-service`, `user-service`. Đã mở rộng thêm `ai-search-service` (semantic search) và `episode-service` (tách quản lý episode/streaming khỏi catalog). Toàn bộ stack dùng Spring WebFlux reactive, tuyệt đối không dùng blocking code (Spring MVC).

## 2. Tech Stack
- Java 17, Spring Boot 3.5.7 (api-gateway dùng 3.3.4)
- Spring WebFlux + Project Reactor (Mono/Flux) — KHÔNG dùng servlet stack
- MongoDB Reactive Driver, Redis Reactive (Upstash), mỗi service 1 DB riêng
- Kafka: spring-kafka + reactor-kafka — topic `anime.episode.new`
- JWT: io.jsonwebtoken (jjwt) 0.12.6
- MapStruct 1.5.5.Final + Lombok
- Spring Cloud Gateway 2023.0.3 (api-gateway)

## 3. Dev Commands
\`\`\`bash
docker-compose up -d   # Kafka, Zookeeper, Mongo, Kafka-UI
cd auth-service && mvn spring-boot:run           # 8085
cd anime-catalog-service && mvn spring-boot:run  # 8080
cd user-service && mvn spring-boot:run           # 8081
cd api-gateway && mvn spring-boot:run             # 4004
mvn clean install -DskipTests                     # build 1 service
\`\`\`

## 4. Code Style Rules (BẮT BUỘC — không ngoại lệ)

### 4.1 Comment
- Chỉ dùng `// ...` một dòng. TUYỆT ĐỐI KHÔNG dùng `/* ... */`.
- Tên hàm + TOÀN BỘ tham số nằm trên 1 dòng, không wrap dù dài.
- Chỉ xuống dòng khi: gọi chuỗi hàm builtin (`.flatMap()`, `.map()`...) hoặc trước `return`.

### 4.2 Quy ước đánh số thứ tự (Clear Code Numbering) — BẮT BUỘC cho mọi Controller/Service/Repository
- Controller: mỗi endpoint = `// N. <Tên chức năng>`.
- Service: method chính phục vụ endpoint N → dùng ĐÚNG số N (dù tên method khác tên endpoint).
- Service: method phụ trợ CHỈ phục vụ riêng method N → đánh `// Na.`, `// Nb.`, `// Nc.` đặt NGAY BÊN DƯỚI method N.
- Service: hàm cache-aside generic (VD: `getFromCacheOrDb`) → đặt TRÊN CÙNG class, không đánh số.
- Service: hàm dùng chung nhiều endpoint (VD: parse JSON, build DTO chung) → đặt DƯỚI CÙNG class, đánh `// Common.`.
- Repository: mỗi method → comment mô tả kèm API nào dùng: `// Tìm ... — dùng cho api /xxx`. Nếu dùng chung nhiều API thì liệt kê hết.
- Xem ví dụ đầy đủ tại `docs/00-project-init/02-coding-conventions.md`.

## 5. Kiến trúc & Index tài liệu (Progressive Disclosure)
Chi tiết KHÔNG nằm ở đây — luôn tra `.docs/` trước khi code:
- Giao tiếp REST/Kafka, API Gateway → `.docs/01-system-design/`
- Chi tiết từng service → `.docs/02-backend/<service-name>/`
- Testing strategy → `.docs/03-testing/`
- Deploy/Docker/CI → `.docs/04-deployment/`
- Đặc tả tính năng đầy đủ → `PROJECT_REQUIREMENTS.md`
- Chuẩn package structure cho service mới → `PROJECT_ARCHITECTURE_TEMPLATE.md`

## 6. Trạng thái thực tế từng service (cập nhật theo tiến độ)
| Service | Trạng thái | Việc còn lại |
|---|---|---|
| api-gateway | Code xong | Refactor theo quy ước 4.2 |
| auth-service | Code xong | Refactor theo quy ước 4.2 |
| anime-catalog-service | Code xong | Refactor theo quy ước 4.2 |
| user-service | Code xong | Refactor theo quy ước 4.2 |
| ai-search-service | Chưa code | Viết mới theo `PROJECT_ARCHITECTURE_TEMPLATE.md` + `PROJECT_REQUIREMENTS.md` |
| episode-service | Chưa code | Viết mới theo `PROJECT_ARCHITECTURE_TEMPLATE.md` + `PROJECT_REQUIREMENTS.md` |

> Lưu ý cho AI: "Refactor theo quy ước 4.2" nghĩa là CHỈ thêm/sửa comment + format số thứ tự, KHÔNG được đổi logic nghiệp vụ, KHÔNG đổi tên method public, KHÔNG đổi signature. Nếu phát hiện bug trong lúc refactor, báo cáo riêng, không tự sửa.

## 7. Key Constraints — AI KHÔNG được tự ý
- KHÔNG chuyển bất kỳ service nào từ reactive (Mono/Flux) sang blocking.
- KHÔNG đổi tên field MongoDB đã có index (`userId`, `aniId`, `animeId`, `epId`...) — sẽ vỡ compound index.
- KHÔNG thêm dependency mới vào `pom.xml` nếu chưa xác nhận.
- Mọi endpoint mới PHẢI trả về `ApiResponse<T>` (đã có sẵn ở `exception/ApiResponse.java` mỗi service).
- userId luôn lấy qua header `X-User-Id` bằng `SecurityContextUtil.getCurrentUserId(exchange)`, KHÔNG tự parse JWT ở service con.
- `ai-search-service` và `episode-service` PHẢI dựng theo đúng `PROJECT_ARCHITECTURE_TEMPLATE.md`.

## 8. Additional Documentation
- `.docs/02-backend/auth-service/phase-01-reactive-core.md`
- `.docs/02-backend/auth-service/phase-02-token-rotation.md`
- `.docs/02-backend/auth-service/phase-03-security.md`
- `.docs/02-backend/user-service/phase-01-watch-activity-tracking.md`
- `.docs/02-backend/user-service/phase-02-favorites-and-recommendation-engine.md`
- `.docs/02-backend/user-service/phase-03-realtime-notification-system.md`
- `.docs/02-backend/user-service/phase-04-preferences-and-user-stats.md`
- `.docs/02-backend/anime-catalog-service/phase-01-anilist-sync-mechanism.md`
- `.docs/02-backend/anime-catalog-service/phase-02-core-catalog-apis-and-caching.md`
- `.docs/02-backend/anime-catalog-service/phase-03-airing-schedule-and-kafka-events.md`
