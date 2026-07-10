# PROJECT_ARCHITECTURE_TEMPLATE.md

Áp dụng cho MỌI service, đặc biệt bắt buộc với `ai-search-service` và `episode-service` khi viết mới từ đầu.

## 1. Package Structure
\`\`\`
com.animeflix.<service-name>/
├── config/
├── controller/
├── dto/{request,response,riêng kafka,websocket tùy service nào có sử dụng}/
├── entity/
├── service/
├── repository/
├── mapper/
├── exception/
└── util/
\`\`\`

## 2. Layer Contract — mỗi layer PHẢI tuân thủ

### 2.1 Controller
- Trả về đúng dạng: `Mono<ResponseEntity<ApiResponse<T>>>`.
- KHÔNG chứa if/else nghiệp vụ — chỉ điều phối: lấy userId → gọi 1 Service method → map response.
- Đọc userId LUÔN qua `SecurityContextUtil.getCurrentUserId(exchange)`.
- Comment theo quy ước 4.2 ở CLAUDE.md (`// N. <chức năng>`).

### 2.2 Service
- Method public LUÔN trả `Mono<T>`/`Flux<T>`, KHÔNG `.block()`.
- Đánh số theo quy ước 4.2 (N, Na, Nb, Common).
- Gọi service khác qua WebClient RIÊNG bean theo `@Qualifier`, có `.timeout()` + `.onErrorResume()`.
- Business exception ném ra là custom RuntimeException, không dùng generic Exception.

### 2.3 Repository
- Interface `extends ReactiveMongoRepository<Entity, String>`.
- CHỈ chứa method-name-query hoặc `@Aggregation`, KHÔNG chứa logic Java.
- Mỗi method có comment ghi rõ API nào dùng.

### 2.4 Entity
- `@Document(collection = "...")`, `@Data @Builder @NoArgsConstructor @AllArgsConstructor`.
- Field dùng làm điều kiện `findBy` đơn → `@Indexed`.
- Field kết hợp (2 field trở lên) dùng chung 1 query → `@CompoundIndex(name = "...", def = "{...}", unique = true/false)`.
- Field cần tự xóa theo thời gian → `@Indexed(expireAfter = "0s")` kiểu `Date`.

### 2.5 DTO
- Response DTO LUÔN có `@JsonInclude(JsonInclude.Include.NON_NULL)`.
- Request DTO validate bằng `jakarta.validation.constraints.*` (`@NotBlank`, `@Min`, `@Max`...).

### 2.6 Mapper
- `@Mapper(componentModel = "spring")`.
- Mapping đơn giản → để MapStruct tự sinh.
- Mapping có điều kiện/null-safe phức tạp → viết `default` method trong cùng interface (tham khảo `AnimeMapper.mapRelations`).

### 2.7 Exception
- 1 class = 1 loại lỗi nghiệp vụ, extends RuntimeException.
- Đăng ký xử lý tập trung ở `GlobalExceptionHandler` (`@RestControllerAdvice`), map đúng HTTP status.
- Copy nguyên `ApiResponse.java` từ `user-service`, không viết lại.

### 2.8 Config
- `SecurityConfig`: permitAll toàn bộ (Gateway đã validate), chỉ enable CORS.
- `WebClientConfig`: mỗi service ngoài cần gọi = 1 `@Bean` riêng, đặt tên rõ (`xxxWebClient`).

### 2.9 Util
- `SecurityContextUtil` copy nguyên bản, không viết parser JWT riêng ở service con.

## 3. Checklist trước khi coi 1 feature/service hoàn thành
- [ ] Toàn bộ Controller/Service/Repository đánh số đúng quy ước 4.2
- [ ] Không có `.block()` ở bất kỳ đâu
- [ ] Entity có đủ index cần thiết (chạy `subagent-database` review)
- [ ] Không đổi field Mongo đã có index của service khác
- [ ] Response DTO có NON_NULL
- [ ] Nếu đụng auth/JWT → chạy `subagent-security` review
- [ ] WebClient gọi chéo service có timeout + fallback an toàn