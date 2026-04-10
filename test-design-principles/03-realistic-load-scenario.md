# 03. 현실적 부하 시나리오 설계 — Access Log 기반

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 실제 트래픽 패턴을 k6 시나리오에 어떻게 반영하는가?
- Think Time이란 무엇이고, 없으면 결과가 어떻게 왜곡되는가?
- Access Log에서 Hot Spot 엔드포인트를 어떻게 추출하는가?
- Warm-up 단계가 없으면 측정 결과가 왜 신뢰할 수 없는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

부하 테스트가 의미 있으려면 실제 사용자 행동을 재현해야 한다. VU 100명이 초당 100번씩 단일 API를 호출하는 테스트는 실제 트래픽이 아니다. 실제 사용자는 페이지를 보고, 클릭하고, 폼을 작성하는 시간이 있다. 이 차이를 무시하면 테스트 결과가 실제 운영 환경과 전혀 다른 이야기를 한다.

---

## 😱 흔한 실수 (Before — 임의의 시나리오로 테스트하는 접근)

```
상황: 팀에서 부하 테스트를 처음 도입

테스트 설계:
  export default function () {
    http.get('http://localhost:8080/api/products');
    // Think Time 없음
    // 단일 엔드포인트만 테스트
    // VU 수는 "100명이면 충분하겠지"로 결정
  }

문제점:
  1. Think Time 없음 → VU 1명이 초당 수십 번 요청
     실제 사용자: 초당 0.5~1번 요청 (페이지 읽는 시간)
     → 실제의 20~30배 요청 압박, 결과 신뢰 불가

  2. 단일 엔드포인트 → 실제로는 상품 조회가 60%, 주문이 20%, 검색이 15%
     → 가장 느린 엔드포인트인 주문 API 병목을 발견 못 함

  3. VU 100명 임의 설정 → 실제 동시 사용자는 450명
     → 테스트 통과했지만 운영에서 장애

출시 후:
  실제 동시 사용자 200명 시점에서 이미 p99 2,000ms 돌파
  "테스트에서 문제 없었는데 왜 이러지?"
```

---

## ✨ 올바른 접근 (After — Access Log 기반 현실적 시나리오)

```
1단계: Access Log 분석 → 실제 트래픽 패턴 파악

$ awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

결과:
  18432  /api/products          (60.2%)  ← 상품 목록
   8201  /api/products/{id}     (26.8%)  ← 상품 상세
   3012  /api/search            (9.8%)   ← 검색
    812  /api/cart              (2.6%)   ← 장바구니
    213  /api/orders            (0.6%)   ← 주문 생성

2단계: 동시 사용자 수 계산 (Little's Law)

동시 사용자(L) = 요청 처리율(λ) × 평균 응답시간(W)
  피크 타임 TPS: 450 req/s
  평균 응답시간: 180ms = 0.18s
  → L = 450 × 0.18 = 81명 동시 처리 중
  → VU 목표: 81명 (여기에 Think Time 반영)

3단계: Think Time 포함 실제 VU 계산
  Think Time: 평균 2초 (사용자가 페이지 보는 시간)
  Cycle Time: 0.18s(응답) + 2s(Think) = 2.18s
  → 실제 필요 VU = 450 × 2.18 = 981명

4단계: 시나리오 가중치 반영
  products 목록: 60%, products 상세: 27%, 검색: 10%, 기타: 3%
```

---

## 🔬 내부 동작 원리

### Little's Law — 동시 사용자 수 계산

```
L = λ × W

L: 시스템 내 평균 요청 수 (≈ 동시 접속 VU)
λ: 평균 도착률 (TPS, 초당 요청 수)
W: 평균 체류 시간 (응답시간 + Think Time)

예시:
  피크 TPS = 300 req/s
  평균 응답시간 = 200ms
  Think Time = 3초

  W = 0.2 + 3 = 3.2초
  L = 300 × 3.2 = 960 VU

→ k6에서 VU 960으로 설정해야 현실적 부하 재현 가능
```

### Think Time의 영향

```
Think Time 없는 VU 100명:
  VU 1명 처리 사이클: 200ms (응답만)
  VU 1명 초당 요청 수: 5 req/s
  VU 100명 총 TPS: 500 req/s

Think Time 3초 포함한 VU 100명:
  VU 1명 처리 사이클: 200ms + 3000ms = 3200ms
  VU 1명 초당 요청 수: 0.31 req/s
  VU 100명 총 TPS: 31 req/s

→ Think Time 없이 VU 100명 = Think Time 포함 VU 1,600명과 동일한 압박
→ Think Time 없으면 테스트가 실제보다 16배 가혹함
```

### Warm-up 단계가 필요한 이유

```
JVM Cold Start 현상:
  서버 재시작 직후:
    JIT 컴파일 미완료 → 인터프리터로 실행 → 느림
    Connection Pool 미워밍업 → 초기 연결 생성 비용
    캐시 비어있음 → DB 직접 조회

  Warm-up 없이 바로 최대 부하 투입:
    첫 1~2분 측정값이 비정상적으로 느림
    → p99에 포함되어 전체 결과 왜곡

Warm-up 적용:
  stages: [
    { duration: '2m', target: 50 },    // Warm-up (결과 제외)
    { duration: '10m', target: 200 },  // 실제 측정 구간
    { duration: '1m', target: 0 },     // Cool-down
  ]
  → startVUs, gracefulStop으로 Warm-up 구간 결과 분리
```

---

## 💻 실전 실험 — Access Log 기반 k6 시나리오

### Access Log 분석 스크립트

```bash
#!/bin/bash
# access-log-analyzer.sh

LOG_FILE="access.log"

echo "=== 엔드포인트별 요청 비율 ==="
awk '{print $7}' $LOG_FILE \
  | sed 's/\/api\/products\/[0-9]*/\/api\/products\/{id}/g' \
  | sort | uniq -c | sort -rn \
  | awk '{total+=$1} END{for(i=1;i<=NR;i++) print $0}' \
  | head -10

echo ""
echo "=== 시간대별 TPS ==="
awk '{print $4}' $LOG_FILE \
  | cut -d: -f2 \
  | sort | uniq -c \
  | awk '{print $2":00 →", $1/60, "req/s"}'

echo ""
echo "=== 피크 타임 동시 사용자 추정 (Little's Law) ==="
PEAK_TPS=$(awk '{print $4}' $LOG_FILE | cut -d: -f2 | sort | uniq -c | sort -rn | head -1 | awk '{print $1/60}')
AVG_RESPONSE=$(awk '{sum+=$NF; count++} END{print sum/count/1000}' $LOG_FILE)
THINK_TIME=2
echo "Peak TPS: $PEAK_TPS"
echo "Avg Response: ${AVG_RESPONSE}s"
echo "Estimated VU: $(echo "$PEAK_TPS * ($AVG_RESPONSE + $THINK_TIME)" | bc)"
```

### 현실적 가중치 적용 k6 시나리오

```javascript
// k6/realistic-scenario.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

// Access Log 분석 기반 엔드포인트 가중치
const ENDPOINTS = [
  { url: '/api/products',      weight: 60, name: 'product_list'   },
  { url: '/api/products/1',    weight: 27, name: 'product_detail' },
  { url: '/api/search?q=shoe', weight: 10, name: 'search'         },
  { url: '/api/cart',          weight:  3, name: 'cart'           },
];

// 가중치 기반 엔드포인트 선택
function weightedRandom() {
  const total = ENDPOINTS.reduce((sum, e) => sum + e.weight, 0);
  let rand = Math.random() * total;
  for (const endpoint of ENDPOINTS) {
    rand -= endpoint.weight;
    if (rand <= 0) return endpoint;
  }
  return ENDPOINTS[0];
}

export const options = {
  scenarios: {
    realistic_load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 100  },  // Warm-up
        { duration: '5m', target: 500  },  // 정상 부하 (Little's Law 계산값)
        { duration: '3m', target: 800  },  // 피크 시뮬레이션
        { duration: '2m', target: 500  },  // 피크 후 안정화
        { duration: '1m', target: 0    },  // Cool-down
      ],
    },
  },
  thresholds: {
    'http_req_duration{name:product_list}':   ['p(99)<200'],
    'http_req_duration{name:product_detail}': ['p(99)<300'],
    'http_req_duration{name:search}':         ['p(99)<500'],
    'http_req_duration{name:cart}':           ['p(99)<400'],
    'http_req_failed': ['rate<0.001'],
  },
};

export default function () {
  const endpoint = weightedRandom();
  const BASE_URL = 'http://localhost:8080';

  const res = http.get(`${BASE_URL}${endpoint.url}`, {
    tags: { name: endpoint.name },
  });

  check(res, { 'status 200': (r) => r.status === 200 }) || errorRate.add(1);

  // Think Time: 실제 사용자처럼 1~3초 대기
  sleep(1 + Math.random() * 2);
}
```

### 주문 플로우 시나리오 (사용자 행동 재현)

```javascript
// k6/user-journey.js — 실제 사용자 구매 여정 시뮬레이션
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  scenarios: {
    // 80%는 탐색만, 20%는 실제 구매
    browse: {
      executor: 'constant-vus',
      vus: 160,
      duration: '10m',
      exec: 'browseOnly',
    },
    purchase: {
      executor: 'constant-vus',
      vus: 40,
      duration: '10m',
      exec: 'fullPurchase',
    },
  },
  thresholds: {
    'http_req_duration{name:order_create}': ['p(99)<1000'],
    'http_req_failed': ['rate<0.005'],
  },
};

// 탐색 시나리오 (구매 없음)
export function browseOnly() {
  group('상품 탐색', () => {
    http.get('http://localhost:8080/api/products', { tags: { name: 'product_list' } });
    sleep(2 + Math.random() * 3);

    http.get('http://localhost:8080/api/products/1', { tags: { name: 'product_detail' } });
    sleep(3 + Math.random() * 5); // 상세 페이지는 더 오래 봄
  });
}

// 구매 시나리오 (로그인 → 상품 → 장바구니 → 주문)
export function fullPurchase() {
  let token;

  group('로그인', () => {
    const res = http.post('http://localhost:8080/api/login',
      JSON.stringify({ username: 'user@test.com', password: 'password' }),
      { headers: { 'Content-Type': 'application/json' }, tags: { name: 'login' } }
    );
    check(res, { 'login success': (r) => r.status === 200 });
    token = res.json('token');
    sleep(1);
  });

  group('상품 조회', () => {
    http.get('http://localhost:8080/api/products/1', {
      headers: { Authorization: `Bearer ${token}` },
      tags: { name: 'product_detail' },
    });
    sleep(2 + Math.random() * 3);
  });

  group('주문 생성', () => {
    const res = http.post('http://localhost:8080/api/orders',
      JSON.stringify({ productId: 1, quantity: 2 }),
      {
        headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' },
        tags: { name: 'order_create' },
      }
    );
    check(res, {
      'order created': (r) => r.status === 201,
      'has order id':  (r) => r.json('id') !== undefined,
    });
    sleep(1);
  });
}
```

---

## 📊 성능 비교 — Think Time 유무에 따른 측정 결과 차이

### 동일 VU 100명, Think Time 유무 비교

| 지표 | Think Time 없음 | Think Time 2초 | 차이 |
|------|----------------|---------------|------|
| 실제 TPS | 480 req/s | 45 req/s | 10배 이상 |
| p99 응답시간 | 2,100ms | 180ms | 11배 차이 |
| CPU 사용률 | 92% | 18% | 5배 차이 |
| 테스트 의미 | 실제와 무관 | 실제 반영 | — |

→ Think Time 없는 테스트는 실제보다 10배 이상 가혹한 조건 → 결과 신뢰 불가

### Warm-up 유무에 따른 초기 측정 왜곡

| 경과 시간 | Warm-up 없음 p99 | Warm-up 포함 p99 |
|----------|----------------|----------------|
| 0~1분 | 1,850ms | (Warm-up 구간, 측정 제외) |
| 1~2분 | 920ms | 210ms |
| 2~5분 | 230ms | 195ms |
| 5~10분 | 185ms | 180ms |

→ Warm-up 없으면 초기 JIT/Cache miss가 p99를 심각하게 왜곡

---

## ⚖️ 트레이드오프

| 접근 | 장점 | 단점 |
|------|------|------|
| 단일 엔드포인트 | 구성 단순, 빠른 실행 | 실제 트래픽 패턴 미반영, 병목 오판 가능 |
| 가중치 기반 다중 엔드포인트 | 실제 반영, 병목 정확히 파악 | 스크립트 복잡도 증가 |
| 사용자 여정 시나리오 | 가장 현실적 | 구성 어려움, 상태(토큰 등) 관리 필요 |

---

## 📌 핵심 정리

- **Think Time 없으면 테스트 결과는 실제와 다르다** — 최소 `sleep(1)` 이상 포함
- **Access Log 분석**으로 Hot Spot 엔드포인트와 실제 트래픽 비율을 파악한 뒤 가중치 반영
- **Little's Law**로 VU 수를 계산하면 "몇 명이면 되겠지" 추측을 데이터로 대체 가능
- **Warm-up 2분** 이상 포함해야 JVM JIT, 커넥션 풀, 캐시가 안정화됨

---

## 🤔 생각해볼 문제

**Q1.** 피크 TPS가 300 req/s이고 평균 응답시간이 150ms, 사용자 Think Time이 평균 2.5초라면, k6에서 설정해야 할 VU는 몇 명인가?

<details>
<summary>해설 보기</summary>

Little's Law: L = λ × W  
W = 0.15s + 2.5s = 2.65s  
L = 300 × 2.65 = **795 VU**  
VU 800으로 설정하면 현실적인 피크 트래픽을 재현할 수 있다.

</details>

**Q2.** 상품 목록 API가 60%, 상품 상세가 30%, 주문이 10%일 때, k6에서 이를 반영하는 가장 간단한 방법은?

<details>
<summary>해설 보기</summary>

`Math.random()` 기반 가중치 선택 함수를 만들거나, k6의 `scenarios` 에 각 엔드포인트를 비율에 맞게 별도 시나리오로 구성하는 방법이 있다. 가장 직관적인 방법은 가중치 배열을 만들어 `weightedRandom()` 함수를 호출하는 것이다. 단순하게는 `if (rand < 0.6)` → products list, `else if (rand < 0.9)` → products detail, `else` → orders 로 분기해도 된다.

</details>

**Q3.** Warm-up 구간의 결과를 측정에서 제외하는 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

방법 1: Grafana에서 Warm-up 시간대를 시각적으로 제외하고 보기.  
방법 2: k6 `startTime` 옵션으로 측정 시작 시점을 지연시키거나, InfluxDB에 태그를 붙여 Warm-up 구간을 필터링.  
방법 3: Warm-up을 별도 스크립트로 분리해 실행한 뒤 본 테스트 스크립트 실행.

</details>

---

<div align="center">

**[⬅️ 이전: 성능 목표 정의 — SLI / SLO / SLA와 p99](./02-performance-goals-slo.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 환경 구성 — Prod-like 원칙 ➡️](./04-test-environment-setup.md)**

</div>
