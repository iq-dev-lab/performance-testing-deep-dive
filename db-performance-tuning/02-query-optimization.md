# 02. 쿼리 성능 최적화 — EXPLAIN ANALYZE와 N+1

---

## 🎯 핵심 질문

- `EXPLAIN ANALYZE`로 어떤 정보를 읽어야 할까? (type, key, rows, Extra)
- N+1 쿼리란 무엇이고, JPA에서 왜 자꾸 발생할까?
- Full Table Scan을 인덱스로 해결할 수 있나?

---

## 🔍 왜 이 개념이 실무에서 중요한가

느린 쿼리는 DB뿐만 아니라 **전체 애플리케이션 성능**을 결정합니다.
- 개별 쿼리 100ms 느림 → 페이지 로딩 1초 지연 (다른 요청 10개)
- N+1 문제 → 1개 요청이 1001개 쿼리로 변환 (DB 부하 1000배!)
- 인덱스 없는 Full Table Scan → 백만 건 테이블에서 몇 초 소요

**실전 팁:** 대부분의 성능 문제는 **쿼리**에서 비롯됩니다.

---

## 😱 흔한 실수 (Before)

### 실수 1: EXPLAIN 결과를 무시하기

```sql
-- 느린 쿼리
SELECT * FROM users WHERE email = 'test@example.com';

-- EXPLAIN 결과 (무시됨)
+----+-------------+-------+------+---------------+------+---------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+-------+
|  1 | SIMPLE      | users | ALL  | NULL          | NULL | NULL    | 50M  | WHERE |
+----+-------------+-------+------+---------------+------+---------+------+-------+
-- 😱 ALL = Full Table Scan (50M 행 모두 스캔!)
```

### 실수 2: N+1 쿼리를 모르고 배포

```java
// JPA Entity
@Entity
public class Order {
    @Id
    private Long id;
    
    @ManyToOne  // Lazy Loading (기본값)
    private Customer customer;
}

// Controller (문제 있는 코드)
@GetMapping("/orders")
public List<OrderDto> getOrders() {
    List<Order> orders = orderRepository.findAll();
    
    // 이 시점에서:
    // 1번 쿼리: SELECT * FROM orders; (1000건)
    // 다음 루프에서:
    // 1000번 쿼리: SELECT * FROM customers WHERE id = ?;
    // 총 1001번의 쿼리 발생!
    
    return orders.stream()
        .map(order -> new OrderDto(
            order.getId(),
            order.getCustomer().getName()  // 😱 여기서 추가 쿼리 발생!
        ))
        .collect(Collectors.toList());
}
```

**결과:**
```
예상: 쿼리 1개, 100ms
실제: 쿼리 1001개, 15초 (컨트롤러가 응답 지연)
```

---

## ✨ 올바른 접근 (After)

### 접근 1: EXPLAIN ANALYZE로 정확히 진단

```sql
-- ✅ 올바른 방법: EXPLAIN ANALYZE 실행
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';

-- 분석 결과:
Planning Time: 0.123 ms
Execution Time: 5234.456 ms  -- 5초 소요! (Full Table Scan)

-- 개선: 인덱스 추가
CREATE INDEX idx_users_email ON users(email);

-- 재실행:
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';

Planning Time: 0.087 ms
Execution Time: 0.234 ms  -- 0.2ms로 개선! (22,000배 빠름)
```

### 접근 2: Fetch Join으로 N+1 해결

```java
// JPA Repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // ❌ Before: N+1 발생
    List<Order> findAll();
    
    // ✅ After: Fetch Join으로 한 번에 조회
    @Query("SELECT o FROM Order o JOIN FETCH o.customer")
    List<Order> findAllWithCustomer();
    
    // 또는 @EntityGraph 사용
    @EntityGraph(attributePaths = {"customer"})
    @Query("SELECT o FROM Order o")
    List<Order> findAllWithCustomerGraph();
}

// Controller (개선된 코드)
@GetMapping("/orders")
public List<OrderDto> getOrders() {
    // 이제 2번의 쿼리만 발생:
    // 1번: SELECT o FROM orders o JOIN FETCH customers c
    // 2번: (연관 로딩은 한 번에)
    
    List<Order> orders = orderRepository.findAllWithCustomer();
    
    return orders.stream()
        .map(order -> new OrderDto(
            order.getId(),
            order.getCustomer().getName()  // ✅ 추가 쿼리 없음!
        ))
        .collect(Collectors.toList());
}
```

---

## 🔬 내부 동작 원리

### EXPLAIN의 각 컬럼 해석

```
+----+-------------+-------+-------+---------------+-------+---------+-------+-------+
| id | select_type | table | type  | possible_keys | key   | key_len | rows  | Extra |
+----+-------------+-------+-------+---------------+-------+---------+-------+-------+

1️⃣ type (가장 중요!)
   const    - 상수 조회 (PRIMARY KEY = 1)           [1ms] ⭐⭐⭐
   eq_ref   - JOIN에서 PRIMARY KEY 조회           [1ms] ⭐⭐⭐
   ref      - 일반 인덱스 조회                     [10ms] ⭐⭐
   range    - 범위 조회 (BETWEEN, >, <)           [100ms] ⭐
   index    - 인덱스 전체 스캔                     [1000ms] ⚠️
   ALL      - Full Table Scan                      [10000ms] 💥

2️⃣ key (사용 중인 인덱스)
   - NULL: 인덱스를 안 쓰고 있음 (문제!)
   - PRIMARY: 기본키 사용
   - idx_email: 이메일 인덱스 사용

3️⃣ rows (스캔할 예상 행 수)
   - 1: 정확한 조회
   - 1000: 1000행 스캔 (나쁨)
   - 50000000: 5천만 행 스캔 (매우 나쁨)

4️⃣ Extra (추가 정보)
   - Using index: 인덱스만으로 완료 (최고)
   - Using where: 필터링 중
   - Using filesort: 메모리에서 정렬 (느림)
   - Using temporary: 임시 테이블 (매우 느림)
```

### JPA의 N+1 발생 메커니즘

```
@ManyToOne(fetch = FetchType.LAZY)  // 기본값
private Customer customer;

동작 흐름:
1️⃣ SELECT o FROM orders o;
   ↓
   [Order1, Order2, Order3, ..., Order1000]
   └─ customer는 아직 로딩 안 됨 (Proxy 객체)

2️⃣ order.getCustomer().getName()
   ↓
   └─ Proxy에서 실제 데이터 접근
   └─ "SELECT * FROM customers WHERE id = 1" 실행!
   └─ (Order마다 반복)

결과: 1 + 1000 = 1001번 쿼리
```

### Fetch Join이 작동하는 방식

```
@Query("SELECT o FROM Order o JOIN FETCH o.customer")

SQL로 변환:
SELECT o.*, c.*
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id

결과:
[Order1-Customer1, Order2-Customer2, ...]
└─ 한 번의 쿼리에 모든 데이터 담김
└─ customer 프록시도 이미 로딩됨

따라서 추가 쿼리 없음!
```

---

## 💻 실전 실험

### 1단계: 느린 쿼리 찾기 (p6spy)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy:mysql://localhost:3306/mydb

# spy.properties
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
appender=com.p6spy.engine.spy.appender.FileLogger
logfile=logs/p6spy.log
```

```bash
# 실행 후 로그 분석
tail -100 logs/p6spy.log | grep -E "(SELECT|UPDATE)" | head -20
```

### 2단계: EXPLAIN ANALYZE로 진단

```java
@Component
public class QueryAnalyzer {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void analyzeQuery(String sql) {
        String explainSql = "EXPLAIN ANALYZE " + sql;
        
        List<Map<String, Object>> results = 
            jdbcTemplate.queryForList(explainSql);
        
        results.forEach(row -> {
            System.out.println("Type: " + row.get("type"));
            System.out.println("Key: " + row.get("key"));
            System.out.println("Rows: " + row.get("rows"));
            System.out.println("Extra: " + row.get("Extra"));
        });
    }
}
```

### 3단계: N+1 감지 (Spring JPA)

```java
// application.yml - SQL 로깅 활성화
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        generate_statistics: true
        use_sql_comments: true

// 실행 후 로그에 1001개 쿼리 보임 😱
```

```java
// 해결: Fetch Join 적용
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query(value = """
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.customer
        LEFT JOIN FETCH o.items
        """)
    List<Order> findAllOptimized();
}
```

### 4단계: 성능 비교 테스트

```java
@SpringBootTest
class QueryPerformanceTest {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void compareNPlusOneVsFetchJoin() {
        // N+1 방식
        long start1 = System.currentTimeMillis();
        List<Order> orders1 = orderRepository.findAll();
        orders1.forEach(o -> System.out.println(o.getCustomer().getName()));
        long time1 = System.currentTimeMillis() - start1;
        
        // Fetch Join 방식
        long start2 = System.currentTimeMillis();
        List<Order> orders2 = orderRepository.findAllOptimized();
        orders2.forEach(o -> System.out.println(o.getCustomer().getName()));
        long time2 = System.currentTimeMillis() - start2;
        
        System.out.println("N+1 시간: " + time1 + "ms");
        System.out.println("Fetch Join 시간: " + time2 + "ms");
        System.out.println("개선율: " + (time1 / (double)time2) + "배");
    }
}
```

### 5단계: 인덱스 추가 및 검증

```sql
-- 현재 인덱스 조회
SHOW INDEXES FROM users;

-- 느린 쿼리 찾기
SELECT * FROM users WHERE email = 'test@example.com';
-- EXPLAIN: type=ALL, rows=50000000 (문제!)

-- 인덱스 생성
CREATE INDEX idx_users_email ON users(email);

-- 검증
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
-- EXPLAIN: type=ref, rows=1 (개선!)

-- 복합 인덱스
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);
```

---

## 📊 성능 비교

### N+1 vs Fetch Join

| 시나리오 | 쿼리 수 | 응답시간 | DB CPU | 메모리 |
|---------|--------|---------|--------|--------|
| N+1 (1000건) | 1001개 | 8,234ms | 85% | 450MB |
| Fetch Join (1000건) | 1개 | 127ms | 15% | 50MB |
| **개선율** | **1000배 감소** | **65배 빠름** | **↓70%** | **↓89%** |

### Full Table Scan vs 인덱스

```
테이블: 5천만 건

[Before] Full Table Scan (type=ALL)
SELECT * FROM users WHERE email = 'test@example.com'
├─ 스캔 행: 50,000,000
├─ 소요시간: 5,234ms
├─ DB I/O: 수백 번
└─ 결과: 1건

[After] 인덱스 사용 (type=ref)
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'test@example.com'
├─ 스캔 행: 1
├─ 소요시간: 0.234ms
├─ DB I/O: 5회 (트리 탐색)
└─ 결과: 1건

개선: 22,000배 빠름!
```

### EXPLAIN 분석 결과 비교

```
[Before] 인덱스 없음
+----+--------+--------+------+-----------+------+-------+-------+
| id | type   | table  | key  | rows      | Extra |
+----+--------+--------+------+-----------+------+-------+-------+
|  1 | ALL    | orders | NULL | 1000000   | WHERE | ❌
+----+--------+--------+------+-----------+------+-------+-------+

[After] 복합 인덱스 추가
+----+--------+--------+------------------+------+-------+-------+
| id | type   | table  | key              | rows | Extra |
+----+--------+--------+------------------+------+-------+-------+
|  1 | range  | orders | idx_date_status  | 1250 | ✅
+----+--------+--------+------------------+------+-------+-------+
```

---

## ⚖️ 트레이드오프

### 더 많은 인덱스 추가

**장점:**
- SELECT 쿼리 빨라짐
- EXPLAIN의 type 개선 (ALL → ref)

**단점:**
- INSERT/UPDATE/DELETE 느려짐 (인덱스도 업데이트)
- 저장소 사용량 증가 (인덱스도 용량)
- 인덱스 관리 복잡도

### Fetch Join으로 모든 관계 로딩

**장점:**
- N+1 문제 해결
- 데이터 일관성 보장

**단점:**
- 불필요한 데이터도 로드 (메모리 낭비)
- 복잡한 쿼리 (유지보수 어려움)
- 조인이 많으면 Cartesian product 발생

### 결론: 균형 잡기

```
1️⃣ 읽기 위주 → 더 많은 인덱스 ✅
2️⃣ 쓰기 위주 → 인덱스 최소화 ✅
3️⃣ N+1 → 항상 해결해야 함 ⭐
4️⃣ 메모리 제약 → 필요한 컬럼만 선택 SELECT ✅
```

---

## 📌 핵심 정리

### EXPLAIN 해석 체크리스트

```
type 확인
├─ ALL/index: 인덱스 추가 필요 ❌
├─ range: 범위 쿼리 (괜찮음)
└─ ref/const: 최고 (인덱스 잘 사용)

rows 확인
├─ > 1000: 많이 스캔함 ⚠️
└─ 1~100: 양호 ✅

Extra 확인
├─ Using filesort: 메모리 정렬 (느림)
├─ Using temporary: 임시 테이블 (매우 느림)
└─ Using index: 최고 (커버링 인덱스)
```

### N+1 해결 체크리스트

```
1️⃣ p6spy로 중복 쿼리 감지
2️⃣ Fetch Join 또는 @EntityGraph 적용
3️⃣ 복잡한 경우 QueryDSL로 명시적 조회
4️⃣ 테스트로 쿼리 수 검증
```

### 인덱스 설계 원칙

```
1️⃣ WHERE절 컬럼에 우선순위
2️⃣ 복합 인덱스: 선택도 높은 것부터
3️⃣ 범위 검색은 마지막에
4️⃣ INSERT/UPDATE 영향도 고려
```

---

## 🤔 생각해볼 문제

### Q1: LEFT JOIN FETCH vs INNER JOIN FETCH의 차이는?

<details>
<summary>💡 해설 보기</summary>

```java
// LEFT JOIN FETCH (외부 조인)
@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.customer")
List<Order> findAllWithLeftJoin();

// INNER JOIN FETCH (내부 조인)
@Query("SELECT DISTINCT o FROM Order o INNER JOIN FETCH o.customer")
List<Order> findAllWithInnerJoin();

차이:
LEFT JOIN FETCH:
├─ Order가 Customer 없어도 포함됨
└─ 사용처: 선택적 관계 (customer 없을 수 있음)

INNER JOIN FETCH:
├─ Customer 있는 Order만 포함
└─ 사용처: 필수 관계 (customer 반드시 있음)

예시:
주문(Order)에 고객(Customer)이 없는 경우:
├─ LEFT JOIN: [Order, Order(customer=null), Order]
└─ INNER JOIN: [Order, Order]
```

</details>

### Q2: 인덱스는 몇 개까지 만드는 게 좋을까?

<details>
<summary>💡 해설 보기</summary>

```
테이블별 인덱스 개수 가이드:

기본 규칙:
├─ PRIMARY KEY: 1개 (필수)
├─ 자주 조회: 1~3개 추가
├─ 복합 인덱스: 1~2개
└─ 총 5개 이하 권장

예시 - 좋은 구조:
CREATE TABLE users (
    id BIGINT PRIMARY KEY,           -- idx_1: PRIMARY
    email VARCHAR(100) UNIQUE,       -- idx_2: email (자주 검색)
    created_at DATETIME,             -- idx_3: created_at (범위 검색)
    status VARCHAR(20),
    INDEX idx_status_date (status, created_at)  -- idx_4: 복합
);

나쁜 예:
├─ 모든 컬럼에 인덱스 추가
├─ 중복되는 인덱스
└─ 사용되지 않는 인덱스

체크:
SHOW INDEXES FROM users;
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage;
```

</details>

### Q3: EXPLAIN ANALYZE vs EXPLAIN의 차이는?

<details>
<summary>💡 해설 보기</summary>

```
EXPLAIN (예상)
├─ 쿼리를 실행하지 않음
├─ 통계 기반 예측값만 표시
├─ 빠름 (밀리초 단위)
└─ 예: rows=1000 (예상)

EXPLAIN ANALYZE (실제)
├─ 쿼리를 실제로 실행함
├─ 실제 수행한 데이터 표시
├─ 느림 (쿼리만큼 걸림)
└─ Execution Time: 5000ms (실제)

사용 시점:
├─ EXPLAIN: 빠른 진단용 (개발 중)
└─ EXPLAIN ANALYZE: 정확한 진단 (느린 쿼리 발견 시)

주의:
├─ ANALYZE는 데이터 변경 쿼리면 실제로 실행됨
├─ DELETE 분석 전 트랜잭션 주의
└─ 프로덕션에서는 신중히 사용
```

</details>

---

<div align="center">

**[⬅️ 이전: HikariCP 튜닝 — Pool 크기 공식과 연결 비용](./01-hikaricp-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: 커넥션 모니터링 — 고갈 시 에러 패턴 분석 ➡️](./03-connection-monitoring.md)**

</div>
