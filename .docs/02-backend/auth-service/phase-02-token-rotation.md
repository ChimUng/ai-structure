# Phase 02 — Token Rotation (JWT Access/Refresh)

## Mục đích
Quản lý vòng đời JWT: cấp mới, xoay vòng (refresh), thu hồi (revoke).

## Class liên quan
`JwtUtil`, `TokenService`, entity `Session`.

## Cơ chế 2-key riêng biệt
- `accessKey` và `refreshKey` là 2 `SecretKey` HOÀN TOÀN KHÁC NHAU, decode từ 2 biến env base64 riêng (`jwt.access-secret`, `jwt.refresh-secret`).
- Cả 2 PHẢI >= 256 bit sau decode, nếu không `JwtUtil` throw `IllegalArgumentException` ngay lúc khởi tạo bean (fail-fast).
- Access token claim: `username`, `email`, `type=access`. Refresh token claim: chỉ `type=refresh` (tối giản, giảm rủi ro lộ thông tin nếu refresh token bị đánh cắp).

## Luồng `createSession` (login)
1. Sinh access token (exp 1h) + refresh token (exp 30 ngày).
2. Lưu `Session` vào Mongo (accessToken, refreshToken, expiresAt=+30 ngày, device, ip).
3. Trả `LoginResponse` — nhưng Controller (`UserController.login`) KHÔNG trả token trong body, chỉ set cookie (xem Phase 03).

## Luồng `refresh`
1. Tìm `Session` theo `refreshToken`.
2. Check tuần tự: `isRevoked` → lỗi nếu true; `expiresAt` đã qua → lỗi; verify signature bằng `jwtUtil.isValid(refreshToken, true)` → lỗi nếu sai.
3. Nếu pass cả 3: cấp access token MỚI, nhưng GIỮ NGUYÊN refresh token cũ (không rotate refresh token mỗi lần — refresh token chỉ đổi khi login lại).
4. Update `lastUsedAt` của session.

## Luồng `revokeRefreshToken` (logout)
Set `isRevoked=true` trên Session tương ứng, KHÔNG xóa record (giữ lại cho audit).

## Constraint
- KHÔNG rotate refresh token ở mỗi lần gọi `/refresh` — đây là quyết định thiết kế hiện tại (đơn giản hơn, đổi lại giảm bảo mật so với refresh-token-rotation chuẩn). Nếu muốn đổi sang rotate-on-use, phải cân nhắc kỹ và cập nhật file này.
- `getJti()` tồn tại trong `JwtUtil` nhưng KHÔNG thấy nơi nào dùng để blacklist — có thể là chỗ để mở rộng sau (JTI-based revocation danh sách đen), không phải bug thiếu code.