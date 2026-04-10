# 05. 외부 API 호출 병목 — Timeout과 Circuit Breaker

---

## 🎯 핵심 질문

외부 API 호출이 느릴 때 응답시간이 왜 이렇게 높아질까?
- 외부 API가 10초 걸리면 → 우리 응답시간도 10초 (더할 수만 있고 뺄 수 없음)
- 하지만 **Timeout 없이 기다리면** → 30초, 60초, 무한 대기 가능
- **Circuit Breaker 없이** 연쇄 장애 발생 → 스레드 풀 고갈, 전체 서비스 다운

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
상황: 사용자 A 요청 → 외부 결제 API 호출 → API 느려짐 (30초)
      사용자 B 요청 → 같은 결제 API 호출 → 30초 대기

스레드 풀: 200개
└─ 결제 API 호출: 150개 스레드 (모두 30초 대기)
└─ 다른 요청: 50개 스레드만 남음

결과: "사이트 먹통" (모든 사용자 영향)

원인:
1. Timeout 없음: 외부 API가 느려지면 우리도 느려짐
2. Circuit Breaker 없음: 이미 느린 API에 계속 요청
3. 재시도 설정: 실패하면 3번 재시도 (×3 시간)

답:
1. Timeout 설정: Connection 10초, Read 5초
2. Circuit Breaker: 연속 5회 실패 → 즉시 차단 (fail fast)
3. Fallback: API 실패 시 캐시된 데이터 반환
```

**올바른 대응:**

```
외부 API 의존성 관리:

요청 1 (정상)           요청 2 (API 느려짐)           요청 3 (API 다운)
  │                        │                           │
  └→ API (50ms)      └→ Timeout 5초            └→ Circuit Open
       결과 반환            자동 차단                     즉시 Fallback
       응답시간 100ms       응답시간 5초                응답시간 100ms
                            (최악을 제한)               (캐시 데이터)
       
설정 없을 때:
       응답시간 600ms+
       (30초 대기 × 여러 요청)
```

---

## 😱 흔한 실수 (Before)

```bash
# 잘못된 접근 1: Timeout 없이 API 호출
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    // Timeout 설정 없음 → 외부 API 느려지면 무한 대기 가능
    ResponseEntity<Order> response = 
        restTemplate.getForEntity("https://api.example.com/order/" + id, Order.class);
    return response.getBody();
}

# 문제:
# - API 서버가 느려지면 → 우리도 느려짐
# - API 서버가 다운되면 → 30~60초 이상 대기 (TCP timeout)
# - 여러 요청이 쌓이면 → 스레드 풀 고갈

# 잘못된 접근 2: 실패하면 무조건 재시도
try {
    return restTemplate.getForEntity(...);
} catch (Exception e) {
    // 재시도 3회 (같은 API가 느려졌는데 또 요청)
    for (int i = 0; i < 3; i++) {
        try {
            return restTemplate.getForEntity(...);
        } catch (Exception ex) {
            // 계속 재시도
        }
    }
}

# 문제:
# - 이미 느린 API에 계속 요청 (더 느려짐)
# - 각 재시도마다 시간 증가
# - 총 시간: 원래 10초 × 4회 = 40초!

# 잘못된 접근 3: Circuit Breaker 없음
// 결제 API가 다운되었는데 계속 요청
@GetMapping("/payment/{id}")
public Payment getPayment(@PathVariable Long id) {
    // 즉시 차단해야 하는데 계속 요청
    // → 스레드 100개가 모두 이 API 대기 중
    // → 다른 요청들은 대기열에만 쌓임
    return paymentService.getPayment(id);  // timeout 없음
}

# 결과: Cascading Failure (연쇄 장애)
# 결제 API 다운 → 결제 관련 요청 모두 느려짐
# → 스레드 풀 고갈 → 주문 조회, 상품 조회도 느려짐
# → 전체 서비스 다운
```

**실제 사례:**

```
상황: 결제 게이트웨이 업체의 API 서버 장애 (1시간)

Before (대응 없음):
- 결제 요청 100개/초
- 각 요청: Timeout 없음 (60초 이상 대기)
- 스레드 풀: 200개 모두 결제 API 대기
- 다른 요청: 차단 (대기열에 1000개 쌓임)
- 사용자 경험: "사이트 먹통" (1시간)
- 피해: 100개/초 × 3600초 = 360,000개 요청 모두 실패

After (Circuit Breaker + Timeout):
- 첫 5개 요청 실패 감지
- Circuit Breaker Open: 즉시 차단
- 나머지 요청: Fallback (캐시된 가격 반환)
- 응답시간: 100ms (그대로 진행)
- 사용자 경험: "결제는 안 되지만 사이트는 동작"
- 피해 최소화: 스레드 풀 정상 → 다른 기능 정상 작동
```

---

## ✨ 올바른 접근 (After)

### 1단계: Timeout 설정 (Connection Timeout vs Read Timeout)

**RestTemplate (Spring Boot):**

```java
// 1. ClientHttpRequestFactory 설정
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        // HTTP 클라이언트 설정
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        
        // Connection Timeout: 연결 수립까지의 시간
        // → API 서버에 TCP 연결 시간
        // → 보통 짧게 (3~10초)
        factory.setConnectTimeout(10 * 1000);  // 10초
        
        // Read Timeout: 데이터 수신까지의 시간
        // → API 서버에서 응답 수신 시간
        // → 실제 API 응답 시간 + 여유 (5~30초)
        factory.setReadTimeout(30 * 1000);  // 30초
        
        return new RestTemplate(factory);
    }
}

// 2. 사용
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    try {
        ResponseEntity<Order> response = restTemplate.getForEntity(
            "https://api.example.com/order/" + id, 
            Order.class
        );
        return response.getBody();
    } catch (ResourceAccessException e) {
        // Timeout 발생
        log.warn("API timeout for order {}", id);
        return getOrderFromCache(id);  // Fallback
    }
}
```

**WebClient (Spring WebFlux, 권장):**

```java
// 1. WebClient 설정 (비동기, 더 효율적)
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            // Connection Timeout
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .responseTimeout(Duration.ofSeconds(30))
                    .connectionTimeout(Duration.ofSeconds(10))
            ))
            .build();
    }
}

// 2. 사용 (비동기)
@GetMapping("/order/{id}")
public Mono<Order> getOrder(@PathVariable Long id) {
    return webClient
        .get()
        .uri("/order/{id}", id)
        .retrieve()
        .bodyToMono(Order.class)
        // Timeout 설정
        .timeout(Duration.ofSeconds(5))
        // Fallback
        .onErrorResume(TimeoutException.class, 
            e -> Mono.just(getOrderFromCache(id)));
}
```

### 2단계: Circuit Breaker (Resilience4j)

**Circuit Breaker 개념:**

```
상태 변화:
CLOSED (정상)
  ↓ (연속 5회 실패)
OPEN (차단)
  ↓ (5초 경과)
HALF_OPEN (시험 모드)
  ↓ (성공하면 CLOSED, 실패하면 OPEN)
CLOSED (정상)

각 상태의 동작:
- CLOSED: 모든 요청 통과
- OPEN: 모든 요청 즉시 차단 (응답: CircuitBreakerOpenException)
- HALF_OPEN: 제한된 수의 요청만 통과 (상태 판단용)
```

**Resilience4j 설정:**

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      order-api:
        # 실패율 계산 윈도우
        sliding-window-size: 100  # 최근 100개 요청
        failure-rate-threshold: 50  # 실패율 50% 이상 → OPEN
        
        # 최소 요청 수 (실패율 계산 최소값)
        minimum-number-of-calls: 5  # 최소 5개 요청 필요
        
        # HALF_OPEN 상태 진입 조건
        wait-duration-in-open-state: 30000  # 30초 후 HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 3  # 시험 요청 3개
        
        # 기타
        automatic-transition-from-open-to-half-open-enabled: true
        
  retry:
    instances:
      order-api:
        max-attempts: 3  # 최대 3회 재시도
        wait-duration: 1000  # 1초 대기 후 재시도
        
  timelimit:
    instances:
      order-api:
        timeout-duration: 5000  # 5초 timeout
```

**Java 설정:**

```java
@Service
public class OrderService {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final TimeLimiter timeLimiter;
    
    public OrderService() {
        CircuitBreakerConfig cbConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .minimumNumberOfCalls(5)
            .build();
        
        this.circuitBreaker = CircuitBreaker.of("order-api", cbConfig);
        
        RetryConfig retryConfig = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(1))
            .build();
        
        this.retry = Retry.of("order-api", retryConfig);
        
        TimeLimiterConfig tlConfig = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(5))
            .build();
        
        this.timeLimiter = TimeLimiter.of("order-api", tlConfig);
    }
    
    // Circuit Breaker + Retry + Timeout 조합
    @GetMapping("/order/{id}")
    public Order getOrder(@PathVariable Long id) {
        Supplier<Order> orderSupplier = 
            Decorators.ofSupplier(() -> fetchOrderFromAPI(id))
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .withTimeLimiter(timeLimiter)
                .withFallback(e -> getOrderFromCache(id))
                .decorate();
        
        return orderSupplier.get();
    }
    
    private Order fetchOrderFromAPI(Long id) {
        // 실제 API 호출
        return restTemplate.getForObject(
            "https://api.example.com/order/" + id,
            Order.class
        );
    }
    
    private Order getOrderFromCache(Long id) {
        // Fallback: 캐시 또는 기본값
        return cache.get(id).orElse(Order.empty());
    }
}
```

### 3단계: Fallback 전략

```java
@Service
public class ExternalAPIService {
    
    private final Cache<Long, Order> orderCache;
    
    // 전략 1: 캐시 반환
    public Order getOrder(Long id) {
        try {
            return apiClient.getOrder(id);
        } catch (CircuitBreakerOpenException e) {
            log.warn("API circuit open, returning cached order");
            return orderCache.get(id)
                .orElse(Order.getDefault());
        }
    }
    
    // 전략 2: 기본값 반환
    public UserProfile getUserProfile(Long userId) {
        try {
            return externalUserAPI.getProfile(userId);
        } catch (Exception e) {
            log.warn("User profile API failed, returning default profile");
            return UserProfile.getDefault(userId);
        }
    }
    
    // 전략 3: 부분 실패 허용 (graceful degradation)
    public OrderDetail getOrderDetail(Long orderId) {
        OrderDetail detail = new OrderDetail();
        
        // 필수: 주문 기본 정보
        try {
            detail.setOrder(apiClient.getOrder(orderId));
        } catch (Exception e) {
            throw new RuntimeException("Critical API failed");
        }
        
        // 선택: 배송 정보 (실패 가능)
        try {
            detail.setShipping(shippingAPI.getShipping(orderId));
        } catch (Exception e) {
            log.warn("Shipping API failed, skipping shipping info");
            detail.setShipping(Shipping.getDefault());  // 기본값
        }
        
        // 선택: 추천 상품 (실패 가능)
        try {
            detail.setRecommendations(recommendationAPI.getRecommendations(orderId));
        } catch (Exception e) {
            log.warn("Recommendation API failed, skipping recommendations");
            detail.setRecommendations(List.of());  // 빈 리스트
        }
        
        return detail;
    }
}
```

---

## 🔬 내부 동작 원리

### Timeout 없을 때의 연쇄 문제

```
외부 API 응답 지연 (30초)
         ↓
요청 1 (Timeout 없음): 30초 대기
요청 2 (Timeout 없음): 30초 대기
...
요청 100 (Timeout 없음): 30초 대기
         ↓
스레드 풀: 200개
├─ 외부 API 호출: 100개 스레드 (모두 30초 대기)
└─ 남은 스레드: 100개 (다른 요청 처리)
         ↓
새 요청이 들어오면: 스레드 부족 (대기열에 쌓임)
         ↓
사용자: "사이트 느림" (모든 요청 느려짐)

결과:
- 외부 API 1곳의 장애 → 전체 서비스 영향
- 응답시간: 100ms (정상) → 30초+ (3000배 악화)
```

### Circuit Breaker의 효과

```
상황: 외부 API 10초 응답 시간

요청 1~5: 실패 (timeout 5초)
Circuit Breaker: 5회 연속 실패 → OPEN

요청 6~100: Circuit Open
├─ 즉시 거부 (응답시간 1ms)
├─ Exception 발생
└─ Fallback 실행 (캐시 반환, 응답시간 50ms)

결과:
- 스레드 풀: 모두 해제 (대기 중인 요청 없음)
- 응답시간: 30초 → 50ms (600배 개선!)
- 사용자: "캐시 데이터 보임 (약간 오래된 정보)"

30초 후:
Circuit Breaker: HALF_OPEN (상태 확인)
요청 101~103: 시험 요청 (API에 보냄)
├─ 성공 → CLOSED (정상 복구)
└─ 실패 → OPEN (계속 차단)
```

### 동시성 문제 시뮬레이션

```
시나리오: 100개 동시 요청, 외부 API 응답 10초

Without Timeout/Circuit Breaker:
Timeline:
T0: 요청 100개 도착
T0-T10: 모든 요청 외부 API 대기
T5: 새로운 요청 50개 도착
T5-T10: 새 요청들도 대기 (스레드 없음, 큐에 쌓임)
T10: API 응답, 처리 시작
T20: 모든 처리 완료
사용자 경험: p50 = 10초, p99 = 20초

With Timeout (5초) + Circuit Breaker:
Timeline:
T0: 요청 100개 도착
T0-T5: 최초 요청들 외부 API 대기
T5: 5개 실패 감지 → Circuit OPEN
T5+: 나머지 95개 요청 즉시 거부 + Fallback (50ms)
T5+: 새로운 요청 50개도 즉시 처리 (50ms)
T30: API 복구 확인 → HALF_OPEN
사용자 경험: p50 = 50ms, p99 = 100ms (200배 개선!)
```

---

## 💻 실전 실험

**시나리오: 외부 결제 API가 느려져 응답시간이 3초 → 30초로 악화**

```bash
#!/bin/bash
# external-api-diagnosis.sh

echo "=== 외부 API 병목 진단 ==="
echo

# 1. 외부 API 응답 시간 측정
echo "1. 외부 API 응답 시간"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
for i in {1..10}; do
    echo "요청 $i:"
    time curl -w "\nTotal: %{time_total}s\n" \
        https://api.example.com/payment/status
done | grep Total

# 2. Circuit Breaker 상태 확인
echo
echo "2. Circuit Breaker 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/health | jq '.components | keys'

curl -s http://localhost:8080/actuator/health/circuitBreakers | \
    jq '.components'

# 3. 스레드 풀 모니터링
echo
echo "3. 스레드 풀 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics/executor.active | \
    jq '.measurements[0].value'

curl -s http://localhost:8080/actuator/metrics/executor.queue | \
    jq '.measurements[0].value'

# 4. 외부 API 호출 메트릭
echo
echo "4. 외부 API 호출 메트릭"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics | \
    grep -E "resilience4j|http.client" | \
    head -20

# 5. 응답 시간 분포 (부하 테스트)
echo
echo "5. 부하 테스트 (100 RPS, 10초)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
# Apache Bench 사용
ab -n 1000 -c 100 http://localhost:8080/api/order

echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== 외부 API 병목 진단 ===

1. 외부 API 응답 시간
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
요청 1: Total: 0.234s
요청 2: Total: 0.245s
요청 3: Total: 10.123s ← 느려짐 시작
요청 4: Total: 9.876s
요청 5: Total: 10.234s
요청 6: Total: timeout (30초)
...

2. Circuit Breaker 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{
  "orderApi": "OPEN",  ← 차단됨
  "paymentApi": "CLOSED"
}

3. 스레드 풀 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Active threads: 178 (200개 중 178개 사용)
Queue size: 156 (대기 중인 요청 156개)

⚠️  WARNING: 스레드 풀 거의 전부 사용 중

4. 외부 API 호출 메트릭
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
resilience4j.circuitbreaker.calls.total: 123
resilience4j.circuitbreaker.calls.failed: 98
resilience4j.circuitbreaker.calls.success: 25
resilience4j.circuitbreaker.calls.buffered: 50

Failure rate: 79.7% (거의 대부분 실패)

5. 부하 테스트 (100 RPS, 10초)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
This is ApacheBench, Version 2.3
Benchmarking localhost:8080 (be patient)...
Completed 100 requests
Completed 200 requests
...

Requests per second:    10.23 [#/sec]  (평상시 100 RPS)
Time per request:      9781.23 [ms]   (평상시 30ms)
Failed requests:       234 (약 23%)

HTML transferred:      1234567 bytes

Percentage of the requests served within a certain time (ms):
  50%   5234
  66%   9876
  75%   10234
  90%   15234
  95%   20123
  99%   30001

=== 진단 결과 ===

주요 문제:
1. 외부 API 응답 시간: 정상 250ms → 10초 이상 (40배 악화)
2. Circuit Breaker: OPEN 상태 (79.7% 실패율)
3. 스레드 풀: 178/200 사용 중, 대기 큐 156개
4. 응답시간: p99 = 30초 (정상 100ms)

원인:
1. 외부 API 서버 장애
2. Timeout 설정 너무 길어서 차단까지 오래 걸림
3. 스레드 풀 고갈로 다른 요청도 영향

해결:
1. Timeout 5초로 단축 (기존 30초)
2. Circuit Breaker 활성화 (즉시 차단)
3. Fallback 실행 (캐시 데이터 반환)
```

---

## 📊 성능 비교

| 설정 | 외부 API 응답 | 응답시간 | 스레드 사용 | 스루풋 | 평가 |
|------|----------|---------|-----------|--------|------|
| **Timeout 없음** | 10초 | 30초 | 200/200 | 10 RPS | 매우 나쁨 |
| **Timeout 30초** | 10초 | 10초 | 150/200 | 50 RPS | 나쁨 |
| **Timeout 5초** | 10초 | 5초 | 100/200 | 100 RPS | 좋음 |
| **Timeout 5초 + CB** | 10초 | 50ms | 10/200 | 990 RPS | 최적 |
| **Timeout 5초 + CB + Fallback** | 10초 | 100ms | 15/200 | 950 RPS | 최적 |

**실제 케이스:**

```
케이스 1: Timeout 없이 외부 API 대기
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  외부 API 응답: 30초
  우리 응답시간: 35초 (30초 + 처리 5초)
  p99: 60초 (TCP timeout)
  
After: Timeout 5초 설정
  외부 API 응답: 30초 (우리는 5초만 기다림)
  우리 응답시간: 6초 + Fallback (캐시)
  p99: 10초
  개선율: p99 83% 개선

케이스 2: Circuit Breaker 없이 연쇄 장애
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  외부 API 장애 (응답 30초 이상)
  우리 스레드: 200개 모두 대기
  다른 요청: 0 처리 (스레드 없음)
  사용자: "사이트 먹통"
  
After: Circuit Breaker 추가
  외부 API 장애 (응답 30초)
  5회 실패 감지 → Circuit OPEN
  이후 요청: 즉시 차단 (Fallback)
  우리 스레드: 10개만 사용
  다른 요청: 190개 스레드로 처리
  사용자: "캐시 데이터 보임" (정상)
  개선율: 스루풋 100배 증가

케이스 3: Graceful Degradation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before: 추천 API 장애
  주문 페이지 로딩 실패 (추천 정보를 못 가져옴)
  사용자: "페이지 로딩 실패"

After: 추천 API는 선택사항으로 설정
  주문 정보: 정상 로드
  배송 정보: 정상 로드
  추천 정보: 실패 → 기본값 또는 빈 리스트
  사용자: "추천 없지만 주문은 가능"
  개선: 기능 부분 서비스 가능
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 선택 기준 |
|------|------|------|---------|
| **Timeout 짧게** | 빨리 차단, 리소스 절약 | Fallback 자주 발생 | 항상 설정 (필수) |
| **Circuit Breaker** | 스레드 보호, 빠른 차단 | 복잡도 증가 | 의존하는 API 많을 때 |
| **Retry** | 일시적 오류 복구 | 느려짐, 부하 증가 | 조건부 (멱등성 필수) |
| **Async/WebClient** | 스레드 절약, 효율적 | 코드 복잡도, 디버깅 어려움 | 높은 동시성 필요 시 |
| **Fallback** | Graceful degradation | 부분적 기능만 제공 | 항상 고려 (필수) |

---

## 📌 핵심 정리

1. **Timeout은 필수**
   - Connection Timeout: 3~10초 (짧을수록 좋음)
   - Read Timeout: API 응답 시간 + 여유 (보통 5~30초)
   - 없으면 외부 API 장애 → 우리 서비스도 다운

2. **Circuit Breaker는 생명보험**
   - 연속 5회 이상 실패 → 즉시 차단
   - 스레드 풀 보호 (스레드 고갈 방지)
   - 다른 요청들도 보호 (연쇄 장애 방지)

3. **Retry는 신중하게**
   - 멱등성 있는 작업에만 (GET, DELETE 등)
   - 몇 회? 보통 2~3회, 간격 1초
   - Circuit Open 상태에서는 재시도 금지

4. **Fallback으로 Graceful Degradation**
   - 캐시 데이터 반환
   - 기본값 사용
   - 부분 기능만 제공
   - 완전 실패보다 부분 성공이 낫다

5. **의사결정 플로우**
   ```
   외부 API 호출
   ├─ Timeout 설정 (필수)
   │  ├─ Connection: 10초
   │  └─ Read: 5~30초
   ├─ Circuit Breaker (권장)
   │  ├─ 5회 실패 → OPEN
   │  ├─ 30초 대기 → HALF_OPEN
   │  └─ 성공 → CLOSED
   ├─ Retry (선택)
   │  ├─ 멱등성 있을 때만
   │  └─ 2~3회, 1초 간격
   └─ Fallback (필수)
      ├─ 캐시 데이터
      ├─ 기본값
      └─ 부분 기능
   ```

---

## 🤔 생각해볼 문제

**Q1: Timeout을 5초로 설정했는데 외부 API가 3초에 응답하면?**

<details>
<summary>해설 보기</summary>

**아무 문제 없다.**

Timeout은 **최대 대기 시간**이다. 3초에 응답하면 3초 후 즉시 반환된다.

```
Timeline:
T0: 요청 시작, Timeout 5초 설정
T0-T3: API 대기 (네트워크 + 처리)
T3: API 응답 수신
T3+: 응답 처리, 반환

응답시간: 3초 (Timeout 5초는 영향 없음)
```

**하지만 다른 시나리오:**

```
시나리오 1: API 응답 느림 (4초)
T0-T4: API 대기
T4: API 응답 (성공)
응답시간: 4초 (정상)

시나리오 2: API 응답 느림 (10초), Timeout 5초
T0-T5: API 대기 (최대)
T5: Timeout 발생
T5+: 예외 처리 + Fallback
응답시간: 5초 (Fallback까지 포함)
실제 API는: T10에 응답했지만 우리는 T5에 포기
```

**결론:** Timeout은 "최악의 시나리오에 대한 보호"이다. 정상 응답은 영향 없고, 지연 응답만 차단한다.

</details>

---

**Q2: Circuit Breaker가 OPEN 상태인데 요청을 보내면?**

<details>
<summary>해설 보기</summary>

**즉시 차단된다 (API에 요청 안 함).**

```java
// Circuit Breaker OPEN 상태
try {
    return circuitBreaker.executeSupplier(() -> callAPI());
} catch (CallNotPermittedException e) {
    // Circuit OPEN: "지금 이 API 못 씀"
    log.warn("Circuit breaker is open");
    return getFallback();  // 즉시 Fallback
}
```

**Timeline:**

```
Circuit Breaker OPEN
요청 1: 즉시 거부 (1ms) → Fallback
요청 2: 즉시 거부 (1ms) → Fallback
요청 3: 즉시 거부 (1ms) → Fallback
... (30초 경과)
Circuit Breaker HALF_OPEN (상태 확인)
요청 4: 시험 요청 (API에 실제 요청)
  ├─ 성공: CLOSED (정상 복구)
  └─ 실패: OPEN (계속 차단)
```

**이점:**

1. **스레드 보호:** API 대기 안 함 (응답시간 1ms)
2. **스루풋 증가:** 스레드를 다른 요청에 사용 가능
3. **자동 복구:** HALF_OPEN에서 주기적으로 확인

**주의:**

Circuit OPEN 중에 외부 API가 복구되어도 즉시 감지 안 함. 30초(wait-duration) 후 HALF_OPEN에서 확인 가능.

</details>

---

**Q3: Retry를 3회로 설정했는데 Circuit Breaker가 OPEN되면?**

<details>
<summary>해설 보기</summary>

**Retry 설정과 무관하게 Circuit OPEN이 우선된다.**

```
설정:
- Retry: 3회
- Circuit Breaker: 5회 실패 후 OPEN

Timeline:
요청 1: 실패 (Circuit breaker 카운트 +1)
요청 2: 실패 (Circuit breaker 카운트 +2)
요청 3: 실패 (Circuit breaker 카운트 +3)
요청 4: 실패 (Circuit breaker 카운트 +4)
요청 5: 실패 (Circuit breaker 카운트 +5) → OPEN

요청 6: Circuit OPEN
├─ 원래는 Retry 3회 예정
├─ 하지만 Circuit OPEN이므로 즉시 차단
├─ Retry 실행 안 됨
└─ CallNotPermittedException 즉시 발생 → Fallback
```

**순서:**

1. **Circuit Breaker 확인** (가장 먼저)
2. **Timeout 설정** (다음)
3. **Retry** (마지막)

**이유:**

Circuit Breaker가 OPEN인데 Retry하면:
- 이미 느린 API에 계속 요청 (더 느려짐)
- 리소스 낭비
- Fallback이 더 빠름

**올바른 설정:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-api:
        failure-rate-threshold: 50
        minimum-number-of-calls: 5  # 최소 5회 이상 필요
        wait-duration-in-open-state: 30000
  
  retry:
    instances:
      payment-api:
        max-attempts: 2  # Circuit 때문에 3회 이상은 의미 없음
        wait-duration: 1000
```

</details>

---

<div align="center">

**[⬅️ 이전: DB 병목 분석 — HikariCP와 Connection Pool](./04-db-bottleneck.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스레드 풀 분석 — Thread Dump로 BLOCKED 원인 특정 ➡️](./06-thread-pool-analysis.md)**

</div>
