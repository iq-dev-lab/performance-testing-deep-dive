# 04. DB 병목 분석 — HikariCP와 Connection Pool

---

## 🎯 핵심 질문

데이터베이스가 병목인지 어떻게 알 수 있을까?
- Connection pool이 대기 중인가? (pending > 0)
- DB 내에서 실행 중인 쿼리가 몇 개인가?
- 느린 쿼리가 전체 응답시간을 잡아당기는가?

**DB 병목은 보통 애플리케이션 서버가 아니라 DB 자체에서 문제를 드러낸다.**

---

## 🔍 왜 이 개념이 실무에서 중요한가

**문제 상황:**

```
상황: p99 응답시간 2000ms
애플리케이션 서버: CPU 50%, Memory 70%, I/O 정상
개발자: "서버는 멀쩡한데... DB?"

SHOW PROCESSLIST 확인:
→ 300개 연결 중 250개가 "SELECT ... FROM users" 실행 중
→ 각 쿼리 실행 시간: 5~10초 (매우 느림!)

원인:
1. Full Table Scan (인덱스 없음)
2. Lock Contention (높은 동시성에서 대기)
3. Connection Pool 고갈

결과:
- 신규 요청: 30초 동안 connection 대기 (queue에 쌓임)
- 사용자 입장: "사이트 먹통"
- 실제 응답: 응답시간 30~40초

해결:
1. 인덱스 추가 (쿼리 5초 → 50ms)
2. Connection Pool 최적화
3. 슬로우 쿼리 분리 (비동기 처리)
```

**올바른 진단 프로세스:**

```
p99 응답시간 높음
├─ 애플리케이션 CPU/Memory 확인 (정상)
├─ iostat 확인 (Disk await 정상)
└─ → DB를 의심

DB 확인:
├─ SHOW FULL PROCESSLIST (실행 중인 쿼리 개수)
├─ SHOW STATUS LIKE 'Threads_connected' (연결 수)
├─ SHOW STATUS LIKE 'Threads_running' (활성 연결)
├─ SELECT * FROM performance_schema.events_statements_summary_by_digest ORDER BY SUM_TIMER_WAIT (느린 쿼리)
└─ → 병목 확인 후 대응

대응:
1. 인덱스 추가
2. Connection Pool 튜닝
3. 쿼리 최적화
4. Lock 경합 해소
```

**실무 효과:**
- 잘못된 진단: 애플리케이션 메모리 증설 → 효과 없음
- 올바른 진단: DB 쿼리 최적화 → p99 90% 개선

---

## 😱 흔한 실수 (Before)

```bash
# 잘못된 접근 1: Connection Pool 개수만 본다
$ curl http://localhost:8080/actuator/metrics/hikaricp.connections
# active=50, idle=0, pending=0, size=100

# 결론: "Connection이 있으니 괜찮다"
# → 근데 대기열에 대기 중인 요청이 없나?
# → 각 연결이 얼마나 오래 사용되는가?
# → pending=0이지만 connection 모두 slow query 실행 중?

# 잘못된 접근 2: 느린 쿼리만 찾는다
SHOW FULL PROCESSLIST;
# Query가 2초 이상이면 느림
KILL QUERY ...;

# 결론: "느린 쿼리 죽이니까 응답시간 개선"
# → 근데 그 쿼리가 필요한 쿼리라면?
# → 근본적으로는 최적화되지 않음
# → 쿼리가 왜 느린지 분석 (인덱스 없음? Lock?)

# 잘못된 접근 3: Lock Contention을 모른다
SHOW ENGINE INNODB STATUS;
# 근데 이거 어떻게 읽는거?
# → Lock row 개수, 대기 중인 트랜잭션 개수 확인 필요

# 잘못된 접근 4: 동시성 테스트 없이 배포
# 개발: Connection Pool 10으로 테스트 (정상)
# 프로덕션: 동시 300 사용자 (connection 100개 필요)
# → Connection 부족 (queue에 대기 중인 요청 1000개)
```

**실제 사례:**

```
상황: 새 기능 배포 후 응답시간 3배 증가

1차 진단:
→ "CPU 사용률 80%니까 CPU 증설하자"
→ CPU 2개 → 4개 증설 (비용 증가)
→ 응답시간 여전히 3배 (개선 없음)

2차 상세 분석:
SHOW FULL PROCESSLIST;
→ users 테이블에 300 연결이 "SELECT * FROM users WHERE ..." 실행 중
→ 각 쿼리 시간: 8~12초 (매우 느림)

EXPLAIN SELECT * FROM users WHERE status='active' AND created_at > '2024-01-01';
→ type=ALL (Full Table Scan)
→ rows=5000000 (5백만 행을 모두 스캔!)

3차 해결:
CREATE INDEX idx_users_status_created ON users(status, created_at);

성능 향상:
SHOW PROCESSLIST: 쿼리 시간 8초 → 50ms
p99 응답시간: 3000ms → 200ms (93% 개선!)
CPU 사용률: 80% → 30% (CPU도 자동으로 낮아짐!)

결론:
- CPU 증설은 불필요했음 (인덱스 추가가 정답)
- 비용 낭비 (CPU 비용) + 문제 미해결
```

---

## ✨ 올바른 접근 (After)

### 1단계: Connection Pool 상태 확인

**HikariCP 메트릭 모니터링:**

```bash
# 1. Actuator 엔드포인트 확인
$ curl http://localhost:8080/actuator/metrics/hikaricp.connections | jq '.'
{
  "name": "hikaricp.connections",
  "measurements": [
    {"statistic": "VALUE", "value": 100}  # 전체 커넥션
  ]
}

$ curl http://localhost:8080/actuator/metrics/hikaricp.connections.active | jq '.'
{"measurements": [{"statistic": "VALUE", "value": 45}]}

$ curl http://localhost:8080/actuator/metrics/hikaricp.connections.idle | jq '.'
{"measurements": [{"statistic": "VALUE", "value": 55}]}

$ curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending | jq '.'
{"measurements": [{"statistic": "VALUE", "value": 0}]}

분석:
- 전체: 100개
- 활성: 45개
- 유휴: 55개
- 대기(pending): 0개

판단: 연결이 충분한 상태? → 아니다! 데이터가 더 필요하다.
```

**더 상세한 메트릭 확인:**

```bash
# HikariCP 설정 조회
$ curl http://localhost:8080/actuator/beans | \
  jq '.contexts.application.beans[] | select(.bean | contains("Hikari"))'

# 또는 로그 확인
$ grep -i "hikari\|connection" application.log | tail -20

# Connection 획득 시간 모니터링
$ curl http://localhost:8080/actuator/metrics/hikaricp.connections.acquire | jq '.'
{"measurements": [{"statistic": "TOTAL", "value": 45000}]}  # ms 단위

평균 connection 획득 시간 = 45000ms / 총 요청 수
> 10ms이면 높음 (connection 부족 신호)
```

### 2단계: DB 내 실행 중인 쿼리 확인

**MySQL (InnoDB):**

```sql
-- 1. 현재 실행 중인 모든 쿼리 확인
SHOW FULL PROCESSLIST;
-- Command, State, Time (초), Info(쿼리)
-- 출력:
-- Id | User | Host | db | Command | Time | State | Info
-- 123 | app | 10.0.0.1 | prod | Query | 8 | Sending data | SELECT * FROM users WHERE ...
-- 124 | app | 10.0.0.2 | prod | Query | 5 | Sending data | SELECT * FROM orders WHERE ...

-- 분석:
-- Time > 5초이면 느린 쿼리
-- 같은 쿼리가 많으면 동시성 문제

-- 2. 트랜잭션 상태 확인
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST;

-- 3. Lock 대기 상황
SHOW ENGINE INNODB STATUS;
-- Output:
-- TRANSACTIONS
-- Trx id counter 12345
-- Purge done for trx's n:o < 12340
-- List of transactions <active>, 1 transactions, process no 456, OS thread id ...
--
-- TRX_ID: 12341, TRX_STATUS: LOCK WAIT
-- TRX_REQUESTED_LOCK_TYPE: RECORD
-- TRX_REQUESTED_LOCK_MODE: X (Exclusive = Write Lock)
-- TRX_IS_READ_ONLY: NO
-- TRX_ROWS_LOCKED: 1

-- Lock 대기가 있으면 "Lock Contention" 발생

-- 4. 연결 상태 요약
SHOW STATUS LIKE 'Threads%';
-- Threads_cached: 유휴 연결 스레드 수
-- Threads_connected: 현재 연결 수
-- Threads_created: 생성된 총 스레드 수
-- Threads_running: 활성 중인 스레드 (쿼리 실행 중)

-- 예:
-- Threads_connected: 250 / max_connections: 300 (83% 사용)
-- Threads_running: 200 (대부분 활성, pool 부족 신호)
```

### 3단계: 느린 쿼리 분석

**Slow Query Log 활성화:**

```sql
-- 1. Slow query log 활성화
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 1초 이상의 쿼리 기록

-- 2. Slow query log 위치 확인
SHOW VARIABLES LIKE 'slow_query_log_file';
-- /var/log/mysql/slow-query.log

-- 3. 분석 도구 사용
-- mysqldumpslow: MySQL 기본 제공
-- pt-query-digest: Percona Toolkit (더 강력)
```

**느린 쿼리 분석 방법:**

```bash
# 1. mysqldumpslow 사용
$ mysqldumpslow -s c -t 10 /var/log/mysql/slow-query.log
# -s c: 쿼리 개수로 정렬 (가장 많은 쿼리)
# -t 10: 상위 10개
# -s at: 평균 시간으로 정렬

# Output:
# Count: 5000  Time=1.23s (6150s)  Lock=0.01s (50s)  Rows_sent=100 (500000)
# SELECT * FROM users WHERE status='active' LIMIT 100;

# 분석:
# - Count 5000: 이 쿼리가 5000번 실행됨
# - Time 1.23s: 평균 1.23초 (정상: <100ms)
# - Lock 0.01s: Lock 시간 (높지 않음)
# - Rows_sent 100: 반환 행 수

# 2. pt-query-digest 사용 (더 상세)
$ pt-query-digest /var/log/mysql/slow-query.log

# Output:
# Profile
# Rank Query ID Response time Calls R/Call Item
# ==== ==================== ================ ===== ======= ===========
#    1 0xABC123... 5000.00 96.1% 5000 1.0000 SELECT FROM users
#    2 0xDEF456... 100.00  1.9%  100 1.0000 SELECT FROM orders
```

**느린 쿼리 최적화 (EXPLAIN ANALYZE):**

```sql
-- 1. 쿼리 실행 계획 확인
EXPLAIN ANALYZE
SELECT * FROM users u
WHERE u.status = 'active'
AND u.created_at > '2024-01-01'
LIMIT 100;

-- Output:
-- -> Limit: 100 row(s) (actual time=8234.567..8234.890 rows=100)
--   -> Filter: (u.`status` = 'active') (cost=123456.78 rows=500000)
--     -> Table scan on u (cost=123456.78 rows=5000000)
--
-- "actual time=8234" = 8초 (느림!)
-- "Table scan" = Full Table Scan (인덱스 없음)

-- 2. 인덱스 생성
CREATE INDEX idx_users_status_created 
ON users(status, created_at);

-- 3. 최적화 후 확인
EXPLAIN ANALYZE
SELECT * FROM users u
WHERE u.status = 'active'
AND u.created_at > '2024-01-01'
LIMIT 100;

-- Output:
-- -> Limit: 100 row(s) (actual time=45.123..45.890 rows=100)
--   -> Index range scan on u using idx_users_status_created
--
-- "actual time=45" = 45ms (8초 → 45ms, 180배 개선!)
```

### 4단계: Connection Pool 크기 결정

**HikariCP 설정:**

```yaml
# application.yml 또는 application.properties
spring:
  datasource:
    hikari:
      # 기본값: cores * 2 + reserve
      maximum-pool-size: 20  # 동시 연결 최대값
      minimum-idle: 5        # 유휴 연결 최소값 (워밍)
      connection-timeout: 30000  # 30초 대기 후 타임아웃
      idle-timeout: 600000       # 10분 유휴 후 연결 종료
      max-lifetime: 1800000      # 30분 최대 유지시간
      auto-commit: true
      leak-detection-threshold: 60000  # 60초 이상 사용 중이면 경고

# 권장값:
# maximum-pool-size = cores * 2 + buffer (버퍼: 10~20)
# 예: 4 cores → 8 + 10 = 18

# 동시 사용자 수에 따른 조정:
# 동시 사용자 100명, 평균 쿼리 시간 100ms
# → 필요한 pool 크기 = (100 사용자 * 100ms) / 1000 = 10개
# → 여유도 고려: 10 * 2 = 20개
```

**동적 모니터링으로 최적값 찾기:**

```bash
# 부하 테스트 중 connection pool 모니터링
watch -n 1 'curl -s http://localhost:8080/actuator/metrics | grep hikaricp'

# 지표:
# hikaricp.connections.active: 최대값을 찾는다
# hikaricp.connections.pending: > 0이면 pool 부족

# 예시:
# load 100 RPS, pool 20: active 15, pending 0 → 충분
# load 100 RPS, pool 10: active 10, pending 100+ → 부족 (pool 증설 필요)
```

---

## 🔬 내부 동작 원리

### Connection Pool의 동작

```
요청 1    요청 2    요청 3    요청 4
  │         │         │         │
  └─────────┴─────────┴─────────┘
         Connection Pool
         ┌─────────────────┐
         │ Idle Connection │  ← 대기 중
         │ [1][2][3][4][5] │
         └─────────────────┘
                  │
            ┌─────┴─────┬─────────┬─────────┐
            │           │         │         │
         DB SQL   DB SQL   DB SQL   DB SQL
         (in)     (in)      (in)      (in)
         
요청 처리 흐름:
1. 요청 1: Pool에서 유휴 연결 [1] 획득 → DB 쿼리 실행 (100ms)
2. 요청 2: Pool에서 유휴 연결 [2] 획득 → DB 쿼리 실행 (50ms)
3. 요청 3: Pool에서 유휴 연결 [3] 획득 → DB 쿼리 실행 (200ms)
4. 요청 4: 유휴 연결 없음 → Queue에서 대기

50ms 후:
→ 요청 2 완료, 연결 [2] 반환 (Available)
→ 요청 4가 [2] 획득 → DB 쿼리 실행

연결 부족 시나리오:
- Pool size: 5개
- 동시 요청: 10개
- 평균 쿼리 시간: 1초

→ 처음 5개 요청: 연결 [1]~[5] 사용
→ 다음 5개 요청: Queue에서 1초 대기
→ 사용자 입장: "응답시간 1초 + 쿼리 시간 1초 = 2초"
```

### Lock Contention의 영향

```
쿼리 1 (Read): SELECT * FROM users WHERE id=1
쿼리 2 (Write): UPDATE users SET age=30 WHERE id=1

Timeline:
T0: 쿼리 1 시작 (Shared Lock 획득)
T100ms: 쿼리 2 시작 (Exclusive Lock 대기)
T200ms: 쿼리 1 완료 (Lock 해제)
T200ms: 쿼리 2 시작 (Exclusive Lock 획득)
T250ms: 쿼리 2 완료

결과:
- 쿼리 1 응답시간: 200ms (정상)
- 쿼리 2 응답시간: 250 - 100 = 150ms (100ms 대기)

응답시간 증가: 50% (쿼리 2 관점)

동시성이 높으면:
- 여러 쓰기 쿼리가 같은 행에 접근
- 모두 Lock 대기
- 평균 응답시간 3배 이상 증가
```

### Index의 성능 차이

```
쿼리: SELECT * FROM users WHERE status='active' AND created_at > '2024-01-01' LIMIT 100

시나리오 1: Index 없음 (Full Table Scan)
┌──────────────────────────────────┐
│ users 테이블 (500만 행)          │
│ [Row 1] [Row 2] ... [Row 5000000]│
│  검사    검사   ...   검사 (모두)│
└──────────────────────────────────┘
처리: 500만 행 모두 확인 → 조건 만족하는 100행 반환
시간: 8초

시나리오 2: Index 있음 (B-Tree Index)
┌─────────────────────────────────┐
│ Index: (status, created_at)     │
│       ┌─ active, 2024-01-01 ─┐  │
│       │  ├─ [Ptr 1234]       │  │
│       │  ├─ [Ptr 5678]       │  │
│       │  ├─ [Ptr 9012] ← 필요한 데이터만
│       │  └─ ...              │  │
│       └─ active, 2024-01-02 ─┘  │
└─────────────────────────────────┘
처리: Index에서 시작점 찾기 → 필요한 행만 접근
시간: 45ms (180배 개선!)
```

---

## 💻 실전 실험

**시나리오: p99 응답시간 2000ms, Connection pool은 충분한 것 같은데 느린 상황**

```bash
#!/bin/bash
# db-diagnosis.sh

echo "=== DB 병목 진단 시작 ==="
echo

# 1. Connection Pool 상태
echo "1. Connection Pool 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s http://localhost:8080/actuator/metrics | \
  jq '.names[] | select(contains("hikaricp"))' | \
  while read metric; do
    value=$(curl -s "http://localhost:8080/actuator/metrics/$metric" | jq '.measurements[0].value')
    echo "$metric: $value"
  done

# 2. DB 연결 수 및 실행 중인 쿼리
echo
echo "2. DB 실행 중인 쿼리"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
mysql -u root -p$MYSQL_ROOT_PASSWORD -e "SHOW FULL PROCESSLIST\G" | \
  grep -E "Id:|Command:|Time:|State:|Info:" | head -50

# 3. 느린 쿼리
echo
echo "3. 느린 쿼리 (1초 이상)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
mysql -u root -p$MYSQL_ROOT_PASSWORD -e "
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST 
WHERE COMMAND != 'Sleep' AND TIME > 1 
ORDER BY TIME DESC;
" | column -t

# 4. Lock 상태
echo
echo "4. Lock 대기 상황"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
mysql -u root -p$MYSQL_ROOT_PASSWORD -e "SHOW ENGINE INNODB STATUS\G" | \
  grep -A 20 "TRANSACTIONS"

# 5. 느린 쿼리 로그 분석 (최근 10개)
echo
echo "5. 느린 쿼리 TOP 10 (실행 횟수)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
if [ -f /var/log/mysql/slow-query.log ]; then
  mysqldumpslow -s c -t 10 /var/log/mysql/slow-query.log | head -20
fi

# 6. Connection 상태 요약
echo
echo "6. DB 연결 상태"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
mysql -u root -p$MYSQL_ROOT_PASSWORD -e "
SHOW STATUS LIKE 'Threads%';
SHOW STATUS LIKE 'Questions';
SHOW STATUS LIKE 'Slow_queries';
"

echo
echo "=== 진단 완료 ==="
```

**실행 결과:**

```
=== DB 병목 진단 시작 ===

1. Connection Pool 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
hikaricp.connections: 20
hikaricp.connections.active: 18
hikaricp.connections.idle: 2
hikaricp.connections.pending: 50

⚠️  WARNING: pending=50 (대기 중인 요청 50개!)

2. DB 실행 중인 쿼리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Id: 1234
Command: Query
Time: 8
State: Sending data
Info: SELECT * FROM users WHERE status='active' LIMIT 100000

Id: 1235
Command: Query
Time: 6
State: Sending data
Info: SELECT * FROM orders WHERE user_id=123

... (총 18개 쿼리 실행 중)

3. 느린 쿼리 (1초 이상)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ID USER HOST DB COMMAND TIME STATE
1234 app 10.0.0.1 prod Query 8 Sending data
1235 app 10.0.0.2 prod Query 6 Sending data
1236 app 10.0.0.3 prod Query 12 Sending data

4. Lock 대기 상황
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TRANSACTIONS
Trx id counter 12345
List of transactions <active>, 18 transactions, process no 1234

Trx id 12340, state RUNNING
Trx id 12341, state LOCK WAIT ← Lock 대기
  TRX_REQUESTED_LOCK_TYPE: RECORD
  TRX_REQUESTED_LOCK_MODE: X

5. 느린 쿼리 TOP 10 (실행 횟수)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Count: 5000  Time=8.23s (41150s)  Lock=0.01s (50s)
SELECT * FROM users WHERE status='active' LIMIT 100000;

Count: 2000  Time=5.12s (10240s)  Lock=0.05s (100s)
SELECT * FROM orders WHERE user_id=? LIMIT 1000;

6. DB 연결 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Variable_name: Threads_cached, Value: 0
Variable_name: Threads_connected, Value: 20 (max: 300)
Variable_name: Threads_running, Value: 18
Variable_name: Questions, Value: 1234567
Variable_name: Slow_queries, Value: 456

=== 진단 완료 ===

🔍 진단 결과:
1. Connection Pool 병목
   - pending=50: 50개 요청이 connection 대기 중
   - max pool=20, active=18 (모두 사용 중)
   - 필요 pool size = 20 + 여유 = 30~40

2. 느린 쿼리 병목 (메인 문제)
   - users 테이블 쿼리: 8.23초 (정상: <100ms)
   - 5000번 실행으로 누적 41150초 손실
   - EXPLAIN ANALYZE 결과: Full Table Scan (인덱스 없음)

3. Lock 대기
   - 1개 트랜잭션이 Lock 대기 중
   - 영향은 적지만 누적되면 문제

다음 단계:
1. 인덱스 추가 (쿼리 8초 → 50ms, 160배 개선)
2. Connection Pool 증설 (20 → 40)
3. 쿼리 최적화 (LIMIT 100000은 너무 많음)
```

---

## 📊 성능 비교

| 메트릭 | 정상 범위 | 병목 기준 | p99 영향 | 해결책 |
|--------|---------|---------|---------|--------|
| **Connection pending** | 0 | >10 | +100ms/개 | Pool size ↑ |
| **Query execution** | <100ms | >1000ms | +900ms | Index, 쿼리 최적화 |
| **Threads_running** | < max_conn/2 | > max_conn*80% | +500ms | Pool ↑, 쿼리 최적화 |
| **Lock wait** | 0ms | >100ms | +100ms~1000ms | Lock 전략 변경 |
| **Full GC pause** | - | - | - | (메모리 섹션 참고) |

**실제 케이스:**

```
케이스 1: Full Table Scan 쿼리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  쿼리 시간: 8초
  Connection pending: 50
  p99: 2000ms
  
원인: SELECT * FROM users WHERE status='active' (Index 없음)

EXPLAIN:
type: ALL (Full Table Scan)
rows: 5000000

After: CREATE INDEX idx_status ON users(status)
  쿼리 시간: 45ms
  Connection pending: 0
  p99: 100ms
  개선율: 쿼리 95% 개선, p99 95% 개선

케이스 2: Connection Pool 부족
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  Pool size: 10
  Active: 10, Idle: 0
  Pending: 100
  p99: 5000ms (1초 pool 대기 + 4초 쿼리)
  
After: Pool size 10 → 30
  Active: 20, Idle: 10
  Pending: 0
  p99: 500ms (pool 대기 없음)
  개선율: p99 90% 개선

케이스 3: Lock Contention
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Before:
  같은 행에 대한 읽기/쓰기 동시 접근
  각 쿼리 Lock 대기: 100~500ms
  p99: 1000ms
  
After: 
  - 배치 처리로 동시성 감소
  - 또는 낙관적 Lock (Version) 사용
  - p99: 150ms
  개선율: p99 85% 개선
```

---

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 | 선택 기준 |
|------|------|------|---------|
| **Index 추가** | 쿼리 성능 대폭 개선 (100배+) | 쓰기 성능 약간 ↓, 스토리지 ↑ | 항상 먼저 고려 |
| **Connection Pool ↑** | 즉시 개선 가능 | DB 부하 증가, 메모리 사용 ↑ | pending > 0일 때 |
| **쿼리 최적화** | 근본 원인 해결 | 시간 오래 걸림, 위험 있음 | 정상 상황 |
| **Read Replica 추가** | 읽기 성능 확장 | 복잡도 증가, 일관성 이슈 | 읽기 부하 극심할 때 |

---

## 📌 핵심 정리

1. **Connection Pool 상태를 먼저 확인**
   - pending > 0이면 즉시 pool size 증설
   - 기준: (동시 사용자 수 × 평균 쿼리 시간/1000) + 여유 20%

2. **느린 쿼리가 병목의 90%**
   - SHOW FULL PROCESSLIST로 실행 중인 쿼리 확인
   - EXPLAIN ANALYZE로 인덱스 여부 확인
   - Full Table Scan이면 인덱스 추가

3. **Index는 마법탄**
   - Full Table Scan → Index Range Scan: 100배 이상 개선
   - 단순하고 효과적
   - 단점은 미미 (쓰기 성능 약 5~10% 저하)

4. **Lock Contention은 뒤늦게 나타난다**
   - 동시성 테스트 필수
   - SHOW ENGINE INNODB STATUS로 진단
   - 배치 처리, 낙관적 Lock 등으로 해결

5. **의사결정 순서**
   ```
   DB 병목 의심
   ├─ Connection pending > 0?
   │  ├─ Yes: Pool size 증설 (즉시)
   │  └─ No: 다음 단계
   ├─ 실행 중인 쿼리 > 1초?
   │  ├─ Yes: 
   │  │  ├─ EXPLAIN에서 Full Scan?
   │  │  │  ├─ Yes: Index 추가 (즉시)
   │  │  │  └─ No: 쿼리 최적화 (코드 검토)
   │  │  └─ No: 다음 단계
   │  └─ No: DB 병목 아님. CPU, 메모리, 네트워크 확인
   ```

---

## 🤔 생각해볼 문제

**Q1: Connection pending이 0이고, 실행 중인 쿼리도 1초 이하인데 응답시간이 500ms이면?**

<details>
<summary>해설 보기</summary>

**DB는 병목이 아니다.**

**원인 분석:**

```
응답시간: 500ms
├─ DB 쿼리 시간: 100ms (확인됨)
├─ 네트워크 왕복: 20ms
├─ 애플리케이션 처리: ???
└─ 나머지: 380ms

380ms는 어디로?
```

**다른 병목 의심:**

1. **애플리케이션 자체 계산**
   ```
   쿼리: 100ms
   결과 처리 (JSON 직렬화, 비즈니스 로직): 300ms
   네트워크 전송: 100ms
   = 500ms
   ```

2. **외부 API 호출**
   ```
   DB 쿼리: 100ms
   외부 API 호출: 300ms (Timeout 없이 대기)
   응답 반환: 100ms
   = 500ms
   ```

3. **CPU 병목**
   ```
   쿼리는 빠르지만 전체 시스템이 느림
   CPU Saturation 높음, Context switch 과다
   모든 작업이 느려짐
   ```

**진단 방법:**

1. 애플리케이션 프로파일링 (Spring Boot Actuator)
2. CPU 상태 확인 (mpstat, top)
3. 외부 API 호출 시간 측정
4. 네트워크 성능 확인

**결론:** DB는 이미 최적화되어 있다. 다른 부분을 봐야 한다.

</details>

---

**Q2: Index를 추가했는데도 쿼리 시간이 개선되지 않으면?**

<details>
<summary>해설 보기</summary>

**Index가 사용되지 않을 수 있다.**

**원인 분석:**

```sql
-- 예: 인덱스 있지만 사용 안 됨
CREATE INDEX idx_users_status ON users(status);

EXPLAIN SELECT * FROM users 
WHERE UPPER(status) = 'ACTIVE';
-- Index not used! (UPPER() 함수 적용으로 Index Range Scan 불가)

EXPLAIN SELECT * FROM users 
WHERE status = 'ACTIVE' OR created_at > '2024-01-01';
-- Multiple OR conditions: Index 사용 여부 불명확 (Optimizer 판단)

EXPLAIN SELECT * FROM users WHERE id LIKE '123%';
-- LIKE는 Index 사용 가능 (Leading wildcard 아닌 경우)

EXPLAIN SELECT * FROM users WHERE id LIKE '%123';
-- LIKE % at start: Index 사용 안 됨 (Full Scan)
```

**Index가 사용되지 않는 이유:**

1. **쿼리가 함수를 적용**
   ```sql
   WHERE UPPER(status) = 'ACTIVE'  -- Index 미사용
   올바른: WHERE status = 'ACTIVE'  -- Index 사용
   ```

2. **데이터 타입 불일치**
   ```sql
   WHERE user_id = '123'  -- Index 미사용 (문자열)
   올바른: WHERE user_id = 123  -- Index 사용 (정수)
   ```

3. **복합 Index의 순서 위반**
   ```sql
   CREATE INDEX idx_users ON users(status, created_at);
   
   WHERE status = 'ACTIVE' AND created_at > '2024-01-01'
   -- Index 사용 OK
   
   WHERE created_at > '2024-01-01'  -- Index 미사용
   -- Index는 (status, created_at) 순서이므로 created_at만으로는 Index 못 씀
   ```

4. **Index 선택도 문제**
   ```sql
   -- Option 1: Full Scan이 더 빠를 수도
   EXPLAIN SELECT * FROM small_table WHERE status='active';
   -- 행이 100개뿐이면 Full Scan이 Index Scan보다 빠를 수 있음
   
   -- Option 2: 옵티마이저가 여러 Index 중 선택
   EXPLAIN SELECT * FROM users 
   WHERE user_id = 1 AND status = 'active';
   -- Index user_id 또는 Index status 중 선택
   -- 잘못된 선택이면 느림 (Query Hint 사용: FORCE INDEX)
   ```

**해결책:**

```sql
-- 1. Index 재생성
ALTER TABLE users DROP INDEX idx_users_status;
CREATE INDEX idx_users_status ON users(status, created_at);
ANALYZE TABLE users;  -- 통계 업데이트

-- 2. Query Hint 사용 (강제로 Index 사용)
SELECT * FROM users 
FORCE INDEX (idx_users_status)
WHERE status = 'ACTIVE';

-- 3. Index 강제 병합 (복합 조건)
SELECT * FROM users 
USE INDEX (idx_status, idx_created)
WHERE status = 'ACTIVE' AND created_at > '2024-01-01';

-- 4. 쿼리 수정 (함수 제거)
-- Before: WHERE UPPER(status) = 'ACTIVE'
-- After: WHERE status = 'ACTIVE'  (data 저장 시 대문자로)
```

**결론:** Index가 있어도 쿼리 작성 방식에 따라 사용되지 않을 수 있다. EXPLAIN으로 항상 확인하자.

</details>

---

**Q3: Connection Pool을 100개까지 늘렸는데도 p99가 개선 안 되면?**

<details>
<summary>해설 보기</summary>

**Connection Pool이 병목이 아니라는 뜻이다.**

**원인 분석:**

1. **DB 자체가 병목**
   ```
   Pool 100개 → DB도 처리 필요
   DB가 동시에 100개 쿼리를 처리할 수 없으면
   각 쿼리가 대기 (I/O, Lock 등)
   
   Pool이 커도 도움 X
   진짜 해결책: 쿼리 최적화 (Index, 쿼리 단순화)
   ```

2. **DB CPU/Memory 부족**
   ```
   여러 쿼리가 동시 실행되면서 DB 자체 부하 증가
   CPU, Memory 부족으로 성능 저하
   
   진짜 해결책: DB 리소스 증설 또는 Read Replica 추가
   ```

3. **쿼리 자체가 느림**
   ```
   Pool: 대기 시간 없음 (pending = 0)
   DB: 각 쿼리 시간 2초 이상
   
   이미 Pool은 최적화됨.
   쿼리를 빠르게 하는 게 목표.
   ```

**진단:**

```bash
# 1. Connection pending 확인
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
# = 0이면 Pool은 충분하다

# 2. DB 실행 중인 쿼리 확인
SHOW FULL PROCESSLIST;
# 각 쿼리 시간이 > 1초면 쿼리 최적화 필요

# 3. DB 부하 확인
SHOW STATUS LIKE 'Questions';  # 초당 쿼리 수
SHOW STATUS LIKE 'Threads_running';  # 활성 쿼리

# 높으면 DB 자체 부족
```

**결론:** Connection Pool을 무한정 늘릴 수는 없다 (메모리 증가). 일정 크기(100개) 이상이면 병목은 다른 곳이다.

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 병목 분석 — JVM Heap vs Off-Heap 구분](./03-memory-bottleneck.md)** | **[홈으로 🏠](../README.md)** | **[다음: 외부 API 호출 병목 — Timeout과 Circuit Breaker ➡️](./05-external-api-bottleneck.md)**

</div>
