# 02. 성능 목표 정의 — SLI / SLO / SLA와 p99

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SLI, SLO, SLA는 각각 무엇이고 어떻게 연결되는가?
- 평균 응답시간이 정상인데 사용자는 왜 느리다고 느끼는가?
- p95, p99 백분위수는 실제로 무엇을 의미하는가?
- k6에서 `thresholds`를 p99 기준으로 어떻게 설정하는가?
- Long Tail Latency란 무엇이고 왜 위험한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

목표 없이 테스트하면 결과를 해석할 수 없다. "응답시간 200ms"라는 결과를 보고 좋은지 나쁜지 판단하려면, 사전에 정의된 기준이 있어야 한다. 그 기준이 SLO다.

더 중요한 것은 **어떤 지표로 SLO를 정의하느냐**다. 평균 응답시간을 SLO 기준으로 쓰는 팀이 많지만, 이는 실제 사용자 경험과 일치하지 않는다. p99가 3,000ms여도 평균이 150ms일 수 있기 때문이다.

---

## 😱 흔한 실수 (Before — 평균으로 성능을 판단하는 접근)

```
상황: 신규 기능 배포 후 성능 모니터링 리뷰

대시보드 확인:
  평균 응답시간: 145ms → "정상"
  에러율: 0.02% → "정상"
  → 팀장: "성능 이상 없음, 배포 완료"

그날 밤 고객 CS 폭발:
  "결제 버튼 눌렀는데 5초 넘게 기다렸어요"
  "가끔 로딩이 너무 오래 걸려서 그냥 껐어요"

원인 분석:
  p99 응답시간: 4,800ms
  → 100명 중 1명은 매 요청마다 5초 가까이 대기
  → 하루 방문자 10만 명 → 1,000명이 매번 느린 경험
  → 평균은 145ms였지만 p99는 완전히 다른 이야기

교훈:
  평균은 소수의 극단적 느린 요청을 숨긴다.
  SLO를 평균으로 정의하면 실제 사용자 경험을 보호할 수 없다.
```

---

## ✨ 올바른 접근 (After — 백분위수 기반 SLO 정의)

```
올바른 SLO 정의 예시:

서비스: 전자상거래 주문 API

SLI (측정 지표):
  - http_req_duration (요청 응답시간)
  - http_req_failed (에러율)
  - http_reqs (처리량, TPS)

SLO (목표치, 내부 약속):
  - p95 응답시간 < 300ms  (95% 사용자는 300ms 이내)
  - p99 응답시간 < 800ms  (99% 사용자는 800ms 이내)
  - 에러율 < 0.1%
  - TPS > 200 req/s

SLA (계약, 외부 약속):
  - 월 가용성 99.9% (다운타임 월 43분 이내)
  - SLO가 지속적으로 위반되면 SLA 위반으로 이어짐

k6 thresholds 반영:
  thresholds: {
    'http_req_duration': ['p(95)<300', 'p(99)<800'],
    'http_req_failed': ['rate<0.001'],
  }
```

---

## 🔬 내부 동작 원리

### SLI → SLO → SLA 계층 구조

```
SLI (Service Level Indicator)
  ↓ 측정 대상 지표 자체
  예: "이번 달 API 응답시간의 p99값"

SLO (Service Level Objective)
  ↓ SLI에 대한 목표치 (내부 약속)
  예: "p99 < 500ms를 월간 99.5% 시간 동안 유지한다"

SLA (Service Level Agreement)
  ↓ SLO 기반의 외부 계약 (고객·파트너와의 약속)
  예: "SLO를 위반할 경우 서비스 크레딧 지급"

구조적 관계:
  SLA는 항상 SLO보다 느슨하게 설정한다.
  (SLO: p99 < 500ms / SLA: p99 < 800ms)
  → SLO를 지키면 SLA는 자동으로 달성됨
  → SLO는 내부 경고, SLA는 외부 약속의 최후 방어선
```

### 백분위수(Percentile)란 무엇인가

1,000개의 요청을 응답시간 순으로 줄 세웠을 때:

```
요청 1,000개를 응답시간 순으로 정렬:
  [5ms, 8ms, 12ms, ... , 145ms(평균 부근), ... , 480ms, 1,200ms, 4,800ms]

p50 (중앙값): 정렬했을 때 500번째 값 → 85ms
p95:          950번째 값 → 280ms   (5%는 이것보다 느림)
p99:          990번째 값 → 820ms   (1%는 이것보다 느림)
p99.9:        999번째 값 → 4,200ms (0.1%는 이것보다 느림)
평균:          모든 값의 합 / 1000  → 145ms
```

**평균이 145ms이지만 p99는 820ms인 이유**: 소수의 매우 느린 요청(4,800ms 등)이 평균을 끌어올리지 않고 p99에 영향을 준다.

### Long Tail Latency — 왜 발생하는가

```
Long Tail이 발생하는 주요 원인:

1. GC Stop-The-World
   → GC 발생 시 수백 ms 동안 모든 처리 중단
   → 이 시간에 요청이 들어오면 GC 시간만큼 응답 지연

2. DB Lock 경합
   → 특정 행을 잠근 트랜잭션이 느릴 때
   → 같은 행을 원하는 다른 요청들이 줄을 서서 대기

3. Connection Pool 고갈
   → Pool이 가득 차면 새 요청은 빈 커넥션이 생길 때까지 대기
   → 대기 시간이 p99를 끌어올림

4. JVM JIT Warm-up
   → 서버 재시작 직후 JIT 컴파일 전 인터프리터 실행으로 느림
   → 이 구간에 들어온 요청들이 p99에 기록됨

5. 외부 API Timeout
   → 외부 서비스 응답이 느릴 때 해당 요청만 극단적으로 느려짐
```

### Error Budget — SLO 기반 의사결정

```
Error Budget 개념:
  SLO: "월간 p99 < 500ms를 99.5% 시간 동안 유지"
  → 허용 위반 시간: 월 720시간 × 0.5% = 3.6시간

Error Budget 활용:
  - 남은 Budget이 충분: 새 기능 출시 적극적으로 진행
  - Budget 소진 임박: 안정성 작업 우선, 신규 기능 보류
  - Budget 소진: 배포 동결, 원인 분석 집중

핵심:
  SLO는 "얼마나 완벽해야 하는가"가 아니라
  "얼마나 위반을 허용할 것인가"를 정의하는 것이다.
  너무 엄격한 SLO(99.999%)는 오히려 혁신을 막는다.
```

---

## 💻 실전 실험 — k6 thresholds 설정

### 기본 백분위수 기반 SLO 설정

```javascript
// k6/slo-validation.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Counter } from 'k6/metrics';

const errorRate = new Rate('errors');
const slowRequests = new Counter('slow_requests'); // p99 초과 요청 수

export const options = {
  scenarios: {
    load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    // SLO 기반 threshold
    'http_req_duration': [
      'p(50)<100',   // 중앙값 100ms 이하
      'p(95)<300',   // 95% 300ms 이하
      'p(99)<800',   // 99% 800ms 이하
    ],
    'http_req_failed': ['rate<0.001'], // 에러율 0.1% 미만
    'slow_requests': ['count<50'],     // 1,000ms 초과 요청 50건 미만
  },
};

export default function () {
  const res = http.post(
    'http://localhost:8080/api/orders',
    JSON.stringify({ productId: 1, quantity: 2 }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'create_order' },
    }
  );

  const success = check(res, {
    'status 201': (r) => r.status === 201,
    'p99 이내': (r) => r.timings.duration < 800,
  });

  if (!success) errorRate.add(1);
  if (res.timings.duration > 1000) slowRequests.add(1);

  sleep(1);
}
```

### 엔드포인트별 개별 SLO 설정

```javascript
// k6/endpoint-slo.js — 엔드포인트마다 다른 SLO 적용
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 100,
  duration: '5m',
  thresholds: {
    // 상품 조회: 읽기 전용, 빠른 SLO
    'http_req_duration{name:product_detail}': ['p(99)<200'],
    // 주문 생성: 쓰기 + DB 처리, 느슨한 SLO
    'http_req_duration{name:create_order}': ['p(99)<1000'],
    // 결제: 외부 API 포함, 더 느슨한 SLO
    'http_req_duration{name:payment}': ['p(99)<3000'],
    // 전체 에러율
    'http_req_failed': ['rate<0.005'],
  },
};

export default function () {
  // 상품 조회
  http.get('http://localhost:8080/api/products/1', {
    tags: { name: 'product_detail' },
  });

  // 주문 생성
  http.post(
    'http://localhost:8080/api/orders',
    JSON.stringify({ productId: 1, quantity: 1 }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'create_order' },
    }
  );

  // 결제
  http.post(
    'http://localhost:8080/api/payments',
    JSON.stringify({ orderId: 1 }),
    {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: 'payment' },
    }
  );

  sleep(1);
}
```

### 응답시간 분포 시각화 (실행 후 결과 읽기)

```bash
# 실행
k6 run --out influxdb=http://localhost:8086/k6 k6/slo-validation.js

# k6 요약 출력 예시:
# http_req_duration............: avg=148ms min=12ms med=87ms
#                                 max=4812ms p(90)=220ms p(95)=310ms p(99)=890ms
# → p(99)=890ms > SLO(800ms) → threshold 위반 → 빌드 실패 처리됨

# Grafana에서 히스토그램으로 분포 확인
# Query: histogram_quantile(0.99, rate(http_req_duration_bucket[5m]))
```

---

## 📊 성능 비교 — 평균 vs 백분위수 실측 비교

### 같은 테스트, 다른 해석

| 지표 | 값 | 해석 |
|------|-----|------|
| 평균 응답시간 | 148ms | "정상" |
| p50 (중앙값) | 87ms | 절반의 사용자는 87ms |
| p95 | 310ms | 5% 사용자는 310ms 이상 대기 |
| p99 | 890ms | 1% 사용자는 900ms 가까이 대기 |
| p99.9 | 4,800ms | 0.1% 사용자는 5초 가까이 대기 |
| 최대값 | 12,400ms | 가장 불운한 사용자 경험 |

→ 하루 방문자 50만 명 기준:
- p99 초과 사용자: 5,000명이 매 요청마다 900ms+ 대기
- p99.9 초과 사용자: 500명이 매 요청마다 5초+ 대기

### SLO 달성률 예시 (1주일 측정)

| 날짜 | p99 | SLO(800ms) | 달성 |
|------|-----|-----------|------|
| 월 | 620ms | 800ms | ✅ |
| 화 | 750ms | 800ms | ✅ |
| 수 | 830ms | 800ms | ❌ |
| 목 | 910ms | 800ms | ❌ |
| 금 | 680ms | 800ms | ✅ |
| 토 | 540ms | 800ms | ✅ |
| 일 | 590ms | 800ms | ✅ |

→ 주간 SLO 달성률: 5/7 = 71.4% → SLO 목표치(99.5%)에 한참 미달

---

## ⚖️ 트레이드오프

| SLO 기준 | 장점 | 단점 |
|---------|------|------|
| 평균 기반 | 계산 단순, 이해 쉬움 | 실 사용자 경험 반영 안 됨, Long Tail 숨김 |
| p95 기반 | 대부분 사용자 커버, 합리적 | 상위 5%는 보호 안 됨 |
| p99 기반 | 거의 모든 사용자 커버 | 달성 어렵고, 인프라 비용 높아질 수 있음 |
| p99.9 기반 | 극단적 안정성 | 달성 매우 어려움, 과도한 엔지니어링 위험 |

**일반적 권장**: 사용자 facing API는 p99, 내부 배치 작업은 p95, 결제·인증 등 핵심 플로우는 p99 또는 p99.9.

---

## 📌 핵심 정리

- **SLI**: 측정 지표 자체 (응답시간, 에러율, 처리량)
- **SLO**: SLI에 대한 내부 목표치 (p99 < 500ms 등)
- **SLA**: SLO 기반의 외부 계약. 항상 SLO보다 느슨하게 설정
- **평균 응답시간은 SLO 기준으로 부적합하다** — Long Tail을 숨기기 때문
- **p99를 SLO 기준으로 쓰는 이유**: 100명 중 99명의 경험을 보호
- k6 `thresholds`에 `p(95)`, `p(99)` 기준을 명시하면 테스트가 자동으로 합격/실패 판정

---

## 🤔 생각해볼 문제

**Q1.** p99 응답시간 목표를 500ms로 설정했을 때와 200ms로 설정했을 때, 인프라와 코드에 요구되는 수준이 어떻게 달라지는가?

<details>
<summary>해설 보기</summary>

500ms는 DB 쿼리 1~2회 + 약간의 비즈니스 로직으로 달성 가능한 수준이다. 200ms는 캐시 적극 활용, 쿼리 최적화, 인덱스 설계가 필수적이며 커넥션 풀 대기 시간도 거의 없어야 한다. 목표를 낮출수록 달성 비용(인프라·엔지니어링)이 기하급수적으로 증가한다. SLO는 사용자 경험에 필요한 최소 수준으로 설정하는 것이 합리적이다.

</details>

**Q2.** SLO 달성률을 99.5%로 설정하면, 한 달(30일) 중 얼마의 시간 동안 SLO를 위반해도 되는가?

<details>
<summary>해설 보기</summary>

30일 × 24시간 × 60분 = 43,200분. 43,200 × 0.005 = **216분(3시간 36분)**이 허용 위반 시간(Error Budget)이다. 이 수치는 배포 빈도, 장애 대응 속도 등 팀의 운영 역량을 고려해 설정해야 한다. 신규 서비스는 99%에서 시작해 점진적으로 높이는 것이 현실적이다.

</details>

**Q3.** 같은 API인데 엔드포인트마다 SLO가 달라야 하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

각 엔드포인트의 처리 복잡도가 다르기 때문이다. 상품 조회(읽기 전용 + 캐시 가능)는 p99 200ms가 현실적이지만, 결제(외부 PG사 API 호출 포함)에 같은 기준을 적용하면 인프라 과잉 투자나 달성 불가 목표가 된다. 엔드포인트의 특성(읽기/쓰기, 외부 의존성, 비즈니스 중요도)을 고려해 개별적으로 설정하는 것이 바람직하다.

</details>

---

<div align="center">

**[⬅️ 이전: 성능 테스트 유형 분류](./01-test-types-classification.md)** | **[홈으로 🏠](../README.md)** | **[다음: 현실적 부하 시나리오 설계 ➡️](./03-realistic-load-scenario.md)**

</div>
