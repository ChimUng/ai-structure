# Phase 02 — Authentication Filter

## Mục đích
Chặn request trái phép trước khi vào các service cần bảo vệ, bằng cách validate `X-API-KEY` qua `auth-service`.

## Class liên quan
`AuthenticationFilter` (extends `AbstractGatewayFilterFactory`), `WebClientConfig` (bean `authServiceWebClient`).

## Luồng xử lý
1. Bỏ qua hoàn toàn nếu path bắt đầu `/api/auth/` hoặc chứa `/actuator`.
2. Đọc header `X-API-KEY` — thiếu hoặc rỗng → trả `401` ngay, không gọi tiếp.
3. Gọi `POST /api/auth/internal/validate-key` (kèm header key) sang `auth-service`.
4. Thành công → `chain.filter(exchange)` cho qua.
5. Lỗi → parse message: nếu chứa `"Rate limit"` → trả `429`, còn lại → `401`.

## Constraint
- Filter này CHỈ validate sự tồn tại/hợp lệ của API key — KHÔNG tự parse hay verify JWT user (đó là việc của `auth-service` nếu route đích cần biết userId, hiện `anime-catalog-service` không cần userId nên không có vấn đề).
- Nếu sau này route thêm cho `user-service` (cần `X-User-Id`) qua Gateway: phải mở rộng filter này để set thêm header đó sau khi validate, KHÔNG chỉ dừng ở việc cho qua như hiện tại.