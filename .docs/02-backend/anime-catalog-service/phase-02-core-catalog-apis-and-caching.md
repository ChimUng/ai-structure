# Phase 02 — Luồng API tra cứu & Chiến lược Caching

## Mục đích
Phục vụ Client (và các service khác) tra cứu anime với hiệu năng cao, tránh mọi request đập thẳng MongoDB.

## Class liên quan
`AnimeController`, `AnimeService`, `AnimeMapper`.

## Pattern Cache-Aside dùng chung
Helper generic `getFromCacheOrDb(key, dbFallback, typeRef)`:
1. Đọc Redis bằng `StringRedisTemplate` (đồng bộ, không phải Reactive — lưu ý khác với `user-service`/`auth-service` dùng `ReactiveRedisTemplate`).
2. Nếu cache hit nhưng parse JSON lỗi → xóa key, coi như cache miss (tự phục hồi, không throw).
3. Nếu cache miss → chạy `dbFallback`, ghi lại cache TTL 6 giờ (`Duration.ofHours(6)`), lỗi ghi cache chỉ log warn, không fail request.

> Cảnh báo kiến trúc: `AnimeService` dùng `StringRedisTemplate` (blocking) bên trong pipeline reactive — đây là điểm KHÁC BIỆT so với 2 service kia (dùng Reactive Redis thuần). Không tự ý "sửa cho nhất quán" nếu chưa xác nhận — có thể là quyết định có chủ đích để đơn giản hóa code cache đồng bộ.

## Danh sách endpoint (khớp bảng số thứ tự ở `04-endpoints.md`)
| # | Endpoint | Cache key pattern | Ghi chú |
|---|---|---|---|
| 1 | `GET /{id}` | `id:{id}` | Có lazy-load (xem Phase 01) nếu thiếu detail |
| 2 | `GET /popular` | `popular:{page}:{perPage}` | Sort theo `popularity` |
| 3 | `GET /trending` | `trending:{page}:{perPage}` | Sort theo `averageScore` |
| 4 | `GET /popularmovie` | `movies:{page}` | Filter `format=MOVIE` |
| 5 | `GET /season` | `currentseason:{season}:{year}:{page}` | Tự tính season/year theo tháng hiện tại |
| 6 | `GET /top100` | `top100:{page}` | Sort theo `averageScore` |
| 7 | `GET /nextseason` | `nextseason:{season}:{year}:{page}` | |
| 8 | `GET /schedule` | `schedule:week` | Query `ScheduleRepository`, group theo ngày trong tuần (tiếng Việt) |
| 9 | `GET /search` | Không cache | Proxy trực tiếp GraphQL AniList realtime |
| 10 | `GET /internal/embed-data` | Không cache | Nội bộ cho `ai-search-service`, không qua Redis vì dùng 1 lần cho batch job |

## Lý do `/search` KHÔNG cache
Search là truy vấn động (nhiều tổ hợp filter), cache theo mọi tổ hợp tham số sẽ tốn Redis vô ích và tỉ lệ hit thấp — quyết định có chủ đích, giữ nguyên khi refactor.

## Constraint
- Mọi endpoint mới thêm vào Controller PHẢI đi qua `getFromCacheOrDb` nếu là read-only list — không tự viết cache riêng lẻ.