# 03. HTTP 요청과 검증 — 로그인 토큰 재사용 패턴

---

## 🎯 핵심 질문

- `http.get()`, `http.post()`, `http.batch()`의 성능 차이는 무엇인가?
- `check()`와 `throw`의 차이점은 무엇이고, 언제 어느 것을 사용하는가?
- 쿠키와 세션을 자동으로 관리하는 메커니즘은 어떻게 동작하는가?
- 로그인 후 토큰을 추출하여 다음 요청에 재사용하는 패턴의 구현은?
- `__ENV`, `__ITER`, `__VU` 같은 내장 변수는 어떤 상황에서 활용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

성능 테스트는 **단순히 API를 호출하는 것**이 아닙니다. 실제 사용자 행동은:

1. **로그인** → 토큰/세션 취득
2. **토큰 검증** → 다음 요청에 포함
3. **응답 검증** → 예상한 데이터 구조 확인
4. **결과 수집** → 선택적 상태 유지

만약 로그인 토큰 재사용이 없다면, **모든 VU가 매번 로그인**해야 합니다. 이는:
- 데이터베이스 부하 증가 (실제 사용 패턴과 무관)
- 세션 풀 고갈 (테스트가 현실성 상실)
- 잘못된 성능 결과 도출

또한 `check()`를 사용하면 **응답 검증 실패**를 메트릭으로 추적할 수 있어, API 버그를 테스트 중 발견할 수 있습니다.

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. 매번 새로운 세션으로 로그인하는 실수

```javascript
// ❌ 성능 테스트가 아니라 로그인 스트레스 테스트 됨
export default function() {
  // 매번 로그인 (불필요한 인증 오버헤드)
  const loginRes = http.post('http://api.example.com/login', {
    username: 'user' + __VU,
    password: 'password123',
  });

  // 토큰을 추출하지 않으므로, 매 반복마다 새 세션
  const res = http.get('http://api.example.com/profile');
  // 401 Unauthorized 에러 발생!

  sleep(1);
}

// 결과:
// - 실제 API 부하 테스트가 아님
// - 로그인 비용으로 인한 과다 평가
// - 대량 에러 (401 Unauthorized)
```

### 2. 응답 검증을 하지 않는 실수

```javascript
// ❌ 응답이 유효한지 확인하지 않음
export default function() {
  const response = http.get('http://api.example.com/users/1');
  
  // response를 사용하지도 않음
  // 만약 API가 버그가 있어서 항상 400을 반환해도 무시됨
  
  sleep(1);
}

// 결과:
// - 테스트가 "성공"이라고 표시됨
// - 실제로는 API가 망가져 있음
// - 프로덕션 배포 후 장애 발견
```

### 3. HTTP 배치 요청을 사용하지 않는 실시 (성능 저하)

```javascript
// ❌ 3개의 API를 순차적으로 호출 (3배 느림)
export default function() {
  http.get('http://api.example.com/products');
  http.get('http://api.example.com/recommendations');
  http.get('http://api.example.com/reviews');
  
  // 각각 순차적으로 200ms씩 = 총 600ms
  sleep(1);
}

// 결과:
// - 불필요한 지연
// - 병렬 처리 불가능 (테스트 속도 낮음)
// - 응답시간 왜곡됨
```

### 4. 환경 변수와 동적 데이터를 구분하지 않는 실수

```javascript
// ❌ 하드코딩된 사용자명 (모든 VU가 같은 계정으로 로그인)
const username = 'testuser';
const password = 'password123';

export default function() {
  const res = http.post('http://api.example.com/login', {
    username: username,
    password: password,
  });

  sleep(1);
}

// 결과:
// - 모든 VU가 같은 계정으로 동시 로그인
// - 실제 다중 사용자 시나리오 아님
// - 세션 충돌 가능성
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. Setup에서 한 번 로그인, 모든 VU가 토큰 공유

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

// setup: 테스트 시작 전 1회만 실행
export function setup() {
  const loginResponse = http.post(
    'http://api.example.com/login',
    {
      username: 'admin@example.com',
      password: 'password123',
    }
  );

  check(loginResponse, {
    'login successful': (r) => r.status === 200,
  });

  // 토큰 추출 (응답 형식에 따라 조정)
  const token = loginResponse.json('token');
  return { token }; // 반환된 데이터는 모든 VU로 전달
}

// 테스트 시나리오
export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

// default: 모든 VU가 이 함수를 반복 실행
export default function(data) {
  // data.token은 setup에서 받은 토큰 (공유됨)
  const token = data.token;

  // 토큰을 Authorization 헤더에 포함
  const headers = {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
  };

  // 이제 토큰 없이 API 호출 가능
  const response = http.get(
    'http://api.example.com/profile',
    { headers }
  );

  check(response, {
    'status is 200': (r) => r.status === 200,
    'has user data': (r) => r.json('data.userId') !== undefined,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });

  sleep(1);
}

// teardown: 테스트 완료 후 1회만 실행
export function teardown(data) {
  // 정리 작업 (선택사항)
  http.post('http://api.example.com/logout', {}, {
    headers: {
      Authorization: `Bearer ${data.token}`,
    },
  });
}
```

**결과:**
- ✓ 로그인은 1회만 (테스트 시작 전)
- ✓ 50 VU × 5분 = 모두 토큰 재사용
- ✓ 데이터베이스 부하 감소
- ✓ 응답 검증으로 API 버그 감지

### 2. 각 VU가 개별 토큰 취득 (다중 사용자 시나리오)

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration{stage:login}': ['p(95)<200'],  // 로그인 빨라야 함
    'http_req_duration{stage:api}': ['p(95)<500'],    // API는 좀 느려도 됨
  },
};

export default function() {
  // VU마다 다른 사용자로 로그인
  const userId = 'user' + __VU;  // VU 1 → user1, VU 2 → user2, ...
  
  const loginRes = http.post(
    'http://api.example.com/login',
    {
      username: userId,
      password: 'password123',
    },
    { tags: { stage: 'login' } }
  );

  check(loginRes, {
    'login succeeded': (r) => r.status === 200,
  });

  // 각 VU가 자신의 토큰을 추출
  const token = loginRes.json('token');
  const headers = {
    Authorization: `Bearer ${token}`,
  };

  sleep(0.5);

  // 각 VU는 자신의 계정으로 API 호출
  const apiRes = http.get(
    'http://api.example.com/dashboard',
    { headers, tags: { stage: 'api' } }
  );

  check(apiRes, {
    'api succeeded': (r) => r.status === 200,
    'has dashboard data': (r) => r.json('data') !== null,
  });

  sleep(1);
}
```

**특징:**
- 각 VU가 고유한 사용자로 로그인
- 실제 다중 사용자 시나리오 재현
- 세션 풀 부하 측정

### 3. HTTP 배치 요청으로 병렬 처리

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<300'],  // 병렬 처리로 인해 더 빠름
  },
};

export default function() {
  // 방법 1: 순차적 요청 (나쁜 예)
  /*
  http.get('http://api.example.com/products');      // ~100ms
  http.get('http://api.example.com/reviews');       // ~100ms
  http.get('http://api.example.com/recommendations'); // ~100ms
  // 총: ~300ms
  */

  // 방법 2: 배치 요청 (좋은 예)
  const responses = http.batch([
    ['GET', 'http://api.example.com/products'],
    ['GET', 'http://api.example.com/reviews'],
    ['GET', 'http://api.example.com/recommendations'],
  ]);

  // 총: ~100ms (병렬 처리)

  // 각 응답 검증
  check(responses[0], {
    'products loaded': (r) => r.status === 200,
  });
  check(responses[1], {
    'reviews loaded': (r) => r.status === 200,
  });
  check(responses[2], {
    'recommendations loaded': (r) => r.status === 200,
  });

  sleep(1);
}
```

### 4. 복잡한 토큰 & 쿠키 관리

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { CookieJar } from 'k6/http';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
  },
};

export default function() {
  // 쿠키 자동 관리 (k6 기본 기능)
  const jar = new CookieJar();
  jar.set('http://api.example.com', 'session_id', 'session_' + __VU);

  // 1단계: 로그인 (쿠키 자동 저장)
  const loginRes = http.post(
    'http://api.example.com/auth/login',
    {
      email: 'user' + __VU + '@example.com',
      password: 'password123',
    },
    { cookieJar: jar }
  );

  check(loginRes, {
    'login succeeded': (r) => r.status === 200,
  });

  const token = loginRes.json('data.token');
  const refreshToken = loginRes.json('data.refreshToken');

  sleep(0.5);

  // 2단계: API 요청 (쿠키 자동 포함 + 수동 토큰 헤더)
  const apiRes = http.get(
    'http://api.example.com/user/profile',
    {
      headers: {
        Authorization: `Bearer ${token}`,
        'X-Refresh-Token': refreshToken,
      },
      cookieJar: jar,
    }
  );

  check(apiRes, {
    'api succeeded': (r) => r.status === 200,
  });

  // 3단계: 토큰 갱신
  if (apiRes.status === 401) {
    // 토큰 만료 시 refreshToken으로 새 토큰 발급
    const refreshRes = http.post(
      'http://api.example.com/auth/refresh',
      { refreshToken: refreshToken }
    );

    const newToken = refreshRes.json('data.token');
    // 새 토큰으로 재시도
  }

  sleep(1);
}
```

---

## 🔬 내부 동작 원리

### Setup-Default-Teardown 실행 순서

```
테스트 시작
    ↓
[Setup 단계] ← 1회만 실행
  - 데이터 준비
  - 토큰 취득
  - DB 초기화
    ↓
  반환 값 → 모든 VU로 전달
    ↓
[Default 단계] ← 각 VU가 반복 실행
  - VU 1: default() → default() → default() ...
  - VU 2: default() → default() → default() ...
  - VU N: default() → default() → default() ...
    ↓
[Teardown 단계] ← 1회만 실행
  - 리소스 정리
  - 테스트 데이터 삭제

시간선:
┌─────────────────────────────────────────────────────┐
│ [Setup] │ [Default × 50] [Default × 50] ... │ [Teardown] │
│         │ (50 VU 병렬 실행)                  │             │
│  1초    │          300초                     │   1초       │
└─────────────────────────────────────────────────────┘
```

### Check의 내부 메커니즘

```javascript
check(response, {
  'status is 200': (r) => r.status === 200,
  'response time < 200ms': (r) => r.timings.duration < 200,
});

// 내부 동작:
// 1. 첫 번째 조건 평가: response.status === 200? → true/false 기록
// 2. 두 번째 조건 평가: response.timings.duration < 200? → true/false 기록
// 3. 메트릭 추가: checks{check:..., group:...} → true/false 집계
// 4. 결과를 stdout에 출력: ✓ 또는 ✗

// 중요: check 실패는 테스트를 중단하지 않음!
// 계속 실행되며, 메트릭에만 기록됨
```

### Throw와 Check의 차이

```javascript
// Check: 실패해도 계속 실행
check(res, { 'status 200': r => r.status === 200 });
console.log('Check 실패해도 여기 실행됨');

// Throw: 실패하면 즉시 종료
if (res.status !== 200) {
  throw new Error('요청 실패! 스크립트 중단');
}
console.log('Throw가 없으면 여기 실행됨');

// 실제 비교:
// ┌─────────────────┐
// │ Check() 실패    │ ← 계속 진행
// │ 메트릭 증가     │
// └─────────────────┘
//
// ┌─────────────────┐
// │ throw 실행      │ ← 즉시 중단
// │ VU 재시작       │
// └─────────────────┘
```

### 내장 변수 활용

```javascript
export default function() {
  // __VU: 현재 VU 번호 (1부터 시작)
  console.log('VU ' + __VU);  // VU 1, VU 2, ...

  // __ITER: 현재 VU의 반복 횟수 (0부터 시작)
  console.log('Iteration ' + __ITER);  // 0, 1, 2, ...

  // __ENV: 환경 변수 접근
  const baseUrl = __ENV.BASE_URL || 'http://localhost:3000';

  // 실제 활용:
  const userId = 'user' + (__VU % 10);  // VU를 10으로 나눈 나머지 사용
  const itemId = Math.floor(Math.random() * 1000);
  
  http.get(baseUrl + '/api/items/' + itemId);

  // 로그:
  if (__ITER % 100 === 0) {
    console.log(`VU ${__VU} completed 100 iterations`);
  }
}
```

---

## 💻 실전 실험

### 실험 1: 로그인 토큰 재사용 vs 매번 로그인

**test-token-reuse.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Trend } from 'k6/metrics';

const loginDuration = new Trend('login_duration');
const apiDuration = new Trend('api_duration');

// 방법 A: Setup에서 토큰 재사용
export function setup() {
  const res = http.post('http://httpbin.org/post', {
    user: 'shared_token_user',
  });
  return { token: res.json('data.token') || 'mock_token_' + Date.now() };
}

export const options = {
  scenarios: {
    reuse_token: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
    },
  },
};

export default function(data) {
  const start = Date.now();
  // 토큰 재사용 (로그인 불필요)
  const res = http.get('http://httpbin.org/anything?token=' + data.token);
  apiDuration.add(Date.now() - start);
  
  check(res, { 'status 200': r => r.status === 200 });
  sleep(1);
}
```

**실행:**
```bash
# 방법 A: 토큰 재사용 (권장)
k6 run test-token-reuse.js

# 기대 결과:
# - http_req_duration 평균: ~150ms
# - http_reqs: 50 VU × (120s / 1s) = 6000 요청
# - 메모리: 낮음
```

**test-login-every-time.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Trend } from 'k6/metrics';

const totalDuration = new Trend('total_duration');

export const options = {
  scenarios: {
    login_every_time: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
    },
  },
};

export default function() {
  const start = Date.now();

  // 방법 B: 매번 로그인
  const loginRes = http.post('http://httpbin.org/post', {
    username: 'user' + __VU,
    password: 'password',
  });

  const token = loginRes.json('json.token') || 'mock_token';

  // 토큰으로 API 호출
  const apiRes = http.get('http://httpbin.org/anything?token=' + token);

  check(apiRes, { 'status 200': r => r.status === 200 });

  totalDuration.add(Date.now() - start);
  sleep(1);
}
```

**실행:**
```bash
# 방법 B: 매번 로그인
k6 run test-login-every-time.js

# 기대 결과:
# - http_req_duration 평균: ~350ms (로그인 오버헤드)
# - http_reqs: 50 VU × (120s / 1s) × 2 = 12000 요청 (2배)
# - 메모리: 높음
```

### 실험 2: Check를 사용한 응답 검증

**test-checks.js:**
```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 20,
      duration: '1m',
    },
  },
  thresholds: {
    'checks': ['rate>=0.95'],  // 95% 이상 check 통과
  },
};

export default function() {
  const res = http.post('http://httpbin.org/post', {
    title: 'Test Item ' + __ITER,
    description: 'This is a test item',
  });

  // 여러 검증 조건
  check(res, {
    'status is 200': (r) => r.status === 200,
    'has Content-Type': (r) => r.headers['Content-Type'] !== undefined,
    'has json body': (r) => {
      try {
        const json = r.json();
        return json !== null;
      } catch (e) {
        return false;
      }
    },
    'title matches': (r) => r.json('json.title').includes('Test Item'),
  });
}
```

**실행:**
```bash
k6 run test-checks.js

# 결과 분석:
# checks............: 95.5%   ✓ 5000/5240
# checks............: 94.8%   ✓ status is 200
# checks............: 100%    ✓ has Content-Type
# checks............: 99.2%   ✓ has json body
# checks............: 90.3%   ✓ title matches
```

### 실험 3: 배치 요청으로 응답시간 단축

**test-batch.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Trend } from 'k6/metrics';

const sequentialDuration = new Trend('sequential_duration');
const batchDuration = new Trend('batch_duration');

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 30,
      duration: '2m',
    },
  },
};

export default function() {
  // 방법 1: 순차 요청
  const start1 = Date.now();
  http.get('http://httpbin.org/delay/0.05');
  http.get('http://httpbin.org/delay/0.05');
  http.get('http://httpbin.org/delay/0.05');
  sequentialDuration.add(Date.now() - start1);

  sleep(0.5);

  // 방법 2: 배치 요청
  const start2 = Date.now();
  const responses = http.batch([
    ['GET', 'http://httpbin.org/delay/0.05'],
    ['GET', 'http://httpbin.org/delay/0.05'],
    ['GET', 'http://httpbin.org/delay/0.05'],
  ]);
  batchDuration.add(Date.now() - start2);

  check(responses[0], { 'batch success': (r) => r.status === 200 });

  sleep(0.5);
}
```

**실행 및 비교:**
```bash
k6 run test-batch.js

# 결과:
# sequential_duration........: avg=155ms  (3 × 50ms + 오버헤드)
# batch_duration.............: avg=65ms   (병렬 처리)
# 차이: 약 2.4배 빠름!
```

---

## 📊 성능 비교

| 항목 | Setup 토큰 재사용 | 매번 로그인 | 배치 요청 |
|------|------------------|-----------|---------|
| **로그인 요청 수** | 1회 | VU × 반복 | 0회 |
| **API 요청 수** | VU × 반복 | VU × 반복 | VU × (반복/3) |
| **평균 응답시간** | 150ms | 300ms | 100ms |
| **에러율** | 0.5% | 1.5% (토큰 오류) | 0.5% |
| **메모리** | 50MB | 65MB | 45MB |
| **현실성** | ✓ 우수 | ✓ 높음 | ✓ 중간 |

### 구체적 예시 (100 VU, 5분 테스트)

```
Setup 토큰 재사용:
- 로그인: 1회 (1초)
- API: 100 VU × 300회/VU = 30,000 요청
- 총 시간: ~1500초 (실제 시간 감안 더 빠름)
- 에러: ~150건 (0.5%)

매번 로그인:
- 로그인: 100 VU × 300회 = 30,000 회
- API: 100 VU × 300회 = 30,000 요청
- 총 시간: ~3000초 (느린 응답)
- 에러: ~450건 (1.5%)

배치 요청 (3개 API 그룹화):
- API: 100 VU × 100회 (배치당 3개) = 10,000 배치 = 30,000 개별 요청
- 응답시간: ~50% 단축
- 처리량: ~2배 증가
```

---

## ⚖️ 트레이드오프

| 방식 | 장점 | 단점 | 선택 기준 |
|------|------|------|----------|
| **Setup 토큰** | 현실성, 빠른 실행 | 다중 세션 테스트 불가 | 일반적인 API 테스트 |
| **VU별 로그인** | 다중 사용자, 현실적 | 느림, 부하 증가 | 인증/세션 테스트 |
| **배치 요청** | 빠른 응답, 효율적 | 실제 UI와 다름 | 백엔드 극한 성능 테스트 |
| **쿠키 자동화** | 간편함 | 복잡한 세션 관리 불가 | 기본 웹 세션 |

---

## 📌 핵심 정리

- **Setup에서 토큰 취득**: 1회 로그인, 모든 VU 공유 (현실적 & 효율적)
- **VU별 로그인**: 다중 사용자 시뮬레이션 (더 현실적이지만 느림)
- **Check 사용**: 응답 검증, 메트릭 기록 (계속 실행)
- **Throw 사용**: 심각한 오류 시 즉시 중단 (선택적)
- **배치 요청**: 3개 이상의 병렬 요청 시 (응답시간 50% 단축)
- **쿠키 자동관리**: CookieJar로 세션 자동 유지
- **내장 변수**: `__VU`, `__ITER`, `__ENV` 활용

---

## 🤔 생각해볼 문제

**Q1**: Setup에서 토큰을 취득했는데, 토큰이 만료되면 어떻게 해야 할까?

<details>
<summary>해설 보기</summary>

**토큰 갱신 로직을 Default에 추가**해야 합니다:

```javascript
export default function(data) {
  let token = data.token;

  const res = http.get('http://api.example.com/protected', {
    headers: { Authorization: `Bearer ${token}` },
  });

  // 401 에러 = 토큰 만료
  if (res.status === 401) {
    console.log('Token expired, refreshing...');
    
    const refreshRes = http.post('http://api.example.com/refresh', {
      refreshToken: data.refreshToken,
    });

    if (refreshRes.status === 200) {
      token = refreshRes.json('token');
      // 재시도
      return http.get('http://api.example.com/protected', {
        headers: { Authorization: `Bearer ${token}` },
      });
    }
  }

  check(res, { 'api succeeded': r => r.status === 200 });
}
```

또는 **처음부터 토큰 만료 시간을 고려해서 설정**:

```javascript
export function setup() {
  // 짧은 만료 시간 (예: 1분)을 설정하지 않음
  // 대신 테스트 시간보다 긴 토큰 사용
  const res = http.post('http://api.example.com/login', {
    username: 'admin',
    password: 'password',
    expiresIn: '24h',  // 24시간 유효한 토큰
  });

  return { token: res.json('token') };
}
```

</details>

---

**Q2**: Check에서 실패 시 테스트가 중단되지 않는데, 이게 정상 동작인가?

<details>
<summary>해설 보기</summary>

**예, 정상입니다.** Check와 Throw는 다른 목적입니다:

```
Check: "이 조건이 만족하는가?" → 메트릭 기록 → 계속 진행
├─ 사용 사례: 응답 상태 확인, 데이터 검증
├─ 실패해도 VU 계속 실행
└─ threshold로 최종 판정

Throw: "이 조건이 만족하지 않으면 즉시 중단"
├─ 사용 사례: 심각한 오류, 데이터 일관성
├─ 실패하면 VU 종료 → 새 VU 시작
└─ 스크립트 논리 오류
```

**선택 기준:**
- 단순 성능 지표 확인 → Check
- API 응답 구조 검증 → Check
- 필수 데이터 부재 → Throw (선택)
- 인증 실패 → Throw (필요 시)

</details>

---

**Q3**: __VU는 1부터 시작하는데, __ITER는 왜 0부터 시작할까?

<details>
<summary>해설 보기</summary>

**프로그래밍 관례의 차이**입니다:

```javascript
// __VU: 사용자 대면 (1부터)
console.log(__VU);  // VU 1, VU 2, VU 3...
// 사람이 이해하기 쉬움: "첫 번째 VU"

// __ITER: 개발자 대면 (0부터)
console.log(__ITER);  // 0, 1, 2, 3...
// 배열 인덱싱 처럼 사용: data[__ITER]
```

**실제 활용:**

```javascript
// __VU로 사용자 식별
const userId = 'user' + __VU;  // user1, user2, ...

// __ITER로 배열 인덱싱
const testData = ['item1', 'item2', 'item3', 'item4', 'item5'];
const item = testData[__ITER % testData.length];
// __ITER = 0 → item1
// __ITER = 1 → item2
// __ITER = 5 → item1 (순환)
```

</details>

---

<div align="center">

**[⬅️ 이전: k6 시나리오 설계 — Ramping / Constant / Spike 패턴](./02-k6-scenario-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: 결과 분석 — 히스토그램과 백분위수 해석 ➡️](./04-result-analysis.md)**

</div>
