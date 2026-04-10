# 04. 캐시 전략과 효과 측정 — Redis 도입 전후 비교

---

## 🎯 핵심 질문

- Cache-Aside vs Write-Through vs Write-Behind의 장단점은 무엇인가?
- Redis를 도입하면 정말 성능이 개선될까?
- Cache Hit Rate는 어떻게 측정하고, 캐시 무효화 버그를 찾을 수 있나?

---

## 🔍 왜 이 개념이 실무에서 중요한가

데이터베이스는 느립니다. (밀리초 단위)
- 쿼리: 10~50ms
- 네트워크 왕복: 5~10ms
- 총: 15~60ms per request

캐시는 매우 빠릅니다.
- Redis 조회: 0.1~1ms
- 메모리 접근: 0.001ms

**차이: 100~1000배!**

대부분의 성능 개선은 캐시에서 나옵니다.

---

## 😱 흔한 실수 (Before)

### 실수 1: 캐시 전략 없이 도입

```java
// 😱 무분별한 캐시
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public User getUser(Long id) {
        // 1️⃣ 캐시에서 먼저 찾기
        String cached = redisTemplate.opsForValue().get("user:" + id);
        if (cached != null) {
            return objectMapper.readValue(cached, User.class);
        }
        
        // 2️⃣ 없으면 DB에서 조회
        User user = userRepository.findById(id).orElse(null);
        
        // 3️⃣ 캐시에 저장 (But 언제까지? 얼마나? 무효화는?)
        if (user != null) {
            redisTemplate.opsForValue().set("user:" + id, 
                objectMapper.writeValueAsString(user));  // 😱 TTL 없음!
        }
        
        return user;
    }
    
    public void updateUser(Long id, UpdateUserRequest req) {
        userRepository.save(new User(id, req.getName()));
        // 😱 캐시 무효화 빠짐!
        // → Stale data 발생 (구 데이터 계속 반환)
    }
}
```

**결과:**
- 데이터 불일치 (사용자: "이름 변경했는데 안 바뀌었네?")
- Cache Stampede (TTL 없어서 영구 저장)

### 실수 2: Hit Rate 측정 없음

```java
// 😱 캐시를 썓는지 안 썼는지 모름
@Service
public class ProductService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public Product getProduct(Long id) {
        String cached = redisTemplate.opsForValue().get("product:" + id);
        if (cached != null) {
            return objectMapper.readValue(cached, Product.class);
        }
        
        // DB 조회 (실제로 이게 얼마나 자주 일어나는지 모름!)
        Product product = productRepository.findById(id).orElse(null);
        redisTemplate.opsForValue().set("product:" + id, 
            objectMapper.writeValueAsString(product), 
            Duration.ofHours(1));
        
        return product;
    }
}

// → 개발자: "캐시 도입했는데 성능 안 바뀌네?"
//   이유: Hit rate 10% (거의 cache miss!)
```

### 실수 3: 캐시 워밍업 없음

```
배포 후 첫 요청들:
1️⃣ user:1 요청 → cache miss → DB 조회 (50ms)
2️⃣ user:2 요청 → cache miss → DB 조회 (50ms)
3️⃣ user:3 요청 → cache miss → DB 조회 (50ms)
...
1000️⃣ user:1000 요청 → cache miss → DB 조회 (50ms)

처음 1000 요청의 응답시간:
├─ 예상: 1ms (캐시)
├─ 실제: 50ms (DB)
└─ 사용자: "배포 후 느려졌네?"
```

---

## ✨ 올바른 접근 (After)

### 접근 1: 캐시 전략 선택

```java
// 1️⃣ Cache-Aside (가장 많이 사용)
@Service
public class UserServiceCacheAside {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    public User getUser(Long id) {
        String key = "user:" + id;
        
        // 1단계: 캐시에서 조회
        User user = (User) redisTemplate.opsForValue().get(key);
        if (user != null) {
            incrementCacheHit();  // 메트릭
            return user;
        }
        
        incrementCacheMiss();  // 메트릭
        
        // 2단계: Cache miss → DB에서 조회
        user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User not found"));
        
        // 3단계: 캐시에 저장 (TTL 명시!)
        redisTemplate.opsForValue().set(key, user, 
            Duration.ofHours(1));  // ✅ 1시간 TTL
        
        return user;
    }
    
    public void updateUser(Long id, UpdateUserRequest req) {
        // 1단계: DB 업데이트
        User updated = userRepository.save(
            new User(id, req.getName()));
        
        // 2단계: 캐시 무효화 ✅ (매우 중요!)
        redisTemplate.delete("user:" + id);
        
        return updated;
    }
}

// 2️⃣ Write-Through (쓰기가 중요한 경우)
@Service
public class UserServiceWriteThrough {
    
    public void updateUser(Long id, UpdateUserRequest req) {
        // 1단계: 캐시 업데이트
        User updated = new User(id, req.getName());
        redisTemplate.opsForValue().set("user:" + id, updated);
        
        // 2단계: DB 업데이트
        userRepository.save(updated);
        
        // → 항상 캐시와 DB 동기화됨 ✅
        // → 쓰기가 느림 (두 번 쓰기)
    }
}

// 3️⃣ Write-Behind (쓰기 성능이 중요한 경우)
@Service
public class UserServiceWriteBehind {
    
    @Autowired
    private MessageQueue messageQueue;
    
    public void updateUser(Long id, UpdateUserRequest req) {
        // 1단계: 캐시만 업데이트 (빠름!)
        User updated = new User(id, req.getName());
        redisTemplate.opsForValue().set("user:" + id, updated);
        
        // 2단계: 비동기로 DB 업데이트 (나중에)
        messageQueue.send(new UpdateEvent(id, updated));
        
        // → 사용자 응답 빠름 ✅
        // → DB와 캐시 불일치 가능성 ⚠️
        // → 메시지 손실 처리 필요
    }
}
```

### 접근 2: Hit Rate 측정 및 모니터링

```java
@Component
public class CacheMetrics {
    
    private final AtomicLong cacheHits = new AtomicLong(0);
    private final AtomicLong cacheMisses = new AtomicLong(0);
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @PostConstruct
    public void registerMetrics() {
        Gauge.builder("cache.hit.count", cacheHits, AtomicLong::get)
            .description("Total cache hits")
            .register(meterRegistry);
            
        Gauge.builder("cache.miss.count", cacheMisses, AtomicLong::get)
            .description("Total cache misses")
            .register(meterRegistry);
            
        Gauge.builder("cache.hit.ratio", this::getHitRatio)
            .description("Cache hit ratio")
            .register(meterRegistry);
    }
    
    public void recordHit() {
        cacheHits.incrementAndGet();
    }
    
    public void recordMiss() {
        cacheMisses.incrementAndGet();
    }
    
    public double getHitRatio() {
        long total = cacheHits.get() + cacheMisses.get();
        if (total == 0) return 0.0;
        return (double) cacheHits.get() / total * 100;
    }
    
    @GetMapping("/cache/metrics")
    public ResponseEntity<?> getCacheMetrics() {
        long hits = cacheHits.get();
        long misses = cacheMisses.get();
        long total = hits + misses;
        
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("hits", hits);
        metrics.put("misses", misses);
        metrics.put("total", total);
        metrics.put("hitRatio", String.format("%.2f%%", getHitRatio()));
        
        // 목표: 80% 이상
        String status = getHitRatio() > 80 ? "GOOD" : "POOR";
        metrics.put("status", status);
        
        return ResponseEntity.ok(metrics);
    }
}

// 사용 예
@Service
public class UserService {
    
    @Autowired
    private CacheMetrics cacheMetrics;
    
    public User getUser(Long id) {
        String key = "user:" + id;
        User user = (User) redisTemplate.opsForValue().get(key);
        
        if (user != null) {
            cacheMetrics.recordHit();  // ✅ 메트릭 기록
            return user;
        }
        
        cacheMetrics.recordMiss();  // ✅ 메트릭 기록
        user = userRepository.findById(id).orElse(null);
        redisTemplate.opsForValue().set(key, user, Duration.ofHours(1));
        
        return user;
    }
}
```

### 접근 3: 캐시 워밍업 (Warm-up)

```java
@Component
public class CacheWarmupService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    // 애플리케이션 시작 시 실행
    @EventListener(ApplicationReadyEvent.class)
    public void warmupCache() {
        log.info("Starting cache warm-up...");
        
        long startTime = System.currentTimeMillis();
        
        // 1️⃣ 자주 사용되는 사용자 선별 (예: 활성 사용자)
        List<User> activeUsers = userRepository.findTopActive(1000);
        
        // 2️⃣ 캐시에 미리 저장
        activeUsers.forEach(user -> {
            String key = "user:" + user.getId();
            redisTemplate.opsForValue().set(key, user, Duration.ofHours(1));
        });
        
        long duration = System.currentTimeMillis() - startTime;
        log.info("Cache warm-up completed: {} users in {}ms", 
            activeUsers.size(), duration);
    }
    
    // 정기적 재워밍업 (매일 자정)
    @Scheduled(cron = "0 0 0 * * *")
    public void regularWarmup() {
        this.warmupCache();
    }
}

// Repository에 자주 사용되는 데이터 조회 추가
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 활성 사용자 (최근 7일 접속)
    @Query("SELECT u FROM User u WHERE u.lastLogin >= DATE_SUB(NOW(), INTERVAL 7 DAY) " +
           "ORDER BY u.id LIMIT 1000")
    List<User> findTopActive(int limit);
    
    // 인기 상품 (최근 1주일 구매 많은 순)
    @Query("SELECT p FROM Product p " +
           "WHERE EXISTS (SELECT 1 FROM Order o WHERE o.product = p " +
           "  AND o.createdAt >= DATE_SUB(NOW(), INTERVAL 7 DAY)) " +
           "ORDER BY p.id LIMIT 100")
    List<Product> findPopularProducts(int limit);
}
```

---

## 🔬 내부 동작 원리

### 캐시 전략의 내부 흐름

```
[Cache-Aside (Lazy Loading)]
요청 들어옴
    ↓
캐시 조회 (Redis)
├─ Hit (0.5ms)
│  └─ 값 반환 ✅
└─ Miss
   ├─ DB 조회 (50ms)
   └─ 캐시 저장 (TTL: 1시간)

장점: 구현 간단, 불필요한 캐시 보관 안 함
단점: 첫 요청은 느림 (Cache miss)

[Write-Through]
업데이트 요청
    ↓
캐시 업데이트 (1ms)
    ↓
DB 업데이트 (10ms)
    ↓
완료 응답 (11ms)

장점: 캐시와 DB 항상 동기화
단점: 쓰기가 두 배 느림

[Write-Behind]
업데이트 요청
    ↓
캐시 업데이트 (1ms)
    ↓
메시지 큐 전송 (0.5ms)
    ↓
사용자에게 응답 (1.5ms) ✅ 빠름!
    ↓
[비동기 처리]
메시지 큐에서 DB 업데이트 (10ms)

장점: 응답 매우 빠름
단점: 캐시-DB 불일치 가능, 복잡한 구현
```

### Hit Rate와 DB 부하의 관계

```
Hit Rate에 따른 DB 영향도:

Hit Rate = 100%
├─ DB 접근: 0회
├─ DB 부하: 0%
└─ 응답시간: 0.5ms (캐시만)

Hit Rate = 80% (🎯 목표)
├─ DB 접근: 20% (1000 중 200회)
├─ DB 부하: 20%
└─ 응답시간: p99 = 15ms

Hit Rate = 50%
├─ DB 접근: 50%
├─ DB 부하: 50%
└─ 응답시간: p99 = 35ms

Hit Rate = 0% (캐시 무용지물)
├─ DB 접근: 100%
├─ DB 부하: 100%
└─ 응답시간: p99 = 50ms
```

### Cache Stampede 문제

```
많은 연결이 TTL 만료 시점을 기다렸다가 동시에 요청:

[Before TTL 만료]
캐시: key=user:1, value={"name":"Alice"}, TTL=10s

[T=10s 정확히 (TTL 만료)]
동시 요청 1000개가 들어옴
    ↓
모두 캐시 miss 감지
    ↓
모두 DB 조회 시작 (동시에 1000개!)
    ↓
DB 연결 풀 고갈 (100개 pool → 부족)
    ↓
응답 지연 또는 에러 💥

[해결 방법]
1️⃣ TTL 변동 (±20%)
   └─ 만료 시점을 분산
   
2️⃣ Probabilistic Early Recomputation (PER)
   └─ TTL 70% 지났을 때 재로드 시작
   
3️⃣ Lock 기반 로딩
   ├─ 첫 요청: Lock 획득 → DB 로드
   └─ 다른 요청: Lock 대기 (DB 요청 안 함)
```

---

## 💻 실전 실험

### 1단계: Redis 설정 및 메트릭

```java
// Redis 연결 설정
@Configuration
public class RedisConfig {
    
    @Bean
    public LettuceConnectionFactory lettuceConnectionFactory() {
        return new LettuceConnectionFactory();
    }
    
    @Bean
    public RedisTemplate<String, User> redisTemplate(
            LettuceConnectionFactory connectionFactory) {
        RedisTemplate<String, User> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        Jackson2JsonRedisSerializer<User> jackson = 
            new Jackson2JsonRedisSerializer<>(User.class);
        template.setValueSerializer(jackson);
        
        return template;
    }
}

// application.yml
spring:
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
    database: 0
    jedis:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
```

### 2단계: p6spy로 DB 쿼리 수 측정

```bash
# p6spy 로그에서 쿼리 수 세기
grep -c "SELECT" logs/p6spy.log  # 캐시 전: 1000 쿼리
# 캐시 도입 후: 200 쿼리 (80% 감소!)
```

```java
// 자동화된 쿼리 카운팅
@Component
public class QueryCounter {
    
    private AtomicInteger queryCount = new AtomicInteger(0);
    
    @Aspect
    @Component
    public static class QueryCountingAspect {
        
        @Pointcut("execution(* org.springframework.data.repository.Repository+.*(..))")
        public void repositoryMethods() {}
        
        @Before("repositoryMethods()")
        public void countQuery(JoinPoint joinPoint) {
            queryCount.incrementAndGet();
        }
    }
    
    public int getQueryCount() {
        return queryCount.get();
    }
    
    public void reset() {
        queryCount.set(0);
    }
}
```

### 3단계: k6로 응답시간 비교

```javascript
// load-test-no-cache.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
  ],
};

export default function () {
  const response = http.get('http://localhost:8080/api/users/1');
  check(response, {
    'status 200': (r) => r.status === 200,
  });
  sleep(1);
}
```

```javascript
// load-test-with-cache.js
// (동일하지만 캐시 활성화 상태)
```

```bash
# 실행 및 비교
# 1️⃣ 캐시 없음
k6 run load-test-no-cache.js 2>&1 | grep "http_req_duration"
# http_req_duration: avg=48.2ms, p99=125.3ms

# 2️⃣ 캐시 있음
k6 run load-test-with-cache.js 2>&1 | grep "http_req_duration"
# http_req_duration: avg=1.5ms, p99=8.2ms

# 개선: 약 30배 빠름!
```

### 4단계: Hit Rate 모니터링

```java
@RestController
public class CacheStatsController {
    
    @Autowired
    private CacheMetrics cacheMetrics;
    
    @GetMapping("/stats/cache/before-after")
    public ResponseEntity<?> compareCacheMetrics() {
        long beforeWithoutCache = 1000;  // 쿼리 수 (캐시 전)
        long afterWithCache = 200;       // 쿼리 수 (캐시 후)
        
        Map<String, Object> comparison = new HashMap<>();
        comparison.put("queries_before", beforeWithoutCache);
        comparison.put("queries_after", afterWithCache);
        comparison.put("reduction", 
            String.format("%.1f%%", 
                (1 - (double)afterWithCache/beforeWithoutCache) * 100));
        
        comparison.put("hit_ratio_current", 
            String.format("%.2f%%", cacheMetrics.getHitRatio()));
        
        return ResponseEntity.ok(comparison);
    }
}
```

### 5단계: 캐시 무효화 버그 테스트

```java
@SpringBootTest
class CacheInvalidationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    @Test
    void testCacheInvalidationOnUpdate() {
        // 1️⃣ 사용자 조회 (캐시에 저장됨)
        User user = userService.getUser(1L);
        assertEquals("Alice", user.getName());
        
        // 2️⃣ 캐시에 있는지 확인
        User cached = (User) redisTemplate.opsForValue().get("user:1");
        assertNotNull(cached);
        assertEquals("Alice", cached.getName());
        
        // 3️⃣ 사용자 업데이트
        userService.updateUser(1L, new UpdateUserRequest("Bob"));
        
        // 4️⃣ 캐시가 무효화되었는지 확인 ✅
        User cachedAfterUpdate = 
            (User) redisTemplate.opsForValue().get("user:1");
        assertNull(cachedAfterUpdate, "캐시가 무효화되지 않았다!");
        
        // 5️⃣ 다시 조회하면 새 데이터
        User updatedUser = userService.getUser(1L);
        assertEquals("Bob", updatedUser.getName());
    }
    
    @Test
    void testStaleDataDetection() {
        // Stale data: 캐시는 "Alice"지만 DB는 "Bob"
        
        User user = userService.getUser(1L);
        assertEquals("Alice", user.getName());
        
        // 😱 캐시 무효화 없이 DB만 업데이트 (버그!)
        userRepository.save(new User(1L, "Bob"));
        // userService.updateUser() 호출 안 함
        
        // 캐시에서 조회 (구 데이터 반환!) ❌
        User stale = userService.getUser(1L);
        assertEquals("Alice", stale.getName());  // 버그!
        
        // 실제 DB는 Bob
        User actual = userRepository.findById(1L).get();
        assertEquals("Bob", actual.getName());
        
        // 일치하지 않음 → 데이터 불일치!
    }
}
```

---

## 📊 성능 비교

### Redis 도입 전후 성능

| 지표 | 캐시 없음 | 캐시 있음 | 개선도 |
|------|---------|---------|--------|
| 평균 응답시간 | 48.2ms | 1.5ms | **32배** |
| p99 응답시간 | 125.3ms | 8.2ms | **15배** |
| DB 쿼리 수 | 1000/s | 200/s | **80% 감소** |
| DB CPU | 85% | 15% | **↓70%** |
| 동시 사용자 | 50명 | 500명 | **10배** |

### 캐시 전략별 성능

```
시나리오: 1000명 동시 요청, 100명 데이터 업데이트

[Cache-Aside]
읽기: 900/s × 0.5ms = 450ms 총 시간
쓰기: 100/s × 50ms = 5,000ms 총 시간 (DB 부하)
평균 응답: 15ms

[Write-Through]
읽기: 900/s × 0.5ms = 450ms (캐시 빠름)
쓰기: 100/s × 11ms (캐시 + DB) = 1,100ms
평균 응답: 2.5ms (일관성 보장)

[Write-Behind]
읽기: 900/s × 0.5ms = 450ms
쓰기: 100/s × 1.5ms = 150ms (캐시만) ⭐ 가장 빠름
평균 응답: 1.2ms
(단, DB 반영 지연 수 초)
```

### Hit Rate와 DB 부하의 관계

```
Load: 1000 requests/s

Hit Rate 100% ✅
├─ 캐시: 1000/s
├─ DB: 0/s
├─ DB CPU: 0%
└─ p99: 0.8ms

Hit Rate 80% 🎯 목표
├─ 캐시: 800/s
├─ DB: 200/s
├─ DB CPU: 20%
└─ p99: 8.2ms

Hit Rate 50%
├─ 캐시: 500/s
├─ DB: 500/s
├─ DB CPU: 50%
└─ p99: 35ms

Hit Rate 0% ❌ 캐시 무용지물
├─ 캐시: 0/s
├─ DB: 1000/s
├─ DB CPU: 100%
└─ p99: 125ms
```

---

## ⚖️ 트레이드오프

### Redis 메모리 비용 vs 성능 개선

**장점:**
- 응답시간 30~50배 개선
- DB 부하 80% 감소
- 동시 사용자 10배 수용 가능

**단점:**
- Redis 메모리 비용 (예: 16GB ≈ $200/월)
- 캐시 관리 복잡도 (무효화, 워밍업)
- 데이터 일관성 처리 필요

### Cache-Aside vs Write-Through 선택

**Cache-Aside:**
- 장점: 구현 간단, 캐시 선택적 사용
- 단점: 첫 요청 느림, 캐시 무효화 필요

**Write-Through:**
- 장점: 데이터 일관성 보장, 구현 명확
- 단점: 쓰기 느림, 캐시 항상 유지

---

## 📌 핵심 정리

### 캐시 도입 체크리스트

```
1️⃣ 전략 선택
   ├─ 읽기 위주 → Cache-Aside ✅
   ├─ 쓰기 중요 → Write-Through ✅
   └─ 성능 최우선 → Write-Behind ✅

2️⃣ 구현
   ├─ TTL 설정 (기본: 1시간)
   ├─ 무효화 로직 추가
   ├─ 워밍업 구현
   └─ Hit Rate 모니터링

3️⃣ 모니터링
   ├─ Hit Rate > 80% 목표
   ├─ 응답시간 p99 < 10ms
   ├─ DB CPU < 20%
   └─ 캐시 메모리 < 전체 RAM 20%

4️⃣ 데이터 일관성
   ├─ 캐시 무효화 테스트
   ├─ Stale data 감지
   └─ Cache Stampede 예방
```

### 측정 항목

```
Before (캐시 없음):
├─ 평균 응답: 50ms
├─ p99: 125ms
├─ 쿼리/초: 1000
├─ DB CPU: 85%
└─ 최대 동시: 50명

After (캐시 있음):
├─ 평균 응답: 1.5ms ✅
├─ p99: 8.2ms ✅
├─ 쿼리/초: 200 ✅
├─ DB CPU: 15% ✅
└─ 최대 동시: 500명 ✅

개선율:
├─ 응답시간: 30배
├─ DB 부하: 80% 감소
└─ 용량: 10배
```

---

## 🤔 생각해볼 문제

### Q1: 캐시 Hit Rate가 50%인데 성능이 2배만 개선됐다. 왜?

<details>
<summary>💡 해설 보기</summary>

```
Hit Rate 50%이면:
├─ 캐시: 500/s (0.5ms)
└─ DB: 500/s (50ms)

평균 응답 시간:
= (500 × 0.5ms + 500 × 50ms) / 1000
= (250 + 25,000) / 1000
= 25.25ms

캐시 없음: 50ms
캐시 있음: 25.25ms
개선도: 약 2배 ✅

낮은 Hit Rate 원인:
1️⃣ 캐시 TTL이 너무 짧음
   └─ 기본값 추천: 1시간 이상

2️⃣ 캐시 키 설계 오류
   └─ 같은 데이터도 다른 키로 저장
   └─ 예: user:1, USER:1, user_1 (3개 캐시)

3️⃣ 워밍업 없음
   └─ 배포 후 처음엔 캐시 비어있음

4️⃣ 데이터 다양성 높음
   └─ 유니크한 ID가 많으면 Hit Rate 낮아짐
   └─ 예: 1000명 다른 사용자 접근 → 1000개 캐시 필요

해결:
├─ Hit Rate 측정 (목표: 80%)
├─ TTL 분석
├─ 키 설계 검토
└─ 워밍업 추가
```

</details>

### Q2: Redis가 다운되면 어떻게 할까?

<details>
<summary>💡 해설 보기</summary>

```java
// Cache Failover 패턴

@Service
public class UserServiceWithFailover {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    public User getUser(Long id) {
        String key = "user:" + id;
        
        try {
            // 1️⃣ Redis 시도
            User user = (User) redisTemplate.opsForValue().get(key);
            if (user != null) {
                return user;
            }
        } catch (Exception e) {
            // 2️⃣ Redis 다운 시 로그
            log.warn("Redis unavailable: {}", e.getMessage());
        }
        
        // 3️⃣ Fallback: DB 직접 조회
        return userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException());
    }
}

@Configuration
public class RedisHealthCheck {
    
    @Bean
    public HealthIndicator redisHealthIndicator(
            LettuceConnectionFactory connectionFactory) {
        return () -> {
            try {
                connectionFactory.getConnection().ping();
                return Health.up().build();
            } catch (Exception e) {
                return Health.down()
                    .withException(e)
                    .build();
            }
        };
    }
}

Failover 전략:
├─ L1: Redis (0.5ms)
├─ L2: Local Cache (5ms, 메모리)
└─ L3: Database (50ms, 마지막 수단)

선택:
├─ 가용성 중요 → Fallback to DB
├─ 성능 중요 → Local Cache + Redis
└─ 대규모 → Multi-Region Redis Cluster
```

</details>

### Q3: 캐시 워밍업 중에 메모리 부족 에러가 나면?

<details>
<summary>💡 해설 보기</summary>

```
워밍업 시나리오:
활성 사용자 100만 명
캐시 per user: 1KB
총 메모리: 1M × 1KB = 1GB

Redis 최대 메모리: 512MB
→ OOM (Out Of Memory) 에러!

해결 방법:

1️⃣ 선택적 워밍업
@EventListener(ApplicationReadyEvent.class)
public void warmupCache() {
    // 상위 10%만 캐시 (10만 명)
    List<User> topUsers = userRepository.findTopActive(100_000);
    topUsers.forEach(u -> cacheUser(u));
}

2️⃣ 배치 워밍업
public void warmupInBatches() {
    int batchSize = 10_000;
    for (int i = 0; i < total; i += batchSize) {
        List<User> batch = userRepository
            .findTopActive(i, batchSize);
        batch.forEach(u -> cacheUser(u));
        
        // 배치마다 메모리 확인
        Runtime.getRuntime().gc();
    }
}

3️⃣ Redis 메모리 설정
maxmemory: 512MB
maxmemory-policy: allkeys-lru
// LRU: 가장 오래 사용 안 한 캐시부터 제거

4️⃣ 캐시 크기 최적화
// 전체 사용자 정보 대신 필요한 필드만
{
    "id": 1,
    "name": "Alice"
    // email, phone 등 제거
}

메모리 계산:
┌─ 사용자 정보 전체: 2KB
└─ 필요 필드만: 0.5KB ✅ 4배 절약!
```

</details>

---

<div align="center">

**[⬅️ 이전: 커넥션 모니터링 — 고갈 시 에러 패턴 분석](./03-connection-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: 페이징과 대용량 조회 최적화 — OFFSET의 성능 저하 ➡️](./05-pagination-optimization.md)**

</div>
