# 03. Kafka 성능 테스트 — Producer/Consumer 처리량 측정

---

## 🎯 핵심 질문

- Kafka의 실제 처리량(Throughput)은 어느 정도 수준일까?
- Producer와 Consumer의 병목은 어디인가?
- Consumer Lag이 증가한다는 것은 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka는 메시지 큐로서, **높은 처리량**이 가장 중요한 성능 지표입니다:

```
Producer (메시지 발행)
  ↓
Kafka Broker (메시지 저장)
  ↓
Consumer (메시지 소비)
```

문제는:

1. **Producer 병목**: Broker가 메시지를 받을 수 있는데 Producer가 느리게 보내면?
   → Batch 크기, 압축, ACK 설정으로 최적화 필요

2. **Consumer Lag**: Producer가 보내는 속도 > Consumer가 처리하는 속도
   → Partition 수 증가, Consumer 병렬 처리 필요

3. **대역폭 포화**: 대량의 큰 메시지
   → 압축, 메시지 크기 최적화 필요

**따라서 실제 환경에 맞는 처리량 측정이 필수입니다.**

---

## 😱 흔한 실수 (Before)

### 패턴 1: 기본 설정으로 성능 측정

```bash
# ❌ 문제: 기본 Kafka 설정으로 처리량 측정
# Producer: acks=1 (기본)
# Consumer: 수동 offset 관리

# 단일 파티션 토픽 생성
kafka-topics.sh --create \
  --topic test \
  --partitions 1 \
  --replication-factor 1

# 기본 설정으로 producer perf test 실행
kafka-producer-perf-test.sh \
  --topic test \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# 결과: ~50,000 msg/s (낮음!)
```

**문제점:**
- `acks=1`: Leader만 확인 (안전하지만 느림)
- Batch 미사용: 메시지를 하나씩 전송
- 압축 미사용: 네트워크 대역폭 낭비
- 파티션 1개: 병렬 처리 불가능

### 패턴 2: Consumer 성능 측정 누락

```bash
# ❌ 문제: Producer만 측정하고 Consumer 성능 무시
# Consumer가 처리할 수 없으면 Lag이 무한 증가

kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test \
  --from-beginning

# 하나씩 메시지 처리 → 매우 느림
# Lag 계속 증가
```

**문제점:**
- Consumer 성능 미측정
- Lag 증가 여부 모니터링 없음
- 병렬 처리 설정 없음

---

## ✨ 올바른 접근 (After)

### 1단계: Kafka 설정 최적화

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      
      # 성능 최적화 설정
      KAFKA_LOG_SEGMENT_BYTES: 1073741824  # 1GB (기본 1MB)
      KAFKA_LOG_RETENTION_HOURS: 24
      KAFKA_NUM_NETWORK_THREADS: 8        # 기본 3
      KAFKA_NUM_IO_THREADS: 8             # 기본 8
      KAFKA_SOCKET_SEND_BUFFER_BYTES: 102400
      KAFKA_SOCKET_RECEIVE_BUFFER_BYTES: 102400
      KAFKA_SOCKET_REQUEST_MAX_BYTES: 104857600  # 100MB

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  jmx-exporter:
    image: sscaling/jmx-exporter:latest
    ports:
      - "5556:5556"
```

실행:
```bash
docker-compose up -d
```

### 2단계: Producer 성능 테스트

#### 테스트 토픽 생성

```bash
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --partitions 10 \
  --replication-factor 1 \
  --config compression.type=snappy \
  --config min.insync.replicas=1
```

#### 기본 설정 성능 측정 (acks=1)

```bash
# 100만 메시지, 1KB 크기, 최대 처리량
kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:9092 \
    acks=1 \
    compression.type=none \
    batch.size=16384 \
    linger.ms=0
```

#### 최적화된 설정 성능 측정 (acks=0, batch, 압축)

```bash
kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:9092 \
    acks=0 \
    compression.type=snappy \
    batch.size=32768 \
    linger.ms=10 \
    buffer.memory=67108864
```

#### 매개변수 설명

- `acks=0`: Broker 응답 대기 없음 (최빠름, 손실 위험)
- `acks=1`: Leader만 확인 (안전성 적절)
- `compression.type=snappy`: 네트워크 압축 (CPU 트레이드오프)
- `batch.size=32768`: 배치 크기 (클수록 처리량 증가)
- `linger.ms=10`: 배치 모아두기 시간 (지연 증가 가능)

### 3단계: Consumer 성능 테스트

```bash
# Consumer 그룹 생성 및 메시지 소비 성능 측정
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 1000000 \
  --threads 1

# 결과:
# start.time, end.time, data.consumed.in.MB, MB.sec, nMsg.sec
```

#### 여러 Consumer 인스턴스 (병렬 처리)

```bash
# 터미널 1
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 500000 \
  --threads 4 \
  --group perf-group

# 터미널 2 (동시 실행)
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 500000 \
  --threads 4 \
  --group perf-group

# 2개 Consumer가 동시에 처리 → 처리량 ~2배 증가
```

### 4단계: Consumer Lag 모니터링

```bash
# Lag 상태 조회
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group perf-group \
  --describe

# 출력:
# TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# perf-test   0          100000          500000          400000
# perf-test   1          150000          500000          350000
# ...
# 총 LAG: 350만 메시지 (Consumer가 아직 처리하지 못한 메시지)
```

#### Lag 실시간 모니터링 (Shell Script)

```bash
#!/bin/bash
# monitor-lag.sh

while true; do
  clear
  echo "=== Consumer Lag Monitor ==="
  echo "Timestamp: $(date)"
  echo
  
  kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --group perf-group \
    --describe \
    --members \
    --verbose
  
  sleep 5
done
```

실행:
```bash
chmod +x monitor-lag.sh
./monitor-lag.sh
```

### 5단계: 배압(Backpressure) 탐지 및 처리

```java
// KafkaProducerBackpressureTest.java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.concurrent.Future;

public class KafkaProducerBackpressureTest {
    
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        
        // 배압 탐지를 위한 설정
        props.put("acks", "1");
        props.put("batch.size", 32768);
        props.put("linger.ms", 10);
        props.put("buffer.memory", 67108864);  // 64MB
        
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        
        int totalMessages = 1000000;
        int successCount = 0;
        int backpressureCount = 0;
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < totalMessages; i++) {
            String key = "key-" + (i % 100);
            String value = "message-" + i + "-" + System.nanoTime();
            
            ProducerRecord<String, String> record = 
                new ProducerRecord<>("perf-test", key, value);
            
            try {
                // 비동기 전송
                Future<RecordMetadata> future = producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        System.err.println("Send failed: " + exception.getMessage());
                    }
                });
                
                // ⚠️ 배압 탐지: Future 대기 시간이 길어지면 배압 발생
                RecordMetadata metadata = future.get();
                successCount++;
                
            } catch (Exception e) {
                // 배치 버퍼 부족 → 배압 발생
                backpressureCount++;
                System.out.println("[BACKPRESSURE] Buffer full at message " + i);
                Thread.sleep(100); // 배압 시간 초과 대기
            }
            
            if ((i + 1) % 100000 == 0) {
                long elapsed = System.currentTimeMillis() - startTime;
                double msgPerSec = (i + 1) * 1000.0 / elapsed;
                System.out.printf("[Progress] %d / %d (%.0f msg/s)%n", 
                    i + 1, totalMessages, msgPerSec);
            }
        }
        
        long totalTime = System.currentTimeMillis() - startTime;
        double throughput = totalMessages * 1000.0 / totalTime;
        
        System.out.println("\n=== Results ===");
        System.out.printf("Total Messages: %d%n", totalMessages);
        System.out.printf("Successful: %d%n", successCount);
        System.out.printf("Backpressure Events: %d (%.1f%%)%n", 
            backpressureCount, 100.0 * backpressureCount / totalMessages);
        System.out.printf("Throughput: %.0f msg/s%n", throughput);
        System.out.printf("Total Time: %.1f seconds%n", totalTime / 1000.0);
        
        producer.close();
    }
}
```

컴파일 및 실행:
```bash
javac -cp ~/kafka/libs/* KafkaProducerBackpressureTest.java
java -cp ~/kafka/libs/*:. KafkaProducerBackpressureTest

# 출력:
# [Progress] 100000 / 1000000 (250000.0 msg/s)
# [BACKPRESSURE] Buffer full at message 450000
# [Progress] 200000 / 1000000 (200000.0 msg/s)
# ...
# === Results ===
# Backpressure Events: 5 (0.0005%)
# Throughput: 180000 msg/s
```

### 6단계: k6 + xk6-kafka로 Kafka 부하 테스트

#### xk6-kafka 플러그인 설치

```bash
# Go 1.17 이상 필요
go install github.com/grafana/xk6-kafka/cmd/xk6-kafka@latest

# 또는 Docker로 미리 빌드된 이미지 사용
docker pull grafana/k6:latest-with-xk6-kafka
```

#### k6 성능 테스트 스크립트

```javascript
// kafka-load-test.js
import { kafka } from 'k6/x/kafka';
import { check, sleep, group } from 'k6';

const brokers = ['kafka:29092'];
const topic = 'perf-test';

export const options = {
  vus: 10,
  duration: '2m',
  thresholds: {
    'kafka_message_send_duration': ['p(95)<500'],
    'kafka_message_receive_duration': ['p(95)<1000'],
  },
};

// Producer 테스트
export function testProducer() {
  const producer = new kafka.Producer({
    brokers: brokers,
  });
  
  for (let i = 0; i < 100; i++) {
    const message = {
      key: `key-${i}`,
      value: `message-${i}-${__VU}-${__ITER}`,
      partition: i % 10,  // Round-robin across 10 partitions
    };
    
    const startTime = new Date();
    producer.produce({
      topic: topic,
      messages: [message],
    });
    const duration = new Date() - startTime;
    
    check(duration, {
      'produce latency < 500ms': (d) => d < 500,
      'produce latency < 1000ms': (d) => d < 1000,
    });
  }
}

// Consumer 테스트
export function testConsumer() {
  const reader = kafka.newReader({
    brokers: brokers,
    topic: topic,
    groupID: 'k6-consumer-group',
    startOffset: -1,  // 최신부터 시작
  });
  
  for (let i = 0; i < 100; i++) {
    const startTime = new Date();
    const message = reader.consume();
    const duration = new Date() - startTime;
    
    check(message, {
      'message received': (msg) => msg !== null,
    });
    
    check(duration, {
      'consume latency < 1000ms': (d) => d < 1000,
      'consume latency < 2000ms': (d) => d < 2000,
    });
  }
  
  reader.close();
}

export default function () {
  group('Kafka Producer', testProducer);
  sleep(1);
  group('Kafka Consumer', testConsumer);
}
```

실행:
```bash
docker run --rm \
  --network kafka-network \
  -v $(pwd):/scripts \
  grafana/k6:latest-with-xk6-kafka \
  run /scripts/kafka-load-test.js
```

---

## 🔬 내부 동작 원리

### Producer의 배치 처리 메커니즘

```
메시지 1 (0ms)   → Batch Buffer
메시지 2 (2ms)   → Batch Buffer  [크기 < 32KB, 시간 < 10ms 대기]
메시지 3 (5ms)   → Batch Buffer
메시지 4 (10ms)  → Broker 전송 [linger.ms=10 초과]
                  (200ms 왕복)
메시지 5 (210ms) → Batch Buffer [새로운 배치]
...

총 시간: 4개 메시지 × (10ms + 200ms) / 4 = 52.5ms per message
         단일 전송: 메시지 1개 × 200ms = 200ms per message
```

**배치의 효과:**
- 배치 크기: 32KB, 메시지 1KB = 32개 메시지 묶음
- 처리량 향상: 32배 증가 가능

### Consumer Lag의 의미

```
[Timeline]
        0s          10s         20s         30s
Broker: |-----------|-----------|-----------|
        0메시지    100메시지   200메시지   300메시지

Producer: 10 msg/s 속도로 발행
Consumer: 5 msg/s 속도로 처리

시간에 따른 Lag:
0s:   Lag = 0 (같은 속도)
10s:  Lag = 50 (Producer 100, Consumer 50)
20s:  Lag = 150 (Producer 200, Consumer 50)
30s:  Lag = 300 (Producer 300, Consumer 0) ← 영원히 따라잡지 못함!
```

---

## 💻 실전 실험

### 실험: 파티션 수 변경 효과

#### 설정 1: 파티션 1개 (❌ 느림)

```bash
kafka-topics.sh --create \
  --topic single-partition \
  --partitions 1 \
  --replication-factor 1

kafka-producer-perf-test.sh \
  --topic single-partition \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092
```

결과: **~100,000 msg/s** (단일 스레드만 사용)

#### 설정 2: 파티션 10개 (✅ 빠름)

```bash
kafka-topics.sh --create \
  --topic multi-partition \
  --partitions 10 \
  --replication-factor 1

kafka-producer-perf-test.sh \
  --topic multi-partition \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092
```

결과: **~650,000 msg/s** (10개 파티션 병렬 처리)

---

## 📊 성능 비교

| 메트릭 | 기본 설정 (acks=1) | 최적화 (acks=0 + batch + snappy) | 개선도 |
|--------|-----------------|--------------------------------|--------|
| **처리량** | 50,000 msg/s | 300,000 msg/s | +500% |
| **P95 지연** | 20ms | 5ms | -75% |
| **P99 지연** | 50ms | 15ms | -70% |
| **배압 이벤트** | 450회 | 5회 | -99% |
| **네트워크 사용** | 1Gbps | 200Mbps (압축) | -80% |
| **Consumer Lag (1분 후)** | 300만 | 0 | 완벽 |

---

## ⚖️ 트레이드오프

### 처리량 최대화 (acks=0, 압축)
✅ 최고 처리량 (300k+ msg/s)  
✅ 최소 네트워크 대역폭  
❌ 메시지 손실 가능 (브로커 장애 시)  
❌ CPU 사용 증가 (압축)  

### 안정성 우선 (acks=all, 압축 제한)
✅ 메시지 손실 방지  
✅ 데이터 무결성 보증  
❌ 처리량 감소 (30k msg/s)  
❌ 지연 증가 (200-500ms)  

### 균형 (acks=1, batch + 압축)
✅ 적절한 처리량 (200k msg/s)  
✅ 리더 노드 보호  
⚠️ 리더 노드 장애 시 메시지 손실 가능 (희귀)  

---

## 📌 핵심 정리

1. **Kafka 처리량**: 기본 설정과 최적화 설정 간 5-10배 차이

2. **파티션의 중요성**: 파티션당 1개 스레드 처리 → 파티션 수 = 병렬도

3. **배치 처리**: `batch.size`와 `linger.ms` 조정으로 처리량 극대화

4. **Consumer Lag**: Producer 속도 > Consumer 속도 → Lag 증가 → Consumer 확장 필수

5. **배압 탐지**: `buffer.memory` 부족 시 발생 → 모니터링 필수

6. **k6 + xk6-kafka**: 클라이언트 성능 테스트로 실제 애플리케이션 수준의 부하 시뮬레이션

---

## 🤔 생각해볼 문제

**Q1. acks=0과 acks=1의 처리량 차이가 크지 않다면, acks=all은 얼마나 느릴까?**

<details>
<summary>해설 보기</summary>

**일반적인 처리량:**

```
acks=0 (응답 대기 없음):       300,000 msg/s
acks=1 (Leader 확인):          200,000 msg/s (응답 대기 ~10ms)
acks=all (ISR 모두 확인):       20,000 msg/s (응답 대기 ~100-200ms)
```

**acks=all이 느린 이유:**

```
acks=0:  Producer → Broker (끝)
         총 시간: 네트워크 왕복 0회 (비동기)

acks=1:  Producer → Leader Broker → 응답
         총 시간: 네트워크 왕복 1회 (~10ms)

acks=all: Producer → Leader Broker → Replica 1 → Replica 2 → ... → 응답
          총 시간: 동기화 대기 (~100-200ms)
```

**실제 숫자:**

3개 Replica, 1개 Leader 가정:
- Leader: 즉시 응답 (1ms)
- Replica 1: 네트워크 지연 (5ms) + 디스크 쓰기 (10ms) = 15ms
- Replica 2: 네트워크 지연 (5ms) + 디스크 쓰기 (10ms) = 15ms
- 최악의 경우: max(1, 15, 15) = 15ms per message

배치 10개 (linger.ms=10):
- acks=0: 10ms / 10 = 1ms per message → 1,000,000 msg/s
- acks=1: (10ms + 10ms) / 10 = 2ms per message → 500,000 msg/s
- acks=all: (10ms + 200ms) / 10 = 21ms per message → 47,000 msg/s

**정리:** acks=all은 acks=1 대비 약 10배 느림

</details>

---

**Q2. Consumer Lag이 증가한다는 것은 Consumer가 느린 건지, Producer가 빠른 건지 어떻게 판단할까?**

<details>
<summary>해설 보기</summary>

**구분 방법:**

```bash
# 현재 상태 조회
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --describe

# 출력:
# TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# events   0          1000000         2000000         1000000 ← Lag 증가 중
# events   1          1000000         1500000         500000
```

**분석 방법 1: Lag 증가율 추적**

```bash
#!/bin/bash
# check-lag-trend.sh

LAG_BEFORE=$(kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --describe | tail -1 | awk '{print $NF}')

sleep 10

LAG_AFTER=$(kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --describe | tail -1 | awk '{print $NF}')

LAG_INCREASE=$((LAG_AFTER - LAG_BEFORE))

if [ $LAG_INCREASE -gt 0 ]; then
    echo "Lag 증가 중! (10초에 +$LAG_INCREASE 메시지)"
    echo "→ Consumer가 Producer를 따라잡지 못함 (Consumer 느림)"
else
    echo "Lag 감소 중. (10초에 $LAG_INCREASE 메시지)"
    echo "→ Consumer가 따라잡는 중 (Producer가 더 빨랐음)"
fi
```

**분석 방법 2: Producer와 Consumer 처리량 비교**

```
최근 1분 동시 처리:

Producer RPS (metrics에서):
- 초당 50,000 메시지 발행

Consumer RPS (Consumer Lag 감소율):
- 초당 10,000 메시지 처리

→ 50,000 > 10,000 → Producer가 빠름!
→ Lag이 계속 증가
```

**분석 방법 3: Consumer Lag 그래프**

```
Prometheus/Grafana에서:

증가 중 ↗️     : Consumer 느림 (확장 필요)
감소 중 ↘️     : Producer 빠름 (과거 데이터 처리 중)
평탄 ─         : 균형 (정상)
급증 ⬆️        : Consumer 장애 또는 멈춤
```

**권장 조치:**

```
Lag > 100만 메시지?
├─ Consumer 확장 (Consumer 인스턴스 추가)
└─ 또는 파티션 수 증가 (재배치)

Lag 증가 속도 > 1% per minute?
└─ 긴급 대응 필요 (Consumer 성능 개선)
```

</details>

---

**Q3. k6-kafka로 5개 VU가 각각 100개 메시지를 1초 안에 전송하면 처리량은 500 msg/s인가?**

<details>
<summary>해설 보기</summary>

**단순 계산: 500 msg/s가 맞다**

```javascript
// 이상적 시나리오
export const options = {
  vus: 5,
  duration: '1m',
};

export default function () {
  for (let i = 0; i < 100; i++) {
    producer.produce({
      topic: 'test',
      messages: [{key: '', value: '...'}],
    });
  }
  // 5 VU × 100 msg = 500 msg (1초 내)
  // Throughput = 500 msg/s
}
```

**그러나 실제로는?**

```
1초 내에 500 메시지를 보낼 수 있지만:

1. Kafka Broker 처리 능력 확인
   - 단일 Broker: 최대 300k msg/s 이상 가능
   - 500 msg/s는 매우 낮음 ✓

2. 네트워크 지연
   - 메시지 직렬화: ~0.1ms per message
   - 네트워크 왕복: ~10ms
   - 5 VU × (0.1 + 10)ms = 50ms per VU
   - 실제 처리: 5 VU 병렬 = ~100 msg/s (배치 미사용 시)

3. 배치 미사용 시
   - 100 메시지 → 100번의 send 호출
   - 각 호출마다 지연 (네트워크)
   - 처리량 감소 가능

4. 동시성 한계
   - VU당 1개 스레드만 사용하면 순차 처리
   - 진정한 병렬은 아님
```

**개선 방안:**

```javascript
// 배치 전송
export default function () {
  const messages = [];
  for (let i = 0; i < 100; i++) {
    messages.push({
      key: 'key-' + i,
      value: 'msg-' + i,
    });
  }
  
  // 배치로 한 번에 전송
  producer.produce({
    topic: 'test',
    messages: messages,
  });
  
  // 처리량 증가 가능
}
```

**결론:** 단순 계산은 500 msg/s이지만, 실제는 배치, 네트워크 지연, VU 수 등에 따라 100-500 msg/s 사이

</details>

---

<div align="center">

**[⬅️ 이전: 마이크로서비스 성능 — 서비스 간 호출 체인 병목 특정](./02-microservice-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: 컨테이너 환경 성능 — JVM의 컨테이너 메모리 인식 문제 ➡️](./04-container-performance.md)**

</div>
