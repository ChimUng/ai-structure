---
name: microservice-resilience
description: Dùng khi viết hoặc review bất kỳ lời gọi liên-service (WebClient) trong AnimeFlix — bắt buộc áp Circuit Breaker/Retry/Timeout theo Resilience4j thay vì chỉ .onErrorResume() thủ công như hiện tại.
---

# Microservice Resilience — AnimeFlix

## Vấn đề hiện tại (căn cứ codebase thật)
`ExternalAnimeService`, `RecommendationService`, `UserStatsController` gọi WebClient chéo service chỉ có `.timeout(Duration.ofSeconds(5))` + `.onErrorResume()` — KHÔNG có Circuit Breaker. Nếu `anime-catalog-service` down/chậm kéo dài, `user-service` sẽ liên tục timeout 5s mỗi request, gây nghẽn thread pool reactive.

## Yêu cầu khi thêm/sửa lời gọi liên-service
1. Thêm dependency (cần xác nhận trước khi thêm vào pom.xml):
\`\`\`xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
    <version>2.2.0</version>
</dependency>
\`\`\`
2. Cấu hình Circuit Breaker riêng cho từng WebClient bean quan trọng (VD: `animeCatalogWebClient`), threshold gợi ý: failure-rate 50%, sliding-window 10 request, wait-duration-in-open-state 10s.
3. Pattern bắt buộc khi gọi service khác:
\`\`\`java
private Mono<AnimeBasicInfo> fetchAnimeBasicInfo(String animeId) {
    return animeCatalogClient.get()
            .uri("/{id}", animeId)
            .retrieve()
            .bodyToMono(JsonNode.class)
            .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(e -> Mono.empty())
            .map(this::toBasicInfo);
}
\`\`\`
4. Không áp Circuit Breaker cho lời gọi nội bộ trong CÙNG service (chỉ áp cho WebClient gọi SANG service khác).
5. Mọi fallback PHẢI trả giá trị an toàn đã có sẵn pattern trong code (VD: `Mono.empty()`, danh sách rỗng) — KHÔNG throw lỗi 500 ra client vì 1 service phụ chết.
6. Ghi log khi Circuit Breaker chuyển state (CLOSED→OPEN) ở mức WARN, không spam log mỗi request bị reject.

## Không áp dụng cho
- Kafka producer/consumer (đã có retry riêng qua `ProducerConfig.RETRIES_CONFIG` và Reactor Kafka).
- Redis (dùng `onErrorResume` trả cache-miss là đủ, không cần Circuit Breaker riêng).