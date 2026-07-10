# Reactive Testing Guide

## Quy tắc cứng
- KHÔNG BAO GIỜ gọi `.block()` trong test — kể cả để "cho nhanh". Luôn dùng `StepVerifier`.
- Test lỗi: dùng `.verifyError(XxxException.class)` hoặc `.expectErrorMatches(...)`.
- Test Flux nhiều phần tử: `.expectNextCount(n)` hoặc `.recordWith(ArrayList::new).consumeRecordedWith(...)`.
- Test timing (delay, timeout): dùng `StepVerifier.withVirtualTime(...)`, không `Thread.sleep`.

## Mẫu test WebClient mock
\`\`\`java
@Test
void fetchAnimeBasicInfo_shouldFallbackEmpty_onTimeout() {
    when(animeCatalogClient.get()).thenThrow(new RuntimeException("timeout"));
    StepVerifier.create(externalAnimeService.getAnimeBasicInfo("1"))
            .verifyComplete(); // Mono.empty() do onErrorResume
}
\`\`\`