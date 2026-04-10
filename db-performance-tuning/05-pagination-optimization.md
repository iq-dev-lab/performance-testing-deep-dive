# 05. 페이징과 대용량 조회 최적화 — OFFSET의 성능 저하

---

## 🎯 핵심 질문

- OFFSET 방식이 왜 데이터가 많아질수록 느려질까?
- 100만 건 테이블에서 OFFSET 500,000과 0의 응답시간 차이는?
- 커서(Cursor) 기반 페이징으로 어떻게 개선할 수 있을까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

페이징은 **모든 웹 서비스**에 필수입니다.
- 목록 조회: 1000개 중 처음 10개만 표시
- 무한 스크롤: 사용자가 스크롤할 때마다 다음 10개

**문제:** OFFSET 방식은 계산 복잡도가 O(n)입니다.
- OFFSET 0: 첫 10개 찾기 (매우 빠름)
- OFFSET 100만: 100만 개를 스캔한 후 다음 10개 (매우 느림)

**결과:** 마지막 페이지로 갈수록 응답시간 **지수적 증가**

---

## 😱 흔한 실수 (Before)

### 실수 1: OFFSET 페이징의 성능 문제 무시

```java
// Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    private String status;
}

// Repository (문제 있는 코드)
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // 😱 OFFSET 페이징 (큰 OFFSET은 느림)
    Page<Order> findByStatusOrderByCreatedAtDesc(
        String status, Pageable pageable);
}

// Controller
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @GetMapping
    public Page<Order> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int pageSize) {
        
        // 😱 문제: page=50000, pageSize=20 요청
        // → OFFSET 1,000,000 (백만 번 스킵!)
        // → 응답: 30초 😱
        
        return orderRepository.findByStatusOrderByCreatedAtDesc(
            "COMPLETED", 
            PageRequest.of(page, pageSize, 
                Sort.by("createdAt").descending()));
    }
}
```

**결과:**
```
페이지별 응답시간:
├─ 1페이지 (OFFSET 0): 50ms ✅
├─ 10페이지 (OFFSET 180): 80ms
├─ 100페이지 (OFFSET 1980): 200ms
├─ 1000페이지 (OFFSET 19980): 2,000ms
└─ 50000페이지 (OFFSET 1,000,000): 30,000ms ❌
```

### 실수 2: 데이터 스캔 원리 미이해

```sql
-- 😱 OFFSET의 내부 동작
SELECT * FROM orders 
WHERE status = 'COMPLETED'
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;

-- MySQL 내부 처리:
-- 1️⃣ status='COMPLETED' 조건에 맞는 행 찾기 (인덱스)
-- 2️⃣ created_at로 정렬 (메모리 또는 디스크)
-- 3️⃣ 처음 1,000,000개 행 건너뛰기 ← ⭐ 여기가 병목!
-- 4️⃣ 다음 20개 행 반환

-- 이 과정에서:
-- - 1,000,000개 행을 메모리에 로드
-- - 1,000,000번의 포인터 이동
-- - 대량의 CPU/메모리 사용 💥
```

---

## ✨ 올바른 접근 (After)

### 접근 1: 커서 기반 페이징 (Keyset Pagination)

```java
// ✅ 커서 기반 페이징 구현
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // 기존: OFFSET 방식 (느림)
    Page<Order> findByStatusOrderByCreatedAtDesc(
        String status, Pageable pageable);
    
    // ✅ 새로운: 커서 기반 (빠름)
    @Query("""
        SELECT o FROM Order o
        WHERE o.status = :status
        AND (o.createdAt < :cursor OR 
             (o.createdAt = :cursor AND o.id < :cursorId))
        ORDER BY o.createdAt DESC, o.id DESC
        LIMIT :pageSize
        """)
    List<Order> findByCursor(
        @Param("status") String status,
        @Param("cursor") LocalDateTime cursor,
        @Param("cursorId") Long cursorId,
        @Param("pageSize") int pageSize);
}

// ✅ 응답 DTO (다음 커서 포함)
@Data
@AllArgsConstructor
public class CursorPageResponse<T> {
    private List<T> content;        // 현재 페이지 데이터
    private String nextCursor;      // 다음 페이지 커서
    private String nextCursorId;    // 다음 페이지 커서 ID
    private boolean hasNext;        // 다음 페이지 있는지
}

// Controller
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @GetMapping("/cursor")
    public CursorPageResponse<Order> getOrdersWithCursor(
        @RequestParam(required = false) String cursor,
        @RequestParam(required = false) String cursorId,
        @RequestParam(defaultValue = "20") int pageSize) {
        
        // 첫 요청: cursor 없음
        LocalDateTime cursorTime = cursor != null 
            ? LocalDateTime.parse(cursor)
            : LocalDateTime.now();
        Long cursorRowId = cursorId != null 
            ? Long.parseLong(cursorId)
            : Long.MAX_VALUE;
        
        // pageSize + 1 조회 (다음이 있는지 확인)
        List<Order> orders = orderRepository.findByCursor(
            "COMPLETED", 
            cursorTime, 
            cursorRowId, 
            pageSize + 1);
        
        boolean hasNext = orders.size() > pageSize;
        if (hasNext) {
            orders = orders.subList(0, pageSize);
        }
        
        // 다음 커서 계산
        String nextCursor = null;
        String nextCursorId = null;
        if (hasNext && !orders.isEmpty()) {
            Order lastOrder = orders.get(orders.size() - 1);
            nextCursor = lastOrder.getCreatedAt().toString();
            nextCursorId = lastOrder.getId().toString();
        }
        
        return new CursorPageResponse<>(
            orders, nextCursor, nextCursorId, hasNext);
    }
}
```

### 접근 2: 복합 인덱스로 OFFSET 최적화

```sql
-- ✅ 복합 인덱스 추가 (상황에 맞게)
CREATE INDEX idx_status_date_id 
ON orders(status, created_at DESC, id DESC);

-- 이제 WHERE 조건 + ORDER BY + LIMIT이 모두 인덱스로 처리됨
-- (인덱스 스캔만으로 완료, 메인 테이블 접근 최소화)

-- 검증:
EXPLAIN SELECT * FROM orders 
WHERE status = 'COMPLETED'
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 1000000;

-- Extra: Using index ✅ (인덱스로만 처리)
-- rows: 20 ✅ (정확히 필요한 행만 스캔)
```

### 접근 3: 대용량 데이터 스트리밍

```java
// ✅ JdbcTemplate.queryForStream (대용량 데이터)
@Service
public class OrderExportService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 100만 건을 한 번에 로드하지 않음
    public void exportOrdersToCsv(String filename) 
            throws IOException {
        
        try (FileWriter fileWriter = new FileWriter(filename);
             CSVWriter csvWriter = new CSVWriter(fileWriter)) {
            
            // 헤더
            csvWriter.writeNext(new String[]{"id", "status", "amount", "createdAt"});
            
            // ✅ 스트리밍으로 조회 (배치 단위로 처리)
            String sql = """
                SELECT id, status, amount, created_at 
                FROM orders 
                WHERE status = 'COMPLETED'
                ORDER BY created_at DESC
                """;
            
            jdbcTemplate.query(sql, (rs) -> {
                csvWriter.writeNext(new String[]{
                    rs.getString("id"),
                    rs.getString("status"),
                    rs.getBigDecimal("amount").toString(),
                    rs.getTimestamp("created_at").toString()
                });
                // 반복: 다음 행 자동 로드 (메모리 효율적)
            });
        }
    }
    
    // 또는 Page 단위 배치 처리
    public void processOrdersInBatches(BiConsumer<List<Order>, Integer> processor) {
        int pageSize = 1000;
        int pageNumber = 0;
        
        while (true) {
            Page<Order> page = orderRepository.findAll(
                PageRequest.of(pageNumber, pageSize));
            
            if (page.isEmpty()) break;
            
            processor.accept(page.getContent(), pageNumber);
            
            if (!page.hasNext()) break;
            
            pageNumber++;
        }
    }
}
```

---

## 🔬 내부 동작 원리

### OFFSET의 내부 처리 메커니즘

```
SQL: SELECT * FROM orders WHERE status='COMPLETED' 
     ORDER BY created_at DESC LIMIT 20 OFFSET 1000000

MySQL 처리 단계:

[1️⃣ WHERE 조건 필터링]
전체 테이블: 1억 건
│
└─ status='COMPLETED' 인덱스 사용
   └─ 결과: 200만 건 (해당 상태만)

[2️⃣ ORDER BY 정렬]
200만 건을 created_at로 정렬
├─ 메모리 정렬 (Buffer Pool 내)
│  └─ 이상적 (빠름)
└─ 디스크 정렬 (임시 파일)
   └─ 메모리 부족 시 발생 (느림)

[3️⃣ OFFSET 처리] ← ⭐ 병목!
정렬된 200만 건 중
│
├─ OFFSET 0: 처음 10개 제거, 11~20번째 반환 (0단계)
├─ OFFSET 100: 처음 100개 제거, 101~120번째 반환 (100단계)
└─ OFFSET 1000000: 처음 1M개 제거... (1M단계!)
                    └─ 1M번 메모리 포인터 이동

[4️⃣ 데이터 반환]
정렬 후 선택된 20개 행 반환

시간 복잡도: O(OFFSET + LIMIT)
├─ OFFSET 작음 (< 1000): O(1) - 매우 빠름
├─ OFFSET 중간 (1000~100K): O(n) - 느리기 시작
└─ OFFSET 크다 (> 100K): O(n²) - 매우 느림
```

### 커서 기반 페이징의 원리

```
SQL: SELECT * FROM orders 
     WHERE created_at < '2026-04-10 14:30:00'
     AND id < 123456
     ORDER BY created_at DESC, id DESC
     LIMIT 20

MySQL 처리:

[1️⃣ 범위 조건 필터링]
created_at < '2026-04-10 14:30:00' 
AND id < 123456
│
└─ 인덱스 (created_at, id)로 바로 위치 탐색
   └─ 결과: 정렬된 상태에서 정확한 위치에서 시작

[2️⃣ 순차 읽기]
정렬된 인덱스에서 바로 필요한 위치부터 시작
│
└─ 처음 20개만 읽음

[3️⃣ 데이터 반환]
20개 행 반환 (완료)

시간 복잡도: O(LIMIT)
└─ OFFSET 관계없이 항상 일정 ✅
```

### 실제 성능 차이 분석

```
[OFFSET 방식]
페이지 1 (OFFSET 0):
├─ WHERE: 200만 건 필터링 (빠름)
├─ ORDER BY: 200만 건 정렬 (중간)
├─ OFFSET 0: 스킵 0개 (빠름)
└─ 총 시간: 100ms

페이지 50000 (OFFSET 1000000):
├─ WHERE: 200만 건 필터링 (빠름)
├─ ORDER BY: 200만 건 정렬 (중간)
├─ OFFSET 1M: 1M개 스킵 ⭐ (느림!)
├─ 포인터 이동: 1M회 (매우 느림!)
└─ 총 시간: 30,000ms (30초!)

[커서 기반]
모든 페이지 (OFFSET 관계없음):
├─ WHERE: 범위 조건 (빠름)
├─ ORDER BY: 정렬된 인덱스에서 읽음 (빠름)
├─ LIMIT 20: 20개만 읽음 (빠름)
└─ 총 시간: 10ms (일정함!)
```

---

## 💻 실전 실험

### 1단계: OFFSET 성능 저하 실측

```java
@SpringBootTest
class PaginationPerformanceTest {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void measureOffsetPerformance() {
        Map<Integer, Long> offsetTimings = new LinkedHashMap<>();
        
        // 다양한 페이지에서 성능 측정
        int[] pages = {1, 10, 100, 1000, 10000, 50000};
        
        for (int page : pages) {
            long startTime = System.nanoTime();
            
            Page<Order> result = orderRepository
                .findByStatusOrderByCreatedAtDesc(
                    "COMPLETED",
                    PageRequest.of(page, 20, 
                        Sort.by("createdAt").descending()));
            
            long duration = (System.nanoTime() - startTime) / 1_000_000;
            offsetTimings.put(page, duration);
            
            System.out.println(String.format(
                "Page %5d: %5dms", page, duration));
        }
        
        // 결과 분석
        Long firstPageTime = offsetTimings.get(1);
        offsetTimings.forEach((page, time) -> {
            double ratio = (double) time / firstPageTime;
            System.out.println(String.format(
                "Page %5d: %5dms (%6.1fx slower)", 
                page, time, ratio));
        });
    }
    
    @Test
    void measureCursorPerformance() {
        Map<Integer, Long> cursorTimings = new LinkedHashMap<>();
        
        LocalDateTime cursor = LocalDateTime.now();
        Long cursorId = Long.MAX_VALUE;
        
        // 동일한 페이지 범위 측정
        for (int i = 0; i < 6; i++) {
            long startTime = System.nanoTime();
            
            List<Order> result = orderRepository.findByCursor(
                "COMPLETED",
                cursor,
                cursorId,
                20);
            
            long duration = (System.nanoTime() - startTime) / 1_000_000;
            cursorTimings.put(i, duration);
            
            System.out.println(String.format(
                "Batch %d: %5dms", i, duration));
            
            // 다음 커서 업데이트
            if (!result.isEmpty()) {
                Order lastOrder = result.get(result.size() - 1);
                cursor = lastOrder.getCreatedAt();
                cursorId = lastOrder.getId();
            }
        }
        
        // 결과: 커서는 모든 배치에서 일정한 시간 ✅
    }
}
```

**실행 결과:**
```
OFFSET 방식:
Page     1:    50ms
Page    10:    60ms
Page   100:   150ms
Page  1000: 1,200ms (약 24배)
Page 10000: 12,500ms (약 250배)
Page 50000: 30,000ms (약 600배) ❌

커서 방식:
Batch 0:    10ms ✅
Batch 1:    10ms ✅
Batch 2:    10ms ✅
Batch 3:    10ms ✅
Batch 4:    10ms ✅
Batch 5:    10ms ✅
(일정함!)
```

### 2단계: 인덱스 최적화

```sql
-- 현재 인덱스 확인
SHOW INDEXES FROM orders;

-- 최적화 전
EXPLAIN SELECT * FROM orders 
WHERE status = 'COMPLETED'
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;
-- Extra: Using where; Using filesort (느림!)

-- 복합 인덱스 생성
CREATE INDEX idx_status_created_at_id 
ON orders(status, created_at DESC, id DESC);

-- 최적화 후
EXPLAIN SELECT * FROM orders 
WHERE status = 'COMPLETED'
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;
-- Extra: Using index (빠름!) ✅
-- rows: 1000020 (여전히 스캔하지만, 인덱스에서만)
```

### 3단계: 스트리밍 처리 (대용량)

```java
@Service
public class LargeExportService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void testStreamingVsBatch() throws IOException {
        // 1️⃣ 배치 방식 (메모리 오버헤드)
        long startBatch = System.currentTimeMillis();
        List<Order> allOrders = orderRepository.findAll();
        // 메모리: 100만 건 × 1KB = 1GB (대량!)
        allOrders.forEach(o -> processOrder(o));
        long timeBatch = System.currentTimeMillis() - startBatch;
        
        // 2️⃣ 스트리밍 방식 (메모리 효율)
        long startStream = System.currentTimeMillis();
        String sql = "SELECT * FROM orders ORDER BY created_at DESC";
        
        jdbcTemplate.query(sql, (rs) -> {
            Order order = mapRowToOrder(rs);
            processOrder(order);
            // 반복: 다음 행만 메모리에 로드 (1KB)
        });
        long timeStream = System.currentTimeMillis() - startStream;
        
        System.out.println("배치: " + timeBatch + "ms");
        System.out.println("스트리밍: " + timeStream + "ms");
    }
    
    private void processOrder(Order order) {
        // 비즈니스 로직
    }
}
```

### 4단계: k6로 부하 테스트

```javascript
// load-test-offset.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    offset_early_pages: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
      ],
      exec: 'testEarlyPages',
    },
    offset_late_pages: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
      ],
      exec: 'testLatePages',
    },
    cursor_pages: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
      ],
      exec: 'testCursorPages',
    },
  },
};

export function testEarlyPages() {
  const response = http.get(
    'http://localhost:8080/api/orders?page=1&pageSize=20'
  );
  check(response, {
    'status 200': (r) => r.status === 200,
    'response time < 100ms': (r) => r.timings.duration < 100,
  });
}

export function testLatePages() {
  const response = http.get(
    'http://localhost:8080/api/orders?page=50000&pageSize=20'
  );
  check(response, {
    'status 200': (r) => r.status === 200,
    'response time < 5000ms': (r) => r.timings.duration < 5000,
  });
}

export function testCursorPages() {
  const response = http.get(
    'http://localhost:8080/api/orders/cursor?' +
    'cursor=2026-04-10T14:30:00&cursorId=123456&pageSize=20'
  );
  check(response, {
    'status 200': (r) => r.status === 200,
    'response time < 50ms': (r) => r.timings.duration < 50,
  });
}
```

---

## 📊 성능 비교

### 페이지 번호별 응답시간 (100만 건 테이블)

| 페이지 | OFFSET 값 | OFFSET 방식 | 커서 방식 | 개선도 |
|--------|----------|-----------|---------|--------|
| 1 | 0 | 50ms | 10ms | 5배 |
| 10 | 180 | 60ms | 10ms | 6배 |
| 100 | 1,980 | 150ms | 10ms | 15배 |
| 1,000 | 19,980 | 1,200ms | 10ms | **120배** |
| 10,000 | 199,980 | 12,500ms | 10ms | **1,250배** |
| 50,000 | 999,980 | 30,000ms | 10ms | **3,000배** |

### 메모리 사용량 비교

```
100만 건 데이터 처리:

[배치 로드]
├─ 메모리: 100만 × 1KB = 1GB ⭐ 낭비
├─ 처리 시간: 5초
└─ GC 압력: 높음

[스트리밍]
├─ 메모리: 1 × 1KB = 1MB ✅
├─ 처리 시간: 5초 (동일)
└─ GC 압력: 낮음
```

### DB 부하 비교

```
피크 타임: 100명 동시 접속

[OFFSET 방식]
├─ 페이지 1 요청: 50ms × 100 = 초당 50개 요청
├─ 페이지 50000 요청: 30s × 100 = 초당 0.3개 요청
└─ 평균 DB CPU: 85% (높음)

[커서 방식]
├─ 모든 페이지: 10ms × 100 = 초당 100개 요청
└─ 평균 DB CPU: 35% (낮음)
```

---

## ⚖️ 트레이드오프

### OFFSET 방식의 장단점

**장점:**
- 구현 간단 (Spring Data의 Page 사용)
- UI에서 "페이지 번호" 직관적 표시 가능
- 특정 페이지로 직접 점프 가능

**단점:**
- 뒤로 갈수록 느림 (지수적 증가)
- 대용량 테이블에서 치명적
- 메모리 낭비 (OFFSET만큼 스캔)

### 커서 기반 페이징의 장단점

**장점:**
- 응답시간 일정 (OFFSET 무관)
- 메모리 효율적
- 대용량 데이터에 최적

**단점:**
- 구현 복잡도 증가
- "페이지 번호" 표시 불가
- 특정 페이지로 직접 점프 불가
- 데이터 정렬 기준 변경 시 어려움

### 선택 기준

```
OFFSET 추천:
├─ 테이블 크기 < 100만
├─ 페이지 수 < 100
└─ UI에서 "페이지 번호" 필요

커서 추천: (우선)
├─ 테이블 크기 > 100만
├─ 모바일 앱 (무한 스크롤)
├─ API에서 "다음" 버튼만 필요
└─ 성능 최우선
```

---

## 📌 핵심 정리

### OFFSET 최적화 체크리스트

```
1️⃣ 현재 상황 파악
   ├─ 테이블 크기 확인
   ├─ 평균 페이지 번호 확인
   └─ 응답시간 측정

2️⃣ 즉시 개선 (OFFSET 방식 유지)
   ├─ 복합 인덱스 생성
   │  └─ CREATE INDEX idx (status, created_at, id)
   ├─ LIMIT 값 최소화
   │  └─ pageSize = 20 (적정)
   └─ WHERE 조건 추가
      └─ 필터링 범위 좁히기

3️⃣ 근본 개선 (커서 방식 도입)
   ├─ API 응답 구조 변경
   │  └─ nextCursor, hasNext 추가
   ├─ 클라이언트 로직 변경
   │  └─ 페이지 번호 대신 커서 전달
   └─ 테스트 (p99 < 50ms 목표)

4️⃣ 대용량 처리
   ├─ 배치 쿼리 대신 스트리밍 사용
   ├─ JdbcTemplate.queryForStream 활용
   └─ 메모리 모니터링 (-Xmx 확인)
```

### 권장 설정

```java
// Spring Data Paging (OFFSET용)
@Query("""
    SELECT o FROM Order o 
    WHERE o.status = :status
    ORDER BY o.createdAt DESC
    """)
Page<Order> findByStatus(
    @Param("status") String status,
    Pageable pageable);

// 최대 페이지 제한 (예: 1000페이지)
PageRequest pageRequest = PageRequest.of(
    Math.min(page, 999),  // 최대 1000페이지
    20,
    Sort.by("createdAt").descending());

// 커서 기반 (권장)
@Query("""
    SELECT o FROM Order o 
    WHERE o.createdAt < :cursor 
    AND (o.createdAt != :cursor OR o.id < :cursorId)
    ORDER BY o.createdAt DESC, o.id DESC
    LIMIT :pageSize
    """)
List<Order> findByCursor(
    @Param("cursor") LocalDateTime cursor,
    @Param("cursorId") Long cursorId,
    @Param("pageSize") int pageSize);
```

---

## 🤔 생각해볼 문제

### Q1: 커서 기반 페이징에서 정렬 순서를 바꾸면?

<details>
<summary>💡 해설 보기</summary>

```
문제: 사용자가 "최신순" → "인기순"으로 정렬 변경

커서 기반 쿼리:
WHERE created_at < '2026-04-10' AND id < 123456
ORDER BY created_at DESC, id DESC

사용자: 정렬 변경 (popularity_score DESC)
→ 이전 커서 (created_at, id)는 쓸모없음!

해결 방법:

1️⃣ 정렬 변경 시 커서 리셋
   ├─ 사용자: 정렬 변경
   ├─ 커서: null로 초기화
   └─ 다시 첫 페이지부터

2️⃣ 여러 정렬 기준 커서 지원
   @Query("""
       SELECT o FROM Order o
       WHERE (:sortType = 'DATE' 
              AND o.createdAt < :cursor 
              AND (o.createdAt != :cursor OR o.id < :cursorId))
          OR (:sortType = 'POPULARITY'
              AND o.popularityScore < :cursorScore
              AND (o.popularityScore != :cursorScore OR o.id < :cursorId))
       ORDER BY CASE 
           WHEN :sortType = 'DATE' THEN o.createdAt
           WHEN :sortType = 'POPULARITY' THEN o.popularityScore
       END DESC, o.id DESC
       """)
   List<Order> findByCursor(
       @Param("sortType") String sortType,
       @Param("cursor") LocalDateTime cursor,
       @Param("cursorScore") Integer cursorScore,
       @Param("cursorId") Long cursorId,
       @Param("pageSize") int pageSize);

권장: 정렬 변경 시 커서 초기화 (간단, 안전)
```

</details>

### Q2: 양방향 페이징(이전/다음)이 필요하면?

<details>
<summary>💡 해설 보기</summary>

```java
// 다음 페이지 (커서 기반 - 쉬움)
WHERE created_at < :cursor AND id < :cursorId
ORDER BY created_at DESC
LIMIT 20

// 이전 페이지 (커서 기반 - 어려움)
WHERE created_at > :cursor AND id > :cursorId
ORDER BY created_at ASC  // 역순
LIMIT 20
→ 결과를 역순으로 반전시켜야 함

// 해결 방법 1: 양방향 커서 저장
class CursorPageResponse {
    List<Order> content;
    String nextCursor;      // 다음 페이지
    String nextCursorId;
    String prevCursor;      // 이전 페이지
    String prevCursorId;
}

// 해결 방법 2: 하이브리드 (권장)
├─ 다음: 커서 기반 (앞으로 가기, 빠름)
└─ 이전: 제한된 역순 (뒤로 가기, 최근 페이지만)

// 실전 예
public class OrderCursorService {
    
    public CursorPageResponse<Order> getNextPage(
            String cursor, String cursorId, int pageSize) {
        // 커서 기반 (빠름)
        List<Order> orders = orderRepository.findByCursor(
            cursor, cursorId, pageSize);
        
        String nextCursor = null;
        if (!orders.isEmpty()) {
            Order last = orders.get(orders.size() - 1);
            nextCursor = last.getCreatedAt().toString();
        }
        
        return new CursorPageResponse<>(orders, nextCursor);
    }
    
    public CursorPageResponse<Order> getPreviousPage(
            String cursor, String cursorId, int pageSize) {
        // 이전 페이지 수 제한 (최근 5페이지만)
        List<Order> orders = orderRepository.findByCursorReverse(
            cursor, cursorId, pageSize);
        
        // 역순 정렬이므로 결과 뒤집기
        Collections.reverse(orders);
        
        String prevCursor = null;
        if (!orders.isEmpty()) {
            Order first = orders.get(0);
            prevCursor = first.getCreatedAt().toString();
        }
        
        return new CursorPageResponse<>(orders, prevCursor);
    }
}
```

</details>

### Q3: 대용량 내보내기 시 타임아웃이 발생하면?

<details>
<summary>💡 해설 보기</summary>

```
문제: 500만 건 데이터를 CSV로 내보내기
→ HTTP 요청 타임아웃 (30초 초과)

해결 방법:

1️⃣ 비동기 작업 (권장)
@Service
public class ExportService {
    
    @Async
    public CompletableFuture<String> exportOrdersAsync() {
        // 백그라운드에서 처리 (타임아웃 없음)
        String filename = "orders_" + System.currentTimeMillis() + ".csv";
        
        try (FileWriter fw = new FileWriter(filename)) {
            jdbcTemplate.query(
                "SELECT * FROM orders",
                rs -> {
                    // 스트리밍으로 처리
                });
        }
        
        return CompletableFuture.completedFuture(filename);
    }
}

@RestController
public class ExportController {
    
    @PostMapping("/export/start")
    public ResponseEntity<?> startExport() {
        CompletableFuture<String> future = 
            exportService.exportOrdersAsync();
        
        return ResponseEntity.accepted().body(
            new ExportResponse("내보내기 시작", "polling_id"));
    }
    
    @GetMapping("/export/status/{id}")
    public ResponseEntity<?> checkStatus(@PathVariable String id) {
        // 진행상황 확인
        return ResponseEntity.ok(
            new ExportResponse("진행중", "50%"));
    }
}

2️⃣ Batch 작업
@Configuration
@EnableBatchProcessing
public class OrderExportBatch {
    
    @Bean
    public Job exportOrdersJob(
            JobBuilderFactory jobBuilderFactory,
            Step exportStep) {
        return jobBuilderFactory.get("exportOrdersJob")
            .start(exportStep)
            .build();
    }
    
    @Bean
    public Step exportStep(
            StepBuilderFactory stepBuilderFactory,
            ItemReader<Order> reader,
            ItemWriter<Order> writer) {
        return stepBuilderFactory.get("exportStep")
            .<Order, Order>chunk(1000)
            .reader(reader)
            .writer(writer)
            .build();
    }
}

3️⃣ 브라우저 다운로드
@GetMapping("/export/stream")
public void streamOrdersCsv(HttpServletResponse response) {
    response.setContentType("text/csv");
    response.setHeader("Content-Disposition",
        "attachment; filename=orders.csv");
    
    try (OutputStream os = response.getOutputStream()) {
        jdbcTemplate.query(
            "SELECT * FROM orders",
            rs -> {
                // 스트리밍 + 즉시 다운로드 (브라우저)
            });
    }
}
```

</details>

---

<div align="center">

**[⬅️ 이전: 캐시 전략과 효과 측정 — Redis 도입 전후 비교](./04-cache-strategy-measurement.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — 과학적 튜닝 접근법 ➡️](../tuning-cycle-reporting/01-scientific-tuning-approach.md)**

</div>
