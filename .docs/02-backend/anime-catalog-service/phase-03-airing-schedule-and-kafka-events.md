# Phase 03 — Lịch chiếu & Kiến trúc hướng sự kiện Kafka

## Mục đích
Phát hiện tập mới sắp/đang phát sóng và bắn sự kiện realtime sang `user-service` để tạo notification cho người dùng đã follow anime đó.

## Class liên quan
`EpisodeScheduler`, `EpisodeEventPublisher`, `KafkaProducerConfig`, DTO `NewEpisodeEvent`.

## Luồng hoạt động
1. `EpisodeScheduler.checkNewEpisodes()` chạy cron `0 */15 * * * ?` (mỗi 15 phút).
2. Query `ScheduleRepository` các schedule có `airingAt` trong khoảng `[now - 1h, now + 2h]` — cửa sổ lệch để không bỏ sót do độ trễ giữa các lần chạy cron.
3. Với mỗi schedule: check Redis key `published:{animeId}:{episode}` — nếu đã tồn tại (TTL 1 ngày) → skip, tránh publish trùng.
4. Nếu chưa publish → gọi `EpisodeEventPublisher.publishNewEpisode(...)` → set Redis key đánh dấu đã publish.

## Kafka Producer Config
- `acks=all`, `retries=3`, `max.in.flight.requests.per.connection=1` (đảm bảo thứ tự, chấp nhận giảm throughput).
- `compression.type=snappy`, `linger.ms=10` (batch nhẹ).
- Key = `animeId` → đảm bảo mọi event của cùng 1 anime luôn vào cùng 1 partition, giữ thứ tự tập.

## Schema `NewEpisodeEvent`
```java
eventId, animeId, animeTitle, episodeNumber, airingAt, coverImage, bannerImage, timestamp, source
```
Field `source` hiện luôn là `"anilist"` — chỗ để mở rộng nếu sau này có nguồn khác.

## Điểm nối sang `user-service`
`user-service` consume topic này ở `EpisodeEventConsumer` — chi tiết xử lý phía nhận nằm ở `docs/02-backend/user-service/phase-03-realtime-notification-system.md`, KHÔNG lặp lại ở đây.

## Constraint
- KHÔNG publish trực tiếp trong `AnimeSyncService` khi sync data — việc phát hiện "tập mới" là trách nhiệm RIÊNG của `EpisodeScheduler`, tách biệt để tránh spam event mỗi lần chạy master sync (vốn có thể update field khác không liên quan đến tập mới).
- Khi thêm `episode-service`: cân nhắc chuyển toàn bộ `EpisodeScheduler` + `EpisodeEventPublisher` sang đó, vì đây thực chất là domain "episode" đang tạm nằm ở catalog.