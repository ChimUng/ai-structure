# Phase 01 — Reactive Core

## Mục đích
Nền tảng phản ứng (WebFlux) và cấu trúc dữ liệu người dùng/session/developer làm nền cho 2 phase sau (token rotation, security).

## Class liên quan
`UserRepository`, `SessionRepository`, `DeveloperRepository`, entity `User`, `Session`, `Developer`, `ReactiveRedisConfig`, `UserService.signup/login/findById`.

## Đặc điểm reactive
- Toàn bộ dùng `ReactiveMongoRepository` — không method nào block.
- `UserService.signup`: check trùng email bằng `findByEmail().flatMap(error).switchIfEmpty(defer(save))` — pattern chuẩn để tránh race-condition đơn giản (dù chưa transactional thật sự ở tầng Mongo).
- Avatar mặc định sinh tự động từ email qua `pravatar.cc` (URL-encode email) khi signup — không cần user upload.

## MongoDB indexing
- `User.email`: `@Indexed(unique = true)`.
- `Session.refreshToken`: `@Indexed(unique = true)` — dùng để tìm nhanh session khi refresh token.
- `Developer.appId`: `@Indexed(unique = true)`.

## Redis
- `ReactiveRedisConfig` dùng `StringRedisSerializer` cho cả key/value, đánh dấu `@Primary` (khác với `anime-catalog-service` dùng `StringRedisTemplate` blocking — auth-service dùng Reactive đúng chuẩn).
- Redis ở đây phục vụ RIÊNG cho rate-limit của Developer API key (xem Phase 03), không cache user data.

## Constraint
- `User.setIsActive(boolean)` hiện là method RỖNG (bug có chủ đích hay lỗi thật? — cần confirm trước khi dùng field `isActive` cho logic nghiệp vụ mới, đừng giả định nó hoạt động).