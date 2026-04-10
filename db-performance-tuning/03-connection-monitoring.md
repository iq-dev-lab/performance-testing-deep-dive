# 03. 커넥션 모니터링 — 고갈 시 에러 패턴 분석

---

## 🎯 핵심 질문

- 커넥션 풀이 고갈되면 어떤 에러가 나타날까?
- MySQL의 `Threads_connected` 메트릭은 무엇을 의미하나?
- HikariCP의 `pending` 메트릭으로 문제를 미리 감지할 수 있나?

---

## 🔍 왜 이 개념이 실무에서 중요한가

커넥션 고갈은 **갑자기** 발생합니다.
- 평소: 50개 쓰던 풀이 어느 날 갑자기 100개 필요
- 원인: 느린 쿼리 1개가 연결을 붙잡음
- 결과: 새 요청 500개 모두 타임아웃 → 서비스 전체 마비

**실전 팁:** 모니터링 없이는 **블랙박스** 상태입니다.

---

## 😱 흔한 실수 (Before)

### 실수 1: 모니터링 없이 운영

```java
// application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 30000

// 모니터링 없음 😱
// → 어느 날 갑자기 "Connection is not available" 에러 폭주
// → 원인 파악 불가 (모니터링이 없으니까)
```

### 실수 2: 에러 로그로 원인 추측

```
2026-04-10 14:32:15 ERROR HikariPool-1 - 
Connection is not available, request timed out after 30000ms.

// 😱 에러 로그만으로는:
// - 몇 개 남았는지?
// - 왜 갑자기 고갈됐는지?
// - 어떤 쿼리가 문제인지?
// → 모두 불명확
```

### 실수 3: 읽기/쓰기를 구분하지 않기

```java
@Service
public class UserService {
    
    // ❌ 문제: 읽기도 트랜잭션
    @Transactional
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // → 읽기만으로도 연결 점유
    // → 동시 요청 많으면 풀 고갈
}
```

---

## ✨ 올바른 접근 (After)

### 접근 1: MySQL 메트릭 실시간 모니터링

```sql
-- ✅ 현재 연결 상태 조회
SHOW STATUS LIKE 'Threads_connected';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 45    |
+-------------------+-------+

-- ✅ 누적 연결 수
SHOW STATUS LIKE 'Connections';
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| Connections   | 1023456 |
+---------------+---------+

-- ✅ 최대 동시 연결
SHOW STATUS LIKE 'Max_used_connections';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| Max_used_connections| 89    |  -- 피크는 89개
+---------------------+-------+

-- ✅ 설정 확인
SHOW VARIABLES LIKE 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 200   |  -- 최대 200개까지 가능
+-----------------+-------+

-- 현재 사용률: 45/200 = 22.5% (양호)
```

### 접근 2: HikariCP 메트릭으로 풀 상태 감시

```java
@RestController
@RequiredArgsConstructor
public class ConnectionPoolMonitorController {
    
    private final MeterRegistry meterRegistry;
    
    @GetMapping("/monitoring/pool")
    public ResponseEntity<?> monitorPool() {
        Map<String, Object> poolStatus = new HashMap<>();
        
        // 1️⃣ Active (현재 사용 중)
        double active = getMetricValue("hikaricp.connections.active");
        
        // 2️⃣ Idle (대기 중)
        double idle = getMetricValue("hikaricp.connections.idle");
        
        // 3️⃣ Pending (연결 대기 중 - 중요!)
        double pending = getMetricValue("hikaricp.connections.pending");
        
        // 4️⃣ Max (최대값)
        double max = getMetricValue("hikaricp.connections.max");
        
        // 상태 판정
        String status = evaluatePoolHealth(active, idle, pending, max);
        
        poolStatus.put("active", active);      // 12개
        poolStatus.put("idle", idle);          // 8개
        poolStatus.put("pending", pending);    // 0개 ✅
        poolStatus.put("max", max);            // 20개
        poolStatus.put("utilization", String.format("%.1f%%", 
            (active / max) * 100));            // 60% ✅
        poolStatus.put("status", status);      // "HEALTHY" ✅
        poolStatus.put("healthAlert", generateAlert(pending, active, max));
        
        return ResponseEntity.ok(poolStatus);
    }
    
    private double getMetricValue(String metricName) {
        return meterRegistry.find(metricName)
            .gauge()
            .map(Gauge::value)
            .orElse(0.0);
    }
    
    private String evaluatePoolHealth(double active, double idle, 
                                      double pending, double max) {
        if (pending > 0) {
            return "WARNING - 연결 대기 중!";
        }
        
        double utilization = (active / max) * 100;
        if (utilization > 80) {
            return "WARNING - 사용률 80% 이상";
        }
        
        if (utilization > 60) {
            return "CAUTION - 사용률 높음";
        }
        
        return "HEALTHY - 정상";
    }
    
    private String generateAlert(double pending, double active, double max) {
        StringBuilder alert = new StringBuilder();
        
        if (pending > 0) {
            alert.append("🚨 연결 고갈 임박! ");
            alert.append(String.format("%.0f개 요청 대기 중\n", pending));
            alert.append("즉시 조치: ");
            alert.append("1) pool size 증가 ");
            alert.append("2) 느린 쿼리 추적 ");
            alert.append("3) 읽기 전용 분리\n");
        }
        
        double utilization = (active / max) * 100;
        if (utilization > 80) {
            alert.append("⚠️ 풀 사용률 ");
            alert.append(String.format("%.0f%%\n", utilization));
        }
        
        return alert.toString();
    }
}

@Component
public class PoolHealthScheduler {
    
    @Autowired
    private ConnectionPoolMonitorController monitor;
    
    // 1분마다 체크
    @Scheduled(fixedDelay = 60000)
    public void checkPoolHealth() {
        ResponseEntity<?> response = monitor.monitorPool();
        Map<String, Object> status = (Map<String, Object>) response.getBody();
        
        double pending = (double) status.get("pending");
        if (pending > 0) {
            // 알람 발송
            sendAlert("커넥션 풀 경고: pending=" + pending);
        }
    }
    
    private void sendAlert(String message) {
        // Slack, Email, PagerDuty 등으로 알림
        System.err.println("[ALERT] " + message);
    }
}
```

### 접근 3: 읽기 전용 트랜잭션 분리

```java
// ✅ 읽기 전용 → 더 가벼운 트랜잭션
@Service
public class UserService {
    
    // 쓰기: 풀에서 연결 사용
    @Transactional
    public void createUser(CreateUserRequest req) {
        userRepository.save(new User(req.getName()));
    }
    
    // ✅ 읽기: 더 빠른 처리 (트랜잭션 최소화)
    @Transactional(readOnly = true)
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // ✅ 단순 조회: 트랜잭션 제거
    public long countUsers() {
        return userRepository.count();
    }
}
```

---

## 🔬 내부 동작 원리

### 커넥션 고갈 시나리오 분석

```
[정상 상태]
Pool: [C1, C2, C3, C4, C5 (idle), C6, C7, C8, C9, C10 (active)]
      (idle: 5)  (active: 5)  (pending: 0)
      응답: 빠름 ✅

[문제 발생 1: 느린 쿼리]
쿼리: SELECT * FROM large_table WHERE status = 'active' ORDER BY created_at;
      (50M 행 Full Table Scan, 30초 소요!)

상태: 
Pool: [C1, C2 (idle), C3-C10 (활성, 느린 쿼리 실행)]
      (idle: 2) (active: 8) (pending: 0)
      하지만 이 중 7개는 실제로는 응답 기다리는 중...

[문제 발생 2: 동시 요청 증가]
100개의 새로운 요청이 들어옴
Pool: [C1 (idle 없음)]
      ├─ C1 사용 중
      ├─ C2-C10: 느린 쿼리 (30초)
      ├─ 나머지 99개 요청: 연결 대기
      └─ pending: 99 ⚠️

[타임아웃 발생]
30초 이후:
└─ 기다리던 요청들: "Connection is not available, timed out"
└─ 서비스 중단
```

### Pending 큐의 작동

```
새 요청 들어옴
    ↓
Available Queue에서 idle 연결 찾기
    ├─ 있음: 즉시 할당 (0ms)
    └─ 없음: Pending Queue에 대기
        ↓
        활성 연결이 반환될 때까지 대기
        ↓
        30초(connectionTimeout) 초과
        ↓
        TimeoutException 발생 💥

모니터링 가치:
├─ pending = 0: 정상 ✅
├─ pending > 0: 문제 시작 ⚠️
└─ pending 계속 증가: 긴급 ⛔
```

### 읽기 전용 트랜잭션의 효과

```java
// 일반 트랜잭션
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}

처리:
1️⃣ 연결 획득: 1ms
2️⃣ 트랜잭션 시작 (BEGIN): 0.5ms
3️⃣ SELECT 쿼리: 10ms
4️⃣ 트랜잭션 커밋 (COMMIT): 0.5ms
5️⃣ 연결 반환: 1ms
총: 13ms (연결 점유 시간)

// 읽기 전용 트랜잭션
@Transactional(readOnly = true)
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}

처리:
1️⃣ 연결 획득: 1ms
2️⃣ 트랜잭션 시작 (BEGIN): 0.5ms
3️⃣ SELECT 쿼리: 10ms
4️⃣ 트랜잭션 커밋 제거 (자동 ROLLBACK): 0.1ms ⬇️
5️⃣ 연결 반환: 1ms
총: 12.6ms (연결 점유 시간) - 약간 빠름

더 중요: DB는 lock을 획득하지 않으므로
└─ 다른 트랜잭션과의 충돌 없음
└─ 동시성 향상 ✅

// 트랜잭션 제거
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}

처리:
1️⃣ 연결 획득: 1ms
2️⃣ SELECT 쿼리: 10ms
3️⃣ 연결 반환: 1ms
총: 12ms (가장 빠름!) ⭐
└─ 트랜잭션 오버헤드 완전 제거
```

---

## 💻 실전 실험

### 1단계: 모니터링 대시보드 설정 (Prometheus + Grafana)

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-app'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
```

```json
// Grafana 대시보드 JSON (시각화)
{
  "dashboard": {
    "title": "HikariCP Connection Pool",
    "panels": [
      {
        "title": "Active Connections",
        "targets": [
          {
            "expr": "hikaricp_connections_active"
          }
        ]
      },
      {
        "title": "Pending Requests",
        "targets": [
          {
            "expr": "hikaricp_connections_pending"
          }
        ]
      },
      {
        "title": "Pool Utilization",
        "targets": [
          {
            "expr": "hikaricp_connections_active / hikaricp_connections_max * 100"
          }
        ]
      }
    ]
  }
}
```

### 2단계: 알람 설정

```yaml
# alerting_rules.yml
groups:
  - name: connection_pool
    interval: 30s
    rules:
      - alert: PoolConnectionHigh
        expr: hikaricp_connections_pending > 0
        for: 1m
        annotations:
          summary: "연결 풀 고갈 경고"
          
      - alert: PoolUtilizationHigh
        expr: (hikaricp_connections_active / hikaricp_connections_max) > 0.8
        for: 5m
        annotations:
          summary: "풀 사용률 80% 초과"
```

### 3단계: 느린 쿼리 추적

```java
@Component
public class SlowQueryInterceptor implements HandlerInterceptor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        long duration = (Long) request.getAttribute("startTime");
        if (duration > 1000) {  // 1초 이상
            String endpoint = request.getRequestURI();
            
            // 메트릭 기록
            meterRegistry.timer("http.request.duration.slow", 
                "endpoint", endpoint).record(duration, TimeUnit.MILLISECONDS);
            
            // 로그 기록
            log.warn("슬로우 요청: {} ({}ms)", endpoint, duration);
            
            // 동시에 풀 상태 로깅
            double active = getMetricValue("hikaricp.connections.active");
            double pending = getMetricValue("hikaricp.connections.pending");
            log.warn("풀 상태: active={}, pending={}", active, pending);
        }
    }
}
```

### 4단계: 부하 테스트 (고갈 재현)

```javascript
// load-test.js - 고갈 상황 만들기
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // 50 users
    { duration: '2m', target: 100 },  // 100 users (고갈 시작)
    { duration: '2m', target: 100 },  // 유지
    { duration: '1m', target: 0 },    // 감소
  ],
};

export default function () {
  // 느린 쿼리 시뮬레이션
  const response = http.get(
    'http://localhost:8080/api/users/search?query=slow'
  );
  
  check(response, {
    'status 200': (r) => r.status === 200,
    'status 200 or 500': (r) => r.status === 200 || r.status === 500,
  });
  
  // 대기 없음 - 빠르게 연쇄 요청
  // (연결이 쌓이게 됨)
}
```

```bash
# 실행
k6 run load-test.js

# 다른 터미널에서 모니터링
watch -n 1 'curl -s http://localhost:8080/monitoring/pool | jq'
```

### 5단계: 해결 과정 검증

```bash
# 문제 상황 확인
curl http://localhost:8080/monitoring/pool
{
  "active": 20,      # 모두 사용 중
  "idle": 0,         # 대기 연결 없음
  "pending": 45,     # ⚠️ 45개 요청 대기!
  "max": 20,
  "utilization": "100%",
  "status": "WARNING - 연결 대기 중!"
}

# 느린 쿼리 분석
tail -100 logs/p6spy.log | grep "SELECT" | grep -v "FROM information_schema"

# 느린 쿼리 강제 종료
SHOW PROCESSLIST;
KILL 12345;  # 느린 쿼리 프로세스 ID

# 풀 상태 모니터링
watch -n 1 "mysql -u root -p -e \"SHOW STATUS LIKE 'Threads_connected'\""

# 개선: pool size 증가
spring.datasource.hikari.maximum-pool-size=30

# 재확인
curl http://localhost:8080/monitoring/pool
{
  "active": 18,
  "idle": 12,
  "pending": 0,      # ✅ 정상!
  "max": 30,
  "utilization": "60%"
}
```

---

## 📊 성능 비교

### 풀 고갈 전후 메트릭

| 지표 | 정상 (Before) | 고갈 (During) | 개선 (After) |
|------|-------------|------------|----------|
| Active | 12 | 20 (max) | 15 |
| Idle | 8 | 0 | 15 |
| Pending | 0 | 45+ | 0 |
| 응답시간 p99 | 28ms | 30,234ms | 32ms |
| 에러율 | 0% | 15% | 0.001% |
| DB CPU | 45% | 95% | 52% |

### 읽기 전용 분리 효과

```
[Before] 모든 조회에 @Transactional
총 요청: 1000개/초
├─ 쓰기: 50개/초 (@Transactional)
├─ 읽기: 950개/초 (@Transactional) ❌
└─ 필요 pool size: 25

[After] 읽기는 @Transactional(readOnly=true)
총 요청: 1000개/초
├─ 쓰기: 50개/초 (@Transactional)
├─ 읽기: 950개/초 (@Transactional(readOnly=true)) ✅
└─ 필요 pool size: 15 (감소!)

개선:
├─ 풀 크기: 25 → 15 (40% 감소)
├─ 메모리: 25MB → 15MB
├─ 동시 처리: 25 → 30명 (더 많은 사용자)
└─ pending 확률: 0.5% → 0.02%
```

---

## ⚖️ 트레이드오프

### 모니터링 추가의 비용

**장점:**
- 문제를 빨리 감지 (분 단위)
- 원인을 정확히 파악 (로그)
- 미리 예방 가능

**단점:**
- 메트릭 수집 오버헤드 (1~3%)
- 모니터링 시스템 유지 (Prometheus, Grafana)
- 알람 설정 복잡도

### 풀 크기 증가 vs 읽기 분리

**풀 크기 증가:**
- 즉각적인 해결
- 메모리 낭비
- 근본 원인 미해결

**읽기 전용 분리 (권장):**
- 근본 원인 해결
- 메모리 절약
- 성능 개선
- 약간의 코드 변경

---

## 📌 핵심 정리

### 모니터링 체크리스트

```
1️⃣ MySQL 메트릭
   └─ Threads_connected, Max_used_connections

2️⃣ HikariCP 메트릭
   ├─ active (사용 중)
   ├─ idle (대기 중)
   ├─ pending (연결 대기) ⭐ 가장 중요
   └─ max (최대값)

3️⃣ 대시보드 구성
   ├─ Grafana로 실시간 시각화
   ├─ 30초 간격 스크랩
   └─ Alertmanager로 알람

4️⃣ 알람 규칙
   ├─ pending > 0: 즉시 조사
   ├─ utilization > 80%: 경고
   └─ 연속 에러: 긴급
```

### 커넥션 고갈 대응 SOP

```
🚨 고갈 상황 발생

1️⃣ 즉시 조치 (10분 이내)
   ├─ 메트릭 확인 (pending, active)
   ├─ 느린 쿼리 분석 (p6spy)
   ├─ SHOW PROCESSLIST
   └─ 필요시 느린 쿼리 강제 종료

2️⃣ 임시 해결 (1시간 이내)
   ├─ pool size +5 증가 (rolling restart)
   ├─ 읽기 전용 쿼리 식별
   └─ 캐시 추가 검토

3️⃣ 근본 해결 (1주)
   ├─ N+1 쿼리 제거
   ├─ 느린 쿼리 인덱스 추가
   ├─ 읽기 @Transactional(readOnly=true) 적용
   └─ 코드 리뷰
```

---

## 🤔 생각해볼 문제

### Q1: pending 메트릭이 0이 아니면서 응답이 빠를 수 있을까?

<details>
<summary>💡 해설 보기</summary>

```
가능하지만 드문 경우:

pending > 0이면서 p99 < 50ms?

이유:
├─ pending이 빠르게 처리됨
│  └─ 대기 큐에 있지만 밀리초 내 처리
├─ 대기 시간이 적음
│  └─ pending 요청들의 wait time < 10ms
└─ 예외 상황
   └─ 초단위 pending 수신 (즉시 처리)

하지만 주의:
├─ pending > 0이 지속 → 구조적 문제
├─ pending이 증가 추세 → 예방 필요
└─ pending 피크 후 에러 → 고갈 직전 신호
```

</details>

### Q2: 읽기 전용 DB 복제본(Replica)을 쓰면 어떻게 될까?

<details>
<summary>💡 해설 보기</summary>

```java
// Primary-Replica 분리
@Configuration
public class DataSourceConfig {
    
    // 쓰기 (Primary)
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return createDataSource("primary-db", 20);
    }
    
    // 읽기 (Replica)
    @Bean
    public DataSource replicaDataSource() {
        return createDataSource("replica-db", 20);
    }
}

@Service
public class UserService {
    
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;
    
    @Autowired
    @Qualifier("replicaDataSource")
    private DataSource replicaDataSource;
    
    // 쓰기: Primary (연결 풀 20)
    @Transactional
    public void createUser(CreateUserRequest req) {
        // Primary 사용
        userRepository.save(new User(req.getName()));
    }
    
    // 읽기: Replica (연결 풀 20)
    @Transactional(readOnly = true)
    public User getUser(Long id) {
        // Replica 사용
        return userRepository.findById(id).orElse(null);
    }
}

효과:
├─ Primary: 20개 (쓰기 전용)
├─ Replica: 20개 (읽기 전용)
└─ 총 가용 연결: 40개 (2배!)

추가 benefit:
├─ Primary: CPU 50% (쓰기)
├─ Replica: CPU 40% (읽기) - 분산됨
└─ 데이터 센터 레벨 확장 가능

단점:
├─ 복제 지연 (일반적 < 1ms, 피크시 1s)
├─ 복제 실패 처리 (fallback to primary)
└─ 운영 복잡도 증가
```

</details>

### Q3: 커넥션 풀 외에 다른 병목이 있을까?

<details>
<summary>💡 해설 보기</summary>

```
커넥션 풀이 해결 안 되는 경우:

1️⃣ DB 자체가 느림
   └─ CPU 100%, I/O 대기
   └─ 쿼리 최적화, 인덱스 필요

2️⃣ 네트워크 지연
   └─ 애플리케이션 ↔ DB 간 RTT 100ms+
   └─ 데이터베이스 인스턴스 위치 확인

3️⃣ 트랜잭션 충돌
   └─ 동시 업데이트 대기
   └─ SERIALIZABLE isolation level 확인

4️⃣ 메모리 부족
   └─ OOM 에러
   └─ JVM heap 설정 확인

진단:
├─ pending = 0 → 풀은 양호
├─ active < max → 풀이 문제 아님
├─ response slow + pending=0 → DB 느림
└─ response slow + pending>0 → 풀 부족
```

</details>

---

<div align="center">

**[⬅️ 이전: 쿼리 성능 최적화 — EXPLAIN ANALYZE와 N+1](./02-query-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: 캐시 전략과 효과 측정 — Redis 도입 전후 비교 ➡️](./04-cache-strategy-measurement.md)**

</div>
