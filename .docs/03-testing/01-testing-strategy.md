# Testing Strategy — AnimeFlix

## Nguyên tắc: Ai chạy test này?
Tài liệu này được viết để AI (Claude Code) tự đọc, tự viết test, tự chạy `mvn test` và tự báo cáo kết quả — không yêu cầu người dùng biết viết test.

## Thứ tự ưu tiên (làm lần lượt, không nhảy cóc)

### Ưu tiên 1 — Integration Test theo từng endpoint (thay thế Postman)
- Dùng `WebTestClient` + Testcontainers (MongoDB, Redis thật, không mock).
- Mỗi endpoint trong bảng số thứ tự (`docs/02-backend/<service>/phase-XX.md`) → 1 test case tối thiểu: happy-path + 1 case lỗi.
- Test PHẢI gọi HTTP thật (`webTestClient.get().uri(...)`), KHÔNG gọi thẳng Service — mục đích là bắt lỗi ở toàn bộ chuỗi Controller→Service→Repo→DB thật, giống hệt việc bạn bấm Postman.

### Ưu tiên 2 — Unit Test cho Service có logic phức tạp
Chỉ viết cho các method sau (đã xác định qua các phase doc trước đó là "điểm dễ vỡ"):
- `AnimeSyncService.mergeTitles` — merge title cũ/mới không mất field
- `AnimeService.getFromCacheOrDb` — cache hit/miss/parse-lỗi
- `AnimePaheClient.calculateSimilarity` — Levenshtein matching
- `TokenService.refresh` — revoked/expired/invalid-signature
- `RecommendationService.calculateScore` — tính điểm genre
- `EmbeddingSearchService.smartFilter` (khi ai-search-service code xong)

### Ưu tiên 3 (tùy chọn) — Repository Test cho @Aggregation
Chỉ 3 method: `WatchHistoryRepository.countDistinctAnimeByUserId`, `getTotalWatchedSeconds`, `ScheduleRepository.countByDay`.

## KHÔNG test
- Config class (`WebClientConfig`, `ReactiveRedisConfig`...) — không có logic, test vô nghĩa.
- Getter/setter Lombok, Mapper MapStruct tự sinh (chỉ test `default` method có logic điều kiện, VD `AnimeMapper.mapRelations`).
- DTO validation annotation (`@NotBlank`...) — Spring tự đảm bảo.

## Quy trình AI thực hiện (mỗi lần được giao "viết test cho service X")
1. Đọc `docs/02-backend/X/phase-*.md` để biết đủ endpoint + logic phức tạp.
2. Viết Integration Test trước (Ưu tiên 1) cho TOÀN BỘ endpoint của service đó.
3. Chạy `mvn test`, sửa tới khi pass.
4. Viết Unit Test cho các method liệt kê ở Ưu tiên 2 (nếu service đó có).
5. Cập nhật kết quả vào `docs/03-testing/03-test-coverage-tracker.md` (xem file dưới).
6. KHÔNG tự sửa code nghiệp vụ để "cho test pass" — nếu test fail do phát hiện bug thật, báo cáo riêng, không âm thầm sửa logic.