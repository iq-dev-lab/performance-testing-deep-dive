# 02. Java Flight Recorder(JFR) — 프로덕션 안전 사용법

---

## 🎯 핵심 질문

**"GC 로그는 메모리만 보여준다. CPU 사용률이 높은 이유, Lock이 많은 이유, I/O가 느린 이유는 어떻게 알지?"**

**Java Flight Recorder(JFR)** 는 GC를 포함한 **JVM 전체의 성능 이벤트를 한 번에 기록**합니다. 가장 중요한 점은 **프로덕션에서 1% 미만의 오버헤드로 동작**한다는 것입니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

GC 로그만으로는 불충분합니다:

```
문제: "응답시간이 500ms 지연된다"

GC 로그만 본다면:
- GC STW: 50ms
- 나머지 450ms는? 🤷

JFR로 본다면:
- GC STW: 50ms
- Database I/O 대기: 200ms
- Lock 경합: 150ms
- CPU 계산: 50ms
→ 원인을 정확히 파악!
```

**JFR의 강점:**
1. **CPU 프로파일링** — 어느 메서드가 CPU를 먹는가
2. **메모리 할당 추적** — 누가 객체를 생성하는가
3. **Lock 분석** — 어느 Lock이 병목인가
4. **I/O 모니터링** — 네트워크/디스크 대기 시간
5. **프로덕션 안전** — 오버헤드 < 1%

---

## 😱 흔한 실수 (Before)

```bash
# ❌ JFR을 활성화하지 않음
java -Xmx2g MyApplication
# 결과: 성능 문제 원인을 알 수 없음
```

```bash
# ❌ JFR 설정이 너무 상세함 (과한 오버헤드)
java -Xmx2g \
  -XX:+UnlockCommercialFeatures \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=24h,filename=jfr.jfr \
  MyApplication
# 결과: 오버헤드 5~10% (프로덕션 부적합)
```

**문제점:**
- JFR 존재를 모르거나 복잡하게 생각
- 과한 설정으로 오버헤드 발생
- 녹화 중지 방법 모름
- JMC(Java Mission Control) 사용 방법 불명

---

## ✨ 올바른 접근 (After)

### 1단계: 애플리케이션 시작 시 JFR 활성화

**Java 11+ (가장 간단):**
```bash
java -Xmx2g -Xms2g \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=5m,filename=/tmp/recording.jfr \
  MyApplication
```

**설명:**
- `+FlightRecorder` — JFR 활성화 (항상 활성화해도 좋음)
- `duration=5m` — 5분 동안 녹화
- `filename=/tmp/recording.jfr` — 저장 위치

**프로덕션용 (저오버헤드):**
```bash
java -Xmx2g -Xms2g \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=10m,filename=/logs/jfr/recording.jfr,\
settings=profile,maxage=1h,maxsize=1g \
  MyApplication
```

**설명:**
- `settings=profile` — 사전 정의된 프로파일 (low-overhead)
- `maxage=1h` — 1시간 이상 된 기록 자동 삭제
- `maxsize=1g` — 최대 1GB까지만 저장

### 2단계: 실행 중인 애플리케이션에서 JFR 녹화

```bash
# 현재 실행 중인 모든 JVM 확인
jps -l
# Output:
# 12345 com.mycompany.MyApplication
# 12346 org.springframework.boot.loader.JarLauncher

# JFR 녹화 시작 (30초)
jcmd 12345 JFR.start duration=30s filename=/tmp/recording.jfr

# 상태 확인
jcmd 12345 JFR.check

# Output:
# Recording 1: name=1 (running)
# 
# File: /tmp/recording.jfr
# Size: 45 MB

# 녹화 중지 (자동으로 저장됨)
jcmd 12345 JFR.stop filename=/tmp/final-recording.jfr
```

### 3단계: JFR 데이터 분석

**JDK Mission Control(JMC) 설치:**
```bash
# JDK 11~16의 경우 JMC가 포함됨
# JDK 17+의 경우 별도 다운로드 필요
wget https://github.com/openjdk/jmc-project/releases/download/8.2.0/jmc-8.2.0_linux.gtk.x86_64.tar.gz
tar -xzf jmc-8.2.0_linux.gtk.x86_64.tar.gz
cd jmc/bin
./jmc
```

**JMC에서 JFR 파일 열기:**
1. File → Open → recording.jfr
2. 자동으로 각 탭이 분석됨:
   - Threads (CPU 사용)
   - Memory (메모리 할당, GC)
   - File I/O
   - Network I/O
   - Locks
   - Events

---

## 🔬 내부 동작 원리

### JFR이 기록하는 이벤트 종류

```
┌─────────────────────────────────────────────────────────────┐
│ CPU 이벤트                                                  │
├─────────────────────────────────────────────────────────────┤
│ • 메서드 샘플링 (Execution Sample)                           │
│   - 매 1ms마다 CPU에서 실행 중인 메서드 기록               │
│   - Flame Graph의 기초 데이터                               │
│                                                              │
│ • 예외 (Exception)                                           │
│   - 발생한 모든 예외 기록                                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 메모리 이벤트                                                │
├─────────────────────────────────────────────────────────────┤
│ • 객체 할당 (Object Allocation)                             │
│   - 크기 > 16KB인 모든 할당                                 │
│   - 또는 매 X번째 할당마다 샘플링                           │
│                                                              │
│ • GC 이벤트                                                 │
│   - GC 시작/종료                                            │
│   - STW 시간, 메모리 해제량                                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Lock 이벤트                                                 │
├─────────────────────────────────────────────────────────────┤
│ • Contention (경합)                                        │
│   - 누가 Lock을 기다렸는가                                  │
│   - 얼마나 오래 기다렸는가                                  │
│                                                              │
│ • Thread Park                                               │
│   - LockSupport.park() 호출                                 │
│   - 대기 시간                                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ I/O 이벤트                                                  │
├─────────────────────────────────────────────────────────────┤
│ • 파일 I/O                                                  │
│   - 읽기/쓰기 작업                                          │
│   - 소요 시간                                                │
│                                                              │
│ • 네트워크 I/O                                              │
│   - Socket 읽기/쓰기                                        │
│   - 대기 시간                                                │
└─────────────────────────────────────────────────────────────┘
```

### JFR 오버헤드가 낮은 이유

```
┌──────────────────────────────────────┐
│ Traditional Profiling (조사식)         │
├──────────────────────────────────────┤
│ 모든 메서드 진입/퇴출 추적            │
│ 스택 트레이스 매번 기록               │
│ → 오버헤드: 10~50%                   │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ JFR Sampling (샘플링 방식)             │
├──────────────────────────────────────┤
│ 1ms마다 "현재 뭐하는가?"만 기록      │
│ 선택적 샘플링만 수행                  │
│ 링 버퍼에 순환식으로 저장             │
│ → 오버헤드: < 1%                     │
└──────────────────────────────────────┘
```

### JFR 데이터 흐름

```
Java 애플리케이션
        ↓
   [JFR 엔진]
        ↓
┌───────┬────────┬────────┬────────┐
│ CPU   │ Memory │ Locks  │ I/O    │ (이벤트 수집)
└───────┴────────┴────────┴────────┘
        ↓
   [링 버퍼]  (메모리 순환 저장, 최대 1GB)
        ↓
   recording.jfr (파일 저장)
        ↓
   JDK Mission Control (분석 및 시각화)
```

---

## 💻 실전 실험

### 실험 1: CPU 프로파일링

```java
// CpuIntensiveApp.java
public class CpuIntensiveApp {
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000); // JFR 준비 시간
        
        for (int iteration = 0; iteration < 10; iteration++) {
            System.out.println("반복 " + iteration);
            
            // 의도적으로 CPU 소비
            busyWork(iteration);
            
            Thread.sleep(1000);
        }
    }
    
    static void busyWork(int iteration) {
        if (iteration % 3 == 0) {
            methodA(); // CPU 많이 사용
        } else if (iteration % 3 == 1) {
            methodB(); // CPU 중간 사용
        } else {
            methodC(); // CPU 적게 사용
        }
    }
    
    static void methodA() {
        long sum = 0;
        for (long i = 0; i < 1000000000L; i++) {
            sum += Math.sqrt(i);
        }
    }
    
    static void methodB() {
        long sum = 0;
        for (long i = 0; i < 500000000L; i++) {
            sum += Math.sqrt(i);
        }
    }
    
    static void methodC() {
        long sum = 0;
        for (long i = 0; i < 100000000L; i++) {
            sum += Math.sqrt(i);
        }
    }
}
```

**실행:**
```bash
java -Xmx512m \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=20s,filename=/tmp/cpu-profile.jfr,\
settings=profile \
  CpuIntensiveApp
```

**JMC 분석:**
- Threads 탭에서 각 메서드의 CPU 사용률 비율 확인
- methodA가 가장 높은 비율 차지 예상

---

### 실험 2: 메모리 할당 프로파일링

```java
// MemoryAllocationApp.java
public class MemoryAllocationApp {
    static class DataObject {
        byte[] data = new byte[1024 * 1024]; // 1MB
    }
    
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000);
        
        for (int i = 0; i < 100; i++) {
            allocateMemory(i);
            Thread.sleep(100);
        }
    }
    
    static void allocateMemory(int iteration) {
        List<DataObject> list = new ArrayList<>();
        
        // 많은 객체 할당
        for (int j = 0; j < 20; j++) {
            list.add(new DataObject());
        }
        
        // 메서드 종료 시 자동 해제
        System.out.println("할당 완료: " + iteration);
    }
}
```

**실행:**
```bash
java -Xmx2g \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=15s,filename=/tmp/memory-alloc.jfr,\
settings=profile \
  MemoryAllocationApp
```

**JFR 출력 예시:**
```
Event Type: jdk.ObjectAllocationOutsideTLAB
Time: 2024-04-10 14:23:45.123
Thread: main
Allocation Size: 1048576 (1MB)
Allocation Class: byte[]
Stack:
  at java.lang.System.arraycopy(System.java:...)
  at java.util.Arrays.copyOf(Arrays.java:...)
  at java.util.ArrayList.grow(ArrayList.java:...)
  at java.util.ArrayList.add(ArrayList.java:...)
  at MemoryAllocationApp.allocateMemory(MemoryAllocationApp.java:19)
  at MemoryAllocationApp.main(MemoryAllocationApp.java:10)
```

---

### 실험 3: Lock 경합 분석

```java
// LockContentionApp.java
import java.util.concurrent.*;

public class LockContentionApp {
    static class Counter {
        private int count = 0;
        
        // synchronized → Lock 경합 발생
        synchronized void increment() {
            count++;
        }
        
        synchronized int getCount() {
            return count;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000);
        
        Counter counter = new Counter();
        
        // 10개 스레드에서 동시에 카운터 증가
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                for (int j = 0; j < 1000000; j++) {
                    counter.increment();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(60, TimeUnit.SECONDS);
        
        System.out.println("최종 카운트: " + counter.getCount());
    }
}
```

**실행:**
```bash
java -Xmx512m \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=30s,filename=/tmp/lock-contention.jfr,\
settings=profile \
  LockContentionApp
```

**JFR 결과:**
```
Event Type: jdk.JavaMonitorBlocked
Time: 2024-04-10 14:24:10.456
Thread: pool-1-thread-2
Duration: 1234ms
Lock: LockContentionApp$Counter.increment()
Blocking Class: java.util.concurrent.locks.ReentrantLock

Monitor: java.lang.Object@123456
Wait Time: 1234ms
Thread Count Waiting: 9
```

---

## 📊 성능 비교

| 측정 항목 | 오버헤드 없음 | JFR 활성화 | JFR + 녹화 중 | 전통 프로파일링 |
|---------|-----------|---------|-----------|------------|
| 처리량 (req/sec) | 10,000 | 10,000 | 9,900 | 5,000 |
| 오버헤드 | 0% | 0% | 1% | 50% |
| 메모리 사용 | 512MB | 512MB | 750MB | 1GB+ |
| CPU 사용 | 50% | 50% | 51% | 75% |
| 데이터 수집 | 불가 | 준비됨 | 활성 | 활성 |
| 프로덕션 적합성 | - | 적합 | 적합 | 부적합 |

**결론:** JFR은 항상 활성화해도 괜찮으며, 필요할 때만 녹화하면 됩니다.

---

## ⚖️ 트레이드오프

### JFR 설정 수준 선택

```
┌────────────────────────────────────────────┐
│ 설정: default (기본)                       │
├────────────────────────────────────────────┤
│ 이벤트 수: 적음                             │
│ 오버헤드: < 0.5%                          │
│ 저장 용량: 500MB/시간                      │
│ 분석 가능: GC, Major events only           │
│ 프로덕션: 항상 켜도 됨                     │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ 설정: profile (추천)                       │
├────────────────────────────────────────────┤
│ 이벤트 수: 중간                             │
│ 오버헤드: < 1%                            │
│ 저장 용량: 1GB/시간                        │
│ 분석 가능: CPU, Memory, Locks               │
│ 프로덕션: 필요할 때 녹화                   │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ 설정: continuous (고상세)                  │
├────────────────────────────────────────────┤
│ 이벤트 수: 많음                             │
│ 오버헤드: 1~2%                            │
│ 저장 용량: 5GB/시간                        │
│ 분석 가능: 완전한 분석                     │
│ 프로덕션: 개발 환경만 권장                 │
└────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | 설명 |
|------|------|
| **JFR 활성화** | `-XX:+FlightRecorder` (권장: 항상 켜두기) |
| **녹화 시작** | `jcmd PID JFR.start ...` 또는 시작 옵션 |
| **녹화 중지** | `jcmd PID JFR.stop filename=...` |
| **오버헤드** | < 1% (프로덕션 안전) |
| **분석 도구** | JDK Mission Control |
| **주요 이벤트** | CPU, Memory, Locks, I/O |
| **저장 위치** | `.jfr` 파일 (이진 형식) |
| **권장 설정** | `settings=profile` (균형잡힌) |

---

## 🤔 생각해볼 문제

**Q1: 다음 JFR 설정에서 문제점을 찾으세요**

```bash
java -Xmx4g \
  -XX:StartFlightRecording=duration=24h,filename=/tmp/recording.jfr,\
settings=continuous,maxsize=10g \
  MyApplication
```

<details>
<summary>💡 해설</summary>

**문제점:**
1. **settings=continuous** → 오버헤드 1~2% (프로덕션 부적합)
2. **duration=24h** → 24시간 녹화는 과함
3. **maxsize=10g** → 디스크 용량 낭비

**개선된 설정:**
```bash
java -Xmx4g \
  -XX:+FlightRecorder \
  MyApplication

# 필요할 때만 녹화
jcmd PID JFR.start duration=5m,filename=/tmp/recording.jfr,\
settings=profile,maxsize=1g
```

**이유:**
- JFR은 항상 활성화해도 오버헤드 없음
- 녹화는 필요할 때만
- 용량 제한으로 디스크 손상 방지

</details>

---

**Q2: JFR 데이터에서 다음을 발견했습니다. 어떤 문제가 있는지 분석하세요**

```
이벤트 1: jdk.JavaMonitorBlocked
  Lock: java.util.concurrent.ConcurrentHashMap.putIfAbsent()
  Wait Time: 5234ms
  Thread Count Waiting: 23
  발생 빈도: 100회/초

이벤트 2: jdk.ObjectAllocationOutsideTLAB
  대상: byte[] (매번 1MB)
  발생: 1000회/초
```

<details>
<summary>💡 해설</summary>

**문제 1: 심각한 Lock 경합**
- 23개 스레드가 동시에 같은 Lock 대기
- 한 번의 작업에 5초 소요
- → ConcurrentHashMap도 병목

**원인 추적:**
```bash
# 스택 트레이스 확인
jfr print --events jdk.JavaMonitorBlocked recording.jfr
```

**해결책:**
- Lock-free 자료구조로 변경 (ConcurrentHashMap → LongAdder)
- ReadWriteLock으로 변경
- 캐시 전략 변경

**문제 2: 과도한 메모리 할당**
- 초당 1,000회 × 1MB = 1GB/초 (비정상)
- Young GC 매우 빈번 필요

**해결책:**
- 객체 풀링 (재사용)
- 할당 크기 감소
- 할당 빈도 감소 (배치 처리)

</details>

---

**Q3: Java 14+ StreamingAPI를 이용한 실시간 JFR 분석 코드를 작성하세요**

```java
// 힌트: jdk.jfr.consumer.RecordingFile 사용
```

<details>
<summary>💡 해설</summary>

```java
import jdk.jfr.consumer.RecordingFile;
import jdk.jfr.ValueDescriptor;

public class JFRStreamingAnalysis {
    public static void main(String[] args) throws IOException {
        // 현재 실행 중인 JFR에서 데이터 읽기
        try (RecordingFile recordingFile = 
             new RecordingFile(Paths.get("/tmp/recording.jfr"))) {
            
            while (recordingFile.hasMoreEvents()) {
                RecordedEvent event = recordingFile.readEvent();
                
                String eventType = event.getEventType().getName();
                Instant eventTime = event.getStartTime();
                
                // GC 이벤트 필터
                if (eventType.contains("gc")) {
                    System.out.printf("[%s] GC 이벤트: %s%n", 
                        eventTime, event);
                }
                
                // Lock 경합 이벤트 필터
                if (eventType.contains("Blocked")) {
                    Duration duration = event.getDuration();
                    System.out.printf("[%s] Lock 대기: %dms%n", 
                        eventTime, duration.toMillis());
                }
                
                // 메모리 할당 이벤트 필터
                if (eventType.contains("Allocation")) {
                    long size = event.getLong("allocationSize");
                    String className = event.getString("objectClass");
                    System.out.printf("[%s] 할당: %s, 크기: %d bytes%n", 
                        eventTime, className, size);
                }
            }
        }
    }
}
```

**실행:**
```bash
javac JFRStreamingAnalysis.java
java --add-modules jdk.jfr JFRStreamingAnalysis
```

**장점:**
- 실시간 이벤트 처리
- 사용자 정의 필터링
- 데이터베이스 저장 등 통합

</details>

---

<div align="center">

**[⬅️ 이전: GC 로그 분석 — Stop-The-World 시간 측정](./01-gc-log-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: async-profiler — Flame Graph로 코드 레벨 병목 찾기 ➡️](./03-async-profiler.md)**

</div>
