# 03. async-profiler — Flame Graph로 코드 레벨 병목 찾기

---

## 🎯 핵심 질문

**"특정 메서드가 느린 건 알겠는데, 그 메서드 내부에서 어디가 병목일까? CPU 샘플링 데이터를 한눈에 보고 싶다."**

**async-profiler**는 JVM 애플리케이션의 CPU, 메모리, Lock을 프로파일링하고 **Flame Graph**로 시각화합니다. Flame Graph는 **병목 메서드를 즉시 시각적으로 파악**할 수 있는 가장 강력한 도구입니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

JFR도 좋지만, **메서드 단위의 정확한 병목**을 찾으려면:

```
문제: methodA() 전체가 느리다 (500ms)
→ 내부 메서드 B, C, D 중 뭐가 병목?

GC 로그: 도움 안 됨
JFR: 상위 메서드만 보임 (methodA 외 상세 안 보임)
async-profiler Flame Graph: 정확히 보임!

    methodA (500ms)
    ├── methodB (10ms)
    ├── methodC (400ms) ← 여기가 병목!
    │   ├── databaseQuery (350ms)
    │   └── dataProcessing (50ms)
    └── methodD (90ms)
```

**async-profiler의 강점:**
1. **CPU 프로파일링** — 메서드 단위 CPU 사용 시간
2. **Flame Graph** — 콜 스택을 가로 길이로 표현
3. **메모리 할당** — 누가 메모리를 할당하는가
4. **Lock 분석** — 누가 Lock에서 기다리는가
5. **프로덕션 안전** — 오버헤드 극히 낮음

---

## 😱 흔한 실수 (Before)

```bash
# ❌ async-profiler를 모르고 다른 도구 사용
# jstack으로 스택 트레이스 샘플링 (효율 낮음)
for i in {1..10}; do
  jstack PID >> stacks.txt
  sleep 1
done
# 결과: 부정확한 데이터, 수동 분석 필요

# ❌ JFR 이벤트 XML 직접 분석
# (가능하지만 너무 복잡)
```

**문제점:**
- 메서드 단위 병목을 정확히 파악 어려움
- 수동으로 콜 스택 정렬해야 함
- 시각화 없음
- 시간 낭비

---

## ✨ 올바른 접근 (After)

### 1단계: async-profiler 설치

```bash
# 최신 버전 다운로드
wget https://github.com/async-profiler/async-profiler/releases/download/v2.9/async-profiler-2.9-linux-x64.tar.gz
tar -xzf async-profiler-2.9-linux-x64.tar.gz
cd async-profiler-2.9

# 실행 권한 확인
ls -la profiler.sh
# -rwxr-xr-x ... profiler.sh
```

### 2단계: CPU 프로파일링 (가장 일반적)

```bash
# 실행 중인 JVM 찾기
jps -l
# Output: 12345 com.mycompany.MyApplication

# 60초 동안 CPU 프로파일링
./profiler.sh -d 60 -f flamegraph.html -e cpu 12345

# 또는 더 짧은 시간
./profiler.sh -d 30 -f cpu-profile.html 12345
```

**결과:** `flamegraph.html` 파일 생성
- 브라우저에서 열기 (인터랙티브)
- 마우스 호버로 상세 정보 확인
- 클릭으로 특정 영역 확대

### 3단계: Flame Graph 해석

생성된 `flamegraph.html`을 브라우저에서 열면:

```
┌─────────────────────────────────────────────────────────────────┐
│                       Stack Frame (위쪽)                         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ main                                      (전체 너비 = 100%) ││
│  │ ┌────────────────────────────────────────────────────────┐││
│  │ │ request()                            (너비 = 80%)       │││
│  │ │ ┌──────────────────────────────────────────────────┐│││
│  │ │ │ databaseQuery()                  (너비 = 50%)    ││││
│  │ │ │ ┌────────────────────────────────────────────┐│││
│  │ │ │ │ executeSQL()               (너비 = 50%)   │││
│  │ │ │ └────────────────────────────────────────────┘│││
│  │ │ ├──────────────────┐                            │││
│  │ │ │dataProcessing()  │ (너비 = 30%)              │││
│  │ │ └──────────────────┘                            │││
│  │ └──────────────────────────────────────────────────┘││
│  │ ┌────────────────────────────────────────────────────┐││
│  │ │ logging()                            (너비 = 20%) │││
│  │ └────────────────────────────────────────────────────┘││
│  └────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘

Flame Graph 읽는 법:
• Y축 (위아래) = 콜 스택 깊이
• X축 (좌우) = CPU 시간 비율 (알파벳순 정렬)
• 가로 길이가 길수록 CPU 많이 소비
• 색깔 = 함수 분류 (자바, C++ 등)
```

### 4단계: 병목 메서드 찾기

**방법 1: 직관적 검색**
```
1. 가장 넓은 스택 프레임 찾기
   → 그것이 병목!
   
2. 예: executeSQL() 너비가 전체 50%
   → CPU의 절반을 executeSQL이 소비
```

**방법 2: 마우스 호버**
```
executeSQL() 영역에 마우스 오버:
- 함수명: java.sql.Statement.executeQuery()
- 샘플 수: 5000/10000 (50%)
- CPU 시간: 50% (추정)
```

**방법 3: 다양한 정렬**
```bash
# 실행 시간(CPU) 순서로 정렬
./profiler.sh -d 60 -f flamegraph.html --cstack dwarf 12345

# 메모리 할당 기준 정렬
./profiler.sh -d 60 -f memory.html -e alloc 12345

# Lock 경합 기준 정렬
./profiler.sh -d 60 -f locks.html -e lock 12345
```

---

## 🔬 내부 동작 원리

### async-profiler의 작동 방식

```
┌─────────────────────────────────────────────────────────────┐
│ async-profiler 샘플링 메커니즘                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1ms 간격으로 (설정 가능):                                    │
│   ↓                                                          │
│ "현재 CPU에서 실행 중인 스레드의 스택은?" 캡처              │
│   ↓                                                          │
│ 스택 프레임 저장                                            │
│   ↓                                                          │
│ 같은 스택이 여러 번 나타나면 집계                           │
│   ↓                                                          │
│ Flame Graph 생성                                            │
│   (스택 프레임 너비 = 샘플 수)                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Wall-Clock vs CPU 프로파일링

```
┌─────────────────────────────────────────┐
│ Wall-Clock 프로파일링                     │
│ (모든 시간 측정)                         │
├─────────────────────────────────────────┤
│ • I/O 대기 포함                          │
│ • Lock 대기 포함                         │
│ • 네트워크 대기 포함                     │
│ • 실제 경과 시간 = 정확함                │
│ • 명령: -e wall                         │
│ • 용도: 전체 지연 분석                   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ CPU 프로파일링 (기본값)                  │
│ (CPU 사용 시간만 측정)                   │
├─────────────────────────────────────────┤
│ • I/O 대기 제외                          │
│ • Lock 대기 제외                         │
│ • 순수 CPU 시간만                        │
│ • 명령: -e cpu (또는 기본값)             │
│ • 용도: CPU 병목 분석                    │
└─────────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: 간단한 CPU 병목 찾기

```java
// BottleneckApp.java
public class BottleneckApp {
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000); // 프로파일러 준비
        
        for (int i = 0; i < 10; i++) {
            processRequest(i);
        }
    }
    
    static void processRequest(int id) {
        long start = System.nanoTime();
        
        // Step 1: 데이터 읽기 (빠름)
        byte[] data = readData();
        
        // Step 2: 계산 (느림)
        int result = expensiveCalculation(data);
        
        // Step 3: I/O 쓰기 (중간)
        writeResult(result);
        
        long elapsed = (System.nanoTime() - start) / 1000000;
        System.out.printf("요청 %d 완료: %dms%n", id, elapsed);
    }
    
    static byte[] readData() {
        // 간단한 작업
        return new byte[10000];
    }
    
    // CPU 집약적
    static int expensiveCalculation(byte[] data) {
        int sum = 0;
        for (int i = 0; i < 100000000; i++) {
            sum += Math.sqrt(i * i + 1);
        }
        return sum;
    }
    
    static void writeResult(int result) {
        // 간단한 작업
        System.out.println("결과: " + result);
    }
}
```

**컴파일 및 실행:**
```bash
javac BottleneckApp.java
java BottleneckApp &
JVM_PID=$!

# JVM 프로세스 ID 확인
sleep 3
jps -l | grep BottleneckApp

# 프로파일링 (30초)
./profiler.sh -d 30 -f bottleneck.html 12345

# HTML 파일로 확인
# expensiveCalculation() 너비가 대부분을 차지할 것
```

**기대 결과:**
```
Flame Graph에서:
┌─────────────────────────────────────────────┐
│ main                                        │
│ ├─ processRequest  (약 80% 너비)            │
│ │  ├─ readData  (약 5% 너비)               │
│ │  ├─ expensiveCalculation  (약 70%)  ← 병목│
│ │  └─ writeResult  (약 5%)                 │
│ └─ (기타)                                   │
└─────────────────────────────────────────────┘
```

---

### 실험 2: 메모리 할당 프로파일링

```java
// MemoryAllocationProfiler.java
import java.util.*;

public class MemoryAllocationProfiler {
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000);
        
        for (int i = 0; i < 10; i++) {
            allocateInMethodA();
            allocateInMethodB();
            allocateInMethodC();
            Thread.sleep(500);
        }
    }
    
    // 많은 작은 객체 할당
    static void allocateInMethodA() {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            list.add(i);
        }
    }
    
    // 적은 수의 큰 객체 할당
    static void allocateInMethodB() {
        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            list.add(new byte[1024 * 1024]); // 1MB
        }
    }
    
    // 중간 정도 할당
    static void allocateInMethodC() {
        List<String> list = new ArrayList<>();
        for (int i = 0; i < 50000; i++) {
            list.add("String_" + i);
        }
    }
}
```

**프로파일링:**
```bash
javac MemoryAllocationProfiler.java
java MemoryAllocationProfiler &

# 메모리 할당 프로파일링
./profiler.sh -d 30 -f memory-alloc.html -e alloc 12345
```

**결과 분석:**
```
각 메서드별 메모리 할당량:
- allocateInMethodB: 100MB (큰 배열들)
- allocateInMethodA: 400KB (Integer 객체들)
- allocateInMethodC: 200KB (String 객체들)

→ allocateInMethodB가 메모리 가장 많이 할당
```

---

### 실험 3: Lock 경합 분석

```java
// LockProfiler.java
import java.util.concurrent.*;

public class LockProfiler {
    static class SafeCounter {
        private int count = 0;
        
        synchronized void increment() {
            count++;
            // 의도적으로 시간 소비
            try { Thread.sleep(1); } catch (InterruptedException e) {}
        }
        
        synchronized int getCount() {
            return count;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(2000);
        
        SafeCounter counter = new SafeCounter();
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        // 10개 스레드에서 카운터 증가
        for (int t = 0; t < 10; t++) {
            executor.submit(() -> {
                for (int i = 0; i < 1000; i++) {
                    counter.increment();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(60, TimeUnit.SECONDS);
        System.out.println("최종: " + counter.getCount());
    }
}
```

**프로파일링:**
```bash
javac LockProfiler.java
java LockProfiler &

# Lock 경합 프로파일링
./profiler.sh -d 30 -f locks.html -e lock 12345
```

**결과:**
```
Lock 경합:
- SafeCounter.increment(): Lock 대기 9000ms 이상
  (10개 스레드가 같은 Lock을 놓고 경합)

→ Lock-free 자료구조나 AtomicInteger로 변경 필요
```

---

## 📊 성능 비교

| 측정 항목 | 프로파일링 없음 | async-profiler | JFR | 전통 프로파일링 |
|---------|------------|-------------|-----|-----------|
| 처리량 | 10,000 op/s | 9,900 op/s | 9,900 op/s | 5,000 op/s |
| 오버헤드 | 0% | < 1% | < 1% | 50%+ |
| 분석 정확도 | - | 매우 높음 | 높음 | 낮음 |
| 시각화 | 없음 | Flame Graph | JMC | 없음 |
| 메서드 수준 병목 | 불가 | 가능 | 제한적 | 가능 (느림) |
| 메모리 할당 | 불가 | 가능 | 가능 | 제한적 |
| 설정 복잡도 | - | 낮음 | 중간 | 높음 |

---

## ⚖️ 트레이드오프

### 프로파일링 도구 선택

```
┌──────────────────────────────────────────────┐
│ async-profiler + Flame Graph                 │
├──────────────────────────────────────────────┤
│ 장점:                                        │
│  • 메서드 단위 병목 정확히 파악              │
│  • 시각적으로 직관적                        │
│  • 설정 간단                                 │
│  • 프로덕션 안전 (오버헤드 < 1%)            │
│                                              │
│ 단점:                                        │
│  • 실행 중인 JVM에만 적용                    │
│  • 사전 계획 필요 (언제 프로파일할지)      │
│  • 짧은 시간 샘플링 가능                     │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ JFR                                          │
├──────────────────────────────────────────────┤
│ 장점:                                        │
│  • 프로덕션에서 항상 기록 가능               │
│  • 나중에 분석 가능                          │
│  • 다양한 이벤트 종류 (GC, Lock, I/O 등)   │
│  • 표준 도구 (JMC)                          │
│                                              │
│ 단점:                                        │
│  • 설정 복잡                                 │
│  • 메서드 수준 분석 제한적                   │
│  • 파일 용량 커짐                            │
└──────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | 설명 |
|------|------|
| **설치** | `async-profiler-2.9-linux-x64.tar.gz` 다운로드 |
| **CPU 프로파일링** | `./profiler.sh -d 60 -f flame.html -e cpu PID` |
| **메모리 프로파일링** | `./profiler.sh -d 60 -f memory.html -e alloc PID` |
| **Lock 프로파일링** | `./profiler.sh -d 60 -f locks.html -e lock PID` |
| **Flame Graph 읽기** | X축=CPU 시간, Y축=콜 스택 깊이 |
| **병목 찾기** | 가장 넓은 스택 프레임 = 병목 메서드 |
| **오버헤드** | < 1% (프로덕션 안전) |
| **결과 형식** | HTML (인터랙티브) |

---

## 🤔 생각해볼 문제

**Q1: 다음 Flame Graph 해석을 하세요**

```
main (너비: 100%)
├─ request() (너비: 80%)
│  ├─ databaseQuery() (너비: 60%)
│  │  ├─ executeSQL() (너비: 40%)
│  │  └─ dataMapping() (너비: 20%)
│  ├─ caching() (너비: 10%)
│  └─ logging() (너비: 10%)
└─ cleanup() (너비: 20%)
```

어느 메서드를 최적화해야 하나요?

<details>
<summary>💡 해설</summary>

**우선순위:**

1. **1순위: executeSQL() (너비 40%)**
   - CPU의 40%를 소비
   - 데이터베이스 쿼리 최적화
   - 인덱스 추가, 쿼리 튜닝

2. **2순위: dataMapping() (너비 20%)**
   - CPU의 20%를 소비
   - ORM 매핑 최적화
   - ResultSet 처리 개선

3. **3순위: caching() + logging() (각각 10%)**
   - 작은 개선은 큰 효과 없음
   - 필요시만 최적화

**기대 효과:**
- executeSQL() 50% 개선 → 전체 20% 성능 향상
- dataMapping() 50% 개선 → 전체 10% 성능 향상

</details>

---

**Q2: Wall-Clock vs CPU 프로파일링 결과가 다른 이유는?**

```
상황:
- CPU 프로파일링: databaseQuery() (너비: 5%)
- Wall-Clock 프로파일링: databaseQuery() (너비: 70%)

왜 차이가 날까?
```

<details>
<summary>💡 해설</summary>

**원인: databaseQuery()가 I/O 대기 중**

```
실제 실행:
databaseQuery() (700ms 전체)
├─ 네트워크 요청 전송: 1ms (CPU 사용)
├─ 데이터베이스 처리 대기: 500ms (CPU 미사용)
├─ 네트워크 응답 대기: 100ms (CPU 미사용)
├─ 결과 파싱: 99ms (CPU 사용)
└─ 총: 700ms 경과, 실제 CPU 사용: 100ms
```

**CPU 프로파일링:**
- CPU에서 실행 중인 시간만 기록
- 결과: 100ms ≈ 5% (전체 중)

**Wall-Clock 프로파일링:**
- 경과 시간 전부 기록
- 결과: 700ms ≈ 70% (전체 중)

**결론:**
- CPU가 높지 않지만 전체 시간이 오래 걸림
- → 데이터베이스 성능 문제
- → SQL 최적화, 인덱스 추가, 커넥션 풀 튜닝 필요

</details>

---

**Q3: async-profiler로 다음 문제를 진단하세요**

```
상황:
- 애플리케이션 처리량: 100 req/s
- 응답시간 p99: 5000ms (목표: < 500ms)
- CPU 사용률: 20%
- 메모리 사용률: 80%

어떤 프로파일링부터 시작하겠습니까?
```

<details>
<summary>💡 해설</summary>

**상황 분석:**
- CPU는 여유있음 (20%)
- 메모리는 부족 (80%)
- 응답시간이 매우 김 (5초)
- → **GC 또는 메모리 할당이 병목**

**조사 순서:**

1. **Wall-Clock 프로파일링 (우선)**
```bash
./profiler.sh -d 60 -f wallclock.html -e wall PID
```
→ I/O, GC, Lock 대기 시간 확인

2. **메모리 할당 프로파일링**
```bash
./profiler.sh -d 60 -f memory.html -e alloc PID
```
→ 누가 메모리를 과도하게 할당하는가?

3. **GC 로그 확인**
```bash
# GC 로그 확인 (별도 옵션)
tail -f logs/gc.log
```

**예상 결과:**
```
Wall-Clock Flame Graph:
메모리 할당 부분이 매우 넓음 (30~40%)

→ 메모리 누수 또는 과도한 할당
→ 힙 부족 → GC 자주 발생 → 응답시간 증가

**해결책:**
1. 메모리 할당 최소화
2. 힙 크기 증가
3. GC 튜닝 (다음 문서에서 상세)
```

</details>

---

<div align="center">

**[⬅️ 이전: Java Flight Recorder(JFR) — 프로덕션 안전 사용법](./02-java-flight-recorder.md)** | **[홈으로 🏠](../README.md)** | **[다음: Heap Dump 분석 — 메모리 누수 원인 특정 ➡️](./04-heap-dump-analysis.md)**

</div>
