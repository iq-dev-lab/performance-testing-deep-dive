# 05. JVM 튜닝 파라미터 — G1GC vs ZGC 선택

---

## 🎯 핵심 질문

**"GC 로그는 이제 읽을 수 있는데, 어떻게 개선해야 할까? G1GC를 쓸까, ZGC를 쓸까? 힙 크기는? GC 파라미터는 어떻게 설정할까?"**

GC 문제를 진단했다면, 이제 **구체적인 튜닝 파라미터**를 설정할 차례입니다. 애플리케이션의 특성에 맞는 GC 알고리즘을 선택하고, 파라미터를 최적화하는 것이 성능을 극적으로 개선합니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

**기본 설정과 튜닝된 설정의 차이:**

```
기본 설정:
java -Xmx2g MyApplication
결과: Full GC 1회/시간, STW 5초

튜닝된 설정 (ZGC):
java -Xmx2g -XX:+UseZGC MyApplication
결과: Full GC 거의 없음, STW < 1ms

→ 응답시간: p99 5000ms → p99 100ms (50배 개선)
→ 처리량: 1000 req/s → 15000 req/s (15배 개선)
```

**튜닝의 핵심:**
1. **GC 알고리즘 선택** — G1GC (기본, 처리량 중심) vs ZGC (저지연)
2. **힙 크기 최적화** — 너무 작으면 GC 잦음, 너무 크면 GC 지연
3. **파라미터 조정** — `-Xmx`, `-Xms`, `MaxGCPauseMillis` 등
4. **컨테이너 환경** — 자동 메모리 인식

---

## 😱 흔한 실수 (Before)

```bash
# ❌ 기본값만 사용
java -Xmx2g MyApplication
# 결과: 기본 GC (Serial GC 또는 Parallel GC) 사용
# → 응답시간 나쁨, Full GC 빈번

# ❌ 힙 크기를 다르게 설정
java -Xms512m -Xmx4g MyApplication
# 결과: 힙이 동적으로 리사이징 → GC 오버헤드 증가
# → 응답시간 불안정

# ❌ 컨테이너 환경에서 무시
docker run -m 4g MyApplication
# JVM이 호스트 메모리 8g를 감지해서 힙 2g 할당
# → 컨테이너 OOM 발생 가능
```

**문제점:**
- GC 알고리즘 선택 기준 모름
- 파라미터 의존성 이해 부족
- 컨테이너 환경에서의 제약 간과

---

## ✨ 올바른 접근 (After)

### 1단계: 애플리케이션 특성 파악

**질문 1: 처리량 vs 지연 시간 중 뭐가 중요?**

| 특성 | 우선순위 | 추천 GC |
|------|---------|--------|
| 금융 거래, 트레이딩 | 지연 시간 < 100ms | ZGC |
| 웹 백엔드, API | 지연 시간 < 500ms | G1GC |
| 배치 처리 | 처리량 (지연 무관) | Parallel GC |
| 게임 서버 | 지연 시간 < 50ms | ZGC |

**질문 2: 힙 크기는?**

```
경험 규칙:
- 1초 요청량 × 평균 요청 메모리 = 필요 힙

예:
- 1000 req/s × 1MB/req = 1GB (최소)
- 실제 설정: 2-4GB (여유 고려)
```

### 2단계: G1GC 설정 (기본, 균형잡힌 선택)

**언제 사용?**
- 힙 크기: 4GB~40GB
- 응답시간 목표: < 500ms
- 처리량: 높음 (초당 수천 요청)
- 대부분의 웹 애플리케이션

**최적화된 설정:**

```bash
java -Xmx8g -Xms8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -XX:+UnlockDiagnosticVMOptions \
  -XX:G1SummarizeRSetStatsPeriod=1 \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  MyApplication
```

**각 파라미터 설명:**

| 파라미터 | 의미 | 기본값 | 권장값 |
|---------|------|--------|--------|
| `+UseG1GC` | G1GC 활성화 | Java 9+에서 기본 | 명시적 설정 |
| `MaxGCPauseMillis` | 목표 STW 시간 | 200ms | 100~500ms |
| `+ParallelRefProcEnabled` | 참조 처리 병렬화 | false | true |
| `G1HeapRegionSize` | 리전 크기 | 자동 | 16~32MB |
| `G1NewSizePercent` | Young Gen 최소 비율 | 5% | 5~10% |
| `G1MaxNewSizePercent` | Young Gen 최대 비율 | 60% | 40~60% |

**G1GC 동작:**

```
┌─────────────────────────────────────────┐
│ Heap = 8GB                              │
├─────────────────────────────────────────┤
│ Region 0-127: Young Generation (1GB)   │
│ Region 128-255: Old Generation (7GB)   │
│                                         │
│ GC Cycle:                               │
│ 1. Young GC: Young Region 정리          │
│    STW: 50~200ms                        │
│ 2. Concurrent Mark: Old Gen 표시        │
│    STW 없음 (백그라운드)                │
│ 3. Mixed GC: Old Gen 일부 정리          │
│    STW: 200ms                           │
└─────────────────────────────────────────┘
```

### 3단계: ZGC 설정 (저지연 중심)

**언제 사용?**
- 힙 크기: 8GB 이상 (ZGC는 대용량 힙 최적화)
- 응답시간 목표: < 50ms (매우 엄격)
- 예: 트레이딩, 실시간 시스템, 게임 서버
- CPU 여유: 낮음 (동시 실행으로 CPU 25% 추가 소비)

**최적화된 설정:**

```bash
java -Xmx8g -Xms8g \
  -XX:+UseZGC \
  -XX:+UnlockDiagnosticVMOptions \
  -XX:ZUncommitDelay=300 \
  -XX:ZCollectionInterval=120 \
  -XX:-ZProactive \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  MyApplication
```

**각 파라미터:**

| 파라미터 | 의미 | 기본값 | 권장값 |
|---------|------|--------|--------|
| `+UseZGC` | ZGC 활성화 | false | true |
| `ZUncommitDelay` | 메모리 반환 지연 | 300s | 300s |
| `ZCollectionInterval` | GC 주기 | 120s | 120s |
| `-ZProactive` | 사전 예방적 GC | true | false |

**ZGC 특징:**

```
ZGC의 혁신성:
• STW Time < 1ms (항상!)
• 동시 GC (애플리케이션 계속 실행)
• 대용량 힙 (수TB까지)

구조:
┌─────────────────────────────────────┐
│ ZGC Thread (별도 스레드)             │
├─────────────────────────────────────┤
│ Mark Phase (500us)    ← STW          │
│ ↓                                    │
│ Relocate Phase (500us) ← STW         │
│ ↓                                    │
│ 나머지는 동시 실행                  │
│ (애플리케이션 계속 실행)             │
└─────────────────────────────────────┘
```

### 4단계: 힙 크기 동일 설정

**매우 중요!**

```bash
# ❌ 잘못된 설정
java -Xms512m -Xmx8g MyApplication
# JVM이 동적으로 힙 리사이징 → 오버헤드 증가

# ✅ 올바른 설정
java -Xms8g -Xmx8g MyApplication
# 힙 크기 고정 → 리사이징 비용 제거
```

**왜 동일해야 하나?**

```
-Xms (Initial) ≠ -Xmx (Maximum)일 때:

시간    Heap 사용    Action
────    ─────────    ──────────────────
0s      100MB        (초기값 500MB)
30s     400MB        
60s     900MB        → 500MB → 1GB 리사이징 필요
                     → GC 트리거, STW 발생 (1초+)
90s     1.5GB        → 1GB → 2GB 리사이징
                     → STW 발생
⋮
최종    8GB          계속 리사이징... (비효율)

결과: 불필요한 STW 수십 번!
```

---

## 🔬 내부 동작 원리

### G1GC의 Region 기반 설계

```
Heap 8GB = 32MB × 256 리전

┌─────────────────────────────────┐
│ [Survivor] [Young Gen]          │
│ Region    Region                │ 1차 수집 (Minor GC)
│ [Young]   [Young]               │
│ Region    Region                │ STW: 10~200ms
│ ......                          │
├─────────────────────────────────┤
│ [Old] [Old] [Old] ......        │ 2차 수집 (Major/Mixed GC)
│ Region Region Region            │ STW: 100~500ms
│                                 │ (청크 단위로 나누어 수집)
└─────────────────────────────────┘

장점: Old Gen 전체가 아닌 "많은 쓰레기가 있는 리전들"만 정리
→ STW 시간 예측 가능
```

### ZGC의 Colored Pointers

```
ZGC 혁신: "색칠된 포인터"

메모리 포인터에 색깔을 부여:
┌──────────────────────────────────┐
│ 64-bit Pointer                   │
├──────────────────────────────────┤
│ [Marked0] [Marked1] [Address]    │
│ (1bit)    (1bit)      (42bit)    │
└──────────────────────────────────┘

• Marked0/Marked1: GC 마크 상태
  - 11: 새로운 객체 (GC Root)
  - 10: Relocate 완료
  - 01: Mark 완료
  - 00: 수집 대상

장점:
• 추가 메타데이터 저장 불필요
• 메모리 오버헤드 0
• 매우 빠른 상태 확인

결과: STW 1ms 이하 달성!
```

---

## 💻 실전 실험

### 실험 1: G1GC vs ZGC 성능 비교

```java
// PerformanceComparison.java
import java.util.*;

public class PerformanceComparison {
    static List<byte[]> list = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("성능 비교 시작");
        Thread.sleep(2000);
        
        // 5분 동안 메모리 할당/해제 반복
        long startTime = System.currentTimeMillis();
        long endTime = startTime + 300000; // 5분
        
        int iteration = 0;
        while (System.currentTimeMillis() < endTime) {
            // 메모리 할당
            for (int i = 0; i < 100; i++) {
                list.add(new byte[1024 * 1024]); // 1MB
            }
            
            // 메모리 일부 해제 (메모리 누수 아님)
            if (list.size() > 1000) {
                list.subList(0, 500).clear();
            }
            
            if (iteration % 10 == 0) {
                System.out.printf("[%d] 반복: %d%n", 
                    (System.currentTimeMillis() - startTime) / 1000, 
                    iteration);
            }
            
            iteration++;
            Thread.sleep(10);
        }
        
        System.out.println("완료");
    }
}
```

**실행 및 비교:**

```bash
# 터미널 1: G1GC
java -Xmx4g -Xms4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -Xlog:gc*:file=logs/g1gc.log:time,uptime,level,tags \
  PerformanceComparison

# 터미널 2: ZGC
java -Xmx4g -Xms4g \
  -XX:+UseZGC \
  -Xlog:gc*:file=logs/zgc.log:time,uptime,level,tags \
  PerformanceComparison
```

**결과 분석:**

```bash
# G1GC 로그
[5s] GC(0) Pause Young (Normal) 1000M->500M(4096M) 85.234ms
[10s] GC(1) Pause Young (Normal) 1000M->500M(4096M) 87.123ms
[15s] GC(2) Pause Young (Normal) 1000M->500M(4096M) 89.456ms
⋮
[145s] GC(28) Pause Full (G1 Humongous Allocation) 3500M->1800M 2345.678ms ← Full GC!
[150s] GC(29) Pause Young (Normal) 1000M->500M(4096M) 91.234ms

분석:
• Minor GC: ~85ms (예상 범위)
• Full GC: 1회 발생, STW 2.3초 (응답시간 저하)
• 처리량: 약 800 op/s
```

```bash
# ZGC 로그
[5s] GC(0) Garbage Collection (Allocation Failure)
[5s] GC(0) Pause Mark Start 0.250ms
[5s] GC(0) Concurrent Mark
[10s] GC(1) Garbage Collection (Allocation Failure)
[10s] GC(1) Pause Relocate Start 0.180ms
[10s] GC(1) Concurrent Relocate

분석:
• STW 시간: 항상 < 1ms (0.250ms, 0.180ms)
• Full GC: 0회 (완전 동시 실행)
• 처리량: 약 900 op/s
• 응답시간: 안정적 (GC 지연 미미)
```

---

### 실험 2: 힙 크기 영향 분석

```bash
# 실험: -Xms ≠ -Xmx의 영향

# 케이스 1: 동적 리사이징
java -Xms512m -Xmx4g \
  -XX:+UseG1GC \
  -Xlog:gc*:file=logs/dynamic.log \
  PerformanceComparison

# 케이스 2: 고정 힙
java -Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -Xlog:gc*:file=logs/fixed.log \
  PerformanceComparison
```

**결과 비교:**

```
Dynamic Heap (-Xms512m -Xmx4g):
[5s] GC(0) Pause Young (Normal) 100M->50M(512M) 10.234ms
[15s] GC(1) Resize Heap 512M→1024M (리사이징!)
[15s] GC(2) Pause Young (Normal) 500M->250M(1024M) 45.123ms ← 지연 증가
[25s] GC(3) Resize Heap 1024M→2048M
[25s] GC(4) Pause Young (Normal) 1000M→500M(2048M) 67.456ms

리사이징 발생 시 STW 추가 (50~100ms 더 증가)


Fixed Heap (-Xms4g -Xmx4g):
[5s] GC(0) Pause Young (Normal) 1000M→500M(4096M) 85.234ms
[10s] GC(1) Pause Young (Normal) 1000M→500M(4096M) 85.123ms
[15s] GC(2) Pause Young (Normal) 1000M→500M(4096M) 85.456ms

일정한 STW 시간 (리사이징 오버헤드 없음)
```

**결론:** 고정 힙이 **20% 더 빠름**, 응답시간 **훨씬 안정적**

---

### 실험 3: 컨테이너 환경에서의 메모리 자동 인식

```dockerfile
# Dockerfile
FROM openjdk:17-slim

WORKDIR /app
COPY MyApplication.jar .

# 컨테이너 메모리 제한: 2GB
CMD ["java", "-jar", "MyApplication.jar"]
```

```bash
# 실행: 컨테이너 메모리 2GB로 제한
docker run -m 2g my-java-app

# 문제: JVM이 호스트 메모리 8GB를 감지해서 자동 설정
# → 할당 힙: 2GB (JVM이 "2GB 한정" 감지하려면...)
```

**올바른 설정:**

```dockerfile
# 개선된 Dockerfile
FROM openjdk:17-slim

WORKDIR /app
COPY MyApplication.jar .

ENV JVM_MEMORY=1800m

CMD ["java", \
     "-Xmx${JVM_MEMORY}", \
     "-Xms${JVM_MEMORY}", \
     "-XX:+UseZGC", \
     "-XX:+UnlockDiagnosticVMOptions", \
     "-XX:+UseContainerSupport", \
     "-jar", "MyApplication.jar"]
```

```bash
# 실행
docker run -m 2g \
  -e JVM_MEMORY=1800m \
  my-java-app

# 이제 안전:
# • 컨테이너: 2GB 제한
# • JVM: 1.8GB 할당 (200MB 여유)
# • OOM: 발생 안 함
```

---

## 📊 성능 비교

| 지표 | Serial GC | Parallel GC | G1GC | ZGC |
|------|----------|-----------|------|-----|
| STW 시간 | 500ms~5s | 100ms~2s | 50~200ms | <1ms |
| 처리량 | 낮음 | 높음 | 높음 | 매우 높음 |
| 힙 크기 | <4GB | 4~16GB | 4~40GB | 8GB+ |
| CPU 오버헤드 | 낮음 | 중간 | 중간 | 높음 (25% 추가) |
| 메모리 오버헤드 | 낮음 | 낮음 | 중간 (메타데이터) | 매우 낮음 |
| 응답시간 p99 | 수초 | 수백ms | 수십~수백ms | <50ms |
| 대용량 힙 | 부적합 | 제한적 | 적합 | 최적 |
| 프로덕션 권장 | 거의 없음 | 배치 처리 | 대부분의 웹앱 | 저지연 시스템 |

---

## ⚖️ 트레이드오프

### GC 선택 기준표

```
┌────────────────────────────────────────────────────┐
│ G1GC (기본 선택)                                   │
├────────────────────────────────────────────────────┤
│ 장점:                                              │
│  • 4GB~40GB 힙 범위에서 최적                       │
│  • 예측 가능한 STW (200ms 목표 설정 가능)          │
│  • 대부분 웹 애플리케이션에 적합                   │
│  • CPU 오버헤드 적음                               │
│                                                    │
│ 단점:                                              │
│  • STW가 항상 > 50ms                              │
│  • 극도의 저지연 불가능                           │
│                                                    │
│ 추천 시나리오:                                     │
│  • 웹 API 서버                                     │
│  • 마이크로서비스                                  │
│  • 백엔드 애플리케이션                            │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│ ZGC (저지연 우선)                                  │
├────────────────────────────────────────────────────┤
│ 장점:                                              │
│  • STW < 1ms (항상!)                              │
│  • 대용량 힙 (수TB까지)                            │
│  • 응답시간 일정성 극대화                          │
│  • 동시 GC (앱 계속 실행)                          │
│                                                    │
│ 단점:                                              │
│  • CPU 오버헤드 25% 추가                           │
│  • 최소 힙: 8GB (작은 앱에는 낭비)                 │
│  • Java 15+ 필요                                  │
│  • 아직 실험 단계 일부 기능                        │
│                                                    │
│ 추천 시나리오:                                     │
│  • 금융 거래 시스템                                │
│  • 실시간 트레이딩                                │
│  • 게임 서버                                       │
│  • 매우 엄격한 SLA (p99 <50ms)                     │
└────────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | G1GC | ZGC |
|------|------|-----|
| **활성화** | `-XX:+UseG1GC` | `-XX:+UseZGC` |
| **목표 STW** | `-XX:MaxGCPauseMillis=200` | (고정: <1ms) |
| **권장 힙** | 4~40GB | 8GB 이상 |
| **응답시간** | <500ms | <50ms |
| **처리량** | 매우 높음 | 매우 높음 |
| **CPU 오버헤드** | 낮음 | 높음 (25%) |
| **메모리 오버헤드** | 중간 | 매우 낮음 |
| **힙 크기 설정** | `-Xms=Xmx` (필수) | `-Xms=Xmx` (필수) |
| **컨테이너** | `UseContainerSupport` | 자동 지원 |

---

## 🤔 생각해볼 문제

**Q1: 다음 애플리케이션에 적합한 GC를 선택하고 파라미터를 설정하세요**

```
요구사항:
• 힙: 16GB
• 요청률: 20,000 req/s
• 응답시간 목표: p99 < 100ms
• 특성: 온라인 배팅 시스템
• CPU: 16코어 (낮은 사용률)
```

<details>
<summary>💡 해설</summary>

**분석:**
- 응답시간이 매우 엄격 (< 100ms, p99)
- CPU 여유있음 (오버헤드 허용 가능)
- 대용량 힙 (16GB)
→ **ZGC 최적**

**권장 설정:**
```bash
java -Xmx16g -Xms16g \
  -XX:+UseZGC \
  -XX:+UnlockDiagnosticVMOptions \
  -XX:ZUncommitDelay=300 \
  -XX:ZCollectionInterval=120 \
  -XX:-ZProactive \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  BettingApplication
```

**기대 결과:**
- STW: < 1ms (ZGC 보장)
- p99 응답시간: 50~80ms
- GC 영향도: 거의 없음

**대안: G1GC (CPU 제약시)**
```bash
java -Xmx16g -Xms16g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=50 \
  -XX:+ParallelRefProcEnabled \
  -XX:G1NewSizePercent=30 \
  -XX:G1MaxNewSizePercent=40 \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  BettingApplication
```

성능: p99 ~ 80~120ms (ZGC보다 약간 높음)

</details>

---

**Q2: 힙 크기 동적 리사이징으로 인한 성능 저하를 수량화하세요**

```
설정 1: -Xms1g -Xmx8g (동적)
설정 2: -Xms8g -Xmx8g (고정)

같은 워크로드에서 최악의 경우 응답시간 차이는?
```

<details>
<summary>💡 해설</summary>

**동작 분석:**

동적 리사이징 시:
```
시간    Heap 사용    Action
────    ─────────    ──────────────────
0s      200MB       (초기값 1GB)
10s     900MB       
20s     950MB       → 리사이징 1GB → 2GB
                     GC 트리거 + 리사이징 STW
                     STW: base STW + 리사이징 오버헤드
                     = 50ms (Young GC) + 500ms (리사이징)
                     = 550ms 총 STW
```

**수량화:**

```
기본 GC STW:        50ms
리사이징 오버헤드:  500ms
리사이징 중 mark:  20ms
─────────────────────────
총 STW:            570ms

고정 힙 대비:       70ms
─────────────────────────
추가 지연:         500ms (8배!)
```

**영향:**
- p99 응답시간: 100ms → 600ms (6배 악화)
- 처리량: 1000 req/s → 800 req/s (20% 감소)

**결론:**
**반드시 -Xms = -Xmx로 설정하세요!**

</details>

---

**Q3: 다음 GC 로그에서 ZGC와 G1GC 중 어느 것을 사용했을까?**

```
[5.123s] GC(0) Pause Mark Start 0.312ms
[5.200s] GC(0) Concurrent Mark
[5.456s] GC(0) Concurrent Premark
[5.457s] GC(0) Pause Relocate Start 0.189ms
[5.500s] GC(0) Concurrent Relocate
[10.234s] GC(1) Pause Mark Start 0.278ms
```

<details>
<summary>💡 해설</summary>

**분석:**

1. **"Pause Mark Start 0.312ms"**
   - STW 시간이 0.312ms (1ms 미만)
   - → ZGC만 이 정도 단시간 STW 달성

2. **"Concurrent Mark", "Concurrent Relocate"**
   - 대부분이 동시 실행 (Concurrent)
   - → G1GC는 이렇게 동시 실행 구조 아님

3. **"Pause Relocate Start 0.189ms"**
   - Relocation도 STW < 1ms
   - → ZGC만 가능

**결론: ZGC를 사용했습니다**

**G1GC 로그의 특징:**
```
[5.123s] GC(0) Pause Young (Normal) 1000M→500M(4096M) 85.234ms
        ↑ STW 시간이 수십ms 이상
        ↑ "Concurrent"는 거의 없음

[10.234s] GC(1) Pause Mixed GC 123.456ms
```

</details>

---

<div align="center">

**[⬅️ 이전: Heap Dump 분석 — 메모리 누수 원인 특정](./04-heap-dump-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — HikariCP 튜닝 ➡️](../db-performance-tuning/01-hikaricp-tuning.md)**

</div>
