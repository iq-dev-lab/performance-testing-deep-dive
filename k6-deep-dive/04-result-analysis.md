# 04. 결과 분석 — 히스토그램과 백분위수 해석

---

## 🎯 핵심 질문

- p95, p99, p99.9는 각각 무엇이며, 실무에서 왜 평균보다 중요한가?
- `http_req_duration` 히스토그램에서 "산 모양"과 "이중봉우리"는 무엇을 의미하는가?
- k6의 표준 출력에서 `http_req_failed`, `vus_max`, `http_reqs` 같은 지표는 어떻게 해석하는가?
- 정규분포(종 모양)와 이상 분포를 어떻게 구분하고, 각각 무엇을 의미하는가?
- InfluxDB + Grafana에서 백분위수를 쿼리로 계산하는 정확한 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**평균만으로 성능을 평가하면 위험합니다.** 예시:

```
시나리오 A: 응답시간 = [100ms, 100ms, 100ms, 100ms, 1000ms]
평균: 260ms

시나리오 B: 응답시간 = [260ms, 260ms, 260ms, 260ms, 260ms]
평균: 260ms

같은 평균이지만:
- A: p95 = 1000ms (한 명의 사용자가 1초 대기)
- B: p95 = 260ms (모든 사용자가 260ms 대기)
```

**실제 사용자 경험은 다릅니다.** A는 5% 사용자가 극도로 느린 응답을 받는 것이고, B는 모두 균등합니다.

또한 k6의 결과 분석은:
- **즉시 시각화**: 히스토그램으로 분포 패턴 이해
- **이상 징후 감지**: 급격한 응답시간 악화, 특정 시점의 에러 집중
- **병목점 정확화**: "어느 API가 느린가" → "어떤 상황에서 느린가" 파악

이를 통해 **정확한 최적화 타겟**을 찾을 수 있습니다.

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. 평균만으로 성능을 판단하는 실수

```javascript
export const options = {
  scenarios: { test: { executor: 'constant-vus', vus: 100, duration: '5m' } },
  thresholds: {
    // ❌ 평균만 본다
    'http_req_duration': ['avg<300'],
  },
};

// 테스트 결과:
// http_req_duration: avg=280ms ✓ PASS
//
// 하지만 실제:
// p50 (중위값) = 200ms
// p95 = 2000ms  ← 5% 사용자는 2초 대기!
// p99 = 5000ms  ← 1% 사용자는 5초 대기!
//
// 테스트는 통과했지만, 프로덕션에서는 "느리다"는 불평

// 결과: 잘못된 배포 허용
```

### 2. 히스토그램을 읽지 않는 실수

```
// k6 출력:
http_req_duration..................: avg=250ms  min=15ms  med=200ms  max=8500ms
                                p(90)=400ms  p(95)=2000ms  p(99)=5000ms

// ❌ 개발자 해석: "응답시간이 250ms네, 괜찮은데?"
// ✓ 정확한 해석: "p95가 2초? 일부 요청이 극도로 느리다. 병목점이 있다!"

// 히스토그램의 모양을 보면 원인 파악:
// - 정상: [=========●========]  (대부분 같은 시간)
// - 이상: [====●==============] (느린 요청이 꼬리처럼 길어짐)
```

### 3. 시간에 따른 추세를 무시하는 실수

```javascript
// ❌ 전체 5분 테스트의 평균만 본다
// 실제 상황:
// [0-1분] 평균 100ms (서버 워밍업)
// [1-2분] 평균 200ms (안정화)
// [2-3분] 평균 500ms (DB 쿼리 느려짐)
// [3-4분] 평균 800ms (병렬 요청 증가)
// [4-5분] 평균 100ms (의도적으로 VU 감소)
//
// 전체 평균: 340ms
//
// 하지만 "3-4분 사이에 성능이 악화되는 패턴" 감지 불가능
// → 병목점 원인 파악 못함
```

### 4. 에러 분포를 무시하는 실수

```
// ❌ 에러 비율만 본다
http_req_failed: rate=0.5% ✓ PASS (1% 미만)

// 하지만 실제:
// 0-2min: 에러 0개
// 2-3min: 에러 100개 (갑작스런 에러 폭증)
// 3-4min: 에러 0개
// 4-5min: 에러 0개
//
// 전체 에러율: 0.5%는 맞지만,
// "2-3분 사이에 무언가 문제가 발생했다" 감지 불가능
// → 근본 원인 파악 못함
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. 백분위수 기반 Threshold 설정

```javascript
import http from 'k6/http';

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
  
  // ✓ 올바른 Threshold 설정 (백분위수 기반)
  thresholds: {
    // p50 (중위값): 대부분 사용자의 경험
    'http_req_duration': ['p(50)<200'],
    
    // p95: 상위 5% 사용자 경험 (업계 표준)
    'http_req_duration': ['p(95)<500'],
    
    // p99: 상위 1% 사용자 경험
    'http_req_duration': ['p(99)<1000'],
    
    // p99.9: 극도로 운이 없는 사용자
    'http_req_duration': ['p(99.9)<2000'],
    
    // 평균은 참고용만 (판정 기준 아님)
    'http_req_duration': ['avg<300'],
    
    // 최대값 제한 (극단적 오류 방지)
    'http_req_duration': ['max<5000'],
    
    // 에러율 제한
    'http_req_failed': ['rate<0.01'],
  },
};

export default function() {
  http.get('http://example.com/api');
}
```

### 2. k6 출력 결과 상세 읽기

```javascript
// test-detailed-output.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },
        { duration: '3m', target: 50 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration{type:static}': ['p(95)<100'],
    'http_req_duration{type:api}': ['p(95)<500'],
    'http_req_failed': ['rate<0.01'],
    'checks': ['rate>0.95'],
  },
};

export default function() {
  // 정적 자산
  http.get('http://httpbin.org/cache/60', {
    tags: { type: 'static' },
  });
  sleep(0.5);

  // API 요청
  const apiRes = http.post('http://httpbin.org/post', {
    data: { test: true },
  }, {
    tags: { type: 'api' },
  });

  check(apiRes, {
    'status 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

**실행 결과 예시:**
```
running (5m00s), 0/50 VUs, 1000 complete and 0 interrupted iterations

✓ checks...........................: 98.7%    ✓ 987/1000
✗ http_req_duration{type:static}..: ✓ p(95)=95ms
✓ http_req_duration{type:api}.....: ✓ p(95)=450ms
✓ http_req_failed..................: 0%      ✓ 0/2000

     http_req_duration....: avg=245ms    min=12ms    med=180ms    max=8500ms
                          p(90)=600ms  p(95)=2000ms  p(99)=5000ms

     ├─ http_req_duration{type:api}:    avg=300ms  p(95)=450ms  p(99)=800ms
     └─ http_req_duration{type:static}: avg=45ms   p(95)=95ms   p(99)=110ms

     http_req_failed........: 0%        ✓ 0/2000
     http_reqs...............: 2000     [2000 req/min]
     vus.....................: 50       [50 VUs]
     vus_max.................: 50

✓ Test passed
```

**해석:**
```
✓ checks: 98.7% 통과 (거의 모든 응답이 유효)
✗ static: p95=95ms OK (대부분 100ms 이하)
✓ api: p95=450ms OK (API도 충분히 빠름)
✓ http_req_failed: 0% (에러 없음)

결론: 모든 threshold 통과 → 테스트 성공
```

### 3. 히스토그램으로 분포 패턴 이해

```javascript
// test-histogram.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '2m', target: 0 },
      ],
    },
  },
};

export default function() {
  // API 호출 (정상적인 응답시간 분포 기대)
  http.get('http://httpbin.org/delay/0.1');
  sleep(1);
}
```

**히스토그램 해석:**

```
정상 분포 (종 모양):
http_req_duration: avg=150ms min=50ms med=140ms max=300ms
                   p(90)=200ms p(95)=230ms p(99)=250ms

시각화:
  빈도
  │    ╱╲
  │   ╱  ╲
  │  ╱    ╲
  │ ╱      ╲
  │╱________╲
  └─────────────→ 응답시간
   50ms  150ms  300ms

→ 대부분의 요청이 중간값 근처에 분포
→ 이상 현상 없음 ✓


비정상 분포 (오른쪽 꼬리):
http_req_duration: avg=450ms min=50ms med=100ms max=8000ms
                   p(90)=200ms p(95)=1000ms p(99)=7000ms

시각화:
  빈도
  │╱╲
  │  ╲        ╲
  │   ╲        ╲
  │    ╲        ╲
  │     ╲        ╲
  └─────────────────→ 응답시간
   50ms  100ms    8000ms

→ 일부 요청이 극도로 느림 (병목점 있음)
→ 자세한 조사 필요 ⚠️


이중봉우리 분포:
http_req_duration: avg=300ms min=50ms med=250ms max=2000ms
                   p(90)=500ms p(95)=800ms p(99)=1800ms

시각화:
  빈도
  │  ╱╲       ╱╲
  │ ╱  ╲     ╱  ╲
  │╱    ╲   ╱    ╲
  │      ╲ ╱      ╲
  └─────────────────→ 응답시간
   50ms 150ms 300ms 1500ms

→ 두 가지 다른 그룹의 요청이 섞여있음
→ 예: 캐시된 요청(50ms) + DB 쿼리(1500ms)
→ 각 그룹을 분리하여 분석 필요 ⚠️
```

---

## 🔬 내부 동작 원리

### 백분위수 계산 방식

```
응답시간 데이터 (정렬됨): [50, 75, 100, 125, 150, 175, 200, 225, 250, 275]

p50 (중위값): 데이터의 50% 지점
├─ 전체 10개 요청 중 50% = 5번째 요청
├─ 값: 150ms
└─ 의미: "평균적인 사용자 경험"

p95: 데이터의 95% 지점
├─ 전체 10개 중 95% = 9.5번째 (올림: 10번째)
├─ 값: 275ms
└─ 의미: "상위 5%는 이보다 느림"

p99: 데이터의 99% 지점
├─ 전체 10개 중 99% = 9.9번째 (올림: 10번째)
├─ 값: 275ms
└─ 의미: "상위 1%는 이보다 느림"

p99.9: 데이터의 99.9% 지점
├─ 더 큰 샘플 필요 (1000개 이상)
├─ 극도로 운이 없는 사용자
└─ 의미: "거의 모든 사용자보다 느림"
```

### k6의 메트릭 수집 메커니즘

```
각 VU가 요청 실행
    ↓
[메트릭 수집] ← k6 내부에서 자동 수집
├─ http_req_duration: 응답시간 (ms)
├─ http_req_received: 수신 바이트
├─ http_req_sent: 송신 바이트
├─ http_req_failed: 성공/실패 (true/false)
├─ http_req_tls_version: TLS 버전
├─ http_req_connection_reused: 커넥션 재사용 여부
└─ ...기타 지표

    ↓
[데이터 저장] ← 메모리의 메트릭 캐시
├─ 1초마다 수집
├─ 백분위수 계산 (p50, p95, p99, p99.9)
└─ 테스트 진행 중 실시간 업데이트

    ↓
[테스트 완료] ← 최종 결과 출력
├─ stdout: 요약 통계
├─ JSON: 상세 메트릭 (--out json=metrics.json)
└─ InfluxDB/Grafana: 실시간 시각화 (--out influxdb)
```

### Threshold 평가 로직

```javascript
// thresholds 설정
thresholds: {
  'http_req_duration': ['p(95)<500'],  // p95 < 500ms?
  'http_req_failed': ['rate<0.01'],    // 에러율 < 1%?
}

// 평가 시점
테스트 실행 중 (1초마다 점진적 평가):
├─ 1초 경과: p(95) = 400ms ✓ (아직 통과)
├─ 2초 경과: p(95) = 450ms ✓ (계속 통과)
├─ 3초 경과: p(95) = 520ms ✗ (실패!)
│            → 콘솔에 경고 메시지
│            → 하지만 테스트는 계속 실행
├─ 4초 경과: p(95) = 480ms ✓ (다시 통과)
└─ ...

테스트 완료 (최종 평가):
├─ 최종 p(95) = 490ms
├─ threshold: p(95)<500 ✓ 통과
└─ exit code: 0 (성공)

// 만약 최종 평가에서 실패:
├─ 최종 p(95) = 520ms
├─ threshold: p(95)<500 ✗ 실패
└─ exit code: 1 (실패) → CI/CD에서 배포 중단
```

---

## 💻 실전 실험

### 실험 1: 백분위수와 평균의 차이 실증

**test-percentiles.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },
        { duration: '3m', target: 50 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    // p50 (중위값)
    'http_req_duration': ['p(50)<100'],
    // p95 (상위 5%)
    'http_req_duration': ['p(95)<300'],
    // p99 (상위 1%)
    'http_req_duration': ['p(99)<1000'],
    // 평균
    'http_req_duration': ['avg<200'],
  },
};

export default function() {
  http.get('http://httpbin.org/delay/0.1');
  sleep(1);
}
```

**실행:**
```bash
k6 run test-percentiles.js

# 결과 예시:
# http_req_duration: avg=150ms  min=50ms  med=120ms  max=2000ms
#                    p(90)=200ms p(95)=280ms p(99)=800ms
#
# 분석:
# - p50 (중위값) = 120ms < 100ms? ✗ 실패 (중위값도 100ms 초과)
# - p95 = 280ms < 300ms? ✓ 통과
# - p99 = 800ms < 1000ms? ✓ 통과
# - avg = 150ms < 200ms? ✓ 통과
```

### 실험 2: 이상 분포 감지

**test-bimodal.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';
import { Trend } from 'k6/metrics';

const responseDuration = new Trend('response_duration');

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
  },
};

export default function() {
  const start = Date.now();

  // 의도적으로 두 종류의 요청 섞기
  if (__ITER % 2 === 0) {
    // 빠른 요청 (캐시)
    http.get('http://httpbin.org/cache/3600');
  } else {
    // 느린 요청 (대기)
    http.get('http://httpbin.org/delay/0.5');
  }

  responseDuration.add(Date.now() - start);
  sleep(0.5);
}
```

**실행 및 해석:**
```bash
k6 run test-bimodal.js

# 예상 결과:
# response_duration: avg=320ms  min=5ms   med=480ms  max=550ms
#                    p(90)=490ms p(95)=510ms p(99)=540ms
#
# 히스토그램 모양: 두 개의 봉우리 (bimodal)
# ├─ 첫 번째 봉우리: ~50ms (캐시된 요청)
# └─ 두 번째 봉우리: ~500ms (지연 요청)
#
# 원인 파악: 두 가지 다른 API가 섞여있음
# 해결책: 시나리오를 분리하여 각각 분석
```

### 실험 3: 시간 경과에 따른 성능 변화

**test-degradation.js:**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },   // 워밍업
        { duration: '2m', target: 50 },   // 안정화
        { duration: '2m', target: 100 },  // 증가
        { duration: '2m', target: 100 },  // 안정화
        { duration: '1m', target: 0 },    // 감소
      ],
    },
  },
  // 시간별 threshold 설정은 불가능하므로,
  // 전체 테스트 기간의 평균값만 사용
  thresholds: {
    'http_req_duration': ['p(95)<500'],  // 전체 p95 < 500ms
  },
};

export default function() {
  const now = Date.now();
  http.get('http://httpbin.org/delay/0.1');
  sleep(1);
}
```

**실행 후 분석 (InfluxDB로는 시간별 추세 확인 가능):**
```
k6 run --out influxdb test-degradation.js

# Grafana에서 시각화하면:
# 
# 응답시간
# (ms)
# 500 │                        ╱╲
#     │                       ╱  ╲
# 300 │      ╱────────────────╱    ╲
#     │     ╱                        ╲
# 100 │────╱                          ╲────
#     │                                    
# 0   └─────────────────────────────────────→ 시간
#     0min   1min    3min    5min   7min   8min
#     
# 분석:
# - 0-1분: p95 = 150ms (워밍업, 빠름)
# - 1-3분: p95 = 180ms (안정화)
# - 3-5분: p95 = 350ms (VU 100으로 증가, 약간 느려짐)
# - 5-7분: p95 = 320ms (안정화)
# - 7-8분: p95 = 100ms (VU 감소)
#
# 결론: 부하 증가 시 응답시간 2배 증가 (정상)
```

---

## 📊 성능 비교

| 지표 | 값 | 의미 | 주의사항 |
|------|-----|------|---------|
| **avg (평균)** | 250ms | 수학적 평균값 | 극값에 영향 받음 |
| **med (중위값)** | 200ms | 50% 지점 | 분포 대칭성 확인 |
| **p90** | 300ms | 상위 10% | 일반 사용자 |
| **p95** | 400ms | 상위 5% (표준) | 수용 가능한 성능 |
| **p99** | 1000ms | 상위 1% | 극단적 경우 |
| **p99.9** | 2000ms | 상위 0.1% | 극히 드문 경우 |

### 실제 테스트 비교 (동일한 API, 다른 부하)

```
부하 50 VU (안정화 상태):
http_req_duration: avg=150ms  p(95)=180ms  p(99)=200ms
  → 대부분의 사용자가 180ms 이하로 응답 받음

부하 200 VU (높은 부하):
http_req_duration: avg=350ms  p(95)=450ms  p(99)=800ms
  → 상위 5% 사용자는 450ms 이상 대기 (체감 차이 있음)

부하 500 VU (극한 부하):
http_req_duration: avg=800ms  p(95)=1500ms  p(99)=3000ms
  → 상위 5% 사용자는 1.5초 이상 대기 (사용성 악화)
```

---

## ⚖️ 트레이드오프

| 항목 | 평균 기반 | 백분위수 기반 | 히스토그램 기반 |
|------|---------|------------|--------------|
| **계산 복잡도** | 매우 낮음 | 중간 | 높음 |
| **이상 분포 감지** | 불가능 | 일부만 | 우수 |
| **병목점 파악** | 어려움 | 중간 | 우수 |
| **의사결정 품질** | 낮음 | 높음 | 매우 높음 |
| **조직이 이해하기** | 쉬움 | 중간 | 어려움 |

---

## 📌 핵심 정리

- **평균은 거짓**: 극값에 영향 받음, 백분위수 사용 필수
- **p95는 업계 표준**: 상위 5% 사용자의 경험을 나타냄
- **정규분포 확인**: 히스토그램이 종 모양이면 정상, 꼬리가 길면 병목점 존재
- **이중봉우리**: 서로 다른 성능의 API가 섞여있음을 의미
- **시간별 추세**: stdout으로는 불가능, InfluxDB + Grafana 필수
- **Threshold 우선순위**: p50 → p95 → p99 → max 순서로 설정

---

## 🤔 생각해볼 문제

**Q1**: 평균이 200ms, p95가 500ms라면, 왜 평균이 p95보다 낮을까?

<details>
<summary>해설 보기</summary>

**극값(outlier) 때문입니다.**

```
데이터: [100, 100, 100, 100, 100, 100, 100, 100, 100, 10000]

평균 = (100×9 + 10000) / 10 = 10900 / 10 = 1090ms
p95 = 상위 5% = 10000ms (9.5번째)

아니다, 예시를 다시:

데이터: [50, 60, 70, 80, 90, 100, 150, 200, 250, 10000]

평균 = (50+60+70+80+90+100+150+200+250+10000) / 10 = 11050 / 10 = 1105ms
p95 = 상위 5% = (9.5번째) = 10000ms

다시:

데이터: [100, 100, 100, 100, 100, 100, 100, 100, 150, 500]

평균 = (100×8 + 150 + 500) / 10 = 1350 / 10 = 135ms
p95 = 90%보다 높은 값 = 90번째 퍼센타일 = 10개 중 9번째 = 500ms

아! 이제 맞습니다:

데이터 (정렬): [100, 100, 100, 100, 100, 100, 100, 100, 150, 500]
평균: 135ms (극값 500때문에 끌어올려짐)
p95: 500ms (상위 5% = 1개)

```

**핵심**: 평균은 **모든 값의 합**을 계산하므로, 하나의 극값(500)이 전체 평균을 올립니다.
반면 p95는 **순서대로 배열한 후 위치만 보므로**, 극값에 덜 영향받습니다.

</details>

---

**Q2**: 히스토그램에서 "오른쪽 꼬리"가 길다면, 정확히 어떤 요청이 느린 건가?

<details>
<summary>해설 보기</summary>

**개별 요청을 분석하지 못합니다.** 오직 "일부 요청이 느리다"는 것만 알 수 있습니다.

```javascript
// 원인 파악을 위해서는 "태그"를 사용해야 합니다:

export default function() {
  // API 종류별로 태그
  http.post('http://api.example.com/order', {...}, {
    tags: { endpoint: '/order', priority: 'high' },
  });
  
  http.get('http://api.example.com/products', {
    tags: { endpoint: '/products', priority: 'low' },
  });
}

// 결과:
// http_req_duration{endpoint:/order}: p95=200ms ✓
// http_req_duration{endpoint:/products}: p95=2000ms ✗ ← 이 API가 느림!

// 이제 원인 파악:
// - /products API 최적화 필요
// - DB 쿼리 개선 또는 캐싱 추가
```

**방법:**
1. 태그로 API/엔드포인트 분류
2. 각 태그별 p95 비교
3. 가장 느린 API 특정
4. 그 API만 깊이 있게 분석

</details>

---

**Q3**: p99.9를 설정하려면, 최소 몇 개의 샘플이 필요할까?

<details>
<summary>해설 보기</summary>

**최소 1000개 샘플**을 추천합니다.

```
p99 = 상위 1% = 100개 중 1개
p99.9 = 상위 0.1% = 1000개 중 1개

따라서:
- 100개 샘플 → p99는 부정확 (1개만으로 계산)
- 1000개 샘플 → p99.9는 부정확 (1개만으로 계산)
- 10000개 샘플 → p99.9는 합리적 (10개로 평균화 가능)
```

**실제 계산:**
```javascript
// VU 50, 테스트 시간 5분 = 300초
// 각 VU가 초당 1회 요청 = 50 VU × 300초 = 15,000 요청

// p99.9 신뢰도:
// 15,000 × 0.001 = 15개 샘플 (합리적)

// 하지만 VU 10, 테스트 시간 1분 = 600개 요청만 생기면:
// 600 × 0.001 = 0.6개 샘플 (너무 적음)
// → p99.9는 신뢰할 수 없음
```

**권장:**
- 단기 테스트: p95, p99만 사용
- 장시간 테스트 (1시간 이상): p99, p99.9 사용

</details>

---

<div align="center">

**[⬅️ 이전: HTTP 요청과 검증 — 로그인 토큰 재사용 패턴](./03-http-requests-validation.md)** | **[홈으로 🏠](../README.md)** | **[다음: k6 확장 — InfluxDB + Grafana 대시보드 연동 ➡️](./05-k6-extensions.md)**

</div>
