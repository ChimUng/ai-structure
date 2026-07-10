# PROJECT_REQUIREMENTS.md

## Phần 1 — Service đã code xong, CHỈ CẦN refactor "Clear Code"

`auth-service`, `anime-catalog-service`, `user-service`, `api-gateway`: nghiệp vụ đã hoàn chỉnh, không cần thêm/sửa logic. Việc còn lại DUY NHẤT là refactor comment + số thứ tự theo CLAUDE.md mục 4.2.

### Checklist refactor cho mỗi service (áp dụng lặp lại 4 lần)
1. Liệt kê toàn bộ endpoint trong từng Controller theo đúng thứ tự khai báo hiện tại trong file.
2. Đánh số N tăng dần cho mỗi endpoint, giữ nguyên thứ tự vật lý trong file (không sắp xếp lại).
3. Tìm method Service tương ứng phục vụ endpoint đó → gán cùng số N.
4. Method phụ trợ riêng cho N → tách thành method riêng (nếu đang inline lambda dài) và đánh Na, Nb...
5. Hàm cache-aside generic → chuyển lên đầu class nếu chưa ở đó.
6. Hàm dùng chung nhiều endpoint → chuyển xuống cuối class, đánh `// Common.`.
7. Repository: thêm/sửa comment ghi rõ API nào dùng method đó.
8. KHÔNG đổi: tên method public, signature, logic nghiệp vụ, tên field entity.

### Phạm vi cụ thể cần refactor
| Service | File cần refactor |
|---|---|
| auth-service | `UserController`, `DeveloperController`, `UserService`, `DeveloperService`, `TokenService`, `UserRepository`, `SessionRepository`, `DeveloperRepository` |
| anime-catalog-service | `AnimeController`, `AnimeService`, `AnimeSyncService`, `AnimeRepository`, `ScheduleRepository` |
| user-service | `FavoriteController`, `NotificationController`, `WatchHistoryController`, `ContinueWatchingController`, `RecommendationController`, `UserPreferenceController`, `UserStatsController` + toàn bộ Service/Repository tương ứng |
| api-gateway | `AuthenticationFilter` (đánh số theo bước xử lý, không phải endpoint vì gateway không có Controller REST thường) |

Kết quả bàn giao: PR/diff CHỈ chứa thay đổi comment + tách method phụ trợ, KHÔNG có thay đổi hành vi runtime.

## Phần 2 — Service mới, cần đặc tả yêu cầu đầy đủ

### 2.1 `ai-search-service`
- Mục tiêu: semantic search anime bằng vector embedding.
- Nguồn dữ liệu: consume `GET /internal/embed-data` (`AnimeEmbedDTO`, paginated) từ `anime-catalog-service` — KHÔNG query trực tiếp MongoDB catalog.
- Đồng bộ: job cron định kỳ (đề xuất mỗi 6h) pull embed-data → generate embedding → upsert vector store.
- Vector store: cần chốt công nghệ (candidate Qdrant/Milvus/pgvector) — quyết định ghi tại `docs/02-backend/ai-search-service/01-vector-embedding-strategy.md`.
- API tối thiểu: `GET /api/search/semantic?q=...&limit=20` → `ApiResponse<List<AnimeSearchResult>>` (id, title, matchScore, coverImage).
- Auth: tái sử dụng `X-API-KEY` qua api-gateway, không tự làm auth riêng.
- Phải viết mới HOÀN TOÀN theo `PROJECT_ARCHITECTURE_TEMPLATE.md` + quy ước 4.2 ngay từ đầu (không có nợ kỹ thuật để refactor sau).

### 2.2 `episode-service`
- Mục tiêu: tách domain "tập phim/streaming" khỏi `anime-catalog-service` và `user-service`.
- Sở hữu dữ liệu: `epId`, `epNum`, `epTitle`, `nextepId`, `nextepNum`, `provider` (gogoanime/zoro/animepahe), `subtype` (sub/dub), stream link.
- API tối thiểu: `GET /api/episodes/{aniId}`, `GET /api/episodes/{aniId}/{epNum}/sources`.
- Migration: `anime-catalog-service`/`user-service` sẽ gọi service này qua WebClient dần dần, KHÔNG xóa field denormalized cũ ngay (xem `docs/02-backend/episode-service/01-data-migration-plan.md`).
- Kafka: tái sử dụng topic `anime.episode.new` đã có, không tạo topic trùng chức năng.
- Viết mới HOÀN TOÀN theo `PROJECT_ARCHITECTURE_TEMPLATE.md` + quy ước 4.2 ngay từ đầu.