# 02. CPU 병목 분석 — User vs System vs IOWait 구분

---

## 🎯 핵심 질문

CPU Utilization이 높을 때, 정확히 뭐가 문제일까?
- User time이 높으면: 애플리케이션 자체의 계산이 느림
- System time이 높으면: 시스템 콜(문맥 교환, 메모리 할당 등)이 과다
- IOWait이 높으면: 실제 계산이 아니라 I/O 대기 중

**각 경우 해결책이 완전히 다르다.** 진단을 잘못하면 엉뚱한 최적화를 하게 된다.

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
대시보드: CPU Utilization 85%
개발자 반응: "CPU 스케일 업 필요!"
→ vCPU 2개 → 4개로 증설 (연간 비용 2배)
→ 응답시간 개선 없음
→ 진짜 원인은 "IOWait 70%"였음 (I/O 대기, CPU 증설은 도움 X)
→ 실제 필요한 건: DB 쿼리 최적화, 디스크 SSD 업그레이드
→ 비용만 낭비하고 문제 해결 X
```

**올바른 진단:**

```
User Time 60% + System Time 15% + IOWait 10%
→ 애플리케이션이 CPU를 많이 쓰고 있음 (정상)
→ System time이 15%는 높은 편 (context switch 의심)
→ IOWait 10%는 무시할 수준

해결책:
1. 불필요한 문맥 교환 줄이기 (스레드 풀 크기 조정)
2. 애플리케이션 알고리즘 최적화 (CPU 시간 자체 감소)
```

**실무 효과:**
- 올바른 진단: 1시간 내 원인 파악 → 비용 0의 코드 수정
- 잘못된 진단: 3일 대기 → 인프라 비용 증가 → 문제 여전히 미해결

---

## 😱 흔한 실수 (Before)

```bash
# 잘못된 접근 1: CPU Utilization만 본다
$ top
# Cpu(s): 80.0%us, 10.0%sy, 0.0%ni, 5.0%id, 5.0%wa, 0.0%hi, 0.0%si
# 결론: "CPU 85% 사용 중. 스케일 업 필요!"
# → 근데 IOWait 5%만 있고, User 80%가 정상 비율인데?
# → 실제로는 충분한 CPU 성능을 쓰고 있는 중

# 잘못된 접근 2: top의 %us, %sy, %wa를 무시한다
$ top | grep Cpu
# Cpu(s): 15.0%us, 35.0%sy, 0.0%ni, 30.0%id, 20.0%wa
# 결론: "CPU 사용률이 50%이니 여유가 있다"
# → 근데 System time 35%는 매우 높음 (context switch 과다)
# → IOWait 20%는 I/O 대기 시간 (디스크 문제)
# → 실제 응답시간이 짧은 이유는 CPU가 아니라 I/O 최적화 필요

# 잘못된 접근 3: top을 한 번만 본다
$ top -bn1
# Cpu(s): 45.0%us, 15.0%sy, ...
# 결론: "현재 상태 파악 완료"
# → 근데 이건 특정 시점의 스냅샷일 뿐
# → 실제로는 매 초마다 변한다
# → 추세를 봐야 문제인지 아닌지 알 수 있다
```

**실제 사례:**

```
상황: p99 응답시간 2000ms
에러 진단:
1. "CPU Utilization 45%이니 문제 아니다" → 엄청 틀림
2. top 이미지 한 장 본 후 판단 → 추세 미파악
3. System time 40%, IOWait 30% 무시 → Context switch와 I/O 대기 심각

올바른 진단:
$ mpstat -P ALL 1 60  # 60초 동안 초마다 출력
→ System time 40%, IOWait 30%의 패턴이 일정하게 유지
→ CPU 자체는 여유가 있지만 I/O 대기 중
→ 실제 원인: DB 쿼리 성능 저하, 스레드 풀 과다 할당으로 context switch 증가

해결책:
1. 스레드 풀 크기 조정 (300 → 100)
   → System time 40% → 10%로 감소
2. DB 쿼리 최적화 (Full Scan → Index 사용)
   → IOWait 30% → 5%로 감소
3. 결과: p99 2000ms → 200ms (10배 개선!)
```

---

## ✨ 올바른 접근 (After)

### 1단계: CPU Time 분해 (top vs mpstat)

**top으로 전체 현황 파악:**

```bash
$ top -bn 3 -d 1  # 3번, 1초 간격
# Cpu(s): 45.0%us, 15.0%sy, 0.0%ni, 35.0%id, 5.0%wa, 0.0%hi, 0.0%si, 0.0%st
#
# %us = User time (애플리케이션 코드 실행)
# %sy = System time (시스템 콜, context switch, 메모리 할당 등)
# %ni = Nice time (우선순위 조정된 프로세스, 무시 가능)
# %id = Idle (CPU 유휴 시간)
# %wa = IOWait (I/O 작업 대기, 실제 계산 아님)
# %hi = Hard IRQ (하드웨어 인터럽트, 무시 가능)
# %si = Soft IRQ (소프트웨어 인터럽트, 네트워크 수신 등)
# %st = Steal (가상화 오버헤드, 무시 가능)

# 해석:
# User 45% + System 15% = CPU 사용률 60%
# IOWait 5% = I/O 작업 대기 중
# Idle 35% = CPU 유휴 (더 많은 작업 처리 가능)

결론:
- User time이 높음 (45%) → 애플리케이션 코드 최적화 필요
- System time이 높음 (15%) → Context switch 과다 의심
- IOWait이 낮음 (5%) → I/O는 병목 아님
```

**mpstat으로 코어별 상세 분석:**

```bash
$ mpstat -P ALL 1 5
# Linux 5.10.0 (prod-server) 03/15/2024
# 15:30:00 CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
# 15:30:01   0   50.00    0.00   10.00    2.00    0.00    0.00    0.00    0.00    0.00   38.00
# 15:30:01   1   40.00    0.00   20.00    5.00    0.00    0.00    0.00    0.00    0.00   35.00
# 15:30:01   2   45.00    0.00   15.00    3.00    0.00    0.00    0.00    0.00    0.00   37.00
# 15:30:01   3   35.00    0.00   18.00    4.00    0.00    0.00    0.00    0.00    0.00   43.00

# 분석:
# - CPU 0: User 50% (높음, 이 코어가 핫스팟)
# - CPU 1: System 20% (높음, context switch 가능성)
# - 코어별 차이가 크면 로드 불균형 또는 핸들로 스레드 몰려 있음

# 다음 단계:
# 1. CPU 0 (user 50%)에서 실행 중인 스레드 찾기 → jstack
# 2. Context switch 과다 (CPU 1 sys 20%) → vmstat로 추가 확인
```

### 2단계: Context Switch 빈도 확인 (vmstat)

```bash
$ vmstat 1 10  # 10초, 1초 간격
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
#  4  0      0 2000000 100000 1000000 0    0     0     0  1000 5000 45 15 35  5  0
#  5  0      0 1900000 100000 1000000 0    0     0     0  1050 5200 46 14 35  5  0
#  3  0      0 1800000 100000 1000000 0    0     0     0  950  4800 44 16 35  5  0
#
# cs (Context Switches) = 5000~5200/sec
# in (Interrupts) = 950~1050/sec
#
# 기준값:
# cs < 1000/sec: 정상
# 1000 < cs < 5000/sec: 약간 높음 (모니터링 필요)
# cs > 5000/sec: 높음 (성능 저하 가능)

분석:
- cs 5200/sec는 높음 (정상의 5배)
- 이 정도면 응답시간에 영향
- 원인: 스레드 풀 크기 너무 크거나, 짧은 타임아웃 설정

해결책: 스레드 풀 크기 조정
```

### 3단계: CPU 병목의 종류별 진단

**A. User Time이 높은 경우 (CPU-Bound)**

```bash
# 진단
$ mpstat -P ALL 1 5 | grep -E "^[0-9]+.*[5-9][0-9]\.[0-9].*%usr"
# → User 50% 이상이면 CPU-Bound 작업

# 원인:
# 1. 알고리즘 자체가 느림 (O(n²) 정렬 등)
# 2. 불필요한 객체 생성 (GC 압력)
# 3. 루프 내 DB 쿼리 (N+1 문제)
# 4. 암호화, 압축 등 CPU 집약적 작업

# 해결책:
# 1. 알고리즘 최적화 (O(n²) → O(n log n))
# 2. 캐싱 추가 (Redis, 로컬 캐시)
# 3. 배치 처리 (N+1 문제 해결)
# 4. 멀티스레드 (병렬 처리, CPU 바인딩)

# 실제 사례:
$ jstack $(jps | grep App | awk '{print $1}') > /tmp/jstack.txt
$ grep -A 5 "cpu" /tmp/jstack.txt  # CPU 많이 쓰는 메서드 찾기
```

**B. System Time이 높은 경우 (Context Switch 과다)**

```bash
# 진단
$ vmstat 1 5 | awk 'NR>2 {print $10}'  # cs 열 출력
# 5000
# 5200
# 4800
# → cs > 2000/sec이면 context switch 과다

# 원인:
# 1. 스레드 풀 크기 너무 큼 (300개 스레드 vs 4코어)
# 2. 락(Lock) 경합 (synchronized, ReentrantLock)
# 3. 짧은 타이밍 기반 스케줄링 (sleep, wait)

# 해결책:
$ cat application.yml
# 스레드 풀 조정
server:
  tomcat:
    threads:
      max: 100  # 기존 300 → 100 (권장: cores * 2)

# 락 경합 확인
$ jstack ... | grep "locked <" | wc -l
# 10
# → 10개 스레드가 락 대기 중 → synchronized 제거 또는 ConcurrentHashMap 사용

# 실제 개선:
Before: cs=5200/sec, System time=35%, p99=1500ms
After:  cs=800/sec, System time=8%, p99=150ms
개선율: Context switch 85% 감소, p99 90% 개선
```

**C. IOWait이 높은 경우 (I/O-Bound)**

```bash
# 진단
$ vmstat 1 5 | awk 'NR>2 {print $9}'  # wa 열 (IOWait) 출력
# 20
# 25
# 18
# → IOWait > 10%이면 I/O 병목

# 원인:
# 1. DB 쿼리 느림 (Full Scan)
# 2. 디스크 I/O 느림 (느린 디스크, SSD 아님)
# 3. 네트워크 대기 (외부 API 호출)
# 4. 캐시 미사용 (매번 디스크에서 읽음)

# 해결책:
# 1. DB 쿼리 최적화
SHOW FULL PROCESSLIST;  # 느린 쿼리 찾기
EXPLAIN SELECT ...;     # 실행 계획 확인

# 2. 디스크 성능 확인
$ iostat -x 1 5
# Devices that are slow (await > 20ms)
$ grep "sda" iostat.txt | awk '$10 > 20'

# 3. 캐시 추가
# Redis, 로컬 캐시(Caffeine, Guava Cache)

# 실제 개선:
Before: IOWait=25%, await=80ms, p99=1000ms
After:  IOWait=3%, await=5ms, p99=100ms
해결책: 인덱스 추가 (테이블 스캔 제거)
```

---

## 🔬 내부 동작 원리

### CPU Time의 구성요소

```
CPU 시간 100%
├─ User Time (45%)
│  ├─ 애플리케이션 코드 실행
│  ├─ 메모리 접근 (L1/L2/L3 캐시)
│  └─ 버추얼 머신 오버헤드 (JVM)
│
├─ System Time (15%)
│  ├─ 문맥 교환 (context switch)
│  │  └─ 현재 스레드 상태 저장, 다음 스레드 로드
│  ├─ 시스템 콜 (syscall)
│  │  └─ open, read, write, malloc 등
│  ├─ 페이지 폴트 (page fault)
│  │  └─ 메모리 접근 시 물리 메모리 할당
│  └─ 인터럽트 처리 (IRQ)
│     └─ 디스크, 네트워크 하드웨어 신호
│
├─ IOWait (5%)
│  ├─ 디스크 I/O 대기 (read/write 완료)
│  ├─ 네트워크 I/O 대기 (send/recv 완료)
│  └─ 기타 I/O 대기
│     └─ 주의: CPU가 유휴 상태 (다른 일 못 함)
│
└─ Idle (35%)
   └─ CPU 아무것도 안 함
```

### Context Switch의 비용

```
┌─────────────────────────────────────────────────┐
│ 현재 스레드 (Thread A) 실행 중                   │
│ Register, Cache, TLB(MMU 캐시)에 데이터 로드    │
└─────────────────────────────────────────────────┘
                       ↓ (OS 스케줄러)
        Thread A ← Context를 저장
             ↓
        Thread B ← Context를 로드
                       ↓
┌─────────────────────────────────────────────────┐
│ Thread B 실행 시작                               │
│ 새로운 Register, Cache, TLB 워밍(warming) 필요  │
│ → L3 캐시 미스 증가, 성능 저하                  │
└─────────────────────────────────────────────────┘

비용 추정:
- 경경 context switch: 1~10us (0.001~0.01ms)
- 나쁜 경우 (캐시 미스 많음): 100us~1ms
- 초당 5000번 × 0.5ms = 2.5초 순손실 (4코어 기준 25% 성능 저하)
```

### IOWait의 특수성

```
일반적인 오해:
"IOWait 20%이면 CPU를 20% 못 쓴다"
→ 틀렸다.

IOWait의 실제 의미:
CPU가 I/O 완료를 기다리는 동안 다른 스레드에게 CPU 시간을 양보
→ 즉, IOWait이 높아도 다른 스레드가 있으면 CPU를 활용할 수 있다.

예시:
Thread A: DB 쿼리 중 (I/O 대기, IOWait 측정)
Thread B: 계산 중 (User time)
Thread C: 락 대기 중 (System time)

→ 한 스레드가 I/O 대기 중이어도 다른 스레드가 CPU를 쓸 수 있다.

하지만:
"IOWait > 20%"이면 충분한 스레드가 있는데도 CPU가 놀고 있다는 뜻
→ 이는 I/O가 느리다는 증거
→ 디스크, 네트워크, DB 병목 의심
```

---

## 💻 실전 실험

**시나리오: p99 응답시간이 갑자기 높아진 상황 진단**

```bash
#!/bin/bash
# cpu-diagnosis.sh

echo "=== CPU 병목 진단 시작 ==="
echo

# 1. 현재 CPU 상태 스냅샷
echo "1. 현재 CPU 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
top -bn1 | grep "Cpu(s)"

# 2. 60초 동안 CPU 추이 확인
echo
echo "2. 60초 CPU 추이 (1초 간격)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
mpstat -P ALL 1 60 > /tmp/mpstat.txt
# 분석: User, System, IOWait 각각의 평균값
awk 'NR>3 {
  user+=$4; sys+=$6; iowait+=$8
  count++
}
END {
  if (count > 0) {
    printf "평균 User: %.1f%%, System: %.1f%%, IOWait: %.1f%%\n",
           user/count, sys/count, iowait/count
  }
}' /tmp/mpstat.txt

# 3. Context Switch 빈도
echo
echo "3. Context Switch 분석"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
vmstat 1 10 | tail -5 | awk '{
  cs+=$10  # context switch 열
  count++
}
END {
  printf "평균 cs/sec: %.0f\n", cs/count
  if (cs/count > 5000) {
    print "⚠️  WARNING: Context switch 높음 (> 5000)"
  }
}'

# 4. 프로세스별 CPU 사용률
echo
echo "4. 프로세스별 CPU 사용률 TOP 10"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
ps aux --sort=-%cpu | head -11 | tail -10

# 5. 스레드 풀 상태 (Java 애플리케이션)
echo
echo "5. 스레드 풀 상태 (Tomcat)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics/executor.active | \
  jq '.measurements[0].value'

# 6. Thread Dump (락 경합 확인)
echo
echo "6. Thread Dump (BLOCKED 스레드 개수)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
APP_PID=$(jps | grep App | awk '{print $1}')
jstack $APP_PID | grep "tid=.*()" | wc -l
jstack $APP_PID | grep "BLOCKED" | wc -l

echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== CPU 병목 진단 시작 ===

1. 현재 CPU 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cpu(s): 45.0%us, 25.0%sy, 0.0%ni, 20.0%id, 10.0%wa, 0.0%hi, 0.0%si

2. 60초 CPU 추이 (1초 간격)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
평균 User: 42.5%, System: 28.0%, IOWait: 8.5%

3. Context Switch 분석
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
평균 cs/sec: 6500
⚠️  WARNING: Context switch 높음 (> 5000)

4. 프로세스별 CPU 사용률 TOP 10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
USER       PID %CPU %MEM    VSZ   RSS COMMAND
root       123  45.2  8.5 2000000 250000 java -jar app.jar
postgres   456  12.3  6.2 1500000 180000 postgres
nginx      789   2.1  0.5  100000  12000 nginx

5. 스레드 풀 상태 (Tomcat)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
125

6. Thread Dump (BLOCKED 스레드 개수)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
총 스레드: 287
BLOCKED: 45

=== 진단 완료 ===

🔍 진단 결과:
1. User time 42.5% + System time 28% = CPU 사용률 70.5%
2. System time 28%는 높음 (정상 <15%)
   → 원인: Context switch 6500/sec (정상 1000)
3. IOWait 8.5%는 중간 (모니터링 필요)
4. 활성 스레드 125개, BLOCKED 45개 (약 36% 블로킹)
   → 원인: 락 경합

추천 조치:
1. 스레드 풀 축소: 300 → 100 (context switch 감소)
2. 락 경합 확인: jstack 분석 → synchronized 제거
3. IOWait 8.5% 추적: DB 쿼리 성능 모니터링
```

---

## 📊 성능 비교

| 메트릭 | 정상 범위 | 높음 기준 | p99 영향 | 주요 원인 |
|--------|---------|---------|---------|---------|
| **User Time** | 40~70% | >80% | +200ms | 알고리즘 느림, GC 압력 |
| **System Time** | <10% | >20% | +300ms | Context switch 과다 |
| **IOWait** | 0~5% | >15% | +500ms | DB/디스크 I/O 느림 |
| **Context Switches** | <1000/sec | >5000/sec | +200~500ms | 스레드 풀 크기 큼 |
| **BLOCKED Threads** | <5% | >30% | +1000ms | 락 경합 심각 |

**실제 케이스 분석:**

```
케이스 1: User Time 높음 (75%), System Time 8%, IOWait 5%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상황: p99 = 800ms, CPU Util = 88%
원인 분석:
- User 75%는 애플리케이션 코드 자체가 CPU를 많이 씀
- 스레드 풀은 정상 (System 8%, cs < 2000)
- I/O는 빠름 (IOWait 5%)

진짜 원인: O(n²) 정렬 알고리즘을 매 요청마다 사용
해결책: Collections.sort() 사용 (O(n log n))
결과: User 75% → 35%, p99 800ms → 150ms (81% 개선)

케이스 2: System Time 높음 (35%), Context Switch 8000/sec
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상황: p99 = 1500ms, CPU Util = 60%
원인 분석:
- System 35%는 매우 높음 (context switch 과다)
- User 25%는 낮음 (실제 계산 적음)
- 스레드는 287개인데 실제 CPU는 4개

진짜 원인: 스레드 풀이 300으로 설정 (cores 4에 대해 75배)
해결책: 스레드 풀을 100으로 축소
결과: cs 8000 → 1200/sec, System 35% → 8%, p99 1500ms → 250ms (83% 개선)

케이스 3: IOWait 높음 (25%), DB 쿼리 느림
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상황: p99 = 2500ms, 디스크 I/O 대기
원인 분석:
- IOWait 25%는 높음 (I/O 병목)
- 디스크 await 80ms (정상 <10ms)
- SHOW PROCESSLIST에 Full Scan 쿼리 다수

진짜 원인: users 테이블에 인덱스 없음 (Full Table Scan)
해결책: CREATE INDEX idx_user_id ON users(user_id)
결과: 디스크 await 80ms → 3ms, IOWait 25% → 2%, p99 2500ms → 180ms (93% 개선)
```

---

## ⚖️ 트레이드오프

| 최적화 | 장점 | 단점 | 선택 기준 |
|--------|------|------|---------|
| **스레드 풀 축소** | Context switch 감소, System time ↓ | 일부 요청 큐잉 (하지만 총 응답시간은 개선) | User+System이 높을 때 |
| **알고리즘 최적화** | User time 직접 감소 | 코드 변경 복잡, 테스트 필요 | User time > 60%일 때 |
| **락 제거** | BLOCKED 스레드 감소, 응답시간 개선 | 동시성 버그 위험 | BLOCKED > 30%일 때 |
| **CPU 스케일 업** | 즉시 개선 가능 | 근본 원인 미해결, 비용 증가 | 긴급 상황, 시간 부족할 때 |

---

## 📌 핵심 정리

1. **CPU Time을 분해하라**
   - User Time: 애플리케이션 코드 실행 시간
   - System Time: 문맥 교환, 시스템 콜 시간
   - IOWait: I/O 작업 대기 시간 (CPU 아님)
   - 각 성분이 높은 이유가 다르므로 진단도 다르다

2. **Context Switch는 성능 저하의 주범**
   - cs > 5000/sec이면 성능 10~30% 저하
   - 원인: 스레드 풀 크기 > cores * 2
   - 해결: 스레드 풀을 cores * 2~4로 조정

3. **IOWait는 CPU 문제가 아님**
   - IOWait이 높아도 CPU 스케일 업은 도움 X
   - 진짜 원인: DB 쿼리, 디스크, 네트워크
   - 진단: iostat, DB slow query log, 네트워크 성능 확인

4. **측정 방법이 중요**
   - 한 번의 top 스냅샷은 신뢰 불가
   - mpstat 1 60 (60초, 1초 간격)으로 추세 파악
   - vmstat로 context switch 동시 확인
   - jstack으로 BLOCKED 스레드 확인

5. **의사결정 순서**
   - System time 높음 → 스레드 풀 축소
   - User time 높음 → 알고리즘 최적화, 캐싱
   - IOWait 높음 → DB/디스크/네트워크 최적화

---

## 🤔 생각해볼 문제

**Q1: System Time 20%, Context Switch 6500/sec이면 응답시간을 몇 %까지 개선할 수 있을까?**

<details>
<summary>해설 보기</summary>

**이론적 계산:**

Context switch 비용:
- 한 번당 0.5ms (캐시 미스 포함)
- cs/sec × 비용 = 6500 × 0.5ms = 3.25초 (초당 순손실)
- 4코어 기준: 3.25초 / (4 × 1000ms) = 81% 손실?

**하지만 실제는:**
- 모든 context switch가 손실을 일으키지는 않음
- 일부는 I/O 완료 신호로 인한 어쩔 수 없는 전환
- System time 20%는 "context switch + 다른 syscall"을 포함

**현실적 기댓값:**
```
Before: System time 20%, p99 = 1500ms
After: 스레드 풀 300 → 100
  Context switch: 6500/sec → 1200/sec (82% 감소)
  System time: 20% → 5% (75% 감소)
  
응답시간 개선:
- System time 20%의 약 50~70%를 개선할 수 있다
- p99: 1500ms → 1500ms × (1 - 0.15) = 1275ms (15% 개선)
- 또는 더 나은 경우: p99 → 800ms (47% 개선)

개선폭은 애플리케이션의 동시성 수준에 따라 다르다.
```

**결론:** System time 20%인 경우 응답시간 15~50% 개선 가능. 평균 30% 정도.

</details>

---

**Q2: IOWait 20%일 때 CPU를 2배로 늘리면 응답시간이 개선될까?**

<details>
<summary>해설 보기</summary>

**답: 아니다. 오히려 악화될 수도 있다.**

**이유:**

IOWait 20%의 의미:
- CPU가 I/O 완료를 기다리는 중
- CPU 자체는 한가한 상태 (다른 스레드에게 양보 가능)
- I/O가 느리므로 CPU를 늘려도 I/O 대기는 그대로

예시:
```
Before: 4 CPU cores, IOWait 20%, p99 = 2000ms
├─ Thread A: DB 쿼리 중 (I/O 대기, 100ms)
├─ Thread B: 계산 중 (User time 30ms)
├─ Thread C: 계산 중 (User time 40ms)
└─ Thread D: I/O 대기 (50ms)

After: 8 CPU cores로 증설 → I/O는 여전히 100ms
├─ Thread A: DB 쿼리 중 (I/O 대기, 100ms) ← 변화 없음!
├─ Thread B: 계산 중 (User time 30ms)
├─ ...
└─ 추가 CPU 코어: 유휴 상태

결론: I/O 대기는 줄어들지 않음. 응답시간 개선 없음.
```

**올바른 해결책:**
```
IOWait 높음 → 다음을 확인:
1. DB 쿼리 성능 (EXPLAIN ANALYZE)
2. 디스크 성능 (iostat -x, await)
3. 네트워크 성능 (ping, 대역폭)
4. 캐싱 (Redis, 로컬 캐시)

예: DB 쿼리를 100ms → 10ms로 최적화
→ IOWait 20% → 2%로 감소
→ p99 2000ms → 200ms (90% 개선)
```

**결론:** IOWait 높음은 CPU 스케일 업 신호가 아니다. I/O 병목을 해결해야 한다.

</details>

---

**Q3: 같은 애플리케이션 서버를 2대로 늘렸는데도 응답시간이 개선되지 않으면?**

<details>
<summary>해설 보기</summary>

**원인 분석:**

1단계: System 시간이 여전히 높은가?
```bash
# Before: 1 instance (4 core)
mpstat -P ALL 1 10
# System time 35%, Context switch 8000/sec

# After: 2 instances (각각 4 core)
mpstat -P ALL 1 10
# System time 35%, Context switch 8000/sec
# → 변화 없음 (인스턴스당 부하 동일)

원인: 각 인스턴스 내부의 스레드 풀이 여전히 크다
해결: 각 인스턴스의 스레드 풀을 cores * 2로 축소
```

2단계: DB가 병목은 아닌가?
```bash
# DB 연결 풀 고갈 확인
SHOW FULL PROCESSLIST;  # 150개 연결 모두 사용 중?
SHOW STATUS LIKE 'Threads_connected';  # 150/150

원인: 2개 인스턴스 × 100 스레드 = DB에 200 연결 시도
      하지만 DB 최대 연결 150 → 50개는 대기
      
해결: 각 인스턴스의 DB 연결 풀을 50으로 축소
      또는 DB 최대 연결을 300으로 증설
```

3단계: 로드 밸런싱이 균등한가?
```bash
# LB를 통한 요청 분배 확인
$ curl http://instance1:8080/metrics | jq '.http_requests_total'
$ curl http://instance2:8080/metrics | jq '.http_requests_total'

한쪽이 80%, 다른 한쪽이 20%?
→ 로드 밸런싱 설정 문제 (sticky session, 가중치 등)
```

**결론:** 단순 스케일 아웃은 근본 원인이 해결되지 않으면 효과 없다. USE 방법론으로 각 인스턴스의 병목을 먼저 해결해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: USE 방법론 — 체계적으로 병목을 찾는 5분 체크리스트](./01-use-methodology.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메모리 병목 분석 — JVM Heap vs Off-Heap 구분 ➡️](./03-memory-bottleneck.md)**

</div>
