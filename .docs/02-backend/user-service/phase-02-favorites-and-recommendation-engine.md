# Phase 02 — Favorites & Recommendation Engine

## Mục đích
Quản lý danh sách yêu thích và sinh gợi ý anime cá nhân hóa dựa trên hành vi xem (Phase 01).

## Class liên quan
`FavoriteService/Controller`, `RecommendationService/Controller`, `ExternalAnimeService`, entity `Favorite`.

## Favorites
- `addFavorite`: chặn trùng bằng `existsByUserIdAndAnimeId` trước khi tạo → `DuplicateResourceException` nếu đã có.
- Denormalized data (title/cover/banner/status/totalEpisodes): nếu request đã có sẵn (client tự gửi) thì dùng luôn; nếu không → gọi `ExternalAnimeService.getAnimeDetails` sang `anime-catalog-service`. Nếu gọi lỗi/timeout → VẪN tạo favorite nhưng thiếu data denormalized (fallback graceful, không chặn user).
- `toggleNotification`: đảo boolean `notifyNewEpisode` — đây chính là field được `EpisodeEventConsumer` (Phase 03) dùng để quyết định có tạo notification hay không.

## Recommendation Engine
Luồng `generateRecommendations`:
1. Lấy 20 record `WatchHistory` gần nhất → nếu rỗng → trả thẳng `getTrendingAnime()`.
2. Nếu có history: lấy 5 `aniId` gần nhất (distinct) → gọi song song `anime-catalog-service` lấy genres từng anime (`flatMap` concurrency=3, lỗi từng anime chỉ log warn + trả list rỗng, không chặn cả luồng).
3. Đếm tần suất genre → lấy top 3 genre.
4. Lấy trending làm "ứng viên" → tính điểm từng anime = `sum(genreScore * 10)` theo genres trùng với top 3 → lọc score > 0 → sort giảm dần → lấy 10.
5. Cache kết quả 6 giờ theo key `recommendations:{userId}`.

## Constraint
- Recommendation KHÔNG dùng dữ liệu `Favorite` — chỉ dùng `WatchHistory`. Đây là quyết định có chủ đích (favorite thể hiện ý định, watch history thể hiện hành vi thật) — không tự ý gộp 2 nguồn nếu chưa thảo luận.
- `getTop3Genres` (dùng ở `UserStatsController` — Phase 04) tái sử dụng đúng logic `analyzeWatchHistory`, không viết lại.