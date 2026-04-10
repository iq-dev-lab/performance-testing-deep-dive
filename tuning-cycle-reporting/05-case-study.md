# 05. 실전 케이스 스터디 — DB 커넥션 고갈부터 튜닝까지

---

## 🎯 핵심 질문

"실제로 성능 문제를 진단하고 해결하는 과정은 어떻게 진행되는가?"

이 문서는 **시작부터 완료까지의 실전 스토리텔링**입니다.

---

## 🔍 상황: 전자상거래 주문 API의 성능 저하

### Timeline 시작: 2024년 3월 1일

**상황:**
```
오전 10시: 마케팅 팀이 대규모 프로모션 시작 (예상 트래픽 3배)
오후 2시: 고객 지원 이메일 폭주
         "주문 화면이 자꾸 튕겨요"
         "결제 버튼을 눌렀는데 반응이 없어요"
오후 3시: PagerDuty 알람 (응답 시간 이상 증가)
오후 3시 15분: 당신에게 전화 걸려옴
         "주문 API가 느려진 것 같은데 뭐가 문제인지 모르겠어!"
```

---

## 😱 증상: 무엇이 떠올랐는가?

### 관찰한 현상

```
1. 응답 시간 증가
   - 평소: 평균 250ms, p99 600ms
   - 지금: 평균 800ms, p99 3,200ms

2. 에러율 증가
   - 평소: 0.1%
   - 지금: 8.2%

3. 에러 메시지
   "java.sql.SQLException: Cannot get a connection, pool error Timeout waiting for idle object"
   → DB 커넥션 풀 고갈?

4. 서버 상태
   - CPU: 정상 (45%)
   - 메모리: 정상 (60%)
   - Disk I/O: 약간 높음 (70%)
```

---

## 🔬 1단계: 문제 재현 및 부하 테스트

### Step 1-1: 현재 상황 기록

```bash
# 오후 3시 20분 - 현재 상황 스냅샷

# 1. 실시간 요청 모니터링
curl -s http://api.example.com/health | jq .

# 결과:
# {
#   "status": "UNHEALTHY",
#   "db_connections": { "active": 30, "idle": 0, "max": 30 },
#   "response_time_p99": "3200ms",
#   "error_rate": "8.2%",
#   "cache_hit_rate": "72%"
# }

# 2. DB 커넥션 상태 확인
mysql -u root -p << 'EOF'
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads%';
EOF

# 결과:
# Threads_connected: 30 (Max: 30) ← 모두 사용 중!
# Threads_running: 28 (활성 중)
```

### Step 1-2: 통제된 환경에서 문제 재현

```bash
# 오후 3시 30분 - k6 부하 테스트로 현재 상황 재현

cat > baseline-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 300,        // 프로모션 중 예상 동시 사용자
  duration: '5m',
  thresholds: {
    'http_req_duration': ['p(99)<2000'],
  },
};

export default function () {
  const res = http.get('http://api.example.com/orders/list');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}
EOF

k6 run baseline-test.js --out json=baseline-current.json
```

**결과:**
```json
{
  "p50": 450,
  "p75": 980,
  "p95": 2100,
  "p99": 3200,
  "error_rate": 0.082,
  "error_type": "connection timeout"
}
```

✅ **문제 재현 성공 → 이제 원인 분석 가능**

---

## 🔍 2단계: USE 방법론으로 병목 특정

### Step 2-1: 리소스 별 사용률 분석

```bash
# CPU, Memory, I/O, Network 상태 확인
# USE = Utilization, Saturation, Errors

# 1. CPU (Utilization)
top -b -n 1 | grep "Cpu(s)"
# Cpu(s):  45.2% user, 12.3% sys, ...
# → CPU는 여유있음

# 2. Memory (Utilization)
free -h | grep "Mem:"
# Mem:    16G   9.6G   6.4G
# → 메모리도 60% 사용 (여유있음)

# 3. Disk I/O (Saturation)
iostat -x 1 2 | grep -A 1 "sda"
# iostat: await 35ms, util 70%
# → I/O 대기 시간 약간 높음

# 4. Network (Utilization)
vnstat -h
# eth0: In: 450 Mbps, Out: 120 Mbps
# → 대역폭 충분 (1Gbps 환경에서)

# 결론: CPU, Memory, Network는 정상
#      Disk I/O가 약간 높지만 병목 아님
#      → 다른 곳이 문제!
```

### Step 2-2: 애플리케이션 레벨 분석

```bash
# APM 도구 (New Relic 등)에서 분석

# 요청 흐름별 시간 분석:
# - 애플리케이션 처리: 50ms (정상)
# - DB 조회: 200ms (정상)
# - DB 커넥션 획득: 2,950ms (!!!!)
# → 커넥션 획득 대기가 전체 시간의 98.3%!

# DB 커넥션 풀 상태:
# - 최대: 30 connections
# - 활성: 30 connections
# - 대기 중: 280 requests (!!!)
# → 280명이 커넥션을 기다리는 중!
```

✅ **원인 파악: DB 커넥션 풀 고갈**

---

## 📊 3단계: 느린 쿼리 분석

### Step 3-1: 슬로우 쿼리 로그 확인

```bash
# MySQL 슬로우 쿼리 로그 확인
# (설정: slow_query_log, long_query_time=0.5)

# 상위 10개 느린 쿼리
mysql -u root -p << 'EOF'
SELECT query_time, lock_time, rows_examined, sql_text 
FROM mysql.slow_log 
ORDER BY query_time DESC 
LIMIT 10;
EOF

# 결과:
# Query Time: 0.8s, Rows: 50,000 (필터링 후 10개만)
# SELECT * FROM orders 
# WHERE user_id = ? 
# ORDER BY created_at DESC

# Query Time: 0.6s, Rows: 120,000 (필터링 후 5개만)
# SELECT * FROM order_items 
# WHERE order_id IN (...)
```

**분석:**
```
문제 쿼리들의 특징:
1. full table scan 실행 중 (인덱스 미사용)
2. 많은 행을 검색 후 필터링
3. 각 쿼리: 0.6-0.8초
4. 동시에 30개 쿼리 실행 중

결과:
- 1개 쿼리: 0.7초
- 30개 쿼리: 30 × 0.7초 = 21초!
- 다음 요청들: 큐에서 대기 (2,950ms)
```

### Step 3-2: 인덱스 분석

```bash
# 느린 쿼리의 실행 계획 확인
mysql -u root -p << 'EOF'
EXPLAIN SELECT * FROM orders 
WHERE user_id = 5 
ORDER BY created_at DESC;
EOF

# 결과:
# type: ALL (full table scan!)
# rows: 1,234,567 (백만 개 이상 스캔)
# key: NULL (인덱스 사용 안 함)

# 해결책: orders 테이블에 인덱스 추가
# CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);
```

✅ **추가 발견: 느린 쿼리가 커넥션 점유 시간 증가**

---

## 🔧 4단계: 단계적 해결 (하나씩, 순서대로)

### 해결책 1: 느린 쿼리에 인덱스 추가

**시간: 오후 4시**

```bash
# Step 1: 인덱스 생성 (온라인, 무중단)
mysql -u root -p << 'EOF'
ALTER TABLE orders ADD INDEX idx_user_created (user_id, created_at DESC), ALGORITHM=INPLACE, LOCK=NONE;
EOF

# 소요 시간: 2분 (백만 건 테이블)
# → 온라인 진행, 서비스 무중단

# Step 2: 인덱스 생성 확인
SHOW INDEX FROM orders;

# Step 3: 쿼리 성능 재확인
EXPLAIN SELECT * FROM orders WHERE user_id = 5 ORDER BY created_at DESC;
# type: range ✅
# rows: 5 (백만 → 5!)
```

**효과 측정:**
```bash
# 부하 테스트 재실행
k6 run baseline-test.js --out json=after-index.json

# 결과:
# p50: 450ms → 380ms (-15%)
# p95: 2100ms → 1400ms (-33%)
# p99: 3200ms → 1800ms (-44%)
# error_rate: 8.2% → 4.1% (절반)

# 개선: 있지만 여전히 부족 (목표: p99 < 800ms)
```

✅ **진전: 44% 개선, 하지만 충분하지 않음**

---

### 해결책 2: DB 커넥션 풀 크기 증가

**시간: 오후 4시 30분**

```bash
# 현재 설정 확인
grep -r "maximum-pool-size" application.yml
# spring.datasource.hikari.maximum-pool-size: 30

# 변경 전: 커넥션 풀 상태
# - 최대: 30
# - 활성: 25-30
# - 대기 요청: 50-100

# 문제 분석:
# - 프로모션: 예상 동시 사용자 300명
# - 각 요청: 평균 1개 쿼리 + 일부는 3-5개
# - 추천 풀 크기: 동시 사용자 / 2 = 300 / 2 = 150?
# 
# 실제 계산:
# pool_size = (core_count) * 2 + spare
# core_count = 4
# pool_size = 4 * 2 + 10 = 18? → 너무 작음
# 
# 더 정확한 계산:
# peak_threads = peak_vus = 300
# pool_size = peak_threads * (1 + overhead)
# pool_size = 300 * 1.2 = 360? → 너무 큼 (메모리 낭비)
#
# 현실적: 30 → 80 으로 증가 (2.7배)

# Step 1: 설정 변경
cat > application.yml << 'EOF'
spring:
  datasource:
    hikari:
      maximum-pool-size: 80  # 30 → 80
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
EOF

# Step 2: 애플리케이션 재시작
docker-compose restart app
sleep 5

# Step 3: 커넥션 풀 상태 확인
curl -s http://api.example.com/health | jq '.db_connections'
# { "active": 8, "idle": 72, "max": 80 }
```

**효과 측정:**
```bash
k6 run baseline-test.js --out json=after-pool.json

# 결과:
# p50: 380ms → 200ms (-47%)
# p95: 1400ms → 550ms (-61%)
# p99: 1800ms → 850ms (-53%)
# error_rate: 4.1% → 0.2% (-95%)

# 개선: 상당함! 하지만 아직 목표 미달성
```

✅ **진전: p99 850ms → 목표 800ms (거의 도달)**

---

### 해결책 3: 쿼리 타임아웃 조정

**시간: 오후 5시**

```bash
# 현재 타임아웃 설정 확인
grep -r "connection-timeout" application.yml
# spring.datasource.hikari.connection-timeout: 30000 (30초)

# 문제:
# - 커넥션 획득 대기: 최대 30초
# - 수동 테스트에서 타임아웃으로 인한 에러 관찰
# - 에러율 0.2%는 이 타임아웃에서 발생
#
# 현재 분석:
# - 대기 큐: 평균 200-300개
# - 커넥션 처리 시간: 0.5초
# - 예상 대기 시간: 300 * 0.5 / 80 = 1.875초
#
# → 타임아웃을 더 길게 설정하면?
# → 30초는 너무 길어 보임

# 문제: 타임아웃이 아니라 느린 쿼리!
# → 혹시 추가로 느린 쿼리가 있나?

mysql -u root -p << 'EOF'
SELECT query_time, sql_text FROM mysql.slow_log 
WHERE query_time > 0.5 
ORDER BY query_time DESC 
LIMIT 20;
EOF

# 결과:
# 0.7초: SELECT * FROM order_items WHERE order_id IN (...)
# 0.6초: SELECT * FROM users WHERE id = ? (age 기반 필터링)
# 0.5초: SELECT * FROM promotions WHERE active = 1

# → 다른 쿼리들도 느림! 인덱스 더 필요
```

**인덱스 추가:**
```bash
# 추가 느린 쿼리들에 인덱스 생성
mysql -u root -p << 'EOF'
ALTER TABLE order_items ADD INDEX idx_order (order_id), ALGORITHM=INPLACE;
ALTER TABLE users ADD INDEX idx_age (age), ALGORITHM=INPLACE;
ALTER TABLE promotions ADD INDEX idx_active (active), ALGORITHM=INPLACE;
EOF

# 재측정
k6 run baseline-test.js --out json=after-indexes.json

# 결과:
# p99: 850ms → 750ms (-12%)
# error_rate: 0.2% → 0.05%
```

✅ **진전: p99 750ms (목표 달성!)**

---

## 📊 5단계: 최종 결과 측정 및 검증

### 최종 부하 테스트 (10분, 신뢰도 높음)

```bash
# 오후 5시 30분 - 최종 성능 측정

cat > final-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 100 },
    { duration: '3m', target: 300 },  // 프로모션 피크
    { duration: '2m', target: 50 },
    { duration: '1m', target: 10 },
  ],
  thresholds: {
    'http_req_duration': [
      'p(50)<200',
      'p(95)<500',
      'p(99)<800',  // 목표
    ],
    'errors': ['rate<0.01'],  // 1% 이하
  },
};

export default function () {
  const res = http.get('http://api.example.com/orders/list');
  
  const success = res.status === 200;
  errorRate.add(!success);
  
  check(res, {
    'status is 200': () => success,
  });
  
  sleep(1);
}
EOF

k6 run final-test.js --out json=final-results.json

# 최종 결과:
# p50:   190ms ✅
# p95:   480ms ✅
# p99:   720ms ✅ (800ms 목표 달성!)
# error_rate: 0.08% ✅ (1% 이하)
# TPS: 155 req/s (이전: 98)
```

### Before/After 비교표

| 지표 | Before | After | 개선 | 개선율 |
|------|--------|-------|------|--------|
| **p50** | 450ms | 190ms | -260ms | -58% |
| **p75** | 980ms | 380ms | -600ms | -61% |
| **p95** | 2,100ms | 480ms | -1,620ms | -77% |
| **p99** | 3,200ms | 720ms | -2,480ms | -77% |
| **평균** | 800ms | 310ms | -490ms | -61% |
| **TPS** | 98 | 155 | +57 | +58% |
| **에러율** | 8.2% | 0.08% | -8.12% | -99% |
| **메모리** | 512MB | 580MB | +68MB | +13% |

**의미:**
- p99 3,200ms → 720ms: 77% 개선!
- 에러율 8.2% → 0.08%: 거의 완전히 제거!
- TPS 98 → 155: 58% 성능 향상 = 향후 2.6배 트래픽 수용 가능

---

## 💼 6단계: 비즈니스 임팩트 계산

```bash
# business-impact.js
cat > business-impact.js << 'EOF'
// 성능 개선의 비즈니스 가치

// 시나리오:
// - 월간 방문자: 1,000,000
// - 현재 전환율: 2%
// - 평균 주문액: 50,000원
// - 현재 피크타임 에러율: 8.2%

const metrics = {
  p99Before: 3200,
  p99After: 720,
  errorRateBefore: 0.082,
  errorRateAfter: 0.0008,
  monthlyVisitors: 1000000,
  conversionRate: 0.02,
  avgOrderValue: 50000,
};

// 1. 응답시간 개선 → 전환율 향상
const p99Improvement = (metrics.p99Before - metrics.p99After) / 100;  // 25.8
const conversionBoost = p99Improvement * 0.01;  // 25.8 × 1% = 0.258
const additionalConversions = metrics.monthlyVisitors * conversionBoost;
const additionalRevenue = additionalConversions * metrics.avgOrderValue;

console.log('📊 전환율 향상:');
console.log(`  p99: ${metrics.p99Before}ms → ${metrics.p99After}ms (${p99Improvement.toFixed(1)} × 100ms)`);
console.log(`  전환율: +${(conversionBoost * 100).toFixed(2)}%`);
console.log(`  추가 전환: ${additionalConversions.toFixed(0)}건/월`);
console.log(`  추가 매출: ${(additionalRevenue / 1000000).toFixed(0)}백만 원/월`);
console.log(`  연간 매출: ${(additionalRevenue * 12 / 1000000).toFixed(0)}백만 원\n`);

// 2. 에러율 감소 → 고객 유지
const errorReduction = metrics.errorRateBefore - metrics.errorRateAfter;
const lostCustomers = metrics.monthlyVisitors * errorReduction;
const ltv = 500000;
const retainedValue = lostCustomers * ltv * 0.9;  // 90%만 돌아옴

console.log('👥 고객 유지:');
console.log(`  에러율: ${(metrics.errorRateBefore * 100).toFixed(1)}% → ${(metrics.errorRateAfter * 100).toFixed(3)}%`);
console.log(`  보존 고객: ${lostCustomers.toFixed(0)}명/월`);
console.log(`  유지 가치: ${(retainedValue / 1000000).toFixed(0)}백만 원/월`);
console.log(`  연간: ${(retainedValue * 12 / 1000000).toFixed(0)}백만 원\n`);

// 3. 인프라 비용 절감
const tpsImprovement = 1.58;  // 58% 증가
const serverReduction = 3;  // 필요 서버 감소
const costPerServer = 2000000;
const monthlySavings = serverReduction * costPerServer;

console.log('💰 비용 절감:');
console.log(`  필요 서버 감소: ${serverReduction}대`);
console.log(`  월간 절감: ${(monthlySavings / 1000000).toFixed(0)}백만 원`);
console.log(`  연간 절감: ${(monthlySavings * 12 / 1000000).toFixed(0)}백만 원\n`);

// 총합
const totalMonthly = (additionalRevenue + retainedValue + monthlySavings) / 1000000;
console.log('🎯 총 비즈니스 가치:');
console.log(`  월간: ${totalMonthly.toFixed(0)}백만 원`);
console.log(`  연간: ${(totalMonthly * 12).toFixed(0)}백만 원`);

// 투자 대비 효과
const devCost = 15000000;  // 팀이 2일 소요 (약 1,500만 원)
const roi = (additionalRevenue + retainedValue) * 12 / devCost;
console.log(`\n📈 ROI: ${roi.toFixed(0)}배`);
console.log(`  투자 회수 기간: ${(devCost / ((additionalRevenue + retainedValue) / 12)).toFixed(1)}개월`);
EOF

node business-impact.js

# 출력:
# 📊 전환율 향상:
#   p99: 3200ms → 720ms (25.8 × 100ms)
#   전환율: +25.80%
#   추가 전환: 258,000건/월
#   추가 매출: 12,900백만 원/월
#   연간 매출: 154,800백만 원

# 👥 고객 유지:
#   에러율: 8.2% → 0.1%
#   보존 고객: 81,000명/월
#   유지 가치: 40,500백만 원/월
#   연간: 486,000백만 원

# 💰 비용 절감:
#   필요 서버 감소: 3대
#   월간 절감: 600백만 원
#   연간 절감: 7,200백만 원

# 🎯 총 비즈니스 가치:
#   월간: 53,400백만 원
#   연간: 640,800백만 원

# 📈 ROI: 3,100배
#   투자 회수 기간: 0.03개월 (약 1일)
```

🚀 **연간 640억 원의 비즈니스 가치 창출!**

---

## 📝 7단계: 튜닝 일지 작성

```markdown
# 주문 API 성능 튜닝 - 최종 보고서

**날짜:** 2024년 3월 1일
**담당자:** 개발팀
**상태:** 완료 ✅

## 문제 상황

### 초기 증상
- 응답시간: p99 3,200ms (매우 느림)
- 에러율: 8.2% (매우 높음)
- 원인: DB 커넥션 풀 고갈

### 비즈니스 영향
- 고객 이탈 증가
- 전환율 저하
- 시스템 신뢰도 하락

## 해결 과정

### Experiment #1: 느린 쿼리에 인덱스 추가
- **변경:** orders 테이블에 idx_user_created 인덱스 추가
- **효과:** p99 3200ms → 1800ms (-44%)
- **부작용:** 없음
- **결론:** 효과 있음, 다음 단계로 진행

### Experiment #2: DB 커넥션 풀 크기 증가
- **변경:** 30 → 80 (2.7배)
- **효과:** p99 1800ms → 850ms (-53%)
- **부작용:** 메모리 +68MB (크지 않음)
- **결론:** 효과 있음, 목표 근처에 도달

### Experiment #3: 추가 인덱스
- **변경:** order_items, users, promotions 테이블에 인덱스 추가
- **효과:** p99 850ms → 720ms (-12%)
- **부작용:** 없음
- **결론:** 성공, 최종 목표 달성

## 최종 결과

| 지표 | Before | After | 개선율 |
|------|--------|-------|--------|
| p99 | 3,200ms | 720ms | -77% |
| 에러율 | 8.2% | 0.08% | -99% |
| TPS | 98 | 155 | +58% |

## 비즈니스 가치

- **추가 매출:** 1,548억 원/년
- **고객 보존:** 4,860억 원/년
- **비용 절감:** 72억 원/년
- **총 가치:** 6,408억 원/년
- **ROI:** 3,100배

## 권장사항

1. **모니터링 강화**
   - p99 > 1000ms 시 알림 설정
   - 슬로우 쿼리 자동 감시

2. **캐싱 검토**
   - 자주 조회되는 데이터 캐싱
   - 프로모션 기간에 캐시 TTL 조정

3. **다른 API 적용**
   - 결제 API
   - 사용자 프로필 API
   - 재고 조회 API
```

---

## 🎓 학습한 교훈

### 기술적 교훈

1. **USE 방법론의 가치**
   - CPU/Memory/I/O를 체계적으로 확인
   - "뭔가 느리다"에서 "DB 커넥션"으로 범위 좁혀짐

2. **인덱스의 영향**
   - 쿼리 시간 0.7초 → 0.02초 (35배 단축!)
   - 가장 효과적인 튜닝

3. **연쇄 효과**
   - 커넥션 풀 증가 → 더 많은 쿼리 병렬 실행
   - → 더 많은 인덱스 필요 발견
   - → 단계적 해결의 중요성

### 조직적 교훈

1. **빠른 대응의 가치**
   - 소요 시간: 총 3시간
   - 자동화 없었으면: 2-3주 (손실 2-3억 원)

2. **정량화의 힘**
   - "느리다" → "p99 3200ms"로 변환
   - 비즈니스 가치: 6,400억 원/년

3. **단계적 접근의 중요성**
   - 한 번에 3개 변경 → "뭐가 효과 있었나?" 불명확
   - 하나씩 → 각 단계의 효과 측정 가능

---

## 📌 핵심 정리

### 타임라인 요약

```
오후 3:00 - 문제 발생 (p99 3200ms, 에러율 8.2%)
오후 3:30 - 문제 재현 (k6 부하 테스트)
오후 4:00 - 원인 파악 (DB 커넥션 + 느린 쿼리)
오후 4:00-4:30 - 인덱스 추가 (44% 개선)
오후 4:30-5:00 - 커넥션 풀 증가 (53% 추가 개선)
오후 5:00-5:30 - 추가 인덱스 (12% 추가 개선)
오후 5:30-6:00 - 검증 및 모니터링
오후 6:00 - 완료 (p99 720ms, 에러율 0.08%)

총 소요 시간: 3시간
```

### 성공의 조건

1. ✅ **빠른 부하 테스트로 문제 재현**
   - 수동 테스트 대신 자동화된 k6 사용
   - 실제 트래픽과 동일하게 재현 가능

2. ✅ **체계적인 원인 분석 (USE)**
   - CPU/Memory/I/O 각각 확인
   - 범위를 좁혀나감

3. ✅ **하나씩, 순차적으로 해결**
   - 각 변경의 효과 측정 가능
   - 부작용 조기 발견

4. ✅ **정량적 기준 설정**
   - "느렸다"가 아닌 "p99 720ms"로 표현
   - 성공/실패 명확함

5. ✅ **비즈니스 언어로 번역**
   - 기술 수치가 아닌 "연 6,400억 원 가치"로 전달
   - 경영진이 이해하고 지원

---

<div align="center">

**[⬅️ 이전: 성능 회귀 방지 — CI Pipeline 자동화](./04-performance-regression-prevention.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — 분산 부하 테스트 ➡️](../distributed-performance-testing/01-distributed-load-testing.md)**

</div>
