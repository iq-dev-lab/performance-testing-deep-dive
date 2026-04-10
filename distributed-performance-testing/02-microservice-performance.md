# 02. 마이크로서비스 성능 — 서비스 간 호출 체인 병목 특정

---

## 🎯 핵심 질문

- 마이크로서비스 아키텍처에서 어느 서비스가 전체 응답 시간을 지연시키는가?
- 서비스 간 호출 체인에서 지연(Latency)과 손실(Loss)을 추적하려면?
- 한 서비스의 장애가 다른 서비스까지 영향을 미치는 메커니즘은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 환경에서는 사용자 요청이 여러 서비스를 거쳐갑니다:

```
사용자 요청
  ↓
API Gateway (10ms)
  ↓
Auth Service (50ms)
  ↓
User Service (30ms)
  ↓
Order Service (200ms) ← ⚠️ 병목!
  ↓
Payment Service (100ms)
  ↓
응답 (총 390ms)
```

문제는:

1. **어느 서비스가 느린지 알기 어려움**: 애플리케이션 로그만으로는 서비스 간 타이밍 파악 불가능

2. **부분 장애의 확산**: Order Service의 응답 시간이 500ms로 증가하면 사용자 요청 전체가 500ms 더 지연

3. **연쇄 장애(Cascade Failure)**: 한 서비스의 느림 → 다른 서비스의 타임아웃 → 전체 실패

**따라서 분산 추적(Distributed Tracing)으로 각 서비스의 기여도를 정량화해야 합니다.**

---

## 😱 흔한 실수 (Before)

### 패턴 1: 애플리케이션 로그만으로 분석

```java
// ❌ 문제: 각 서비스가 로그를 남기지만 연관성 없음
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest req) {
        System.out.println("[Order] Start: " + System.currentTimeMillis());
        
        // 외부 서비스 호출
        User user = userService.getUser(req.userId);
        Payment payment = paymentService.charge(req.amount);
        
        System.out.println("[Order] End: " + System.currentTimeMillis());
        return new Order(user, payment);
    }
}
```

**문제점:**
- Order 서비스의 로그: "Start 1000ms, End 1500ms = 500ms"
- User 서비스의 로그: "Start 1010ms, End 1100ms = 90ms"
- **로그의 시간이 동기화되지 않음** (서버 시간 차이, 포맷 차이)
- 누가 500ms 중 어디에 시간을 쓰는지 알 수 없음

### 패턴 2: 각 서비스 성능을 독립적으로만 측정

```bash
# ❌ 문제: 각 서비스를 따로 테스트
# 단일 호출로는 빠르지만, 실제 사용 패턴과 다름

# User Service 단독 테스트: p95 = 50ms
k6 run -u 100 user-service-test.js

# Order Service 단독 테스트: p95 = 100ms
k6 run -u 100 order-service-test.js

# Payment Service 단독 테스트: p95 = 60ms
k6 run -u 100 payment-service-test.js

# 그런데 실제 사용자 요청 (Order → User + Payment):
# p95 = 300ms ?? (개별 합 = 210ms)
```

**왜 차이나는가?**
- Order가 User를 호출하는 동안, User는 Database 쿼리
- 동시에 Order가 Payment 호출, Payment는 외부 API 호출
- 네트워크 지연, 컨텍스트 스위칭 등 오버헤드 축적

---

## ✨ 올바른 접근 (After)

### 1단계: OpenTelemetry 설치 및 설정

#### 의존성 추가 (Maven)

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-bom</artifactId>
            <version>1.32.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Boot Starter for OpenTelemetry -->
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
        <version>1.0.0</version>
    </dependency>
    
    <!-- Jaeger Exporter -->
    <dependency>
        <groupId>io.opentelemetry.exporter</groupId>
        <artifactId>opentelemetry-exporter-jaeger-thrift</artifactId>
    </dependency>
</dependencies>
```

#### 설정 (application.yml)

```yaml
# application.yml
spring:
  application:
    name: order-service

otel:
  sdk:
    disabled: false
  
  # Jaeger 내보내기
  exporter:
    jaeger:
      endpoint: http://jaeger:14250
  
  # 샘플링: 100% (개발/테스트)
  traces:
    sampler: always_on
  
  # 측정항목 간격
  metric:
    export:
      interval: 1000
```

### 2단계: 서비스별 Span 추가

```java
// OrderService.java
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;

@Service
public class OrderService {
    
    private final Tracer tracer;
    private final UserService userService;
    private final PaymentService paymentService;
    
    @Autowired
    public OrderService(Tracer tracer, 
                       UserService userService,
                       PaymentService paymentService) {
        this.tracer = tracer;
        this.userService = userService;
        this.paymentService = paymentService;
    }
    
    public Order createOrder(OrderRequest req) {
        // 최상위 Span 생성
        Span orderSpan = tracer.spanBuilder("order.create")
            .setAttribute("user.id", req.userId)
            .setAttribute("amount", req.amount)
            .startSpan();
        
        try (var scope = orderSpan.makeCurrent()) {
            // 1. User 조회 (자식 Span)
            Span userSpan = tracer.spanBuilder("order.getUser")
                .setParent(orderSpan.getSpanContext())
                .startSpan();
            
            User user;
            try (var userScope = userSpan.makeCurrent()) {
                user = userService.getUser(req.userId);
            } finally {
                userSpan.end();
            }
            
            // 2. 결제 처리 (자식 Span)
            Span paymentSpan = tracer.spanBuilder("order.charge")
                .setParent(orderSpan.getSpanContext())
                .startSpan();
            
            Payment payment;
            try (var paymentScope = paymentSpan.makeCurrent()) {
                payment = paymentService.charge(req.amount);
            } finally {
                paymentSpan.end();
            }
            
            return new Order(user, payment);
            
        } finally {
            orderSpan.end();
        }
    }
}

// UserService.java (동일하게 Span 추가)
@Service
public class UserService {
    
    private final Tracer tracer;
    private final UserRepository repository;
    
    @Autowired
    public UserService(Tracer tracer, UserRepository repository) {
        this.tracer = tracer;
        this.repository = repository;
    }
    
    public User getUser(Long userId) {
        Span dbSpan = tracer.spanBuilder("user.db.query")
            .setAttribute("user.id", userId)
            .startSpan();
        
        try (var scope = dbSpan.makeCurrent()) {
            return repository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException());
        } finally {
            dbSpan.end();
        }
    }
}
```

### 3단계: Jaeger Docker Compose 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"  # Thrift compact
      - "16686:16686"    # Jaeger UI
      - "14250:14250"    # gRPC receiver
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    environment:
      OTEL_EXPORTER_JAEGER_ENDPOINT: http://jaeger:14250
    depends_on:
      - jaeger

  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile
    ports:
      - "8082:8080"
    environment:
      OTEL_EXPORTER_JAEGER_ENDPOINT: http://jaeger:14250
    depends_on:
      - jaeger

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile
    ports:
      - "8083:8080"
    environment:
      OTEL_EXPORTER_JAEGER_ENDPOINT: http://jaeger:14250
    depends_on:
      - jaeger
```

실행:
```bash
docker-compose up -d
# Jaeger UI: http://localhost:16686
```

### 4단계: 성능 테스트 및 분석

```javascript
// k6-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  vus: 10,
  duration: '2m',
  thresholds: {
    http_req_duration: ['p(95)<500'],
  },
};

export default function () {
  group('Create Order', () => {
    const payload = JSON.stringify({
      userId: Math.floor(Math.random() * 100),
      amount: Math.random() * 1000,
    });
    
    const res = http.post('http://order-service:8081/orders', payload, {
      headers: { 'Content-Type': 'application/json' },
    });
    
    check(res, {
      'status 201': (r) => r.status === 201,
      'response time < 500ms': (r) => r.timings.duration < 500,
    });
  });

  sleep(1);
}
```

실행:
```bash
k6 run k6-test.js
```

---

## 🔬 내부 동작 원리

### Span 계층 구조

```
Trace ID: 8a7b3d4e-5f6c-9d2a-1e5f-8c3a7b2d9e6f
│
├─ Span: order.create (0ms ~ 300ms, 총 300ms)
│  ├─ 속성: user.id=42, amount=99.99
│  │
│  ├─ Span: order.getUser (10ms ~ 110ms, 100ms)
│  │  ├─ 속성: user.id=42
│  │  ├─ Span: user.db.query (20ms ~ 90ms, 70ms)
│  │  │  └─ 속성: table=users, query_type=SELECT
│  │  └─ 네트워크 오버헤드: 30ms
│  │
│  └─ Span: order.charge (120ms ~ 280ms, 160ms)
│     ├─ 속성: amount=99.99
│     ├─ Span: payment.api.call (130ms ~ 270ms, 140ms)
│     │  └─ 속성: api=stripe, timeout=5s
│     └─ 네트워크 오버헤드: 20ms
```

### 병목 특정 알고리즘

Jaeger는 다음 기준으로 병목을 표시합니다:

1. **절대 시간**: 가장 오래 걸린 Span (order.charge = 160ms)
2. **상대 비율**: 전체 시간 대비 비율 (order.charge = 160/300 = 53%)
3. **차단 시간**: 다른 Span이 끝날 때까지 기다리는 시간

---

## 💻 실전 실험

### 실험 1: 순차 호출 vs 병렬 호출

#### 시나리오 A: 순차 호출 (❌ 느림)

```java
@Service
public class OrderServiceSequential {
    
    public Order createOrder(OrderRequest req) {
        User user = userService.getUser(req.userId);
        // User 응답 대기: 100ms
        
        Payment payment = paymentService.charge(req.amount);
        // Payment 응답 대기: 150ms
        
        return new Order(user, payment);
        // 총 시간: 100 + 150 = 250ms
    }
}
```

#### 시나리오 B: 병렬 호출 (✅ 빠름)

```java
@Service
public class OrderServiceParallel {
    
    public Order createOrder(OrderRequest req) {
        // 2개 호출을 동시 실행
        CompletableFuture<User> userFuture = 
            CompletableFuture.supplyAsync(() -> 
                userService.getUser(req.userId));
        
        CompletableFuture<Payment> paymentFuture = 
            CompletableFuture.supplyAsync(() -> 
                paymentService.charge(req.amount));
        
        // 둘 다 완료 대기
        return CompletableFuture.allOf(userFuture, paymentFuture)
            .thenApply(_ -> new Order(
                userFuture.join(), 
                paymentFuture.join()
            ))
            .join();
        // 총 시간: max(100, 150) = 150ms
    }
}
```

---

## 📊 성능 비교

| 메트릭 | 순차 호출 | 병렬 호출 | 개선도 |
|--------|----------|----------|--------|
| **평균 응답시간** | 248ms | 152ms | -39% |
| **p95 응답시간** | 320ms | 195ms | -39% |
| **p99 응답시간** | 380ms | 240ms | -37% |
| **User 서비스 기여도** | 40% (100/250) | 66% (100/150) | - |
| **Payment 서비스 기여도** | 60% (150/250) | 100% (150/150)* | - |
| **병목 서비스** | Payment (150ms) | Payment (150ms) | - |

*병렬 호출에서 Payment가 User보다 오래 걸리므로 Payment가 전체 시간 결정

---

## ⚖️ 트레이드오프

### 분산 추적(OpenTelemetry) 사용의 이점
✅ 서비스 간 호출 경로를 시각적으로 파악 가능  
✅ 각 서비스의 정확한 기여도 측정  
✅ 병목을 즉시 특정 가능  
✅ 개발 팀과 운영 팀 간 소통 용이  

### 트레이드오프
❌ 오버헤드: Span 생성, 직렬화, 전송 (~1-2% CPU)  
❌ 메모리: Trace 데이터 버퍼링  
❌ 저장소: Jaeger/Zipkin 데이터 증가  
❌ 샘플링 필요 (100% 추적 불가능 in production)  

### 샘플링 전략

```java
// 개발/테스트: 100% 샘플링
sampler: always_on

// 프로덕션 (트래픽 많음): 1% 샘플링
sampler: probability{0.01}

// 오류만: 오류 span에 대해서만 전체 trace 보존
sampler: jaeger_remote
```

---

## 📌 핵심 정리

1. **마이크로서비스 환경의 병목**: 단일 서비스 측정으로는 파악 불가능, 전체 호출 체인 분석 필수

2. **OpenTelemetry의 역할**: 각 서비스의 Span을 추적하여 정확한 기여도 측정

3. **Span 계층 구조**: 부모-자식 span으로 호출 경로를 계층적으로 표현

4. **병렬 호출 최적화**: 순차 호출을 병렬 호출로 변경하여 max(t1, t2) 형태로 개선

5. **Jaeger 시각화**: Trace 분석 UI로 각 Span의 시간, 에러, 속성을 시각적 확인

6. **프로덕션 고려사항**: 샘플링으로 오버헤드 관리 필수

---

## 🤔 생각해볼 문제

**Q1. 한 User Service 호출에 평균 100ms가 걸리는데, Jaeger에서는 95%ile이 95ms로 표시된다. 이는 가능한가?**

<details>
<summary>해설 보기</summary>

**가능한 시나리오:**

1. **캐싱 효과**
   ```java
   @Cacheable(value = "users")
   public User getUser(Long userId) {
       return repository.findById(userId); // DB 쿼리 100ms
   }
   ```
   - 첫 요청: 100ms (DB 쿼리)
   - 캐시 히트 (95%): 5ms (메모리)
   - 평균: (100 × 5% + 5 × 95%) ≈ 9.75ms
   
   아니, 이건 평균 100ms와 안 맞음.

2. **조건부 실행**
   ```java
   public User getUser(Long userId) {
       if (cache.contains(userId)) {
           return cache.get(userId); // 5ms
       }
       return repository.findById(userId); // 100ms
   }
   ```
   - 95%의 요청이 캐시 히트 → 5ms
   - 5%의 요청이 DB 쿼리 → 100ms
   - 95%ile = 5ms ✓
   - 평균 = (100 × 0.05 + 5 × 0.95) = 9.75ms

3. **결과**: 평균 100ms가 아니라 평균 ~10ms
   
   **가능성 재검토:** "평균 100ms"는 틀렸을 가능성 높음. 실제는:
   - 평균 ~10ms
   - p95 ~5ms
   - p99 ~100ms

</details>

---

**Q2. Order Service에서 User Service와 Payment Service를 병렬로 호출할 때, 한 서비스 장애로 전체 요청이 실패하지 않으려면?**

<details>
<summary>해설 보기</summary>

**문제점:**

```java
// ❌ 문제: 한 서비스 실패 → 전체 실패
CompletableFuture<User> userFuture = ...
CompletableFuture<Payment> paymentFuture = ...

CompletableFuture.allOf(userFuture, paymentFuture)
    .thenApply(...)
    .join(); // Payment 실패 시 예외 발생
```

**해결책 1: 타임아웃 추가**

```java
CompletableFuture<User> userFuture = 
    CompletableFuture.supplyAsync(() -> userService.getUser(userId))
        .orTimeout(1, TimeUnit.SECONDS)
        .exceptionally(e -> new User(userId, "Unknown"));

CompletableFuture<Payment> paymentFuture = 
    CompletableFuture.supplyAsync(() -> paymentService.charge(amount))
        .orTimeout(3, TimeUnit.SECONDS)
        .exceptionally(e -> new Payment(amount, "PENDING"));

// 둘 다 완료 또는 실패하면 진행
return CompletableFuture.allOf(userFuture, paymentFuture)
    .thenApply(_ -> new Order(userFuture.join(), paymentFuture.join()))
    .join();
```

**해결책 2: Circuit Breaker 패턴**

```java
@RestController
public class OrderController {
    
    @CircuitBreaker(name = "userService", 
                   fallbackMethod = "getUserFallback")
    public User getUser(Long userId) {
        return userService.getUser(userId);
    }
    
    public User getUserFallback(Long userId, Exception e) {
        return new User(userId, "Unknown"); // 대체 응답
    }
}
```

**해결책 3: Bulkhead 격리**

```java
// User Service 호출용 스레드풀: 10개 스레드
ExecutorService userExecutor = Executors.newFixedThreadPool(10);

// Payment Service 호출용 스레드풀: 20개 스레드
ExecutorService paymentExecutor = Executors.newFixedThreadPool(20);

// 리소스 격리 → 한 서비스 과부하가 다른 서비스에 영향 없음
CompletableFuture<User> userFuture = 
    CompletableFuture.supplyAsync(
        () -> userService.getUser(userId), 
        userExecutor
    );

CompletableFuture<Payment> paymentFuture = 
    CompletableFuture.supplyAsync(
        () -> paymentService.charge(amount), 
        paymentExecutor
    );
```

**권장 조합: 타임아웃 + Circuit Breaker + Bulkhead**

</details>

---

**Q3. Span 생성 오버헤드가 1-2% CPU라고 했는데, 프로덕션에서는 어떻게 관리할까?**

<details>
<summary>해설 보기</summary>

**프로덕션 관리 전략:**

1. **샘플링 적용**
   ```yaml
   # 개발: 100%
   otel.traces.sampler: always_on
   
   # 프로덕션: 0.1% (1000개 요청 중 1개 추적)
   otel.traces.sampler: probability{0.001}
   ```
   - 오버헤드: 1-2% → 0.001-0.002%
   - 장점: 일부 trace는 유지하여 장애 분석 가능

2. **조건부 샘플링**
   ```java
   // 오류 발생 시만 100% 샘플링
   public class ErrorSampler implements Sampler {
       @Override
       public SamplingResult shouldSample(...) {
           if (isError) {
               return SamplingResult.recordAndSample();
           }
           return SamplingResult.drop(); // 샘플 안 함
       }
   }
   ```

3. **느린 요청만 샘플링**
   ```java
   // 응답 시간 > 1초인 요청만 추적
   public class SlowRequestSampler implements Sampler {
       @Override
       public SamplingResult shouldSample(...) {
           if (spanStartTime - requestStartTime > 1000) {
               return SamplingResult.recordAndSample();
           }
           return SamplingResult.drop();
       }
   }
   ```

4. **Jaeger Remote Sampling**
   ```yaml
   # Jaeger 서버에서 샘플링 정책 동적 변경
   otel.traces.sampler: jaeger_remote
   otel.exporter.jaeger.sampler.url: http://jaeger:14250
   ```
   - 운영 중 샘플링 비율 변경 가능 (재배포 불필요)

5. **메트릭 기반 경보**
   ```
   CPU 사용률 증가 → Span 생성 오버헤드 감지
   → 자동으로 샘플링 비율 감소 (0.1% → 0.01%)
   ```

**결론:** 프로덕션에서는 **동적 샘플링**으로 오버헤드 제어, 필요시 조사 목적 추적 유지

</details>

---

<div align="center">

**[⬅️ 이전: 분산 부하 테스트 — k6 Operator로 Kubernetes 실행](./01-distributed-load-testing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Kafka 성능 테스트 — Producer/Consumer 처리량 측정 ➡️](./03-kafka-performance.md)**

</div>
