# 06. 실전 시나리오 — 전자상거래 주문 플로우

---

## 🎯 핵심 질문

- 상품 조회부터 결제까지의 전체 주문 플로우를 k6로 어떻게 시뮬레이션하는가?
- JWT 토큰 기반 인증을 포함한 멀티 단계 API 테스트는 어떻게 구성하는가?
- `group()`으로 시나리오의 각 단계를 구분하고 분석하는 방법은?
- WebSocket을 통한 실시간 알림/채팅의 부하 테스트는 어떻게 하는가?
- CSV 파일이나 SharedArray를 사용하여 테스트 데이터를 동적으로 파라미터화하는 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**실제 사용자는 단일 API를 호출하지 않습니다.** 대신:
1. 로그인 (인증)
2. 상품 검색 (조회)
3. 상품 상세 확인 (조회)
4. 장바구니 추가 (쓰기)
5. 결제 정보 입력 (쓰기)
6. 결제 처리 (쓰기)
7. 주문 확인 (조회)

**각 단계마다 다른 성능 특성**이 있습니다:
- 로그인: CPU 부하 높음 (암호화)
- 상품 검색: DB 부하 높음 (전문검색)
- 결제: 외부 API 호출 (높은 레이턴시)

**전체 흐름을 테스트해야만**:
- 병목점의 정확한 위치 파악 (어느 단계에서 느려지는가?)
- 상태 전이 오류 감지 (장바구니 없이 결제할 수 없는가?)
- 실제 사용자 행동 시뮬레이션 (동시성 테스트)

---

## 😱 흔한 실수 (Before — 잘못된 접근)

### 1. 각 API를 독립적으로 테스트하는 실수

```javascript
// ❌ API별 독립 테스트 (현실성 부족)
export default function() {
  // 단순히 /products API만 테스트
  http.get('http://api.example.com/products');
  sleep(1);
}

// 문제:
// - 실제로는 로그인 → 검색 → 상세 → 결제 순서
// - API는 빠르지만, 전체 플로우는 느릴 수 있음
// - 결제 시스템의 부하를 전혀 측정하지 못함
// - 토큰 만료, 세션 오류 같은 문제 발견 불가능
```

### 2. 하드코딩된 테스트 데이터 사용

```javascript
// ❌ 모든 VU가 동일한 데이터 사용
export default function() {
  const userId = 'user123';  // 모든 VU가 같은 사용자
  const productId = 999;     // 모든 VU가 같은 상품
  
  http.post('http://api.example.com/order', {
    userId: userId,
    productId: productId,
  });
}

// 문제:
// - 모든 VU가 같은 상품을 주문하므로 DB 캐시 효율 100%
// - 실제 다양한 상품 선택 패턴 미반영
// - 데이터 일관성 체크 불가능 (stock 감소 등)
```

### 3. Group을 사용하지 않아 단계별 성능을 못 파악하는 경우

```javascript
// ❌ 전체 응답시간만 측정
export default function() {
  http.get('http://api.example.com/products');      // 50ms
  http.get('http://api.example.com/details');       // 100ms
  http.post('http://api.example.com/cart');         // 30ms
  http.post('http://api.example.com/checkout');     // 200ms
  
  // 전체: 380ms가 출력되지만,
  // "어느 단계가 느린가"를 알 수 없음
}

// 결과:
// http_req_duration: avg=380ms (막연함)
// 최적화할 지점: 불명확
```

### 4. 토큰 갱신 로직 없이 장시간 테스트하는 경우

```javascript
// ❌ 토큰 만료 후 테스트 실패
export function setup() {
  const res = http.post('http://api.example.com/login', {...});
  return { token: res.json('token') };  // 30분 만료
}

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 100,
      duration: '1h',  // 1시간 테스트
    },
  },
};

export default function(data) {
  // 30분 후 토큰 만료 → 401 에러 급증
  const res = http.get('http://api.example.com/data', {
    headers: { Authorization: `Bearer ${data.token}` },
  });
}

// 결과:
// 처음 30분: 성공
// 이후 30분: 모든 요청 401 에러 (테스트 무효)
```

---

## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)

### 1. 완전한 주문 플로우 (그룹별 단계 분리)

**test-ecommerce-flow.js:**
```javascript
import http from 'k6/http';
import { sleep, check, group } from 'k6';
import { Counter, Trend, Rate } from 'k6/metrics';

// 커스텀 메트릭
const ordersCompleted = new Counter('orders_completed');
const orderTotal = new Trend('order_total');
const paymentSuccessRate = new Rate('payment_success');

export const options = {
  scenarios: {
    checkout_flow: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },   // 워밍업
        { duration: '5m', target: 100 },  // 정상 부하
        { duration: '3m', target: 100 },  // 유지
        { duration: '2m', target: 0 },    // 감소
      ],
      gracefulRampDown: '30s',
    },
  },

  thresholds: {
    'http_req_duration{group:login}': ['p(95)<300'],
    'http_req_duration{group:search}': ['p(95)<200'],
    'http_req_duration{group:details}': ['p(95)<250'],
    'http_req_duration{group:checkout}': ['p(95)<800'],
    'payment_success': ['rate>=0.95'],
  },
};

// Setup: 테스트 환경 준비
export function setup() {
  const baseUrl = 'http://api.example.com';

  // 어드민 계정으로 로그인하여 테스트용 상품 확인
  const adminRes = http.post(baseUrl + '/auth/login', {
    email: 'admin@example.com',
    password: 'admin123',
  });

  check(adminRes, {
    'admin login': (r) => r.status === 200,
  });

  return {
    baseUrl: baseUrl,
    testProducts: [1, 2, 3, 4, 5], // 테스트할 상품 ID들
  };
}

export default function(data) {
  const baseUrl = data.baseUrl;
  const testProducts = data.testProducts;

  // ========== 단계 1: 로그인 ==========
  let token;
  group('Login', function() {
    const loginRes = http.post(baseUrl + '/auth/login', {
      email: 'user' + __VU + '@example.com',
      password: 'password123',
    });

    check(loginRes, {
      'login successful': (r) => r.status === 200,
      'has token': (r) => r.json('data.token') !== undefined,
    });

    token = loginRes.json('data.token');
  });

  const headers = {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
  };

  sleep(0.5);

  // ========== 단계 2: 상품 검색 ==========
  let searchResults;
  group('Search Products', function() {
    const query = ['laptop', 'phone', 'tablet', 'monitor'][
      Math.floor(Math.random() * 4)
    ];

    const searchRes = http.get(baseUrl + '/products/search?q=' + query, {
      headers: headers,
    });

    check(searchRes, {
      'search successful': (r) => r.status === 200,
      'has results': (r) => r.json('data.length') > 0,
    });

    searchResults = searchRes.json('data');
  });

  sleep(1);

  // ========== 단계 3: 상품 상세 조회 ==========
  let selectedProduct;
  group('View Product Details', function() {
    // 검색 결과 중 첫 번째 상품 선택
    const productId = searchResults[0] ? searchResults[0].id : 1;

    const detailRes = http.get(
      baseUrl + '/products/' + productId,
      { headers: headers }
    );

    check(detailRes, {
      'details loaded': (r) => r.status === 200,
      'has price': (r) => r.json('data.price') !== undefined,
      'in stock': (r) => r.json('data.stock') > 0,
    });

    selectedProduct = detailRes.json('data');
  });

  sleep(1);

  // ========== 단계 4: 장바구니 추가 ==========
  let cartId;
  group('Add to Cart', function() {
    const quantity = Math.floor(Math.random() * 3) + 1;

    const cartRes = http.post(
      baseUrl + '/cart/add',
      {
        productId: selectedProduct.id,
        quantity: quantity,
        price: selectedProduct.price,
      },
      { headers: headers }
    );

    check(cartRes, {
      'added to cart': (r) => r.status === 200,
      'cart updated': (r) => r.json('data.itemCount') > 0,
    });

    cartId = cartRes.json('data.cartId');
  });

  sleep(0.5);

  // ========== 단계 5: 결제 정보 확인 ==========
  let checkoutData;
  group('Checkout', function() {
    const checkoutRes = http.post(
      baseUrl + '/checkout/prepare',
      {
        cartId: cartId,
        shippingAddress: '123 Main St, Seoul',
        billingAddress: '123 Main St, Seoul',
      },
      { headers: headers }
    );

    check(checkoutRes, {
      'checkout prepared': (r) => r.status === 200,
      'has total': (r) => r.json('data.total') > 0,
    });

    checkoutData = checkoutRes.json('data');
  });

  sleep(1);

  // ========== 단계 6: 결제 처리 ==========
  let paymentSuccess = false;
  group('Payment', function() {
    const paymentRes = http.post(
      baseUrl + '/payment/process',
      {
        checkoutId: checkoutData.checkoutId,
        paymentMethod: 'credit_card',
        cardToken: 'tok_test_' + Math.random(),
      },
      { headers: headers }
    );

    paymentSuccess = paymentRes.status === 200;

    check(paymentRes, {
      'payment successful': (r) => r.status === 200 || r.status === 202,
      'has order id': (r) =>
        r.json('data.orderId') !== undefined ||
        r.json('data.orderRef') !== undefined,
    });

    if (paymentSuccess) {
      paymentSuccessRate.add(1);
      const total = checkoutData.total;
      orderTotal.add(total);
      ordersCompleted.add(1);
    } else {
      paymentSuccessRate.add(0);
    }
  });

  sleep(2);

  // ========== 단계 7: 주문 확인 ==========
  if (paymentSuccess) {
    group('Order Confirmation', function() {
      const confirmRes = http.get(baseUrl + '/orders/latest', {
        headers: headers,
      });

      check(confirmRes, {
        'order confirmed': (r) => r.status === 200,
        'order details present': (r) =>
          r.json('data.orderId') !== undefined,
      });
    });
  }

  sleep(1);
}

export function teardown(data) {
  console.log(`Total orders completed: ${ordersCompleted.value}`);
}
```

**실행 및 결과 해석:**
```bash
k6 run test-ecommerce-flow.js

# 결과 예시:
# ✓ http_req_duration{group:login}......: p(95)=250ms ✓
# ✓ http_req_duration{group:search}.....: p(95)=180ms ✓
# ✓ http_req_duration{group:details}....: p(95)=220ms ✓
# ✓ http_req_duration{group:checkout}...: p(95)=650ms ✓
# ✓ payment_success...................: 96.5%      ✓

# 분석:
# - 각 단계별 성능 명확함
# - checkout이 가장 느림 (650ms) → 최적화 대상
# - payment 성공률 96.5% (목표 95% 달성)
```

### 2. 테스트 데이터 파라미터화 (CSV 파일)

**test-data.csv:**
```csv
email,password,firstName,lastName,cardNumber
john@example.com,pass123,John,Doe,4111111111111111
jane@example.com,pass456,Jane,Smith,4222222222222222
bob@example.com,pass789,Bob,Johnson,4333333333333333
alice@example.com,pass012,Alice,Williams,4444444444444444
charlie@example.com,pass345,Charlie,Brown,4555555555555555
```

**test-parameterized.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';
import { SharedArray } from 'k6/data';

// CSV 데이터 로드 (모든 VU가 공유)
const testData = new SharedArray('test users', function() {
  return open('./test-data.csv')
    .split('\n')
    .slice(1) // 헤더 제외
    .filter(line => line.trim() !== '')
    .map(line => {
      const fields = line.split(',');
      return {
        email: fields[0],
        password: fields[1],
        firstName: fields[2],
        lastName: fields[3],
        cardNumber: fields[4],
      };
    });
});

export const options = {
  scenarios: {
    test: {
      executor: 'per-vu-iterations',
      vus: 5,
      iterations: 1, // 각 VU는 1번 반복 (전체 5개 사용자)
    },
  },
};

export default function() {
  // __VU로 사용자 선택 (VU 1 → user[0], VU 2 → user[1], ...)
  const user = testData[(__VU - 1) % testData.length];

  const baseUrl = 'http://api.example.com';

  // 로그인
  const loginRes = http.post(baseUrl + '/auth/login', {
    email: user.email,
    password: user.password,
  });

  check(loginRes, {
    'login successful': (r) => r.status === 200,
  });

  const token = loginRes.json('data.token');
  const headers = { Authorization: `Bearer ${token}` };

  sleep(1);

  // 프로필 업데이트
  const updateRes = http.put(
    baseUrl + '/user/profile',
    {
      firstName: user.firstName,
      lastName: user.lastName,
    },
    { headers: headers }
  );

  check(updateRes, {
    'profile updated': (r) => r.status === 200,
  });

  sleep(1);

  // 결제 정보 등록
  const paymentRes = http.post(
    baseUrl + '/payment/method/add',
    {
      cardNumber: user.cardNumber,
      cardHolder: user.firstName + ' ' + user.lastName,
      expiryMonth: '12',
      expiryYear: '2025',
    },
    { headers: headers }
  );

  check(paymentRes, {
    'payment method added': (r) => r.status === 200,
  });

  sleep(1);
}
```

**실행:**
```bash
k6 run test-parameterized.js

# 결과:
# - 5명의 실제 사용자로 테스트
# - 각 사용자의 고유 데이터 사용
# - 현실적인 다중 사용자 시나리오
```

### 3. WebSocket 실시간 통신 테스트

**test-websocket.js:**
```javascript
import ws from 'k6/ws';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const wsConnections = new Counter('ws_connections');
const messageLatency = new Trend('message_latency');

export const options = {
  scenarios: {
    websocket_chat: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },   // 20개 채팅방
        { duration: '3m', target: 20 },   // 유지
        { duration: '1m', target: 0 },    // 감소
      ],
    },
  },
  thresholds: {
    'message_latency': ['p(95)<500'],  // 메시지 지연 < 500ms
  },
};

export default function() {
  const baseUrl = 'ws://api.example.com';
  const roomId = 'room_' + Math.floor(Math.random() * 100);
  const url = baseUrl + '/chat?roomId=' + roomId;

  const res = ws.connect(url, function(socket) {
    wsConnections.add(1);

    // 웹소켓 연결 확인
    socket.on('open', function open() {
      console.log('연결됨');

      // 메시지 전송
      const start = Date.now();
      socket.send(
        JSON.stringify({
          type: 'message',
          userId: __VU,
          text: 'Hello from VU ' + __VU,
          timestamp: Date.now(),
        })
      );
    });

    // 메시지 수신
    socket.on('message', function message(data) {
      const latency = Date.now() - JSON.parse(data).timestamp;
      messageLatency.add(latency);

      check(latency, {
        'message received within 500ms': (l) => l < 500,
      });
    });

    // 에러 처리
    socket.on('error', function(e) {
      console.error('웹소켓 에러:', e);
    });

    // 주기적으로 메시지 전송
    let messageCount = 0;
    const sendInterval = setInterval(function() {
      if (messageCount++ < 10) {
        socket.send(
          JSON.stringify({
            type: 'message',
            userId: __VU,
            text: 'Message ' + messageCount,
            timestamp: Date.now(),
          })
        );
      } else {
        clearInterval(sendInterval);
        socket.close();
      }
    }, 1000); // 1초마다 메시지

    // 30초 후 연결 종료
    socket.setTimeout(function() {
      socket.close();
    }, 30000);
  });

  check(res, {
    'websocket status is 101 (upgrade)': (r) => r.status === 101,
  });

  sleep(5);
}
```

### 4. 토큰 갱신 로직 포함 (장시간 테스트)

**test-token-refresh.js:**
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  scenarios: {
    test: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2h', // 2시간 테스트
    },
  },
};

export default function() {
  // 현재 토큰이 유효한지 확인 (매 반복마다)
  const token = getCurrentToken();

  // 토큰이 만료 근처면 갱신
  if (isTokenExpiringSoon(token)) {
    const refreshRes = http.post('http://api.example.com/auth/refresh', {
      refreshToken: getStoredRefreshToken(),
    });

    if (refreshRes.status === 200) {
      const newToken = refreshRes.json('data.token');
      const newRefreshToken = refreshRes.json('data.refreshToken');
      saveTokens(newToken, newRefreshToken);
    }
  }

  // API 호출 (최신 토큰 사용)
  const apiRes = http.get('http://api.example.com/user/orders', {
    headers: {
      Authorization: `Bearer ${getCurrentToken()}`,
    },
  });

  check(apiRes, {
    'api call successful': (r) => r.status === 200,
    'not unauthorized': (r) => r.status !== 401,
  });

  sleep(5);
}

// 토큰 관리 함수들
let storedToken = null;
let storedRefreshToken = null;
let tokenExpireTime = 0;

function getCurrentToken() {
  if (!storedToken) {
    const loginRes = http.post('http://api.example.com/auth/login', {
      email: 'user' + __VU + '@example.com',
      password: 'password123',
    });
    storedToken = loginRes.json('data.token');
    storedRefreshToken = loginRes.json('data.refreshToken');
    tokenExpireTime = Date.now() + 30 * 60 * 1000; // 30분 유효
  }
  return storedToken;
}

function isTokenExpiringSoon(token) {
  return Date.now() > tokenExpireTime - 5 * 60 * 1000; // 5분 여유
}

function saveTokens(token, refreshToken) {
  storedToken = token;
  storedRefreshToken = refreshToken;
  tokenExpireTime = Date.now() + 30 * 60 * 1000;
}

function getStoredRefreshToken() {
  return storedRefreshToken;
}
```

---

## 📊 성능 비교

| 항목 | 단일 API | 전체 플로우 | 파라미터화 | WebSocket |
|------|---------|-----------|----------|----------|
| **현실성** | 낮음 | 높음 | 매우 높음 | 높음 |
| **병목점 파악** | 불가 | 우수 | 우수 | 우수 |
| **복잡도** | 낮음 | 중간 | 중간 | 높음 |
| **테스트 시간** | 빠름 | 중간 | 중간 | 느림 |
| **데이터 다양성** | 없음 | 제한 | 무제한 | 제한 |

### 실제 테스트 비교 (동일 API, 다른 접근법)

```
단일 API 테스트:
GET /api/products
평균 응답시간: 50ms ✓ 빠르다!

전체 플로우 테스트:
로그인 → 검색 → 상세 → 장바구니 → 결제
평균 응답시간: 250ms (느리다!)
원인: 결제 API의 느린 응답 (800ms p95)

결론: 단일 API 테스트만으로는 문제 발견 불가능
```

---

## ⚖️ 트레이드오프

| 방식 | 장점 | 단점 | 선택 시기 |
|------|------|------|----------|
| **단일 API** | 빠른 테스트 | 현실성 낮음 | 개발 단계 |
| **전체 플로우** | 현실적 | 복잡함 | 정식 테스트 |
| **파라미터화** | 다양한 데이터 | CSV 관리 필요 | 회귀 테스트 |
| **WebSocket** | 실시간 측정 | 복잡한 로직 | 채팅/실시간 앱 |

---

## 📌 핵심 정리

- **Group 사용**: 단계별 성능 측정 (login, search, checkout 등)
- **전체 플로우**: 실제 사용자 행동 시뮬레이션 필수
- **파라미터화**: CSV 또는 SharedArray로 다양한 테스트 데이터 주입
- **토큰 갱신**: 장시간 테스트는 토큰 만료 고려
- **WebSocket**: ws 모듈로 실시간 통신 테스트

---

## 🤔 생각해볼 문제

**Q1**: Group을 사용하면 왜 각 단계별 성능이 따로 나타나는가?

<details>
<summary>해설 보기</summary>

**Group은 메트릭을 태그별로 분류**합니다:

```javascript
group('Login', function() {
  http.get('http://api.example.com/login');
});

group('Checkout', function() {
  http.get('http://api.example.com/checkout');
});

// k6가 내부적으로:
// 1. Login 그룹의 모든 http_req_duration을 수집
// 2. tag: { group: "Login" }를 자동으로 추가
// 3. Checkout 그룹의 모든 http_req_duration을 수집
// 4. tag: { group: "Checkout" }를 자동으로 추가

// 결과:
// http_req_duration{group:Login}: avg=150ms  p(95)=180ms
// http_req_duration{group:Checkout}: avg=300ms  p(95)=450ms
```

**이것이 가능한 이유**: k6의 메트릭 시스템이 **태그를 자동으로 추적**하기 때문입니다.

</details>

---

**Q2**: CSV 데이터를 모든 VU가 공유해도 괜찮을까? 동시성 문제는 없나?

<details>
<summary>해설 보기</summary>

**SharedArray는 읽기 전용이므로 동시성 문제 없습니다.**

```javascript
const testData = new SharedArray('data', function() {
  return open('./data.csv').split('\n').map(...);
  // ↑ 1회만 실행 (메모리에 1번 로드)
});

// 모든 VU가 이 배열을 읽음 (쓰지 않음)
// VU 1: testData[0] 읽기 ✓
// VU 2: testData[1] 읽기 ✓
// VU 3: testData[2] 읽기 ✓
// ... (동시에 읽어도 안전)

// 반대: 동적 배열은 VU마다 복제됨
const dynamicData = open('./data.csv').split('\n').map(...);
// ↑ 100 VU × 1MB = 100MB 메모리 낭비

// SharedArray: 1MB (모든 VU 공유)
```

**규칙:**
- 수정하지 않는 데이터 → SharedArray (효율적)
- VU별로 수정해야 하는 데이터 → 일반 변수 (격리됨)

</details>

---

**Q3**: WebSocket 테스트에서 "메시지 지연"을 어떻게 측정하는가?

<details>
<summary>해설 보기</summary>

**메시지에 타임스탬프를 포함**하고, 서버의 응답 메시지에서 계산합니다:

```javascript
// 클라이언트 (k6):
const sendTime = Date.now();
socket.send(
  JSON.stringify({
    text: 'Hello',
    clientTime: sendTime,  // 클라이언트 시간
  })
);

// 서버:
// 받은 메시지를 모든 클라이언트에 브로드캐스트

// 클라이언트 (k6) - 수신:
socket.on('message', function(data) {
  const msg = JSON.parse(data);
  const latency = Date.now() - msg.clientTime;  // 왕복 지연 측정
  messageLatency.add(latency);
});

// 결과:
// messageLatency: avg=120ms  p(95)=180ms
// 의미: 메시지 왕복 지연이 평균 120ms
```

**정확한 측정을 위한 팁:**
- 클라이언트 시간: `Date.now()` (ms 단위)
- 네트워크 지연: 클라이언트 시간 - 클라이언트 시간 = 왕복
- 서버 처리 시간을 분리하려면 `serverTime`도 포함

</details>

---

<div align="center">

**[⬅️ 이전: k6 확장 — InfluxDB + Grafana 대시보드 연동](./05-k6-extensions.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — USE 방법론 ➡️](../bottleneck-identification/01-use-methodology.md)**

</div>
