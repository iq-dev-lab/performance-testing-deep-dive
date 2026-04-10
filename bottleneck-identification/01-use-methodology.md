# 01. USE 방법론 — 체계적으로 병목을 찾는 5분 체크리스트

---

## 🎯 핵심 질문

성능이 떨어졌을 때, 어디서부터 진단을 시작해야 할까?
- CPU가 문제일까, 메모리일까, 디스크일까, 네트워크일까?
- 무작정 모니터링 대시보드를 들여다보면 시간만 낭비된다.
- **USE 방법론**은 5분 안에 병목을 범위화(narrowing)하는 체계적인 접근법이다.

USE는 3가지 메트릭을 각 리소스마다 확인한다:
- **U**tilization (사용률): 0~100%
- **S**aturation (포화도): 대기 큐의 크기
- **E**rrors (에러): 리소스 관련 에러율

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**
- 대시보드에 20개의 메트릭이 떴다. 뭘 먼저 봐야 하나?
- "CPU 사용률이 60%라고?"→ 문제가 아니다 (CPU는 60%여도 포화될 수 있다).
- "메모리 사용률이 80%"→ 문제가 아니다 (메모리는 사용하는 게 정상이다).

**USE 방법론의 가치:**
1. **의사결정 트리**: 각 리소스의 U→S→E 순으로 확인하면 병목이 어디인지 5분 안에 좁혀진다.
2. **거짓 양성 제거**: "CPU 사용률 높음" 같은 허상 알림을 무시하고 진짜 병목에만 집중한다.
3. **문제 영역 특정**: 병목이 CPU, 메모리, 디스크, 네트워크 중 정확히 어느 것인지 특정 가능하다.

**실무 사례:**
```
성능 저하 발생 → USE 체크리스트 실행
├─ CPU Utilization: 45% (낮음) ✓
├─ CPU Saturation: run queue 12 (1 core 기준 높음) ✗ ← 병목!
│
├─ Memory Utilization: 70% (정상 범위) ✓
├─ Memory Saturation: swap 사용 없음 ✓
│
├─ Disk Utilization: 20% (낮음) ✓
└─ Disk Saturation: I/O 대기 큐 2 (낮음) ✓

결론: Context Switch 과다 발생. 스레드 수 조사 필요.
```

---

## 😱 흔한 실수 (Before)

```bash
# 잘못된 접근 1: CPU 사용률만 본다
$ top
# Cpu(s): 45.0%us, 15.0%sy, 0.0%ni, 40.0%id, 0.0%wa, 0.0%hi, 0.0%si, 0.0%st
# 결론: "CPU 좋음. 다른 곳 찾아봐."
# → 근데 run queue는 뭔데 20이야? 무시하면 안 된다!

# 잘못된 접근 2: 한 가지 메트릭으로 판단한다
$ free -h
#                total        used        free      shared  buff/cache   available
# Mem:           31Gi        24Gi       2.0Gi       200Mi        4.5Gi       6.0Gi
# 결론: "메모리 부족! GC를 줄여야 한다."
# → 근데 GC 빈도가 정상인데? swap도 안 쓰는데?

# 잘못된 접근 3: 리소스별로 체계적으로 확인하지 않는다
# 어라, 어떤 순서로 봐야 하지? 각 리소스마다 뭘 봐야 하지?
```

**실제 사례:**
```
상황: p95 응답시간 2000ms → 8000ms로 증가
개발자 판단: "메모리 사용률이 85%니까 메모리 늘려야 한다!"
→ 메모리 32GB → 64GB로 증가 (비용 증가)
→ 응답시간 개선 안 됨
→ 나중에 알고 보니 DB 스레드 풀 고갈이 원인
→ 메모리 확대는 무의미했음 (비용만 낭비)
```

---

## ✨ 올바른 접근 (After)

**USE 방법론 체크리스트:**

### 1단계: CPU 리소스

```bash
# Utilization: CPU 사용률 (idle 제외한 부분)
$ mpstat -P ALL 1 5
# 15:30:45 CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
# 15:30:46   0   12.00    0.00    8.00   5.00    0.00    0.00    0.00    0.00    0.00   75.00
# 15:30:46   1   18.00    0.00    6.00   3.00    0.00    0.00    0.00    0.00    0.00   73.00
# → Utilization = (100 - idle) = 25~27% (낮음, 정상)

# Saturation: run queue (실행 대기 중인 프로세스 수)
$ vmstat 1 5
# r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
# 8  0      0 2000000 100000 1000000 0    0     0     0  1000 5000 12 8 75 5
# → r=8, CPU 코어=2 → saturation = 8/2 = 4 (높음! 심각)

# Errors: CPU 관련 에러 (거의 없음, 생략 가능)
```

### 2단계: 메모리 리소스

```bash
# Utilization: 사용 중인 메모리 비율
$ free -h
#                total        used        free      shared  buff/cache   available
# Mem:           31Gi        24Gi       2.0Gi       200Mi        4.5Gi       6.0Gi
# → Utilization = (used - cache) / total = (24 - 4.5) / 31 = 63% (정상)

# Saturation: swap 사용량 + GC 빈도
$ vmstat 1 5
# si   so = swap in/out (0이어야 정상)
# si=0, so=0 → saturation 정상

# GC 빈도 확인
$ jstat -gc -h10 $(jps | grep App | awk '{print $1}') 1000
# S0C    S1C    S0U    S1U      EC        EU       OC         OU
# 1024 1024   512    512  10240      7180    20480      15360
# → Minor GC가 초당 1회 이상이면 saturation 높음

# Errors: OOM, GC overhead 관련 에러
$ grep -i "OutOfMemory\|GC overhead" /var/log/app.log | wc -l
# 0 → 정상
```

### 3단계: 디스크 리소스

```bash
# Utilization: 디스크 사용률
$ df -h
# Filesystem      Size  Used Avail Use%
# /dev/sda1       100G   60G   40G  60%
# → Utilization = 60% (정상)

# Saturation: I/O 대기 큐 길이 (avgqu-sz)
$ iostat -x 1 5
# Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  await  avgqu-sz
# sda              50.0   100.0      2.5       5.0      0.0      5.0    15.0       3.5
# → await=15ms (평균 I/O 대기 시간)
# → avgqu-sz=3.5 (디스크가 처리 못하는 I/O가 대기 중, 포화도 높음)

# Errors: I/O 에러
$ dmesg | grep -i "error\|ata\|sda" | tail -20
```

### 4단계: 네트워크 리소스

```bash
# Utilization: 네트워크 인터페이스 사용률
$ ss -s
# TCP:   1234 (estab 850, closed 384, orphaned 0, synrecv 0, timewait 0)
# Tcp:   850 ESTABLISHED (대략 850개 연결)
# → 연결 수 포화도 확인: max_connections(보통 4096~65535) 대비 850은 낮음

# Saturation: TCP retransmit, drop, overflow
$ netstat -s | grep -E "retransmit|dropped|overflow"
# 234 segments retransmitted
# 0 connections reset
# → retransmit이 높으면 네트워크 신뢰성 문제, drop/overflow 높으면 포화도 높음

# Errors: TCP errors, socket errors
$ cat /proc/net/dev | grep eth0
# eth0: 1234567 12345  0    0    0     0          654321 9876  0    0    0     0    0     0
#       RX bytes RX pkt RX err RX drop RX overflow ... TX bytes TX pkt TX err ...
# → RX err, RX drop 값 확인
```

---

## 🔬 내부 동작 원리

### Utilization vs Saturation의 차이

**Utilization (사용률):**
- 리소스가 **얼마나 바쁜가**를 측정
- 0%: 유휴, 100%: 전부 사용 중
- **높은 utilization이 문제는 아니다** (리소스를 쓰는 게 정상)

**Saturation (포화도):**
- 리소스가 **더 이상 수용할 수 없어서 대기 중인 작업**이 몇 개인가
- 0: 대기 없음, >0: 포화도 높음
- **포화도가 문제다** (응답시간 악화의 주범)

```
예시: CPU
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

상황 1: Util=90%, Saturation=0
┌─────────────────────────────┐
│ CPU [========== 작업 ========│  ← 90% 사용 중
│                                │
│ Run Queue: 0개 대기            │  ← 0개 대기
└─────────────────────────────┘
결론: 정상. CPU가 열심히 일 중이고, 대기는 없다.
응답시간: 20ms

상황 2: Util=45%, Saturation=8 (2 core)
┌─────────────────────────────┐
│ CPU [===== 작업 =====       │  ← 45% 사용 중
│                                │
│ Run Queue: [1][2][3][4] 4개   │  ← 4개 대기 (큐 길이 기준)
│           [5][6][7][8]        │
└─────────────────────────────┘
결론: 심각. CPU는 여유가 있는데 스레드가 대기 중.
응답시간: 200ms (10배 악화!)

원인: Context Switch 과다, Lock Contention 등
```

### 각 리소스별 포화도 지표

```
┌─────────────┬──────────────────────────┬─────────────────────┐
│  리소스      │  Saturation 지표         │  높음 기준값        │
├─────────────┼──────────────────────────┼─────────────────────┤
│ CPU         │ run queue length         │ > core 수           │
│ Memory      │ swap in/out rate, GC     │ > 0 (swap)          │
│             │ frequency                │ > 10 GC/sec (minor) │
│ Disk        │ I/O wait queue (avgqu)   │ > disk count        │
│ Network     │ TCP retransmit, overflow │ > 0.1%              │
└─────────────┴──────────────────────────┴─────────────────────┘
```

---

## 💻 실전 실험

**시나리오: p99 응답시간이 갑자기 2초로 증가했을 때**

```bash
#!/bin/bash
# use-check.sh - 5분 안에 병목 특정하기

echo "=== USE 체크리스트 시작 ==="
echo

echo "1. CPU 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "[Utilization]"
mpstat -P ALL 1 3 | tail -3

echo
echo "[Saturation - Run Queue]"
vmstat 1 3 | tail -1 | awk '{print "Current run queue: " $1}'
nproc=$(nproc)
echo "CPU cores: $nproc"

echo
echo "2. 메모리 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "[Utilization]"
free -h | grep Mem

echo
echo "[Saturation - Swap]"
vmstat 1 1 | tail -1 | awk '{print "Swap in: " $7 " MB/s, Swap out: " $8 " MB/s"}'

echo
echo "3. 디스크 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "[Utilization]"
df -h | grep /dev/

echo
echo "[Saturation - I/O Queue]"
iostat -x 1 2 | tail -10 | awk '{print $1, "await: " $10 "ms, avgqu: " $9}'

echo
echo "4. 네트워크 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "[Connections]"
ss -s | grep -E "TCP|TCP:"

echo
echo "[Saturation - Retransmits]"
netstat -s 2>/dev/null | grep -E "retransmitted|dropped"

echo
echo "=== 분석 완료 ==="
```

**실행 결과:**

```
=== USE 체크리스트 시작 ===

1. CPU 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Utilization]
15:45:30   0   45.00    0.00   12.00    3.00    0.00    0.00    0.00    0.00    0.00   40.00
15:45:31   1   38.00    0.00   10.00    2.00    0.00    0.00    0.00    0.00    0.00   50.00

[Saturation - Run Queue]
Current run queue: 12
CPU cores: 2
⚠️  WARNING: run queue 12 >> cores 2 (포화도 심각!)

2. 메모리 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Utilization]
Mem:           31Gi        24Gi       2.0Gi       200Mi        4.5Gi       6.0Gi

[Saturation - Swap]
Swap in: 0 MB/s, Swap out: 0 MB/s
✓ OK: No swap activity

3. 디스크 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Utilization]
/dev/sda1       100G   60G   40G  60%

[Saturation - I/O Queue]
sda await: 15ms, avgqu: 3.5
⚠️  WARNING: avgqu-sz 3.5 (I/O 대기 중)

4. 네트워크 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Connections]
TCP:   850 ESTABLISHED

[Saturation - Retransmits]
445 segments retransmitted
⚠️  WARNING: retransmits 높음 (네트워크 신뢰성 문제)

=== 분석 완료 ===

🔍 결론:
1. CPU: Saturation 높음 (run queue 12 > cores 2)
2. 디스크: Saturation 높음 (avgqu-sz 3.5)
3. 네트워크: Saturation 높음 (retransmit 445)

다음 단계:
→ CPU: 스레드 풀 상태 확인 (jstack)
→ 디스크: DB 쿼리 성능 분석 (SHOW PROCESSLIST)
→ 네트워크: DB 연결 풀 확인 (HikariCP)
```

---

## 📊 성능 비교

| 지표 | 정상 상태 | 병목 상태 | 영향 범위 |
|------|---------|---------|---------|
| **CPU Utilization** | 30~70% | >80% | 응답시간 +100ms |
| **CPU Saturation (run queue/core)** | <1 | >3 | 응답시간 +500ms~2000ms |
| **Memory Util** | 60~80% | >90% | 응답시간 +200ms (GC 빈도↑) |
| **Swap Usage** | 0MB | >100MB | 응답시간 +1000ms (디스크 I/O) |
| **Minor GC Freq** | 1~5/sec | >10/sec | p99 응답시간 +300ms |
| **Disk Util** | 30~60% | >80% | - |
| **Disk Saturation (avgqu-sz)** | <1 | >5 | 응답시간 +500ms~1000ms |
| **Disk await** | <10ms | >50ms | 응답시간 +200ms |
| **TCP Retransmit** | <0.01% | >1% | 응답시간 +100ms~500ms |
| **Connection Pool Pending** | 0 | >10 | 응답시간 +1000ms+ |

**실제 케이스:**

```
케이스 1: CPU Saturation 높음 (run queue 8 on 2 core)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p50: 50ms, p95: 100ms, p99: 200ms
  CPU Utilization: 45%, CPU Saturation: 8

원인: 스레드 풀 크기 너무 크음 (300 스레드)
→ Context Switch 과다

After: 스레드 풀 100으로 축소
  p50: 25ms, p95: 40ms, p99: 80ms
  CPU Utilization: 65%, CPU Saturation: 0.5
  개선율: p99 60% 개선, CPU Utilization 실제로 올라갔지만 응답시간은 개선!

케이스 2: Disk Saturation 높음 (avgqu-sz 8)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p99: 1500ms
  Disk await: 80ms, Disk Saturation: 8
  문제: DB 쿼리 Full Scan

After: 인덱스 추가, 느린 쿼리 최적화
  p99: 150ms
  Disk await: 5ms, Disk Saturation: 0.2
  개선율: p99 90% 개선

케이스 3: Memory Saturation (Swap 사용, GC 빈도 높음)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  p99: 800ms
  GC freq: 15/sec, Swap: 200MB/s
  GC pause: 200ms
  
After: Heap 크기 최적화 (4GB → 8GB)
  p99: 120ms
  GC freq: 2/sec, Swap: 0MB/s
  GC pause: 50ms
  개선율: p99 85% 개선
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 적용 시점 |
|------|------|------|---------|
| **리소스 증설** (CPU, 메모리) | 빠른 개선, 구현 간단 | 비용 증가, 근본 원인 해결 X | 긴급 상황, 구조 개선 시간 없을 때 |
| **코드 최적화** | 근본 원인 해결, 비용 ↓ | 시간 오래 걸림, 기술 부채 | 정상 상황, 지속 개선 |
| **아키텍처 변경** | 확장성 개선, 장기 효과 | 고위험, 구현 복잡 | 구조적 병목, 장기 계획 |

**선택 기준:**

```
즉시 대응 필요?
  ├─ Yes → 리소스 증설 (CPU, 메모리, 디스크)
  └─ No → 근본 원인 분석 후 코드 최적화

근본 원인이 명확한가?
  ├─ Yes → 바로 수정 (쿼리 최적화, 스레드 풀 튜닝 등)
  └─ No → USE 방법론으로 범위 좁혀서 진단

구조적 문제인가?
  ├─ Yes → 아키텍처 재설계 (비동기 처리, 마이크로서비스 등)
  └─ No → 파라미터 튜닝
```

---

## 📌 핵심 정리

1. **USE 방법론은 의사결정 트리**
   - 각 리소스(CPU, Memory, Disk, Network)에 대해 U→S→E 순으로 확인
   - 5분 안에 병목 범위를 90% 이상 좁힐 수 있다

2. **Utilization ≠ 문제, Saturation = 문제**
   - CPU 사용률 90%도 saturation 0이면 정상
   - Memory utilization 95%도 saturation 0이면 정상
   - Saturation이 실제 응답시간 악화의 주범

3. **각 리소스별 체크 명령어 암기:**
   - CPU: `vmstat` (run queue), `mpstat` (utilization)
   - Memory: `free`, `vmstat` (swap)
   - Disk: `iostat -x` (avgqu-sz, await)
   - Network: `ss -s`, `netstat -s` (retransmit)

4. **의사결정 우선순위:**
   - CPU saturation > Disk saturation > Memory saturation > Network saturation
   - CPU 포화는 응답시간에 직접 영향 (run queue 1 = +20~50ms)
   - 메모리 포화는 간접 영향 (GC 빈도 증가)

5. **체크리스트는 자동화:**
   - `use-check.sh` 같은 스크립트로 5분마다 자동 실행
   - 알림 설정: CPU saturation > cores * 2 → 페이징
   - 트렌드 수집: 일일 변화 추적 (정상 범위 파악)

---

## 🤔 생각해볼 문제

**Q1: CPU Utilization 90%, CPU Saturation 0.1인 상황에서 CPU를 스케일 업해야 할까?**

<details>
<summary>해설 보기</summary>

**답: 아니다.**

CPU Saturation이 거의 0이라는 것은 대기하는 프로세스가 거의 없다는 뜻이다. 즉, 현재 CPU가 충분한 성능을 낼 수 있는 상황이다. Utilization 90%는 단순히 "CPU가 열심히 일 중"이라는 의미일 뿐이다.

**응답시간이 높다면 병목은 다른 곳:**
- 디스크 I/O 대기? (Disk saturation 확인)
- 메모리 부족으로 GC 빈번? (GC freq, swap 확인)
- DB 연결 풀 부족? (DB pending 확인)
- 외부 API 응답 지연? (네트워크 대기)

CPU 스케일 업은 근본 해결이 아니고, 비용만 낭비하게 된다.

**실제 사례:** Netflix는 CPU Utilization 70~85% 범위에서 운영하면서도 응답시간을 밀리초 단위로 관리한다. Saturation이 낮기 때문이다.

</details>

---

**Q2: 디스크 avgqu-sz가 5.0이면 SSD로 업그레이드해야 할까? 아니면 쿼리를 최적화해야 할까?**

<details>
<summary>해설 보기</summary>

**단기: 둘 다 필요할 수 있다. 단계별 접근:**

1. **먼저 쿼리 최적화 (24시간 안에)**
   ```sql
   SHOW FULL PROCESSLIST;  -- 실행 중인 쿼리 확인
   SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST;  -- 각 쿼리별 시간 확인
   ```
   - Full Scan이 많으면 인덱스 추가
   - 불필요한 JOIN 제거
   - 배치 쿼리를 비동기로 분리
   
   **기댓값:** avgqu-sz 5.0 → 2.0 (비용 0)

2. **응답시간 개선되었는가?**
   - Yes → 완료. SSD 업그레이드 불필요
   - No → 다음 단계

3. **SSD 업그레이드 (1~2주)**
   - Disk I/O가 여전히 병목이라면 SSD로 전환
   - IOPS 향상으로 avgqu-sz 감소
   
   **기댓값:** SSD는 avgqu-sz를 50% 이상 감소시킴

**결론:** 항상 코드 최적화 먼저, SSD는 나중. 인프라는 "필요할 때" 추가하는 것이 정석.

</details>

---

**Q3: 같은 서버에서 CPU Saturation, Disk Saturation, Memory Saturation이 모두 높으면?**

<details>
<summary>해설 보기</summary>

**이것은 "동시 다중 병목" 상황이다.**

**우선순위 (영향도 순서):**

1. **CPU Saturation 먼저 해결**
   - 응답시간에 가장 직접적인 영향 (run queue 1 = +50ms)
   - CPU 포화 → 모든 작업이 슬로우 다운
   - 해결책: 스레드 풀 크기 조정, 스레드 할당 정책 변경

2. **Disk Saturation 다음**
   - 응답시간에 간접 영향 (디스크 I/O 대기)
   - 디스크 포화 → DB 쿼리 응답 지연
   - 해결책: 쿼리 최적화, 인덱스 추가, 배치 작업 분리

3. **Memory Saturation 마지막**
   - GC 빈도 증가로 응답시간 악화
   - 영향도는 상대적으로 낮음
   - 해결책: Heap 크기 최적화, 불필요한 객체 생성 제거

**실제 케이스:**

```
상황: p99 = 3000ms (목표: <500ms)

Use 진단:
├─ CPU Saturation: run queue 20 (cores 4)
├─ Disk Saturation: avgqu-sz 8
└─ Memory Saturation: GC freq 25/sec, swap 100MB/s

단계별 해결:
1. 스레드 풀 축소 (300→100)
   → p99: 3000ms → 1500ms (50% 개선)
   
2. 느린 쿼리 최적화 (Full Scan → Index 사용)
   → p99: 1500ms → 700ms (추가 53% 개선)
   
3. Heap 최적화 (4GB → 6GB)
   → p99: 700ms → 450ms (추가 36% 개선)

최종: 목표 달성!
```

**핵심:** 동시 다중 병목에서는 우선순위가 중요하다. CPU부터 해결하지 않으면 다른 최적화들의 효과가 제한된다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: CPU 병목 분석 — User vs System vs IOWait 구분 ➡️](./02-cpu-bottleneck.md)**

</div>
