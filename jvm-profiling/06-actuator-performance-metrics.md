# 06. Spring Boot Actuator 성능 메트릭

---

## 🎯 핵심 질문

**"모니터링 도구(Prometheus, Grafana)를 구축했는데, 정확히 어떤 메트릭을 수집해야 할까? JVM 성능을 한눈에 보는 대시보드를 만들 수 있을까?"**

**Spring Boot Actuator**는 애플리케이션의 실시간 성능 메트릭을 HTTP 엔드포인트로 제공합니다. 이를 Prometheus로 수집하고 Grafana로 시각화하면, 성능 문제를 **즉시 감지하고 대응**할 수 있습니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 감지의 시차:**

```
[문제 없음] → [문제 발생] → [감지] → [대응]
                  |         |       |
              보통 5~10분   |       |
                        수동 모니터링    긴급 호출
                        (사람이 봐야 함)  (늦음)

vs.

[문제 없음] → [문제 발생] → [즉시 감지] → [자동 알람]
                  |         | (1초)      |
              실시간 메트릭  수집        [자동 조치]
                수집        (1초)        또는 [신속 대응]
```

**Actuator의 가치:**
1. **실시간 모니터링** — 언제든 현재 상태 파악
2. **자동 알림** — 임계값 초과 시 알람
3. **성능 추이** — 시간에 따른 변화 분석
4. **용량 계획** — 증가 추세 예측
5. **SLA 검증** — p99 응답시간 검증

---

## 😱 흔한 실수 (Before)

```properties
# ❌ Actuator 활성화하지 않음
# application.properties에 아무 설정 안 함
# 결과: /actuator 엔드포인트 403 Forbidden

# ❌ 불완전한 노출 설정
management.endpoints.web.exposure.include=health,info
# 결과: 성능 메트릭 수집 불가능
# (prometheus, metrics, heapdump 등이 필수)

# ❌ 메트릭 수집은 하지만 분석 안 함
# Actuator에서 메트릭 제공하지만 아무도 보지 않음
# 결과: 문제 발생해도 인식 못 함
```

**문제점:**
- Actuator 필요성 인식 부족
- 어떤 엔드포인트가 필요한지 모름
- Prometheus, Grafana 연동 복잡

---

## ✨ 올바른 접근 (After)

### 1단계: Spring Boot Actuator 활성화

```properties
# application.properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,heapdump,threaddump,loggers
management.endpoints.web.base-path=/actuator
management.endpoint.health.show-details=always

# 성능 메트릭 상세 설정
management.metrics.export.prometheus.enabled=true
management.metrics.tags.application=my-app
management.metrics.tags.environment=production

# 헬스체크 상세 정보
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
```

**의존성 추가:**

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 2단계: 주요 메트릭 엔드포인트 이해

```bash
# 헬스 체크
curl http://localhost:8080/actuator/health
# 응답:
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "livenessState": { "status": "UP" },
    "readinessState": { "status": "UP" }
  }
}

# 전체 메트릭 목록
curl http://localhost:8080/actuator/metrics
# 응답:
{
  "names": [
    "jvm.memory.used",
    "jvm.memory.committed",
    "jvm.threads.live",
    "jvm.gc.pause",
    "http.server.requests",
    "hikaricp.connections.active",
    ...
  ]
}

# Prometheus 형식 (시계열 데이터)
curl http://localhost:8080/actuator/prometheus
# 응답:
jvm_memory_used_bytes{area="heap",id="PS Survivor Space",} 1048576.0
jvm_memory_committed_bytes{area="heap",id="PS Survivor Space",} 20971520.0
jvm_threads_live 23.0
http_server_requests_seconds_count{method="GET",status="200",uri="/api/users",} 1234.0
http_server_requests_seconds_sum{method="GET",status="200",uri="/api/users",} 234.567
```

### 3단계: 핵심 JVM 메트릭 모니터링

**메모리 메트릭:**

```bash
# JVM 메모리 사용량
curl http://localhost:8080/actuator/metrics/jvm.memory.used
# 응답:
{
  "name": "jvm.memory.used",
  "baseUnit": "bytes",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 536870912.0
    }
  ],
  "availableTags": [
    {
      "tag": "area",
      "values": ["heap", "nonheap"]
    },
    {
      "tag": "id",
      "values": [
        "PS Survivor Space",
        "PS Old Generation",
        "PS Eden Space",
        "Metaspace",
        "Code Cache",
        "Compressed Class Space"
      ]
    }
  ]
}

# 필터링 (Heap만)
curl "http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap"
```

**GC 메트릭:**

```bash
# GC 일시 중지 시간
curl http://localhost:8080/actuator/metrics/jvm.gc.pause
# 응답:
{
  "name": "jvm.gc.pause",
  "baseUnit": "seconds",
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 42.0
    },
    {
      "statistic": "TOTAL_TIME",
      "value": 1.234567  # 총 GC 시간: 1.23초
    },
    {
      "statistic": "MAX",
      "value": 0.0856   # 최대 GC STW: 85.6ms
    }
  ]
}

# GC 최대 데이터
curl "http://localhost:8080/actuator/metrics/jvm.gc.memory.allocated"
# 시간 단위로 할당된 메모리
```

**스레드 메트릭:**

```bash
# 활성 스레드 수
curl http://localhost:8080/actuator/metrics/jvm.threads.live
# 응답:
{
  "measurements": [{"statistic": "VALUE", "value": 23.0}]
}

# 스레드 상태 상세
curl http://localhost:8080/actuator/metrics/jvm.threads.live
# 추가 태그:
# - state: runnable, blocked, waiting, timed-waiting, new, terminated
```

### 4단계: HTTP 요청 메트릭 분석

```bash
# HTTP 엔드포인트별 응답시간
curl "http://localhost:8080/actuator/prometheus" | \
  grep "http_server_requests_seconds"

# 응답:
http_server_requests_seconds_count{method="GET",status="200",uri="/api/users",} 1234.0
http_server_requests_seconds_sum{method="GET",status="200",uri="/api/users",} 234.567
http_server_requests_seconds_max{method="GET",status="200",uri="/api/users",} 0.5234

http_server_requests_seconds_count{method="POST",status="201",uri="/api/users",} 567.0
http_server_requests_seconds_sum{method="POST",status="201",uri="/api/users",} 89.123
http_server_requests_seconds_max{method="POST",status="201",uri="/api/users",} 0.2345

# 분석:
# GET /api/users:
#   count=1234 (총 요청 수)
#   sum=234.567초 (총 시간)
#   평균 = 234.567 / 1234 = 190ms
#   max=523.4ms (최악)
```

### 5단계: HikariCP(데이터베이스 커넥션 풀) 메트릭

```bash
# 활성 커넥션
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
# 응답: 12개 (현재 사용 중인 연결)

# 대기 중인 요청
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
# 응답: 2개 (커넥션 할당 대기)

# 유휴 커넥션
curl http://localhost:8080/actuator/metrics/hikaricp.connections.idle
# 응답: 8개 (가용 상태)

# 최대 커넥션
curl http://localhost:8080/actuator/metrics/hikaricp.connections
# 응답: 20개 (설정된 최대값)

# 분석:
# active(12) + idle(8) = 20 (풀이 가득함)
# pending(2) > 0 → 커넥션 부족!
# → hikaricp.maximum-pool-size 증가 필요
```

---

## 🔬 내부 동작 원리

### Actuator 메트릭 수집 아키텍처

```
┌──────────────────────────────────────────────┐
│ 애플리케이션                                  │
├──────────────────────────────────────────────┤
│ • 메트릭 생성                                │
│   - 메서드 실행 시간                        │
│   - 메모리 할당                             │
│   - DB 쿼리 시간                            │
│   - HTTP 요청 수                            │
│                                              │
│ MeterRegistry (중앙 수집기)                 │
│   ├─ Timer (응답시간)                       │
│   ├─ Counter (횟수)                         │
│   ├─ Gauge (현재값)                         │
│   └─ DistributionSummary (분포)             │
└──────────────────────────────────────────────┘
        ↓
┌──────────────────────────────────────────────┐
│ Actuator Exporter                            │
├──────────────────────────────────────────────┤
│ /actuator/metrics         (JSON)             │
│ /actuator/prometheus      (시계열)            │
│ /actuator/health          (헬스)             │
└──────────────────────────────────────────────┘
        ↓
┌──────────────────────────────────────────────┐
│ 외부 모니터링 시스템                         │
├──────────────────────────────────────────────┤
│ Prometheus: 시계열 데이터베이스              │
│ Grafana: 시각화 및 알람                      │
│ DataDog/NewRelic: APM                        │
└──────────────────────────────────────────────┘
```

### 메트릭 종류별 특징

```
┌──────────────────────────────────────┐
│ Gauge (숫자)                          │
├──────────────────────────────────────┤
│ 용도: 현재 상태를 나타내는 수치      │
│ 예시:                                │
│  • jvm.memory.used = 512MB           │
│  • jvm.threads.live = 23             │
│  • hikaricp.connections.active = 12  │
│ 특징: 증가/감소 모두 가능             │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Counter (누적)                        │
├──────────────────────────────────────┤
│ 용도: 누적 횟수                       │
│ 예시:                                │
│  • http_server_requests_count = 10k  │
│  • jvm.gc.count = 42                 │
│ 특징: 항상 증가만 함 (리셋 안 됨)     │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Timer (분포)                          │
├──────────────────────────────────────┤
│ 용도: 응답시간 분포                   │
│ 예시:                                │
│  • http_server_requests_seconds      │
│    - count: 1000                     │
│    - sum: 230초                      │
│    - max: 0.5초                      │
│ 특징: 분위수 계산 가능                │
│  (p50, p99, p999 등)                 │
└──────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: Prometheus + Grafana 구축

**docker-compose.yml:**

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

  my-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=production
```

**prometheus.yml:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    static_configs:
      - targets: ['my-app:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
```

**실행:**

```bash
docker-compose up -d

# 확인
curl http://localhost:9090/api/v1/targets
# Prometheus 상태 확인: UP/DOWN

curl http://localhost:9090/api/v1/query?query=jvm_memory_used_bytes
# 메트릭 조회
```

---

### 실험 2: Grafana 대시보드 구성

**Grafana 접속:**
```
http://localhost:3000
id: admin
password: admin
```

**대시보드 생성:**

```
1. Create → Dashboard
2. Add Panel → Prometheus 데이터소스 선택

Panel 1: JVM 메모리 사용량
Query: jvm_memory_used_bytes{area="heap"}
Title: "Heap Memory Used"
Unit: bytes
Visualization: Gauge (또는 Graph)

Panel 2: 활성 스레드
Query: jvm_threads_live
Title: "Active Threads"
Unit: short
Visualization: Gauge

Panel 3: GC 일시 중지 시간
Query: increase(jvm_gc_pause_seconds_sum[1m])
Title: "GC Pause Time (1min)"
Unit: seconds
Visualization: Graph

Panel 4: HTTP 요청 응답시간 (p99)
Query: histogram_quantile(0.99, http_server_requests_seconds_bucket)
Title: "HTTP p99 Latency"
Unit: seconds
Visualization: Graph

Panel 5: 활성 DB 커넥션
Query: hikaricp_connections_active
Title: "Active DB Connections"
Visualization: Gauge
Alert: > 15

Panel 6: 대기 중인 DB 요청
Query: hikaricp_connections_pending
Title: "Pending DB Requests"
Alert: > 0 (즉시 알람)
```

**Grafana에서 시각화된 예시:**

```
┌─────────────────────────────────────────────┐
│ JVM Health Dashboard                        │
├─────────────────────────────────────────────┤
│                                             │
│  ┌───────┐  ┌────────┐  ┌──────┐         │
│  │ Heap  │  │Threads │  │ GC   │         │
│  │ 512MB │  │  23    │  │ 42ms │         │
│  └───────┘  └────────┘  └──────┘         │
│                                             │
│  ┌─ Memory Usage (1h) ──────────────────┐ │
│  │                      /‾‾‾‾‾          │ │
│  │              /‾‾‾‾‾‾                │ │
│  │        /‾‾‾‾                        │ │
│  │ ____/                               │ │
│  └─────────────────────────────────────┘ │
│                                             │
│  ┌─ HTTP Latency p99 (1h) ──────────────┐ │
│  │                                  •   │ │
│  │                              •   •   │ │
│  │                          •   •       │ │
│  │ • • • • • • • • • • • • •           │ │
│  └─────────────────────────────────────┘ │
│                                             │
│  Active DB Connections: 12/20             │
│  Pending Requests: 0                      │
│                                             │
└─────────────────────────────────────────────┘
```

---

### 실험 3: 커스텀 메트릭 추가

```java
// CustomMetricsExample.java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

@Service
public class UserService {
    private final MeterRegistry meterRegistry;
    private final Timer userCreationTimer;
    
    public UserService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // 커스텀 메트릭 등록
        this.userCreationTimer = Timer
            .builder("user.creation.time")
            .description("사용자 생성 시간")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
    }
    
    public User createUser(String name) {
        return userCreationTimer.recordCallable(() -> {
            // 비즈니스 로직
            return new User(name);
        });
    }
    
    public void loginUser(String userId) {
        // 카운터: 로그인 횟수
        meterRegistry.counter("user.login.count", 
            "userId", userId
        ).increment();
        
        // Gauge: 현재 로그인한 사용자 수
        meterRegistry.gauge("user.active.count",
            () -> getActiveUserCount()
        );
    }
    
    private int getActiveUserCount() {
        // DB 조회 또는 캐시
        return 42;
    }
}
```

**메트릭 확인:**

```bash
curl http://localhost:8080/actuator/metrics/user.creation.time
# 응답:
{
  "measurements": [
    {"statistic": "COUNT", "value": 156.0},
    {"statistic": "TOTAL_TIME", "value": 12.34},
    {"statistic": "MAX", "value": 0.234}
  ]
}

curl http://localhost:8080/actuator/metrics/user.login.count
# 응답: 로그인 총 횟수

curl http://localhost:8080/actuator/metrics/user.active.count
# 응답: 현재 활성 사용자 = 42
```

---

## 📊 성능 비교

| 메트릭 | 의미 | 정상 범위 | 경고 | 위험 |
|--------|------|---------|------|------|
| Heap Memory Used | 힙 메모리 사용률 | 30~70% | >80% | >90% |
| GC Pause Time | STW 시간 | <100ms | 100~500ms | >500ms |
| Active Threads | 활성 스레드 수 | <100 | 100~500 | >500 |
| HTTP p99 Latency | 응답시간 p99 | <100ms | 100~500ms | >500ms |
| Active DB Connections | 활성 커넥션 | <50% | 50~80% | >80% |
| Pending DB Requests | 대기 요청 | 0 | 1~5 | >5 |
| Full GC Count/hour | 시간당 Full GC | 0 | 1~2 | >2 |

---

## ⚖️ 트레이드오프

### 메트릭 수집 수준

```
┌───────────────────────────────────────┐
│ 최소 (기본)                           │
├───────────────────────────────────────┤
│ 메트릭: health, jvm.memory, jvm.gc   │
│ 오버헤드: < 1%                       │
│ 저장 용량: 10MB/day                  │
│ 분석 가능: 기본 모니터링              │
│ 권장: 모든 환경                      │
└───────────────────────────────────────┘

┌───────────────────────────────────────┐
│ 표준 (권장)                           │
├───────────────────────────────────────┤
│ 메트릭: + http.requests, db.* metrics │
│ 오버헤드: 2~3%                       │
│ 저장 용량: 100MB/day                 │
│ 분석 가능: 상세 성능 분석             │
│ 권장: 프로덕션 필수                  │
└───────────────────────────────────────┘

┌───────────────────────────────────────┐
│ 고상세 (모든 메트릭)                  │
├───────────────────────────────────────┤
│ 메트릭: + 모든 메서드, 캐시 등       │
│ 오버헤드: 5~10%                      │
│ 저장 용량: 1GB+/day                  │
│ 분석 가능: 완전한 분석                │
│ 권장: 개발 환경만                    │
└───────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | 설명 |
|------|------|
| **활성화** | `management.endpoints.web.exposure.include` |
| **주요 엔드포인트** | `/actuator/health`, `/actuator/metrics`, `/actuator/prometheus` |
| **메모리 메트릭** | `jvm.memory.used`, `jvm.memory.committed` |
| **GC 메트릭** | `jvm.gc.pause`, `jvm.gc.memory.*` |
| **스레드 메트릭** | `jvm.threads.live`, `jvm.threads.*` |
| **HTTP 메트릭** | `http.server.requests` (응답시간, 상태코드별) |
| **DB 메트릭** | `hikaricp.connections.*` |
| **수집 도구** | Prometheus |
| **시각화 도구** | Grafana (대시보드, 알람) |
| **수집 주기** | 5~15초 (Prometheus scrape_interval) |

---

## 🤔 생각해볼 문제

**Q1: 다음 Prometheus 쿼리를 작성하세요**

```
1. 지난 1시간 동안의 평균 Heap 메모리 사용량
2. 지난 1시간 동안의 HTTP 응답시간 p99
3. 지난 5분간의 활성 DB 커넥션 개수
4. 지난 1시간 동안 Full GC 횟수
```

<details>
<summary>💡 해설</summary>

**Prometheus 쿼리:**

```promql
# 1. 지난 1시간 동안의 평균 Heap 메모리 사용량
avg_over_time(jvm_memory_used_bytes{area="heap"}[1h])

# 또는 현재 값만
jvm_memory_used_bytes{area="heap"}

# 2. 지난 1시간 동안의 HTTP 응답시간 p99
histogram_quantile(0.99, 
  http_server_requests_seconds_bucket[1h]
)

# 또는 최근 시간의 p99
histogram_quantile(0.99, 
  rate(http_server_requests_seconds_bucket[1h])
)

# 3. 지난 5분간의 활성 DB 커넥션 개수
hikaricp_connections_active

# 4. 지난 1시간 동안 Full GC 횟수
increase(jvm_gc_count{action="end of major GC"}[1h])

# 또는
increase(jvm_gc_max_data_size_bytes[1h]) > 0 # Full GC 신호
```

</details>

---

**Q2: 다음 알람 규칙을 Prometheus alert로 작성하세요**

```
1. Heap 메모리 사용률 > 80% (5분 이상)
2. HTTP 응답시간 p99 > 500ms (10분 이상)
3. 활성 DB 커넥션 > 15개 (2분 이상)
4. Full GC 발생 > 1회/시간
```

<details>
<summary>💡 해설</summary>

**prometheus-rules.yml:**

```yaml
groups:
  - name: jvm_alerts
    interval: 30s
    rules:
      # 1. Heap 메모리 사용률 > 80%
      - alert: HighHeapMemoryUsage
        expr: |
          (jvm_memory_used_bytes{area="heap"} / 
           jvm_memory_max_bytes{area="heap"}) > 0.8
        for: 5m
        annotations:
          summary: "높은 Heap 메모리 사용률"
          description: "Heap 사용률: {{ $value | humanizePercentage }}"
      
      # 2. HTTP 응답시간 p99 > 500ms
      - alert: HighHttpLatency
        expr: |
          histogram_quantile(0.99, 
            rate(http_server_requests_seconds_bucket[5m])
          ) > 0.5
        for: 10m
        annotations:
          summary: "높은 HTTP 응답시간"
          description: "p99 latency: {{ $value | humanizeDuration }}"
      
      # 3. 활성 DB 커넥션 > 15
      - alert: HighDbConnections
        expr: hikaricp_connections_active > 15
        for: 2m
        annotations:
          summary: "높은 DB 커넥션 사용률"
          description: "활성 커넥션: {{ $value }}"
      
      # 4. Full GC > 1회/시간
      - alert: HighFullGCRate
        expr: |
          rate(jvm_gc_count{action="end of major GC"}[1h]) > 0.0167
        # 0.0167 = 1회/시간 in per-second rate
        for: 5m
        annotations:
          summary: "빈번한 Full GC"
          description: "1시간당 Full GC: {{ $value | humanize }}"
```

**Grafana Alert 설정 (UI 방식):**

```
1. Alert 탭 → Create Alert
2. Condition: jvm_memory_used_bytes / jvm_memory_max_bytes > 0.8
3. For: 5m
4. Message: "Heap memory usage is {{ $value | percent }}"
5. Notification: Slack, Email, PagerDuty 등
```

</details>

---

**Q3: 다음 메트릭에서 병목을 진단하세요**

```
시점 08:00:
- Heap Memory: 512MB (60% 사용)
- Active Threads: 45
- HTTP p99 Latency: 85ms
- Active DB Connections: 8
- GC Pause Time: 45ms

시점 08:05:
- Heap Memory: 650MB (80% 사용)
- Active Threads: 87
- HTTP p99 Latency: 250ms (3배 악화)
- Active DB Connections: 18
- GC Pause Time: 95ms

시점 08:10:
- Heap Memory: 900MB (95% 사용)
- Active Threads: 150
- HTTP p99 Latency: 800ms (10배 악화)
- Active DB Connections: 20 (최대)
- GC Pause Time: 340ms
```

병목이 뭔가요?

<details>
<summary>💡 해설</summary>

**단계별 분석:**

1. **시점 08:00 → 08:05 (5분 사이)**
   ```
   Heap: 512MB → 650MB (+138MB, +27%)
   Threads: 45 → 87 (+42, +93%)
   HTTP p99: 85ms → 250ms (3배)
   DB Conn: 8 → 18 (225%, 거의 가득 참)
   ```
   
   신호:
   - 스레드가 급증 (대기 중?)
   - Heap이 빠르게 증가
   - DB 커넥션 고갈 시작
   - 응답시간 급증
   
   원인: **데이터베이스 성능 저하**
   (DB가 느리면 커넥션이 해제되지 않음)

2. **시점 08:05 → 08:10 (다음 5분)**
   ```
   Heap: 650MB → 900MB (+250MB)
   Threads: 87 → 150 (+63, 누적 중)
   HTTP p99: 250ms → 800ms (더 악화)
   DB Conn: 18 → 20 (가득 찼음)
   GC Pause: 95ms → 340ms (4배 증가)
   ```
   
   진행 중:
   - Heap이 거의 가득 → GC 빈번 → STW 증가
   - DB 커넥션이 최대 (대기 요청 증가)
   - 스레드 계속 쌓임 (반응 못 함)
   
   악순환:
   ```
   DB 느림
      ↓
   커넥션 해제 안 됨
      ↓
   새 요청이 커넥션 대기
      ↓
   스레드 계속 증가
      ↓
   Heap 메모리 증가 (스레드 스택)
      ↓
   GC 빈번
      ↓
   응답시간 더 악화
      ↓
   DB 더 느려짐 (부하 증가)
   ```

**진단:**
1. **근본 원인: 데이터베이스 응답 지연**
   - DB 쿼리 분석 필요
   - 느린 쿼리 로그 확인
   - 인덱스 부족?
   - 연결 정체?

2. **즉시 조치:**
   ```
   # HikariCP 커넥션풀 키우기
   spring.datasource.hikari.maximum-pool-size=30
   
   # 타임아웃 설정
   spring.datasource.hikari.connection-timeout=5000
   
   # DB 쿼리 타임아웃
   spring.jpa.properties.hibernate.jdbc.fetch_size=50
   ```

3. **근본 해결:**
   - 느린 쿼리 최적화
   - 인덱스 추가
   - 캐싱 추가
   - DB 읽기 복제본 사용

</details>

---

<div align="center">

**[⬅️ 이전: JVM 튜닝 파라미터 — G1GC vs ZGC 선택](./05-jvm-tuning-parameters.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — HikariCP 튜닝 ➡️](../db-performance-tuning/01-hikaricp-tuning.md)**

</div>
