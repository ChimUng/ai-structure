# Phase 04 — User Preferences & Aggregated Stats

## Mục đích
Quản lý cài đặt cá nhân của user và tổng hợp số liệu thống kê từ các phase khác.

## Class liên quan
`UserPreferenceService/Controller`, entity `UserPreference`, `UserStatsController`.

## User Preferences
- `getPreferences`: nếu chưa có → tự tạo `createDefaultPreferences` (notification bật, autoplay bật, quality 1080p, subtype sub...).
- `updatePreferences`: dùng `UserPreferenceMapper` (MapStruct, `NullValuePropertyMappingStrategy.IGNORE`) — field null trong request KHÔNG ghi đè giá trị cũ.
- `isNotificationEnabled`: dùng RIÊNG bởi `EpisodeEventConsumer` (Phase 03), default `true` nếu chưa có preference (tránh block notification cho user mới).

## User Stats (tổng hợp đa nguồn)
`UserStatsController.buildStats` chạy SONG SONG qua `Mono.zip` 6 nguồn:
1. `historyService.countAnimeWatched` (distinct aniId, Phase 01)
2. `historyService.getTotalWatchedSeconds` (Phase 01)
3. `favoriteService.countFavorites` (Phase 02)
4. `notificationService.countUnread` (Phase 03)
5. `recommendationService.getTop3Genres` — tái sử dụng logic Phase 02, `onErrorReturn(List.of())`
6. Gọi `authServiceWebClient` lấy `memberSince` từ `/api/auth/user/me`, kèm cookie `access_token` lấy từ request — `onErrorReturn(LocalDateTime.now())` nếu lỗi.

## Constraint
- Việc gọi sang `auth-service` bằng cookie forward thủ công (không qua Gateway) là điểm DUY NHẤT trong `user-service` gọi `auth-service` — nếu `episode-service` cần thông tin tương tự, nên cân nhắc chuẩn hóa lại cách forward credential thay vì copy pattern này thêm lần nữa.
- Mọi nguồn trong `Mono.zip` PHẢI có `onError` fallback riêng — không để 1 nguồn lỗi làm sập toàn bộ `/api/user/stats`.