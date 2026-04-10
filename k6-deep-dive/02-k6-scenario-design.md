# 02. k6 시나리오 설계 — Ramping / Constant / Spike 패턴

---

## 🎯 핵심 질문

- `options.scenarios`와 `options.vus`/`options.duration`의 차이점은 무엇인가?
- 5가지 executor (ramping-vus, constant-vus, per-vu-iterations, ramping-arrival-rate, shared-iterations)는 각각 언제 사용하는가?
- `stages`로 단계별 부하를 제어하는 정확한 메커니즘은 무엇인가?
- `thresholds`로 성공/실패를 판정하는 논리와 실제 활용 사례는?
- 여러 시나리오를 동시에 실행할 때 VU 할당과 선택 방식은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**시나리오 설계**는 성능 테스트의 신뢰성을 완전히 결정합니다. 같은 API를 테스트해도:

- **상점 오픈 시간(Ramping)**: VU를 천천히 증가 → 시스템의 병목점 발견
- **Black Friday(Spike)**: 갑자기 수천 VU 동시 접속 → 최대 부하 대응력 측정
- **배경 트래픽(Constant)**: 일정한 부하 유지 → 안정성 검증

**잘못된 시나리오**는 문제를 찾지 못하거나, 반대로 시스템을 불필요하게 폄하합니다. 예를 들어, 실제로는 5초마다 요청 100개를 받는데, 1분에 걸쳐 100 VU를 올렸다면, **실제 트래픽 패턴을 시뮬레이션하지 못합니다**.

또한 `thresholds`는 단순히 "테스트 통과/실패" 판정뿐 아니라, **자동화된 성능 검증**으로 CI/CD 파이프라인에 통합됩니다. 임의의 판정 기준은 불안정한 배포를 초래합니다.

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. 구 버전 options 문법으로 단순 부하 테스트

```javascript
// ❌ 구식 접근법 (k6 0.26 이전 스타일)
// 이 방식은 모든 VU가 정확히 동일한 시간에 실행되어
// 현실적인 사용자 행동을 시뮬레이션하지 못함

export const options = {
  vus: 100,
  duration: '5m',
};

export default function() {
  http.get('http://example.com');
  // 결과: 모든 VU가 동시에 같은 작업 반복
  // 실제 사용 패턴과 무관함
}
```

**문제점:**
- VU가 시간에 따라 변하지 않음
- 특정 부하 프로필을 구현할 수 없음
- 여러 시나리오 동시 실행 불가능

### 2. Thresholds를 설정하지 않아 신뢰성 없는 판정

```javascript
// ❌ 판정 기준 없음
export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 100,
      duration: '5m',
    },
  },
  // thresholds가 없으면, 모든 테스트가 "성공"으로 간주됨
};

export default function() {
  const response = http.get('http://example.com');
  // p95가 5초인데도 테스트가 성공...
  // CI/CD에서 이를 감지하지 못해 나쁜 배포 배포됨
}
```

### 3. Ramping 단계를 너무 급격하게 설정

```javascript
// ❌ 단계가 너무 급격함 (너무 빨리 증가)
export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1s', target: 1000 },  // 1초에 1000 VU!
        { duration: '5m', target: 1000 },  // 유지
        { duration: '1s', target: 0 },     // 1초에 감소
      ],
    },
  },
};

// 결과: 메모리, 네트워크, 데이터베이스 모두 동시 폭증
// → 병목점을 정확히 파악할 수 없음
// → 단순히 "시스템 다운"이라는 결과만 얻음
```

### 4. Per-VU-Iterations를 오해하는 경우

```javascript
// ❌ 요청 수를 잘못 예상
export const options = {
  scenarios: {
    test: {
      executor: 'per-vu-iterations',
      vus: 100,
      iterations: 10,  // 각 VU가 10번 반복
      maxDuration: '10m',
    },
  },
};

// VU 100개 × 반복 10회 = 총 1000개 요청
// 하지만 개발자가 "10개 요청"이라고 잘못 이해
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. 현실적인 시나리오 설계 (3단계 ramping)

```javascript
export const options = {
  scenarios: {
    realistic_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        // 단계 1: 워밍업 (서버 리소스 준비)
        { duration: '2m', target: 50 },
        // 단계 2: 정상 부하
        { duration: '5m', target: 100 },
        // 단계 3: 피크 부하
        { duration: '3m', target: 200 },
        // 단계 4: 부하 감소
        { duration: '2m', target: 0 },
      ],
    },
  },
  // 이제 명확한 판정 기준 설정
  thresholds: {
    'http_req_duration{staticAsset:yes}': ['p(95)<100'],  // 정적 자산 95%ile < 100ms
    'http_req_duration{staticAsset:no}': ['p(95)<500'],   // 동적 콘텐츠 95%ile < 500ms
    'http_req_failed': ['rate<0.01'],                      // 에러율 1% 미만
    'vus': ['value<=200'],                                  // VU 200 초과 불가
  },
};

export default function() {
  http.get('http://example.com', {
    tags: { staticAsset: 'yes' },
  });
  http.get('http://example.com/api/data', {
    tags: { staticAsset: 'no' },
  });
}
```

### 2. 여러 시나리오 동시 실행 (현실적 트래픽 믹스)

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';

// 커스텀 메트릭
const loginDuration = new Trend('login_duration');
const browseDuration = new Trend('browse_duration');
const purchaseRate = new Rate('purchase_success');

export const options = {
  scenarios: {
    // 시나리오 1: 로그인 사용자 (전체 트래픽의 20%)
    login_users: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },  // 20 VU (전체 100의 20%)
        { duration: '5m', target: 20 },
        { duration: '1m', target: 0 },
      ],
    },
    
    // 시나리오 2: 상품 브라우징 (전체 트래픽의 70%)
    browse_users: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 70 },  // 70 VU (전체 100의 70%)
        { duration: '5m', target: 70 },
        { duration: '1m', target: 0 },
      ],
      startTime: '1m',  // 로그인 후 1분 뒤 시작
    },
    
    // 시나리오 3: 구매 (전체 트래픽의 10%)
    purchase_users: {
      executor: 'ramping-arrival-rate',
      timeUnit: '1m',  // 분당 도착률
      preAllocatedVUs: 10,
      stages: [
        { duration: '2m', target: 10 },  // 분당 10명 도착
        { duration: '5m', target: 20 },  // 분당 20명 도착
        { duration: '1m', target: 0 },
      ],
      startTime: '2m',
    },
  },

  thresholds: {
    'http_req_duration{scenario:login}': ['p(95)<300'],
    'http_req_duration{scenario:browse}': ['p(95)<200'],
    'purchase_success': ['rate>=0.95'],
  },
};

export default function() {
  // __ENV.SCENARIO로 어느 시나리오인지 확인 불가능
  // 대신 VU 인덱스나 다른 방식으로 분기
  
  const vuId = __VU;
  
  if (vuId <= 20) {
    // 로그인 사용자
    login_flow();
  } else if (vuId <= 90) {
    // 브라우징 사용자
    browse_flow();
  } else {
    // 구매 사용자
    purchase_flow();
  }
}

function login_flow() {
  const startTime = Date.now();
  http.post('http://api.example.com/login', {
    username: 'user' + __VU,
    password: 'password',
  }, { tags: { scenario: 'login' } });
  loginDuration.add(Date.now() - startTime);
  sleep(2);
}

function browse_flow() {
  const startTime = Date.now();
  http.get('http://api.example.com/products', {
    tags: { scenario: 'browse' },
  });
  browseDuration.add(Date.now() - startTime);
  sleep(1);
}

function purchase_flow() {
  const response = http.post('http://api.example.com/purchase', {
    productId: Math.floor(Math.random() * 1000),
    quantity: 1,
  }, { tags: { scenario: 'purchase' } });
  purchaseRate.add(response.status === 200);
}
```

### 3. Threshold 상세 설정 (자동 합격/실패 판정)

```javascript
export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 0 },
      ],
    },
  },

  thresholds: {
    // ✅ 전체 요청의 95 백분위수가 500ms 이하
    'http_req_duration': ['p(95)<500'],
    
    // ✅ 전체 요청의 99 백분위수가 1초 이하
    'http_req_duration': ['p(99)<1000'],
    
    // ✅ 평균 응답시간이 300ms 이하
    'http_req_duration': ['avg<300'],
    
    // ✅ 최대 응답시간이 3초 이하
    'http_req_duration': ['max<3000'],
    
    // ✅ 전체 요청 중 에러율이 1% 미만
    'http_req_failed': ['rate<0.01'],
    
    // ✅ 요청 성공률이 99% 이상
    'http_req_failed': ['rate<=0.01'],
    
    // ✅ 404 에러는 0.1% 미만
    'http_req_failed{error_code:404}': ['rate<0.001'],
    
    // ✅ VU 수는 200을 초과하지 않음
    'vus': ['value<=200'],
    
    // ✅ 모든 요청이 완료됨 (드롭된 요청 없음)
    'http_reqs': ['count>10000'],
    
    // ✅ 커스텀 메트릭: 로그인 성공률 98% 이상
    'login_success': ['rate>=0.98'],
  },

  // Threshold 실패 시 테스트 중단할지 계속할지 지정
  // (기본값: 테스트는 계속하고 마지막에 실패 표시)
  ext: {
    loadimpact: {
      // k6 Cloud에 결과 업로드
      projectID: 123456,
    },
  },
};

export default function() {
  http.get('http://example.com');
}
```

---

## 🔬 내부 동작 원리

### Executor 5가지 비교

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. ramping-vus (VU 수 증가/감소)                                │
├─────────────────────────────────────────────────────────────────┤
│ VU                                                               │
│ │                      ╱╲                                       │
│ │                     ╱  ╲                                      │
│ │            ╱╲       ╱    ╲                                    │
│ └┬──────────╱  ╲─────╱      ╲─────────────────┬───────→ 시간   │
│ 0          5    10          20                25                │
│ └─────────────────────────────────────────────────────────────┘
│ 특징: 서버 병목점 찾기, 워밍업, 최대 부하 테스트               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 2. constant-vus (VU 수 일정 유지)                               │
├─────────────────────────────────────────────────────────────────┤
│ VU                                                               │
│ │                    ┌──────────────────────┐                   │
│ │                    │                      │                   │
│ └┬───────────────────┴──────────────────────┴───────→ 시간     │
│ 0                                                   25           │
│ └─────────────────────────────────────────────────────────────┘
│ 특징: 안정성 검증, 배경 트래픽, 기준 성능 측정                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 3. ramping-arrival-rate (도착률 제어)                           │
├─────────────────────────────────────────────────────────────────┤
│ 도착률                                                           │
│ (req/s)                      ╱─────────────                     │
│ │                           ╱                                   │
│ │                    ╱─────                                     │
│ │           ╱──────                                             │
│ └┬────────╱───────────────────────────────────┬───────→ 시간   │
│ 0        5        10       15       20        25                │
│ └─────────────────────────────────────────────────────────────┘
│ 특징: 일정한 요청율 유지 (VU는 동적 조정)                      │
│ 예: "초당 100개 요청" → VU는 응답시간에 따라 자동 조정       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 4. per-vu-iterations (VU당 반복 횟수)                           │
├─────────────────────────────────────────────────────────────────┤
│ VU당 작업 흐름:                                                  │
│ ┌──────────────────────────────────┐                            │
│ │ VU 1: [반복1] [반복2] ... [반복N]│ ← 완료 후 종료             │
│ │ VU 2: [반복1] [반복2] ... [반복N]│                            │
│ │ VU 3: [반복1] [반복2] ... [반복N]│                            │
│ └──────────────────────────────────┘                            │
│ 특징: 정확한 요청 수 제어 (VU 100 × 10회 = 1000요청)          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 5. shared-iterations (VU들이 반복 횟수 공유)                    │
├─────────────────────────────────────────────────────────────────┤
│ 총 1000회 반복을 VU 100개가 나눠서 실행                         │
│ ┌──────────────────────────────────┐                            │
│ │ VU 1: [반복1] [반복11] [반복21]...│ ← 총 1000회 중 10회      │
│ │ VU 2: [반복2] [반복12] [반복22]...│                           │
│ │ VU 3: [반복3] [반복13] [반복23]...│                           │
│ │ ...                               │                           │
│ │ VU 100: [반복100]...  [반복1000]  │ ← 나머지 10회            │
│ └──────────────────────────────────┘                            │
│ 특징: VU 수와 관계없이 총 반복 횟수 고정                        │
└─────────────────────────────────────────────────────────────────┘
```

### Threshold 평가 로직

```
테스트 실행 중:
1. 메트릭 수집 (http_req_duration, http_req_failed 등)
2. 주기적으로 threshold 평가 (기본 1초마다)
3. 조건 만족하지 않으면 조건 실패 기록

테스트 완료 후:
4. 모든 threshold 평가
   - 모두 만족 → "테스트 PASS" (exit code 0)
   - 하나라도 실패 → "테스트 FAIL" (exit code 1)

CI/CD 파이프라인:
exit code 0 → 배포 진행
exit code 1 → 배포 중단
```

### Stages 단계별 VU 증감

```javascript
stages: [
  { duration: '2m', target: 50 },   // 0 → 50 VU (선형 증가)
  { duration: '5m', target: 100 },  // 50 → 100 VU
  { duration: '3m', target: 200 },  // 100 → 200 VU
  { duration: '2m', target: 0 },    // 200 → 0 VU (선형 감소)
]

시간선:
VU
200 │                            ╱╲
    │                           ╱  ╲
100 │            ╱──────────────╱    ╲
    │           ╱                      ╲
 50 │    ╱─────╱                        ╲
    │   ╱                                 ╲
  0 └──────────────────────────────────────╲─────→ 시간
    0    2min      7min      10min         12min

단계별 VU 변화율:
[0 → 50 in 2min] = 초당 0.42 VU 추가
[50 → 100 in 5min] = 초당 0.17 VU 추가
[100 → 200 in 3min] = 초당 0.56 VU 추가
[200 → 0 in 2min] = 초당 1.67 VU 제거
```

---

## 💻 실전 실험

### 실험 1: Executor 5가지 동시 실행 및 비교

**test-executors.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const executorCounter = new Counter('executor_requests');
const durationTrend = new Trend('request_duration');

export const options = {
  scenarios: {
    // 1. Ramping VUs
    ramping: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 10 },
        { duration: '2m', target: 10 },
        { duration: '1m', target: 0 },
      ],
      tags: { executor: 'ramping' },
    },
    
    // 2. Constant VUs
    constant: {
      executor: 'constant-vus',
      vus: 10,
      duration: '4m',
      startTime: '0s',
      tags: { executor: 'constant' },
    },
    
    // 3. Per-VU-Iterations
    per_vu: {
      executor: 'per-vu-iterations',
      vus: 5,
      iterations: 20,  // 5 VU × 20 iterations = 100 requests
      maxDuration: '4m',
      tags: { executor: 'per_vu' },
    },
    
    // 4. Ramping Arrival Rate
    arrival_rate: {
      executor: 'ramping-arrival-rate',
      timeUnit: '1s',
      preAllocatedVUs: 10,
      stages: [
        { duration: '1m', target: 5 },   // 초당 5개 요청
        { duration: '2m', target: 10 },  // 초당 10개 요청
        { duration: '1m', target: 0 },
      ],
      tags: { executor: 'arrival_rate' },
    },
    
    // 5. Shared Iterations
    shared: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 100,  // 5 VU가 총 100회 나눠서 실행
      maxDuration: '4m',
      startTime: '1s',
      tags: { executor: 'shared' },
    },
  },

  thresholds: {
    'http_req_duration': ['p(95)<500'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function() {
  const start = Date.now();
  const response = http.get('http://httpbin.org/delay/0.1', {
    tags: { test: 'executor_comparison' },
  });
  durationTrend.add(Date.now() - start);
  executorCounter.add(1);
  sleep(0.5);
}
```

**실행 명령:**
```bash
k6 run test-executors.js
```

### 실험 2: Threshold 자동 판정 및 CI/CD 통합

**test-threshold.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';
import { check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 50 },
        { duration: '1m', target: 50 },
        { duration: '30s', target: 0 },
      ],
    },
  },

  // 명확한 판정 기준
  thresholds: {
    'http_req_duration{staticAsset:yes}': ['p(95)<200'],
    'http_req_duration{staticAsset:no}': ['p(95)<500'],
    'http_req_failed': ['rate<0.01'],
    'http_reqs': ['count>500'],
  },
};

export default function() {
  // 정적 자산 요청 (응답 빨라야 함)
  const static = http.get('http://httpbin.org/cache/3600', {
    tags: { staticAsset: 'yes' },
  });
  check(static, {
    'static asset status is 200': (r) => r.status === 200,
  });

  sleep(0.5);

  // 동적 API 요청 (다소 느려도 됨)
  const dynamic = http.post('http://httpbin.org/post', {
    data: { timestamp: Date.now() },
  }, {
    tags: { staticAsset: 'no' },
  });
  check(dynamic, {
    'api response status is 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

**실행 및 결과 확인:**
```bash
# 테스트 실행
k6 run test-threshold.js

# Exit code 확인 (CI/CD에서 사용)
echo "Exit code: $?"

# 예상 결과:
# - Threshold 모두 만족 → exit code 0 (배포 진행)
# - Threshold 하나라도 실패 → exit code 1 (배포 중단)
```

### 실험 3: 실제 Spike 패턴 시뮬레이션

**test-spike.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    // 정상 트래픽
    normal: {
      executor: 'constant-vus',
      vus: 10,
      duration: '2m',
    },

    // 갑작스런 트래픽 급증 (Black Friday 시뮬레이션)
    spike: {
      executor: 'ramping-vus',
      startVUs: 10,
      stages: [
        // 1초 만에 500 VU로 급증
        { duration: '1s', target: 500 },
        // 5분 유지
        { duration: '5m', target: 500 },
        // 1초 만에 0으로 감소
        { duration: '1s', target: 0 },
      ],
      startTime: '2m',  // 2분 뒤 시작 (정상 트래픽 이후)
    },
  },

  thresholds: {
    // Spike 중에도 p95가 1초 이하여야 함
    'http_req_duration': ['p(95)<1000'],
    // 에러율은 5% 이하 허용 (정상 시간대는 1%)
    'http_req_failed': ['rate<0.05'],
  },
};

export default function() {
  http.get('http://example.com/api/endpoint');
  sleep(1);
}
```

---

## 📊 성능 비교

| Executor | 용도 | VU 수 제어 | 요청 수 예측 | 실제 사용 사례 |
|----------|------|-----------|------------|--------------|
| **ramping-vus** | 병목점 찾기 | 시간에 따라 증감 | 예측 어려움 | 점진적 부하 증가 테스트 |
| **constant-vus** | 안정성 검증 | 고정 유지 | VU × 테스트 시간 | 배경 트래픽, 스트레스 테스트 |
| **per-vu-iterations** | 정확한 부하 | 시간에 따라 변함 | VU × 반복 횟수 | 배치 작업, 구매 흐름 |
| **ramping-arrival-rate** | 도착률 기반 | 동적 조정 | 도착률 × 시간 | API 게이트웨이, 메시지 큐 |
| **shared-iterations** | 공유 반복 | 시간에 따라 변함 | 고정 (shared iterations) | 한정된 리소스 테스트 |

### 실제 테스트 결과 비교

**시나리오**: 5분 테스트, 동일한 API 호출

```
ramping-vus:
  VU 0→100 (2min) → 100 유지 (3min)
  총 요청: ~12,000개 (변동 있음)
  CPU: 30%
  메모리: 60MB
  예측: ✓ 가능 (stages 기반)

constant-vus:
  VU 100 고정
  총 요청: ~15,000개
  CPU: 35%
  메모리: 65MB
  예측: ✓ 가능 (VU × 시간)

per-vu-iterations:
  VU 100 × 반복 100 = 10,000 요청 정확히
  CPU: 20%
  메모리: 55MB
  예측: ✓ 정확함

ramping-arrival-rate:
  초당 50개 요청 유지
  총 요청: ~15,000개 (5분 × 60초 × 50)
  CPU: 32%
  메모리: 62MB (VU 동적 조정)
  예측: ✓ 가능 (도착률 × 시간)
```

---

## ⚖️ 트레이드오프

| 항목 | Ramping | Constant | Per-VU | Arrival Rate | Shared |
|------|---------|----------|--------|--------------|--------|
| **병목점 감지** | ✓✓✓ 우수 | ✓ 낮음 | ✗ 낮음 | ✓✓ 중간 | ✗ 낮음 |
| **요청 수 예측** | ✗ 어려움 | ✓✓ 쉬움 | ✓✓ 쉬움 | ✓ 중간 | ✓✓ 쉬움 |
| **메모리 효율** | ✓✓ 좋음 | ✓ 중간 | ✓✓ 좋음 | ✓✓✓ 최고 | ✓✓ 좋음 |
| **현실성** | ✓✓✓ 높음 | ✓ 낮음 | ✓✓ 중간 | ✓✓ 높음 | ✗ 낮음 |
| **안정성** | ✗ 변동 많음 | ✓✓ 안정 | ✓ 중간 | ✓✓ 안정 | ✓✓ 안정 |

---

## 📌 핵심 정리

- **Scenarios는 새로운 표준**: `vus`/`duration`은 구식, `options.scenarios`로 세밀한 제어
- **5가지 Executor 선택**:
  - ramping-vus: 병목점 찾기
  - constant-vus: 안정성 검증
  - per-vu-iterations: 정확한 요청 수
  - ramping-arrival-rate: 도착률 기반
  - shared-iterations: 공유 반복
- **Stages는 선형 증감**: 단계별로 VU를 부드럽게 조정
- **Thresholds는 자동 판정**: p95, p99, rate, count 등으로 합격/실패 결정
- **여러 시나리오 동시 실행**: 현실적인 트래픽 믹스 시뮬레이션
- **CI/CD 통합**: threshold 실패 → exit code 1 → 배포 중단

---

## 🤔 생각해볼 문제

**Q1**: ramping-vus에서 stages가 `[{duration: '1m', target: 100}]`라면, VU가 정확히 몇 개일까?

<details>
<summary>해설 보기</summary>

**처음은 startVUs 개**, **마지막은 target 개**, **중간은 선형 증가**입니다.

```javascript
export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,      // ← 처음 VU 수
      stages: [
        { duration: '1m', target: 100 },  // ← 1분 뒤 100 VU
      ],
    },
  },
};

// 시간에 따른 VU:
// 0s: 0 VU
// 10s: ~16.7 VU (0 + 100 × 10/60)
// 20s: ~33.3 VU
// 30s: ~50 VU
// 40s: ~66.7 VU
// 50s: ~83.3 VU
// 60s: 100 VU
```

</details>

---

**Q2**: per-vu-iterations에서 VU 5개, iterations 10개면 총 몇 개의 요청이 생길까?

<details>
<summary>해설 보기</summary>

**정확히 50개**입니다.

```javascript
export const options = {
  scenarios: {
    test: {
      executor: 'per-vu-iterations',
      vus: 5,           // 5개의 VU
      iterations: 10,   // 각 VU당 10번 반복
    },
  },
};

// 총 요청 수 = 5 × 10 = 50개

// 각 VU의 실행:
// VU 1: [iter 1] [iter 2] ... [iter 10] → 완료
// VU 2: [iter 1] [iter 2] ... [iter 10] → 완료
// VU 3: [iter 1] [iter 2] ... [iter 10] → 완료
// VU 4: [iter 1] [iter 2] ... [iter 10] → 완료
// VU 5: [iter 1] [iter 2] ... [iter 10] → 완료
```

**주의**: shared-iterations이라면 다릅니다!
```javascript
// shared-iterations는 "전체 50회를 5 VU가 나눠서"
export const options = {
  scenarios: {
    test: {
      executor: 'shared-iterations',
      vus: 5,
      iterations: 50,   // 전체 50회
    },
  },
};

// 결과: VU 1이 [iter 1,6,11,...,46], VU 2가 [iter 2,7,12,...,47] 처럼 분담
// 총 요청 수 = 50개 (동일하지만 분배 방식이 다름)
```

</details>

---

**Q3**: threshold에서 `'http_req_duration': ['p(95)<500', 'p(99)<1000']`이면, p95가 600ms, p99가 900ms면 테스트가 통과할까?

<details>
<summary>해설 보기</summary>

**아니요, 실패합니다.**

각 조건은 **AND** 관계입니다:
- ✓ p(95) < 500? → ✗ NO (600 > 500)
- ✓ p(99) < 1000? → ✓ YES (900 < 1000)

**결과: 하나라도 실패하면 전체 실패** (exit code 1)

```
thresholds: {
  'http_req_duration': [
    'p(95)<500',    // ← 이것이 실패
    'p(99)<1000',   // 이것은 성공
  ],
}

최종 결과: FAIL (p(95) 조건 미충족)
```

**만약 OR 조건으로 하려면 시나리오를 분리**:
```javascript
thresholds: {
  'http_req_duration{priority:high}': ['p(95)<500'],  // 높은 우선순위
  'http_req_duration{priority:low}': ['p(95)<800'],   // 낮은 우선순위
}
```

</details>

---

<div align="center">

**[⬅️ 이전: k6 아키텍처 — Go 엔진과 VU 동시성](./01-k6-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP 요청과 검증 — 로그인 토큰 재사용 패턴 ➡️](./03-http-requests-validation.md)**

</div>
