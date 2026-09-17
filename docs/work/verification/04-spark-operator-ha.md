---
title: 04. Spark Operator HA & 리더 일렉션
---

# ⚡ Spark Operator HA & 리더 일렉션 — 소스 레벨 분석

!!! abstract "요약"
    kubeflow/spark-operator master(controller-runtime v0.23.3, client-go v0.35.0) 소스를 직접 읽어 검증한 HA / failover 분석.
    190노드, Spark 자원 60%, Airflow(SparkKubernetesOperator) → Spark Operator v2 → YuniKorn 구조.

## TL;DR

Spark Operator의 HA 자체는 걱정할 대상이 아니었습니다. 리더가 죽어도 실행 중인 작업은 계속 돌고, failover 자체는 30초 내외의 이벤트입니다. 정작 문제는 **failover 직후**에서 생겨났습니다.

1. **failover 자체는 "지연"이지 "유실"이 아니다** — 공백 구간 SparkApplication CR은 etcd에 저장되고 새 리더가 처리한다.
2. **workqueue 전역 토큰버킷이 진짜 함정이다.** CR 생성 이벤트도 `AddRateLimited`로 큐잉되는데, executor pod churn이 같은 버킷을 공유하기 때문에 이벤트 폭주 시 신규 작업이 10분씩 조용히 서 있게 된다. 이 지연은 `workqueue_depth` 지표에 잡히지 않는다.
3. **Completed CR 방치가 failover 비용을 키운다.** 4만 건의 Completed CR이 failover burst에서 각각 최소 1회의 API write를 유발해, 기본 QPS 20 기준 약 33분의 burst를 만들었다.
4. **Operator pod의 CFS throttling**이 failover 복구를 20분까지 늘릴 수 있다. worker 10개가 동시에 spark-submit JVM을 fork하는 순간 CPU limit에 걸리면 전체가 기어간다.
5. **Airflow는 리더 교체를 사실상 감지하지 못한다.** 유일한 즉시 실패 경로는 webhook 동반 사망이며, 방어책은 `webhook.replicas: 2+`와 podAntiAffinity.

## HA 구조 — "standby 옵션"은 없다

leader election을 켜고 replica 2개 이상을 띄우면, 리더가 아닌 replica가 자동으로 standby가 되는 구조입니다. standby는 옵션이 아니라 **결과**입니다.

```yaml
controller:
  replicas: 3          # 1 leader + 2 standby
  leaderElection:
    enable: true       # Helm 기본값 true (바이너리 기본값은 false)
    leaseDuration: 15s
    renewDeadline: 10s
    retryPeriod: 2s
```

lock은 `coordination.k8s.io/v1` **Lease** 리소스로 생성됩니다. (플래그 설명에는 "ConfigMap"이라고 적혀 있지만 v1 시절 문구가 남은 것이고, v2는 controller-runtime 기반이라 실제로는 Lease를 씁니다.) ServiceAccount에 `leases` get/create/update 권한이 없으면 리더 선출 실패로 controller가 기동 중 죽습니다.

```bash
kubectl -n spark-operator get lease spark-operator-lock -o yaml
# spec.holderIdentity = 현재 리더 pod
```

## 소스 레벨에서 확인한 두 가지 핵심 발견

1. **`LeaderElectionReleaseOnCancel` 이 주석 처리되어 있다.**
   계획된 재시작(rollout restart)도 lease를 자발적으로 반납하지 않아, crash와 동일하게 ~15초의 lease 만료 대기를 치릅니다. 즉 **"무중단 계획 재기동"은 현재 불가능**합니다.
2. **healthz / readyz에 `healthz.Ping` 만 등록되어 있다.**
   캐시 sync 여부가 readyz에 반영되지 않아 standby의 warm 상태를 readyz로 판단할 수 없습니다.

!!! note "소스 위치"
    - HA 설정 진입점 — `cmd/operator/controller/start.go`
    - reconciler 게이팅 — controller-runtime `manager/internal.go` 의 `OnStartedLeading` → `startLeaderElectionRunnables()`
    - lease 획득 — client-go `leaderelection.go` 의 `acquire()` / `tryAcquireOrRenew()`

    만료 판정은 renewTime이 아니라 **관찰자 로컬 `observedTime` 기준**이라 clock skew에 안전합니다.

## Failover 타임라인과 workqueue 함정

```text
리더 사망 → lease 만료 대기(~15s) → 획득 경쟁(retryPeriod jitter ~2.4s)
  → informer 초기 sync(전체 CR에 Add 이벤트) → reconcile burst(CR 수 ÷ 유효 처리율)
```

reconcile burst가 변수입니다. 앱 100개면 기본 QPS 20 기준 ~10초지만, **Completed CR 4만 건이 쌓여 있으면 33분**이 됩니다(40,000 ÷ 20 = 2,000초).

원인은 `updateSparkApplicationStatus` 가 DeepEqual 생략 없이 무조건 `Status().Update()` 를 호출하기 때문입니다. 완료된 CR조차 failover burst에서 각각 쓰기를 유발합니다.

또 다른 사례에서는 operator pod의 **CFS CPU throttling**이 20분 지연의 보다 직접적인 원인으로 지목됐습니다 — `--controller-threads`(기본 10)가 동시에 spark-submit JVM을 fork할 때 CPU limit에 걸려 전체가 지연됩니다.

!!! warning "해결 방향"
    CPU limit 자체를 제거하는 것(request만 유지)이 해법이며, `controller-threads` 를 늘리는 것은 **오히려 악화**시킵니다.

## Failover 중 작업 처리 시나리오

| 상황 | 동작 | 결과 |
| --- | --- | --- |
| 이미 Running 중인 앱 | driver / executor pod는 리더와 무관하게 계속 실행 | 영향 없음, status 갱신만 일시 정지 |
| 공백 구간에 제출된 앱 | CR은 etcd에 저장, status는 빈 채(New) 방치 | 새 리더가 큐 소화 후 정상 제출 — **지연**만 발생 |
| submit 직후 status 기록 전 리더 사망 | at-least-once 재제출 race | deterministic driver pod 이름으로 `AlreadyExists` 방어, 드물게 FAILED 마킹 |

새 리더의 workqueue에는 기존 Running 앱과 신규 New 앱이 **구분 없이 함께** 들어가고, `--controller-threads`(기본 10, `MaxConcurrentReconciles`)개 worker가 병렬 처리합니다. 우선순위 큐가 아니므로 순서는 비결정적이고, 같은 key(같은 앱)에 대한 직렬화만 보장됩니다.

## 운영 튜닝 파라미터

- `--kube-api-qps 100` / `--kube-api-burst 200`
- `--workqueue-ratelimiter-bucket-qps 200` / `bucket-size 5000`
- `--controller-threads` 는 queue_duration vs work_duration 분석 후에만 증가
- 신규 SparkApplication에 `spec.timeToLiveSeconds` 적용 → **Completed CR 방치 방지** (미적용 시 4만 건처럼 적체될 수 있음)
- `webhook.replicas: 2+` + podAntiAffinity

## 진단 메트릭

```text
container_cpu_cfs_throttled_periods_total
workqueue_work_duration_seconds (p99)
workqueue_longest_running_processor_seconds
apiserver_registered_watchers
controller_runtime_active_workers / max_concurrent_reconciles
```

## 결론

failover 자체는 걱정할 것이 아니지만, **방치된 Completed CR + 공유 토큰버킷 + CFS throttling** 이 겹치면 수십 분짜리 지연이 됩니다. 핵심 조치는 **CR TTL 도입**과 **CPU limit 제거** 두 가지로 요약됩니다.
