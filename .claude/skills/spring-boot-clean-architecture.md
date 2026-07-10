---
name: spring-boot-clean-architecture
description: Dùng khi tạo service Spring Boot mới (ai-search-service, episode-service) hoặc khi thêm feature vào service cũ, để đảm bảo đúng package structure và tách lớp Controller/Service/Repository/DTO/Entity/Exception theo chuẩn dự án AnimeFlix.
---

# Spring Boot Clean Architecture — AnimeFlix

## Khi nào dùng
- Tạo service mới từ đầu.
- Thêm 1 feature (VD: "thêm endpoint report") vào service đã có.
- Review xem code mới có đặt sai layer không.

## Quy tắc cứng
1. Copy nguyên `PROJECT_ARCHITECTURE_TEMPLATE.md` làm khung package trước khi viết bất kỳ file nào.
2. Thứ tự viết code khi thêm 1 feature: Entity → Repository → DTO (request/response) → Mapper → Service → Controller. KHÔNG viết Controller trước.
3. Controller method LUÔN theo pattern:
\`\`\`java
@PostMapping
public Mono<ResponseEntity<ApiResponse<XxxResponse>>> doAction(@Valid @RequestBody XxxRequest request, ServerWebExchange exchange) {
    return SecurityContextUtil.getCurrentUserId(exchange)
            .flatMap(userId -> xxxService.doAction(userId, request))
            .map(response -> ResponseEntity.ok(ApiResponse.success(response)));
}
\`\`\`
4. Mapper LUÔN dùng MapStruct interface `@Mapper(componentModel = "spring")`, không viết converter thủ công trừ khi mapping có logic điều kiện phức tạp (dùng `default` method trong mapper như `AnimeMapper.mapRelations`).
5. Mỗi exception nghiệp vụ là 1 class riêng extends RuntimeException, đăng ký handler tương ứng trong `GlobalExceptionHandler` — không dùng generic `Exception` cho lỗi nghiệp vụ có thể lường trước.
6. Toàn bộ code phải tuân CLAUDE.md mục Code Style (comment 1 dòng, tên hàm 1 dòng).

## Checklist trước khi coi 1 feature hoàn thành
- [ ] Entity có đủ index cần thiết
- [ ] Repository chỉ chứa method Spring Data hoặc @Aggregation, không viết logic
- [ ] Response DTO có @JsonInclude NON_NULL
- [ ] Controller không chứa if/else nghiệp vụ
- [ ] Đã chạy `subagent-security` nếu đụng tới auth
- [ ] Đã chạy `subagent-database` nếu thêm entity/query mới