# 04. 컨테이너 환경 성능 — JVM의 컨테이너 메모리 인식 문제

---

## 🎯 핵심 질문

- 컨테이너에서 설정한 메모리 제한을 JVM이 왜 인식하지 못할까?
- JVM이 인식한 메모리와 실제 컨테이너 제한이 다르면 어떤 일이?
- CPU 제한은 어떻게 JVM의 성능에 영향을 미칠까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kubernetes에서 JVM 애플리케이션을 실행할 때:

```yaml
# 컨테이너 리소스 제한 설정
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

문제는:

```
▼ Java 8u191 이전
├─ /proc/meminfo 읽음: 전체 호스트 메모리 (64GB) 인식
├─ Heap 자동 설정: 64GB × 25% = 16GB 할당 시도
└─ 결과: OOMKilled (실제 제한은 1GB)

▼ Java 8u191 이후 (UseContainerSupport)
├─ cgroup 읽음: 실제 컨테이너 제한 (1GB) 인식
├─ Heap 자동 설정: 1GB × 75% = 750MB
└─ 결과: 정상 동작 ✓
```

**따라서 JVM 옵션 설정이 성능을 크게 좌우합니다.**

---

## 😱 흔한 실수 (Before)

### 패턴 1: 레거시 JVM 버전 (Java 8u191 이전)

```dockerfile
# ❌ 문제: Java 8 (이전 버전)
FROM openjdk:8

ENV JAVA_OPTS="-Xmx512m"

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**실행 환경:**
```yaml
# Kubernetes pod
resources:
  limits:
    memory: "1Gi"  # 컨테이너 제한: 1GB
```

**발생 문제:**
```
JVM 메모리 확인:
$ jmap -heap $(pidof java)
Max Heap Size       : 512m ← 정상

그런데 포드는 OOMKilled...

원인 분석:
- JVM Heap: 512MB
- Non-heap (Metaspace, Code Cache 등): ~200MB
- Native Memory: ~300MB
- 총합: ~1GB 초과 → OOMKilled!
```

### 패턴 2: 고정 Heap 크기만 설정

```dockerfile
# ❌ 문제: Heap만 설정, 스택, Metaspace 미고려
FROM openjdk:11

ENV JAVA_OPTS="-Xmx1g -Xms1g"

ENTRYPOINT ["java", "$JAVA_OPTS", "-jar", "app.jar"]
```

**문제점:**
- Heap 1GB + Non-heap (300MB) + 스택 (수십 MB) = 1.3GB+
- 컨테이너 제한 1GB와 충돌

### 패턴 3: CPU 제한 무시

```yaml
# ❌ 문제: CPU 제한 없음
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    # CPU 제한 미설정
    # → 다른 pod이 없으면 전체 CPU 사용
    # → 다른 pod이 있으면 스케줄러가 CPU 나눔
    # → 성능 변동성 큼
```

---

## ✨ 올바른 접근 (After)

### 1단계: 현대식 JVM 버전 선택

```dockerfile
# ✅ Java 11 LTS (충분함)
FROM openjdk:11-jre-slim

# 또는
# FROM eclipse-temurin:11-jre-focal (권장)

# 또는
# FROM openjdk:21-oracle (최신)
```

### 2단계: 컨테이너 인식 JVM 옵션 설정

```dockerfile
# Dockerfile
FROM eclipse-temurin:11-jre-focal

WORKDIR /app

COPY target/app.jar .

ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof"

ENTRYPOINT ["java", "-jar", "$JAVA_OPTS", "app.jar"]
```

#### 옵션 설명

| 옵션 | 의미 | 기본값 |
|------|------|--------|
| `UseContainerSupport` | cgroup에서 메모리 읽기 | Java 9+ 자동 활성화 |
| `MaxRAMPercentage=75.0` | 메모리 제한의 75%를 Heap으로 | 25% (서버용) |
| `InitialRAMPercentage=50.0` | 초기 Heap 크기 (메모리 제한의 50%) | MaxRAMPercentage와 동일 |
| `UseG1GC` | G1 Garbage Collector (최신, 안정) | Java 9+ 기본 |
| `MaxGCPauseMillis=200` | GC 최대 정지 시간 (ms) | 200 (목표값) |
| `HeapDumpOnOutOfMemoryError` | OOM 시 Heap dump 생성 | disabled |

### 3단계: Kubernetes 리소스 설정

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        
        # 메모리 설정
        resources:
          requests:
            memory: "512Mi"  # 초기 할당량
            cpu: "250m"      # 초기 CPU
          limits:
            memory: "1Gi"    # 최대 메모리 (OOM 기준)
            cpu: "1000m"     # CPU 스로틀링 기준
        
        # 헬스 체크
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

      # HPA 트리거 설정
      # (나중 섹션)
```

### 4단계: JVM 메모리 구조 파악

```
컨테이너 메모리 제한: 1GB
│
├─ JVM Heap (MaxRAMPercentage=75%)
│  └─ 750MB (Young + Old)
│
├─ Non-Heap
│  ├─ Metaspace: ~100MB
│  ├─ Code Cache: ~50MB
│  └─ Other: ~50MB
│
├─ 스택
│  └─ 약 1MB per thread
│
└─ Direct Buffer: 설정에 따라 ~50-100MB

합계: ~1GB (안전한 범위)
```

계산:
```
MaxRAMPercentage = 75% (권장)
메모리 제한 = 1GB

Heap = 1GB × 0.75 = 750MB
Non-Heap = ~200MB
여유 = 1GB - 750MB - 200MB = 50MB
```

### 5단계: 실제 메모리 확인

```bash
# Pod 내부에서 실행
kubectl exec <pod-name> -- jmap -heap $(pidof java)

# 출력 예시
# Heap Configuration:
#    MinHeapFreeRatio         = 40
#    MaxHeapFreeRatio         = 70
#    MaxHeapSize              = 805306368 (768.0MB)
#    NewSize                  = 268435456 (256.0MB)
#    MaxNewSize               = 268435456 (256.0MB)
#    ...
```

### 6단계: CPU 제한과 성능

```yaml
# CPU 제한 설정
resources:
  requests:
    cpu: "500m"  # 예약 CPU (스케줄링 기준)
  limits:
    cpu: "1000m" # 스로틀링 기준
```

#### CPU 스로틀링 메커니즘 (CFS - Completely Fair Scheduler)

```
CFS는 CPU 할당을 "기간(period)" 단위로 관리:

Period = 100ms (기본)
Quota = limits.cpu / 1000 × period = 1000m / 1000 × 100ms = 100ms

→ 100ms 마다 100ms 동안의 CPU 사용 권리

실행 예시:
[시간대]     [CPU 사용]                           [상태]
0-50ms:      50ms 사용 ▮▮▮▮▮ (정상)
50-100ms:    50ms 사용 ▮▮▮▮▮ (정상, 합계 100ms)
100-150ms:   CPU 스로틀! (100ms 초과 시도) ███
150-200ms:   다음 period까지 대기
200-300ms:   새로운 기간 시작, 다시 100ms 사용 가능
```

---

## 🔬 내부 동작 원리

### Java 8u191 vs 이후의 메모리 감지

#### Before (Java 8u191 이전)

```java
// /proc/meminfo 읽기
public long getMaxHeap() {
    return Runtime.getRuntime().maxMemory();
    // → /proc/meminfo의 MemTotal 읽음
    // → 호스트 전체 메모리 반환
}

// 예: 호스트 64GB
// Heap 자동 설정: 64GB × 25% = 16GB
// 컨테이너 제한: 1GB
// 결과: 1GB 초과 → OOMKilled
```

#### After (Java 8u191+, -XX:+UseContainerSupport)

```java
// cgroup에서 읽기
public long getMaxHeap() {
    // /sys/fs/cgroup/memory/memory.limit_in_bytes 읽음
    long cgroupLimit = readCgroupMemoryLimit();
    
    // 컨테이너 제한: 1GB
    // Heap 설정: 1GB × 75% = 750MB
    return cgroupLimit * 0.75;
}
```

### GC 동작

```
G1GC의 목표: 최대 200ms 정지 시간

Heap: 750MB (32개 영역, 각 약 23.4MB)

GC 실행:
┌─────────────────────────┐
│ Young GC (Minor GC)     │
├─────────────────────────┤
│ Old 영역 수집 비율:     │
│ (Young 가득 참 비율) /  │
│ MaxGCPauseMillis로 조정 │
└─────────────────────────┘

목표: 각 GC 정지 < 200ms
(MaxGCPauseMillis 미충족 시 사용자 요청 지연)
```

---

## 💻 실전 실험

### 실험: 메모리 제한 변경에 따른 성능 비교

#### 테스트 애플리케이션 (Spring Boot)

```java
// MemoryLoadApp.java
@SpringBootApplication
@RestController
public class MemoryLoadApp {
    
    @GetMapping("/")
    public Map<String, Object> index() {
        Runtime rt = Runtime.getRuntime();
        return Map.of(
            "total_memory", rt.totalMemory(),
            "max_memory", rt.maxMemory(),
            "free_memory", rt.freeMemory(),
            "heap_used", rt.totalMemory() - rt.freeMemory()
        );
    }
    
    @GetMapping("/allocate")
    public String allocate(
            @RequestParam(defaultValue = "100") int sizeMB,
            @RequestParam(defaultValue = "1000") int iterations) {
        List<byte[]> list = new ArrayList<>();
        
        long startTime = System.currentTimeMillis();
        
        try {
            for (int i = 0; i < iterations; i++) {
                byte[] arr = new byte[sizeMB * 1024 * 1024];
                list.add(arr);
                
                if (i % 100 == 0) {
                    System.gc();
                }
            }
            
            long duration = System.currentTimeMillis() - startTime;
            return "Allocated: " + (sizeMB * iterations) + "MB in " + duration + "ms";
            
        } catch (OutOfMemoryError e) {
            long duration = System.currentTimeMillis() - startTime;
            return "OOM after " + (list.size() * sizeMB) + "MB, " + duration + "ms";
        }
    }
    
    public static void main(String[] args) {
        SpringApplication.run(MemoryLoadApp.class, args);
    }
}
```

Dockerfile:
```dockerfile
FROM eclipse-temurin:11-jre-focal

WORKDIR /app
COPY target/memory-load-app.jar .

ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200"

ENTRYPOINT ["java", "$JAVA_OPTS", "-jar", "memory-load-app.jar"]
```

#### 시나리오 1: 메모리 제한 1GB

```yaml
# deployment-1gb.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-1gb
spec:
  containers:
  - name: app
    image: memory-load-app:latest
    resources:
      limits:
        memory: "1Gi"
```

실행:
```bash
kubectl apply -f deployment-1gb.yaml

# 부하 테스트
kubectl exec app-1gb -- curl "http://localhost:8080/allocate?sizeMB=100&iterations=10"
# 응답: "Allocated: 1000MB in 5000ms"

# 메모리 확인
kubectl exec app-1gb -- curl "http://localhost:8080/"
# {
#   "max_memory": 750000000,     // 750MB (1GB × 75%)
#   "heap_used": 950000000,      // 950MB (거의 포화)
# }
```

#### 시나리오 2: 메모리 제한 2GB

```yaml
# deployment-2gb.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-2gb
spec:
  containers:
  - name: app
    image: memory-load-app:latest
    resources:
      limits:
        memory: "2Gi"
```

실행:
```bash
kubectl apply -f deployment-2gb.yaml

# 부하 테스트
kubectl exec app-2gb -- curl "http://localhost:8080/allocate?sizeMB=100&iterations=20"
# 응답: "Allocated: 2000MB in 4500ms"

# 메모리 확인
kubectl exec app-2gb -- curl "http://localhost:8080/"
# {
#   "max_memory": 1500000000,    // 1.5GB (2GB × 75%)
#   "heap_used": 1600000000,     // 1.6GB (안정적)
# }
```

#### 시나리오 3: CPU 제한 변경

```bash
# CPU 제한 500m (느림)
kubectl set resources pod app-1gb \
  --limits=cpu=500m

# k6 부하 테스트
k6 run -u 10 -d 1m test.js
# p95: 1200ms (느림)
# throughput: 8 req/s

---

# CPU 제한 2000m (빠름)
kubectl set resources pod app-1gb \
  --limits=cpu=2000m

# k6 부하 테스트
k6 run -u 10 -d 1m test.js
# p95: 300ms (빠름)
# throughput: 32 req/s
```

---

## 📊 성능 비교

### 메모리 제한 효과

| 메트릭 | 1GB 제한 | 2GB 제한 | 4GB 제한 |
|--------|----------|----------|----------|
| **Max Heap** | 750MB | 1.5GB | 3GB |
| **할당 가능 메모리** | 1000MB | 2000MB | 4000MB |
| **할당 시간** | 5000ms | 4500ms | 4000ms |
| **GC 횟수** | 25회 | 8회 | 2회 |
| **GC 정지 시간** | ~200ms × 25 = 5s | ~150ms × 8 = 1.2s | ~100ms × 2 = 0.2s |
| **p95 응답시간** | 800ms | 200ms | 50ms |
| **OOM 위험** | 높음 | 중간 | 낮음 |

### CPU 제한 효과

| 메트릭 | 500m | 1000m | 2000m |
|--------|-------|--------|---------|
| **요청/초 (throughput)** | 8 | 25 | 50 |
| **p50 응답시간** | 500ms | 150ms | 50ms |
| **p95 응답시간** | 1200ms | 300ms | 100ms |
| **p99 응답시간** | 2000ms | 600ms | 200ms |
| **CPU 사용률** | 500m (100% 스로틀) | ~800m (효율적) | ~1200m (미사용) |
| **컨텍스트 스위칭** | 높음 (지연) | 중간 | 낮음 |

---

## ⚖️ 트레이드오프

### 높은 메모리 설정 (4GB+)
✅ GC 빈도 감소 → 응답시간 개선  
✅ OOM 위험 낮음  
❌ 비용 증가 (메모리 리소스 비싼 편)  
❌ Pod 밀도 감소 (워커 노드당 pod 수 감소)  

### 낮은 메모리 설정 (512MB)
✅ 비용 절감  
✅ Pod 밀도 증가 (노드 활용률 증가)  
❌ GC 빈도 높음 → 응답시간 저하  
❌ OOM 위험 높음  

### 권장 설정

```
마이크로서비스 기준:
- 요청 처리 시간이 짧으면 (< 100ms): 512MB ~ 1GB
- 요청 처리 시간이 길거나 배치라면: 2GB ~ 4GB

엔터프라이즈 기준:
- requests = limits의 50% (여유 보유)
- limits = 예상 필요 메모리의 150% (GC 포함)
```

---

## 📌 핵심 정리

1. **Java 8u191 이전**: JVM이 호스트 메모리를 인식 → 컨테이너 제한 무시 → OOMKilled

2. **UseContainerSupport**: cgroup 메모리 제한 감지 → Heap 자동 최적화 → 안정적

3. **MaxRAMPercentage=75%**: 메모리 제한의 75%를 Heap으로 → 나머지 25%는 Non-Heap/버퍼

4. **메모리 부족 시 GC 빈도 증가**: 응답시간 저하 → Heap 크기 증가 필수

5. **CPU 제한의 영향**: CFS 스로틀링 → CPU 대기 → 응답시간 증가 → 처리량 감소

6. **Kubernetes 리소스 관리**: requests로 스케줄링, limits로 OOM 기준 설정 → 안정성 확보

---

## 🤔 생각해볼 문제

**Q1. MaxRAMPercentage=75%, 메모리 제한 1GB일 때, 실제로 할당 가능한 메모리는?**

<details>
<summary>해설 보기</summary>

**계산:**

```
메모리 제한: 1GB = 1024MB

MaxRAMPercentage=75%:
Heap = 1024MB × 0.75 = 768MB

Non-Heap 구성:
- Metaspace: ~100MB (클래스 메타데이터)
- Code Cache: ~50MB (JIT 컴파일된 코드)
- Compressed Class Space: ~50MB
- 기타: ~50MB
- 합계: ~250MB

실제 할당 구조:
┌─────────────────────┐
│ 1024MB (제한)       │
├─────────────────────┤
│ Heap: 768MB         │ ← New + Old Generation
│ Non-Heap: 250MB     │ ← Metaspace, Code Cache 등
│ 직접 버퍼: 0MB      │ ← 기본값
│ 스레드 스택: ~20MB  │ ← 10개 스레드 × 2MB
│ 여유: ~10MB         │ ← 안전 마진
└─────────────────────┘
합계: ~1048MB (거의 제한 근처)
```

**문제:** 합계가 1024MB를 초과할 수 있음!

**해결책:**

```
더 안전한 설정:
MaxRAMPercentage=60% → Heap = 614MB
또는
메모리 제한 증가: 1.5GB

실제 권장:
- 메모리 제한: 1.5GB
- MaxRAMPercentage: 70%
- Heap: 1.5GB × 0.7 = 1050MB
- 여유: 450MB (Non-Heap, 버퍼)
```

</details>

---

**Q2. CPU 제한 1000m, 4개 스레드가 각각 250ms 동작하면 총 시간은?**

<details>
<summary>해설 보기</summary>

**단순 계산:**

```
4개 스레드 × 250ms = 1000ms (1초)

그런데 CPU 제한이 1000m(= 1개 코어)이면?
```

**실제 동작:**

```
CFS 스케줄링 (period = 100ms):
1초 = 10개 period

각 period마다 100ms의 CPU 시간 할당:

Period 1-10: 4개 스레드가 순차 실행
- 스레드 1: 0-25ms (25ms 사용)
- 스레드 2: 25-50ms (25ms 사용)
- 스레드 3: 50-75ms (25ms 사용)
- 스레드 4: 75-100ms (25ms 사용)
- 합계: 100ms (period 꽉 찼음)

실제 총 시간:
스레드 1: 0-25ms 실행
스레드 2: 25-50ms 실행
스레드 3: 50-75ms 실행
스레드 4: 75-100ms 실행
→ 1초 만에 완료 ✓

BUT, 4개 스레드가 독립적으로 250ms를 의도했다면?
→ 전체 시간: 250ms × 4 / 1 core = 1000ms
```

**결론:** CPU가 충분하면 병렬, 부족하면 순차 → 총 시간 길어짐

</details>

---

**Q3. requests와 limits를 같은 값으로 설정하면 어떤 문제가 생길까?**

<details>
<summary>해설 보기</summary>

**예시:**

```yaml
resources:
  requests:
    memory: "1Gi"
  limits:
    memory: "1Gi"    # ← 같은 값
```

**문제점:**

1. **GC 오버헤드 증가**
   ```
   JVM은 Heap의 일부를 확보하려고 함:
   - Heap full 70% → GC 실행
   - Heap full 90% → Full GC 실행
   
   메모리 여유가 없으면 → 자주 Full GC
   ```

2. **OOM 위험 증가**
   ```
   메모리 사용:
   - Heap: 750MB
   - Non-Heap: 250MB
   - 스택, 버퍼: 30MB
   - 합계: 1030MB > 1024MB (제한)
   
   → OOMKilled 위험!
   ```

3. **Pod 스케줄링 경쟁 심화**
   ```
   requests = limits라면:
   - 클러스터가 이 Pod에 정확히 1GB 예약
   - 다른 Pod이 들어올 공간 없음
   - 노드 활용률 저하
   ```

**권장 설정:**

```yaml
# 방법 1: requests < limits
resources:
  requests:
    memory: "512Mi"   # 예상 정상 사용
  limits:
    memory: "1Gi"     # 최악의 경우 (여유 보유)

# 방법 2: limits 제거 (권장하지 않음)
# 메모리 부족 시 전체 노드 영향

# 방법 3: 동적 할당 (권장)
# requests를 낮게, limits를 높게
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "2Gi"
```

</details>

---

<div align="center">

**[⬅️ 이전: Kafka 성능 테스트 — Producer/Consumer 처리량 측정](./03-kafka-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chaos Engineering 기초 — 의도적 장애 주입 ➡️](./05-chaos-engineering.md)**

</div>
