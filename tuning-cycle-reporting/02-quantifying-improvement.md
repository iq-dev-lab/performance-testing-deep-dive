# 02. 튜닝 효과 정량화 — p95/p99 비교표 작성

---

## 🎯 핵심 질문

"p99가 1200ms에서 650ms로 줄었어. 이게 비즈니스에는 얼마만큼 도움이 될까? 비용은 얼마나 절감할 수 있을까?"

수치가 없으면 비즈니스 결정을 할 수 없습니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

**개발팀과 비즈니스팀의 언어 차이:**

- 개발팀: "p99 성능 500ms 개선했습니다."
- 비즈니스팀: "그래서 매출이 얼마나 늘어나요?"

이 간극을 메우는 것이 성능 개선의 진정한 가치입니다.

**성능 개선 = 돈**

```
p99 응답시간 1초 → 0.5초로 단축
= 100명 중 1명이 기다리던 0.5초 절감
= 전환율 +3-5% (연구에 따르면)
= 매월 1억 원 추가 매출 (연 12억 원)
```

또한, 성능 개선으로 인한 인프라 비용 절감도 정량화해야:
- DB 인스턴스 다운그레이드 가능 여부
- 서버 대수 감소 가능 여부
- 대역폭 절감 가능 여부

이 모든 것을 **정확한 수치와 비교표**로 보여줄 때, 비로소 경영진과 개발팀이 같은 언어로 소통할 수 있습니다.

---

## 😱 흔한 실수 (Before)

### 사례 1: 단순 비교만 제시
```
성능 개선 보고서

Before: p99 = 1200ms
After: p99 = 650ms
개선율: 45.8%

결론: 성공!
```

**문제점:**
- 왜 45.8%가 중요한가?
- 비즈니스 영향은?
- 비용 절감은?
- 투자 대비 효과는?
- 답변 불가능

### 사례 2: 불완전한 지표 비교
```
변경 전: 평균 응답시간 250ms
변경 후: 평균 응답시간 180ms
개선: 28%

[문제 발생 3주일 후]
"100명 중 1명만 느린데 왜 자꾸 불평하지?"
→ p99를 비교하지 않아서 tail latency 문제를 놓침
```

**문제점:**
- 평균만 보면 top 1%의 고통 모름
- 사용자 불만의 원인: 대부분 tail latency
- p50, p95, p99를 모두 봐야 전체 상황 파악 가능

### 사례 3: 에러율 증가를 간과
```
성능 개선 보고서:
- p99: 1200ms → 500ms ✅ (58% 개선)
- TPS: 100 → 150 ✅ (50% 증가)

[실제 문제]
- 에러율: 0.2% → 2.5% ❌ (12배 증가!)
- DB 타임아웃으로 인한 사용자 경험 악화
```

**문제점:**
- 응답시간만 단축되고 안정성 악화
- 일부 사용자는 에러를 받음
- 비즈니스 관점에서는 실패

### 사례 4: 인프라 비용 절감 미계산
```
성능 개선만 보고
- p99 단축됨
- TPS 증가함

[실제 상황]
- 여전히 고사양 DB 인스턴스 필요 (비용 변화 없음)
- 캐시만 증가해서 메모리 오버헤드 발생
- 투자 대비 효과 불명확
```

---

## ✨ 올바른 접근 (After)

### 포괄적 성능 비교표

**Template: 성능 지표 종합 비교**

```markdown
## 성능 개선 결과 (총 3차 튜닝)

### 응답시간 비교

| 백분위수 | Before | After | 개선값 | 개선율 | 의미 |
|---------|--------|-------|--------|--------|------|
| **p50** | 200ms | 180ms | -20ms | -10% | 중간값 |
| **p75** | 400ms | 330ms | -70ms | -17.5% | 4명 중 3명 |
| **p95** | 600ms | 420ms | -180ms | -30% | 20명 중 19명 |
| **p99** | 1200ms | 650ms | -550ms | -45.8% | 100명 중 1명 |
| **평균** | 350ms | 280ms | -70ms | -20% | 전체 평균 |

**의미:**
- p99 기준: 100명 중 가장 느린 1명이 550ms 더 빨라짐
- p95 기준: 대부분의 사용자(95%)가 180ms 이상 빨라짐
```

### 처리량(TPS) 비교

```markdown
## 처리량 및 리소스 효율성

| 지표 | Before | After | 변화 | 평가 |
|------|--------|-------|------|------|
| **TPS** | 98 req/s | 155 req/s | +57 req/s (+58%) | ✅ |
| **p99 @ 동일 부하** | 1200ms | - | - | - |
| **p99 @ 1000 req/s** | N/A (불가능) | 950ms | - | ✅ 스케일링 개선 |

**의미:**
- 현재 부하에서 TPS 58% 증가
- 향후 동시 사용자 증가 시 더 안정적인 성능 제공 가능
```

### 에러율 비교

```markdown
## 안정성 비교

| 에러 유형 | Before | After | 변화 |
|----------|--------|-------|------|
| **전체 에러율** | 0.50% | 0.08% | -0.42% (-84%) |
| **Timeout 에러** | 0.35% | 0.02% | -0.33% (-94%) |
| **DB Connection 에러** | 0.10% | 0.01% | -0.09% (-90%) |
| **기타 에러** | 0.05% | 0.05% | 0% |

**의미:**
- 모든 타입의 에러가 감소
- 타임아웃으로 인한 사용자 경험 악화 거의 제거
```

---

## 🔬 내부 동작 원리

### 왜 p95/p99를 봐야 하는가?

**평균만으로는 불충분한 이유:**

```
시나리오 A:
모든 사용자가 500ms씩 기다림
→ 평균: 500ms, p99: 500ms

시나리오 B:
99%는 100ms, 1%는 50,000ms 기다림
→ 평균: 600ms, p99: 50,000ms

[관찰]
평균은 시나리오 A가 더 짧음 (500ms < 600ms)
하지만 사용자 경험:
- A: 모두가 같은 불편함
- B: 대부분은 빠르지만, 1%는 극악의 경험

[결론]
평균은 거짓말. 분포를 봐야 한다.
```

### 꼬리 분포(Tail Distribution)의 비즈니스 임팩트

```
응답시간 분포를 그래프로 표현:

Before (튜닝 전):
  빈도
   |     
  50|    ██
  40|    ██      ██
  30|    ██  ██  ██  ██
  20|    ██  ██  ██  ██      ██
  10|    ██  ██  ██  ██  ██  ██
   0|____________________________
     100 200 300 400 500 1000 응답시간(ms)

After (튜닝 후):
  빈도
   |     
 100|    ██████
  80|    ██████  ██
  60|    ██████  ██  ██
  40|    ██████  ██  ██  ██
  20|    ██████  ██  ██  ██
   0|____________________________
     100 200 300 400 500 꼬리 거의 없음

[관찰]
Before: 꼬리가 길고 넓음 → 극악의 경험 가능
After: 꼬리가 짧음 → 극악의 경험 거의 불가능
```

---

## 💻 실전 실험

### Step 1: k6에서 결과 추출

```bash
# k6 부하 테스트 실행 (JSON 형식 출력)
k6 run load-test.js --out json=results.json

# 결과 파일 구조 확인
cat results.json | jq '.' | head -20
```

**결과 예시:**
```json
{
  "type": "Metric",
  "metric": "http_req_duration",
  "data": {
    "value": 245,
    "tags": {
      "expected_response": "true",
      "status": "200"
    }
  },
  "time": "2024-03-15T10:00:01Z"
}
```

### Step 2: 결과를 CSV로 변환

```bash
# Node.js 스크립트로 JSON 파싱 및 CSV 생성
cat > extract-metrics.js << 'EOF'
const fs = require('fs');

const resultsFile = process.argv[2] || 'results.json';
const results = fs
  .readFileSync(resultsFile, 'utf-8')
  .split('\n')
  .filter(line => line.trim())
  .map(line => JSON.parse(line));

// HTTP 요청 지속시간 추출
const durations = results
  .filter(r => r.type === 'Metric' && r.metric === 'http_req_duration')
  .map(r => r.data.value)
  .sort((a, b) => a - b);

// 백분위수 계산 함수
function percentile(arr, p) {
  const idx = Math.ceil(arr.length * (p / 100)) - 1;
  return arr[Math.max(0, idx)];
}

// 에러율 계산
const totalRequests = results.filter(r => 
  r.type === 'Metric' && r.metric === 'http_req_duration'
).length;

const errorRequests = results.filter(r =>
  r.type === 'Metric' && r.metric === 'http_reqs_failed'
).reduce((sum, r) => sum + r.data.value, 0);

const errorRate = (errorRequests / totalRequests * 100).toFixed(2);

// 결과 출력
console.log('=== Performance Metrics ===');
console.log(`p50: ${percentile(durations, 50)}ms`);
console.log(`p75: ${percentile(durations, 75)}ms`);
console.log(`p95: ${percentile(durations, 95)}ms`);
console.log(`p99: ${percentile(durations, 99)}ms`);
console.log(`Mean: ${(durations.reduce((a, b) => a + b) / durations.length).toFixed(0)}ms`);
console.log(`Min: ${Math.min(...durations)}ms`);
console.log(`Max: ${Math.max(...durations)}ms`);
console.log(`\nError Rate: ${errorRate}%`);
console.log(`Total Requests: ${totalRequests}`);

// CSV로 저장
const csv = `percentile,duration_ms
p50,${percentile(durations, 50)}
p75,${percentile(durations, 75)}
p95,${percentile(durations, 95)}
p99,${percentile(durations, 99)}
mean,${(durations.reduce((a, b) => a + b) / durations.length).toFixed(0)}`;

fs.writeFileSync('metrics.csv', csv);
console.log('\nSaved to metrics.csv');
EOF

node extract-metrics.js results.json
```

### Step 3: 비교 분석 스크립트

```bash
# 두 개의 결과 파일을 비교
cat > compare-improvements.js << 'EOF'
const fs = require('fs');

function extractMetrics(filename) {
  const lines = fs.readFileSync(filename, 'utf-8').split('\n').filter(l => l);
  const durations = lines
    .map(l => JSON.parse(l))
    .filter(r => r.type === 'Metric' && r.metric === 'http_req_duration')
    .map(r => r.data.value)
    .sort((a, b) => a - b);

  function p(arr, percentile) {
    const idx = Math.ceil(arr.length * (percentile / 100)) - 1;
    return arr[Math.max(0, idx)];
  }

  return {
    p50: p(durations, 50),
    p75: p(durations, 75),
    p95: p(durations, 95),
    p99: p(durations, 99),
    mean: durations.reduce((a, b) => a + b) / durations.length,
  };
}

const before = extractMetrics('baseline.json');
const after = extractMetrics('after-tuning.json');

console.log('\n=== 성능 개선 비교 ===\n');
console.log('| 백분위수 | Before | After | 개선값 | 개선율 |');
console.log('|---------|--------|-------|--------|--------|');

const percentiles = ['p50', 'p75', 'p95', 'p99', 'mean'];
percentiles.forEach(p => {
  const beforeVal = before[p].toFixed(0);
  const afterVal = after[p].toFixed(0);
  const diff = (before[p] - after[p]).toFixed(0);
  const rate = ((before[p] - after[p]) / before[p] * 100).toFixed(1);
  console.log(`| ${p.padEnd(7)} | ${beforeVal.padStart(6)}ms | ${afterVal.padStart(5)}ms | ${diff.padStart(6)}ms | ${rate.padStart(5)}% |`);
});

console.log('\n');
EOF

node compare-improvements.js
```

### Step 4: 비즈니스 임팩트 계산

```bash
# 비즈니스 지표로 변환
cat > business-impact.js << 'EOF'
// 성능 개선 → 비즈니스 지표 변환

const metrics = {
  p99Before: 1200,  // ms
  p99After: 650,    // ms
  tpsBefore: 98,    // req/s
  tpsAfter: 155,    // req/s
  errorRateBefore: 0.50,  // %
  errorRateAfter: 0.08,   // %
  monthlyUsers: 1000000,  // 월간 사용자
  conversionRateBefore: 0.02,  // 2%
};

// 1. 응답시간 개선이 전환율에 미치는 영향
// 연구에 따르면: 응답시간 100ms 단축 = 전환율 +1%
const p99Improvement = (metrics.p99Before - metrics.p99After) / 100; // 550 / 100 = 5.5
const conversionRateBoost = p99Improvement * 0.01; // 5.5 * 1% = 5.5%

const conversionRateAfter = metrics.conversionRateBefore + conversionRateBoost;
const additionalConversions = metrics.monthlyUsers * conversionRateBoost;
const aov = 50000; // 평균 주문 금액 (원)
const additionalRevenue = additionalConversions * aov;

// 2. 에러율 감소 = 고객 유지율 개선
const errorReduction = metrics.errorRateBefore - metrics.errorRateAfter;
const retainedCustomers = metrics.monthlyUsers * errorReduction;

// 3. 인프라 비용 절감
// TPS 58% 증가 = 동일 성능에서 필요한 서버 수 감소
const tpsIncrease = (metrics.tpsAfter - metrics.tpsBefore) / metrics.tpsBefore;
const serverReduction = Math.ceil(10 * (1 - 1 / (1 + tpsIncrease))); // 10대 중
const costPerServer = 2000000; // 월간 비용 (원)
const serverCostSavings = serverReduction * costPerServer;

// 출력
console.log('=== 비즈니스 임팩트 분석 ===\n');
console.log('📊 매출 증대:');
console.log(`  • p99 ${metrics.p99Before}ms → ${metrics.p99After}ms 개선`);
console.log(`  • 전환율: ${(metrics.conversionRateBefore * 100).toFixed(1)}% → ${(conversionRateAfter * 100).toFixed(2)}% (+${(conversionRateBoost * 100).toFixed(2)}%)`);
console.log(`  • 추가 전환: ${additionalConversions.toFixed(0)}건/월`);
console.log(`  • 추가 매출: ${(additionalRevenue / 1000000).toFixed(0)}백만 원/월 (연간 ${(additionalRevenue * 12 / 1000000).toFixed(0)}백만 원)`);

console.log('\n👥 고객 유지율 개선:');
console.log(`  • 에러율: ${metrics.errorRateBefore.toFixed(2)}% → ${metrics.errorRateAfter.toFixed(2)}% (-${errorReduction.toFixed(2)}%)`);
console.log(`  • 보존 고객: ${retainedCustomers.toFixed(0)}명/월`);

console.log('\n💰 인프라 비용 절감:');
console.log(`  • TPS: ${metrics.tpsBefore} → ${metrics.tpsAfter} (+${tpsIncrease * 100 > 0 ? '+' : ''}${(tpsIncrease * 100).toFixed(1)}%)`);
console.log(`  • 서버 감소: ${serverReduction}대`);
console.log(`  • 월간 절감: ${(serverCostSavings / 1000000).toFixed(0)}백만 원`);
console.log(`  • 연간 절감: ${(serverCostSavings * 12 / 1000000).toFixed(0)}백만 원`);

console.log('\n🎯 총 비즈니스 가치:');
const totalMonthlyValue = (additionalRevenue + serverCostSavings) / 1000000;
console.log(`  • 월간: ${totalMonthlyValue.toFixed(0)}백만 원`);
console.log(`  • 연간: ${(totalMonthlyValue * 12).toFixed(0)}백만 원`);
EOF

node business-impact.js
```

---

## 📊 성능 비교

### 실제 사례: 전자상거래 주문 API

#### Before (튜닝 전)

| 지표 | 값 |
|------|-----|
| p50 | 200ms |
| p75 | 400ms |
| p95 | 600ms |
| p99 | 1200ms |
| 평균 | 350ms |
| TPS | 98 req/s |
| 에러율 | 0.50% |
| 메모리 | 512MB |

#### After (튜닝 후)

| 지표 | 값 |
|------|-----|
| p50 | 180ms |
| p75 | 330ms |
| p95 | 420ms |
| p99 | 650ms |
| 평균 | 280ms |
| TPS | 155 req/s |
| 에러율 | 0.08% |
| 메모리 | 580MB |

#### 비교 분석

| 지표 | Before | After | 개선값 | 개선율 | 평가 |
|------|--------|-------|--------|--------|------|
| **p99** | 1200ms | 650ms | -550ms | -45.8% | ✅ 매우 우수 |
| **p95** | 600ms | 420ms | -180ms | -30.0% | ✅ 우수 |
| **평균** | 350ms | 280ms | -70ms | -20.0% | ✅ 양호 |
| **TPS** | 98 | 155 | +57 | +58.2% | ✅ 매우 우수 |
| **에러율** | 0.50% | 0.08% | -0.42% | -84.0% | ✅ 매우 우수 |

### 비즈니스 임팩트 정량화

#### 1. 매출 증대

```
p99 응답시간 개선: 1200ms → 650ms (550ms = 5.5 × 100ms)

연구 근거 (Amazon, Google):
- 응답시간 100ms 단축 = 전환율 +1%

계산:
- 월간 사용자: 1,000,000명
- 현재 전환율: 2%
- 개선 후 전환율: 2% + 5.5% = 7.5%
- 추가 전환: 1,000,000 × 5.5% = 55,000건
- 평균 주문액: 50,000원
- 추가 매출: 55,000 × 50,000 = 2,750,000,000원/월
- 연간: 33,000,000,000원

🎉 연 33억 원의 매출 증대!
```

#### 2. 고객 유지 개선

```
에러율 감소: 0.50% → 0.08% (0.42% 감소)

계산:
- 타임아웃으로 실패하는 사용자: 1,000,000 × 0.42% = 4,200명/월
- 이들 중 90%는 다시 돌아옴: 4,200 × 0.9 = 3,780명
- 평생 가치 (LTV): 500,000원 (예상)
- 보존 가치: 3,780 × 500,000 = 1,890,000,000원/월
- 연간: 22,680,000,000원

🎉 연 22억 6천만 원의 고객 보존 가치!
```

#### 3. 인프라 비용 절감

```
TPS 증가: 98 → 155 (+58.2%)

의미: 동일한 응답시간(p99 650ms)에서 58% 더 많은 사용자 처리 가능

현재 서버 구성: 10대의 웹 서버
필요 서버 수 감소: 10 × (1 - 155/98) = 약 감소 불가능?
→ 역으로 생각: 10대로 155%의 부하 처리 = 1.55배 용량 증가
→ 또는: 동일 부하 시 필요 서버 수 = 10 × (98/155) = 6.3대 = 약 4대 감소

월간 서버 비용:
- DB 인스턴스 다운그레이드: r5.xlarge → r5.large = 월 200만 원 절감
- 캐시 비용 증가: +50만 원 (메모리 증가)
- 순 절감: 150만 원/월
- 연간: 1,800만 원

🎉 연 1천 8백만 원의 비용 절감!
```

#### 4. 총 비즈니스 가치

```
매출 증대:         33,000,000,000원/년
고객 보존:         22,680,000,000원/년
비용 절감:             18,000,000원/년
───────────────────────────────────
총 가치:           55,698,000,000원/년

= 약 56억 원!
```

---

## ⚖️ 트레이드오프

### 성능 vs 메모리

```
튜닝으로 인한 메모리 증가:
- DB 풀 크기 증가: +50MB
- 캐시 TTL 증가: +18MB
- 총 증가: +68MB

월간 비용: 50만 원
연간 비용: 600만 원

비즈니스 임팩트:
- 매출 증대: 55,698,000,000원
- 메모리 비용: 6,000,000원
- 순 이득: 55,692,000,000원

✅ 메모리 비용은 무시할 수 있는 수준
```

### 성능 vs 관리 복잡도

```
증가한 설정 항목:
1. DB 풀 크기 (30→50): 모니터링 필요
2. 타임아웃 (5s→8s): 튜닝 가능성
3. 캐시 TTL (300s→600s): 데이터 신선도 트레이드

관리 비용:
- 추가 모니터링: 주 2시간
- 분기별 튜닝: 주 4시간
- 월 추정 비용: 20만 원

✅ 비즈니스 가치(56억)에 비해 무시할 수 있는 수준
```

---

## 📌 핵심 정리

1. **p50, p95, p99를 모두 봐야 한다**
   - 평균은 거짓
   - 사용자 경험은 tail distribution에서 결정됨

2. **에러율을 함께 기록하라**
   - 성능만 좋아지고 에러가 늘면 실패
   - 안정성과 성능의 균형이 중요

3. **비즈니스 언어로 번역하라**
   - "p99 45% 개선" → "연 56억 원의 비즈니스 가치"
   - 경영진과 개발팀의 소통 가능

4. **인프라 비용 절감을 계산하라**
   - 성능 개선 = 인스턴스 다운그레이드 가능성
   - 비용 절감도 비즈니스 가치

5. **모든 수치를 비교표로 정리하라**
   - Before/After 동시 표시
   - 개선값과 개선율 명시
   - 비즈니스 임팩트 함께 기재

---

## 🤔 생각해볼 문제

### Q1: p95와 p99 중 어디에 집중해야 할까?

```
p95: 20명 중 19명의 경험 = 대부분의 사용자
p99: 100명 중 1명의 경험 = 극소수지만 고통

어디에 집중해야 하는가?
```

<details>
<summary>해설 보기</summary>

**상황에 따라 다릅니다:**

1. **전자상거래 (결제/주문):**
   - → p99 집중
   - 이유: 100명 중 1명이 실패하면 → 연간 4,000명 × 50,000원 = 2억 원 손실
   - "극소수"가 "극악의 경험"을 함

2. **뉴스 피드 조회:**
   - → p95 집중
   - 이유: 대부분의 사용자가 영향받음
   - p99는 극히 드문 사례

3. **결제 게이트웨이:**
   - → p99 + 에러율 집중
   - 이유: 0.1%의 실패도 신뢰도 크게 하락
   - 금융 거래는 "거의 완벽"이어야 함

**결론:**
- B2C 거래 기능: p99 중시
- 정보 조회 기능: p95 중시
- 금융 결제: p99 + 에러율 = 0

</details>

### Q2: 에러율 증가가 발생했을 때 어떻게 판단할까?

```
성능 개선:
- p99: 1200ms → 500ms (58% ✅)
- TPS: 100 → 200 (100% ✅)

문제:
- 에러율: 0.1% → 2.0% (20배 증가 ❌)

투자가 가치가 있을까?
```

<details>
<summary>해설 보기</summary>

**비용-편익 분석:**

1. **에러로 인한 손실:**
   ```
   월간 요청: 100,000,000
   이전 에러: 100,000 × 0.1% = 100,000건
   현재 에러: 100,000 × 2.0% = 2,000,000건
   추가 에러: 1,900,000건
   
   에러당 손실: 5,000원 (고객 불만, 재처리 비용)
   월간 손실: 1,900,000 × 5,000 = 9,500,000,000원
   ```

2. **성능 개선으로 인한 이득:**
   ```
   전환율 향상: +3% (500ms 개선으로)
   추가 매출: 500,000명 × 3% × 50,000원 = 750,000,000원
   ```

3. **판단:**
   ```
   손실 (95억) > 이득 (7.5억)
   → 이 튜닝은 실패. 롤백해야 함!
   ```

**교훈:**
- 성능 개선도 중요하지만 안정성이 더 중요
- 에러율 증가는 즉시 롤백 대상
- "모든 지표가 개선"되어야만 성공

</details>

### Q3: 비즈니스팀이 "p99를 500ms로 줄여달라"고 요청했는데, 테스트 결과 650ms만 가능하면?

```
목표: p99 < 500ms
달성: p99 = 650ms
비즈니스 요구사항: 미달성

어떻게 해야 할까?
```

<details>
<summary>해설 보기</summary>

**전략적 접근:**

1. **"650ms의 비즈니스 임팩트" 제시:**
   ```
   p99 650ms로도:
   - 전환율 +4.5% (전환율 2% → 6.5%)
   - 월간 추가 매출: 2.25억 원
   - 에러율 84% 감소로 고객 만족도 향상
   
   vs 500ms 목표:
   - 전환율 +5% (전환율 2% → 7%)
   - 월간 추가 매출: 2.5억 원
   - 차이: 250만 원/월 = 3,000만 원/년
   ```

2. **추가 투자 vs 비용 분석:**
   ```
   650ms → 500ms를 위한 추가 튜닝:
   - 쿼리 최적화: 2주 개발
   - 인덱싱: 1주 개발
   - 총 3주 × 500만 원 = 1,500만 원 개발 비용
   
   vs 이득:
   - 추가 매출: 3,000만 원/년
   
   → 비용 대비 효과: 양호하지만 경계선
   ```

3. **협상:**
   ```
   "650ms로 먼저 가고,
   실제 사용자 반응을 3개월 보자.
   
   만약 전환율이 목표치에 미달하면
   그때 500ms 목표로 추가 튜닝하자."
   ```

**핵심:**
- 숫자만으로 협상하지 말 것
- 비즈니스 임팩트로 대화할 것
- 점진적 접근이 종종 최적

</details>

---

<div align="center">

**[⬅️ 이전: 과학적 튜닝 접근법 — 하나씩 변경하는 이유](./01-scientific-tuning-approach.md)** | **[홈으로 🏠](../README.md)** | **[다음: 성능 테스트 결과 보고서 — 비기술 직군을 위한 작성법 ➡️](./03-performance-report.md)**

</div>
