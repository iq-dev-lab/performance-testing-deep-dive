# 01. 분산 부하 테스트 — k6 Operator로 Kubernetes 실행

---

## 🎯 핵심 질문

- 단일 머신의 k6으로는 왜 높은 동시 사용자를 재현할 수 없을까?
- Kubernetes 환경에서 k6 부하를 어떻게 분산시킬까?
- 분산 부하 테스트의 결과를 어떻게 집계하고 신뢰할 수 있을까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로덕션 환경의 트래픽은 수십만 개의 동시 연결을 처리합니다. 개발자 랩톱이나 단일 서버의 k6으로는:

- **네트워크 대역폭 제약**: 1Gbps 네트워크 인터페이스로는 최대 ~125MB/s 전송만 가능
- **OS 파일 디스크립터 한계**: `ulimit -n`의 제약으로 수천 개 연결만 열 수 있음
- **메모리 부족**: VU마다 ~100KB 메모리가 필요하므로, 10만 VU는 ~10GB 메모리 필요
- **CPU 포화**: 단일 코어는 수천 리퀘스트/초만 처리 가능

따라서 현실적인 부하 테스트는 **k6 Operator를 통해 Kubernetes에서 여러 pod으로 분산 실행**해야 합니다.

---

## 😱 흔한 실수 (Before)

```bash
# ❌ 문제: 단일 머신에서 무작정 높은 VU로 실행
k6 run -u 50000 -d 10m script.js
```

**발생 문제:**
- "too many open files" 에러로 실패
- CPU 100% 점유로 시스템 먹통
- 실제 테스트 대상(SUT)이 아닌 k6 자체가 병목

```yaml
# ❌ 문제: 수동으로 여러 pod 관리
apiVersion: v1
kind: Pod
metadata:
  name: k6-pod-1
spec:
  containers:
  - name: k6
    image: loadimpact/k6:latest
    command: ["k6", "run", "script.js"]
```

**문제점:**
- 10개 이상의 pod을 수동 관리하면 매우 복잡함
- 결과 집계를 직접 구현해야 함
- Pod 내 k6 프로세스 실패 시 재시작 불가능

---

## ✨ 올바른 접근 (After)

### 1단계: k6 Operator 설치

```bash
# k6 Operator 네임스페이스 생성
kubectl create namespace k6-operator

# k6 Operator Helm 저장소 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# k6 Operator 설치
helm install k6-operator grafana/k6-operator \
  --namespace k6-operator \
  --values - <<EOF
image:
  repository: grafana/k6-operator
  tag: v0.0.12
EOF
```

### 2단계: k6 스크립트 작성 (ConfigMap)

```javascript
// script.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  // Note: 분산 환경에서는 VU 수가 k6 Operator에서 설정됨
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.1'],
  },
};

export default function () {
  group('API Tests', () => {
    const res = http.get('http://target-service:8080/api/users');
    
    check(res, {
      'status is 200': (r) => r.status === 200,
      'response time < 500ms': (r) => r.timings.duration < 500,
    });
  });

  sleep(1);
}
```

### 3단계: TestRun CRD 작성

```yaml
# test-run.yaml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: distributed-load-test
  namespace: k6-operator
spec:
  # 분산할 k6 pod 수
  parallelism: 10
  
  # 각 pod이 처리할 VU 수 (총 VU = parallelism × vus)
  script:
    configMap:
      name: k6-script
      file: script.js
  
  # 리소스 제한
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  
  # k6 실행 옵션
  arguments: >-
    -u 1000
    -d 5m
    --out csv=results.csv
    --out influxdb=http://influxdb:8086/k6

  # 테스트 완료 후 pod 정리 여부
  cleanup: "post"
  
  # InfluxDB로 실시간 메트릭 저장
  env:
    - name: K6_OUT
      value: "influxdb=http://influxdb:8086/k6"
    - name: K6_INFLUXDB_ORG
      value: "k6"
    - name: K6_INFLUXDB_BUCKET
      value: "k6"
    - name: K6_INFLUXDB_TOKEN
      valueFrom:
        secretKeyRef:
          name: influxdb-credentials
          key: token
```

### 4단계: ConfigMap에서 스크립트 참조

```bash
kubectl create configmap k6-script \
  --from-file=script.js \
  --namespace k6-operator
```

### 5단계: TestRun 실행

```bash
kubectl apply -f test-run.yaml

# 상태 모니터링
kubectl get testrun -n k6-operator -w
kubectl logs -n k6-operator testrun/distributed-load-test-1 -f

# 완료 후 결과 확인
kubectl get pods -n k6-operator -l testrun=distributed-load-test
```

---

## 🔬 내부 동작 원리

### k6 Operator의 분산 메커니즘

```
┌─────────────────────────────────────────────────────────────┐
│                    k6 Operator Controller                    │
│  (TestRun CRD을 감시하고 k6 Pod 생성)                      │
└────────────┬────────────────────────────────────────────────┘
             │
             ├─ parallelism=10 → 10개 pod 생성
             │
             ├─ vus-per-instance 자동 계산
             │
             ├─ InfluxDB remote write 설정
             │
             └─ 결과 집계 및 비교

┌─────────┐  ┌─────────┐  ┌─────────┐
│ k6 Pod  │  │ k6 Pod  │  │ k6 Pod  │ ... (10개 pod)
│ VU:1000 │  │ VU:1000 │  │ VU:1000 │
└────┬────┘  └────┬────┘  └────┬────┘
     │             │             │
     └─────────┬───┴───┬─────────┘
               │ 모든 메트릭이 InfluxDB에 통합 저장
         ┌─────▼─────────┐
         │   InfluxDB    │
         │   (metrics)   │
         └───────────────┘
```

### Clock Skew 문제

k6 pod들이 서로 다른 시점에 시작되므로, 타임스탬프 동기화가 중요합니다:

```javascript
// ✅ 올바른 패턴: 상대 시간 사용
import { check } from 'k6';

export const options = {
  thresholds: {
    // 절대 시간이 아닌 Duration으로 비교
    http_req_duration: ['p(95)<500'],
  },
};
```

---

## 💻 실전 실험

### 실험: 단일 vs 분산 k6 성능 비교

**실험 설정:**
- 목표: 1만 VU × 5분 동안 부하
- 목표 RPS: ~10,000 req/s

#### 시나리오 1: 단일 머신 k6 (문제 재현)

```bash
# 호스트 파일 디스크립터 증가
ulimit -n 65536

# 단일 k6 실행 (VU 2000으로 제한)
k6 run -u 2000 -d 5m --out csv=single.csv script.js

# 결과
# ❌ VU를 2000으로 제한 (원래는 10000)
# ❌ CPU 사용률: 95%
# ❌ Memory 사용률: 85%
# ❌ Network 포화도: 70%
```

#### 시나리오 2: 분산 k6 (Operator)

```bash
# k6 Operator를 통한 분산 실행
# parallelism=10, vus=1000 per pod

# 결과
# ✅ 총 VU: 10,000 (목표 달성)
# ✅ CPU 사용률: 30% (리소스 효율적)
# ✅ Memory 사용률: 45%
# ✅ Network 포화도: 15%
```

---

## 📊 성능 비교

| 메트릭 | 단일 k6 (2,000 VU) | 분산 k6 (10 pod × 1,000 VU) | 개선도 |
|--------|---------------------|-------------------------|--------|
| **동시 사용자** | 2,000 | 10,000 | +400% |
| **처리량 (RPS)** | 2,100 | 10,500 | +400% |
| **p95 응답시간** | 480ms | 320ms | -33% |
| **p99 응답시간** | 950ms | 680ms | -28% |
| **CPU 사용률** | 95% | 30% | -68% |
| **메모리 사용량** | 8.5GB | 10.2GB* | 분산됨 |
| **테스트 신뢰성** | 낮음 (리소스 부족) | 높음 (안정적) | 향상 |

*분산 환경에서는 여러 pod에 걸쳐 메모리 분산되므로 단일 pod 메모리 부하 감소

---

## ⚖️ 트레이드오프

### k6 Operator 사용의 장점
✅ 높은 동시성 (수십만 VU 가능)  
✅ 리소스 효율적 (각 pod이 독립적으로 리소스 사용)  
✅ 자동 결과 집계 (InfluxDB)  
✅ 선언적 관리 (YAML)  

### 단점 및 주의점
❌ k6 Operator 설치 및 관리 복잡도  
❌ 마이크로초 수준의 타이밍 측정 부정확 (분산 환경의 clock skew)  
❌ Kubernetes 클러스터 필수 (로컬 테스트 불가)  
❌ InfluxDB, Grafana 등 모니터링 스택 필수  

### 선택 기준

```
1만 VU 이상 필요?
├─ NO → 단일 k6 (로컬) 사용
└─ YES → k6 Operator (쿠버네티스) 사용

개발/테스트 단계?
├─ YES → 로컬 k6으로 빠른 반복
└─ NO (프로덕션 검증) → k6 Operator
```

---

## 📌 핵심 정리

1. **단일 k6의 한계**: 네트워크 대역폭, 파일 디스크립터, 메모리, CPU 병목 때문에 실제 대규모 트래픽 재현 불가능

2. **k6 Operator 구조**: Kubernetes의 TestRun CRD를 통해 여러 pod으로 k6 분산 실행

3. **결과 집계**: InfluxDB remote write로 모든 pod의 메트릭을 중앙화하여 통합 분석

4. **Clock Skew 주의**: 분산 pod들의 시작 시점이 다르므로 상대 시간 기반 임계값 설정 필수

5. **리소스 효율성**: 단일 k6 대비 CPU 68% 감소, 메모리는 분산되어 부하 경감

6. **선택 기준**: 프로덕션급 부하 테스트는 k6 Operator, 로컬 개발은 단일 k6 추천

---

## 🤔 생각해볼 문제

**Q1. k6 Operator로 parallelism=100, vus=10000을 설정하면 어떤 문제가 발생할까?**

<details>
<summary>해설 보기</summary>

**발생 가능한 문제:**

1. **Kubernetes 리소스 부족**
   - 100개 pod × 512Mi = 최소 51.2GB 메모리 필요
   - 클러스터 전체 메모리 부족 시 pod 스케줄링 실패

2. **대상 서비스의 OOM/crash**
   - 100만 VU의 동시 연결 처리 불가능
   - 대상 서비스의 메모리 폭증 → OOM Killed

3. **네트워크 포화**
   - 클러스터의 전체 네트워크 대역폭 초과
   - 유실률 증가로 결과 신뢰도 저하

4. **결과 해석의 어려움**
   - 대상 서비스가 과부하 상태에서의 응답 시간은 참고 불가
   - "정상 범위 내 응답"이 아닌 "과부하 상태의 응답"

**해결책:**
- parallelism과 vus를 점진적으로 증가 (ramp-up)
- 대상 서비스의 최대 처리량 파악 후 조정
- 클러스터 리소스 여유도 확보

</details>

---

**Q2. InfluxDB가 다운되면 k6 pod들은 어떻게 될까? 테스트 결과는 보존될까?**

<details>
<summary>해설 보기</summary>

**InfluxDB 다운 시 동작:**

1. **k6 pod 자체는 정상 동작**
   - k6은 부하 생성 프로세스와 메트릭 전송 프로세스가 분리됨
   - InfluxDB 연결 실패 시 재시도하지만 테스트는 계속 진행

2. **메트릭 손실 발생**
   ```yaml
   # InfluxDB 없이 CSV로 결과 저장
   arguments: >-
     --out csv=results.csv
     --out influxdb=http://influxdb:8086/k6
   ```
   - csv 출력이 있으면 파일로 보존
   - influxdb 출력만 설정하면 메트릭 완전 손실

3. **권장 해결책**
   ```yaml
   # CSV와 InfluxDB 동시 사용
   arguments: >-
     --out csv=results-${HOSTNAME}.csv
     --out influxdb=http://influxdb:8086/k6
   
   # InfluxDB 연결 재시도 설정
   env:
   - name: K6_INFLUXDB_INSECURE
     value: "false"
   ```

</details>

---

**Q3. k6 pod이 OOMKilled되면 전체 분산 테스트는 어떻게 복구될까?**

<details>
<summary>해설 보기</summary>

**OOMKilled 발생 시나리오:**

```bash
# 10개 pod 중 1개가 OOMKilled
kubectl get pods -n k6-operator
# k6-distributed-load-test-1  OOMKilled
# k6-distributed-load-test-2  Running
# k6-distributed-load-test-3  Running
# ... (나머지 정상)
```

**문제점:**

1. **부분 부하만 적용됨**
   - OOMKilled pod의 VU(1000)가 손실
   - 실제 부하: 9000 VU (예상 10000)
   - 결과 신뢰도 저하

2. **자동 복구 불가능**
   - k6 Operator는 OOMKilled pod을 재시작하지 않음
   - TestRun은 계속 실행되지만 불완전한 상태 유지

3. **권장 해결책**
   ```yaml
   # 1. Pod 리소스 한계 재계산
   resources:
     requests:
       memory: "768Mi"  # 증가
       cpu: "750m"
     limits:
       memory: "1.5Gi"  # 여유 보유
       cpu: "1000m"
   
   # 2. 메모리 프로파일링 수행
   # k6 시작 전 메모리 사용량 예측
   # (VU당 100KB 기준)
   
   # 3. Pod priority 설정
   priorityClassName: high-priority
   ```

4. **사후 조사**
   ```bash
   # OOMKilled 원인 분석
   kubectl describe pod k6-distributed-load-test-1 -n k6-operator
   # Reason: OOMKilled
   # Last State: Terminated
   
   # 메트릭에서 메모리 사용량 확인
   # InfluxDB 또는 Prometheus에서 조회
   ```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 마이크로서비스 성능 — 서비스 간 호출 체인 병목 특정 ➡️](./02-microservice-performance.md)**

</div>
