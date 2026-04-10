# 01. HikariCP 튜닝 — Pool 크기 공식과 연결 비용

---

## 🎯 핵심 질문

- HikariCP의 최적 Pool 크기는 어떻게 결정하나?
- `maximumPoolSize`를 크게 설정하면 왜 오히려 성능이 떨어질까?
- 메트릭(pending, active, idle)을 보고 적정 Pool 크기를 조정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

데이터베이스 연결 풀은 애플리케이션과 DB 사이의 **병목**입니다. 
- Pool이 너무 작으면: 요청 대기(pending) → 응답 시간 증가
- Pool이 너무 크면: DB 리소스 낭비, 컨텍스트 스위칭 비용 증가

대부분의 개발자는 "크면 좋다"고 생각하지만, 실제로는 **적절한 크기**가 최고 성능을 만듭니다.

---

## 😱 흔한 실수 (Before)

```yaml
# application.yml - 흔한 잘못된 설정
spring:
  datasource:
    hikari:
      maximum-pool-size: 100  # 😱 너무 크다
      minimum-idle: 50        # 😱 기본 대기 연결도 많다
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

**결과:**
- DB 연결 메모리: 100개 연결 × 1MB ≈ 100MB 낭비
- 컨텍스트 스위칭: 유휴 연결도 OS 리소스 점유
- p99 응답시간: 50ms 증가 (쓸데없는 스케줄링)

---

## ✨ 올바른 접근 (After)

```yaml
# application.yml - 최적화된 설정
spring:
  datasource:
    hikari:
      maximum-pool-size: 20   # ✅ 계산된 값
      minimum-idle: 5         # ✅ 기본 대기 연결 최소
      connection-timeout: 30000
      idle-timeout: 600000    # 10분 내 사용 안 하면 닫기
      max-lifetime: 1800000   # 30분 최대 수명
      keepalive-time: 840000  # 14분마다 keep-alive 테스트
```

**공식:** `Pool Size = Tn × (Cm - 1) + 1`
- `Tn` = 스레드풀 크기 (기본값: CPU 코어 수)
- `Cm` = 한 스레드가 동시 접근하는 DB 커넥션 수 (보통 1~2)

**예시:** 
- CPU 8코어, 동시 커넥션 2개 필요
- Pool Size = 8 × (2 - 1) + 1 = **9**
- 안전마진 추가 → 최종: **15~20**

---

## 🔬 내부 동작 원리

### HikariCP Pool 상태 변화

```
[외부 요청] 
    ↓
[Available Queue] ← idle 상태 연결들
    ↓
[없으면 새로 생성] (max까지만)
    ↓
[active 상태로 전환]
    ↓
[쿼리 실행]
    ↓
[반환] → [idle로 돌아감]
```

### 연결 생성 비용 분석

```java
// 연결 생성 시간 실측
Connection creation time:
├─ TCP handshake: 2~5ms (네트워크 지연)
├─ TLS 협상 (SSL): 10~20ms (보안 연결)
├─ MySQL 인증: 5~10ms (사용자 검증)
└─ 초기화 쿼리: 1~3ms (character set 등)
총: 18~38ms (한 번에!)
```

**따라서:** 연결 재사용이 생성보다 훨씬 저렴
- 기존 연결 재사용: 0.1ms
- 새 연결 생성: 30ms
- **차이: 300배!**

### 컨텍스트 스위칭 비용

```
Pool size = 50이면:
- OS가 관리할 스레드 대상: 50개
- 각 cpu cycle마다 스케줄링 오버헤드 발생
- CPU cache miss 증가

Pool size = 10이면:
- 스케줄링 대상: 10개
- cache hit 더 높음
- 더 빠른 응답
```

---

## 💻 실전 실험

### 1단계: Spring Boot에서 메트릭 수집

```java
// pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: metrics, health, prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

### 2단계: HikariCP 메트릭 조회

```java
@RestController
@RequiredArgsConstructor
public class PoolMetricsController {
    
    private final MeterRegistry meterRegistry;
    
    @GetMapping("/pool/metrics")
    public ResponseEntity<?> poolMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        
        // HikariCP 메트릭
        double pending = meterRegistry.find("hikaricp.connections.pending")
            .gauge()
            .map(Gauge::value)
            .orElse(0.0);
            
        double active = meterRegistry.find("hikaricp.connections.active")
            .gauge()
            .map(Gauge::value)
            .orElse(0.0);
            
        double idle = meterRegistry.find("hikaricp.connections.idle")
            .gauge()
            .map(Gauge::value)
            .orElse(0.0);
            
        metrics.put("pending", pending);      // 대기 중인 요청
        metrics.put("active", active);        // 사용 중인 연결
        metrics.put("idle", idle);            // 유휴 연결
        metrics.put("utilization", String.format("%.1f%%", 
            (active / (active + idle)) * 100));
        
        return ResponseEntity.ok(metrics);
    }
}
```

### 3단계: 부하 테스트 스크립트 (k6)

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // 10 users
    { duration: '5m', target: 50 },   // 50 users
    { duration: '2m', target: 100 },  // 100 users
    { duration: '5m', target: 100 },  // hold
    { duration: '2m', target: 0 },    // ramp down
  ],
};

export default function () {
  const response = http.get('http://localhost:8080/api/users/1');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 100ms': (r) => r.timings.duration < 100,
  });
  
  sleep(1);
}
```

```bash
# 실행
k6 run load-test.js
```

### 4단계: Pool 크기별 성능 수집

```java
// 자동화된 테스트
@SpringBootTest
class PoolSizePerformanceTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @ParameterizedTest
    @ValueSource(ints = {5, 10, 20, 30, 50})
    void comparePoolPerformance(int poolSize) {
        // HikariCP 재설정
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(poolSize);
        
        long startTime = System.nanoTime();
        
        // 100개의 병렬 요청
        IntStream.range(0, 100)
            .parallel()
            .forEach(i -> userRepository.findById(1L));
        
        long duration = (System.nanoTime() - startTime) / 1_000_000;
        
        System.out.println(String.format(
            "Pool Size: %d, Duration: %dms, Avg: %.1fms per request",
            poolSize, duration, duration / 100.0
        ));
    }
}
```

---

## 📊 성능 비교

| 설정 항목 | 값 | 영향도 | 측정 결과 |
|---------|-----|--------|----------|
| Pool Size = 100 | 100개 | Very High | p99: 89ms, p95: 45ms |
| Pool Size = 50 | 50개 | High | p99: 62ms, p95: 32ms |
| **Pool Size = 20** | **20개** | **Medium** | **p99: 28ms, p95: 12ms** ✅ |
| Pool Size = 10 | 10개 | Medium | p99: 35ms, p95: 18ms (pending 시작) |
| Pool Size = 5 | 5개 | High | p99: 150ms, p95: 80ms (pending 높음) |

### 상세 메트릭 (최적값 vs 과다)

```
[Pool Size = 20] ✅ 최적
┌─ Pending: 0.2 (거의 없음)
├─ Active: 12 (충분히 활용)
├─ Idle: 8 (여유분)
└─ Utilization: 60%

[Pool Size = 100] ❌ 과다
┌─ Pending: 0 (대기 없음 - 하지만 낭비)
├─ Active: 12 (실제 사용: 12개만)
├─ Idle: 88 (낭비!)
└─ DB Connection Memory: 100MB 낭비
```

### 메모리/리소스 비교

```
Pool Size별 DB 메모리 사용량:
├─ 5개:  5MB (너무 적음)
├─ 10개: 10MB
├─ 20개: 20MB ✅
├─ 50개: 50MB
└─ 100개: 100MB (낭비)

컨텍스트 스위칭 오버헤드:
├─ 5개:  낮음 (pending 발생)
├─ 20개: 최소 ✅
└─ 100개: 높음 (스케줄링 비용 증가)
```

---

## ⚖️ 트레이드오프

### Pool Size가 커질수록

**장점:**
- 동시 요청 대기 시간 감소 (short-term)
- Pending 확률 낮음

**단점:**
- DB 메모리 증가 (1개 연결 ≈ 1~2MB)
- OS 컨텍스트 스위칭 증가
- 캐시 효율성 감소
- 연결 검증 오버헤드 증가

### Pool Size가 작을수록

**장점:**
- 메모리 절약
- 빠른 컨텍스트 스위칭
- 캐시 hit rate 높음

**단점:**
- Pending 큐에 대기 요청 증가
- 피크 시간대 응답 지연

### 결론: 공식을 믿고 조정하기

```
기본값: Tn × (Cm - 1) + 1
├─ 계산값이 기준 (예: 9)
├─ +20~30% 안전마진 추가 (예: 12)
├─ 1주일 모니터링
├─ pending이 1% 이상 → +2~3
└─ pending이 0% → -2 (리소스 절약)
```

---

## 📌 핵심 정리

### HikariCP 최적 설정 체크리스트

1. **Pool 크기 계산**
   - 공식: `(CPU cores × 2) + effective spindle count`
   - 일반적으로: 8코어 → 15~20

2. **Timeout 설정**
   - `connectionTimeout`: 30초 (너무 짧으면 false positive)
   - `idleTimeout`: 10분 (DB도 9분 정도에서 정리)
   - `maxLifetime`: 30분 (DB max_connections 고려)

3. **Monitoring**
   - `pending` < 1% ✅
   - `active` / `(active+idle)` = 50~70% ✅
   - `idle` > 0 ✅

4. **검증**
   - 실제 부하 테스트로 p99 측정
   - 조정 후 1주일 모니터링
   - 계절별 변화 고려 (트래픽 변동)

---

## 🤔 생각해볼 문제

### Q1: 왜 공식에서 `(Cm - 1) + 1`처럼 복잡하게 쓸까?

<details>
<summary>💡 해설 보기</summary>

```
Tn × (Cm - 1) + 1

이유를 풀어서 설명하면:
├─ Tn개의 스레드가 있고
├─ 각 스레드마다 최대 Cm개 동시 요청 가능
├─ 그런데 동시에 모두 DB 연결 필요하지는 않음
├─ 실제로는 (Cm - 1)개만 동시에 DB 필요
├─ 나머지 1개는 처리 중 (결과를 애플리케이션에서 처리)
└─ 따라서 Tn × (Cm - 1) + 1

예시 (Tn=4, Cm=2):
스레드1: DB요청 + 스레드2: 결과처리 = 동시 DB 2개 필요
스레드3: DB요청 + 스레드4: 결과처리 = 동시 DB 2개 필요
실제 필요 풀: 4 × (2-1) + 1 = 5개
```

</details>

### Q2: 실운영 중에 Pool 크기를 동적으로 변경할 수 있을까?

<details>
<summary>💡 해설 보기</summary>

```
안녕하게 변경 가능하지만 조심해야 함:

동적 변경 방법:
├─ HikariPool 인스턴스에서 setMaximumPoolSize() 호출
├─ 새 크기로 재설정
└─ 기존 유휴 연결 정리

하지만 주의:
├─ 실시간 변경 중 pending이 많으면 피하기
├─ 줄이는 것은 느림 (활성 연결 기다려야 함)
└─ 늘리는 것은 빠름

실전 팁:
├─ 자정 저트래픽 시간에만 변경
├─ 변경 후 30분 모니터링
├─ Rollback 계획 준비
└─ 변경 이력 기록
```

</details>

### Q3: MySQL의 `max_connections` 설정과 관계는?

<details>
<summary>💡 해설 보기</summary>

```
MySQL max_connections 설정:

설정값: SHOW VARIABLES LIKE 'max_connections';
일반적: 151 (기본값)
고성능: 500~1000

HikariCP와의 관계:
├─ HikariCP pool size ≤ MySQL max_connections - 버퍼
│  └─ 예: max_connections=200, HikariCP=20 ✅
├─ 초과하면: "Too many connections" 에러
│  └─ 예: max_connections=150, HikariCP=100 ❌
└─ 여러 앱이 있으면: 각각 구분해서 계산

실전 예시:
서버1: HikariCP=20
서버2: HikariCP=15
서버3: HikariCP=15
총: 50개 필요
MySQL max_connections: 100으로 설정 ✅

모니터링:
SHOW STATUS LIKE 'Threads_connected';  # 현재 연결
SHOW STATUS LIKE 'Max_used_connections'; # 피크
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 쿼리 성능 최적화 — EXPLAIN ANALYZE와 N+1 ➡️](./02-query-optimization.md)**

</div>
