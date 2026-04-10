# 01. 성능 테스트 유형 분류 — Load / Stress / Spike / Soak

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Load / Stress / Spike / Soak 테스트는 각각 무엇을 측정하려는 것인가?
- 같은 API에 같은 k6 스크립트를 써도 목적이 다르면 왜 결과 해석이 달라지는가?
- "서버가 안 죽으면 OK"는 왜 성능 테스트 목표가 될 수 없는가?
- 각 테스트 유형에서 k6 `options`를 어떻게 구성해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

성능 테스트를 "부하를 넣어보는 것"으로만 정의하면, 어떤 문제를 찾으려는 것인지 불분명해진다. 테스트 결과가 나와도 "괜찮은 건지 아닌지"를 판단하지 못한다.

4가지 테스트 유형은 각각 서로 다른 질문에 답한다:

| 테스트 유형 | 핵심 질문 | 측정 대상 |
|------------|----------|----------|
| **Load** | "정상 트래픽에서 버티는가?" | 목표 부하에서의 응답시간·에러율 |
| **Stress** | "언제 무너지기 시작하는가?" | 한계(Breaking Point) TPS·에러 급증 시점 |
| **Spike** | "갑작스러운 급증을 회복하는가?" | 급증 중·급증 후 회복 시간 |
| **Soak** | "장시간 운영에서 무엇이 새는가?" | 메모리 누수·커넥션 고갈 누적 속도 |

이 구분 없이 "부하 테스트 통과"라고 말하는 것은, 달리기 선수가 100m를 뛴 결과로 마라톤 적합성을 판단하는 것과 같다.

---

## 😱 흔한 실수 (Before — 유형 구분 없이 테스트하는 접근)

```
상황: 출시 전 성능 테스트 완료 보고서 제출

테스트 내용:
  VU 50명, 5분간 부하, 에러율 0% → "성능 테스트 통과"

문제점:
  1. 정상 트래픽이 VU 50명인지 근거 없음
  2. 5분은 Soak에서 드러나는 메모리 누수를 발견하기 불충분
  3. Spike 없음 → 이벤트·프로모션 트래픽 급증 대응 여부 미검증
  4. Stress 없음 → 실제 한계치가 VU 51명인지 VU 500명인지 모름

출시 후 발생한 일:
  - 이벤트 시작 30초 만에 서버 응답 불가
  - 12시간 후 메모리 OOM으로 자동 재시작 반복
  → "성능 테스트 통과했는데 왜 죽지?"
```

---

## ✨ 올바른 접근 (After — 목적에 따라 유형을 선택하는 접근)

```
배포 전 성능 테스트 계획:

1. [Load Test]  정상 트래픽(VU 200) 30분 — SLO 달성 여부 확인
2. [Stress Test] VU 200→400→600→800 단계 증가 — Breaking Point 탐색
3. [Spike Test]  0→500→0 VU 급증 5분 — 이벤트 트래픽 시뮬레이션
4. [Soak Test]   VU 150, 4시간 — 메모리 누수·커넥션 고갈 탐지

각 테스트마다 목표 지표가 다름:
  Load  → p99 < 500ms, 에러율 < 0.1%
  Stress → Breaking Point VU 확인, 에러 첫 발생 시점
  Spike  → 급증 후 5분 내 p99 복구
  Soak   → 4시간 동안 힙 메모리 증가율 < 5%
```

---

## 🔬 내부 동작 원리

### Load Test — 정의된 부하에서의 안정성 검증

목표 트래픽 수준에서 SLO를 달성하는지 확인하는 테스트다. "정상 상태에서 잘 작동하는가"라는 가장 기본적인 질문이다.

```
부하 패턴:
  Warm-up(1분) → Sustained Load(5~30분) → Cool-down(1분)

VU 수:
  프로덕션 평균 동시 사용자 수 기준 (Access Log 분석 또는 APM 데이터)

합격 기준:
  - p95 응답시간 < 목표치 (예: 300ms)
  - 에러율 < 0.1%
  - TPS > 목표치 (예: 500 TPS)
```

### Stress Test — 한계 탐색

부하를 단계적으로 올려 시스템이 어느 시점에 한계를 드러내는지 측정한다. 이 테스트의 핵심은 **Breaking Point** — 에러율이 급증하거나 응답시간이 비선형적으로 증가하기 시작하는 VU 수를 찾는 것이다.

```
Breaking Point 탐색 패턴:
  VU 100 → 200 → 300 → 400 → 500 (각 단계 3분 유지)

관찰 지점:
  - 응답시간이 선형으로 증가하다가 갑자기 급등하는 구간
  - 에러율이 0%에서 5% 이상으로 넘어가는 VU 수
  - CPU·메모리·DB 연결 수 중 먼저 한계에 도달하는 자원
```

### Spike Test — 급격한 부하 변화 대응

갑작스러운 트래픽 급증이 발생했을 때 시스템이 살아남는지, 살아남는다면 얼마나 빨리 회복하는지를 측정한다. 이벤트 시작, 마케팅 메일 발송, 언론 보도 후 트래픽 급증 등의 시나리오를 재현한다.

```
핵심 측정 지표:
  - Spike 진입 시 에러율 (서버가 즉시 죽는가)
  - Spike 최고점에서의 p99 (얼마나 느려지는가)
  - Spike 후 정상 VU로 돌아왔을 때 회복 시간 (얼마나 걸려 정상화되는가)

회복 실패 패턴:
  - 스레드 풀이 가득 찬 채로 Spike가 끝나도 큐에 요청이 남아있어 회복 지연
  - DB 커넥션이 Spike 중 고갈된 후 Timeout으로 해제되기까지 대기
```

### Soak Test — 시간에 따른 누적 문제 탐지

낮은 부하를 장시간(수 시간~수십 시간) 유지하면서 시간이 지남에 따라 악화되는 문제를 탐지한다. 짧은 테스트로는 드러나지 않는 문제들이 대상이다.

```
Soak Test가 찾는 문제들:
  - 메모리 누수: 힙 사용량이 시간에 따라 단조 증가
  - 커넥션 누수: HikariCP 풀에서 반환되지 않는 커넥션 누적
  - 스레드 누수: Thread Pool 크기가 점점 증가
  - 캐시 팽창: 만료되지 않는 캐시로 메모리 고갈
  - 로그 파일 증가: 디스크 풀로 인한 쓰기 실패
```

---

## 💻 실전 실험 — k6 스크립트

### Load Test

```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 200 },  // Warm-up: 0 → 200 VU
        { duration: '10m', target: 200 }, // Sustained: 200 VU 유지
        { duration: '1m', target: 0 },    // Cool-down: 200 → 0
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<500'], // SLO
    errors: ['rate<0.001'],                         // 에러율 0.1% 미만
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/products/1', {
    tags: { name: 'product_detail' },
  });

  check(res, {
    'status 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  sleep(1); // Think time: 실제 사용자가 페이지를 보는 시간
}
```

### Stress Test

```javascript
// k6/stress-test.js
import http from 'k6/http';
import { check } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const responseTime = new Trend('response_time', true);

export const options = {
  scenarios: {
    stress: {
      executor: 'ramping-vus',
      stages: [
        { duration: '3m', target: 100 },  // 1단계
        { duration: '3m', target: 200 },  // 2단계
        { duration: '3m', target: 400 },  // 3단계
        { duration: '3m', target: 600 },  // 4단계 — Breaking Point 탐색
        { duration: '3m', target: 800 },  // 5단계
        { duration: '2m', target: 0 },    // 회복 관찰
      ],
    },
  },
  // Stress Test는 의도적으로 threshold를 느슨하게 설정
  // — 실패가 발생하는 시점을 관찰하는 것이 목적
  thresholds: {
    errors: ['rate<0.2'], // 20%까지는 계속 실행
  },
};

export default function () {
  const start = Date.now();
  const res = http.get('http://localhost:8080/api/orders', {
    tags: { name: 'order_list' },
  });
  responseTime.add(Date.now() - start);

  check(res, { 'status 200': (r) => r.status === 200 }) || errorRate.add(1);
}
```

### Spike Test

```javascript
// k6/spike-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    spike: {
      executor: 'ramping-vus',
      stages: [
        { duration: '30s', target: 50 },   // 정상 트래픽
        { duration: '10s', target: 500 },  // Spike 급증 (10배)
        { duration: '3m',  target: 500 },  // Spike 유지
        { duration: '10s', target: 50 },   // 급감
        { duration: '3m',  target: 50 },   // 회복 관찰 ← 핵심
      ],
    },
  },
  thresholds: {
    // Spike 후 회복 여부를 보기 위해 전체 기간 threshold는 느슨하게
    http_req_duration: ['p(95)<2000'],
    errors: ['rate<0.05'],
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/products', {
    tags: { name: 'product_list' },
  });
  check(res, { 'status ok': (r) => r.status === 200 }) || errorRate.add(1);
  sleep(0.5);
}
```

### Soak Test

```javascript
// k6/soak-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const heapUsage = new Trend('jvm_heap_mb'); // 커스텀 메트릭

export const options = {
  scenarios: {
    soak: {
      executor: 'constant-vus',
      vus: 150,
      duration: '4h', // 4시간 지속
    },
  },
  thresholds: {
    http_req_duration: ['p(99)<800'],
    errors: ['rate<0.01'],
  },
};

export default function () {
  // API 요청
  const res = http.get('http://localhost:8080/api/products/1');
  check(res, { 'status 200': (r) => r.status === 200 }) || errorRate.add(1);

  // JVM 힙 메트릭 주기적으로 수집 (Actuator 연동)
  if (__ITER % 100 === 0) {
    const metricsRes = http.get(
      'http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap'
    );
    if (metricsRes.status === 200) {
      const heapBytes = metricsRes.json('measurements.0.value');
      heapUsage.add(heapBytes / 1024 / 1024); // MB 단위
    }
  }

  sleep(1);
}
```

### 실행 명령어

```bash
# Load Test 실행 + InfluxDB 저장
k6 run --out influxdb=http://localhost:8086/k6 k6/load-test.js

# Stress Test (결과를 별도 태그로 구분)
k6 run --out influxdb=http://localhost:8086/k6 \
  --tag testType=stress \
  k6/stress-test.js

# Spike Test
k6 run --out influxdb=http://localhost:8086/k6 \
  --tag testType=spike \
  k6/spike-test.js

# Soak Test (백그라운드로 실행 후 로그 저장)
k6 run --out influxdb=http://localhost:8086/k6 \
  --tag testType=soak \
  k6/soak-test.js 2>&1 | tee soak-result.log
```

---

## 📊 성능 비교 — 각 테스트 유형별 실측 결과 예시

### Load Test 결과 (VU 200, 10분)

| 지표 | 측정값 | 목표(SLO) | 판정 |
|------|--------|----------|------|
| p50 응답시간 | 85ms | — | — |
| p95 응답시간 | 210ms | < 300ms | ✅ |
| p99 응답시간 | 380ms | < 500ms | ✅ |
| 에러율 | 0.02% | < 0.1% | ✅ |
| TPS | 187 req/s | > 150 | ✅ |

### Stress Test — Breaking Point 탐색 결과

| VU 수 | p99 응답시간 | 에러율 | 판정 |
|-------|------------|-------|------|
| 100 | 180ms | 0% | 정상 |
| 200 | 290ms | 0% | 정상 |
| 400 | 560ms | 0.3% | 주의 |
| 600 | 2,400ms | 8.2% | ⚠️ Breaking Point |
| 800 | 12,000ms+ | 34% | ❌ 장애 |

→ **Breaking Point: VU 400~600 구간** — 이 범위에서 DB 커넥션 풀 고갈 시작

### Spike Test — 회복 시간 측정

| 구간 | p99 응답시간 | 에러율 |
|------|------------|-------|
| 정상(VU 50) | 120ms | 0% |
| Spike 진입 직후(VU 500) | 3,800ms | 12% |
| Spike 최고점(VU 500, 2분 후) | 1,200ms | 3% |
| Spike 해소 후 1분(VU 50) | 280ms | 0.1% |
| **완전 회복(VU 50, 3분 후)** | **125ms** | **0%** |

→ 회복 시간: 약 3분 — 이 시간이 너무 길다면 스레드 풀 또는 DB 커넥션 큐 설정 검토

### Soak Test — 4시간 메모리 추이

| 경과 시간 | 힙 사용량 | p99 응답시간 |
|----------|----------|------------|
| 0분 | 180 MB | 310ms |
| 60분 | 220 MB | 315ms |
| 120분 | 280 MB | 320ms |
| 180분 | 410 MB | 355ms |
| 240분 | **680 MB** | **520ms** |

→ 힙 사용량이 4시간 동안 180 → 680 MB로 증가 = **메모리 누수 의심**  
→ async-profiler로 어느 객체가 해제되지 않는지 확인 필요

---

## ⚖️ 트레이드오프

| 테스트 유형 | 소요 시간 | 발견 가능한 문제 | 발견 불가 |
|------------|---------|---------------|---------|
| Load | 30분~1시간 | SLO 달성 여부, 정상 운영 병목 | 한계치, 메모리 누수 |
| Stress | 30분~1시간 | Breaking Point, 자원 한계 | 장시간 누적 문제 |
| Spike | 20~40분 | 급증 대응, 회복 속도 | 장시간 누적 문제 |
| Soak | 4~24시간 | 메모리·커넥션 누수, 장기 안정성 | Breaking Point |

**Soak Test는 배포 직전 매번 실행하기 어렵다.** 최소한 메이저 릴리즈 전, 또는 메모리 관련 코드 변경 후에는 반드시 실행해야 한다.

---

## 📌 핵심 정리

- **Load Test**: 평상시 트래픽에서 SLO 달성 여부 확인. 가장 기본이며 매 배포마다 실행.
- **Stress Test**: 시스템의 실제 한계(Breaking Point)를 데이터로 특정. 결과는 운영 용량 계획에 활용.
- **Spike Test**: 이벤트·프로모션처럼 예측 가능한 급증 시나리오 사전 검증. 회복 시간 측정이 핵심.
- **Soak Test**: 장시간 운영에서 드러나는 누수성 문제 탐지. 메이저 릴리즈 전 필수.
- 4가지 테스트는 서로 다른 질문에 답한다. 하나로 나머지를 대체할 수 없다.

---

## 🤔 생각해볼 문제

**Q1.** Stress Test에서 VU 600에서 에러가 급증했다면, 운영에서 허용 가능한 최대 VU 수는 얼마로 설정해야 할까? 그 근거는?

<details>
<summary>해설 보기</summary>

Breaking Point가 VU 600이라면 안전 마진을 고려해 **VU 400 수준을 운영 최대치**로 잡는 것이 합리적이다. 일반적으로 Breaking Point의 60~70% 수준을 운영 상한으로 설정한다. 이 수치를 넘는 트래픽이 예상된다면 수평 확장(Scale-out) 또는 Breaking Point 자체를 높이는 튜닝이 필요하다.

</details>

**Q2.** Soak Test 4시간 동안 힙 메모리가 180 → 680 MB로 증가했다. 이것이 반드시 메모리 누수를 의미하는가? 누수가 아닐 수 있는 경우는?

<details>
<summary>해설 보기</summary>

반드시 누수는 아니다. 다음 경우에는 정상적인 힙 증가일 수 있다: ①캐시가 점진적으로 채워지는 정상 동작(Ehcache, Caffeine의 최대 크기 도달 전), ②JVM이 GC를 늦게 수행하는 것(G1GC의 배치 전략), ③초기에 적은 데이터로 시작했다가 실제 운영 볼륨에 수렴하는 과정. 확인 방법: GC 이후에도 힙이 줄어들지 않는다면 누수다. `jmap -histo`로 어느 클래스의 인스턴스가 계속 증가하는지 확인한다.

</details>

**Q3.** CI Pipeline에 4가지 테스트를 모두 넣으면 좋을까? 현실적인 전략은?

<details>
<summary>해설 보기</summary>

모든 PR에 Soak Test(4시간)를 돌리는 것은 현실적으로 불가능하다. 현실적인 전략: ①**PR마다**: Load Test의 경량 버전(Smoke Test) — 5분, VU 20으로 최소 동작 확인. ②**배포 브랜치**: 완전한 Load Test + Stress Test. ③**메이저 릴리즈 전·주기적(주 1회)**: Soak Test. Spike Test는 이벤트·프로모션이 있는 배포 전에 선택적으로 실행한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 성능 목표 정의 — SLI / SLO / SLA와 p99 ➡️](./02-performance-goals-slo.md)**

</div>
