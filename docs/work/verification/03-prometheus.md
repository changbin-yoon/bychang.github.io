---
title: 03. Prometheus Agent 모드, RBAC, 멀티클러스터
---

# 📊 모니터링 — Prometheus Agent 모드, RBAC, 멀티클러스터

!!! abstract "요약"
    kube-prometheus-stack을 Agent 모드로 운영하며 겪은 구조적 이슈와 apiserver 메트릭 해석 기록.

## Prometheus Agent 모드의 구조적 특성

5-master HA 클러스터에서 kube-prometheus-stack을 사용하되, **Prometheus Operator가 관리하는 CR이 `Prometheus` 가 아니라 `PrometheusAgent`** 라는 점이 초반 혼동 지점이었습니다.

```bash
kubectl get prometheusagent -A
```

!!! important "Agent 모드의 핵심 제약 — 로컬 TSDB가 없다"
    데이터는 remote_write 대상(Thanos Receive / Mimir / VictoriaMetrics 등)에 저장됩니다.

    - `delete_series` 같은 admin API 조작은 Agent 자체에는 **의미가 없습니다** — 지우려면 remote_write 대상에서 처리해야 합니다.
    - 같은 이유로 **`defaultRules.create: false` 가 필요**합니다. Agent는 규칙을 평가(evaluate)할 수 없기 때문입니다.

## 메트릭 드롭 전략

특정 메트릭(`apiserver_request_duration_seconds_*` 등)을 걷어내고 싶을 때, 이미 원격지에 적재된 과거 데이터를 지우는 것보다 **ServiceMonitor의 `metricRelabelings` 로 수집 시점에 걷어내는 것이 정공법**입니다.

| 설정 | 대상 |
| --- | --- |
| `relabelings` | 타겟 자체를 필터링 |
| `metricRelabelings` | 수집된 메트릭의 라벨 / 값을 필터링 |

## apiserver 인증서 메트릭 해석

`apiserver_client_certificate_expiration_seconds_bucket` 은 카운터 계열 히스토그램이라 **원칙적으로 감소하지 않습니다.** 그래프에서 값이 떨어져 보이는 경우는 실제 감소가 아니라 아래 중 하나입니다.

| 관찰 현상 | 실제 원인 |
| --- | --- |
| 그래프가 절벽처럼 뚝 떨어짐 | apiserver 파드 재시작으로 카운터 리셋 (`rate()` / `increase()` 는 자동 보정) |
| 완만하게 내려감 | `rate()` 로 보고 있어서 — 요청량 감소가 정상 반영된 것 |
| 라인이 끊김 | 해당 라벨 조합의 시계열이 stale 처리됨 |

5-master 환경에서는 `sum by (instance)(...)` 로 **특정 마스터만 영향받는지 구분**하는 것이 진단의 시작점입니다.

## 멀티클러스터 Agent 구성

- `externalLabels` 로 클러스터 식별 라벨을 remote_write 페이로드에 주입
- `writeRelabelConfigs` 로 remote_write 이전 단계에서 메트릭 사전 드롭
- **etcd 스크레이핑은 예외적으로 mTLS client cert를 사용**하며, 이 Secret은 자동 갱신되지 않으므로 별도 관리가 필요 (다른 컴포넌트는 kubelet이 자동 회전하는 projected ServiceAccount 토큰 사용)

## 결론

Agent 모드는 **"로컬 저장이 없는 Prometheus"** 로 이해하면 대부분의 혼동이 풀립니다 — admin API, 규칙 평가, 메트릭 삭제 모두 이 전제 위에서 판단해야 합니다.
