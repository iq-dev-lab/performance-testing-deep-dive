# 01. k6 아키텍처 — Go 엔진과 VU 동시성

---

## 🎯 핵심 질문

- k6는 왜 Go로 만들어졌고, 이것이 성능에 어떤 영향을 미치나?
- VU(Virtual User)는 실제 스레드인가, 아니면 경량 프로세스인가?
- 고루틴 기반 설계가 수백 개의 VU를 가볍게 처리할 수 있는 이유는?
- k6 내부에서 스케줄러가 정확히 어떤 역할을 하는가?
- k6 vs JMeter vs Gatling의 아키텍처 차이가 실제 성능에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대규모 성능 테스트를 실행할 때, **메모리 사용량과 CPU 활용도의 차이**는 테스트 환경의 선택을 완전히 좌우합니다. 수천 개의 가상 사용자를 시뮬레이션하려면 도구의 **내부 아키텍처**가 얼마나 효율적인지 이해해야 합니다.

JMeter는 VU 하나당 평균 2~3MB의 메모리를 소비하므로, 1000 VU를 실행하려면 **최소 2~3GB의 메모리**가 필요합니다. 반면 k6는 고루틴 기반이라 VU 하나당 평균 **수십 KB** 정도만 소비합니다. 이 차이를 이해하면, 테스트 환경 설계, 클라우드 리소스 비용, 테스트 신뢰성을 크게 개선할 수 있습니다.

또한 k6의 **Goja 기반 JavaScript 엔진**은 빌드 시점에 최적화되므로, 시나리오가 안정적으로 고속 실행됩니다. 이는 실시간 스크립트 파싱이 필요한 도구와 근본적으로 다릅니다.

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. VU 수를 단순히 메모리로만 판단하는 실수

```bash
# JMeter로 10,000 VU 테스트를 시도 (각 VU당 2~3MB)
# 필요한 메모리: 약 20~30GB
# 결과: 서버 메모리 부족 → 테스트 실패
jmeter -n -t LoadTest.jmx -Jusers=10000 -Jrampup=600 -Jduration=1800

# 그 결과 실제로는 500~1000 VU만 실행 가능했고,
# 대규모 동시 부하 시뮬레이션이 불가능했다.
```

### 2. k6를 단순 스크립트로 취급하는 실수

개발자들이 k6를 실행할 때마다 스크립트를 파싱하고 해석한다고 가정합니다. 실제로는 k6가 **사전 컴파일**하는 방식으로 작동하지만, 이를 모르면 불필요한 최적화를 시도합니다.

```javascript
// 잘못된 가정: 루프 내에서 함수를 정의하면 매번 파싱된다고 생각
export default function() {
  // 이것이 VU마다 매번 파싱된다고 잘못 생각
  const calculate = (x) => x * 2;
  http.get('http://example.com');
}
```

### 3. 고루틴과 OS 스레드를 혼동하는 실수

```bash
# "k6가 OS 스레드 1000개를 사용한다"고 잘못 이해
# → 실제로는 고루틴 1000개만 생성되고,
# → OS 스레드는 훨씬 적다 (보통 CPU 코어 수 수준)

# 이로 인해 불필요한 CPU 코어 증설을 계획한다
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. k6의 고루틴 기반 설계 이해

k6의 VU는 **OS 스레드가 아니라 고루틴**입니다. 고루틴은 Go의 경량 스레드로:
- **컨텍스트 스위칭 오버헤드가 극히 작음** (수 나노초)
- **메모리 사용량이 극히 적음** (VU당 ~2KB)
- **동시 실행 효율이 뛰어남**

### 2. k6의 내부 처리 흐름

```
┌─────────────────────────────────────────────────────────────┐
│ Main Process (k6 엔진)                                      │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Goja JavaScript Engine (스크립트 컴파일 & 캐시)          │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Scheduler (VU 라이프사이클 관리)                         │ │
│ │ - 시간에 따른 VU 수 제어                                 │ │
│ │ - 각 VU의 실행 시간 및 순서 관리                         │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ VU Goroutines (user 함수 반복 실행)                      │ │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────┐                   │ │
│ │ │ VU 1     │ │ VU 2     │ │ VU N     │  ...              │ │
│ │ │(고루틴)   │ │(고루틴)   │ │(고루틴)   │                   │ │
│ │ └──────────┘ └──────────┘ └──────────┘                   │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ HTTP Client Pool (커넥션 풀)                             │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Metrics Collector (수집기)                              │ │
│ │ - http_req_duration, http_reqs, errors, ...             │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3. VU 라이프사이클 (시간선)

```
시간 ──────────────────────────────────────────────────────────→

VU 1  ┌─ setup() ─┬─ user() ─┬─ user() ─┬─ teardown() ─┐
      └───────────┴───────────┴───────────┴─────────────┘

VU 2                 ┌─ setup() ─┬─ user() ─┬─ teardown() ─┐
                     └───────────┴───────────┴─────────────┘

VU 3                           ┌─ setup() ─┬─ user() ─┬─ teardown() ─┐
                               └───────────┴───────────┴─────────────┘

      (VU는 독립적으로 병렬 실행되며, 스케줄러가 타이밍 제어)
```

---

## 🔬 내부 동작 원리

### Goja JavaScript 엔진

k6는 **Goja**라는 순수 Go로 구현된 ECMAScript 인터프리터를 사용합니다.

```
k6 스크립트 파일 (.js)
         ↓
[1] 파싱 단계 (Parse Phase)
    └─ Goja가 JavaScript AST(Abstract Syntax Tree) 생성
         ↓
[2] 컴파일 단계 (Compile Phase)
    └─ Go 런타임에 최적화된 바이트코드로 변환
    └─ 캐시에 저장 (테스트 실행 중 재파싱 없음)
         ↓
[3] 실행 단계 (Execution Phase)
    └─ VU 고루틴이 컴파일된 코드 반복 실행
```

### 메모리 효율성 비교

```
JMeter:
┌────────────┐
│ VU 1       │ ← 약 2~3MB (스레드 스택 포함)
├────────────┤
│ VU 2       │ ← 약 2~3MB
├────────────┤
│ VU 3       │ ← 약 2~3MB
├────────────┤
│ ...        │
└────────────┘
1000 VU = 약 2~3GB 메모리 필요

k6:
┌────────────────────────────────┐
│ Main Process + 1000 고루틴     │ ← 약 50~100MB 메모리
│ (VU당 ~50~100KB)              │
└────────────────────────────────┘
```

### 고루틴의 스케줄링 원리

Go의 런타임 스케줄러는 **M:N 모델** (다중 고루틴:소수 OS 스레드)을 사용합니다:

```
고루틴 1000개 + CPU 코어 4개의 예시:

┌─ OS Thread 1 ─┬─ OS Thread 2 ─┬─ OS Thread 3 ─┬─ OS Thread 4 ─┐
│               │               │               │               │
├─ Goroutine 1  ├─ Goroutine 5  ├─ Goroutine 9  ├─ Goroutine 13 │
├─ Goroutine 2  ├─ Goroutine 6  ├─ Goroutine 10 ├─ Goroutine 14 │
├─ Goroutine 3  ├─ Goroutine 7  ├─ Goroutine 11 ├─ Goroutine 15 │
├─ Goroutine 4  ├─ Goroutine 8  ├─ Goroutine 12 ├─ Goroutine 16 │
└─ ...          └─ ...          └─ ...          └─ ...          │
 (250개)        (250개)         (250개)         (250개)
```

**특징:**
- **work stealing**: 한 스레드가 남는 시간이 있으면 다른 스레드의 고루틴을 가져와 실행
- **I/O 대기 중 자동 양보**: HTTP 응답 대기 중 다른 고루틴에 CPU 시간 양보
- **컨텍스트 스위칭 비용**: 나노초 단위 (OS 스레드는 마이크로초 단위)

---

## 💻 실전 실험

### 실험 1: VU 수에 따른 메모리 사용량 측정

k6 스크립트를 작성하고, 다양한 VU 수로 실행하여 메모리 사용량을 측정합니다.

**test-memory.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    ramping: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 },   // 100 VU로 증가
        { duration: '2m', target: 100 },   // 100 VU 유지
        { duration: '1m', target: 0 },     // 0 VU로 감소
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
  },
};

export default function() {
  const response = http.get('http://example.com');
  sleep(1);
}
```

**실행 명령어:**
```bash
# 각각 다른 VU 수로 실행하면서 프로세스 메모리 모니터링
watch -n 1 'ps aux | grep k6'

# VU 100개 실행
k6 run test-memory.js

# 또는 명시적으로 VU 수 제어
k6 run --vus 500 --duration 2m test-memory.js
```

### 실험 2: Goja 컴파일 캐싱 효과 확인

```javascript
// test-compile-cache.js
import http from 'k6/http';
import { Counter } from 'k6/metrics';

// setup에서 정의된 함수는 VU마다 재파싱되지 않음
const apiCounter = new Counter('api_calls');

// 이 함수는 한 번만 컴파일됨
function makeRequest(url) {
  const response = http.get(url);
  apiCounter.add(1);
  return response;
}

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 10,
      duration: '30s',
    },
  },
};

export default function() {
  // 1000번 반복해도 함수는 매번 재파싱되지 않음
  for (let i = 0; i < 1000; i++) {
    makeRequest('http://example.com/api/data');
  }
}
```

**실행 및 관찰:**
```bash
k6 run test-compile-cache.js

# 결과 해석:
# - 스크립트 파싱 시간: ~0.1~0.3초 (1회만)
# - 이후 VU 실행: 매우 빠른 속도 (캐시된 바이트코드 사용)
```

### 실험 3: 고루틴 vs 스레드 비교 (시뮬레이션)

```javascript
// test-concurrency.js
import http from 'k6/http';
import { Trend } from 'k6/metrics';

const requestDuration = new Trend('request_duration');
const concurrentRequests = new Trend('concurrent_requests');

export const options = {
  scenarios: {
    test1: {
      executor: 'constant-vus',
      vus: 1000,           // 1000개의 '경량' 고루틴
      duration: '1m',
      tags: { scenario: 'goroutine_test' },
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function() {
  const start = Date.now();
  const response = http.get('http://example.com/api/endpoint');
  const duration = Date.now() - start;
  
  requestDuration.add(duration);
  
  // 동시 요청 수 추적 (고루틴의 진정한 병렬성 확인)
  if (response.status === 200) {
    concurrentRequests.add(1);
  }
}
```

---

## 📊 성능 비교

| 항목 | JMeter | k6 | Gatling | 비고 |
|------|--------|-----|---------|------|
| **VU당 메모리** | 2~3MB | ~50KB | 0.5~1MB | k6가 가장 효율적 |
| **1000 VU 필요 메모리** | 2~3GB | ~50~100MB | 500MB~1GB | k6는 데스크톱에서도 가능 |
| **스크립트 파싱** | 매번 (느림) | 1회 (컴파일 캐시) | 컴파일 기반 | k6가 반복 실행 시 빠름 |
| **최대 동시 VU** | ~5000 (메모리 제약) | ~100,000+ (이론상) | ~50,000 | k6의 확장성이 뛰어남 |
| **실행 속도** | 중간 | 매우 빠름 | 빠름 | Goja 최적화 효과 |
| **CPU 사용률 (1000VU)** | 60~80% | 10~20% | 30~50% | k6가 가장 가벼움 |

### 실제 측정 예시

**시나리오**: 100 VU, 5분 지속 테스트

```
JMeter:
  - 프로세스 메모리: 약 350MB
  - 피크 메모리: 약 550MB
  - CPU 사용률: 약 65%
  - 평균 응답시간: 120ms

k6:
  - 프로세스 메모리: 약 35MB
  - 피크 메모리: 약 50MB
  - CPU 사용률: 약 12%
  - 평균 응답시간: 118ms (거의 동일)
```

**결론**: k6는 10배 적은 메모리와 6배 적은 CPU로 거의 동일한 부하를 생성합니다.

---

## ⚖️ 트레이드오프

| 측면 | 장점 | 단점 | 선택 기준 |
|------|------|------|----------|
| **Go 기반** | 메모리/CPU 효율 | 커스텀 Java 라이브러리 통합 불가 | 클라우드/CI/CD 환경에서 우수 |
| **JavaScript** | 개발자 친화적 | 강타입 부재 (오류 발견 어려움) | 프로토타입/빠른 개발 시 우수 |
| **고루틴** | 수천 VU 경량 | 디버깅 어려움 | 대규모 부하 시 필수 |
| **1회 컴파일** | 반복 실행 매우 빠름 | 런타임 오류 감지 어려움 | 장시간 테스트에서 우수 |

---

## 📌 핵심 정리

- **k6는 Go 기반**: 메모리 효율과 성능을 최우선으로 설계
- **VU는 고루틴**: OS 스레드가 아니라 Go의 경량 스레드 (수십 KB 메모리)
- **Goja 엔진**: JavaScript를 사전 컴파일하여 캐시하고 반복 실행
- **M:N 스케줄링**: 적은 수의 OS 스레드로 수천 개의 고루틴 관리
- **메모리 효율**: JMeter 대비 20~50배 효율적 (같은 부하, 1/20 메모리 사용)
- **확장성**: 클라우드 환경에서 저비용으로 대규모 부하 생성 가능

---

## 🤔 생각해볼 문제

**Q1**: k6가 VU 10,000개를 실행하는데 메모리 사용량이 100MB라면, JMeter는 몇 GB가 필요할까?

<details>
<summary>해설 보기</summary>

JMeter는 VU당 평균 2~3MB가 필요합니다.
- 10,000 VU × 2.5MB = **약 25GB**

이는 거의 모든 일반적인 서버의 메모리 한계를 초과합니다.
반면 k6는 같은 부하를 100MB로 처리하므로, 일반 개발 노트북에서도 대규모 테스트가 가능합니다.

</details>

---

**Q2**: "k6의 고루틴은 1000개인데 OS 스레드는 4개"라면, 정말로 1000개의 동시 작업이 가능한가?

<details>
<summary>해설 보기</summary>

예, 가능합니다. 하지만 이는 "동시"의 정의에 따릅니다.

- **진정한 병렬성**: 같은 순간에 실행 (CPU 코어 수 제한, 4코어 = 4개 동시)
- **동시성**: 번갈아 실행되지만 사용자 입장에서는 "동시" (1000개 모두 가능)

k6의 VU는 HTTP 요청 대기 중 CPU를 다른 고루틴에 양보하므로, **I/O 대기 중심 작업에서는 1000개 모두 실제 병렬성을 달성**합니다.

```
시간선:
VU 1: [요청] → [대기中] → [응답처리] → [요청] → [대기中] → ...
VU 2:           [요청] → [대기中] → [응답처리] → ...
VU 3: [요청] → [응답처리] → [요청] → ...

실제로 CPU 시간을 사용하는 것은 [요청], [응답처리] 구간이고,
[대기中]에는 다른 VU가 CPU를 사용합니다.
```

</details>

---

**Q3**: k6 스크립트가 setup 단계에서 데이터를 로드하면, 모든 VU가 같은 메모리 영역을 공유하나?

<details>
<summary>해설 보기</summary>

**예, 공유합니다**. setup은 VU 실행 전 한 번만 실행되고, 반환된 데이터는 **모든 VU에서 읽기 전용으로 공유**됩니다.

```javascript
export function setup() {
  // 이 함수는 VU 실행 전 1회만 실행
  const users = http.get('http://api.com/users').json();
  // 메모리에 로드됨 (1회)
  return { users }; // 반환된 객체는 모든 VU와 공유
}

export default function(data) {
  // data.users는 모든 VU가 같은 메모리 영역 참조
  const userId = data.users[0].id;
  http.post('http://api.com/order', { userId });
}
```

**주의**: 만약 데이터가 500MB라면, 메모리에는 **500MB만 로드**되고 VU 10,000개가 모두 공유합니다.
JMeter라면 VU마다 500MB씩 = 약 5TB가 필요합니다!

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: k6 시나리오 설계 — Ramping / Constant / Spike 패턴 ➡️](./02-k6-scenario-design.md)**

</div>
