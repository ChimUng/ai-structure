# Phase 01 — Watch Activity Tracking

## Mục đích
Ghi nhận và quản lý tiến độ xem của người dùng — nền tảng dữ liệu cho Recommendation (Phase 02) và Notification liên quan.

## Class liên quan
`WatchHistoryService/Controller`, `ContinueWatchingService/Controller`, entity `WatchHistory`, `ContinueWatching`.

## Luồng chính (`addOrUpdateHistory`)
1. Tìm record cũ theo `(userId, aniId, epId)` — nếu có thì update, không thì tạo mới.
2. Tính `progress = timeWatched / duration` (an toàn với `duration null/0` → trả 0.0).
3. Sau khi save `WatchHistory` → gọi `syncContinueWatchingAsync` KHÔNG chờ kết quả (`subscribe()` riêng, không block response client).

## Continue Watching — logic tự động dọn dẹp
- `updateFromHistory`: nếu `completed=true` HOẶC `progress >= 0.9` → XÓA khỏi continue-watching (coi như đã xem xong, không cần "xem tiếp" nữa).
- Ngược lại: upsert record, cập nhật `lastWatchedAt = now()`.
- `cleanupOldEntries`: sau mỗi lần update, đếm tổng số item của user — nếu vượt `features.continue-watching.max-items` (default 20) → xóa bớt các entry cũ nhất theo `lastWatchedAt ASC`.

## Index quan trọng
- `ContinueWatching`: compound unique `(userId, aniId)` — đảm bảo mỗi anime chỉ có 1 entry continue-watching/user.
- `WatchHistory`: compound `(userId, aniId)` và `(userId, createdAt DESC)`.

## Constraint
- KHÔNG gọi `ContinueWatchingService` đồng bộ (blocking) trong `addOrUpdateHistory` — bắt buộc giữ pattern `subscribe()` riêng để response về client không bị chờ 2 lần ghi Mongo.
- Ngưỡng `0.9` (90%) để coi là "xem xong" là hardcode trong `ContinueWatchingService` — nếu cần cấu hình được, phải đưa ra `application.yml` giống `max-items`.