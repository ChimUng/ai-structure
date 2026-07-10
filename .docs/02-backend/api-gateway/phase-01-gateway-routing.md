# Phase 01 — Gateway Routing

## Mục đích
Điểm vào duy nhất cho client, định tuyến request tới đúng service phía sau.

## Class/file liên quan
`application.yml` (route config), `ApiGatewayApplication`.

## Route hiện có
| Route ID | Predicate | Uri đích | Filter |
|---|---|---|---|
| auth-service | `Path=/api/auth/**` | `service.auth.url` (8085) | `StripPrefix=0` (giữ nguyên path) |
| anime-service | `Path=/anime/**` | `service.anime.url` (8080) | `AuthenticationFilter`, `StripPrefix=1` (`/anime/{id}` → `/{id}`) |

## Ghi chú path mapping
`anime-catalog-service` expose endpoint ở root (`/{id}`, `/popular`...) — Gateway phải `StripPrefix=1` để bỏ `/anime` trước khi forward, khớp đúng `@RequestMapping("/")` của `AnimeController`.

## Thiếu sót đã biết
- CHƯA có route cho `user-service` (port 8081) — client hiện gọi thẳng, bỏ qua Gateway hoàn toàn.
- Khi thêm `episode-service`/`ai-search-service`: thêm route mới theo đúng pattern trên, quyết định có cần `AuthenticationFilter` hay không tùy endpoint có nhạy cảm không.

## Constraint
- Route `/api/auth/**` KHÔNG áp `AuthenticationFilter` — vì auth-service tự lo xác thực nội bộ của chính nó (đăng nhập/đăng ký không thể yêu cầu API key trước).