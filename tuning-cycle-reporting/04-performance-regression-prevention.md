# 04. 성능 회귀 방지 — CI Pipeline 자동화

---

## 🎯 핵심 질문

"우리가 힘들게 튜닝한 성능을 어떻게 지켜낼까? 누군가 모르고 설정을 바꿔서 성능이 떨어지면?"

자동화된 성능 테스트가 없으면 **성능 회귀는 필연입니다.**

---

## 🔍 왜 이 개념이 실무에서 중요한가

**실제 시나리오:**

```
Timeline:

2024-03-15: p99 성능 튜닝 완료 (1200ms → 650ms)
2024-04-01: 신입 개발자 온보딩
2024-04-05: 신입이 캐시 설정 변경 (합리적 이유 있음)
           → p99 650ms → 1000ms로 악화
2024-04-20: 버그 리포트 "주문 API가 느려요"
2024-04-21: 원인 분석 시작 (2주 지난 후!)
2024-04-25: 원인 파악: 캐시 설정 변경
2024-04-26: 롤백 및 수정
2024-05-01: 3주 동안 월 2-3억 원 손실

= "자동화된 성능 테스트가 있었으면 4월 5일에 바로 알았을 텐데"
```

**성능 회귀 방지의 가치:**

1. **즉시 탐지:** 변경 시점에 바로 알림
2. **원인 파악 용이:** 최근 변경 사항 중심으로 분석
3. **신속한 대응:** 피해 최소화
4. **신뢰도 향상:** "성능 저하"를 안심하고 사용

---

## 😱 흔한 실수 (Before)

### 사례 1: 성능 테스트가 없는 경우
```
개발 프로세스:
1. 코드 작성
2. 단위 테스트 ✅
3. 통합 테스트 ✅
4. 성능 테스트? ❌
5. 배포

[결과]
버그는 없는데 성능이 자꾸 악화됨
언제 악화되었는지 모름
```

**문제점:**
- 성능 회귀는 탐지 불가
- 악화 시점 파악 불가
- 원인 분석에 시간 소요

### 사례 2: 수동 성능 테스트
```
배포 전 체크리스트:
☐ 빌드 성공
☐ 테스트 통과
☐ 코드 리뷰 완료
☐ "성능 좋아보이네" (주관적 판단)
☐ 배포

문제:
- 매번 실행하지 않음
- 누가 해야 하는지 불명확
- 정량적 기준 없음
```

**문제점:**
- 일관성 없음
- 주관적 판단
- 자동화 효과 없음

### 사례 3: 성능 테스트는 있지만 기준값 미설정
```
배포 후:
테스트 결과:
p99: 950ms
"흠... 이게 좋은 건가 나쁜 건가?"

비교 기준이 없어서:
- 그냥 배포
- 나중에 느려졌다는 리포트 받음
```

**문제점:**
- "pass/fail" 기준 없음
- 정량적 판단 불가능

### 사례 4: 경보 설정 없음
```
매일 성능 테스트 실행하지만
아무도 결과를 안 봤음

결과:
- p99 650ms → 1200ms로 악화 (한 달간)
- 누구도 모름
- 손실 2-3억 원
```

**문제점:**
- 자동화는 했지만 알림 없음
- "돌고 있다"는 자기기만
- 의미 없는 자동화

---

## ✨ 올바른 접근 (After)

### 성능 회귀 방지 전략

```
1단계: Baseline 설정
   ↓
2단계: 자동화된 성능 테스트
   ↓
3단계: 임계값(Threshold) 설정
   ↓
4단계: CI Pipeline 통합
   ↓
5단계: 자동 경보 (Slack 등)
   ↓
6단계: 자동 롤백 또는 차단
```

### 1단계: Baseline 설정

```bash
# 현재 성능 측정 (신뢰할 수 있는 환경에서)
k6 run performance-test.js \
  --vus 100 \
  --duration 5m \
  --out json=baseline.json

# 결과 저장
cat baseline.json | jq '.metrics | 
{
  p50: .http_req_duration[0].values.p(50),
  p95: .http_req_duration[0].values.p(95),
  p99: .http_req_duration[0].values.p(99),
  error_rate: .http_reqs_failed[0].value
}' > baseline.json
```

**Baseline 기준값:**
```json
{
  "p50": 180,
  "p95": 420,
  "p99": 650,
  "error_rate": 0.0008,
  "tps": 155
}
```

---

## 🔬 내부 동작 원리

### k6 Thresholds 개념

```javascript
// k6에서 지원하는 성능 기준 정의

export const options = {
  thresholds: {
    // HTTP 응답시간 기준
    'http_req_duration': [
      'p(95)<500',        // p95가 500ms 이하여야 pass
      'p(99)<1000',       // p99가 1초 이하여야 pass
      'p(99)<650',        // p99가 650ms (baseline) 이하여야 pass
    ],
    
    // 에러 기준
    'http_req_failed': [
      'rate<0.01',        // 에러율 1% 이하
    ],
    
    // 처리량 기준
    'http_reqs': [
      'count>10000',      // 최소 10,000 requests 수행
    ],
  },
};
```

**동작:**
- Threshold 위반 → 테스트 FAIL
- CI Pipeline에서 빌드 중단
- 배포 차단

### Baseline 대비 상대적 기준

```javascript
// 절대값 대신 baseline 대비 % 기준 설정

const baseline = {
  p99: 650,  // ms
  errorRate: 0.0008,
};

export const options = {
  thresholds: {
    // baseline 대비 10% 이상 악화되면 fail
    'http_req_duration': [
      `p(99)<${baseline.p99 * 1.1}`,  // 715ms 이상 허용
    ],
    
    // baseline 대비 50% 이상 악화되면 fail
    'http_req_failed': [
      `rate<${baseline.errorRate * 1.5}`,  // 0.0012 이상 허용
    ],
  },
};
```

---

## 💻 실전 실험

### Step 1: k6 성능 테스트 스크립트

```bash
# performance-test.js 작성
cat > performance-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';

// Baseline 값
const BASELINE = {
  p99: 650,      // ms
  p95: 420,      // ms
  errorRate: 0.0008,
};

export const options = {
  // 부하 설정
  stages: [
    { duration: '1m', target: 50 },  // 50 VU로 1분
    { duration: '3m', target: 100 }, // 100 VU로 3분
    { duration: '1m', target: 50 },  // 50 VU로 1분
  ],
  
  // 성능 기준 (Thresholds)
  thresholds: {
    // p99는 baseline 대비 ±10% 범위
    'http_req_duration{staticAsset:no}': [
      'p(99)<' + (BASELINE.p99 * 1.1),    // 715ms 이상 fail
      'p(95)<' + (BASELINE.p95 * 1.15),   // 483ms 이상 fail
    ],
    
    // 에러율은 baseline 대비 ±50% 범위
    'http_req_failed{staticAsset:no}': [
      'rate<' + (BASELINE.errorRate * 1.5), // 0.0012 이상 fail
    ],
  },
};

export default function () {
  // 주문 API 테스트
  let response = http.get('http://api.example.com/products/1', {
    tags: { staticAsset: 'no' },
  });
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 1s': (r) => r.timings.duration < 1000,
    'has product name': (r) => r.body.includes('productName'),
  });
  
  sleep(1);
}
EOF
```

### Step 2: GitHub Actions CI Pipeline 설정

```bash
# .github/workflows/performance-test.yml
cat > .github/workflows/performance-test.yml << 'EOF'
name: Performance Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  performance:
    runs-on: ubuntu-latest
    
    steps:
      # 1. 코드 체크아웃
      - uses: actions/checkout@v3
      
      # 2. Node.js 설치
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      # 3. k6 설치
      - name: Install k6
        run: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6-stable.list
          sudo apt-get update
          sudo apt-get install -y k6
      
      # 4. 애플리케이션 시작 (배경 실행)
      - name: Start application
        run: |
          # 실제로는 docker, docker-compose 등으로 시작
          docker-compose up -d
          sleep 10  # 애플리케이션 시작 대기
      
      # 5. 성능 테스트 실행
      - name: Run performance test
        id: perf_test
        run: |
          k6 run performance-test.js \
            --out json=performance-results.json \
            --summary-export=summary.json \
            2>&1 | tee perf-test.log
          
          # 테스트 결과 저장
          echo "test_output=$(cat perf-test.log)" >> $GITHUB_OUTPUT
      
      # 6. 성능 결과 파싱
      - name: Parse performance results
        id: parse_results
        run: |
          node << 'ENDSCRIPT'
          const fs = require('fs');
          const results = JSON.parse(fs.readFileSync('performance-results.json'));
          
          const p99 = results.metrics.http_req_duration?.[0]?.values?.['p(99)'] || 0;
          const p95 = results.metrics.http_req_duration?.[0]?.values?.['p(95)'] || 0;
          const errorRate = results.metrics.http_req_failed?.[0]?.value || 0;
          
          console.log(`p99=${p99}ms, p95=${p95}ms, errorRate=${(errorRate*100).toFixed(2)}%`);
          
          // GitHub output에 저장
          fs.appendFileSync(process.env.GITHUB_OUTPUT, 
            `results=p99:${Math.round(p99)}ms p95:${Math.round(p95)}ms errorRate:${(errorRate*100).toFixed(2)}%\n`
          );
          ENDSCRIPT
      
      # 7. Slack 알림 (성공)
      - name: Notify Slack (success)
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Performance Test Passed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Performance Test Result*\n${{ steps.parse_results.outputs.results }}\n:green_circle: All thresholds passed!"
                  }
                },
                {
                  "type": "context",
                  "elements": [
                    {
                      "type": "mrkdwn",
                      "text": "Commit: <${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      # 8. Slack 알림 (실패)
      - name: Notify Slack (failure)
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Performance Test Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Performance Test Failed!*\n${{ steps.parse_results.outputs.results }}\n:red_circle: Some thresholds exceeded"
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "```\n${{ steps.perf_test.outputs.test_output }}\n```"
                  }
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Build"
                      },
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      # 9. 빌드 실패 처리
      - name: Fail build if thresholds exceeded
        run: |
          if grep -q "threshold.*exceeded" perf-test.log; then
            echo "❌ Performance thresholds exceeded. Build failed."
            exit 1
          fi
      
      # 10. 성능 결과 아티팩트로 저장
      - name: Upload performance results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: performance-test-results
          path: |
            performance-results.json
            summary.json
            perf-test.log

EOF
```

### Step 3: Slack 웹훅 설정

```bash
# 1. Slack에서 Incoming Webhook 생성
# https://api.slack.com/messaging/webhooks
# → "Create New App" → "Incoming Webhooks" → URL 복사

# 2. GitHub Secrets에 저장
# GitHub Repo → Settings → Secrets and variables → Actions
# → New repository secret
# 이름: SLACK_WEBHOOK_URL
# 값: https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# 3. 테스트
curl -X POST \
  -H 'Content-type: application/json' \
  -d '{"text":"Test"}' \
  https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### Step 4: Baseline 자동 업데이트 전략

```bash
# baseline-update.js - 성능 개선 후 baseline 업데이트

cat > baseline-update.js << 'EOF'
const fs = require('fs');
const path = require('path');

// 현재 성능 테스트 결과 읽기
const resultsFile = process.argv[2] || 'performance-results.json';
const results = JSON.parse(fs.readFileSync(resultsFile));

// 기존 baseline 읽기
const baselineFile = 'baseline.json';
let baseline = {};
if (fs.existsSync(baselineFile)) {
  baseline = JSON.parse(fs.readFileSync(baselineFile));
}

// 새로운 값 계산
const newBaseline = {
  p99: results.metrics.http_req_duration?.[0]?.values?.['p(99)'] || baseline.p99,
  p95: results.metrics.http_req_duration?.[0]?.values?.['p(95)'] || baseline.p95,
  errorRate: results.metrics.http_req_failed?.[0]?.value || baseline.errorRate,
  updatedAt: new Date().toISOString(),
};

// Baseline 개선된 경우만 업데이트
if (newBaseline.p99 < baseline.p99 * 0.95) {
  // p99가 5% 이상 개선되면 업데이트
  console.log(`✅ Performance improved! Updating baseline...`);
  console.log(`  p99: ${baseline.p99}ms → ${newBaseline.p99}ms`);
  
  fs.writeFileSync(baselineFile, JSON.stringify(newBaseline, null, 2));
  process.exit(0);
} else if (newBaseline.p99 > baseline.p99 * 1.1) {
  // p99가 10% 이상 악화되면 경고
  console.error(`❌ Performance degraded!`);
  console.error(`  p99: ${baseline.p99}ms → ${newBaseline.p99}ms`);
  process.exit(1);
} else {
  console.log(`✓ Performance stable. Baseline unchanged.`);
  process.exit(0);
}
EOF

node baseline-update.js performance-results.json
```

### Step 5: 성능 히스토리 추적

```bash
# performance-history.js - 성능 변화 추적 및 그래프화

cat > performance-history.js << 'EOF'
const fs = require('fs');

// 히스토리 파일
const historyFile = 'performance-history.json';

function getHistory() {
  if (!fs.existsSync(historyFile)) {
    return [];
  }
  return JSON.parse(fs.readFileSync(historyFile));
}

function saveHistory(history) {
  fs.writeFileSync(historyFile, JSON.stringify(history, null, 2));
}

function addRecord(resultsFile) {
  const results = JSON.parse(fs.readFileSync(resultsFile));
  
  const record = {
    timestamp: new Date().toISOString(),
    commit: process.env.GITHUB_SHA || 'unknown',
    p99: results.metrics.http_req_duration?.[0]?.values?.['p(99)'] || 0,
    p95: results.metrics.http_req_duration?.[0]?.values?.['p(95)'] || 0,
    errorRate: results.metrics.http_req_failed?.[0]?.value || 0,
  };
  
  const history = getHistory();
  history.push(record);
  
  // 최근 30개만 유지
  if (history.length > 30) {
    history.shift();
  }
  
  saveHistory(history);
  
  // 그래프로 출력
  console.log('\n📊 Performance History (Last 10):');
  console.log('Date              p99    p95    Error%');
  console.log('─'.repeat(40));
  
  const recent = history.slice(-10);
  recent.forEach(r => {
    const date = new Date(r.timestamp).toLocaleDateString();
    console.log(
      `${date} ${String(r.p99).padStart(6)}ms ${String(r.p95).padStart(6)}ms ${(r.errorRate*100).toFixed(2)}%`
    );
  });
}

addRecord(process.argv[2] || 'performance-results.json');
EOF

# 히스토리 추가
node performance-history.js performance-results.json
```

---

## 📊 성능 비교

### CI Pipeline 적용 전후

| 항목 | Before (수동) | After (자동화) | 개선 |
|------|---------------|----------------|------|
| **성능 회귀 탐지** | 2-3주 후 | 변경 직후 | 즉시 |
| **원인 파악** | 어려움 | 쉬움 (최근 변경) | 명확 |
| **대응 속도** | 느림 | 빠름 (자동) | 수동 vs 자동 |
| **테스트 비용** | 월 40시간 | 자동 (거의 0) | 생산성 향상 |
| **신뢰도** | 낮음 | 높음 | 문화 개선 |

### 성능 회귀 방지 효과

```
시나리오: 매달 성능 테스트에서 회귀 발견

Before (수동):
- 탐지 지연: 15일 평균
- 손실 비용: 월 2-3억 원
- 복구 시간: 2-3일

After (자동화):
- 탐지 지연: < 1시간
- 손실 비용: < 1천만 원 (즉시 차단)
- 복구 시간: < 1시간 (자동 롤백)

연간 절감: 24개월 × 2억 원 = 48억 원!
```

---

## ⚖️ 트레이드오프

### 자동화 수준 vs 설정 복잡도

| 레벨 | 기능 | 설정 시간 | 효과 | 추천 |
|------|------|---------|------|------|
| **1단계** | k6 스크립트만 | 2시간 | 기본 보호 | ✅ 필수 |
| **2단계** | + GitHub Actions | 4시간 | 자동 실행 | ✅ 권장 |
| **3단계** | + Slack 경보 | 1시간 | 즉시 알림 | ✅ 권장 |
| **4단계** | + 자동 롤백 | 4시간 | 자동 복구 | ⚠️ 신중 |
| **5단계** | + 성능 히스토리 추적 | 3시간 | 트렌드 분석 | ⚠️ 선택 |

**추천:**
- 초기: 1-3단계부터 시작 (총 7시간)
- 성숙도 높아지면: 4-5단계 추가

### 경보 민감도 vs 거짓 경보

```
너무 민감한 기준:
- p99 < 600ms (baseline 650ms에서 -7.7%)
- 장점: 회귀를 빨리 잡음
- 단점: 거짓 경보 많음 (주 3-4회)
- 비용: 개발자 피로도 증가

적절한 기준:
- p99 < 715ms (baseline 650ms에서 +10%)
- 장점: 실제 문제만 경보
- 단점: 약간의 완화된 성능도 허용
- 비용: 합리적 트레이드오프

너무 관대한 기준:
- p99 < 1000ms (baseline 650ms에서 +53%)
- 장점: 거짓 경보 거의 없음
- 단점: 실제 성능 악화를 놓칠 수 있음
- 비용: 회귀 탐지 불가

✅ 추천: 적절한 기준 (±10-15%)
```

---

## 📌 핵심 정리

1. **Baseline을 설정하라**
   - 현재의 신뢰할 수 있는 성능값
   - 모든 비교의 기준

2. **k6 Thresholds를 사용하라**
   - pass/fail 기준 명시
   - 정량적 의사결정

3. **CI Pipeline에 통합하라**
   - 모든 push/PR에서 자동 실행
   - 배포 전 검사

4. **Slack 알림을 설정하라**
   - 실패 시 즉시 알림
   - 팀이 반응 가능하도록

5. **성능 히스토리를 추적하라**
   - 장기 트렌드 파악
   - 점진적 악화 조기 발견

6. **Baseline을 적절히 업데이트하라**
   - 성능 개선 후: baseline 상향
   - 성능 악화: 원인 분석 후 대응

---

## 🤔 생각해볼 문제

### Q1: threshold 기준값을 어떻게 정할까?

```
baseline p99 = 650ms

옵션 1: p99 < 650ms (절대값, 악화 불가)
옵션 2: p99 < 715ms (baseline + 10%)
옵션 3: p99 < 780ms (baseline + 20%)

어느 것이 최선?
```

<details>
<summary>해설 보기</summary>

**상황에 따라 다릅니다:**

1. **옵션 1 (절대값):**
   ```
   장점: 매우 엄격 (성능 유지)
   단점: 거짓 경보 많음 (매 변경마다 실패 가능)
   
   사용 사례: 금융거래, 매우 중요한 API
   ```

2. **옵션 2 (baseline +10%):**
   ```
   장점: 실제 문제만 경보
   단점: 약간의 완화 허용
   
   사용 사례: 대부분의 프로덕션 API
   추천 ✅
   ```

3. **옵션 3 (baseline +20%):**
   ```
   장점: 거짓 경보 거의 없음
   단점: 눈에 띄는 성능 저하를 놓칠 수 있음
   
   사용 사례: 비-중요 API, 초기 단계
   ```

**결론:**
```
초기: 옵션 3 (낮은 부담)
→ 3개월 후: 옵션 2로 강화 (적응 후 엄격화)
→ 6개월 후: 옵션 1로 강화 (성숙도 높음)
```

**개별 API별로 다르게 설정 가능:**
```javascript
// 매우 중요한 API (결제)
'payment_api': { p99: '<650' },

// 중요 API (주문)
'order_api': { p99: '<715' },  // baseline + 10%

// 일반 API (조회)
'product_api': { p99: '<780' },  // baseline + 20%
```

</details>

### Q2: Threshold를 초과했을 때 자동으로 빌드를 실패시켜도 될까?

```
성능이 예상보다 조금 늘었다.
하지만 논리적으로는 변경이 필요한 상황.

자동 차단이 맞을까?
```

<details>
<summary>해설 보기</summary>

**전략적 접근:**

1. **자동 차단 (Strict):**
   ```
   장점: 규율 유지, 성능 저하 원천 차단
   단점: 때때로 정당한 변경도 차단
   
   사용: 금융, 의료 등 극도로 중요한 시스템
   ```

2. **경고만 (Warning):**
   ```
   장점: 유연성 있음, 필요한 변경 가능
   단점: 무시할 가능성
   
   사용: 일반 웹 서비스
   ```

3. **수동 검토 필수:**
   ```
   threshold 실패 → 자동으로 요청자에게 알림
   → 개발자/PM이 이유 검토 후 승인/거부
   
   사용: 성능이 중요하지만 유연성 필요한 경우
   ```

**권장:**

```yaml
성능 중요도별 전략:

매우 높음 (결제, 금융):
  → 자동 차단 (threshold 초과 = 배포 불가)

높음 (주문, 검색):
  → 자동 차단 + 예외 승인 프로세스
  → threshold 초과 시 CTO 승인 필요

중간 (일반 API):
  → Slack 경보만 (경보는 하지만 차단 안 함)

낮음 (사내 도구):
  → 모니터링만 (경보 없음)
```

**구현 예시:**

```yaml
# GitHub Actions에서
- name: Performance Check
  run: |
    if grep -q "threshold.*exceeded" perf-test.log; then
      # Critical API: 자동 실패
      if [ "$API_CRITICALITY" = "critical" ]; then
        exit 1
      fi
      
      # Important API: 경고
      if [ "$API_CRITICALITY" = "important" ]; then
        echo "⚠️ Performance degraded. Requires approval."
        exit 1
      fi
      
      # Standard API: 통지만
      echo "⚠️ Performance warning (non-blocking)"
    fi
```

</details>

### Q3: 성능 개선 후 baseline을 언제 업데이트해야 할까?

```
Before: p99 = 650ms
After 튜닝: p99 = 400ms (38% 개선!)

새로운 baseline:
옵션 1: 400ms (즉시 업데이트)
옵션 2: 450ms (보수적)
옵션 3: 600ms (여유있게)

어디로?
```

<details>
<summary>해설 보기</summary>

**전략에 따라 다릅니다:**

1. **옵션 1 (즉시 400ms로):**
   ```
   장점: 새로운 성능 수준 유지
   단점: 거짓 경보 위험 (측정 편차 고려 안 함)
   
   위험: 400ms는 이상적이지만
        실제는 변동성 때문에 410-430ms일 수 있음
        → threshold 400ms로 설정하면 바로 실패
   ```

2. **옵션 2 (보수적 450ms로):**
   ```
   장점: 측정 편차 고려 (약 5-10% 여유)
   단점: 약간의 회귀 허용
   
   추천 ✅
   → 400ms의 90% 수준 = 360ms 이상 필요
   ```

3. **옵션 3 (여유있게 600ms로):**
   ```
   장점: 거짓 경보 거의 없음
   단점: 회귀 탐지 기준이 너무 완화
   
   미권장
   → 원래 목표 650ms 근처로 돌아옴
   ```

**추천 방식:**

```
1단계: 성능 개선 완료 (400ms)
2단계: 1주간 모니터링 (평균값 확인)
3단계: 평균값의 90% 설정 (380ms 측정 시 342ms로 설정)

또는:

1단계: 성능 개선 완료 (400ms)
2단계: 신뢰할 수 있는 환경에서 5회 반복 측정
       → 평균: 405ms, 표준편차: 8ms
3단계: 평균 - 2×표준편차 = 389ms로 설정
```

**코드화:**

```javascript
// baseline.json 업데이트 규칙

// 규칙 1: 5% 이상 개선된 경우만 업데이트
if (newP99 < baseline.p99 * 0.95) {
  updateBaseline(newP99);
}

// 규칙 2: 업데이트할 때는 보수적으로 (95% 수준)
const conservativeBaseline = newP99 * 0.95;

// 규칙 3: 너무 빈번히 변경하지 않기
if (lastUpdateTime < 1주일 전) {
  // 업데이트 스킵
  return;
}
```

</details>

---

<div align="center">

**[⬅️ 이전: 성능 테스트 결과 보고서 — 비기술 직군을 위한 작성법](./03-performance-report.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실전 케이스 스터디 — DB 커넥션 고갈부터 튜닝까지 ➡️](./05-case-study.md)**

</div>
