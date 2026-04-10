# 05. Chaos Engineering 기초 — 의도적 장애 주입

---

## 🎯 핵심 질문

- 장애가 발생했을 때 애플리케이션이 정말 복구될까?
- Circuit Breaker는 실제로 효과가 있을까?
- 장애를 먼저 발견하는 최선의 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대규모 시스템의 장애는 예상 밖입니다:

```
▼ 기존 방식: 테스트만으로는 불충분
├─ 단위 테스트: 개별 함수만 검증
├─ 통합 테스트: 정상 경로만 검증
├─ 부하 테스트: 처리량만 검증
└─ 문제: 장애 상황은 테스트하지 않음!

▼ 실제 운영 환경의 장애 (예상 밖)
├─ "API 응답이 500ms에서 5초로 증가"
├─ "데이터베이스 연결 풀 고갈"
├─ "메모리 누수로 Heap 부족"
├─ "한 마이크로서비스의 느림 → 전체 시스템 마비"
└─ 결과: 고객 영향 → 회사 손실
```

**따라서 프로덕션과 동일한 환경에서 의도적으로 장애를 주입하고 복구를 검증해야 합니다.**

---

## 😱 흔한 실수 (Before)

### 패턴 1: 장애 테스트 없음

```java
// ❌ 문제: Happy path만 테스트
@Test
public void testOrderCreation() {
    // 정상 케이스만 검증
    Order order = orderService.createOrder(req);
    
    assertNotNull(order);
    assertEquals(OrderStatus.CONFIRMED, order.getStatus());
}

// Payment Service 장애 시나리오? → 테스트 안 함!
// User Service 느려짐? → 테스트 안 함!
```

**문제점:**
- 장애 상황을 미리 알 수 없음
- 프로덕션에서 처음 발생 → 고객에게 영향

### 패턴 2: Circuit Breaker 설정만 하고 검증 안 함

```yaml
# ❌ 문제: 설정만 있고 동작 확인 없음
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 5000
        
# 실제로 작동하는가? 모름!
```

**문제점:**
- 설정 값이 실제 환경에 적합한가 불명확
- 장애 감지 시간은 괜찮은가?
- 복구 시간은?

---

## ✨ 올바른 접근 (After)

### 1단계: Chaos Monkey 설정 (Spring Boot)

#### 의존성 추가

```xml
<!-- pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>2.7.1</version>
</dependency>
```

#### 설정

```yaml
# application.yml
chaos:
  monkey:
    enabled: true
    # 초 단위로 활성화 일정 설정 (선택)
    # chaos 활성화 예: 10초~60초
    
    # Chaos Monkey 공격 설정
    assaults:
      # 지연 (Latency Attack)
      latency:
        enabled: true
        level: 3  # 범위 1-5
        latencyRangeStart: 1000  # ms
        latencyRangeEnd: 5000
        latencyActive: true
      
      # 예외 발생 (Exception Attack)
      exceptions:
        enabled: true
        level: 3
        exceptionActive: true
      
      # 메모리 압박 (Memory Pressure)
      memory:
        enabled: true
        level: 3
      
      # 애플리케이션 종료 (KillApplication)
      killApplication:
        enabled: false  # 신중하게!
        level: 5
    
    # 공격할 메서드 지정
    pointcuts:
      - "execution(public * com.example.service.OrderService.*(..))"
      - "execution(public * com.example.service.PaymentService.*(..))"

spring:
  profiles:
    active: chaos
```

### 2단계: Resilience4j Circuit Breaker 설정

```yaml
# application.yml (이전 설정과 병합)
resilience4j:
  circuitbreaker:
    instances:
      # Order → User Service 호출
      userService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        slidingWindowType: COUNT_BASED
        failureRateThreshold: 50  # 50% 실패율 시 OPEN
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2000  # 2초 이상 = 느린 호출
        waitDurationInOpenState: 10000   # 10초 대기 후 HALF_OPEN
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
      
      # Order → Payment Service 호출
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
  
  retry:
    instances:
      userService:
        maxAttempts: 3
        waitDuration: 1000
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
  
  timelimit:
    instances:
      userService:
        timeoutDuration: 3000  # 3초 타임아웃
        cancelRunningFuture: true
```

### 3단계: Circuit Breaker 적용

```java
// OrderService.java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimit.annotation.TimeLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {
    
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @Retry(name = "userService")
    @TimeLimiter(name = "userService")
    public User getUser(Long userId) {
        log.info("Fetching user {}", userId);
        return userService.getUser(userId);
    }
    
    // Circuit Breaker가 OPEN 상태일 때 호출
    public User getUserFallback(Long userId, Exception e) {
        log.warn("getUserFallback triggered for user {}: {}", userId, e.getMessage());
        // 대체 응답: 캐시된 데이터 또는 기본값
        return new User(userId, "Unknown User", "CACHED");
    }
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "chargeFallback")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public Payment charge(Long orderId, BigDecimal amount) {
        log.info("Charging {} for order {}", amount, orderId);
        return paymentService.charge(orderId, amount);
    }
    
    public Payment chargeFallback(Long orderId, BigDecimal amount, Exception e) {
        log.warn("chargeFallback triggered for order {}: {}", orderId, e.getMessage());
        // 결제 보류: 나중에 재시도
        return new Payment(orderId, amount, PaymentStatus.PENDING);
    }
    
    @Transactional
    public Order createOrder(OrderRequest req) {
        // User 조회 (Circuit Breaker 포함)
        User user = getUser(req.userId);
        
        // 결제 처리 (Circuit Breaker 포함)
        Payment payment = charge(req.orderId, req.amount);
        
        return new Order(user, payment);
    }
}
```

### 4단계: Actuator 엔드포인트로 상태 모니터링

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,resilience4j
  endpoint:
    health:
      show-details: always
```

모니터링 API:

```bash
# Circuit Breaker 상태 확인
curl http://localhost:8080/actuator/circuitbreakers
# {
#   "circuitBreakers": [
#     {
#       "name": "userService",
#       "state": "CLOSED",  # CLOSED | OPEN | HALF_OPEN
#       "failureRate": 0.0,
#       "slowCallRate": 0.0,
#       "numberOfBufferedCalls": 5,
#       "numberOfFailedCalls": 0,
#       "numberOfSlowCalls": 0
#     }
#   ]
# }

# 상세 메트릭
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.calls
# {
#   "name": "resilience4j.circuitbreaker.calls",
#   "measurements": [
#     {"statistic": "COUNT", "value": 100},  # 총 호출 수
#     {"statistic": "TOTAL_TIME", "value": 50000}  # 총 시간 (ms)
#   ]
# }
```

---

## 🔬 내부 동작 원리

### Circuit Breaker 상태 변화

```
┌─────────────────────────────────────────────────────────┐
│                    Circuit Breaker 상태 머신             │
└─────────────────────────────────────────────────────────┘

┌──────────────────────────────────────┐
│   CLOSED (정상, 요청 통과)           │
│                                      │
│ 실패율 < 50% ⟹ 계속 CLOSED          │
│ 실패율 ≥ 50% ⟹ OPEN으로 전환         │
│                                      │
│ 상태 추적 window: 최근 10개 호출      │
└──────────────────────────────────────┘
          ▲                    │
          │ (복구 대기 완료)   │ (장애 감지)
          │                   ▼
┌──────────────────────────────────────┐
│ HALF_OPEN (복구 시도)                │
│                                      │
│ 3개 호출 시도 (permittedCalls=3)    │
│ 성공 ⟹ CLOSED로 복구               │
│ 실패 ⟹ OPEN으로 돌아감              │
│                                      │
│ 대기 시간: 10초                      │
└──────────────────────────────────────┘
          ▲                    │
          │ (성공)            │ (실패)
          │                   ▼
┌──────────────────────────────────────┐
│   OPEN (차단, 즉시 실패)             │
│                                      │
│ 모든 요청 → Fallback 응답           │
│ 대기 시간 경과 후 HALF_OPEN으로 전환 │
│                                      │
│ 대기 기간: 10초                      │
└──────────────────────────────────────┘
```

### Chaos Monkey의 공격 메커니즘

```
┌─────────────────────────────────────┐
│  OrderService.createOrder() 호출    │
└────────────┬────────────────────────┘
             │
             ▼ (AOP Interceptor)
┌─────────────────────────────────────┐
│      Chaos Monkey 검사               │
├─────────────────────────────────────┤
│ ① enabled=true인가? ✓              │
│ ② pointcut 매칭? ✓                │
│ ③ 활성화 기간인가? ✓              │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│      공격 유형 선택                  │
├─────────────────────────────────────┤
│ • Latency: sleep(2500ms)            │
│ • Exception: throw new Exception() │
│ • Memory: allocate 50MB             │
│ • KillApplication: System.exit(1)  │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  실제 메서드 실행                    │
│  (공격 후 또는 공격 중)             │
└────────────┬────────────────────────┘
             │
             ▼
    결과: 정상 | 지연 | 실패
```

---

## 💻 실전 실험

### 실험: Circuit Breaker의 동작 검증

#### 1. 정상 상태 테스트 (CLOSED)

```bash
# 애플리케이션 시작
java -jar app.jar --spring.profiles.active=chaos

# 정상 요청 10개 전송
for i in {1..10}; do
  curl http://localhost:8080/orders \
    -H "Content-Type: application/json" \
    -d '{"userId": 1, "amount": 100}'
  
  echo "[$i] Response OK"
  sleep 0.5
done

# Circuit Breaker 상태 확인
curl http://localhost:8080/actuator/circuitbreakers | jq '.circuitBreakers[0]'
# {
#   "name": "userService",
#   "state": "CLOSED",
#   "failureRate": 0.0,
#   "numberOfBufferedCalls": 10,
#   "numberOfFailedCalls": 0
# }
```

#### 2. 장애 유입 (OPEN으로 전환)

```bash
# Chaos Monkey 활성화: 80% 요청에 지연 추가
# → User Service 응답 5초로 증가
# → 타임아웃(3초) 초과 → 실패

# 요청 10개 다시 전송
for i in {1..10}; do
  start=$(date +%s%N)
  
  response=$(curl -s -w "\n%{http_code}" http://localhost:8080/orders \
    -H "Content-Type: application/json" \
    -d '{"userId": 1, "amount": 100}' \
    --max-time 6)
  
  end=$(date +%s%N)
  duration=$(( (end - start) / 1000000 ))
  
  echo "[$i] Duration: ${duration}ms, Response: $response"
  sleep 1
done

# Circuit Breaker 상태 확인
curl http://localhost:8080/actuator/circuitbreakers | jq '.circuitBreakers[0]'
# {
#   "name": "userService",
#   "state": "OPEN",  # ← OPEN으로 변경!
#   "failureRate": 80.0,  # ← 80% 실패
#   "numberOfBufferedCalls": 10,
#   "numberOfFailedCalls": 8
# }
```

#### 3. Circuit Breaker 효과 (차단)

```bash
# Circuit Breaker가 OPEN 상태에서 요청 전송
# → 즉시 Fallback 응답 (User Service 호출 안 함)

for i in {1..5}; do
  start=$(date +%s%N)
  
  response=$(curl -s http://localhost:8080/orders \
    -H "Content-Type: application/json" \
    -d '{"userId": 1, "amount": 100}')
  
  end=$(date +%s%N)
  duration=$(( (end - start) / 1000000 ))
  
  echo "[$i] Duration: ${duration}ms, Response: $response"
done

# 출력:
# [1] Duration: 5ms, Response: {"status": "PENDING", "message": "Service unavailable"}
# [2] Duration: 3ms, Response: {"status": "PENDING", "message": "Service unavailable"}
# [3] Duration: 4ms, Response: {"status": "PENDING", "message": "Service unavailable"}
# [4] Duration: 2ms, Response: {"status": "PENDING", "message": "Service unavailable"}
# [5] Duration: 3ms, Response: {"status": "PENDING", "message": "Service unavailable"}

# → 응답 시간이 5000ms에서 5ms로 개선!
# (User Service에 다시 요청하지 않음 = 빠른 실패)
```

#### 4. 복구 (HALF_OPEN → CLOSED)

```bash
# 10초 대기 (waitDurationInOpenState=10000)
echo "Waiting 10 seconds for HALF_OPEN transition..."
sleep 10

# 3개 요청 전송 (permittedNumberOfCallsInHalfOpenState=3)
# 이 3개가 모두 성공하면 CLOSED로 복구

for i in {1..3}; do
  curl http://localhost:8080/orders \
    -H "Content-Type: application/json" \
    -d '{"userId": 1, "amount": 100}' > /dev/null
  
  echo "[$i] Probe request sent"
  sleep 1
done

# 모두 성공 가정 → CLOSED로 복구
curl http://localhost:8080/actuator/circuitbreakers | jq '.circuitBreakers[0]'
# {
#   "name": "userService",
#   "state": "CLOSED",  # ← 복구됨!
#   "failureRate": 0.0,
#   "numberOfBufferedCalls": 3,
#   "numberOfFailedCalls": 0
# }
```

### 실험: k6으로 Chaos 상황에서의 성능 측정

```javascript
// chaos-load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  vus: 10,
  duration: '3m',
  thresholds: {
    http_req_duration: ['p(95)<1000'],
    http_req_failed: ['rate<0.5'],
  },
};

export default function () {
  group('Order Creation with Chaos', () => {
    const payload = JSON.stringify({
      userId: Math.floor(Math.random() * 100),
      amount: Math.random() * 1000,
    });
    
    const start = new Date();
    const res = http.post('http://order-service:8080/orders', payload, {
      headers: { 'Content-Type': 'application/json' },
      timeout: '5s',
    });
    const duration = new Date() - start;
    
    check(res, {
      'status 200 or 202': (r) => r.status === 200 || r.status === 202,
      'response time < 1s': (r) => r.timings.duration < 1000,
    });
    
    // 상태에 따른 로깅
    if (res.status === 500) {
      console.log(`[ERROR] At ${start}: ${res.body}`);
    }
  });

  sleep(1);
}
```

실행:

```bash
# 터미널 1: 애플리케이션 (Chaos Monkey 활성화)
java -jar app.jar --spring.profiles.active=chaos

# 터미널 2: k6 부하 테스트 (시간대 기록)
k6 run chaos-load-test.js

# 0-30초: 정상 (Circuit Breaker CLOSED)
#   → p95: 100ms, 실패율: 0%

# 30-70초: 장애 (Circuit Breaker OPEN)
#   → p95: 50ms (Fallback 빠름), 실패율: 80%

# 70-180초: 복구 (Circuit Breaker HALF_OPEN → CLOSED)
#   → p95: 100ms, 실패율: 0%
```

---

## 📊 성능 비교

### Circuit Breaker 없음 vs 있음

| 시기 | 상황 | CB 없음 | CB 있음 | 개선 |
|------|------|--------|--------|------|
| **0-30초** | 정상 | p95: 100ms | p95: 100ms | - |
| **30-70초** | User 지연(5s) | p95: 5500ms | p95: 50ms | **-99%** |
| | | 실패율: 0% | 실패율: 80%* | *의도적 |
| | | timeout 대기 | 즉시 fallback | 빠른 실패 |
| **70-180초** | 복구 | p95: 100ms | p95: 100ms | - |

**결론:** Circuit Breaker 활성화 시 장애 상황에서 응답 시간 99% 개선

### Chaos 공격 유형별 효과

| 공격 유형 | 설명 | 감지 시간 | 복구 시간 | 영향도 |
|----------|------|----------|----------|--------|
| **Latency** | 2-5초 지연 추가 | 즉시 | 3-5초 | 높음 |
| **Exception** | 예외 발생 (50%) | 즉시 | 1초 | 매우 높음 |
| **Memory** | 50MB 할당 | 10초 | 1분 | 중간 |
| **KillApp** | 프로세스 종료 | 1초 | 30-60초 | 극대 |

---

## ⚖️ 트레이드오프

### Chaos Engineering의 이점
✅ 장애를 사전에 발견 → 프로덕션 문제 예방  
✅ 복구 절차 검증 → 신뢰도 증가  
✅ 팀의 장애 대응 능력 향상  

### 리스크
❌ 프로덕션 환경에서는 신중해야 함 (일부 고객에게만 영향)  
❌ Chaos 설정이 과도하면 의도하지 않은 실제 장애 발생  
❌ 학습곡선 (설정, 해석)  

### 권장 실행 전략

```
1단계: 개발/테스트 환경에서만
   - Chaos Monkey 활성화
   - 모든 공격 유형 테스트

2단계: 스테이징 환경에서 검증
   - 프로덕션과 동일한 규모
   - 제한된 공격만 실행

3단계: 프로덕션 (선택적)
   - Canary 배포: 1% 트래픽만 영향
   - 비즈니스 시간 외에만 실행
   - 항상 인력 대기 상태
```

---

## 📌 핵심 정리

1. **Chaos Engineering의 목표**: 약점을 먼저 발견하는 것

2. **Chaos Monkey 역할**: AOP로 메서드에 가로채기 → 의도적 장애 주입 (지연, 예외, 메모리)

3. **Circuit Breaker 동작**: CLOSED → OPEN (장애 감지) → HALF_OPEN (복구 시도) → CLOSED (복구)

4. **감지 시간**: 윈도우(최근 10개) 내 실패율 50% 이상 → 즉시 OPEN

5. **복구 메커니즘**: 10초 대기 후 HALF_OPEN → 3개 성공적 호출 후 CLOSED

6. **빠른 실패의 가치**: 지연 대신 즉시 실패 → 다른 시스템에 영향 최소화

---

## 🤔 생각해볼 문제

**Q1. failureRateThreshold=50, slidingWindowSize=10일 때, 정확히 몇 개 실패로 OPEN이 될까?**

<details>
<summary>해설 보기</summary>

**계산:**

```
failureRateThreshold: 50%
slidingWindowSize: 10

실패율 계산:
실패율 = (실패한 호출 수) / (전체 호출 수) × 100

OPEN 조건: 실패율 ≥ 50%

최소 실패 호출:
50% = 5 / 10 × 100
→ 10개 호출 중 5개 이상 실패 → OPEN
```

**예시 시나리오:**

```
호출 1: 성공 ✓ (실패율: 0/1 = 0%)
호출 2: 성공 ✓ (실패율: 0/2 = 0%)
호출 3: 성공 ✓ (실패율: 0/3 = 0%)
호출 4: 실패 ✗ (실패율: 1/4 = 25%)
호출 5: 실패 ✗ (실패율: 2/5 = 40%)
호출 6: 실패 ✗ (실패율: 3/6 = 50%) ← OPEN!
```

**주의:** 정확히 몇 개부터가 아니라, "비율"로 판단

</details>

---

**Q2. Circuit Breaker가 HALF_OPEN 상태에서 3개 중 2개 성공, 1개 실패하면?**

<details>
<summary>해설 보기</summary>

**동작:**

```yaml
permittedNumberOfCallsInHalfOpenState: 3
failureRateThreshold: 50
```

**상황:**
- 호출 1: 성공 ✓
- 호출 2: 성공 ✓
- 호출 3: 실패 ✗

**분석:**
실패율 = 1 / 3 = 33.3% (50% 미만)

→ **CLOSED로 복구됨!**

**이유:**
- 설정: 실패율이 50% 이상이면 OPEN으로 돌아가기
- 실제: 33.3% → 50% 미만 → 복구 성공으로 간주

**결과:**
```
이 시점부터:
- 새로운 호출들이 CLOSED 상태로 정상 처리
- 1개 실패했지만, 3개 중 2개 성공 = 복구됨
```

**주의:** 
만약 failureRateThreshold=30이었다면?
→ 33.3% > 30% → OPEN으로 돌아감

</details>

---

**Q3. Chaos Monkey의 latency 공격에서 latencyRangeStart=1000, latencyRangeEnd=5000일 때, 모든 요청이 정확히 1000ms 지연될까?**

<details>
<summary>해설 보기</summary>

**답:** 아니다. 범위 내 무작위 지연

**동작:**

```java
// Chaos Monkey의 지연 구현
Random random = new Random();
long delay = random.nextLong(
    latencyRangeEnd - latencyRangeStart
) + latencyRangeStart;

// 예시:
// random.nextLong(4000) → 0-3999 중 하나
// + 1000 → 1000-4999
```

**실제 지연 분포:**

```
요청 1: 1000ms + 2341ms = 3341ms
요청 2: 1000ms + 127ms = 1127ms
요청 3: 1000ms + 3999ms = 4999ms
요청 4: 1000ms + 0ms = 1000ms
요청 5: 1000ms + 1500ms = 2500ms
...

평균: ~3000ms (1000 + (0 + 4000) / 2)
```

**검증:**

```bash
# 요청 10개 보내고 지연 측정
for i in {1..10}; do
  time curl http://localhost:8080/...
done

# 결과:
# real    0m3.341s
# real    0m1.127s
# real    0m4.999s
# real    0m1.000s
# ...
```

**설정 팁:**

명확한 지연을 원한다면:
```yaml
latencyRangeStart: 5000
latencyRangeEnd: 5001  # ← 거의 같은 값
# → 거의 5000ms 고정 지연
```

</details>

---

<div align="center">

**[⬅️ 이전: 컨테이너 환경 성능 — JVM의 컨테이너 메모리 인식 문제](./04-container-performance.md)** | **[홈으로 🏠](../README.md)**

</div>
