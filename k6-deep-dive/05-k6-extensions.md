# 05. k6 확장 — InfluxDB + Grafana 대시보드 연동

---

## 🎯 핵심 질문

- k6의 표준 출력은 충분한가, 왜 InfluxDB에 데이터를 보내야 하는가?
- `--out influxdb` 플래그의 동작 방식과 InfluxDB 2.x 설정은 어떻게 하는가?
- Grafana에서 k6 공식 대시보드를 임포트하고 커스터마이징하는 방법은?
- `new Counter`, `new Gauge`, `new Trend`, `new Rate` 커스텀 메트릭은 각각 언제 사용하는가?
- k6 Cloud의 분산 실행과 온프레미스 설치의 차이점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**k6의 표준 출력은 테스트 완료 후에만 결과를 보여줍니다.** 하지만 성능 문제는 테스트 **진행 중에** 발생합니다.

예를 들어:
- 0-2분: 정상 (p95 = 150ms)
- 2-3분: 메모리 누수 시작 (p95 = 500ms)
- 3-4분: DB 연결 풀 고갈 (p95 = 2000ms)
- 4-5분: 서버 복구 (p95 = 300ms)

표준 출력은 전체 평균값만 보여주므로, **"어느 시점에 문제가 발생했는가"** 를 알 수 없습니다.

InfluxDB + Grafana를 연동하면:
- **실시간 모니터링**: 테스트 진행 중 응답시간 추세 시각화
- **정밀한 분석**: 초 단위로 문제 발생 시점 파악
- **경향성 비교**: 이전 테스트와 비교하여 성능 변화 추적
- **자동화**: 특정 조건에서 알림 발생

또한 커스텀 메트릭을 사용하면, **비즈니스 지표**를 추적할 수 있습니다:
- "구매 완료율": 계속 낮아지는가?
- "장바구니 추가 지연": 시간에 따라 증가하는가?
- "로그인 실패율": 특정 시간에 집중되는가?

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. stdout 결과만 믿고 성능 문제를 놓치는 경우

```bash
# ❌ 테스트 완료 후 stdout만 확인
$ k6 run test.js

running (5m00s), 0/100 VUs, 10000 complete and 0 interrupted iterations

     http_req_duration: avg=250ms  p(95)=300ms  p(99)=500ms
     http_req_failed: 0%

✓ Test passed

# 문제:
# - 5분 동안 어떤 변화가 있었는지 모름
# - 만약 1분~3분 사이에 응답시간이 2000ms까지 올라갔다가 내려왔다면?
# - 평균 250ms에 묻혀서 문제를 발견하지 못함
```

### 2. InfluxDB 설정 없이 JSON 출력으로 저장하는 경우

```bash
# ❌ JSON으로 모든 데이터 저장 (파일이 너무 커짐)
$ k6 run --out json=metrics.json test.js
# 결과: metrics.json이 1GB 이상

# 분석:
# - JSON 파일을 수동으로 파싱해야 함
# - 실시간 모니터링 불가능
# - 시각화가 어려움
# - 이전 테스트와 비교 불가능
```

### 3. 기본 메트릭만 사용하는 경우

```javascript
// ❌ 기술적 메트릭만 측정 (비즈니스 가치 불명확)
export default function() {
  const res = http.post('http://api.example.com/order', {
    items: [1, 2, 3],
  });

  // http_req_duration, http_req_failed 자동 수집
  // 하지만 "주문 완료율", "결제 실패율" 같은 비즈니스 지표 없음
  
  sleep(1);
}

// 문제:
// - "성능이 좋다" vs "사용자 만족도가 높다"는 다른 개념
// - 응답시간이 100ms이어도 50% 주문이 실패하면 서비스 가치 0
```

### 4. Grafana 대시보드를 처음부터 만드는 경우

```
❌ 수작업으로 대시보드 구성 (시간 낭비)
├─ 메트릭 선택
├─ 그래프 생성
├─ 축 설정
├─ 색상 조정
├─ 알림 규칙 설정
└─ 각 조직마다 반복... (비효율)

✓ 권장: k6 공식 대시보드 템플릿 임포트 (5분)
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. InfluxDB 2.x 설정 및 k6 연동

**Docker로 InfluxDB 설치:**
```bash
# docker-compose.yml
version: '3'
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - '8086:8086'
    environment:
      INFLUXDB_DB: k6
      INFLUXDB_ADMIN_USER: admin
      INFLUXDB_ADMIN_PASSWORD: password123
      INFLUXDB_HTTP_AUTH_ENABLED: "true"
    volumes:
      - influxdb_data:/var/lib/influxdb2
  
  grafana:
    image: grafana/grafana:10.0
    ports:
      - '3000:3000'
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  influxdb_data:
  grafana_data:
```

**시작:**
```bash
docker-compose up -d

# InfluxDB 설정 접근
# http://localhost:8086 (user: admin, pass: password123)
```

**InfluxDB 2.x API 토큰 생성:**
```bash
# Docker 컨테이너 진입
docker exec -it <influxdb-container> influx

# 토큰 생성
> influx auth create --org myorg --description "k6 token" --read-bucket 'k6' --write-bucket 'k6'

# 결과: abc123... (이 토큰을 나중에 사용)
```

### 2. k6 스크립트에서 InfluxDB로 데이터 전송

**test-with-influxdb.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Counter, Gauge, Trend, Rate } from 'k6/metrics';

// 커스텀 메트릭 정의
const ordersCompleted = new Counter('orders_completed');
const cartSize = new Gauge('cart_size');
const checkoutDuration = new Trend('checkout_duration');
const paymentSuccessRate = new Rate('payment_success');

export const options = {
  scenarios: {
    ramping: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  
  thresholds: {
    'http_req_duration': ['p(95)<500'],
    'http_req_failed': ['rate<0.01'],
    'payment_success': ['rate>=0.95'],
  },
};

export default function() {
  const baseUrl = 'http://api.example.com';

  // 1. 상품 목록 조회
  const productsRes = http.get(baseUrl + '/products');
  check(productsRes, {
    'products loaded': (r) => r.status === 200,
  });

  sleep(0.5);

  // 2. 장바구니에 상품 추가
  const addCartRes = http.post(baseUrl + '/cart/add', {
    productId: Math.floor(Math.random() * 100),
    quantity: Math.floor(Math.random() * 5) + 1,
  });

  const cartData = addCartRes.json();
  const currentCartSize = cartData.items ? cartData.items.length : 0;
  
  // 커스텀 메트릭: 장바구니 크기
  cartSize.add(currentCartSize);

  check(addCartRes, {
    'item added to cart': (r) => r.status === 200,
  });

  sleep(1);

  // 3. 결제 프로세스
  const startCheckout = Date.now();
  
  const checkoutRes = http.post(baseUrl + '/checkout', {
    cartId: cartData.cartId,
    shippingAddress: 'Seoul, Korea',
  });

  const checkoutTime = Date.now() - startCheckout;
  checkoutDuration.add(checkoutTime);

  // 4. 결제
  const paymentRes = http.post(baseUrl + '/payment', {
    checkoutId: checkoutRes.json('checkoutId'),
    paymentMethod: 'credit_card',
    amount: cartData.total,
  });

  const paymentSuccess = paymentRes.status === 200;
  paymentSuccessRate.add(paymentSuccess);

  if (paymentSuccess) {
    ordersCompleted.add(1);
    check(paymentRes, {
      'payment successful': (r) => r.status === 200,
      'order confirmed': (r) => r.json('orderId') !== undefined,
    });
  } else {
    check(paymentRes, {
      'payment failed (expected)': (r) => r.status !== 200,
    });
  }

  sleep(2);
}

export function teardown(data) {
  console.log(`Total orders completed: ${ordersCompleted.value}`);
}
```

**k6 실행 (InfluxDB로 데이터 전송):**
```bash
# InfluxDB에 데이터 전송
k6 run \
  --out influxdb=http://localhost:8086/api/v1/write?db=k6 \
  test-with-influxdb.js

# InfluxDB 2.x 사용 시:
k6 run \
  --out influxdb \
  -e INFLUXDB_ADDR=http://localhost:8086 \
  -e INFLUXDB_ORG=myorg \
  -e INFLUXDB_BUCKET=k6 \
  -e INFLUXDB_TOKEN=abc123... \
  test-with-influxdb.js
```

### 3. Grafana에 k6 공식 대시보드 임포트

**웹 브라우저에서:**
1. Grafana 접속: http://localhost:3000 (admin/admin)
2. 데이터 소스 추가: InfluxDB 선택, URL: http://influxdb:8086
3. 대시보드 임포트: ID `2587` (k6 공식 대시보드)
4. 결과: 자동으로 k6 메트릭 대시보드 생성

**또는 CLI로:**
```bash
# 대시보드 JSON 다운로드
curl -s https://raw.githubusercontent.com/grafana-labs/k6-grafana-dashboard/main/k6-dashboard.json > k6-dashboard.json

# Grafana API로 임포트
curl -X POST http://localhost:3000/api/dashboards/db \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <API_TOKEN>' \
  -d @k6-dashboard.json
```

### 4. 커스텀 메트릭 상세 이해

```javascript
import {
  Counter,   // 계속 증가하는 값 (주문 수, 총 요청 수)
  Gauge,     // 현재 값 (VU 수, 장바구니 크기)
  Trend,     // 분포 측정 (응답시간, 처리 시간)
  Rate,      // 비율 (성공률, 에러율)
} from 'k6/metrics';

// 1. Counter: 계속 증가만 함
const failedLogins = new Counter('failed_logins');
export default function() {
  const res = http.post('http://api.example.com/login', {...});
  if (res.status !== 200) {
    failedLogins.add(1);  // 계속 증가
  }
}

// 2. Gauge: 현재 값을 갱신
const activeSessions = new Gauge('active_sessions');
export default function() {
  const res = http.get('http://api.example.com/sessions/count');
  activeSessions.set(res.json('count'));  // 현재 값 설정
}

// 3. Trend: 시계열 데이터 (통계 계산)
const pageLoadTime = new Trend('page_load_time');
export default function() {
  const start = Date.now();
  http.get('http://api.example.com/page');
  pageLoadTime.add(Date.now() - start);
  // InfluxDB에서 자동으로 평균, p95, p99 계산
}

// 4. Rate: true/false 비율 추적
const loginSuccess = new Rate('login_success');
export default function() {
  const res = http.post('http://api.example.com/login', {...});
  loginSuccess.add(res.status === 200);
  // InfluxDB에서 자동으로 성공률 계산
}
```

### 5. InfluxDB에서 고급 쿼리

**Grafana의 Query Editor에서:**
```flux
// 모든 k6 메트릭 조회
from(bucket: "k6")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "http_req_duration")
  |> mean()

// 특정 엔드포인트의 p95 응답시간
from(bucket: "k6")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "http_req_duration")
  |> filter(fn: (r) => r.name == "/api/products")
  |> quantile(q: 0.95)

// 시간별 주문 완료 수
from(bucket: "k6")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "orders_completed")
  |> increase()
  |> aggregateWindow(every: 1m, fn: sum)
```

---

## 🔬 내부 동작 원리

### InfluxDB로의 데이터 전송 흐름

```
k6 스크립트 실행
    ↓
[메트릭 수집] (1초마다)
├─ http_req_duration (각 요청의 응답시간)
├─ http_req_failed (성공/실패)
├─ vus (현재 VU 수)
├─ orders_completed (커스텀 Counter)
├─ cart_size (커스텀 Gauge)
└─ payment_success (커스텀 Rate)
    ↓
[InfluxDB Line Protocol로 변환]
└─ "http_req_duration,name=/api/products value=150.5 1234567890"
    ↓
[InfluxDB에 HTTP POST]
    ↓
[InfluxDB 저장]
├─ 시계열 데이터베이스에 인덱싱
├─ 시간별로 쿼리 가능
└─ 자동 집계 및 통계 계산
    ↓
[Grafana에서 시각화]
├─ 실시간 그래프 (rolling)
├─ 통계 계산 (평균, 백분위수)
└─ 알림 설정 가능
```

### k6 Cloud와 온프레미스의 차이

```
k6 Cloud:
┌─────────────────────────────────────┐
│ k6 스크립트                          │
└─────┬───────────────────────────────┘
      │
      ↓
┌─────────────────────────────────────┐
│ k6 Cloud Platform (로그인 필요)      │
├─ 분산 실행 (여러 리전)              │
├─ 자동 대시보드                      │
├─ 결과 저장 (7년)                    │
├─ 팀 협업                            │
└─────────────────────────────────────┘

온프레미스:
┌─────────────────────────────────────┐
│ k6 스크립트                          │
└─────┬───────────────────────────────┘
      │
      ↓
┌─────────────────────────────────────┐
│ 로컬/클라우드 서버 (--out influxdb) │
└─────┬───────────────────────────────┘
      │
      ↓
┌─────────────────────────────────────┐
│ InfluxDB (직접 관리)                │
└─────┬───────────────────────────────┘
      │
      ↓
┌─────────────────────────────────────┐
│ Grafana (직접 관리)                 │
├─ 완전 커스터마이징                   │
├─ 완전 제어 (데이터 소유)            │
├─ 자체 운영 (책임)                    │
└─────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: InfluxDB + Grafana 로컬 설정

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    ports:
      - "8086:8086"
    environment:
      INFLUXDB_DB: k6
      INFLUXDB_ADMIN_USER: admin
      INFLUXDB_ADMIN_PASSWORD: k6password
      INFLUXDB_HTTP_AUTH_ENABLED: "true"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - k6_network

  grafana:
    image: grafana/grafana:10.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_INSTALL_PLUGINS: grafana-clock-panel
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb
    networks:
      - k6_network

volumes:
  influxdb_data:
  grafana_data:

networks:
  k6_network:
    driver: bridge
```

**시작 및 설정:**
```bash
# 시작
docker-compose up -d

# InfluxDB 초기 설정 (웹 UI)
# http://localhost:8086
# - Organization: myorg
# - Bucket: k6
# - API Token 생성 및 복사

# Grafana 설정 (웹 UI)
# http://localhost:3000 (admin/admin)
# 1. InfluxDB 데이터 소스 추가
#    - URL: http://influxdb:8086
#    - Organization: myorg
#    - Bucket: k6
#    - Token: (위에서 생성한 토큰)
# 2. k6 대시보드 임포트 (ID: 2587)
```

### 실험 2: 커스텀 메트릭으로 비즈니스 추적

**test-business-metrics.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Counter, Gauge, Trend, Rate } from 'k6/metrics';

// 비즈니스 메트릭
const productsViewed = new Counter('products_viewed');
const cartAbandoned = new Counter('cart_abandoned');
const ordersPlaced = new Counter('orders_placed');
const revenueGenerated = new Gauge('revenue_generated');
const avgOrderValue = new Trend('order_value');
const checkoutSuccessRate = new Rate('checkout_success');

export const options = {
  scenarios: {
    ecommerce: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 30 },
        { duration: '5m', target: 30 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration': ['p(95)<500'],
    'checkout_success': ['rate>=0.80'],
  },
};

export default function() {
  const baseUrl = 'http://api.example.com';

  // 1. 상품 조회
  http.get(baseUrl + '/products');
  productsViewed.add(1);
  sleep(1);

  // 2. 장바구니 추가
  const itemPrice = Math.floor(Math.random() * 100) + 10;
  const addCartRes = http.post(baseUrl + '/cart', {
    productId: Math.floor(Math.random() * 1000),
    price: itemPrice,
  });

  if (addCartRes.status !== 200) {
    cartAbandoned.add(1);
    return; // 장바구니 추가 실패 시 중단
  }

  sleep(2);

  // 3. 결제 진행
  const checkoutRes = http.post(baseUrl + '/checkout', {
    cartId: addCartRes.json('cartId'),
  });

  if (checkoutRes.status === 200) {
    ordersPlaced.add(1);
    const orderValue = checkoutRes.json('total');
    avgOrderValue.add(orderValue);
    revenueGenerated.set(revenueGenerated.value + orderValue);
    checkoutSuccessRate.add(1);
    
    check(checkoutRes, {
      'order placed': (r) => r.status === 200,
    });
  } else {
    checkoutSuccessRate.add(0);
  }

  sleep(1);
}
```

**실행:**
```bash
# InfluxDB 2.x 토큰 설정 후 실행
export INFLUXDB_ADDR=http://localhost:8086
export INFLUXDB_ORG=myorg
export INFLUXDB_BUCKET=k6
export INFLUXDB_TOKEN=your_token_here

k6 run --out influxdb test-business-metrics.js

# Grafana에서 확인:
# - products_viewed: 증가 추세
# - orders_placed: 최종 주문 수
# - revenue_generated: 총 매출
# - checkout_success: 성공률 (80% 이상)
```

### 실험 3: 다중 테스트 비교

**테스트 1 (최적화 전):**
```bash
k6 run --tag env=pre-optimization test.js
# 결과 저장 (InfluxDB)
```

**최적화 적용**

**테스트 2 (최적화 후):**
```bash
k6 run --tag env=post-optimization test.js
# 결과 저장 (InfluxDB)
```

**Grafana에서 비교:**
```flux
// 두 버전의 p95 응답시간 비교
from(bucket: "k6")
  |> range(start: -2h)
  |> filter(fn: (r) => r._measurement == "http_req_duration")
  |> filter(fn: (r) => r.env == "pre-optimization" or r.env == "post-optimization")
  |> quantile(q: 0.95)
  |> group(columns: ["env"])
```

---

## 📊 성능 비교

| 항목 | 표준 출력 | JSON | InfluxDB | k6 Cloud |
|------|---------|------|---------|----------|
| **실시간 모니터링** | ✗ | ✗ | ✓ 우수 | ✓ 우수 |
| **시간별 추세** | ✗ | 수동 파싱 | ✓ 자동 | ✓ 자동 |
| **시각화** | ✗ | ✗ | ✓ Grafana | ✓ 웹 UI |
| **데이터 저장** | ✗ (화면만) | ✓ 로컬 | ✓ DB | ✓ 클라우드 |
| **테스트 비교** | ✗ | 수동 | ✓ 쿼리 | ✓ 자동 |
| **분산 실행** | ✗ | ✗ | ✗ | ✓ 있음 |
| **비용** | 무료 | 무료 | 무료 (자관) | 유료 |
| **복잡도** | 간단 | 간단 | 중간 | 간단 |

### 실제 설정 시간

```
표준 출력:
├─ k6 설치: 5분
├─ 스크립트 작성: 20분
└─ 실행: 즉시 ✓ (총 25분)

JSON:
├─ k6 설치: 5분
├─ 스크립트 작성: 20분
├─ 분석 스크립트 작성: 30분
└─ 실행 및 분석: 10분 (총 65분)

InfluxDB + Grafana:
├─ Docker 설치: 5분
├─ docker-compose 작성: 5분
├─ 컨테이너 시작: 2분
├─ InfluxDB 설정: 10분
├─ Grafana 설정: 15분
├─ 대시보드 임포트: 5분
├─ 스크립트 수정: 15분
└─ 실행: 즉시 (총 57분)

k6 Cloud:
├─ 가입: 5분
├─ 설정: 5분
├─ 스크립트 수정: 10분
└─ 실행: 즉시 (총 20분) ← 가장 빠름!
```

---

## ⚖️ 트레이드오프

| 방식 | 장점 | 단점 | 선택 시기 |
|------|------|------|----------|
| **표준 출력** | 설정 불필요 | 실시간 불가 | 빠른 테스트 |
| **JSON** | 데이터 저장 | 분석 복잡함 | 아카이빙 |
| **InfluxDB** | 완전 제어 | 운영 필요 | 장기 프로젝트 |
| **k6 Cloud** | 최고 편의 | 구독료 | 팀 협업 |

---

## 📌 핵심 정리

- **InfluxDB**: 시계열 데이터베이스, k6 메트릭을 시간별로 저장 및 분석
- **Grafana**: 대시보드, InfluxDB 데이터를 시각화 (k6 공식 템플릿 활용)
- **커스텀 메트릭**:
  - Counter: 계속 증가 (주문 수, 에러 수)
  - Gauge: 현재 값 (VU 수, 메모리 사용)
  - Trend: 분포 (응답시간 p95, p99)
  - Rate: 비율 (성공률, 실패율)
- **실시간 모니터링**: stdout은 불가능, InfluxDB + Grafana 필수
- **k6 Cloud**: 가장 편하지만 구독료 발생

---

## 🤔 생각해볼 문제

**Q1**: Gauge와 Counter의 차이가 뭔가?

<details>
<summary>해설 보기</summary>

**Counter**: 계속 증가만 함 (감소 불가)
```javascript
const failedRequests = new Counter('failed_requests');

// 시간에 따른 값:
// 1초: 0
// 2초: 5 (5개 에러)
// 3초: 8 (3개 더 추가)
// 4초: 12 (4개 더 추가)
// 그래프: 계단 형태 (올라만 감)
```

**Gauge**: 현재 값 (증가/감소 가능)
```javascript
const activeUsers = new Gauge('active_users');

// 시간에 따른 값:
// 1초: 100
// 2초: 150 (증가)
// 3초: 80 (감소)
// 4초: 120 (증가)
// 그래프: 오르락내리락 형태
```

**선택 기준:**
- Counter: "지금까지 몇 개 발생했는가?" (누적)
- Gauge: "지금 몇 개인가?" (현재)

</details>

---

**Q2**: k6 Cloud는 왜 구독료를 받을까?

<details>
<summary>해설 보기</summary>

**k6 Cloud의 가치:**

1. **분산 실행**: 여러 리전에서 동시에 테스트
   - k6 자체: 로컬/단일 머신만 가능
   - k6 Cloud: 미국, 유럽, 아시아 등 동시 실행

2. **인프라 관리**: Grafana, InfluxDB 직접 운영 불필요
   - 온프레미스: Docker, 백업, 보안 등 자체 관리
   - k6 Cloud: 모두 관리됨

3. **팀 협업**: 대시보드 공유, 결과 저장, 알림
   - 온프레미스: 직접 설정해야 함
   - k6 Cloud: 모두 통합

4. **장기 저장**: 모든 테스트 결과 자동 저장 (7년)
   - 온프레미스: DB 공간 자체 관리

```
예시 가격:
- k6 Cloud: $100/월 ~ (분산 실행 포함)
- 온프레미스 자체 구축: $0/월 (하지만 인력비, 유지보수 비용)

중소 팀: k6 Cloud가 더 저렴 (인력비 절감)
대규모 조직: 온프레미스가 더 저렴 (대량 테스트)
```

</details>

---

**Q3**: Grafana 쿼리에서 `quantile(q: 0.95)`는 정확히 뭔가?

<details>
<summary>해설 보기</summary>

**백분위수(percentile)를 계산하는 Flux 함수**입니다.

```flux
// q: 0.95 = 95번째 백분위수 = p95
quantile(q: 0.95)  // 상위 5% (상한값)

// 다른 예시:
quantile(q: 0.50)  // p50 (중위값)
quantile(q: 0.99)  // p99 (상위 1%)
quantile(q: 0.999) // p99.9 (상위 0.1%)

// 실제 Grafana 쿼리:
from(bucket: "k6")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "http_req_duration")
  |> quantile(q: 0.95)  // 최근 1시간의 p95 계산

// 결과: 단일 값 (예: 450ms)
```

**K6의 기본 출력과 비교:**
```
k6 표준 출력 (테스트 완료 시):
http_req_duration: p(95)=450ms

Grafana Flux 쿼리:
quantile(q: 0.95) = 450ms (동일)

차이: 실시간으로 매초 계산 가능 (추세 추적)
```

</details>

---

<div align="center">

**[⬅️ 이전: 결과 분석 — 히스토그램과 백분위수 해석](./04-result-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실전 시나리오 — 전자상거래 주문 플로우 ➡️](./06-real-world-scenarios.md)**

</div>
