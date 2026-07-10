# Phase 01 — Cơ chế đồng bộ dữ liệu AniList

## Mục đích
Nạp và duy trì dữ liệu anime từ AniList GraphQL API vào MongoDB nội bộ, làm nguồn dữ liệu gốc cho toàn bộ service (kể cả `ai-search-service` sau này qua endpoint embed-data).

## Class liên quan
`AnimeSyncService`, `AnimeRepository`, `ScheduleRepository`, entity `Anime`, `AnimeSchedule`.

## 3 cơ chế đồng bộ song song

### 1. Master Sync toàn bộ (`syncAllData`)
- Cron: `0 0 2 * * ?` (2h sáng mỗi ngày).
- Query GraphQL `advanced-search.graphql`, phân trang 100 page × 50 item = tối đa 5000 anime.
- `delayElements(800ms)` giữa mỗi page để né rate-limit AniList (~75 req/phút).
- `concatMap` (tuần tự, không song song) + `takeUntil(hasNextPage == false)` để dừng đúng lúc.
- Merge thông minh: nếu anime đã tồn tại trong DB, giữ lại field chi tiết cũ (`characters`, `relations`, `studios`, `recommendations`, `trailer`) nếu response mới không có — tránh mất dữ liệu do `advanced-search.graphql` không trả field chi tiết.
- `mergeTitles`: merge từng field con của Title (romaji/english/userPreferred/native) thay vì merge cả object, vì AniList có thể trả thiếu 1 field ngôn ngữ.

### 2. Lazy-load chi tiết (`fetchAndSaveAnimeDetail`)
- Được gọi khi 1 anime được truy vấn chi tiết (`AnimeService.getAnimeInfo`) nhưng đang thiếu `characters`/`relations` trong DB.
- Query riêng `anime-info.graphql` (đầy đủ hơn advanced-search) theo đúng 1 ID.
- Có retry `Retry.backoff(3, 10s)` CHỈ khi lỗi `TooManyRequests` (429), không retry lỗi khác.
- Merge title với data cũ trước khi save (tương tự master sync).

### 3. Bulk Enrichment nền (`syncMissingDetailsInBulk`)
- Cron: `0 0 4 * * ?` (4h sáng, sau master sync 2 tiếng để tránh trùng tải).
- Quét toàn bộ anime đang thiếu `characters`, buffer 3 ID/lần, `delayElements(3s)` giữa mỗi batch — mục tiêu làm đầy dần 5000 bộ mà không spam AniList.
- Gọi lại chính `fetchAndSaveAnimeDetail` (không viết logic riêng).

## Sync lịch chiếu (`syncSchedules`)
- Cron: `0 0 */4 * * ?` (mỗi 4 tiếng).
- Query `schedule-anime.graphql`, tối đa 10 page, `delayElements(1s)`.
- Trước khi save: `filterWhen` check `existsByAnimeIdAndEpisode` để tránh insert trùng (không dùng unique index bắt lỗi vì muốn skip êm, không throw).
- Sau khi có list mới: xóa toàn bộ schedule cũ hơn thời điểm hiện tại (`deleteByAiringAtLessThan`) rồi mới insert list mới — đảm bảo dữ liệu lịch luôn "nhìn về tương lai".
- Field `expiresAt` dùng TTL index tự xóa 1 giờ sau khi phát sóng — không cần cron dọn riêng.

## Constraint quan trọng
- KHÔNG được gọi AniList API đồng thời (song song) — luôn `concatMap`/tuần tự có `delayElements`, vì AniList rate-limit nghiêm ngặt và code hiện tại xử lý 429 bằng retry backoff thủ công, không phải circuit breaker.
- Merge logic (`mergeTitles`, giữ field chi tiết cũ) là phần DỄ VỠ NHẤT — bất kỳ thay đổi nào ở đây phải test kỹ với case "anime đã có detail cũ, sync mới về ít field hơn".