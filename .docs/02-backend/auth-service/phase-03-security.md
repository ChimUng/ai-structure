# Phase 03 — Security (Filter Chain & API Key)

## Mục đích
Bảo vệ endpoint bằng 2 cơ chế song song: JWT cookie/header (cho user thường) và API Key (cho developer/gateway).

## Class liên quan
`SecurityConfig`, `JwtAuthenticationFilter`, `ApiKeyAuthenticationFilter`, `DeveloperService.validateApiKey`, `ForbiddenHandler`, `UnauthorizedEntryPoint`.

## Filter Chain Order
`ApiKeyAuthenticationFilter` và `JwtAuthenticationFilter` đều add trước `SecurityWebFiltersOrder.AUTHENTICATION` — 2 filter chạy độc lập, không phụ thuộc thứ tự lẫn nhau vì check path khác nhau:
- `ApiKeyAuthenticationFilter`: bỏ qua nếu path bắt đầu `/api/auth/internal/`, `/api/auth/dev/register`, `/api/auth/dev/login`, `/api/auth/user/**`, `/actuator` — nghĩa là API key CHỈ áp cho endpoint dev tùy chỉnh khác (hiện paths permitAll ở SecurityConfig phủ hầu hết, nên filter này chủ yếu bảo vệ các route dev key mở rộng trong tương lai).
- `JwtAuthenticationFilter`: đọc token từ header `Authorization: Bearer` ưu tiên, fallback cookie `access_token`. Nếu invalid → cho qua KHÔNG set context (không chặn ở đây, việc chặn do `authorizeExchange` quyết định).

## `authorizeExchange` rules
permitAll: /api/auth/user/signup, /login, /refresh, /logout, /api/auth/dev/register, /dev/login, /api/auth/internal/**
authenticated: /api/auth/user/**  (còn lại, VD /me)
anyExchange: permitAll  (mặc định mở, vì hầu hết đã xử lý ở trên)

## Rate Limit (Developer API Key)
`DeveloperService.checkRateLimit`:
- Redis `INCR` key `rate_limit:{apiKey}`, nếu count == 1 → set TTL 1 giờ.
- Nếu count > `dev.rateLimit` → reject.
- Redis lỗi kết nối → `ApiKeyAuthenticationFilter` bắt riêng, trả `503 REDIS_ERROR` (fail-safe rõ ràng, KHÔNG fail-open cho qua âm thầm).

## Error Handling riêng cho Reactive Security
`ForbiddenHandler` (403) và `UnauthorizedEntryPoint` (401) tự viết response JSON thủ công bằng `bufferFactory().wrap()` — vì `ServerHttpResponse` reactive không dùng được cách viết response kiểu servlet thông thường.

## Constraint
- KHÔNG gộp 2 filter (API key + JWT) thành 1 — chúng bảo vệ 2 nhóm client khác nhau (end-user vs developer/gateway), gộp sẽ làm rối logic bỏ-qua-path.
- Mọi endpoint mới thêm vào `auth-service` PHẢI khai báo tường minh trong `authorizeExchange`, không dựa vào `anyExchange().permitAll()` mặc định cho endpoint nhạy cảm.