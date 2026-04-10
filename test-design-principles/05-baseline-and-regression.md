# 05. 성능 기준선(Baseline) 수립 — 회귀 감지 자동화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 성능 기준선(Baseline)이란 무엇이고, 왜 튜닝 전에 반드시 측정해야 하는가?
- CI Pipeline에 k6를 통합해 배포마다 자동으로 성능 회귀를 감지하는 방법은?
- 성능 회귀가 발생했을 때 배포를 자동으로 차단하려면 어떻게 설정하는가?
- Baseline을 자동으로 업데이트하는 전략은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"이번 배포 전까지는 빨랐는데"라는 말은 기준선 없이 성능을 논할 때 나온다. 성능 회귀(Regression)는 새 기능 추가, 의존성 업그레이드, 설정 변경 등 어디서든 발생할 수 있다. 기준선이 없으면 회귀를 발견하는 시점이 사용자 불만이나 장애가 발생한 후가 된다. Baseline을 수립하고 CI에 통합하면 코드 리뷰 단계에서 성능 회귀를 차단할 수 있다.

---

## 😱 흔한 실수 (Before — 기준선 없이 개선을 논하는 접근)

```
상황: 쿼리 최적화 후 성능 개선 보고

개선 후: p99 = 380ms

팀장: "좋아졌나요?"
개발자: "네, 분명히 빨라진 것 같습니다."
팀장: "수치로 보여주세요."
개발자: "...전에 측정한 게 없어서..."

문제 1: 기준선 없으면 개선 효과를 수치로 증명 불가
문제 2: 다음 배포에서 새 기능을 추가했더니 p99 = 620ms
  → "이번 배포로 느려진 건지" 판단 불가 (기준선 없으니까)
  → 사용자 CS 2주 후에야 발견

올바른 상황:
  기준선: p99 = 820ms (측정 날짜 기록)
  최적화 후: p99 = 380ms → 개선율 54% 증명 가능
  다음 배포 후: p99 = 420ms → CI에서 기준선 대비 10% 이상 상승 감지 → 자동 경고
```

---

## ✨ 올바른 접근 (After — Baseline + CI 자동화)

```
Baseline 수립 전략:

1. 첫 Baseline 측정 (배포 전)
   k6 run --out json=baseline-YYYY-MM-DD.json k6/load-test.js
   → p50/p95/p99 기록

2. CI Pipeline 통합
   모든 PR: Smoke Test (5분, VU 20)
   develop 브랜치 머지: 완전한 Load Test
   Baseline 대비 p99 10% 이상 상승 시 → 빌드 실패

3. Baseline 자동 갱신
   main 브랜치 배포 성공 시 Baseline 자동 업데이트
   → "지금이 새 기준"

자동화 결과:
  성능 회귀 발견 시점: 사용자 CS → 코드 리뷰 단계로 앞당김
  회귀 발견 비용: 운영 장애 복구 vs PR 수정
```

---

## 🔬 내부 동작 원리

### Baseline의 구성 요소

```
성능 Baseline = {측정 조건} + {측정 결과}

측정 조건:
  - 날짜/시간 (환경 변화 추적)
  - Git 커밋 해시 (코드 버전 특정)
  - 환경 설정 (JVM 옵션, DB 버전, 데이터 볼륨)
  - 테스트 설정 (VU 수, 부하 패턴, 실행 시간)

측정 결과:
  - p50, p95, p99 응답시간 (엔드포인트별)
  - 에러율
  - TPS (초당 처리 요청 수)
  - 리소스 사용률 (CPU, 메모리)

저장 형식:
  k6 --out json=results/baseline-$(git rev-parse --short HEAD).json
  → InfluxDB에 태그로 커밋 해시 포함
```

### 성능 회귀 감지 기준

```
회귀 판정 기준 예시:

경고 (Warning):
  - 현재 p99 > Baseline p99 × 1.1 (10% 이상 상승)
  - 에러율 > 0.1%

실패 (Failure, 배포 차단):
  - 현재 p99 > Baseline p99 × 1.3 (30% 이상 상승)
  - 에러율 > 1%
  - TPS < Baseline TPS × 0.8 (20% 이상 하락)

주의:
  - 너무 엄격한 기준(5% 이상 상승 시 실패)은
    노이즈(측정 변동성)로 인한 오탐이 많아짐
  - 너무 느슨한 기준(50% 이상)은 회귀를 놓침
  - 통상 10~20% 범위가 적절
```

### CI Pipeline에서의 흐름

```
PR 생성
  ↓
[자동] Smoke Test 실행 (5분, VU 20)
  - SLO 위반 시: PR 차단 (빠른 피드백)
  ↓
코드 리뷰 + Merge to develop
  ↓
[자동] Full Load Test 실행 (15분, VU 200)
  - Baseline 대비 비교
  - 10% 이상 회귀: Slack 경고
  - 30% 이상 회귀: 배포 파이프라인 차단
  ↓
[조건부] 통과 시 → Staging 배포
  ↓
[수동 or 자동] Baseline 업데이트
  ↓
Production 배포
```

---

## 💻 실전 실험 — GitHub Actions CI 통합

### k6 Smoke Test (모든 PR)

```javascript
// k6/smoke-test.js — CI용 최소 성능 검증
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 20,
  duration: '5m',
  thresholds: {
    // SLO 절대 기준 — Baseline 관계없이 항상 통과해야 함
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/products/1', {
    tags: { name: 'product_detail' },
  });
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

### GitHub Actions 워크플로우 — PR Smoke Test

```yaml
# .github/workflows/performance-smoke.yml
name: Performance Smoke Test

on:
  pull_request:
    branches: [develop, main]

jobs:
  smoke-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Compose
        run: docker compose up -d --wait

      - name: Wait for app to be ready
        run: |
          until curl -sf http://localhost:8080/actuator/health; do
            sleep 5
          done

      - name: Run k6 Smoke Test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: k6/smoke-test.js
          flags: --out json=results/smoke-result.json

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: smoke-test-results
          path: results/

      - name: Tear down
        if: always()
        run: docker compose down
```

### GitHub Actions 워크플로우 — Baseline 비교 Load Test

```yaml
# .github/workflows/performance-load.yml
name: Performance Load Test (Baseline Compare)

on:
  push:
    branches: [develop]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up environment
        run: docker compose up -d --wait

      - name: Download baseline
        uses: actions/download-artifact@v4
        with:
          name: performance-baseline
          path: results/baseline/
        continue-on-error: true  # 최초 실행 시 Baseline 없을 수 있음

      - name: Run Load Test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: k6/load-test.js
          flags: --out json=results/current.json

      - name: Compare with Baseline
        id: compare
        run: |
          python3 scripts/compare-baseline.py \
            --baseline results/baseline/result.json \
            --current results/current.json \
            --threshold 0.3
          echo "regression=$?" >> $GITHUB_OUTPUT

      - name: Update Baseline (성공 시)
        if: steps.compare.outputs.regression == '0'
        uses: actions/upload-artifact@v4
        with:
          name: performance-baseline
          path: results/current.json

      - name: Notify Slack (회귀 발견 시)
        if: steps.compare.outputs.regression != '0'
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: "⚠️ 성능 회귀 감지: p99가 Baseline 대비 30% 이상 상승했습니다."
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Tear down
        if: always()
        run: docker compose down
```

### Baseline 비교 스크립트

```python
#!/usr/bin/env python3
# scripts/compare-baseline.py

import json
import sys
import argparse
from pathlib import Path

def load_k6_results(file_path):
    """k6 JSON 결과 파일에서 주요 메트릭 추출"""
    if not Path(file_path).exists():
        return None

    metrics = {}
    with open(file_path) as f:
        for line in f:
            data = json.loads(line)
            if data.get('type') == 'Point':
                metric = data['metric']
                if metric == 'http_req_duration':
                    val = data['data']['value']
                    tag = data['data'].get('tags', {}).get('name', 'all')
                    if tag not in metrics:
                        metrics[tag] = []
                    metrics[tag].append(val)
    return metrics

def percentile(values, p):
    if not values:
        return 0
    sorted_vals = sorted(values)
    idx = int(len(sorted_vals) * p / 100)
    return sorted_vals[min(idx, len(sorted_vals) - 1)]

def compare(baseline_file, current_file, threshold):
    baseline = load_k6_results(baseline_file)
    current = load_k6_results(current_file)

    if baseline is None:
        print("⚠️  Baseline 없음 — 현재 결과를 새 Baseline으로 저장합니다")
        return 0

    print("\n=== 성능 비교 결과 ===\n")
    print(f"{'엔드포인트':<25} {'Baseline p99':>15} {'현재 p99':>12} {'변화율':>10} {'판정':>6}")
    print("-" * 70)

    regression = False
    for tag in current:
        if tag not in baseline:
            continue
        b_p99 = percentile(baseline[tag], 99)
        c_p99 = percentile(current[tag], 99)
        change = (c_p99 - b_p99) / b_p99 if b_p99 > 0 else 0
        status = "✅" if change < threshold else "❌"
        if change >= threshold:
            regression = True
        print(f"{tag:<25} {b_p99:>14.1f}ms {c_p99:>11.1f}ms {change:>+9.1%} {status:>6}")

    print()
    if regression:
        print(f"❌ 성능 회귀 감지 — p99가 Baseline 대비 {threshold:.0%} 이상 상승")
        return 1
    else:
        print("✅ 성능 회귀 없음 — Baseline 이내")
        return 0

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--baseline', required=True)
    parser.add_argument('--current', required=True)
    parser.add_argument('--threshold', type=float, default=0.3)
    args = parser.parse_args()

    result = compare(args.baseline, args.current, args.threshold)
    sys.exit(result)
```

---

## 📊 성능 비교 — Baseline 도입 전후 회귀 발견 시점

### Baseline 없는 팀 vs 있는 팀

| 항목 | Baseline 없음 | Baseline + CI 자동화 |
|------|-------------|-------------------|
| 회귀 발견 시점 | 사용자 불만 or 장애 발생 후 | PR 코드 리뷰 단계 |
| 발견까지 걸리는 시간 | 수일~수주 | 수십 분 |
| 수정 비용 | 핫픽스 배포, 장애 대응 | PR 수정 (작업 중 컨텍스트 유지) |
| 회귀 원인 특정 | 여러 배포 중 어느 것인지 불명 | 정확히 어느 커밋인지 특정 |

### 실제 회귀 감지 예시

| 배포 | p99 | Baseline 대비 | CI 판정 |
|------|-----|-------------|--------|
| v1.0 (Baseline) | 185ms | — | — |
| v1.1 | 192ms | +3.8% | ✅ 통과 |
| v1.2 | 198ms | +7.0% | ✅ 통과 (경고 없음) |
| v1.3 | 280ms | **+51.4%** | ❌ **배포 차단** |

→ v1.3에서 신규 기능 추가 시 N+1 쿼리 도입 → CI에서 즉시 감지 → 머지 전 수정

---

## ⚖️ 트레이드오프

| 전략 | 장점 | 단점 |
|------|------|------|
| 절대 기준(SLO만 사용) | 간단, 오탐 적음 | 서서히 느려지는 회귀 미감지 |
| Baseline 대비 상대 기준 | 서서히 느려지는 회귀도 감지 | 측정 변동성으로 오탐 발생 가능 |
| 절대 + 상대 병행 | 가장 촘촘한 감지 | 관리 복잡도 증가 |
| 통계적 유의성 검정(t-test) | 오탐 최소화 | 구현 복잡, 충분한 샘플 필요 |

---

## 📌 핵심 정리

- **Baseline 없이는 개선도 회귀도 증명할 수 없다** — 측정일과 Git 해시를 함께 기록
- **Smoke Test**: 모든 PR마다 5분, VU 20으로 SLO 최소 기준 검증
- **Load Test + Baseline 비교**: develop 브랜치 머지마다 실행, p99 30% 이상 상승 시 배포 차단
- **Baseline 자동 갱신**: 배포 성공 후 현재 결과를 새 Baseline으로 저장
- 회귀 감지 기준은 **10~20% 경고, 30% 실패**가 오탐과 미탐의 균형점

---

## 🤔 생각해볼 문제

**Q1.** Baseline 업데이트를 배포마다 자동으로 해야 하는가, 아니면 수동으로 승인 후 해야 하는가?

<details>
<summary>해설 보기</summary>

두 가지 방법 모두 타당하다. 자동 업데이트: 매 성공적인 배포 후 Baseline을 갱신하면 점진적인 성능 저하(예: 배포마다 5%씩 느려짐)를 감지하기 어려워진다. 수동 승인: 팀이 "이 성능 수준이 새 기준"임을 명시적으로 확인하는 과정. 권장 절충안: 자동 업데이트하되, 월 1회 절대 기준(최초 Baseline 대비)과 비교하는 리뷰를 추가한다.

</details>

**Q2.** 성능 테스트를 CI에 통합했더니 매 PR마다 5~10분씩 추가 대기가 발생했다. 팀 불만이 높아지는데 어떻게 균형을 맞추는가?

<details>
<summary>해설 보기</summary>

①Smoke Test 시간 단축: VU를 줄이거나 기간을 3분으로 단축. ②병렬 실행: 단위 테스트와 성능 테스트를 동시 실행. ③선택적 실행: `[skip perf]` 커밋 메시지로 긴급 핫픽스 시 건너뛰기 허용. ④성능에 영향 없는 변경(문서, 스타일)은 제외 경로 설정. 결국 팀이 "성능 회귀로 인한 핫픽스 비용 > 5분 대기 비용"임을 체감하면 인식이 바뀐다.

</details>

**Q3.** 측정 변동성(노이즈)으로 인해 실제 회귀가 없는데도 CI가 실패하는 경우를 어떻게 줄이는가?

<details>
<summary>해설 보기</summary>

①판정 기준을 완화(10% → 20%)해 노이즈 영역을 허용. ②동일 테스트를 3회 실행해 중간값 사용. ③통계적 유의성 검정(Welch's t-test) 도입 — p-value < 0.05일 때만 회귀로 판정. ④InfluxDB에 장기 데이터를 쌓아 이동 평균과 비교 — 단발적 측정치보다 안정적. 완벽한 제거는 불가능하며, 오탐율과 미탐율의 트레이드오프를 팀이 결정해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: 테스트 환경 구성 — Prod-like 원칙](./04-test-environment-setup.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — k6 아키텍처 ➡️](../k6-deep-dive/01-k6-architecture.md)**

</div>
