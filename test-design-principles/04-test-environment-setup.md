# 04. 테스트 환경 구성 — Prod-like 원칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 로컬 환경의 테스트 결과를 운영에 그대로 적용하면 안 되는가?
- 데이터 볼륨이 성능에 미치는 영향은 얼마나 큰가?
- Prod-like 환경을 구성할 때 반드시 고려해야 할 요소는 무엇인가?
- Docker Compose로 일관된 실험 환경을 어떻게 구성하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"로컬에서는 빠른데 운영에서 느려요"는 테스트 환경과 운영 환경의 차이에서 기인하는 가장 흔한 문제다. 테스트 환경이 운영 환경과 다르면 테스트 결과는 의미 없는 숫자가 된다. 특히 데이터 볼륨 차이는 가장 과소평가되는 요소다 — 빈 DB와 1억 건 DB의 쿼리 성능은 수백 배 차이가 날 수 있다.

---

## 😱 흔한 실수 (Before — 환경 차이를 무시하는 접근)

```
상황: 개발 서버에서 성능 테스트 후 운영 배포

테스트 환경:
  서버: 개발자 맥북 (M2, 32GB RAM)
  DB: 도커 컨테이너, 데이터 1,000건
  네트워크: localhost (0ms latency)
  JVM: -Xmx256m

운영 환경:
  서버: AWS t3.medium (2 vCPU, 4GB RAM)
  DB: RDS MySQL (다른 AZ, 네트워크 레이턴시 1~3ms)
  데이터: 5,000만 건
  JVM: -Xmx2g

결과:
  개발 환경 테스트: p99 45ms ← 아무 인덱스 없어도 빠름 (1,000건)
  운영 배포 후: p99 12,000ms ← 5,000만 건 Full Table Scan

교훈:
  - 데이터 1,000건 vs 5,000만 건: 성능 수백 배 차이
  - localhost vs 네트워크 레이턴시: 매 쿼리마다 1~3ms 추가
  - 테스트 서버 스펙이 높으면 운영의 병목이 숨겨짐
```

---

## ✨ 올바른 접근 (After — Prod-like 환경 구성)

```
Prod-like 환경 체크리스트:

[하드웨어]
  ✅ 운영 서버와 동일한 CPU 코어 수 (중요: JVM 스레드 풀 크기 영향)
  ✅ 운영과 동일한 메모리 및 JVM 힙 설정
  ✅ 운영 GC 설정 동일하게 (-XX:+UseG1GC 등)

[데이터]
  ✅ 최소 운영 데이터의 10% 이상 (권장: 동일)
  ✅ 데이터 분포가 운영과 유사 (skew 재현)
  ✅ 인덱스 통계 최신화 (ANALYZE TABLE)

[네트워크]
  ✅ DB와 앱 서버 간 네트워크 레이턴시 재현
  ✅ 외부 API 호출은 Mock 또는 WireMock으로 대체

[설정]
  ✅ 운영과 동일한 Connection Pool 설정
  ✅ 운영과 동일한 캐시 설정 (캐시 크기, TTL)
  ✅ 운영과 동일한 로그 레벨 (DEBUG → INFO 차이가 성능 영향)
```

---

## 🔬 내부 동작 원리

### 데이터 볼륨이 성능에 미치는 영향

```
인덱스 없는 쿼리: SELECT * FROM orders WHERE user_id = 1

데이터 건수별 Full Table Scan 시간:
  1,000건:     2ms   ← 테스트에서는 문제없어 보임
  10만 건:    18ms
  100만 건:  180ms
  1,000만 건: 1,800ms
  5,000만 건: 9,200ms ← 운영에서 장애

→ 데이터 1,000건으로 테스트하면 인덱스 필요성 자체가 드러나지 않음
→ 운영 배포 후에야 심각한 성능 문제 발견

인덱스 있는 쿼리:
  건수와 무관하게 ~1ms (B-Tree Index Seek)
  → 데이터 볼륨 차이가 테스트에서 인덱스 필요성을 숨기는 이유
```

### JVM Heap 설정 차이의 영향

```
테스트 환경: -Xmx4g (개발 서버)
운영 환경:   -Xmx1g (비용 절감)

영향:
  테스트: GC 거의 발생 안 함 (힙이 충분)
  운영:   GC 빈번 (힙 부족) → Stop-The-World로 p99 급등

→ JVM 힙 설정은 테스트와 운영을 반드시 동일하게

테스트 환경: -XX:+UseSerialGC (기본)
운영 환경:   -XX:+UseG1GC (설정)

→ GC 알고리즘 차이로 응답시간 분포 완전히 달라짐
```

### 네트워크 레이턴시의 누적 효과

```
DB 쿼리 1회당 네트워크 비용:
  localhost:     0~0.1ms
  같은 AZ:       0.5~1ms
  다른 AZ:       1~3ms

N+1 쿼리 100회 실행 시:
  localhost:     0~10ms  ← 성능 문제 없어 보임
  다른 AZ (3ms): 300ms   ← 운영에서 명백한 병목

→ 로컬 테스트에서 N+1 쿼리가 발견되지 않는 이유
```

---

## 💻 실전 실험 — Docker Compose 실험 환경

### 완전한 실험 환경 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/perf_test
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
      SPRING_DATA_REDIS_HOST: redis
      JAVA_OPTS: >-
        -Xms512m -Xmx512m
        -XX:+UseG1GC
        -XX:MaxGCPauseMillis=200
        -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
        -XX:StartFlightRecording=filename=/logs/app.jfr,duration=300s,settings=profile
    volumes:
      - ./logs:/logs
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - perf-net

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: perf_test
    ports:
      - "3306:3306"
    volumes:
      - ./docker/init-data.sql:/docker-entrypoint-initdb.d/01-init.sql
      - ./docker/seed-data.sql:/docker-entrypoint-initdb.d/02-seed.sql
      - mysql-data:/var/lib/mysql
    command: >-
      --innodb-buffer-pool-size=512M
      --max-connections=200
      --slow-query-log=1
      --slow-query-log-file=/var/log/mysql/slow.log
      --long-query-time=0.1
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - perf-net

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    networks:
      - perf-net

  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: adminpass
      DOCKER_INFLUXDB_INIT_ORG: perf-test
      DOCKER_INFLUXDB_INIT_BUCKET: k6
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: my-super-secret-token
    volumes:
      - influxdb-data:/var/lib/influxdb2
    networks:
      - perf-net

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./docker/grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    depends_on:
      - influxdb
    networks:
      - perf-net

volumes:
  mysql-data:
  influxdb-data:
  grafana-data:

networks:
  perf-net:
    driver: bridge
```

### 대용량 테스트 데이터 생성 스크립트

```sql
-- docker/init-data.sql
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(100) NOT NULL,
  name VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  stock INT DEFAULT 0
);

CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  total_price DECIMAL(10, 2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  -- 의도적으로 인덱스 없음: 병목 재현용
);
```

```bash
#!/bin/bash
# docker/generate-seed-data.sh
# 운영 환경과 유사한 대용량 데이터 생성

echo "사용자 100,000명 생성..."
mysql -h 127.0.0.1 -uroot -proot perf_test <<'SQL'
INSERT INTO users (email, name)
SELECT
  CONCAT('user', seq, '@test.com'),
  CONCAT('User ', seq)
FROM (
  SELECT a.N + b.N * 1000 + 1 AS seq
  FROM
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
     UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9
     UNION SELECT 10 UNION SELECT 11 UNION SELECT 12 UNION SELECT 13 UNION SELECT 14
     UNION SELECT 15 UNION SELECT 16 UNION SELECT 17 UNION SELECT 18 UNION SELECT 19
     UNION SELECT 20 UNION SELECT 21 UNION SELECT 22 UNION SELECT 23 UNION SELECT 24
     UNION SELECT 25 UNION SELECT 26 UNION SELECT 27 UNION SELECT 28 UNION SELECT 29
     UNION SELECT 30 UNION SELECT 31 UNION SELECT 32 UNION SELECT 33 UNION SELECT 34
     UNION SELECT 35 UNION SELECT 36 UNION SELECT 37 UNION SELECT 38 UNION SELECT 39
     UNION SELECT 40 UNION SELECT 41 UNION SELECT 42 UNION SELECT 43 UNION SELECT 44
     UNION SELECT 45 UNION SELECT 46 UNION SELECT 47 UNION SELECT 48 UNION SELECT 49
     UNION SELECT 50 UNION SELECT 51 UNION SELECT 52 UNION SELECT 53 UNION SELECT 54
     UNION SELECT 55 UNION SELECT 56 UNION SELECT 57 UNION SELECT 58 UNION SELECT 59
     UNION SELECT 60 UNION SELECT 61 UNION SELECT 62 UNION SELECT 63 UNION SELECT 64
     UNION SELECT 65 UNION SELECT 66 UNION SELECT 67 UNION SELECT 68 UNION SELECT 69
     UNION SELECT 70 UNION SELECT 71 UNION SELECT 72 UNION SELECT 73 UNION SELECT 74
     UNION SELECT 75 UNION SELECT 76 UNION SELECT 77 UNION SELECT 78 UNION SELECT 79
     UNION SELECT 80 UNION SELECT 81 UNION SELECT 82 UNION SELECT 83 UNION SELECT 84
     UNION SELECT 85 UNION SELECT 86 UNION SELECT 87 UNION SELECT 88 UNION SELECT 89
     UNION SELECT 90 UNION SELECT 91 UNION SELECT 92 UNION SELECT 93 UNION SELECT 94
     UNION SELECT 95 UNION SELECT 96 UNION SELECT 97 UNION SELECT 98 UNION SELECT 99) a,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
     UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9
     UNION SELECT 10 UNION SELECT 11 UNION SELECT 12 UNION SELECT 13 UNION SELECT 14
     UNION SELECT 15 UNION SELECT 16 UNION SELECT 17 UNION SELECT 18 UNION SELECT 19
     UNION SELECT 20 UNION SELECT 21 UNION SELECT 22 UNION SELECT 23 UNION SELECT 24
     UNION SELECT 25 UNION SELECT 26 UNION SELECT 27 UNION SELECT 28 UNION SELECT 29
     UNION SELECT 30 UNION SELECT 31 UNION SELECT 32 UNION SELECT 33 UNION SELECT 34
     UNION SELECT 35 UNION SELECT 36 UNION SELECT 37 UNION SELECT 38 UNION SELECT 39
     UNION SELECT 40 UNION SELECT 41 UNION SELECT 42 UNION SELECT 43 UNION SELECT 44
     UNION SELECT 45 UNION SELECT 46 UNION SELECT 47 UNION SELECT 48 UNION SELECT 49
     UNION SELECT 50 UNION SELECT 51 UNION SELECT 52 UNION SELECT 53 UNION SELECT 54
     UNION SELECT 55 UNION SELECT 56 UNION SELECT 57 UNION SELECT 58 UNION SELECT 59
     UNION SELECT 60 UNION SELECT 61 UNION SELECT 62 UNION SELECT 63 UNION SELECT 64
     UNION SELECT 65 UNION SELECT 66 UNION SELECT 67 UNION SELECT 68 UNION SELECT 69
     UNION SELECT 70 UNION SELECT 71 UNION SELECT 72 UNION SELECT 73 UNION SELECT 74
     UNION SELECT 75 UNION SELECT 76 UNION SELECT 77 UNION SELECT 78 UNION SELECT 79
     UNION SELECT 80 UNION SELECT 81 UNION SELECT 82 UNION SELECT 83 UNION SELECT 84
     UNION SELECT 85 UNION SELECT 86 UNION SELECT 87 UNION SELECT 88 UNION SELECT 89
     UNION SELECT 90 UNION SELECT 91 UNION SELECT 92 UNION SELECT 93 UNION SELECT 94
     UNION SELECT 95 UNION SELECT 96 UNION SELECT 97 UNION SELECT 98 UNION SELECT 99) b
  WHERE a.N + b.N * 1000 < 100000
) AS nums;
SQL

echo "주문 데이터 1,000,000건 생성..."
mysql -h 127.0.0.1 -uroot -proot perf_test -e "
INSERT INTO orders (user_id, product_id, quantity, total_price)
SELECT
  FLOOR(RAND() * 100000) + 1,
  FLOOR(RAND() * 1000) + 1,
  FLOOR(RAND() * 5) + 1,
  ROUND(RAND() * 500000, 2)
FROM information_schema.columns a, information_schema.columns b
LIMIT 1000000;
"

echo "✅ 테스트 데이터 생성 완료"
echo "   - users: 100,000건"
echo "   - orders: 1,000,000건"
```

### 환경 시작 및 검증 스크립트

```bash
#!/bin/bash
# scripts/start-env.sh

echo "🚀 성능 테스트 환경 시작..."

# 환경 시작
docker compose up -d

# MySQL 준비 대기
echo "MySQL 준비 대기..."
until docker compose exec mysql mysqladmin ping -h localhost --silent; do
  sleep 2
done

# 데이터 건수 확인
echo ""
echo "=== 데이터 볼륨 확인 ==="
docker compose exec mysql mysql -uroot -proot perf_test -e "
  SELECT 'users' as table_name, COUNT(*) as count FROM users
  UNION ALL
  SELECT 'products', COUNT(*) FROM products
  UNION ALL
  SELECT 'orders', COUNT(*) FROM orders;
"

# 앱 서버 헬스체크
echo ""
echo "=== 앱 서버 헬스체크 ==="
until curl -sf http://localhost:8080/actuator/health > /dev/null; do
  echo "앱 서버 대기 중..."
  sleep 3
done
echo "✅ 앱 서버 준비 완료"

echo ""
echo "=== 접속 정보 ==="
echo "  앱 서버:  http://localhost:8080"
echo "  Grafana:  http://localhost:3000 (admin/admin)"
echo "  InfluxDB: http://localhost:8086"
echo "  MySQL:    localhost:3306 (root/root)"
```

---

## 📊 성능 비교 — 환경 차이에 따른 측정 결과

### 데이터 볼륨별 Full Table Scan 응답시간

| 데이터 건수 | 인덱스 없음 | 인덱스 있음 | 비고 |
|------------|-----------|-----------|------|
| 1,000건 | 2ms | 1ms | 차이 미미 → 인덱스 필요성 미발견 |
| 10만 건 | 18ms | 1ms | 18배 차이 |
| 100만 건 | 180ms | 1ms | 180배 차이 |
| 1,000만 건 | 1,800ms | 1ms | 1,800배 차이 |
| 5,000만 건 | 9,200ms | 1ms | **운영에서 장애** |

### JVM 힙 설정별 GC 영향

| 힙 설정 | GC 빈도 | Stop-The-World | p99 응답시간 |
|---------|--------|---------------|------------|
| -Xmx4g (테스트) | 매우 낮음 | < 50ms | 180ms |
| -Xmx1g (운영) | 높음 | 200~800ms | 1,200ms |
| -Xmx512m (과소) | 매우 높음 | > 1,000ms | 3,000ms+ |

---

## ⚖️ 트레이드오프

| 접근 | 장점 | 단점 |
|------|------|------|
| 로컬 환경 | 빠른 개발, 비용 없음 | 결과 신뢰 불가 |
| 전용 스테이징 환경 | 현실적 결과 | 인프라 비용 발생 |
| 운영 환경 직접 테스트 | 가장 정확 | 실 사용자 영향, 위험 |
| Prod-like Docker Compose | 비용/현실성 균형 | 네트워크 레이턴시 재현 한계 |

---

## 📌 핵심 정리

- **데이터 볼륨 차이**가 성능 결과를 가장 크게 왜곡한다 — 최소 운영 데이터의 10% 이상 사용
- **JVM 힙 설정**은 테스트와 운영을 반드시 동일하게 맞춰야 한다
- **네트워크 레이턴시**는 N+1 쿼리 같은 문제를 로컬에서 숨기는 원인
- **Docker Compose**로 일관된 환경을 팀 전체가 공유하면 "내 PC에서는 됐는데" 문제를 제거
- 로그 레벨도 운영과 동일하게 (DEBUG → INFO 전환만으로 10~15% 성능 차이 발생 가능)

---

## 🤔 생각해볼 문제

**Q1.** 로컬에서 p99 50ms였던 API가 운영에서 p99 3,000ms로 측정됐다. 원인으로 가능한 것들은 무엇인가?

<details>
<summary>해설 보기</summary>

①데이터 볼륨 차이(로컬 1,000건 vs 운영 5,000만 건) + 인덱스 부재로 Full Table Scan 발생. ②JVM 힙 설정 차이로 GC 빈도 증가. ③네트워크 레이턴시(로컬 0ms vs 운영 다른 AZ 2ms) × N+1 쿼리 100회 = 200ms 추가. ④로그 레벨 DEBUG(로컬)와 INFO(운영) 차이. ⑤Connection Pool 크기 차이 — 로컬에서 높게 설정해 두면 병목이 숨겨짐. 이 중 하나가 아닌 여러 요소의 복합 작용인 경우가 많다.

</details>

**Q2.** 운영 데이터를 테스트 환경에 복사할 때 고려해야 할 점은?

<details>
<summary>해설 보기</summary>

①개인정보 마스킹 필수(이메일, 전화번호, 결제 정보 등). ②데이터 분포가 운영과 동일한지 확인(특정 사용자에 쏠린 주문 데이터 등). ③인덱스 통계 재수집(`ANALYZE TABLE`). ④데이터 크기가 너무 크면 샘플링 — 하지만 비율을 일정하게 유지. ⑤외래키 참조 무결성 유지.

</details>

**Q3.** 외부 API(PG사 결제, 문자 발송 등)에 의존하는 API를 성능 테스트할 때 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

WireMock을 사용해 외부 API를 Mock 서버로 대체하는 것이 가장 일반적이다. 응답 지연을 설정할 수 있어 외부 API가 느릴 때의 영향도 시뮬레이션 가능하다. 주의: Mock 서버 응답시간을 실제 외부 API 평균 응답시간(예: PG사 100ms)으로 설정해야 현실적인 결과를 얻을 수 있다. Testcontainers로 WireMock 컨테이너를 관리하면 격리된 환경에서 일관성 있게 실행 가능하다.

</details>

---

<div align="center">

**[⬅️ 이전: 현실적 부하 시나리오 설계 — Access Log 기반](./03-realistic-load-scenario.md)** | **[홈으로 🏠](../README.md)** | **[다음: 성능 기준선(Baseline) 수립 — 회귀 감지 자동화 ➡️](./05-baseline-and-regression.md)**

</div>
