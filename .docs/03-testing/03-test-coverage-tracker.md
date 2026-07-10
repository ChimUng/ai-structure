# Test Coverage Tracker

> Bảng này là nguồn sự thật duy nhất về "đã test cái gì, còn thiếu cái gì". Cập nhật MỖI LẦN sau khi chạy `mvn test` xong cho 1 service. Số thứ tự (#) khớp với bảng endpoint ở `docs/02-backend/<service>/`.

## auth-service
| # | Endpoint | Integration Test | Unit Test (Service) | Trạng thái |
|---|---|---|---|---|
| 1 | POST /signup | ⏳ | - | Chưa làm |
| 2 | POST /login | ⏳ | ⏳ (TokenService.createSession) | Chưa làm |
| 3 | POST /refresh | ⏳ | ⏳ (TokenService.refresh) | Chưa làm |
| 4 | POST /logout | ⏳ | - | Chưa làm |
| 5 | POST /dev/register | ⏳ | - | Chưa làm |
| 6 | POST /dev/login | ⏳ | - | Chưa làm |
| 7 | POST /internal/validate-key | ⏳ | ⏳ (checkRateLimit) | Chưa làm |

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