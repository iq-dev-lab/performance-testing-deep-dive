<div align="center">

# 📊 Performance Testing Deep Dive

**"성능이 느리다고 느끼는 것과, 병목 지점을 수치로 특정하고 개선을 증명하는 것은 다르다"**

<br/>

> *"`p99 응답시간이 3초이고 병목은 Connection Pool 고갈이며, Pool 크기를 20→50으로 늘리면 p99가 340ms로 줄어든다" — 를 증명하는 레포"*

k6로 실제 트래픽 패턴을 재현하고, USE 방법론으로 CPU·DB·스레드 중 어디가 한계인지 데이터로 특정하며, async-profiler Flame Graph로 코드 레벨 병목을 찾고, 튜닝 전후를 p95/p99로 정량 비교하는 것까지  
**"체감상 빨라진 것 같다"가 아니라 수치로 증명하는** 성능 엔지니어링 레포입니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![k6](https://img.shields.io/badge/k6-Latest-7D64FF?style=flat-square&logo=k6&logoColor=white)](https://grafana.com/docs/k6/latest/)
[![Java](https://img.shields.io/badge/Java-21-007396?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io/projects/spring-boot)
[![Docs](https://img.shields.io/badge/Docs-39개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

성능 개선에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 설정하나"** 에서 멈춥니다.

| 일반 접근 | 이 레포 |
|----------|---------|
| "Connection Pool 크기를 늘리면 됩니다" | Pool 크기 공식(`Tn * (Cm - 1) + 1`), pending 메트릭으로 고갈 감지, 크기를 늘렸을 때 DB 서버가 받는 부하 변화까지 |
| "인덱스를 추가하면 빨라집니다" | `EXPLAIN ANALYZE` 실행 계획 해석, 인덱스 추가 전후 p95/p99 수치 비교, N+1 쿼리가 누적되면 Pool 고갈로 이어지는 인과관계 |
| "GC가 문제일 수 있어요" | `-Xlog:gc*`로 로그 수집, Stop-The-World 시간 측정, JFR + async-profiler Flame Graph로 어느 메서드가 GC 압박을 유발하는지 특정 |
| "k6로 부하 테스트 하면 됩니다" | `http_req_duration` 백분위수 해석, `thresholds`로 자동 합격/실패 판정, InfluxDB + Grafana 대시보드로 실시간 시각화 |
| "p99가 높습니다" | CPU·Memory·DB·Network 중 어디가 병목인지 USE 방법론으로 체계적 점검, Thread Dump로 BLOCKED 스레드 원인 특정 |
| "성능이 개선됐습니다" | 튜닝 전후 p50/p95/p99 비교표, TPS 변화, 에러율 변화, 인프라 비용 대비 개선 효과를 수치로 |
| 이론 설명 | 실행 가능한 k6 스크립트 + async-profiler + GC 로그 분석 + Docker Compose 실험 환경 |

---

## 📌 선행 학습 권장

이 레포는 다음 레포들과 시너지를 냅니다:

| 선행 레포 | 연결 지점 |
|----------|----------|
| **jvm-deep-dive** | GC 알고리즘, JIT 컴파일, 메모리 모델 — Chapter 4 JVM 프로파일링의 기반 |
| **linux-for-backend-deep-dive** | CPU, 메모리, I/O, 네트워크 시스템 메트릭 — Chapter 3 병목 특정의 기반 |
| **observability-deep-dive** | Prometheus, Grafana로 메트릭 수집 — k6 결과를 Grafana로 시각화하는 시너지 |
| **spring-data-transaction** | HikariCP Connection Pool, 트랜잭션 범위 — Chapter 5 DB 튜닝의 기반 |
| **database-internals** | 슬로우 쿼리, 인덱스 — Chapter 5 쿼리 최적화의 기반 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Chapter1](https://img.shields.io/badge/🔹_Ch1-성능_테스트_설계_원칙-7D64FF?style=for-the-badge)](./test-design-principles/01-test-types-classification.md)
[![Chapter2](https://img.shields.io/badge/🔹_Ch2-k6_완전_분해-7D64FF?style=for-the-badge)](./k6-deep-dive/01-k6-architecture.md)
[![Chapter3](https://img.shields.io/badge/🔹_Ch3-병목_특정_방법론-7D64FF?style=for-the-badge)](./bottleneck-identification/01-use-methodology.md)
[![Chapter4](https://img.shields.io/badge/🔹_Ch4-JVM_성능_프로파일링-7D64FF?style=for-the-badge)](./jvm-profiling/01-gc-log-analysis.md)
[![Chapter5](https://img.shields.io/badge/🔹_Ch5-DB_성능_튜닝-7D64FF?style=for-the-badge)](./db-performance-tuning/01-hikaricp-tuning.md)
[![Chapter6](https://img.shields.io/badge/🔹_Ch6-튜닝_사이클_보고-7D64FF?style=for-the-badge)](./tuning-cycle-reporting/01-scientific-tuning-approach.md)
[![Chapter7](https://img.shields.io/badge/🔹_Ch7-분산_환경_성능_테스트-7D64FF?style=for-the-badge)](./distributed-performance-testing/01-distributed-load-testing.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 성능 테스트 설계 원칙

> **핵심 질문:** 무엇을 측정해야 성능이 "개선됐다"고 말할 수 있는가? Load/Stress/Spike/Soak 테스트는 언제 쓰고, SLO를 평균이 아닌 p99로 정의해야 하는 이유는?

<details>
<summary><b>테스트 유형 분류부터 CI Pipeline 성능 회귀 감지까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 성능 테스트 유형 분류 — Load / Stress / Spike / Soak](./test-design-principles/01-test-types-classification.md) | 정상 부하(Load)·한계 탐색(Stress)·급증(Spike)·장시간 안정성(Soak) 테스트의 목적 차이, 각 유형별 k6 시나리오 설계 방법, 언제 어떤 테스트를 선택하는가 |
| [02. 성능 목표 정의 — SLI / SLO / SLA와 p99](./test-design-principles/02-performance-goals-slo.md) | SLI(측정 지표)·SLO(목표치)·SLA(계약) 차이, 평균 응답시간이 이상 징후를 숨기는 이유, Long Tail Latency와 p50/p95/p99 백분위수가 실제로 의미하는 바 |
| [03. 현실적 부하 시나리오 설계 — Access Log 기반](./test-design-principles/03-realistic-load-scenario.md) | 프로덕션 Access Log로 실제 트래픽 패턴 분석, Think Time 설정과 Hot Spot 엔드포인트 특정, Warm-up 단계가 없을 때 발생하는 측정 왜곡 |
| [04. 테스트 환경 구성 — Prod-like 원칙](./test-design-principles/04-test-environment-setup.md) | 빈 DB vs 1억 건 DB에서 성능 결과가 달라지는 이유, 네트워크·하드웨어 차이가 결과를 왜곡하는 방식, Docker Compose로 실험 환경 구성하기 |
| [05. 성능 기준선(Baseline) 수립 — 회귀 감지 자동화](./test-design-principles/05-baseline-and-regression.md) | 튜닝 전 기준 측정이 반드시 필요한 이유, CI Pipeline에 k6 Smoke Test 통합, 배포마다 자동으로 Baseline과 비교하는 전략 |

</details>

<br/>

### 🔹 Chapter 2: k6 완전 분해

> **핵심 질문:** k6는 어떻게 수백 명의 가상 사용자를 시뮬레이션하는가? `thresholds`로 자동 합격/실패 판정을 설정하고, InfluxDB + Grafana로 실시간 모니터링을 구성하는 방법은?

<details>
<summary><b>k6 아키텍처부터 전자상거래 실전 시나리오까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. k6 아키텍처 — Go 엔진과 VU 동시성](./k6-deep-dive/01-k6-architecture.md) | Go 기반 엔진이 JavaScript 시나리오를 실행하는 방식, VU(Virtual User)가 실제 사용자를 시뮬레이션하는 원리, 고루틴 기반 동시성이 수백 VU를 가볍게 처리하는 이유 |
| [02. k6 시나리오 설계 — Ramping / Constant / Spike 패턴](./k6-deep-dive/02-k6-scenario-design.md) | `options.scenarios`로 부하 패턴 구성, `stages`로 단계별 VU 제어, `thresholds`로 p95/p99·에러율 자동 합격/실패 판정 설정 |
| [03. HTTP 요청과 검증 — 로그인 토큰 재사용 패턴](./k6-deep-dive/03-http-requests-validation.md) | `http.get/post`, `check`로 응답 검증, 쿠키·세션 처리, 로그인 토큰을 이전 응답에서 추출해 다음 요청에 재사용하는 패턴 |
| [04. 결과 분석 — 히스토그램과 백분위수 해석](./k6-deep-dive/04-result-analysis.md) | `http_req_duration` 백분위수 읽기, `http_req_failed` 에러율, `vus_max` 동시 접속자, 히스토그램으로 응답 시간 분포 확인 및 이상 징후 감지 |
| [05. k6 확장 — InfluxDB + Grafana 대시보드 연동](./k6-deep-dive/05-k6-extensions.md) | `--out influxdb` 플래그로 결과 저장, Grafana 대시보드 구성, Custom Metrics(`new Counter/Gauge/Trend`) 정의, k6 Cloud 분산 실행 |
| [06. 실전 시나리오 — 전자상거래 주문 플로우 부하 테스트](./k6-deep-dive/06-real-world-scenarios.md) | 상품 조회 → 장바구니 → 결제 전체 플로우 시뮬레이션, 인증 포함 API 테스트, WebSocket 부하 테스트 패턴 |

</details>

<br/>

### 🔹 Chapter 3: 병목 특정 방법론

> **핵심 질문:** p99가 높을 때 CPU인가, 메모리인가, DB인가, 스레드인가? USE 방법론으로 체계적으로 후보를 좁히고, Thread Dump와 HikariCP 메트릭으로 실제 원인을 특정하는 방법은?

<details>
<summary><b>USE 방법론부터 네트워크 병목 분석까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. USE 방법론 — 병목 후보를 체계적으로 좁히는 법](./bottleneck-identification/01-use-methodology.md) | Utilization(사용률)·Saturation(포화도)·Errors(에러)로 CPU·Memory·Disk·Network를 점검하는 체계, 시스템 메트릭에서 병목 후보를 좁히는 순서 |
| [02. CPU 병목 분석 — User vs System vs IOWait 구분](./bottleneck-identification/02-cpu-bottleneck.md) | `top`, `mpstat`으로 CPU 사용률 확인, User·System·IOWait 시간 구분, CPU-Bound vs I/O-Bound를 판단하는 기준과 각각의 튜닝 방향 |
| [03. 메모리 병목 분석 — GC 로그로 메모리 누수 패턴 감지](./bottleneck-identification/03-memory-bottleneck.md) | JVM Heap vs Off-Heap 구분, GC 빈도와 Stop-The-World 시간이 응답시간에 미치는 영향, GC 로그에서 메모리 누수 패턴을 감지하는 방법 |
| [04. DB 병목 분석 — Connection Pool 고갈 감지](./bottleneck-identification/04-db-bottleneck.md) | HikariCP `pending` 메트릭으로 Pool 고갈 감지, 슬로우 쿼리 식별, `SHOW PROCESSLIST`로 Lock Contention 확인, Connection 대기 시간 측정 |
| [05. 외부 API 호출 병목 — 스레드 풀 고갈의 연쇄 장애](./bottleneck-identification/05-external-api-bottleneck.md) | Timeout 없는 외부 API 호출이 스레드 풀을 고갈시키는 원리, CircuitBreaker 없는 연쇄 장애 시나리오, 비동기 처리로 전환 판단 기준 |
| [06. 스레드 풀 분석 — Thread Dump로 BLOCKED 원인 특정](./bottleneck-identification/06-thread-pool-analysis.md) | Spring MVC 스레드 풀 고갈 감지, `jstack`으로 Thread Dump 수집, BLOCKED·WAITING 스레드의 원인을 특정하는 분석 패턴 |
| [07. 네트워크 병목 분석 — 직렬화 비용과 Keep-Alive](./bottleneck-identification/07-network-bottleneck.md) | 응답 크기 vs 응답시간 상관관계, TCP Connection 재사용(Keep-Alive)의 효과, DNS 조회 지연 측정, JSON 직렬화/역직렬화 비용 정량화 |

</details>

<br/>

### 🔹 Chapter 4: JVM 성능 프로파일링

> **핵심 질문:** GC 로그에서 무엇을 봐야 하는가? async-profiler Flame Graph에서 어느 메서드가 CPU를 독점하는지 어떻게 찾는가? JFR을 프로덕션에서 안전하게 사용하는 방법은?

<details>
<summary><b>GC 로그 분석부터 Spring Boot Actuator 메트릭까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. GC 로그 분석 — Stop-The-World 시간 측정](./jvm-profiling/01-gc-log-analysis.md) | `-Xlog:gc*` 옵션으로 GC 로그 수집, GC 유형별(Minor/Major/Full) Stop-The-World 시간 측정, GC 빈도가 과도한 원인(객체 생성 속도 vs 수집 속도) 분석 |
| [02. Java Flight Recorder(JFR) — 프로덕션 안전 사용법](./jvm-profiling/02-java-flight-recorder.md) | JVM 내부 이벤트 녹화(CPU 샘플링·메모리 할당·Lock 경합·I/O 대기), JDK Mission Control로 시각화, 오버헤드 1% 미만으로 프로덕션에서 사용하는 방법 |
| [03. async-profiler — Flame Graph로 코드 레벨 병목 찾기](./jvm-profiling/03-async-profiler.md) | CPU·메모리 할당·Lock 프로파일링, Flame Graph 읽는 법(넓을수록 CPU 비중 높음), 특정 메서드가 전체 CPU의 몇 %를 차지하는지 측정하는 실험 |
| [04. Heap Dump 분석 — 메모리 누수 원인 특정](./jvm-profiling/04-heap-dump-analysis.md) | `jmap -dump` 또는 Actuator로 Heap Dump 수집, MAT(Memory Analyzer Tool)로 메모리 누수 원인 특정, 대용량 객체 보유 체인(Retention Path) 추적 |
| [05. JVM 튜닝 파라미터 — G1GC vs ZGC 선택](./jvm-profiling/05-jvm-tuning-parameters.md) | G1GC vs ZGC 선택 기준(처리량 vs 저지연), `-Xms/-Xmx` 동일 설정 이유(힙 크기 재조정 비용), GC 튜닝의 트레이드오프 |
| [06. Spring Boot Actuator 성능 메트릭 — 핵심 지표 해석](./jvm-profiling/06-actuator-performance-metrics.md) | `/actuator/metrics/jvm.*`·`/actuator/metrics/hikaricp.*`·`/actuator/metrics/http.server.requests` 핵심 지표 해석, Prometheus 연동으로 시계열 수집 |

</details>

<br/>

### 🔹 Chapter 5: DB 성능 튜닝

> **핵심 질문:** HikariCP Pool 크기는 어떻게 계산하는가? EXPLAIN ANALYZE에서 Full Table Scan을 발견했을 때 어떤 인덱스를 추가해야 하는가? 캐시 도입 전후를 어떻게 정량 비교하는가?

<details>
<summary><b>HikariCP 튜닝부터 커서 기반 페이징까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HikariCP 튜닝 — Pool 크기 공식과 연결 비용](./db-performance-tuning/01-hikaricp-tuning.md) | Pool 크기 공식(`Tn * (Cm - 1) + 1`), `maximumPoolSize`가 클수록 좋지 않은 이유(DB 연결 비용, 컨텍스트 스위치), `connectionTimeout`·`idleTimeout` 설정 원칙 |
| [02. 쿼리 성능 최적화 — EXPLAIN ANALYZE와 N+1](./db-performance-tuning/02-query-optimization.md) | `EXPLAIN ANALYZE`로 실행 계획 해석, 인덱스 추가 전후 응답시간 비교, `p6spy`·`spring.jpa.show-sql`로 N+1 쿼리 감지, Fetch Join으로 해결 |
| [03. 커넥션 모니터링 — 고갈 시 에러 패턴 분석](./db-performance-tuning/03-connection-monitoring.md) | `SHOW STATUS LIKE 'Threads_connected'`, HikariCP `pending` 메트릭, 커넥션 풀 고갈 시 에러 폭발 패턴, 읽기 전용 트랜잭션 분리 효과 |
| [04. 캐시 전략과 효과 측정 — Redis 도입 전후 비교](./db-performance-tuning/04-cache-strategy-measurement.md) | Redis 캐시 도입 전후 DB 쿼리 수 비교, Cache Hit Rate 측정, 캐시 워밍업 전략, 캐시 무효화 버그 탐지 패턴 |
| [05. 페이징과 대용량 조회 최적화 — OFFSET의 성능 저하](./db-performance-tuning/05-pagination-optimization.md) | OFFSET 방식이 데이터 볼륨에 따라 얼마나 느려지는지 수치화, 커서 기반 페이징으로 전환 전후 비교, `JdbcTemplate.queryForStream`으로 스트리밍 처리 |

</details>

<br/>

### 🔹 Chapter 6: 튜닝 사이클과 결과 보고

> **핵심 질문:** 여러 변경을 동시에 적용하면 안 되는 이유는? 비기술 직군에게 성능 개선을 어떻게 설명하는가? CI Pipeline에 성능 회귀 감지를 통합하는 방법은?

<details>
<summary><b>과학적 튜닝 접근법부터 실전 케이스 스터디까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 과학적 튜닝 접근법 — 하나씩 변경하는 이유](./tuning-cycle-reporting/01-scientific-tuning-approach.md) | 다중 변경 시 원인 불명이 되는 이유, 가설 → 측정 → 결론 순서의 중요성, 튜닝 일지 작성법과 실험 기록 템플릿 |
| [02. 튜닝 효과 정량화 — p95/p99 비교표 작성](./tuning-cycle-reporting/02-quantifying-improvement.md) | 개선 전후 p50/p95/p99 비교표, TPS(처리량) 변화, 에러율 변화, 인프라 비용 대비 개선 효과를 수치로 표현하는 방법 |
| [03. 성능 테스트 결과 보고서 — 비기술 직군을 위한 작성법](./tuning-cycle-reporting/03-performance-report.md) | 개발자가 아닌 사람도 이해할 수 있는 보고서 구조, 개선 전후 시각화 그래프, 리스크와 추가 권장 사항 작성 방법 |
| [04. 성능 회귀 방지 — CI Pipeline 자동화](./tuning-cycle-reporting/04-performance-regression-prevention.md) | CI Pipeline에 k6 Smoke Test 통합, `thresholds` 초과 시 배포 차단, Baseline 자동 업데이트 전략, GitHub Actions 예시 |
| [05. 실전 케이스 스터디 — DB 커넥션 고갈부터 튜닝까지](./tuning-cycle-reporting/05-case-study.md) | 전자상거래 주문 API 병목 분석 전 과정: DB 커넥션 고갈 → 캐시 도입 → 인덱스 최적화 → 결과 측정, p99 3,200ms → 180ms로 줄인 실제 시나리오 |

</details>

<br/>

### 🔹 Chapter 7: 분산 환경 성능 테스트

> **핵심 질문:** 단일 머신으로 실제 프로덕션 트래픽을 재현하기 어려울 때 어떻게 하는가? Kafka Consumer Lag 시나리오와 JVM의 컨테이너 메모리 인식 문제는 어떻게 분석하는가?

<details>
<summary><b>분산 부하 테스트부터 Chaos Engineering까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 분산 부하 테스트 — k6 Operator로 Kubernetes 실행](./distributed-performance-testing/01-distributed-load-testing.md) | k6 Operator로 Kubernetes에서 분산 실행, 단일 머신 한계 극복, 분산 테스트 결과 집계 방법과 결과 해석 주의점 |
| [02. 마이크로서비스 성능 — 서비스 간 호출 체인 병목 특정](./distributed-performance-testing/02-microservice-performance.md) | 분산 추적(OpenTelemetry)으로 서비스 간 호출 체인의 병목 특정, 각 서비스의 독립적인 SLO 설정, 연쇄 장애 시뮬레이션 |
| [03. Kafka 성능 테스트 — Producer/Consumer 처리량 측정](./distributed-performance-testing/03-kafka-performance.md) | Producer·Consumer 처리량 측정, Consumer Lag 시나리오 재현, 파티션 수 변경 전후 처리량 비교, 배압(Backpressure) 탐지 |
| [04. 컨테이너 환경 성능 — JVM의 컨테이너 메모리 인식 문제](./distributed-performance-testing/04-container-performance.md) | JVM의 컨테이너 CPU·메모리 인식 문제(`UseContainerSupport`), K8s Resource Limit이 성능에 미치는 영향, HPA 트리거 시점과 스케일아웃 지연 |
| [05. Chaos Engineering 기초 — 의도적 장애 주입](./distributed-performance-testing/05-chaos-engineering.md) | Chaos Monkey for Spring Boot로 장애 주입, 성능 저하 시나리오에서 Circuit Breaker 동작 확인, 장애 주입 전후 p99 변화 측정 |

</details>

---

## 🔬 성능 병목 특정 흐름 (핵심 분석 예시)

```
k6 부하 테스트 실행
  → p99 응답시간 3,200ms (목표: 500ms)
  → 에러율 8% (목표: 1% 미만)
  │
  ▼ [1단계: 시스템 메트릭 확인 — USE 방법론]
  CPU: 45% → 병목 아님
  Memory: 75% → 의심
  DB 연결 대기: pending=18 / max=20 → 🎯 병목 후보

  ▼ [2단계: JVM 분석]
  GC 로그: Full GC 5초마다 → Stop-The-World 800ms
  Thread Dump: 18개 스레드 WAITING on HikariPool
  → Connection Pool 고갈 확인

  ▼ [3단계: DB 분석]
  SHOW PROCESSLIST: 20개 연결 모두 사용 중
  슬로우 쿼리: SELECT * FROM orders WHERE user_id=? → 0.8초
  EXPLAIN ANALYZE: Full Table Scan (인덱스 없음)

  ▼ [병목 원인 특정]
  1. user_id 인덱스 누락 → 쿼리 0.8초 (Full Table Scan)
  2. 느린 쿼리로 Connection 오래 점유
  3. Pool 고갈(max=20) → 대기 스레드 급증
  4. 높은 객체 생성률 → GC 압박으로 추가 지연

  ▼ [튜닝 적용 및 재측정]
  1. ALTER TABLE orders ADD INDEX idx_user_id (user_id)
     쿼리: 0.8s → 3ms
  2. maximumPoolSize: 20 → 30
  3. 재측정: p99 = 180ms ✓ | 에러율 = 0.02% ✓
```

> **p99 vs 평균 차이가 중요한 이유**
> - 평균 응답시간: 120ms → 정상처럼 보임
> - p99 응답시간: 3,200ms → 100명 중 1명은 3.2초를 기다림
> - 평균은 이상 징후를 숨기고, SLO는 반드시 백분위수(p95/p99)로 정의해야 한다

---

## 🛠️ 실험 환경

```yaml
# docker-compose.yml — 전체 실험 스택
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      JAVA_OPTS: >
        -Xms512m -Xmx512m
        -XX:+UseG1GC
        -Xlog:gc*:file=/logs/gc.log
        -XX:StartFlightRecording=filename=/logs/app.jfr,duration=60s

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: perf_test
    volumes:
      - ./docker/init-data.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
```

```bash
# 성능 분석 핵심 명령어 세트

# k6 실행 + InfluxDB 결과 저장
k6 run --out influxdb=http://localhost:8086/k6 k6/load-test.js

# JVM Thread Dump (스레드 상태 분석)
jstack $(jps | grep App | awk '{print $1}')

# async-profiler CPU 프로파일링 (60초)
./profiler.sh -d 60 -f flamegraph.html $(jps | grep App | awk '{print $1}')

# GC 로그 실시간 분석
tail -f /logs/gc.log | grep -E "GC|pause"

# HikariCP pending 연결 모니터링
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending

# MySQL 현재 실행 중인 쿼리 확인
mysql -e "SHOW FULL PROCESSLIST" | grep -v Sleep
```

---

## 📁 디렉토리 구조

```
performance-testing-deep-dive/
├── README.md
│
├── test-design-principles/          # 성능 테스트 설계 원칙 (5개)
├── k6-deep-dive/                    # k6 완전 분해 (6개)
├── bottleneck-identification/       # 병목 특정 방법론 (7개)
├── jvm-profiling/                  # JVM 성능 프로파일링 (6개)
├── db-performance-tuning/          # DB 성능 튜닝 (5개)
├── tuning-cycle-reporting/         # 튜닝 사이클과 결과 보고 (5개)
├── distributed-performance-testing/ # 분산 환경 성능 테스트 (5개)
│
├── k6/                             # k6 스크립트 모음
├── grafana/dashboards/             # Grafana 대시보드 설정
└── docker/                         # Docker Compose 실험 환경
```

**총 39개 문서**

---

## 📚 참고 자료

- [k6 공식 문서](https://grafana.com/docs/k6/latest/)
- [async-profiler GitHub](https://github.com/async-profiler/async-profiler)
- [Java Flight Recorder](https://docs.oracle.com/javacomponents/jmc.htm)
- [HikariCP Pool Sizing 가이드](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Google SRE Book — SLI/SLO/SLA](https://sre.google/sre-book/)
- [Brendan Gregg — Systems Performance](https://www.brendangregg.com/systems-performance.html) (USE 방법론, Flame Graph 저자)
- [Gatling 공식 문서](https://docs.gatling.io/)

---

<div align="center">

**"측정하지 않은 것은 개선할 수 없다"**

</div>
