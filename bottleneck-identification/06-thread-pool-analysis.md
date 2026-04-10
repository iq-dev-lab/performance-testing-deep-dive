# 06. 스레드 풀 분석 — Thread Dump로 BLOCKED 원인 특정

---

## 🎯 핵심 질문

스레드 풀이 고갈되었을 때, 정확히 뭐가 블로킹되어 있을까?
- BLOCKED: Lock 대기 (다른 스레드가 synchronized 또는 ReentrantLock 가지고 있음)
- WAITING: notify() 대기 (누군가 notify해줄 때까지 영구 대기)
- TIMED_WAITING: timeout이 있는 대기 (Thread.sleep, wait(timeout) 등)

**각 상태를 정확히 파악하지 못하면 엉뚱한 최적화를 하게 된다.**

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
상황: p95 응답시간 1500ms, 스레드 풀 200개 중 190개 사용 중

개발자 판단 1:
→ "스레드가 부족하다! 스레드 풀 200 → 300으로 증설"
→ 응답시간 여전히 1500ms (개선 안 됨)
→ 원인: BLOCKED 스레드가 많아서 추가 스레드도 의미 없음

개발자 판단 2:
→ "CPU 사용률이 높다! CPU 증설"
→ 응답시간 여전히 1500ms
→ 원인: CPU 사용률 높은 게 아니라 Lock 대기

올바른 진단:
→ jstack 으로 Thread Dump 분석
→ 스레드 상태 확인:
   - BLOCKED: 95개 (Lock 대기)
   - WAITING: 50개 (notify 대기)
   - RUNNABLE: 55개 (실제 실행 중)

원인:
OrderService.saveOrder() 메서드가 synchronized (전체 메서드)
→ 1개 스레드가 3초 점유 → 95개 스레드 대기

해결:
synchronized 제거 또는 ConcurrentHashMap 사용
→ 응답시간 1500ms → 150ms (90% 개선!)
```

---

## 😱 흔한 실수 (Before)

```java
// 잘못된 접근 1: synchronized를 과도하게 사용
public synchronized Order saveOrder(Order order) {
    // 전체 메서드가 Lock 보유
    // → 이 메서드 실행 중에는 다른 스레드 못 옴
    validateOrder(order);  // 100ms
    database.save(order);  // 500ms
    cache.invalidate();    // 50ms
    return order;
}

// 문제: 3개 스레드만 동시 실행 가능 (300 스레드 풀은 의미 없음)
// 3개 × 650ms = 초당 4.6개만 처리

// 잘못된 접근 2: Thread Dump를 모른다
// 스레드가 부족하다고 판단
// → 스레드 풀 증설 (비용만 증가)
// → 실제 원인은 synchronized (Lock 경합)

// 잘못된 접근 3: Deadlock을 방치한다
// Thread A: Lock 1 획득 → Lock 2 대기
// Thread B: Lock 2 획득 → Lock 1 대기
// → 영구 교착 상태 (Deadlock)
// → 스레드 2개 점유, 나머지는 대기

// 잘못된 접근 4: notify를 이해 못한다
synchronized (buffer) {
    while (buffer.isEmpty()) {
        buffer.wait();  // notify 대기
    }
    Object item = buffer.take();
}

// notify를 호출하는 곳이 없으면?
// → 스레드는 영구 대기 (응답 안 옴)
```

**실제 사례:**

```
상황: 신규 기능 배포 후 응답시간 3배 증가

초기 진단:
→ 개발자: "스레드 풀 크기 부족"
→ 스레드 풀 100 → 200으로 증설
→ 응답시간 여전히 높음

2차 진단 (정말 필요한 것):
$ jstack $(jps | grep App | awk '{print $1}') > /tmp/jstack.txt
$ grep "tid=" /tmp/jstack.txt | wc -l
# 스레드 수: 200개

$ grep "BLOCKED on" /tmp/jstack.txt | wc -l
# BLOCKED 스레드: 150개 (75%!)

원인: 새 UserService 클래스가 synchronized (전체 메서드)
→ 동시에 1개 스레드만 접근 가능
→ 150개 스레드가 Lock 대기

해결: synchronized 제거, ConcurrentHashMap 사용
→ BLOCKED 스레드: 150 → 5개 (97% 감소)
→ 응답시간: 2000ms → 200ms (90% 개선!)
```

---

## ✨ 올바른 접근 (After)

### 1단계: Thread Dump 수집

**Thread Dump 수집 방법:**

```bash
# 방법 1: jstack 사용 (Java 8+, 권장)
$ jps  # 프로세스 ID 찾기
12345 org.springframework.boot.loader.JarLauncher

$ jstack 12345 > /tmp/jstack_$(date +%s).txt
# 또는 반복적으로 수집 (추이 파악)
$ for i in {1..5}; do
    jstack 12345 > /tmp/jstack_$i.txt
    sleep 10
  done

# 방법 2: JDK의 jcmd 사용
$ jcmd 12345 Thread.print > /tmp/jstack.txt

# 방법 3: Spring Boot Actuator
$ curl http://localhost:8080/actuator/threaddump > /tmp/jstack.json

# 방법 4: Kill -3 신호 (Unix)
$ kill -3 12345
# stdout/stderr에 thread dump 출력됨
```

### 2단계: Thread Dump 분석

**스레드 상태 확인:**

```bash
# 1. 스레드별 상태 정렬
$ grep "^\"" /tmp/jstack.txt | sort | uniq -c | sort -rn
#     150 "http-nio-8080-exec-12" - BLOCKED
#      50 "http-nio-8080-exec-45" - WAITING  
#      55 "http-nio-8080-exec-78" - RUNNABLE
#      30 "http-nio-8080-exec-99" - TIMED_WAITING

# 분석:
# - BLOCKED 150: Lock 대기 중
# - WAITING 50: notify 대기 중
# - RUNNABLE 55: 실제 실행 중
# - TIMED_WAITING 30: timeout 있는 대기

# 2. BLOCKED 스레드 확인
$ grep -A 3 "tid=.* BLOCKED" /tmp/jstack.txt | head -20
# "http-nio-8080-exec-12" #45 daemon prio=5 os_prio=0 tid=0x00007f8e9c001000
# nid=0x1234 waiting to lock <0x00000000d1234567> (a java.lang.Object)
# at com.example.OrderService.saveOrder(OrderService.java:25)
# - waiting to lock <0x00000000d1234567>
# - locked <0x00000000d1234568>

# 분석:
# - 스레드가 Lock 0xd1234567을 기다리는 중
# - 이미 Lock 0xd1234568를 가지고 있음 (Deadlock 가능성?)
# - 발생 위치: OrderService.java:25 줄

# 3. Lock을 가지고 있는 스레드 찾기
$ grep -B 5 "locked <0x00000000d1234567>" /tmp/jstack.txt | \
  grep "^\"" | head -1
# "http-nio-8080-exec-99" - RUNNABLE
# → 이 스레드가 Lock을 가지고 있음

# 4. Deadlock 자동 탐지
$ grep "Found one Java-level deadlock:" /tmp/jstack.txt
# 있으면 Deadlock이 있다는 뜻
```

**실제 예시:**

```
Thread Dump 출력:

"http-nio-8080-exec-1" #24 daemon prio=5 os_prio=0 tid=0x00007f1234567800
  nid=0x1234 waiting to lock <0x00000000c1234567> (a com.example.OrderService)
  at com.example.OrderService.saveOrder(OrderService.java:42)
  at com.example.OrderController.createOrder(OrderController.java:20)
  - waiting to lock <0x00000000c1234567> (a com.example.OrderService)
  - locked <0x00000000c1234568> (a java.lang.Object)

"http-nio-8080-exec-2" #25 daemon prio=5 os_prio=0 tid=0x00007f1234567900
  nid=0x5678 runnable
  at com.example.OrderService.saveOrder(OrderService.java:50)
  at com.example.OrderController.createOrder(OrderController.java:20)
  - locked <0x00000000c1234567> (a com.example.OrderService)

분석:
- Thread 1: Lock 0xc1234567을 기다리는 중 (BLOCKED)
- Thread 2: Lock 0xc1234567을 가지고 있음 (RUNNABLE)
- 병목: OrderService.saveOrder() 메서드의 Lock 대기
- 해결: synchronized 제거 또는 Lock 범위 축소
```

### 3단계: 스레드 상태별 원인과 해결책

**BLOCKED (Lock 대기):**

```java
// 문제 코드
public synchronized void updateUser(User user) {  // ← Lock 획득
    // 이 영역에서만 Lock 필요한데 전체에 적용
    validateUser(user);      // 100ms
    database.save(user);     // 500ms
    cache.update(user);      // 100ms
    // 이 메서드 중 최대 550ms 동안 Lock 보유
}

// 해결책 1: synchronized 범위 축소
public void updateUser(User user) {
    validateUser(user);  // Lock 불필요
    database.save(user);  // Lock 불필요
    
    synchronized (this) {  // 최소한의 Lock
        cache.update(user);  // 이 부분만 Lock (100ms)
    }
}

// 해결책 2: ConcurrentHashMap 사용
private ConcurrentHashMap<Long, User> cache = new ConcurrentHashMap<>();

public void updateUser(User user) {
    validateUser(user);
    database.save(user);
    cache.put(user.getId(), user);  // Atomic operation (Lock 불필요)
}

// 해결책 3: ReentrantLock의 시간 제한
private Lock lock = new ReentrantLock();

public void updateUser(User user) {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {  // 1초 대기
        try {
            cache.update(user);
        } finally {
            lock.unlock();
        }
    } else {
        log.warn("Lock acquisition timeout");
        return;  // 또는 cached data 사용
    }
}
```

**WAITING (무조건 대기):**

```java
// 문제 코드: notify 안 함
synchronized (buffer) {
    while (buffer.isEmpty()) {
        buffer.wait();  // ← notify가 없으면 영구 대기
    }
    process(buffer.take());
}

// 해결책 1: notify 호출
public synchronized void add(Item item) {
    buffer.add(item);
    this.notify();  // 대기 중인 스레드 깨우기
}

// 해결책 2: BlockingQueue 사용 (권장)
BlockingQueue<Item> buffer = new LinkedBlockingQueue<>();

// Producer
public void add(Item item) {
    buffer.put(item);  // 자동으로 대기 스레드 깨움
}

// Consumer
public Item take() {
    return buffer.take();  // 자동으로 대기
}

// 해결책 3: CountDownLatch (1회용 동기화)
CountDownLatch latch = new CountDownLatch(1);

// 대기
public void wait() {
    latch.await();  // Signal 대기
}

// Signal
public void signal() {
    latch.countDown();  // 대기 스레드 깨우기
}
```

**TIMED_WAITING (timeout 있는 대기):**

```java
// 원인: Thread.sleep, wait(timeout) 등
Thread.sleep(1000);  // 1초 대기 (의도된 지연)

// 또는 외부 리소스 대기
socket.setSoTimeout(5000);  // 5초 timeout
inputStream.read();  // 5초 동안 대기

// 분석:
// - TIMED_WAITING은 정상 (timeout이 설정된 것)
// - 문제는 TIMED_WAITING 시간이 길면 응답시간 증가
// - 예: sleep 1초 × 100 스레드 = 초당 100개 요청만 처리

// 해결책: Timeout 시간 단축 또는 비동기 처리
// Before: sleep(5000)
// After: sleep(100) 또는 CompletableFuture로 비동기
```

### 4단계: 스레드 풀 크기 최적화

**스레드 풀 크기 결정:**

```yaml
# application.yml (Spring Boot + Tomcat)
server:
  tomcat:
    threads:
      max: 200  # 최대 스레드 수
      min-spare: 20  # 최소 대기 스레드 (워밍)
    accept-count: 100  # 큐 크기 (Connection 대기)
    connection-timeout: 20000  # 20초 connection 대기
    processor-cache: 200  # NIO 버퍼 풀 크기

# 권장값 계산:
# max = (동시 사용자 수 × 평균 요청 시간) / 1000 + 여유
# 예: 동시 100명, 평균 100ms
# max = (100 × 100) / 1000 + 20 = 30개

# 고려 사항:
# 1. CPU-Bound: cores * 2
# 2. I/O-Bound: cores * 4~8 (DB 대기 시간 때문에)
# 3. 외부 API: cores * 2~4 (API timeout)
# 4. 혼합: cores * 3~5 (보통 이 경우)
```

**동적 모니터링:**

```bash
#!/bin/bash
# thread-pool-monitor.sh

APP_PID=$(jps | grep App | awk '{print $1}')

while true; do
    echo "=== Thread Pool Status ($(date '+%H:%M:%S')) ==="
    
    # 1. 활성 스레드
    active=$(curl -s http://localhost:8080/actuator/metrics/executor.active | \
        jq '.measurements[0].value')
    echo "Active threads: $active"
    
    # 2. Queue 크기
    queue=$(curl -s http://localhost:8080/actuator/metrics/executor.queue | \
        jq '.measurements[0].value')
    echo "Queue size: $queue"
    
    # 3. BLOCKED 스레드
    blocked=$(jstack $APP_PID | grep "BLOCKED" | wc -l)
    echo "BLOCKED threads: $blocked"
    
    # 4. WAITING 스레드
    waiting=$(jstack $APP_PID | grep "WAITING" | wc -l)
    echo "WAITING threads: $waiting"
    
    # 5. 경고
    if [ $queue -gt 100 ]; then
        echo "⚠️  Queue overflow! Consider increasing pool size"
    fi
    
    if [ $blocked -gt 50 ]; then
        echo "⚠️  High BLOCKED threads! Check for Lock contention"
    fi
    
    sleep 10
done
```

---

## 🔬 내부 동작 원리

### Synchronized Lock의 메커니즘

```
시나리오: synchronized (order) 블록

Thread A:
├─ 진입 전: Lock 대기 큐에서 대기
├─ Lock 획득: Monitor (intrinsic lock) 획득
├─ 임계 영역(critical section) 실행
│  ├─ 코드 라인 1 (500ms)
│  ├─ 코드 라인 2 (200ms)
│  └─ 코드 라인 3 (100ms)
└─ Lock 해제: Monitor 해제

Thread B, C, D, ... (모두 BLOCKED 상태):
├─ 진입 시도: Lock 필요
├─ Lock 불가: Monitor 사용 중 (Thread A)
├─ 대기 큐에서 대기
└─ A의 Lock 해제 후 하나씩 진입 가능

결과:
- Thread A: 800ms 실행
- Thread B: 800ms 대기 후 실행
- Thread C: 1600ms 대기 후 실행
- Thread D: 2400ms 대기 후 실행

응답시간:
- A: 800ms
- B: 1600ms
- C: 2400ms
- D: 3200ms
평균: 2000ms (정상은 800ms)
```

### Deadlock의 발생 원인

```
Deadlock 조건 (모두 만족해야 발생):
1. Mutual Exclusion: Lock은 한 번에 1개 스레드만
2. Hold and Wait: 스레드가 Lock을 들고 다른 Lock 대기
3. No Preemption: Lock을 강제로 빼앗을 수 없음
4. Circular Wait: Lock 대기 순환

예시:

Thread A:
├─ Lock 1 획득 ✓
├─ Lock 2 대기 ✗

Thread B:
├─ Lock 2 획득 ✓
├─ Lock 1 대기 ✗

결과: 둘 다 영구 대기 (무한 교착)

Timeline:
T0: A locks Lock1, B locks Lock2
T1: A waits Lock2 (held by B)
T2: B waits Lock1 (held by A)
T3-T∞: Deadlock (아무것도 진행 안 됨)
```

### 스레드 풀의 포화 현상

```
스레드 풀 크기: 10개
동시 요청: 20개
각 요청 시간: 500ms

Timeline:
T0: 요청 20개 도착
T0-T10: 스레드 10개에 요청 배당, 10개 대기
T500: 처음 10개 완료, 다음 10개 시작
T1000: 마지막 10개 완료

결과:
- 처음 10개 응답시간: 500ms (정상)
- 다음 10개 응답시간: 1000ms (500ms 큐 대기)
- 평균: 750ms
- 스루풋: 20개 / 1초 = 20 RPS

스레드 풀 크기 20개로 증가:
T0-T20: 모든 요청 즉시 처리
T500: 모든 요청 완료

결과:
- 모든 응답시간: 500ms (정상)
- 스루풋: 20개 / 0.5초 = 40 RPS (2배!)
```

---

## 💻 실전 실험

**시나리오: p99 응답시간 1500ms, 스레드 풀 거의 모두 BLOCKED 상태**

```bash
#!/bin/bash
# thread-pool-diagnosis.sh

APP_PID=$(jps | grep App | awk '{print $1}')

echo "=== 스레드 풀 병목 진단 ==="
echo

# 1. 스레드 풀 상태
echo "1. 스레드 풀 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics | \
  jq '.names[] | select(contains("executor"))' | \
  while read metric; do
    value=$(curl -s "http://localhost:8080/actuator/metrics/$metric" | \
      jq '.measurements[0].value')
    echo "$metric: $value"
  done

# 2. Thread Dump 수집
echo
echo "2. Thread Dump 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
jstack $APP_PID > /tmp/jstack.txt

# 스레드 상태 통계
echo "스레드 상태:"
grep "^\"" /tmp/jstack.txt | awk -F'- ' '{print $2}' | sort | uniq -c

# 3. BLOCKED 스레드 상세 분석
echo
echo "3. BLOCKED 스레드 원인"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "BLOCKED 스레드 TOP 5:"
grep -B 2 "BLOCKED\|waiting to lock" /tmp/jstack.txt | \
  grep "at com.example" | \
  sort | uniq -c | sort -rn | head -5

# 4. Lock 분석
echo
echo "4. Lock 분석 (누가 Lock을 가지고 있는가?)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
# Lock ID 추출
lock_id=$(grep "waiting to lock" /tmp/jstack.txt | head -1 | \
  grep -o "0x[0-9a-f]*" | head -1)

if [ -n "$lock_id" ]; then
  echo "Lock ID: $lock_id"
  echo "Lock을 가지고 있는 스레드:"
  grep -B 5 "locked $lock_id" /tmp/jstack.txt | \
    grep "^\"" | head -1
fi

# 5. Deadlock 확인
echo
echo "5. Deadlock 확인"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
if grep -q "Found one Java-level deadlock" /tmp/jstack.txt; then
  echo "⚠️  DEADLOCK DETECTED!"
  grep -A 20 "Found one Java-level deadlock" /tmp/jstack.txt
else
  echo "✓ No deadlock detected"
fi

# 6. 성능 영향
echo
echo "6. 성능 지표"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "응답시간 분포:"
curl -s http://localhost:8080/actuator/metrics/http.server.requests | \
  jq '.measurements[]'

echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== 스레드 풀 병목 진단 ===

1. 스레드 풀 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
executor.active: 195
executor.queue: 150
executor.pool-size: 200

⚠️  CRITICAL: Queue 150 (매우 높음!)

2. Thread Dump 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
스레드 상태:
    150 BLOCKED
     40 WAITING
      5 TIMED_WAITING
      5 RUNNABLE

⚠️  75% BLOCKED (거의 대부분 Lock 대기)

3. BLOCKED 스레드 원인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BLOCKED 스레드 TOP 5:
    145 at com.example.OrderService.saveOrder
      3 at com.example.PaymentService.process
      2 at com.example.UserService.update

4. Lock 분석 (누가 Lock을 가지고 있는가?)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lock ID: 0x00000000d1234567
Lock을 가지고 있는 스레드:
"http-nio-8080-exec-99" - RUNNABLE
    at com.example.OrderService.saveOrder (OrderService.java:42)

5. Deadlock 확인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ No deadlock detected

6. 성능 지표
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
응답시간 분포:
{
  "statistic": "COUNT",
  "value": 10000
}
{
  "statistic": "TOTAL",
  "value": 15000000
}

평균: 15000000 / 10000 = 1500ms

=== 진단 결과 ===

🔍 주요 문제:
1. OrderService.saveOrder() 메서드: synchronized (전체 메서드)
   - Lock 점유 시간: ~500ms
   - BLOCKED 스레드: 145개 (75%)

2. Queue 크기: 150 (스레드 부족 신호)
   - 예상 응답시간: 500ms (쿼리) + 1500ms (대기) = 2000ms

원인 분석:
synchronized public Order saveOrder(Order order) {
    validateOrder(order);      // 100ms
    database.save(order);      // 300ms
    cache.invalidate();        // 100ms
}

전체 메서드가 Lock 보유 → 최대 500ms 동안 다른 스레드 접근 불가

해결책:
1. synchronized 범위 축소 (캐시 invalidate만 Lock)
2. ConcurrentHashMap 사용 (Lock 제거)
3. 스레드 풀 증설은 비효과적 (Lock이 문제이기 때문)
```

---

## 📊 성능 비교

| 상황 | 스레드 | BLOCKED | Queue | p99 응답시간 | 해결책 |
|------|--------|---------|--------|------------|--------|
| **synchronized 전체** | 200 | 150 | 100 | 1500ms | Lock 범위 축소 |
| **synchronized 축소** | 200 | 10 | 0 | 150ms | - |
| **ConcurrentHashMap** | 200 | 0 | 0 | 120ms | - |
| **스레드 풀 300** | 300 | 150 | 200 | 1800ms | 효과 없음 |

**실제 케이스:**

```
케이스 1: synchronized 메서드 (문제)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  public synchronized void updateUser(User user) {
      validate(user);       // 100ms
      saveDB(user);         // 300ms
      updateCache(user);    // 100ms
  }
  
  결과: Lock 시간 500ms, BLOCKED 145개, p99 1500ms

After: Lock 범위 축소
  public void updateUser(User user) {
      validate(user);  // Lock 불필요
      saveDB(user);    // Lock 불필요
      synchronized (this) {
          updateCache(user);  // 이 부분만 Lock (100ms)
      }
  }
  
  결과: Lock 시간 100ms, BLOCKED 10개, p99 150ms
  개선율: p99 90% 개선

케이스 2: WAITING 스레드
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  synchronized (queue) {
      while (queue.isEmpty()) {
          queue.wait();  // notify 없음
      }
      process(queue.poll());
  }
  
  결과: WAITING 50개 (영구 대기), p99 무한

After: BlockingQueue 사용
  BlockingQueue<Item> queue = new LinkedBlockingQueue<>();
  
  Producer: queue.put(item);  // 자동 notify
  Consumer: queue.take();     // 자동 대기 해제
  
  결과: WAITING 0개, p99 정상

케이스 3: Lock 문제가 아닌 경우
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  BLOCKED: 50개
  Queue: 0
  p99: 1500ms
  
  → Lock이 아니라 다른 병목 (DB, CPU, I/O)

확인:
- BLOCKED 스레드가 없는데 응답시간 높음?
  → Lock이 아니다
- 스레드 풀 여유 있는데 응답시간 높음?
  → Lock이 아니다
  
다른 병목 확인: CPU, Memory, I/O, 네트워크
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 선택 기준 |
|------|------|------|---------|
| **synchronized** | 간단함, 안전함 | 느림, Lock 경합 | 단순 데이터 (많지 않은 경우) |
| **ReentrantLock** | 유연함, tryLock 가능 | 복잡, 데드락 위험 | 고급 제어 필요 시 |
| **ConcurrentHashMap** | Lock-free, 빠름 | HashMap과 다른 동작 | 고동시성 (권장) |
| **스레드 풀 증설** | 즉시 효과 | 비용, 근본 해결 X | 정말 부족할 때만 |

---

## 📌 핵심 정리

1. **Thread Dump가 답이다**
   - 스레드 문제 = Thread Dump 분석
   - jstack으로 BLOCKED, WAITING 확인
   - Lock ID를 통해 원인 특정

2. **BLOCKED는 Lock 경합 신호**
   - BLOCKED > 50개이면 심각
   - synchronized 범위 축소
   - ConcurrentHashMap 등으로 Lock 제거

3. **WAITING은 notify 문제**
   - notify를 호출하는 곳이 있는가?
   - BlockingQueue로 대체 권장
   - CountDownLatch, Condition 등으로 해결

4. **스레드 풀 증설은 마지막 수단**
   - BLOCKED가 높으면 스레드 증설은 비효과적
   - 근본 원인(Lock, I/O)부터 해결
   - 마지막으로 필요하면 증설

5. **의사결정 플로우**
   ```
   응답시간 높음
   ├─ Thread Dump 수집
   ├─ BLOCKED > 50?
   │  ├─ Yes: Lock 문제 (synchronized, ReentrantLock)
   │  │  ├─ Lock 범위 축소
   │  │  └─ ConcurrentHashMap 사용
   │  └─ No: 다음 단계
   ├─ WAITING > 20?
   │  ├─ Yes: notify 문제
   │  │  └─ BlockingQueue, Condition 사용
   │  └─ No: 다음 단계
   └─ Queue > 50?
      ├─ Yes: 스레드 풀 부족
      │  └─ cores * 3~4로 설정
      └─ No: 다른 병목 (CPU, I/O, DB)
   ```

---

## 🤔 생각해볼 문제

**Q1: BLOCKED 150개 스레드를 가진 상황에서 스레드 풀을 300으로 늘리면 개선될까?**

<details>
<summary>해설 보기</summary>

**아니다. 오히려 악화될 수 있다.**

**이유:**

```
원인: synchronized 메서드가 Lock을 500ms 점유

스레드 풀 200:
├─ BLOCKED: 150개 (Lock 대기)
├─ RUNNABLE: 5개 (실제 실행)
└─ 처리율: 200개 동시 요청 중 5개만 처리

스레드 풀 300으로 증설:
├─ BLOCKED: 200+개 (Lock 대기가 더 많아짐!)
├─ RUNNABLE: 5개 (여전히 5개만 처리)
└─ 처리율: 변화 없음 (또는 더 악화)

이유:
- Lock을 가지고 있는 스레드는 여전히 1개
- 대기 스레드만 300개로 증가
- 더 많은 스레드가 Lock을 기다림
```

**해결책:**

1. **Lock 문제 해결**
   ```java
   // Before
   public synchronized void updateUser(User user) {
       // 500ms 소요
   }
   
   // After
   public void updateUser(User user) {
       // 400ms 동안 Lock 불필요
       synchronized (cache) {  // 100ms만 Lock
           cache.update(user);
       }
   }
   ```

2. **ConcurrentHashMap 사용**
   ```java
   // Lock 제거, BLOCKED 스레드 0개
   cache.put(user.getId(), user);  // Atomic operation
   ```

**결론:** BLOCKED 스레드 많으면 스레드 풀 증설은 비효과적. Lock 문제부터 해결해야 한다.

</details>

---

**Q2: Thread Dump에서 같은 Lock을 기다리는 스레드가 200개이고, 그 Lock을 가진 스레드가 없으면?**

<details>
<summary>해설 보기</summary>

**이것은 심각한 문제다. 원인을 찾아야 한다.**

**가능한 원인:**

1. **Lock을 가진 스레드가 이미 종료됨**
   ```
   Thread A가 Lock을 획득 후 예외로 중단
   → Lock이 해제되지 않음
   → 다른 스레드들 영구 대기
   
   해결: 항상 finally에서 Lock 해제
   lock.lock();
   try {
       // 작업
   } finally {
       lock.unlock();  // 필수!
   }
   ```

2. **Deadlock (Lock A ↔ Lock B 교착)**
   ```
   Thread A: Lock X → Lock Y 대기
   Thread B: Lock Y → Lock X 대기
   
   Lock X를 가진 스레드 = A (Lock Y 대기)
   Lock Y를 가진 스레드 = B (Lock X 대기)
   
   → 서로 Lock을 기다림 (교착)
   ```

3. **Lock이 released되었는데 Thread Dump 시점 차이**
   ```
   T0: Thread Dump 시작
   T1: Lock 소유자가 Lock 해제
   T2: Thread Dump 분석
   
   → Lock을 기다리는 스레드는 보이지만
   → 소유자는 이미 해제했을 수 있음
   
   해결: 여러 번 Thread Dump를 수집해서 비교
   ```

**진단 방법:**

```bash
# 1. Deadlock 확인
$ grep "Found one Java-level deadlock" /tmp/jstack.txt
# 있으면 Deadlock

# 2. Lock 소유자 확인
$ lock_id="0x00000000d1234567"
$ grep -B 5 "locked <$lock_id>" /tmp/jstack.txt
# 출력 없으면 Lock 소유자 없음

# 3. 여러 번 수집하여 추세 파악
$ for i in {1..5}; do
    jstack 12345 | grep "waiting to lock" | wc -l
    sleep 5
  done
  
# 감소 추세 = 정상 (Lock이 풀리는 중)
# 증가 추세 = 위험 (Lock이 영구 점유)
# 변화 없음 = 위험 (영구 대기)
```

**결론:** Lock 소유자가 없으면서 많은 스레드가 대기하는 것은 불가능해야 한다. 이 경우 Thread Dump 수집 시점의 차이이거나 이전에 예외로 Lock 해제 안 된 경우다.

</details>

---

**Q3: RUNNABLE 상태이지만 응답시간이 높은 경우는?**

<details>
<summary>해설 보기</summary>

**Lock 문제가 아니라 다른 병목이다.**

**확인 사항:**

1. **CPU 병목**
   ```
   RUNNABLE 스레드가 많은데 응답시간 높음
   → CPU 사용률 확인
   
   $ top
   # Cpu(s): 90%us, 5%sy, 0%id
   
   → CPU 사용률 높음 (CPU-Bound)
   → 알고리즘 최적화 필요
   ```

2. **I/O 병목**
   ```
   RUNNABLE이지만 실제로는 I/O 대기 중?
   → Thread state가 정확하지 않을 수 있음
   
   $ iostat -x 1 5
   # await > 50ms, avgqu-sz > 5
   
   → I/O 병목 (DB, 디스크)
   ```

3. **Context Switch 과다**
   ```
   RUNNABLE 스레드 200개 + 2 CPU core
   → Context switch 과다 (CPU 자체는 여유)
   
   $ vmstat 1 5
   # cs > 5000/sec
   
   해결: 스레드 풀 축소 (200 → 50)
   ```

4. **메모리/GC 문제**
   ```
   RUNNABLE이지만 GC 때문에 멈춤
   → Thread state에는 RUNNABLE로 표시되지만
   → 실제로는 GC pause 중일 수 있음
   
   확인: GC 로그, jstat -gc
   ```

**의사결정:**

```
RUNNABLE 많음 + 응답시간 높음

1단계: CPU 확인
  - CPU 사용률 높음? → CPU-Bound (알고리즘)
  - CPU 사용률 낮음? → I/O 대기 또는 Context Switch

2단계: I/O 확인
  - await > 20ms? → I/O 병목 (DB, 디스크)
  - await < 5ms? → Context Switch 문제

3단계: Context Switch 확인
  - cs > 2000/sec? → 스레드 풀 축소
  - cs < 1000/sec? → GC 또는 다른 문제
```

**결론:** RUNNABLE이 많아도 Lock 문제가 아니다. CPU, I/O, Context Switch 등 다른 병목을 찾아야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: 외부 API 호출 병목 — Timeout과 Circuit Breaker](./05-external-api-bottleneck.md)** | **[홈으로 🏠](../README.md)** | **[다음: 네트워크 병목 분석 — 응답 크기와 직렬화 비용 ➡️](./07-network-bottleneck.md)**

</div>
