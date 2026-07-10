# Test Coverage Tracker

> Bảng này là nguồn sự thật duy nhất về "đã test cái gì, còn thiếu cái gì". Cập nhật MỖI LẦN sau khi chạy `mvn test` xong cho 1 service. Số thứ tự (#) khớp với bảng endpoint ở `docs/02-backend/<service>/`.

## auth-service
| # | Endpoint | Integration Test | Unit Test (Service) | Trạng thái |
|---|---|---|---|---|
| 1 | GET /api/auth/user/me | ✅ Pass | - | Hoàn thành |
| 2 | POST /api/auth/user/signup | ✅ Pass | - | Hoàn thành (Gặp lỗi mapping `userId` - Đã báo cáo) |
| 3 | POST /api/auth/user/login | ✅ Pass | ✅ Pass (TokenService.createSession) | Hoàn thành |
| 4 | POST /api/auth/user/refresh | ✅ Pass | ✅ Pass (TokenService.refresh) | Hoàn thành |
| 5 | POST /api/auth/user/logout | ✅ Pass | - | Hoàn thành |
| 6 | POST /api/auth/dev/register | ✅ Pass | - | Hoàn thành |
| 7 | POST /api/auth/dev/login | ✅ Pass | - | Hoàn thành |
| 8 | GET /api/auth/dev/test-limit | ✅ Pass | - | Hoàn thành |
| 9 | POST /api/auth/internal/validate-key | ✅ Pass | ✅ Pass (checkRateLimit) | Hoàn thành |

## anime-catalog-service
| # | Endpoint | Integration Test | Unit Test (Service) | Trạng thái |
|---|---|---|---|---|
| 1 | GET /{id} | ⏳ | ⏳ (mergeTitles, getFromCacheOrDb) | Chưa làm |
| 2-9 | (list/search) | ⏳ | - | Chưa làm |
| 10 | GET /internal/embed-data | ⏳ | - | Chưa làm |

## user-service
| # | Endpoint | Integration Test | Unit Test (Service) | Trạng thái |
|---|---|---|---|---|
| ... | (điền theo phase-01 → 04) | | | |

## Ghi chú trạng thái dùng chung
- ⏳ Chưa làm
- 🔄 Đang làm (test viết rồi nhưng chưa pass hết)
- ✅ Pass, đã merge
- ❌ Fail — có bug thật cần fix (ghi rõ mô tả bug ở cột riêng nếu có)

## Coverage tổng (đo bằng JaCoCo — chạy `mvn test jacoco:report`, xem `target/site/jacoco/index.html`)
| Service | Line Coverage | Ngày đo gần nhất |
|---|---|---|
| auth-service | -% | - |
| anime-catalog-service | -% | - |
| user-service | -% | - |
| api-gateway | -% | - |