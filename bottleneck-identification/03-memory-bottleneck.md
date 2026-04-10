# 03. 메모리 병목 분석 — JVM Heap vs Off-Heap 구분

---

## 🎯 핵심 질문

메모리 사용률 90%는 문제일까?
- Heap이 90%면: 다음 GC에서 500ms 이상 멈출 수 있음 (문제!)
- Cache buffer가 90%면: 정상 (OS가 관리)
- Off-Heap이 90%면: Native 메모리 누수 가능성 (문제!)

**메모리 병목을 판단하려면 "어느 메모리"인지 정확히 파악해야 한다.**

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
모니터링 대시보드: Memory Usage 85%
개발자 반응 1: "메모리 부족! 32GB → 64GB로 증설"
→ 비용 2배 증가, 응답시간 개선 없음
→ 진짜 원인: "GC 빈도가 초당 15회, GC pause 200ms"
→ 증설보다는 GC 튜닝이 필요했음

개발자 반응 2: "Heap 사용률 40%인데 GC가 왜?"
→ GC 로그: "Full GC 1회/분, 각 500ms"
→ Heap 40%에서 Full GC가 발생 = Promotion Failure
→ Young 영역에서 Old 영역으로 승격되는 객체가 너무 많음
→ 메모리 증설이 아니라 애플리케이션 튜닝 필요 (객체 생성 줄이기)
```

**올바른 진단:**

```
메모리 병목 판단 프로세스:
1단계: GC 로그 분석 (GC 빈도, pause time)
       → GC pause > 100ms이면 메모리 병목 가능성
       
2단계: Heap 연령 분포 확인 (jmap)
       → 큰 객체가 많으면 객체 생성 최소화 필요
       
3단계: Heap vs Off-Heap 구분
       → Heap 부족 vs Native 메모리 누수
       
4단계: 메모리 누수 확인 (힙덤프)
       → 같은 클래스의 인스턴스가 점점 늘어나면 누수
```

**실무 효과:**
- 잘못된 진단: 메모리 증설 → 비용 낭비, 문제 미해결
- 올바른 진단: 객체 생성 최소화 또는 GC 튜닝 → 비용 0, 응답시간 개선

---

## 😱 흔한 실수 (Before)

```bash
# 잘못된 접근 1: 메모리 사용률만 본다
$ free -h
#                total        used        free      shared  buff/cache   available
# Mem:           31Gi        24Gi       2.0Gi       200Mi        4.5Gi       6.0Gi
# 결론: "메모리 사용률 77%. 메모리 부족!"
# → 근데 buff/cache 4.5Gi를 뺀 실제 사용은 19.5Gi (63%)
# → 응답시간이 높은 건 메모리 때문이 아니라 GC 때문일 수 있음

# 잘못된 접근 2: GC 빈도만 본다
$ jstat -gc -h10 $(jps | grep App | awk '{print $1}') 1000
# S0C    S1C    S0U    S1U      EC        EU       OC         OU
# 1024 1024   512    512  10240      7180    20480      15360
# Minor GC 빈도: 5회/초
# 결론: "GC 많으니 Heap 크기를 늘려야 한다"
# → 근데 GC pause time은 50ms로 정상인데?
# → Heap 크기가 아니라 객체 생성 줄이기가 필요할 수도

# 잘못된 접근 3: Heap 사용률과 GC 빈도를 분리해서 본다
$ jcmd $(jps | grep App | awk '{print $1}') GC.class_histogram | head -20
# Instance Count, Total, Class Name
# 1000000, 50MB, byte[]
# 결론: "큰 객체들이 많다. Heap 증설 필요"
# → 근데 매 초마다 이 객체들이 생성되고 폐기된다면?
# → Heap 증설은 도움 X, 객체 생성 최소화가 필요

# 잘못된 접근 4: Off-Heap을 무시한다
$ jps -l
# 12345 com.example.App
$ ps -p 12345 -o vsz,rss
# VSZ: 4GB, RSS: 2GB
# 근데 "java -Xmx1.5G" 설정
# → Heap 1.5GB + Off-Heap 0.5GB = 2GB
# → 응답시간 높은 게 Off-Heap 누수일 수도?
```

**실제 사례:**

```
상황: p99 응답시간 1500ms, 메모리 사용률 85%

1차 진단 (잘못됨):
→ "메모리 부족하면 성능 저하. 메모리 증설하자."
→ 32GB → 64GB (비용 2배)

2차 성능 측정:
→ p99 응답시간 1400ms (거의 개선 없음)

3차 상세 분석 (올바름):
$ jstat -gcutil $(jps | grep App | awk '{print $1}') 1000
# S0    S1    E     O     M    CCS   YGC   YGCT    FGC   FGCT   CGC  CGCT   GCT
# 40.0  0.0 85.0  75.0 95.0 80.0  1234  12.5     15   45.0    1    0.5   58.0

→ Young GC 1234회, Cumulative Time 12.5초 (초당 1.2초 GC 중)
→ Full GC 15회, Cumulative Time 45초 (각 3초씩!)
→ 메모리 증설이 아니라 GC pause time 자체가 문제

4차 해결책:
→ GC 튜닝 (G1GC 사용, Young 영역 크기 조정)
→ 그리고 객체 생성 최소화 (객체 풀 사용)
→ 결과: GC pause 3초 → 0.1초로 감소
→ p99 응답시간 1500ms → 150ms (90% 개선!)
→ 메모리 증설 취소 (비용 절감)
```

---

## ✨ 올바른 접근 (After)

### 1단계: GC 로그 분석 (GC 빈도와 Pause Time)

```bash
# GC 로그 활성화 (JVM 시작 옵션)
# Java 8:
java -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
     -Xloggc:gc.log -XX:+UseG1GC ...

# Java 9+:
java -Xlog:gc*:gc.log ...

# 실시간 GC 모니터링
$ jstat -gcutil -h10 $(jps | grep App | awk '{print $1}') 1000
# S0     S1     E      O      M     CCS    YGC    YGCT     FGC    FGCT    CGC   CGCT      GCT
# 0.00  50.00  95.00  65.00  90.00 75.00  1000  5.20      5     2.50     0     0.00   7.70
#
# S0, S1 = Survivor 0/1 사용률 (Young 영역의 임시 저장소)
# E = Eden 사용률 (새 객체 생성 영역)
# O = Old 사용률 (오래된 객체 저장)
# YGC = Young GC 횟수, YGCT = Young GC 누적 시간
# FGC = Full GC 횟수, FGCT = Full GC 누적 시간

분석:
YGC 1000회, YGCT 5.20초 → 평균 GC pause = 5200ms / 1000 = 5.2ms
FGC 5회, FGCT 2.50초 → 평균 Full GC pause = 2500ms / 5 = 500ms

결론:
- Young GC: 평균 5.2ms (정상, 응답시간에 거의 영향 X)
- Full GC: 평균 500ms (심각!, p99에 직접 영향)
```

**GC 로그 상세 읽기:**

```
[2024-03-15T15:30:45.123+0900] GC(1234) Pause Young (G1 Evacuation Pause)
  (G1 Humongous Allocation)
  3156M->1089M(4096M) 234.567ms
  CPU 450.34ms / Real 234.57ms
  GC Worker Other: Min 0.0ms, Avg 45.2ms, Max 78.0ms

해석:
- GC 1234번 발생
- Young Generation 가비지 컬렉션 (Full GC 아님)
- 메모리: 3156MB → 1089MB (2067MB 회수)
- 멈춘 시간: 234.567ms (0.23초)
- CPU 사용률: 450ms (4 코어 기준 고루 사용)

판단:
- 234ms pause는 높음 (100ms 이상이면 이미 문제)
- 2GB 이상의 메모리가 매번 회수됨 = 객체 생성 많음
- 이 정도면 응답시간 p99에 영향 미침
```

### 2단계: Heap 연령 분포 (jmap)

```bash
# 현재 Heap에 있는 모든 객체 개수 확인
$ jmap -histo:live $(jps | grep App | awk '{print $1}') | head -30
#  num     #instances         #bytes  class name
#    1        1234567        1234567  [B (byte array)
#    2        456789          456789  java.lang.String
#    3        234567          234567  java.util.HashMap$Node
#    4        123456          123456  java.util.LinkedList$Node
#    5         89012           89012  com.example.User

분석 방법:
1. 첫 5개 클래스만 Heap의 70~80%를 차지하는가?
   → Yes: 특정 객체가 과도하게 많음 = 객체 생성 최소화 필요
   → No: 여러 종류의 객체가 고르게 분포 = 메모리 구조 정상

2. byte[]가 1위인가?
   → Yes: 바이트 배열(버퍼, 스트림 등) 많음 = Direct Buffer 재사용 고려
   → No: 일반 객체들이 주류

3. com.example.* 클래스가 많은가?
   → Yes: 애플리케이션 객체 누수 가능성 = Heap dump 분석 필요
   → No: 표준 라이브러리 객체 주류 = 정상
```

**Heap 덤프를 통한 메모리 누수 확인:**

```bash
# Heap dump 수집 (애플리케이션 멈춤)
$ jmap -dump:live,format=b,file=heap.bin $(jps | grep App | awk '{print $1}')

# 또는 자동 dump (OutOfMemory 발생 시)
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.bin \
     -jar app.jar

# Eclipse Memory Analyzer 또는 JHat으로 분석
$ jhat -J-Xmx4g heap.bin

접속: http://localhost:7000

분석:
1. "Show instance counts for all classes" 확인
2. 같은 클래스의 인스턴스 개수가 계속 증가하는가?
   예: 1시간 전 User 객체 100,000개 → 현재 500,000개
   → 메모리 누수 (누군가 reference를 계속 유지)

3. 특정 클래스의 retention size를 확인
   예: StaticCache 클래스가 전체 Heap의 40%?
   → 이 캐시를 비우거나 제한해야 함
```

### 3단계: Heap vs Off-Heap 구분

```bash
# 1. 프로세스 메모리 확인
$ ps -p $(jps | grep App | awk '{print $1}') -o vsz,rss
# VSZ: 4000000, RSS: 2000000
# VSZ = Virtual Size (할당된 모든 메모리)
# RSS = Resident Set Size (물리 메모리 사용)

# 2. JVM Heap 설정 확인
$ jcmd $(jps | grep App | awk '{print $1}') VM.flags | grep "Heap"
# -Xms1024m, -Xmx2048m
# Heap max = 2GB

# 3. 오프-힙 메모리 계산
# RSS (2000MB) - Heap (2000MB) = Off-Heap (0MB)? 오류!
# 실제로는:
# RSS = Heap (사용 중) + Code cache + Stack + Off-Heap + 기타

# 정확한 Off-Heap 측정
$ jcmd $(jps | grep App | awk '{print $1}') VM.native_memory summary

# 또는 jstat로 메모리 세부 사항 확인
$ jstat -gc $(jps | grep App | awk '{print $1}') 1000
# NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN    OGCMX     OGC     OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCS    YGC     FGCT
# 21248.0  341504.0 21248.0 2688.0 2688.0 16384.0 42496.0  683008.0 42496.0 42496.0 0.0   1048576.0 3840.0 0.0 1048576.0 512.0  1000  2.50

분석:
- NGCMN + OGCMN = Young + Old 최소값 = 63.5MB
- NGCMX + OGCMX = Young + Old 최대값 = 1GB
- MC (Metaspace Current) = 3.8MB
- CCS (Compressed Class Space) = 512KB

Off-Heap 계산:
Off-Heap = RSS - (현재 Heap 사용량)
```

### 4단계: GC 빈도 기준

```bash
# GC 빈도 분석
$ jstat -gc -h10 $(jps | grep App | awk '{print $1}') 1000 | grep YGC

정상 기준:
┌─────────────────┬──────────────┬─────────────┐
│ Young GC 빈도   │  Pause Time  │  평가       │
├─────────────────┼──────────────┼─────────────┤
│ < 1회/sec       │  < 10ms      │ 최적 (Green) │
│ 1~5회/sec       │  10~50ms     │ 좋음 (Green) │
│ 5~10회/sec      │  50~100ms    │ 경고 (Yellow)│
│ > 10회/sec      │ > 100ms      │ 심각 (Red)  │
├─────────────────┼──────────────┼─────────────┤
│ Full GC 빈도    │  Pause Time  │  평가       │
├─────────────────┼──────────────┼─────────────┤
│ 0회/시간        │  -           │ 최적        │
│ 1회/시간        │  < 500ms     │ 좋음        │
│ 1회/분          │  > 500ms     │ 심각        │
│ 1회/초 이상     │ > 1000ms     │ 매우 위험   │
└─────────────────┴──────────────┴─────────────┘
```

---

## 🔬 내부 동작 원리

### JVM Heap 구조

```
Heap (2GB max)
├─ Young Generation (33%, ~660MB)
│  ├─ Eden (80%, 530MB)
│  │  └─ 새 객체 생성 영역
│  └─ Survivor 2개 (각 10%, 65MB)
│     └─ GC 후 생존한 객체 임시 저장
│
└─ Old Generation (66%, ~1320MB)
   └─ Young에서 오래 살아남은 객체 (tenuring threshold > 15번 생존)

GC 프로세스:
1. 새 객체 → Eden에 할당
2. Eden 가득 참 → Minor GC 실행 (50~200ms)
   - 생존한 객체 → Survivor S0로 이동
   - 죽은 객체 → 제거
3. S0 가득 찬 후 → S1로 이동 (회수한 S0)
4. tenuring threshold 도달 → Old로 이동
5. Old 가득 참 → Full GC 실행 (200ms~3초)
   - 모든 세대 검사
   - Stop-The-World (애플리케이션 멈춤)
```

### GC Pause Time이 응답시간에 미치는 영향

```
요청 처리 중 Minor GC 발생:

요청 A: 시간 0 ~ 50ms (처리 완료)
요청 B: 시간 10 ~ 150ms
       ├─ 10~30ms: 정상 처리
       ├─ 30~230ms: Minor GC 실행 (200ms, Stop-The-World)
       └─ 230~250ms: 나머지 처리 완료
       
결과: 요청 B의 응답시간 = 240ms
(만약 GC가 없었다면 40ms 정도였을 것)

Full GC의 경우:
요청 C: 시간 100 ~ 3500ms
       ├─ 100~500ms: 정상 처리
       ├─ 500~3000ms: Full GC 실행 (2500ms)
       └─ 3000~3500ms: 나머지 처리 완료
       
결과: 요청 C의 응답시간 = 3400ms
(정상이라면 400ms 수준)

p99 응답시간:
- Minor GC (50ms): p99 + 50ms
- Full GC (1000ms): p99 + 1000ms
- Full GC 빈도가 높으면 응답시간 매우 악화
```

### Off-Heap 메모리의 위험성

```
Off-Heap 메모리:
- GC 관리 대상 아님
- 명시적으로 해제해야 함
- 누수되면 프로세스 메모리가 계속 증가

사례: ByteBuffer를 Direct Buffer로 할당
┌─────────────────────────────────────────┐
│ ByteBuffer.allocateDirect(1MB)          │
│ ← Off-Heap에 1MB 할당                  │
│ ← 해제되지 않으면 계속 메모리 증가      │
└─────────────────────────────────────────┘

1시간 후:
초기: Off-Heap 100MB
현재: Off-Heap 1500MB (+1400MB 누수!)

진단:
$ jcmd $(jps | grep App | awk '{print $1}') VM.native_memory summary

Off-Heap 메모리 누수를 찾으려면:
1. jstack으로 Off-Heap 할당하는 코드 찾기
2. Heap dump에서 ByteBuffer 참조 확인
3. Cleaner 메커니즘 확인 (자동 해제)
```

---

## 💻 실전 실험

**시나리오: p99 응답시간 1500ms, Heap 사용률 75%**

```bash
#!/bin/bash
# memory-diagnosis.sh

APP_PID=$(jps | grep App | awk '{print $1}')

echo "=== 메모리 병목 진단 시작 ==="
echo

# 1. GC 로그 분석
echo "1. GC 성능 분석 (1분)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
jstat -gc -h20 $APP_PID 1000 60 > /tmp/gc_stats.txt

# GC pause time 계산
awk '
NR > 2 {
  if (prev_ygct != 0) {
    ygc_delta = $18 - prev_ygct
    ygct_delta = $19 - prev_ygct_time
    if (ygc_delta > 0) {
      pause = (ygct_delta * 1000) / ygc_delta
      total_pause += pause
      count++
    }
  }
  prev_ygct = $18
  prev_ygct_time = $19
}
END {
  if (count > 0) {
    avg_pause = total_pause / count
    printf "Young GC 평균 pause: %.1fms\n", avg_pause
    if (avg_pause > 100) print "⚠️  WARNING: Pause time 높음"
  }
}' /tmp/gc_stats.txt

# 2. Heap 연령 분포
echo
echo "2. Heap 객체 분포"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
jmap -histo:live $APP_PID | head -15

# 3. GC 빈도
echo
echo "3. GC 빈도 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
tail -1 /tmp/gc_stats.txt | awk '{
  ygc = $18
  fgc = $20
  print "Young GC 횟수: " ygc
  print "Full GC 횟수: " fgc
  if (fgc > 5) print "⚠️  WARNING: Full GC 많음"
}'

# 4. Heap vs Off-Heap
echo
echo "4. Heap vs Off-Heap 메모리"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
ps -p $APP_PID -o rss | tail -1 | awk '{
  rss = $1 / 1024
  print "RSS (물리 메모리): " int(rss) "MB"
}'

jstat -gc $APP_PID | tail -1 | awk '{
  heap = $4 + $5 + $6 + $8 + $9
  print "Heap 사용량: " int(heap/1024) "MB"
}'

# 5. 메모리 사용 추이 (10분 모니터링)
echo
echo "5. 메모리 사용 추이 (10분)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
watch -n 10 "jstat -gc $APP_PID | tail -1 | awk '{print ((\$4+\$5+\$6+\$8+\$9)/1024) \" MB\"}'"

# 6. 정리
echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== 메모리 병목 진단 시작 ===

1. GC 성능 분석 (1분)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Young GC 평균 pause: 234.5ms
⚠️  WARNING: Pause time 높음 (정상: <50ms)

2. Heap 객체 분포
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 num     #instances         #bytes  class name
   1        5234567       1234567890  [B (byte array)
   2        1234567        234567890  java.lang.String
   3         456789         45678901  java.util.HashMap$Node
   4         234567         23456789  com.example.UserCache$Entry
   5         123456         12345678  java.util.LinkedList$Node

3. GC 빈도 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Young GC 횟수: 2345
Full GC 횟수: 23
⚠️  WARNING: Full GC 많음 (1분에 23회 = 초당 0.38회!)

4. Heap vs Off-Heap 메모리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RSS (물리 메모리): 2456MB
Heap 사용량: 1847MB

5. 메모리 사용 추이 (10분)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1850 MB → 1860 MB → 1855 MB → ... (계속 변동)

=== 진단 완료 ===

🔍 진단 결과:
1. Young GC pause 234ms (정상 <50ms)
   → Full GC 빈도 높음 (초당 0.4회)
   → 응답시간에 직접 영향

2. Heap 객체:
   - byte[]가 1.2GB (전체 70%)
   - UserCache$Entry 많음 (정상 아님)

3. Full GC 23회/분:
   → 정상은 Full GC 0회/분
   → 심각한 상황

원인 분석:
- UserCache$Entry가 계속 증가 = 캐시 메모리 누수?
- Full GC 빈도 높음 = Young에서 Old로 넘어가는 객체 많음

다음 단계:
1. UserCache 코드 검토 (eviction policy 확인)
2. Heap 덤프 분석 (누수 확인)
3. 객체 생성 최소화
4. GC 튜닝 (G1GC 고려)
```

---

## 📊 성능 비교

| 메트릭 | 정상 범위 | 높음 기준 | p99 영향 | 해결책 |
|--------|---------|---------|---------|--------|
| **Young GC Pause** | <20ms | >100ms | +100~300ms | Young 영역 크기 조정 |
| **Full GC Pause** | 0회/시간 | 1회/분 | +500ms~3000ms | 객체 생성 ↓, Heap ↑ |
| **GC 빈도** | <1회/sec | >10회/sec | +50ms | 객체 생성 최소화 |
| **Heap 사용률** | 60~80% | >90% | +200ms | Heap 증설 또는 캐시 제거 |
| **Off-Heap** | 증가 없음 | 계속 증가 | 응답시간 ↑ | 메모리 누수 수정 |

**실제 케이스:**

```
케이스 1: Young GC Pause 234ms, Full GC 빈도 높음
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p99: 1500ms
  Young GC pause: 234ms, Full GC: 23회/분
  GC 누적 시간: 45초/분 (75% 시간이 GC!)

원인: UserCache가 Heap의 40% 차지
→ TTL 없는 정적 캐시가 계속 메모리 증가

After: LRU 캐시로 변경 (최대 10만 개 제한)
  p99: 150ms (90% 개선)
  Young GC pause: 45ms, Full GC: 0회/분
  GC 누적 시간: 2초/분 (4% 감소)

투자: 0 (코드 수정만), 비용 절감 (메모리 증설 불필요)

케이스 2: Heap 사용률 90%, Full GC pause 1초
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p99: 2000ms
  Heap: 2GB, 사용률 90%
  Full GC: 1초, 주기 10분마다

원인: Heap 크기가 너무 작아 자주 부족 상태

After: Heap 2GB → 4GB (메모리 증설)
  p99: 400ms (80% 개선)
  Heap: 4GB, 사용률 45%
  Full GC: 0.3초, 주기 1시간마다

투자: 메모리 비용 (연 $500~1000)

케이스 3: Off-Heap 메모리 누수 (1GB 증가/시간)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p99: 정상 (100ms)
  RSS: 초기 2GB → 24시간 후 5GB
  응답시간 악화

원인: DirectBuffer 할당 후 해제 안 됨
→ jmap -histo로 java.nio.DirectByteBuffer 검색
→ 15만 개 인스턴스 (총 1GB)

After: Cleaner 메커니즘 수정, try-with-resources 사용
  RSS: 안정적으로 2GB 유지
  응답시간: 정상 유지
  
투자: 0 (코드 수정만)
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 선택 기준 |
|------|------|------|---------|
| **Heap 증설** | 빠른 개선, 구현 간단 | 근본 원인 미해결, 비용 증가 | 긴급 상황, 단기 해결 필요 |
| **GC 튜닝** | 기존 Heap에서 최적화, 비용 0 | 복잡도 높음, 테스트 필요 | Full GC 빈도 높을 때 |
| **객체 생성 최소화** | 근본 원인 해결, 장기 효과 | 시간 오래 걸림, 코드 리팩토링 | 정상 상황, 지속 개선 |
| **캐시 제거/제한** | 응답시간 향상 + 메모리 절감 | 기능 변경 (캐시 응답 시간 ↑) | 누수된 캐시 있을 때 |

---

## 📌 핵심 정리

1. **메모리 사용률 ≠ 메모리 병목**
   - 메모리 사용률 90%는 정상 (메모리는 쓰는 게 당연)
   - GC pause time > 100ms이면 병목 (응답시간에 직접 영향)
   - 메모리 누수는 사용률이 아니라 추이로 판단

2. **GC 로그가 진실**
   - jstat으로 Young GC, Full GC 빈도와 시간 확인
   - Full GC > 0회/시간이면 조사 필요
   - GC pause time이 응답시간에 직접 더해진다 (p99 + pause time)

3. **Heap vs Off-Heap 구분**
   - Heap: GC 관리, 메모리 누수 자동 정리
   - Off-Heap: 수동 해제 필요, 누수 시 프로세스 메모리 계속 증가
   - jcmd VM.native_memory로 Off-Heap 추적

4. **문제 진단 우선순위**
   - Full GC 빈도 높음 → 객체 생성 줄이기
   - Heap 사용률 높음 (>85%) → Heap 증설 또는 캐시 제거
   - Off-Heap 증가 중 → 메모리 누수 찾아 수정
   - Minor GC pause 정상이면 Heap 증설 아님

5. **의사결정 기준**
   ```
   GC pause time > 100ms?
     ├─ Yes: Full GC 빈도 확인
     │   ├─ Full GC > 1회/분: 객체 생성 최소화 (우선)
     │   └─ Full GC < 1회/시간: Heap 증설 (차선)
     └─ No: 메모리는 병목 아님. CPU, I/O 확인
   ```

---

## 🤔 생각해볼 문제

**Q1: Heap 2GB, 사용률 40%인데 Full GC가 초당 1회 발생하면?**

<details>
<summary>해설 보기</summary>

**이것은 심각한 상황이다.**

**정상적인 Full GC:**
- Heap 사용률 > 85%에서 발생
- 또는 Promotion Failure (Young → Old 승격 실패)

**현재 상황 분석:**
```
Heap 2GB, 사용률 40% (남은 공간 1.2GB)
→ 충분한 공간이 있는데 Full GC 1회/초?

원인 1: Explicit GC 호출
System.gc()를 애플리케이션에서 명시적으로 호출
→ 코드에서 System.gc() 찾아서 제거

원인 2: Promotion Failure
Young 영역에서 Old로 넘어가는 객체가 많아서
Old 공간이 충분해도 승격 실패
→ Young 영역 크기 조정: -XX:NewRatio=3 (기본값) → 2

원인 3: G1GC의 Concurrent Cycle
-XX:+UseG1GC에서 관리 차원의 Full GC
→ Full GC를 피할 수 없으나, Young GC로 충분할 수도

원인 4: JVM 버전 이슈
구형 JVM(Java 7)의 알려진 버그
→ 최신 버전으로 업그레이드
```

**해결책 (우선순위):**
1. System.gc() 호출 제거
2. Young 영역 크기 조정
3. GC 알고리즘 변경 (G1GC → ZGC 또는 Shenandoah)
4. JVM 버전 업그레이드

**결론:** Heap에 여유가 있는데 Full GC가 빈번하면 근본 원인을 찾아야 한다. Heap 증설은 도움이 안 될 수 있다.

</details>

---

**Q2: 메모리 사용률 85%, Young GC pause 50ms, Full GC 0회/시간이면 응답시간을 개선할 수 있을까?**

<details>
<summary>해설 보기</summary>

**답: 메모리 측면에서는 개선할 게 없다.**

**분석:**
```
메모리 사용률 85%: 정상 (목표 범위)
  → Heap이 충분히 사용되고 있음

Young GC pause 50ms: 정상 (목표 <100ms)
  → 응답시간에 거의 영향 없음

Full GC 0회/시간: 최적
  → Full GC로 인한 응답시간 악화 없음
```

**결론:**
메모리는 정상 상태다. 응답시간이 높다면 다른 곳을 봐야 한다:
- CPU: User time, Context switch 확인 → 02-cpu-bottleneck.md
- I/O: Disk await, DB 쿼리 → 04-db-bottleneck.md
- 네트워크: 응답 크기, 연결 재사용 → 07-network-bottleneck.md
- 스레드 풀: BLOCKED 스레드 개수 → 06-thread-pool-analysis.md

**의사결정:**
메모리 증설은 불필요. CPU, I/O, 네트워크를 우선 진단하자.

</details>

---

**Q3: Heap dump 분석에서 int[] 배열이 1GB인 것으로 나타나면 메모리 누수인가?**

<details>
<summary>해설 보기</summary>

**반드시 누수는 아니다.**

**int[] 배열의 정상 사용 사례:**
```
1. 행렬 계산 (과학 / ML)
   int[][] matrix = new int[10000][10000]  // 400MB
   → 정상적인 알고리즘
   → 계산 후 사용 안 하면 GC에서 회수

2. 이미지 처리
   byte[] pixels = new byte[1920 * 1080 * 4]  // 8MB
   × 100개 이미지 = 800MB
   → 일시적으로 높을 수 있음

3. 버퍼 풀 (정상)
   List<byte[]> buffers = new ArrayList<>();
   for (int i = 0; i < 1000; i++) {
     buffers.add(new byte[1MB]);
   }
   → 재사용 목적, 메모리는 필요

4. 버퍼 풀 (누수)
   List<byte[]> buffers = new ArrayList<>();
   for (int i = 0; i < 999999; i++) {
     buffers.add(new byte[1KB]);  // 1GB 누적
   }
   // 버퍼를 반환하지 않고 계속 누적
   → 메모리 누수!
```

**메모리 누수 판단 방법:**

1. **같은 시간에 두 Heap dump 비교**
```
첫 번째 dump (10:00): int[] 총 1GB
두 번째 dump (10:10): int[] 총 1.5GB (증가!)
→ 누수 가능성

두 번째 dump (10:10): int[] 총 1GB (동일)
→ 정상 (필요한 메모리일 뿐)
```

2. **Heap dump에서 참조 경로 추적**
```
Eclipse Memory Analyzer:
int[] → List → StaticCache → ??? (정적 참조)
→ 정적 필드가 계속 참조 = 누수 가능

int[] → HashMap → GC root 없음
→ 잠깐 사용했다가 버려질 것 = 정상
```

3. **생성 시점과 해제 시점 일치 확인**
```
정상: request 들어옴 → int[] 할당 → 사용 → 반환
누수: request 들어옴 → int[] 할당 → 참조만 저장
      → 다음 request → 또 할당 → 계속 쌓임
```

**결론:** int[] 1GB가 있다고 해서 누수는 아니다. 트렌드(증가 추세)와 참조 경로를 확인해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: CPU 병목 분석 — User vs System vs IOWait 구분](./02-cpu-bottleneck.md)** | **[홈으로 🏠](../README.md)** | **[다음: DB 병목 분석 — HikariCP와 Connection Pool ➡️](./04-db-bottleneck.md)**

</div>
