# 서킷브레이커 (Circuit Breaker)
- 마이크로서비스 간의 통신 중 오류가 발생할 경우 연쇄적인 오류를 방지하기 위해 우회로 설정을 해주는 패턴.
- 서킷브레이커는 정상 상태에서 닫혀있다가, 오류 발생 시 열리며, 일정 시간 후 반개방(half-open) 상태가 됩니다.
- 서킷브레이커 설정 예시
```java
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
    .failureRateThreshold(4) // 서킷브레이커를 열지 결정하는 실패 확률 설정
    .waitDurationInOpenState(Duration.ofMillis(1000)) // Open 상태를 유지하는 시간 설정
    .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED) // 호출 결과를 시간 기반 또는 카운트 기반으로 설정
    .slidingWindowSize(2) // 슬라이딩 창의 크기 설정
    .build();

- 타임리미터 설정 예시

TimeLimiterConfig timeLimiterConfig = TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofSeconds(4)) // 서비스 응답 시간 초과 제한 설정
    .build();

return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .timeLimiterConfig(timeLimiterConfig)
                .circuitBreakerConfig(circuitBreakerConfig)
                .build()
        );
```
- 사용 예시
```java
ordersList = orderServiceClient.getOrders(userId);
CircuitBreaker circuitBreaker = circuitBreakerFactory.create("circuitBreaker1");
CircuitBreaker circuitBreaker2 = circuitBreakerFactory.create("circuitBreaker2");
ordersList = circuitBreaker.run(() -> orderServiceClient.getOrders(userId),
             throwable -> new ArrayList<>());

```