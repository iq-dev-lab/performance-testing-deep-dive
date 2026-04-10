# 07. 네트워크 병목 분석 — 응답 크기와 직렬화 비용

---

## 🎯 핵심 질문

네트워크가 병목인지 어떻게 알 수 있을까?
- 응답 크기가 크면 → 전송 시간 증가
- 직렬화/역직렬화 시간이 길면 → CPU 낭비
- Keep-Alive 없으면 → 매 요청마다 TCP 연결 수립 (50~300ms)
- DNS 조회가 오래 걸리면 → 100~1000ms 낭비

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
상황: p99 응답시간 2000ms

검사:
├─ 애플리케이션 처리: 100ms (DB 쿼리 포함)
├─ 애플리케이션 메모리: 70% (정상)
├─ CPU: 60% (정상)
└─ → 나머지 1900ms는 어디로?

원인 분석:
1. 응답 바디 크기: 50MB (JSON)
   - 직렬화 시간: 500ms
   - 네트워크 전송: 1200ms (1Gbps 네트워크)
   - 클라이언트 역직렬화: 300ms
   = 2000ms (응답시간 대부분)

2. Keep-Alive 없음:
   - TCP Connection 수립: 100ms
   - TLS Handshake: 50ms (HTTPS)
   - 데이터 전송: 50ms
   = 200ms × 요청 10개 = 2000ms 손실

3. DNS 조회:
   - 첫 요청: DNS 조회 300ms
   - 응답 100ms
   = 400ms 낭비

결과:
사용자: "API 응답이 느리다" (실제로는 네트워크/직렬화)
```

**올바른 대응:**

```
네트워크 병목 해결:

1. 응답 크기 최소화
   - 불필요한 필드 제거
   - 압축 (gzip)
   - 페이지네이션
   50MB → 1MB (50배 감소)

2. 직렬화 최적화
   - JSON 대신 Protobuf/MessagePack
   - 직렬화 시간 500ms → 50ms (10배 감소)

3. Keep-Alive 활성화
   - TCP 재사용
   - 연결 수립 100ms → 0ms (재사용)

4. DNS 캐싱
   - 첫 조회 300ms → 1ms (캐시)

결과:
p99: 2000ms → 200ms (90% 개선!)
```

---

## 😱 흔한 실수 (Before)

```java
// 잘못된 접근 1: 응답 크기를 무시한다
@GetMapping("/users")
public List<User> getAllUsers() {
    // 모든 필드를 다 반환
    // SELECT id, name, email, phone, address, zip, country, ...
    List<User> users = userRepository.findAll();  // 10000개 사용자
    return users;  // 500MB 응답 (직렬화 후)
}

# 문제:
# - 응답 크기: 500MB
# - 직렬화 시간: 2초
# - 전송 시간: 5초 (1Mbps 네트워크)
# - 총 응답시간: 7초 (클라이언트 역직렬화 포함)

# 잘못된 접근 2: Keep-Alive 없이 반복 요청
RestTemplate restTemplate = new RestTemplate();

for (int i = 0; i < 100; i++) {
    // 매 요청마다 TCP 연결 수립 (100ms)
    ResponseEntity<Order> response = 
        restTemplate.getForEntity("https://api.example.com/order/" + i, Order.class);
    // TCP 연결 수립 100ms × 100 = 10초 낭비
}

# 해결책: Connection Pooling 사용
PoolingHttpClientConnectionManager connectionManager = 
    new PoolingHttpClientConnectionManager();
connectionManager.setMaxTotal(100);  // Keep-Alive

# 잘못된 접근 3: 압축 설정 안 함
@GetMapping("/api/data")
public ResponseEntity<LargeObject> getData() {
    LargeObject data = new LargeObject();  // 10MB
    return ResponseEntity.ok(data);  // 압축 없음
}

# 문제: 10MB 데이터를 그대로 전송
# 해결책: gzip 압축 (10MB → 1MB, 90% 감소)

# 잘못된 접근 4: DNS를 매번 조회
for (int i = 0; i < 1000; i++) {
    URL url = new URL("https://api.example.com/data");
    // 매 요청마다 DNS 조회 (300ms)
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
    // DNS 조회 300ms × 1000 = 300초 낭비!
}

# 문제: DNS resolver가 매번 조회
# 해결책: DNS 캐싱 또는 Connection Pooling
```

---

## ✨ 올바른 접근 (After)

### 1단계: 응답 크기 측정 및 최적화

**응답 크기 측정:**

```bash
# 1. 네트워크 요청 크기 확인 (curl)
$ curl -w "Size: %{size_download} bytes\n" \
    https://api.example.com/users | head -1
# Size: 52428800 bytes (50MB)

# 2. 압축 전후 비교
$ curl -H "Accept-Encoding: gzip" \
    -w "Compressed: %{size_download} bytes\n" \
    https://api.example.com/users
# Compressed: 5242880 bytes (5MB, 90% 감소)

# 3. Chrome DevTools에서 확인
# Network tab → Response tab → Size 확인
# 예: 5.2 MB / 1.2 MB (압축됨)

# 4. 애플리케이션 메트릭
$ curl http://localhost:8080/actuator/metrics/http.server.requests | \
    jq '.measurements[] | select(.statistic == "MEAN")'
# 평균 응답 크기 확인
```

**응답 크기 최적화:**

```java
// 문제 코드: 모든 필드 반환
@GetMapping("/users")
public List<User> getAllUsers() {
    return userRepository.findAll();  // 50MB
}

// 해결책 1: 필요한 필드만 선택 (DTO)
@GetMapping("/users")
public List<UserDTO> getAllUsers() {
    return userRepository.findAll()
        .stream()
        .map(user -> new UserDTO(
            user.getId(),
            user.getName(),
            user.getEmail()
            // 필요한 필드만
        ))
        .collect(Collectors.toList());
    // 5MB (10배 감소)
}

// 해결책 2: 페이지네이션
@GetMapping("/users")
public Page<UserDTO> getAllUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) {
    return userRepository.findAll(PageRequest.of(page, size))
        .map(user -> new UserDTO(user.getId(), user.getName(), user.getEmail()));
    // 20개 × 1KB = 20KB (2500배 감소!)
}

// 해결책 3: 압축 활성화 (Spring Boot)
# application.yml
server:
  compression:
    enabled: true
    min-response-size: 1024  # 1KB 이상만 압축
    mime-types:
      - application/json
      - application/xml
      - text/html

# 결과: 5MB → 500KB (10배 감소)

// 해결책 4: 캐싱 헤더
@GetMapping("/users")
public ResponseEntity<List<UserDTO>> getAllUsers() {
    HttpHeaders headers = new HttpHeaders();
    headers.setCacheControl("max-age=3600");  // 1시간 캐시
    
    List<UserDTO> users = ...;
    return new ResponseEntity<>(users, headers, HttpStatus.OK);
    
    // 클라이언트: 반복된 요청에서 캐시 사용 (네트워크 요청 0)
}
```

### 2단계: 직렬화/역직렬화 최적화

**직렬화 성능 비교:**

```bash
# 벤치마크: 100,000개 객체 직렬화

JSON (Jackson):
- 크기: 10MB
- 직렬화 시간: 500ms
- 역직렬화 시간: 600ms
- 총 시간: 1100ms

Protobuf:
- 크기: 2MB (80% 감소)
- 직렬화 시간: 50ms
- 역직렬화 시간: 60ms
- 총 시간: 110ms (10배 빠름!)

MessagePack:
- 크기: 3MB (70% 감소)
- 직렬화 시간: 100ms
- 역직렬화 시간: 120ms
- 총 시간: 220ms (5배 빠름)
```

**직렬화 최적화 구현:**

```java
// 1. Jackson 최적화 (JSON 사용할 때)
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // 직렬화 설정
        mapper.setSerializationInclusion(Include.NON_NULL);
        // null 값 제외 (크기 감소)
        
        mapper.configure(
            SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, 
            false
        );
        // 날짜를 timestamp 대신 ISO 형식
        
        // 역직렬화 설정
        mapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, 
            false
        );
        // 미알려진 필드 무시
        
        return mapper;
    }
}

// 2. Protobuf 사용 (최고 성능)
// build.gradle
dependencies {
    implementation 'com.google.protobuf:protobuf-java:3.21.0'
}

// user.proto
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

// Java 코드
@GetMapping("/users", produces = "application/octet-stream")
public ResponseEntity<byte[]> getAllUsers() {
    User.Builder builder = User.newBuilder()
        .setId(1)
        .setName("John")
        .setEmail("john@example.com");
    
    byte[] serialized = builder.build().toByteArray();
    // 크기: JSON 1KB → Protobuf 100 bytes
    
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(serialized);
}

// 3. 커스텀 직렬화 (응급 상황)
public class FastUserSerializer {
    
    public static byte[] serialize(User user) {
        ByteBuffer buffer = ByteBuffer.allocate(100);
        buffer.putLong(user.getId());
        buffer.putInt(user.getName().length());
        buffer.put(user.getName().getBytes());
        // ...
        return buffer.array();
    }
    
    public static User deserialize(byte[] data) {
        ByteBuffer buffer = ByteBuffer.wrap(data);
        long id = buffer.getLong();
        int nameLen = buffer.getInt();
        byte[] nameBytes = new byte[nameLen];
        buffer.get(nameBytes);
        // ...
        return new User(id, new String(nameBytes));
    }
}
```

### 3단계: Keep-Alive 및 Connection Pooling

**RestTemplate 설정:**

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        // Connection Manager 설정
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
        
        // 전체 최대 연결 수
        connectionManager.setMaxTotal(100);
        
        // 호스트당 최대 연결 수
        connectionManager.setDefaultMaxPerRoute(20);
        
        // 유휴 연결 유지 시간
        // Keep-Alive 헤더가 없으면 자동 닫음
        connectionManager.setDefaultConnectionConfig(
            ConnectionConfig.custom()
                .setSocketTimeout(5000)  // 5초
                .setConnectTimeout(3000)  // 3초
                .build()
        );
        
        HttpClientBuilder builder = HttpClients.custom()
            .setConnectionManager(connectionManager)
            // Keep-Alive 재사용 (기본 활성화)
            .setDefaultHeaders(
                Arrays.asList(
                    new BasicHeader(
                        HttpHeaders.CONNECTION, 
                        "keep-alive"
                    )
                )
            );
        
        HttpClient httpClient = builder.build();
        
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory(httpClient);
        
        return new RestTemplate(factory);
    }
}

// 사용
@Service
public class ApiService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void callAPI() {
        // 첫 요청: TCP 연결 + TLS 수립 (100ms)
        ResponseEntity<Data> response1 = 
            restTemplate.getForEntity("https://api.example.com/data", Data.class);
        
        // 두 번째 요청: Keep-Alive로 연결 재사용 (0ms)
        ResponseEntity<Data> response2 = 
            restTemplate.getForEntity("https://api.example.com/data", Data.class);
        
        // 세 번째 요청: Keep-Alive 재사용 (0ms)
        ResponseEntity<Data> response3 = 
            restTemplate.getForEntity("https://api.example.com/data", Data.class);
    }
}
```

**WebClient 설정 (권장, Spring WebFlux):**

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            // Connection Pooling
            .protocol(HttpProtocol.HTTP11)  // HTTP/1.1 Keep-Alive
            
            // 타임아웃
            .responseTimeout(Duration.ofSeconds(30))
            .connectTimeout(Duration.ofSeconds(10))
            
            // Keep-Alive 설정
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true);
        
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}

// 사용 (비동기, 더 효율적)
@Service
public class ApiService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<Data> callAPI() {
        return webClient.get()
            .uri("https://api.example.com/data")
            .retrieve()
            .bodyToMono(Data.class)
            .timeout(Duration.ofSeconds(5));
    }
}
```

### 4단계: DNS 최적화

**DNS 조회 시간 측정:**

```bash
# 1. DNS 조회 시간 측정
$ nslookup api.example.com
# Query time: 45 msec

# 2. 반복 조회 (캐시 있을 때)
$ nslookup api.example.com
# Query time: 1 msec (45배 빠름)

# 3. dig로 상세 분석
$ dig api.example.com
# Query time: 45 msec
# Server: 8.8.8.8#53(8.8.8.8)

# 4. Java 애플리케이션 DNS 캐싱
$ java -Dnetworkaddress.cache.ttl=300 ...
# 300초 동안 DNS 결과 캐싱
```

**DNS 최적화:**

```java
// 1. 시스템 레벨 DNS 캐싱
// java -Dnetworkaddress.cache.ttl=300 -jar app.jar
// JVM 시작 옵션에서 설정

// 2. Connection URL을 IP 주소로 사용 (DNS 조회 회피)
// Before: https://api.example.com
// After: https://10.0.0.1 (또는 ELB/ALB 사용)

// 3. DNS 클라이언트 라이브러리 사용
@Configuration
public class DnsConfig {
    
    @Bean
    public DnsResolver dnsResolver() {
        // dnsjava 라이브러리 사용
        Resolver resolver = new ExtendedResolver();
        resolver.setTimeout(Duration.ofMillis(100));
        
        return new CachingDnsResolver(resolver, 300);
        // 300초 캐싱
    }
}

// 4. 로컬 /etc/hosts 설정 (개발/테스트)
# /etc/hosts
10.0.0.100 api.example.com
10.0.0.101 db.example.com
```

---

## 🔬 내부 동작 원리

### TCP 연결 수립 과정

```
TCP Three-Way Handshake:

Client                          Server
  │                              │
  ├─ SYN (seq=1000)             │
  │──────────────────────────→  │
  │                              │
  │                    SYN+ACK (seq=2000, ack=1001)
  │  ←──────────────────────────┤
  │                              │
  ├─ ACK (seq=1001, ack=2001)  │
  │──────────────────────────→  │
  │                              │
  └─ Connection Established ────┘

총 시간: 왕복 시간 (RTT) × 1.5
예: RTT 50ms → TCP 수립 75ms

HTTPS의 경우 (TLS Handshake 추가):
TCP: 75ms
TLS ClientHello: 50ms
TLS ServerHello: 50ms
TLS Certificate: 50ms
= 총 225ms (매 요청마다)

Keep-Alive:
첫 요청: 225ms
두 번째: 0ms (연결 재사용)
세 번째: 0ms
...

Keep-Alive 없음:
모든 요청: 225ms
10개 요청: 2250ms (응답시간의 대부분!)
```

### 직렬화의 CPU 비용

```
JSON 직렬화 과정:

객체 (메모리)
  │
  ├─ toString() 호출 (재귀)
  ├─ Date 포맷팅 (SimpleDateFormat)
  ├─ Escape 문자 처리 (", \, 등)
  ├─ UTF-8 인코딩
  └─ 바이트 배열 생성

총 CPU 사이클: 매우 많음
예: 100,000개 객체 × 5ms = 500ms

CPU 낭비:
- 복잡한 객체 → 직렬화 시간 증가
- 큰 컬렉션 → 반복 오버헤드
- 깊은 중첩 → 재귀 비용

Protobuf의 경우:
- 미리 정의된 형식 (직렬화 빠름)
- 바이너리 형식 (Escape 불필요)
- 최적화된 라이브러리
→ JSON의 10배 빠름
```

---

## 💻 실전 실험

**시나리오: 응답시간 1000ms인데 네트워크가 원인인 경우**

```bash
#!/bin/bash
# network-diagnosis.sh

echo "=== 네트워크 병목 진단 ==="
echo

# 1. 응답 크기 측정
echo "1. 응답 크기"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -w "Size: %{size_download} bytes\n" \
    http://localhost:8080/api/users 2>/dev/null | tail -1

# 압축 비교
curl -H "Accept-Encoding: gzip" \
    -w "Compressed: %{size_download} bytes\n" \
    http://localhost:8080/api/users 2>/dev/null | tail -1

# 2. 응답 시간 분해
echo
echo "2. 응답 시간 분해"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -w "
Total: %{time_total}s
Connect: %{time_connect}s
DNS: %{time_namelookup}s
TLS: %{time_appconnect}s
TTFB: %{time_starttransfer}s
Transfer: %(time_total)s
" http://localhost:8080/api/users 2>/dev/null | tail -6

# 3. 직렬화 시간 측정
echo
echo "3. 직렬화 시간 측정"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics | \
    grep -i "serializ" | head -10

# 4. Keep-Alive 확인
echo
echo "4. Keep-Alive 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
# 첫 요청
echo -n "First request: "
curl -w "%{time_total}s\n" -o /dev/null -s http://localhost:8080/api/users

# 두 번째 요청 (Keep-Alive 활용)
echo -n "Second request: "
curl -w "%{time_total}s\n" -o /dev/null -s http://localhost:8080/api/users

# 세 번째 요청
echo -n "Third request: "
curl -w "%{time_total}s\n" -o /dev/null -s http://localhost:8080/api/users

# 5. 네트워크 성능
echo
echo "5. 네트워크 성능"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
# 대역폭 측정
echo "Bandwidth test:"
curl -s http://localhost:8080/api/large-data \
    --progress | tail -20

# 6. k6로 부하 테스트 (network 지표)
echo
echo "6. k6 부하 테스트 (네트워크 지표)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
cat > /tmp/k6-test.js << 'EOF'
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 10,
  duration: '30s',
};

export default function () {
  let res = http.get('http://localhost:8080/api/users');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
}
EOF

# k6 실행 (설치 필요: brew install k6)
if command -v k6 &> /dev/null; then
  k6 run /tmp/k6-test.js
else
  echo "k6 not installed. Install with: brew install k6"
fi

echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== 네트워크 병목 진단 ===

1. 응답 크기
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Size: 52428800 bytes (50MB)
Compressed: 5242880 bytes (5MB, 90% 감소)

⚠️  WARNING: 50MB는 너무 크다!

2. 응답 시간 분해
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: 6.234s
Connect: 0.075s
DNS: 0.045s
TLS: 0.025s
TTFB: 0.500s (서버 처리)
Transfer: 5.589s (⚠️  대부분 전송 시간!)

분석:
- 서버 처리: 500ms (정상)
- 네트워크 전송: 5589ms (비정상!)

3. 직렬화 시간 측정
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
직렬화 시간: 1200ms (메트릭에서)
역직렬화 시간: 800ms (클라이언트)

⚠️  WARNING: 직렬화가 2초 소요!

4. Keep-Alive 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
First request: 6.234s
Second request: 6.123s (변화 없음, Keep-Alive 미활성)
Third request: 6.145s

⚠️  WARNING: Keep-Alive 미활성 (매번 새 연결)

5. 네트워크 성능
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
######################################################################## 100.0%
100 bytes, 1.2 MB/s 평균

네트워크 대역폭: 1.2 MB/s
필요 시간: 50MB / 1.2 MB/s = 41.7초 (하지만 6초?)
→ 실제 전송 시간: 5.589초 (대략 10MB/s?)

6. k6 부하 테스트 (네트워크 지표)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
iterations.....................: 300 / 300
duration........................: 30.123s
checks..........................: 100%
  http_req_duration............: avg=2000ms   p(95)=3500ms  p(99)=5000ms
  http_req_connecting...........: avg=75ms    p(95)=100ms   p(99)=150ms
  http_req_tls_handshaking.....: avg=25ms    p(95)=50ms    p(99)=75ms
  http_req_sending..............: avg=10ms    p(95)=20ms    p(99)=30ms
  http_req_waiting..............: avg=500ms   p(95)=800ms   p(99)=1200ms
  http_req_receiving............: avg=1400ms  p(95)=2500ms  p(99)=3500ms

분석:
- 연결: 75ms (정상)
- TLS: 25ms (정상)
- 송신: 10ms (정상)
- 서버 처리(대기): 500ms (정상)
- 수신(전송): 1400ms (⚠️  문제!)

=== 진단 결과 ===

🔍 주요 문제:

1. 응답 크기: 50MB (압축되면 5MB)
   - 영향: 전송 시간 5.6초
   - 개선: 필드 축소 + 페이지네이션 + 압축
   - 기댓값: 50MB → 100KB (500배 감소!)

2. 직렬화 비용: 1200ms
   - 영향: JSON 객체 → 바이트 배열
   - 개선: Protobuf 또는 MessagePack
   - 기댓값: 1200ms → 100ms (12배 개선)

3. Keep-Alive 미활성
   - 영향: 매 요청마다 TCP 연결 수립 (75ms)
   - 개선: Connection Pooling 활성화
   - 기댓값: 매 요청 75ms → 0ms 절감

4. DNS 캐싱 부족
   - 영향: 45ms (한 번)
   - 개선: TTL 300초로 설정
   - 기댓값: 반복 요청에서 45ms → 1ms

다음 단계 (우선순위):
1. 응답 크기 최소화 (필드 축소, 페이지네이션)
2. 압축 활성화 (gzip)
3. 직렬화 포맷 변경 (Protobuf)
4. Keep-Alive 활성화
5. DNS 캐싱
```

---

## 📊 성능 비교

| 개선 항목 | Before | After | 개선율 | 적용 난이도 |
|----------|--------|--------|--------|------------|
| **응답 크기** | 50MB | 100KB | 500배 | 쉬움 |
| **압축** | 50MB | 5MB | 10배 | 매우 쉬움 |
| **직렬화** | JSON 1200ms | Protobuf 100ms | 12배 | 중간 |
| **Keep-Alive** | 매 요청 75ms | 0ms | 75ms 절감 | 쉬움 |
| **DNS 캐싱** | 45ms | 1ms | 45배 | 쉬움 |
| **조합 효과** | 6000ms | 100ms | 60배 | - |

**실제 케이스:**

```
케이스 1: 큰 응답 크기 (50MB)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  직렬화: 1200ms
  전송: 5600ms (1.2 MB/s)
  역직렬화: 800ms
  p99: 8000ms
  
After: 필드 축소 + 압축 + Protobuf
  크기: 50MB → 100KB (압축 후)
  직렬화: 1200ms → 50ms (Protobuf)
  전송: 5600ms → 90ms (100KB)
  역직렬화: 800ms → 20ms
  p99: 160ms
  
개선율: p99 98% 개선

비용:
- 개발 시간: 4시간
- 투자 비용: 0 (코드 수정)
- 수익: 응답시간 50배 개선

케이스 2: Keep-Alive 미설정
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  API 호출 100회
  TCP 연결: 100회 × 75ms = 7500ms
  p99: 7500ms + 처리 시간
  
After: Connection Pooling
  API 호출 100회 (같은 호스트)
  TCP 연결: 1회 × 75ms = 75ms
  p99: 75ms + 처리 시간 (99% 개선)

케이스 3: 직렬화 최적화
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  JSON 직렬화: 1200ms (복잡한 객체)
  
After: Protobuf
  직렬화: 100ms
  
개선: 1100ms 절감
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 선택 기준 |
|------|------|------|---------|
| **응답 크기 축소** | 전송 시간 ↓, 비용 저감 | 필드 제한, UX 영향 | 항상 우선 |
| **압축** | 크기 90% 감소, 구현 쉬움 | CPU 오버헤드 5~10% | 항상 활성화 |
| **Protobuf** | 크기/성능 최고 | 호환성, 스키마 관리 | 성능 critical |
| **Keep-Alive** | TCP 연결 재사용, 지연 ↓ | 메모리 증가 (미미) | 항상 활성화 |
| **캐싱** | 반복 요청 최적화 | 캐시 무효화 관리 | 정적 데이터 대상 |

---

## 📌 핵심 정리

1. **응답 크기가 모든 병목의 근원**
   - 50MB → 100KB (500배 개선 가능)
   - 필드 축소, 페이지네이션으로 접근
   - 필드를 제거하면 직렬화도 빨라짐

2. **압축은 필수**
   - gzip: 구현 1줄, 효과 90% (50MB → 5MB)
   - CPU 오버헤드: 5~10% (무시할 수준)
   - 항상 활성화할 것

3. **직렬화 포맷 선택이 중요**
   - JSON: 가독성 좋음, 느림 (1200ms)
   - Protobuf: 빠름, 스키마 필요 (100ms)
   - 성능 critical이면 Protobuf

4. **Keep-Alive는 기본**
   - 한 번 설정하면 자동으로 효과 봄
   - TCP 연결 재사용 (75ms → 0ms)
   - 반복적인 API 호출에서 필수

5. **의사결정 순서**
   ```
   네트워크 병목 의심
   ├─ 응답 크기 확인
   │  ├─ > 1MB: 필드 축소, 페이지네이션 (우선!)
   │  └─ < 1MB: 압축 활성화
   ├─ 직렬화 시간
   │  ├─ > 500ms: Protobuf 전환
   │  └─ < 500ms: 유지
   ├─ Keep-Alive 확인
   │  ├─ 비활성: 활성화
   │  └─ 활성: 다음
   └─ DNS 캐싱 설정
   ```

---

## 🤔 생각해볼 문제

**Q1: 응답 크기 50MB를 10MB로 줄었는데도 응답시간 개선이 5배만 되었으면?**

<details>
<summary>해설 보기</summary>

**네트워크 전송이 유일한 병목이 아니라는 뜻이다.**

**시간 분해:**

Before:
- 직렬화: 1000ms
- 전송: 5000ms (50MB)
- 역직렬화: 600ms
- 총: 6600ms

After (응답 10MB):
- 직렬화: 200ms (데이터 5배 감소)
- 전송: 1000ms (데이터 5배 감소)
- 역직렬화: 120ms
- 총: 1320ms

개선: 6600 → 1320 = 5배 (예상대로)

만약 개선이 2배만 되었다면:

After (실제):
- 직렬화: 500ms (여전히 높음)
- 전송: 1000ms
- 역직렬화: 300ms
- 기타: 500ms (코드 처리?)
- 총: 2300ms

개선: 6600 → 2300 = 2.8배

**원인 분석:**
1. 직렬화가 여전히 오래 걸림 (데이터 축소보다 포맷 문제)
2. 코드에서 다른 오버헤드 발생
3. 캐시 효율 변화 (더 적은 데이터, 더 많은 쿼리?)

**해결책:**
1. 직렬화 포맷 변경 (JSON → Protobuf)
2. 코드 프로파일링 (CPU 사용률 확인)
3. 데이터 생성 로직 최적화
</details>

---

**Q2: 압축(gzip)을 활성화했는데 응답시간이 100ms 더 느려졌으면?**

<details>
<summary>해설 보기</summary>

**이것은 정상적인 현상일 수 있다.**

**타이밍 분석:**

Before (압축 없음):
- 서버 처리: 100ms
- 전송: 5000ms (50MB)
- 총: 5100ms

After (gzip 압축):
- 서버 처리 + 압축: 100 + 100 = 200ms (100ms 추가)
- 전송: 500ms (5MB로 감소)
- 총: 700ms

개선: 5100ms → 700ms (7배 개선!)

하지만 느껴지는 것:
- 첫 응답: 약간 느려짐 (압축 오버헤드 100ms)
- 전체: 빨라짐

**주의점:**

1. **CPU 호스트 리소스 체크**
   ```bash
   # Before: CPU 30%
   # After: CPU 35% (압축 오버헤드)
   ```

2. **네트워크 대역폭에 따라 달라짐**
   ```
   느린 네트워크 (1 Mbps):
   - 50MB: 50초 필요
   - 5MB: 5초 필요
   - 압축 가치: 매우 높음
   
   빠른 네트워크 (1 Gbps):
   - 50MB: 50ms 필요
   - 5MB: 5ms 필요
   - 압축 가치: 낮음 (오버헤드 > 절감)
   ```

3. **응답 크기에 따라 달라짐**
   ```
   작은 응답 (< 1KB):
   - 압축하면 더 커질 수 있음 (헤더 오버헤드)
   
   설정:
   server.compression.min-response-size=1024  # 1KB 이상만
   ```

**결론:** 100ms 증가는 괜찮다. 전체 응답시간은 5배 이상 개선되기 때문이다. CPU 오버헤드(5~10%)는 무시할 수준이다.

</details>

---

**Q3: 같은 데이터를 여러 번 요청하는데 매번 50MB를 전송해야 할까?**

<details>
<summary>해설 보기</summary>

**아니다. 캐싱을 사용하자.**

**캐싱 전략:**

1. **HTTP 캐싱 헤더**
```java
@GetMapping("/users")
public ResponseEntity<List<UserDTO>> getAllUsers() {
    HttpHeaders headers = new HttpHeaders();
    headers.setCacheControl("max-age=3600");  // 1시간 캐시
    headers.setETag("\"v1-hash\"");  // 변경 감지
    
    return new ResponseEntity<>(users, headers, HttpStatus.OK);
}
```

2. **클라이언트 동작**
```
첫 요청:
GET /users
응답: 50MB (저장)

두 번째 요청 (1분 후):
GET /users
클라이언트: 캐시 있음 (1시간 유효) → 요청 안 함
→ 네트워크 0, 응답시간 1ms (메모리에서)

한 시간 후:
GET /users
응답: 304 Not Modified (ETag 일치)
→ 네트워크 1KB, 응답시간 10ms
```

3. **서버 캐싱**
```java
@GetMapping("/users")
@Cacheable("users")  // Spring Cache
public List<UserDTO> getAllUsers() {
    // DB 쿼리 실행 (1초)
    // 캐시 저장 (key: "users")
}

// 두 번째 호출: 메모리에서 반환 (1ms)
```

4. **CDN 캐싱** (최고 성능)
```
CloudFront, Cloudflare 등
- 엣지 노드에 캐시
- 지역별 최적 응답
- 50MB → 10MB 네트워크 (캐시 히트율 80%)
```

**결론:** 정적 또는 자주 변경되지 않는 데이터는 반드시 캐싱하자. 응답시간을 1000배까지 개선할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: 스레드 풀 분석 — Thread Dump로 BLOCKED 원인 특정](./06-thread-pool-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — GC 로그 분석 ➡️](../jvm-profiling/01-gc-log-analysis.md)**

</div>
