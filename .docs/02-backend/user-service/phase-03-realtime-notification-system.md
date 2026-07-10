# Phase 03 — Realtime Notification System

## Mục đích
Nhận sự kiện "tập mới" từ Kafka (do `anime-catalog-service` phát), lọc đúng người cần thông báo, lưu + đẩy realtime qua WebSocket.

## Class liên quan
`EpisodeEventConsumer`, `NotificationService/Controller`, `NotificationWebSocketHandler`, `WebSocketNotificationService`, `ReactiveWebSocketConfig`, entity `Notification`.

## Luồng xử lý event (`EpisodeEventConsumer.handleNewEpisode`)
1. Nhận `NewEpisodeEvent` từ topic `anime.episode.new` (manual ack).
2. `favoriteService.getFavoritesToNotify(animeId)` — lấy toàn bộ Favorite có `notifyNewEpisode=true` cho anime đó.
3. Với mỗi user: check `UserPreferenceService.isNotificationEnabled(userId)` (Phase 04) — chỉ tạo notification nếu preference bật.
4. Tạo `Notification` + đẩy WebSocket nếu user đang online.
5. LUÔN `acknowledge()` ở `doFinally` dù thành công hay lỗi — chấp nhận rủi ro mất 1 event lỗi hiếm gặp, đổi lại tránh partition bị nghẽn do retry vô hạn.

## Chống trùng lặp Notification
`createNotification` check `existsByUserIdAndAnimeIdAndEpisodeNumberAndCreatedAtAfter(now - 1h)` — nếu đã có notification tương tự trong 1 giờ gần đây → skip im lặng (không lỗi).

## WebSocket — giới hạn quan trọng
- `NotificationWebSocketHandler` giữ session vật lý trong `ConcurrentHashMap` LOCAL của từng instance — KHÔNG share được giữa nhiều pod/instance khi scale ngang.
- Trạng thái "online/offline" lưu ở Redis Set (`online:users`) — trạng thái này ĐÚNG toàn cụm, nhưng không đảm bảo session vật lý nằm ở instance đang check.
- Hệ quả: nếu user connect vào instance A nhưng notification được xử lý bởi instance B, `sendToUser` ở B sẽ không tìm thấy session (vì local map của B không có) → notification chỉ lưu Mongo, không đẩy realtime được. Đây là GIỚI HẠN ĐÃ BIẾT của kiến trúc hiện tại (chưa dùng Redis Pub/Sub để relay giữa instance).

## TTL Notification
`expiresAt = now + 30 ngày`, dùng `@Indexed(expireAfter = "0s")` tự xóa, không cần cron riêng.

## Constraint
- KHÔNG tự ý sửa `EpisodeEventConsumer` sang auto-ack — manual ack + always-acknowledge trong `doFinally` là quyết định có chủ đích.
- Nếu scale `user-service` nhiều instance, PHẢI giải quyết giới hạn WebSocket ở trên trước (Redis Pub/Sub hoặc sticky session) — ghi rõ trong `docs/04-deployment/` khi triển khai multi-instance.