# 04. Heap Dump 분석 — 메모리 누수 원인 특정

---

## 🎯 핵심 질문

**"메모리 누수가 있다는 건 알겠는데, 정확히 뭐가 메모리를 잡고 있는가? 어느 객체가 해제되지 않는가?"**

**Heap Dump**는 특정 순간의 JVM 메모리 스냅샷입니다. **MAT(Memory Analyzer Tool)**로 분석하면 메모리 누수의 **정확한 원인을 보유 체인(Retention Path)으로 추적**할 수 있습니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

메모리 누수 문제 해결의 흐름:

```
1단계: GC 로그 확인
   ↓ Full GC 빈번, 메모리 증가 → 메모리 누수 의심
   
2단계: async-profiler 메모리 할당 분석
   ↓ 누가 메모리를 과도하게 할당하는지 확인
   
3단계: Heap Dump 수집
   ↓ 현재 시점의 메모리 스냅샷
   
4단계: MAT 분석
   ↓ 정확한 누수 원인 파악 ✓ (이 문서의 목표)

예:
"Session 객체 100,000개가 메모리에 있다"
→ 왜? Retention Path 추적
→ UserCache.sessions → CacheEntry[] → Session 객체 → 문제 발견!
→ Cache 비우기 로직 누락 확인
→ 해결!
```

**Heap Dump 분석의 가치:**
1. **현재 메모리 상태 정확히 파악** — 어떤 객체가 몇 개?
2. **누수 원인 추적** — GC Root에서부터의 보유 경로
3. **메모리 낭비 지점 발견** — 의도하지 않은 대용량 객체
4. **스냅샷 비교** — 시간에 따른 변화 추적

---

## 😱 흔한 실수 (Before)

```bash
# ❌ Heap Dump를 수집하지 않음
# 로그와 추측만으로 원인 파악 시도
tail -f /var/log/app.log
# 결과: 정확한 원인 알 수 없음, 무작정 힙 증가

# ❌ Heap Dump는 수집하지만 분석 안 함
jmap -dump:format=b,file=heap.hprof PID
# 생성: heap.hprof (5GB)
# 이후: "너무 크니까 분석할 수 없다"는 식으로 방치
```

**문제점:**
- Heap Dump를 모르거나 복잡하게 생각
- 수집했어도 분석 방법 모름
- MAT 없이 raw 데이터 분석 불가능

---

## ✨ 올바른 접근 (After)

### 1단계: Heap Dump 수집

**방법 1: jmap 사용 (간단)**

```bash
# 현재 실행 중인 JVM 찾기
jps -l
# Output: 12345 com.mycompany.MyApplication

# Heap Dump 수집 (시간이 걸릴 수 있음)
jmap -dump:format=b,file=heap.hprof 12345

# 진행 상황 모니터링 (별도 터미널)
watch -n 1 'ls -lh heap.hprof'

# 완료 후 확인
ls -lh heap.hprof
# -rw-r--r-- ... 5.2G heap.hprof
```

**방법 2: Spring Boot Actuator (더 편함)**

```properties
# application.properties
management.endpoints.web.exposure.include=heapdump
management.endpoint.heapdump.enabled=true
```

```bash
# HTTP 요청으로 Heap Dump 수집
curl -X GET http://localhost:8080/actuator/heapdump > heap.hprof

# 또는 jcmd 사용
jcmd 12345 GC.heap_dump /tmp/heap.hprof
```

**방법 3: 자동 Dump (OOM 발생 시)**

```bash
java -Xmx2g \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/logs/heap-dumps/ \
  MyApplication
# OOM 발생 시 자동으로 /logs/heap-dumps/java_pid*.hprof 생성
```

### 2단계: MAT 설치 및 열기

```bash
# Eclipse MAT 다운로드 (독립형)
wget https://download.eclipse.org/mat/1.14.0/MemoryAnalyzer-1.14.0.20230405-linux.gtk.x86_64.tar.gz
tar -xzf MemoryAnalyzer-1.14.0.20230405-linux.gtk.x86_64.tar.gz

# GUI 실행 (X11 필요)
./mat/MemoryAnalyzer

# 또는 cmdline 모드 (서버 환경)
./mat/ParseHeapDump.sh /path/to/heap.hprof -console
```

### 3단계: Leak Suspects 리포트 생성

MAT 실행 후:

```
1. File → Open Heap Dump → heap.hprof 선택
2. "Memory Analyzer is analysing..." 대기 (수분)
3. 분석 완료 → "Leak Suspects" 탭 자동 표시
4. 리포트에서 문제점 확인
```

**Leak Suspects 출력 예시:**

```
PROBLEM 1: Suspected Leak (Problem Type: Probable Suspected Leak)

Suspected Type: com.example.UserCache
Count: 98,765
Memory Footprint: 1.2GB (95%)

Shortest Path to GC Roots:
  UserCache (1.2GB)
  ├─ sessions (HashMap) 
  │  ├─ 98,765 Session objects
  │  └─ 각 Session: byte[] data[100KB]
  └─ GC Root: static UserCache instance

Leak Chain:
  static UserCache → sessions map → Session → byte[]
  
Recommendation:
  - UserCache.sessions.clear() 호출 누락
  - Cache eviction policy 부재
  - 또는 Session TTL 설정 부재
```

### 4단계: 상세 분석 (Dominator Tree)

MAT 좌측 메뉴에서 "Dominator Tree" 선택:

```
Dominator Tree 구조:

Shallow Heap  Retained Heap  Class
────────────  ─────────────  ─────────────
10MB          1.2GB          byte[]  ← 가장 많은 메모리 점유
8MB           1.1GB          HashMap$Node
5MB           900MB          Session
2MB           50MB           String[]
```

**각 항목 설명:**
- **Shallow Heap**: 객체 자신 크기 (예: Session 객체 = 100KB)
- **Retained Heap**: 객체가 보유한 모든 메모리 (Session → byte[100MB] 포함)
- 오른쪽 클래스명을 클릭하면 상세 분석

### 5단계: Retention Path 추적

`Dominator Tree`에서 의심 객체(예: byte[]) 선택:

```
1. 우클릭 → "Show in Dominator Tree"
2. 또는 "GC Root Path" 선택
3. 어떤 객체가 이 byte[]를 참조하는가 표시:

   UserCache (static)
   └─ sessions (HashMap)
      └─ CacheEntry (value)
         └─ Session (session)
            └─ byte[] userData (field)

이 경로가 "Retention Path" = 메모리를 유지하는 경로
```

---

## 🔬 내부 동작 원리

### Heap Dump 구조

```
┌────────────────────────────────────────────────────────┐
│ Heap Dump 파일 (heap.hprof)                            │
├────────────────────────────────────────────────────────┤
│ • 클래스 정보 메타데이터                               │
│   - 클래스명, 필드명, 메서드명 등                      │
│                                                        │
│ • 객체 인스턴스 정보                                    │
│   - 객체 주소, 타입, 필드 값                            │
│   - 예: Object@0x123456: userData[100KB] 참조         │
│                                                        │
│ • GC Root 정보                                         │
│   - 정적 변수, 스택 변수, 활성 스레드 등             │
│   - 이들이 "살아있는" 객체의 시작점                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 메모리 누수 감지 메커니즘

```
1단계: 모든 객체 순회
   ↓
   UserCache (살아있음)
   ├─ Session #1 (참조됨)
   ├─ Session #2 (참조됨)
   ├─ ...
   └─ Session #98765 (참조됨)

2단계: 각 객체의 보유 메모리 계산
   ↓
   UserCache의 Retained Heap:
   = UserCache 자신 (1MB)
   + sessions HashMap (100MB)
   + 모든 Session 객체들 (1GB)
   + 각 Session의 byte[] (100MB)
   = 약 1.2GB

3단계: 이상 탐지
   ↓
   "하나의 객체가 전체 메모리의 95%를 보유"
   → Suspected Leak!
```

---

## 💻 실전 실험

### 실험 1: 의도적 메모리 누수 생성 및 분석

```java
// MemoryLeakApp.java
import java.util.*;

public class MemoryLeakApp {
    // 정적 필드 = GC Root (절대 정리되지 않음)
    static List<byte[]> leakedCache = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("메모리 누수 시뮬레이션 시작");
        Thread.sleep(3000); // Heap dump 시간 확보
        
        // 메모리 누수: 1GB 이상 할당하고 절대 해제 안 함
        for (int i = 0; i < 100; i++) {
            byte[] largeData = new byte[10 * 1024 * 1024]; // 10MB
            leakedCache.add(largeData);
            
            if (i % 10 == 0) {
                System.out.printf("누적: %dMB%n", i * 10);
            }
            
            Thread.sleep(100);
        }
        
        System.out.println("누적: 1000MB");
        System.out.println("이제 Heap Dump를 수집하세요");
        
        // 프로그램 실행 유지
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

**실행:**
```bash
javac MemoryLeakApp.java

# 터미널 1: 애플리케이션 실행
java -Xmx2g -Xms2g MemoryLeakApp

# 터미널 2: PID 확인 후 Heap Dump 수집
jps -l | grep MemoryLeakApp
# 12345 MemoryLeakApp

jmap -dump:format=b,file=leak.hprof 12345
```

**MAT 분석:**

```
1. File → Open Heap Dump → leak.hprof
2. "Leak Suspects" 탭 확인:

   PROBLEM 1: Probable Suspected Leak
   
   Suspected Type: byte[]
   Count: 100
   Memory Footprint: 1.0GB (99%)
   
   Shortest Path:
   byte[] (1.0GB)
   └─ ArrayList.elementData
      └─ static leakedCache
         └─ MemoryLeakApp

Recommendation:
   - leakedCache를 절대 정리하지 않음
   - 정적 필드이므로 프로그램 종료까지 유지
   - 해결: leakedCache.clear() 또는 WeakReference 사용
```

---

### 실험 2: Session 캐시 누수 분석

```java
// SessionLeakApp.java
import java.util.*;

public class SessionLeakApp {
    static class Session {
        String sessionId;
        byte[] userData = new byte[1024 * 1024]; // 1MB per session
        
        Session(String id) {
            this.sessionId = id;
        }
    }
    
    // 캐시 (절대 비워지지 않음)
    static Map<String, Session> sessionCache = 
        Collections.synchronizedMap(new HashMap<>());
    
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(3000);
        
        System.out.println("Session 누수 시뮬레이션");
        
        // 10,000개 세션 생성
        for (int i = 0; i < 10000; i++) {
            String sessionId = "session_" + i;
            Session session = new Session(sessionId);
            sessionCache.put(sessionId, session);
            
            if (i % 1000 == 0) {
                System.out.printf("생성: %d개 세션 (%dMB)%n", 
                    i, i); // 각 세션 1MB
            }
        }
        
        System.out.println("10,000개 세션 = 10GB 메모리 사용");
        System.out.println("Heap Dump를 수집하세요");
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

**MAT 분석:**

```
Dominator Tree:
Shallow Heap  Retained Heap  Class
────────────  ─────────────  ──────────────────
1KB           10GB           byte[] ← 메모리 대부분
1KB           9.9GB          Session
10KB          9.8GB          HashMap$Node
```

**GC Root Path:**
```
HashMap (sessionCache)
├─ Node[0]
│  ├─ Session #1
│  │  └─ byte[] userData (1MB)
├─ Node[1]
│  ├─ Session #2
│  │  └─ byte[] userData (1MB)
└─ ... (9998개 더)

GC Root: static sessionCache → 모든 Session 유지됨
```

---

### 실험 3: Heap Dump 비교 (시간에 따른 변화)

```bash
# 1시간 간격으로 Heap Dump 수집
while true; do
  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
  jmap -dump:format=b,file=heap_$TIMESTAMP.hprof 12345
  echo "Dump collected: heap_$TIMESTAMP.hprof"
  sleep 3600 # 1시간 대기
done
```

**비교 분석:**

```
heap_20240410_100000.hprof (1시간 후)
  - 메모리 사용: 500MB
  - String 객체: 5,000개

heap_20240410_110000.hprof (2시간 후)
  - 메모리 사용: 700MB
  - String 객체: 7,000개

heap_20240410_120000.hprof (3시간 후)
  - 메모리 사용: 900MB
  - String 객체: 9,000개

→ String 객체가 매시간 2,000개씩 증가
→ String 누수 의심
→ 어디서? 캐시? 로깅? 캐시키?

MAT에서 "Compare with" 기능 사용:
1. heap_100000.hprof 열기
2. Preferences → "Compare with heap_120000.hprof"
3. 새로 추가된 객체만 보기
→ 정확한 누수 원인 파악!
```

---

## 📊 성능 비교

| 측정 항목 | Heap Dump 수집 | Heap Dump 분석 | MAT 실행 |
|---------|------------|-------------|---------|
| 소요 시간 | 5~20분 (힙 크기에 따라) | 5~30분 | 즉시 (~1분) |
| 애플리케이션 영향 | STW (수초~수분) | 없음 | 없음 |
| 요구 메모리 | 힙 크기와 동일 | 디스크 공간 (파일) | 디스크 + 8GB RAM |
| 정확도 | 100% (현재 상태 스냅샷) | 100% (정확한 분석) | - |
| 점유 메모리 비율 | - | - | - |
| Top 3 메모리 사용 객체 | 불가 | 가능 | 시각화 |

---

## ⚖️ 트레이드오프

### Heap Dump 수집 방법

```
┌────────────────────────────────────────┐
│ jmap (명령어)                           │
├────────────────────────────────────────┤
│ 장점:                                  │
│  • 추가 설정 불필요                    │
│  • 실행 중 언제든 가능                  │
│                                        │
│ 단점:                                  │
│  • STW 발생 (수초~수분)                │
│  • 파일 크기 큼 (힙과 동일)            │
│  • 전체 힙 덤프 (불필요 부분 포함)    │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Spring Boot Actuator                   │
├────────────────────────────────────────┤
│ 장점:                                  │
│  • HTTP로 편하게 다운로드              │
│  • STW 시간 짧음                       │
│                                        │
│ 단점:                                  │
│  • Spring Boot 필수                    │
│  • 네트워크 전송 시간 (큰 파일)        │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ HeapDumpOnOutOfMemoryError              │
├────────────────────────────────────────┤
│ 장점:                                  │
│  • 자동 수집 (수동 개입 불필요)        │
│  • OOM 전 상태 캡처                    │
│                                        │
│ 단점:                                  │
│  • 반응성 문제 (OOM 발생)              │
│  • 결과: 서비스 중단                   │
└────────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | 설명 |
|------|------|
| **Heap Dump 수집** | `jmap -dump:format=b,file=heap.hprof PID` |
| **Spring Boot** | `GET /actuator/heapdump` |
| **자동 수집** | `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/` |
| **분석 도구** | MAT (Memory Analyzer Tool) |
| **주요 리포트** | Leak Suspects, Dominator Tree |
| **누수 추적** | Retention Path (GC Root → 객체) |
| **메모리 크기** | Shallow Heap (객체 크기) vs Retained Heap (보유 메모리) |
| **비교 분석** | 시간에 따른 Heap Dump 비교로 누수 확인 |

---

## 🤔 생각해볼 문제

**Q1: 다음 MAT 리포트를 분석하세요**

```
Leak Suspects:

PROBLEM 1
- Type: java.util.HashMap
- Count: 1
- Memory: 2.5GB (100%)
- Retention Path:
  HashMap
  └─ table (Node[])
     └─ 1,000,000개 Entry
        └─ CacheKey + CacheValue (각 2.5KB)
        
GC Root: static CacheManager.cache
```

메모리 누수인가, 아니면 정상 사용인가?

<details>
<summary>💡 해설</summary>

**분석:**

정상 사용일 가능성이 높지만, 다음을 확인해야 함:

1. **캐시 크기가 제한되는가?**
   - `maxSize=1,000,000`인가? → 정상
   - 아니면 무제한 증가? → 누수

2. **캐시 제거 정책이 있는가?**
   - TTL(Time To Live)이 설정되어 있는가?
   - LRU(Least Recently Used) 제거가 작동하는가?
   - 없으면: 누수!

3. **비즈니스 요구사항?**
   - 정말 1,000,000개를 메모리에 유지해야 하나?
   - 데이터베이스 쿼리로 대체 가능한가?

**진단 명령:**
```java
// 코드에서 확인
System.out.println("캐시 크기: " + CacheManager.cache.size());

// 매시간 체크
for (int hour = 0; hour < 24; hour++) {
  System.out.printf("%d:00 - %d entries%n", 
    hour, CacheManager.cache.size());
  Thread.sleep(3600000);
}
```

**캐시 크기가 증가하면: 누수**
**캐시 크기가 안정적이면: 정상 (예: 항상 1M 유지)**

</details>

---

**Q2: 다음 시나리오에서 메모리 누수를 확인하는 방법은?**

```
상황:
- 애플리케이션 1주일 운영
- Heap 사용: 10MB → 500MB → 1000MB → 1500MB
- GC 로그: Full GC 빈번 (6시간마다)
- CPU: 정상 (20%)
- 응답시간: 증가 추세 (100ms → 500ms)

누수 원인을 정확히 파악하는 단계별 과정?
```

<details>
<summary>💡 해설</summary>

**단계별 진단:**

1. **GC 로그 확인 (이미 완료)**
   ```
   ✓ Full GC가 6시간마다 → 메모리 누수 신호
   ✓ Heap 계속 증가 → 누수 확인됨
   ```

2. **async-profiler 메모리 할당 분석**
   ```bash
   ./profiler.sh -d 60 -f memory-alloc.html -e alloc PID
   # 결과: 누가 메모리를 과다 할당하는가?
   # 예: methodA()가 60% 메모리 할당
   ```

3. **Heap Dump 수집 (2개)**
   ```bash
   # 1주일 후
   jmap -dump:format=b,file=heap_1week.hprof PID
   
   # 2주일 후 (또는 문제 악화 후)
   jmap -dump:format=b,file=heap_2weeks.hprof PID
   ```

4. **MAT로 두 덤프 비교**
   ```
   - heap_1week.hprof 열기
   - "Compare with" → heap_2weeks.hprof
   - "New Objects" 탭에서 새로 추가된 객체 확인
   
   결과 예:
   - Session 객체 1,000 → 50,000 (49,000 증가)
   - CacheEntry 객체 100,000 → 500,000 (400,000 증가)
   - → Session이나 CacheEntry 누수 확인!
   ```

5. **Retention Path 추적**
   ```
   Dominator Tree에서 Session 클릭
   → GC Root Path 확인
   
   결과 예:
   HashMap (UserSessionCache)
   └─ Entry[]
      └─ Session (user_12345)
         └─ byte[] userData
   
   → UserSessionCache.sessions에 Session이 계속 추가되고 제거 안 됨
   ```

6. **코드 확인 및 해결**
   ```java
   // 문제 코드
   static Map<String, Session> sessions = new HashMap<>();
   
   // 추가만 함
   public static void addSession(String userId, Session s) {
     sessions.put(userId, s);
   }
   
   // 제거 로직이 없음!!! ← 문제
   
   // 해결: TTL 또는 명시적 제거
   public static void removeSession(String userId) {
     sessions.remove(userId);
   }
   
   // 또는 CacheBuilder 사용
   Map<String, Session> sessions = 
     CacheBuilder.newBuilder()
       .expireAfterAccess(30, TimeUnit.MINUTES)
       .build()
       .asMap();
   ```

</details>

---

**Q3: MAT에서 다음을 보았습니다. 대처 방법은?**

```
분석 결과:
- 분석 불가: "OutOfMemoryError: Java heap space"
- 현재 컴퓨터 메모리: 8GB
- Heap Dump 파일: 6GB
```

<details>
<summary>💡 해설</summary>

**문제:** MAT 자체가 메모리 부족으로 분석 실패

**해결책:**

1. **MAT 메모리 증가**
   ```bash
   # ~/.eclipse/eclipse.ini 또는 mat/MemoryAnalyzer.ini 수정
   -Xmx8g  # 8GB 할당
   
   # 또는 실행 시
   ./mat/MemoryAnalyzer.exe -vmargs -Xmx8g
   ```

2. **Heap Dump 크기 줄이기**
   ```bash
   # 더 나은 분석: 부분 덤프 사용 (필요시)
   # jmap으로 특정 클래스만 덤프 불가능하므로,
   # 대신 프로덕션 환경에서 이미 충분할 수 있음
   ```

3. **명령줄 분석 도구 사용 (서버 환경)**
   ```bash
   # Eclipse MAT는 메모리 많이 필요하므로,
   # 대신 다른 도구 사용
   
   # jprofiler, YourKit 등 시도
   # 또는 온라인 분석 서비스 사용
   ```

4. **더 큰 메모리 서버에서 분석**
   ```bash
   # 로컬 데스크톱 (8GB) → 분석 서버 (32GB) 전송
   # 분석 후 결과 리포트 받기
   ```

**예방:**
- 프로덕션: 작은 Heap 설정 (필요 범위)
- 개발: 테스트 데이터로 축소
- 정기적 분석: 1주일마다 (누적 전에)

</details>

---

<div align="center">

**[⬅️ 이전: async-profiler — Flame Graph로 코드 레벨 병목 찾기](./03-async-profiler.md)** | **[홈으로 🏠](../README.md)** | **[다음: JVM 튜닝 파라미터 — G1GC vs ZGC 선택 ➡️](./05-jvm-tuning-parameters.md)**

</div>
