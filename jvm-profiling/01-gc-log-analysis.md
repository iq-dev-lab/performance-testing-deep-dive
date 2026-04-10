# 01. GC 로그 분석 — Stop-The-World 시간 측정

---

## 🎯 핵심 질문

**"우리 애플리케이션은 왜 갑자기 느려질까? 어느 순간 응답 시간이 500ms 이상 지연되는데, 로그에는 아무것도 없다."**

이 현상은 대부분 **GC(Garbage Collection)의 Stop-The-World** 때문입니다. GC 로그를 제대로 읽을 수 있다면, 성능 저하의 원인을 정확히 파악할 수 있습니다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 애플리케이션에서 **메모리 할당과 해제**는 자동으로 일어나기 때문에, 개발자는 GC의 존재를 무시하기 쉽습니다. 하지만:

- **한 번의 Full GC가 수 초간의 응답 지연**을 초래합니다
- **GC 튜닝 없이는 아무리 좋은 코드도 성능이 제한**됩니다
- **프로덕션 장애의 30~40%가 GC 튜닝 부재**로 발생합니다

GC 로그 분석은 다음을 가능하게 합니다:
1. GC 빈도와 지연 시간 정량화
2. Minor GC / Major GC / Full GC 구분
3. 메모리 누수 vs 정상적인 메모리 사용 판별
4. GC 튜닝의 우선순위 결정

---

## 😱 흔한 실수 (Before)

```java
// ❌ GC 로그를 수집하지 않음
java -Xmx2g -Xms2g MyApplication
// 결과: 성능 저하 원인을 알 수 없음
```

```bash
# ❌ 불완전한 GC 로그 옵션
java -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
  -Xlog:gc:logs/gc.log MyApplication
# 결과: 현대식 로그 포맷이 혼재되어 분석 어려움
```

**문제점:**
- GC 로그가 없어서 병목을 추측만 함
- 여러 JVM 버전의 로그 포맷이 섞임
- 타임스탐프가 없어서 시간 상관관계 파악 불가
- 로그 로테이션으로 중요한 정보 손실

---

## ✨ 올바른 접근 (After)

### 1단계: 올바른 GC 로그 옵션 설정

Java 9+ (권장):
```bash
java -Xmx2g -Xms2g \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags:filecount=10,filesize=100m \
  -Xlog:gc+ergo*=trace:file=logs/gc-ergo.log \
  MyApplication
```

Java 8:
```bash
java -Xmx2g -Xms2g \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -XX:+PrintGCApplicationStoppedTime \
  -XX:+PrintGCApplicationConcurrentTime \
  -Xloggc:logs/gc.log \
  -XX:+UseGCLogFileRotation \
  -XX:NumberOfGCLogFiles=10 \
  -XX:GCLogFileSize=100m \
  MyApplication
```

### 2단계: 실제 GC 로그 수집 예시

```
[0.123s][info][gc,start     ] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
[0.156s][info][gc,start     ] GC(1) Pause Young (Concurrent Start) (G1 Evacuation Pause)
[0.187s][info][gc          ] GC(1) Pause Young (Concurrent Start) (G1 Evacuation Pause) 31M->15M(2048M) 34.123ms
[0.450s][info][gc,start     ] GC(2) Pause Young (Normal) (G1 Evacuation Pause)
[0.523s][info][gc          ] GC(2) Pause Young (Normal) (G1 Evacuation Pause) 512M->256M(2048M) 73.456ms
[2.100s][info][gc,start     ] GC(3) Pause Full (System.gc())
[5.234s][info][gc          ] GC(3) Pause Full (System.gc()) 1024M->512M(2048M) 3134.567ms
```

### 3단계: GC 로그 해석

```
[0.187s]                          — 애플리케이션 시작 후 187ms 경과
[info][gc]                        — 정보 수준, GC 관련 로그
GC(1)                             — GC 이벤트 번호 1
Pause Young (Concurrent Start)    — Young Generation 수집, Concurrent Mark 시작
31M->15M                          — 수집 전 31MB -> 수집 후 15MB
(2048M)                           — 전체 힙 크기 2GB
34.123ms                          — Stop-The-World 시간: 34.123ms
```

---

## 🔬 내부 동작 원리

### GC 종류별 Stop-The-World 시간

```
┌─────────────────────────────────────────────────────────────────┐
│ Minor GC (Young Generation 수집)                                 │
├─────────────────────────────────────────────────────────────────┤
│ • 빈도: 수 초~수십 초마다                                          │
│ • STW 시간: 10~100ms (일반적)                                     │
│ • 영향: 보통 무시할 수 있는 수준                                   │
│ • 이유: Young Gen은 작음 (256MB~512MB)                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Major GC (Old Generation 수집)                                   │
├─────────────────────────────────────────────────────────────────┤
│ • 빈도: 수분~수십분마다                                            │
│ • STW 시간: 100ms~수 초                                          │
│ • 영향: 사용자가 느낄 수 있는 지연                                 │
│ • 이유: Old Gen은 크고 복잡한 관계도 추적                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Full GC (전체 힙 수집)                                           │
├─────────────────────────────────────────────────────────────────┤
│ • 빈도: 비정상 (메모리 누수의 신호)                               │
│ • STW 시간: 수 초~수십 초                                        │
│ • 영향: 심각한 응답 시간 저하, 타임아웃                          │
│ • 이유: 전체 메모리를 압축해야 함                                 │
└─────────────────────────────────────────────────────────────────┘
```

### GC 큐 작동 원리

```
Young Generation (Eden + Survivor):
┌─────────────────────────────────────┐
│ 신규 객체 할당 → 빠르게 가득참 → Minor GC │
│                                      │
│ 생존 객체 → Old Generation 승격      │
└─────────────────────────────────────┘
                    ↓
Old Generation:
┌──────────────────────────────────────────┐
│ 승격된 객체 누적 → 느리게 가득참 → Major GC │
│                                          │
│ 여전히 가득하면 → Full GC (System.gc())  │
└──────────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: 정상적인 GC 패턴 관찰

```java
// NormalGCPatternApp.java
public class NormalGCPatternApp {
    static class DataHolder {
        byte[] data = new byte[1024 * 1024]; // 1MB 객체
    }

    public static void main(String[] args) throws InterruptedException {
        List<DataHolder> list = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            // 매 반복마다 1MB 객체 생성
            list.add(new DataHolder());
            
            // 일부만 유지 (나머지는 GC 대상)
            if (list.size() > 100) {
                list.remove(0);
            }
            
            if (i % 100 == 0) {
                System.out.println("반복: " + i);
            }
            
            Thread.sleep(100);
        }
    }
}
```

**실행:**
```bash
java -Xmx512m -Xms512m \
  -Xlog:gc*:file=logs/normal.log:time,uptime,level,tags \
  NormalGCPatternApp
```

**출력 분석:**
```
[0.234s][info][gc,start     ] GC(0) Pause Young (Normal)
[0.245s][info][gc          ] GC(0) Pause Young (Normal) 45M->20M(512M) 11.234ms
[1.456s][info][gc,start     ] GC(1) Pause Young (Normal)
[1.467s][info][gc          ] GC(1) Pause Young (Normal) 68M->30M(512M) 11.345ms
[2.789s][info][gc,start     ] GC(2) Pause Young (Normal)
[2.800s][info][gc          ] GC(2) Pause Young (Normal) 82M->35M(512M) 11.567ms
```

**해석:** 10~12ms의 Minor GC가 규칙적으로 반복 → 정상 패턴

---

### 실험 2: 메모리 누수 GC 패턴

```java
// MemoryLeakApp.java
public class MemoryLeakApp {
    static List<byte[]> leakedMemory = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 100000; i++) {
            // 매번 1MB를 할당하지만 절대 해제하지 않음
            byte[] data = new byte[1024 * 1024];
            leakedMemory.add(data);
            
            if (i % 100 == 0) {
                System.out.println("할당: " + (i / 100) * 100 + "MB");
            }
            
            Thread.sleep(10);
        }
    }
}
```

**실행:**
```bash
java -Xmx2g -Xms2g \
  -Xlog:gc*:file=logs/leak.log:time,uptime,level,tags \
  MemoryLeakApp
```

**출력 분석:**
```
[0.345s][info][gc,start     ] GC(0) Pause Young (Normal) 
[0.356s][info][gc          ] GC(0) Pause Young (Normal) 50M->45M(2048M) 11.234ms
[1.234s][info][gc,start     ] GC(1) Pause Young (Normal)
[1.245s][info][gc          ] GC(1) Pause Young (Normal) 200M->190M(2048M) 11.567ms
[2.456s][info][gc,start     ] GC(2) Pause Young (Normal)
[2.467s][info][gc          ] GC(2) Pause Young (Normal) 500M->480M(2048M) 12.345ms
[3.100s][info][gc,start     ] GC(3) Pause Full (Allocation Failure)
[8.234s][info][gc          ] GC(3) Pause Full (Allocation Failure) 1800M->1200M(2048M) 5134.567ms
```

**해석:** 
- Young Gen 메모리 사용량이 계속 증가 (45→190→480M)
- Full GC 발생 (정상이 아님)
- STW 시간이 급증 (11ms → 5134ms)

---

## 📊 성능 비교

| 지표 | 정상 애플리케이션 | 메모리 누수 | 최악의 경우 |
|------|-------------------|-----------|-----------|
| Minor GC 빈도 | 10~30초마다 | 2~5초마다 | 1초마다 |
| Minor GC STW | 10~50ms | 50~200ms | 500ms+ |
| Major GC 빈도 | 5~10분마다 | 1~2분마다 | 몇 초마다 |
| Major GC STW | 100~500ms | 500ms~2s | 5s~10s |
| Full GC 발생 | 0 (정상) | 1~2회/시간 | 10회+/시간 |
| Full GC STW | - | 1~5s | 10s~30s |
| 전체 GC 시간 비율 | <1% | 5~10% | 30~50%+ |
| 응답시간 p99 | <100ms | 100~500ms | 1s~10s |

---

## ⚖️ 트레이드오프

### GC 로그 상세도 vs 오버헤드

```
┌─────────────────────────────────────────────────────────────┐
│ 최소한 (간략)                                               │
│ -Xlog:gc:file=logs/gc.log                                   │
│ 장점: 오버헤드 < 0.1%                                       │
│ 단점: 상세한 분석 불가능                                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 권장 수준 (균형)                                            │
│ -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags          │
│ 장점: 오버헤드 ~0.5%, 충분한 정보                           │
│ 단점: 약간의 성능 영향                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 고상세 (디버깅)                                             │
│ -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags,pid      │
│ +all detailed GC options                                    │
│ 장점: 완전한 분석 가능                                       │
│ 단점: 오버헤드 1~2% (프로덕션 비권장)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

| 항목 | 설명 |
|------|------|
| **GC 로그 수집** | `-Xlog:gc*:file=logs/gc.log:time,uptime,level,tags` 필수 |
| **Minor GC** | Young Gen 수집, 10~50ms, 정상적 |
| **Major GC** | Old Gen 수집, 100ms~2s, 주의 필요 |
| **Full GC** | 전체 힙 수집, 수 초~수십 초, 매우 위험 |
| **STW 측정** | 로그의 마지막 숫자 (ms 단위) |
| **로그 분석 도구** | GCViewer, GCEasy, 직접 분석 |
| **우선순위** | Full GC 제거 > Major GC 최소화 > Minor GC 튜닝 |

---

## 🤔 생각해볼 문제

**Q1: 다음 GC 로그에서 문제점을 찾아보세요**

```
[2.123s][info][gc] GC(0) Pause Young (Normal) 512M->256M(2048M) 45.234ms
[2.200s][info][gc] GC(1) Pause Young (Normal) 512M->256M(2048M) 43.567ms
[2.300s][info][gc] GC(2) Pause Young (Normal) 512M->256M(2048M) 44.890ms
[2.400s][info][gc] GC(3) Pause Young (Normal) 512M->256M(2048M) 46.123ms
[2.500s][info][gc] GC(4) Pause Full (Allocation Failure) 1500M->800M(2048M) 3456.789ms
```

<details>
<summary>💡 해설</summary>

문제점:
1. **Young GC가 너무 빈번** (100ms마다) → 메모리 할당 속도가 너무 빠름
2. **Young GC STW가 큼** (40~46ms) → Young Gen이 너무 큼 (기본값 문제)
3. **Full GC 발생** → Old Gen 공간 부족 (메모리 누수 또는 힙 부족)

해결방법:
- Young Gen 크기 조정: `-XX:NewSize=256m -XX:MaxNewSize=256m`
- 힙 크기 증가: `-Xms4g -Xmx4g`
- 코드 검토: 객체 할당 최소화

</details>

---

**Q2: 다음 애플리케이션에서 어떤 GC 옵션을 설정하겠습니까?**

```
설정:
- 응답시간 p99: < 200ms 필수
- 처리량: 10,000 req/s
- 힙: 8GB
- 특성: 트레이딩 시스템 (지연 매우 민감)
```

<details>
<summary>💡 해설</summary>

**권장 설정:**

```bash
java -Xmx8g -Xms8g \
  -XX:+UseZGC \
  -XX:+UnlockDiagnosticVMOptions \
  -XX:ZUncommitDelay=300 \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  MyTradingApp
```

**이유:**
1. **ZGC 선택** → STW < 1ms (p99 200ms 달성 가능)
2. **힙 크기 동일** → 리사이징 오버헤드 제거
3. **상세 로그** → 모니터링 필수

**G1GC 대안:**
```bash
java -Xmx8g -Xms8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:+PrintGCDetails \
  -Xlog:gc*:file=logs/gc.log:time,uptime,level,tags \
  MyTradingApp
```

</details>

---

**Q3: 다음 시나리오에서 메모리 누수를 의심해야 하는 이유를 설명하세요**

```
시나리오:
- 애플리케이션 실행 1시간
- Heap 사용량: 10MB → 50MB → 200MB → 800MB → 1900MB (계속 증가)
- Young GC STW: 10ms → 20ms → 50ms → 100ms (점점 증가)
- Full GC 발생: 5회/시간
```

<details>
<summary>💡 해설</summary>

**메모리 누수의 신호:**

1. **Heap 사용량이 계속 증가**
   - 정상: Young Gen 회전, Old Gen 안정
   - 누수: Old Gen이 계속 증가 (해제되지 않음)

2. **Young GC STW가 증가**
   - 정상: 10~20ms 유지
   - 누수: 할당 속도가 빠르므로 GC 더 자주 필요

3. **Full GC가 자주 발생**
   - 정상: 거의 없음 (0~1회/일)
   - 누수: Old Gen 부족으로 강제 수집

**확인 방법:**
```bash
# Heap dump 수집
jmap -dump:format=b,file=heap.hprof PID

# MAT로 분석 (다음 문서에서 상세 설명)
```

**원인:**
- 캐시가 계속 누적
- 리스너/콜백 미등록
- 글로벌 컬렉션에 무한 추가
- 정적 필드 참조 누수

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Java Flight Recorder(JFR) — 프로덕션 안전 사용법 ➡️](./02-java-flight-recorder.md)**

</div>
