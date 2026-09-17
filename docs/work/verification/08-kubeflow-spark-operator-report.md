---
title: 08. Kubeflow Spark Operator 운영 분석 보고서
---

# Kubeflow Spark Operator 운영 분석 보고서

!!! abstract "대상 환경"
    베어메탈 Kubernetes 약 190노드(마스터 5 + 워커 ~185, 노드당 96코어/1TB), 자원의 약 60%가 Spark, 30%가 Trino 워크로드. Airflow(SparkKubernetesOperator) → Spark Operator v2 → YuniKorn 갱 스케줄러 스택, Cilium CNI, AIStor(MinIO) 스토리지, 폐쇄망(air-gapped).

    **분석 기준** — kubeflow/spark-operator master(2026-08), controller-runtime v0.23.3, client-go v0.35.0 소스 코드를 직접 확인한 내용. 버전에 따라 파일 위치와 기본값이 다를 수 있음.

이 보고서는 [Spark Operator HA & 리더 일렉션](04-spark-operator-ha.md)의 조사를 바탕으로, 실제 운영 이슈 3건의 근본 원인 분석과 튜닝 가이드까지 종합한 최종 보고서입니다.

## 1. 요약 (TL;DR)

Spark Operator를 운영하며 던졌던 질문들을 관통하는 결론은 하나입니다. **HA failover 자체는 30초짜리 이벤트지만, 운영 사고는 전부 failover "직후"와 "평상시의 누적"에서 터집니다.**

1. **failover는 "지연"이지 "유실"이 아니다.** 리더 공백 구간에 제출된 SparkApplication CR은 API 서버가 받아 etcd에 저장되고, 새 리더 선출 후 처리됩니다. 이미 Running 중인 작업은 리더와 무관하게 계속 돕니다.
2. **진짜 함정은 workqueue의 전역 토큰버킷이다.** CR 생성 이벤트조차 `AddRateLimited` 로 큐잉되는데, executor pod churn이 같은 버킷을 공유하기 때문에 이벤트 폭주 시 신규 작업이 10분씩 조용히 밀립니다. 이 지연은 `workqueue_depth` 에 잡히지 않습니다.
3. **Completed CR 방치는 failover 비용을 키운다.** 약 4만 건의 Completed CR이 failover burst에서 각각 최소 1회의 API write를 유발해, 기본 QPS 20 기준 약 33분(40,000 ÷ 20 = 2,000초)의 burst를 만들었습니다.
4. **Operator pod의 CFS throttling이 failover 복구를 20분까지 늘릴 수 있다.** worker 10개가 동시에 spark-submit JVM을 fork하는 순간 CPU limit에 걸리면 전체가 기어갑니다.
5. **Airflow는 리더 교체를 사실상 감지하지 못한다.** 유일한 즉시 실패 경로는 webhook 동반 사망이며, 방어책은 `webhook.replicas: 2+` 와 podAntiAffinity입니다.

## 2. HA 구조와 리더 일렉션

### 2.1 "standby 옵션"은 없다 — standby는 결과다

Spark Operator에 standby라는 별도 설정은 존재하지 않습니다. leader election을 켜고 replica를 2개 이상 띄우면, 리더가 아닌 replica가 자동으로 standby가 되는 구조입니다.

```yaml
controller:
  replicas: 3          # 1 leader + 2 standby
  leaderElection:
    enable: true       # Helm chart 기본값 true (바이너리 기본값은 false)
    leaseDuration: 15s
    renewDeadline: 10s
    retryPeriod: 2s
```

lock은 `{--leader-election-lock-namespace}/{--leader-election-lock-name}` 위치의 **Lease 리소스**(coordination.k8s.io/v1)로 생성됩니다. 코드상 플래그 설명에 "ConfigMap"이라 적혀 있는 것은 v1 시절 문구가 남은 것이고, v2는 controller-runtime 기반이라 실제로는 Lease를 씁니다.

```bash
kubectl -n spark-operator get lease spark-operator-lock -o yaml
# spec.holderIdentity = 현재 리더 pod
```

운영상 주의점 두 가지입니다.

- ServiceAccount에 해당 네임스페이스의 `leases` 에 대한 get/create/update 권한이 없으면 리더 선출 실패로 controller가 기동 중 죽습니다.
- 같은 lock을 바라보는 replica들만 하나의 HA 그룹입니다.

### 2.2 소스 레벨에서 확인한 리더 선출 메커니즘

- **HA 설정 진입점** — `cmd/operator/controller/start.go`
- **reconciler 게이팅** — controller-runtime `manager/internal.go` 의 `OnStartedLeading` → `startLeaderElectionRunnables()`. 리더가 되기 전까지 reconciler는 시작 자체가 안 됩니다.
- **lease 획득** — client-go `leaderelection.go` 의 `acquire()` blocking loop + `tryAcquireOrRenew()` CAS. 만료 판정은 renewTime이 아니라 **관찰자 로컬의 `observedTime` 기준**이라 노드 간 clock skew에 안전합니다.
- **standby는 warm 상태다.** 리더가 아니어도 informer 캐시는 sync를 유지하므로, 리더 승격 시 캐시 재빌드 비용은 없습니다.

두 가지 소스 레벨 발견이 운영에 직접 영향을 줍니다.

1. **`LeaderElectionReleaseOnCancel` 이 주석 처리되어 있다.** 계획된 재시작(rollout restart)도 lease를 자발적으로 반납하지 않아서, crash와 동일하게 ~15초의 lease 만료 대기를 치릅니다. 즉 "무중단 계획 재기동"은 현재 불가능합니다.
2. **healthz/readyz에 `healthz.Ping` 만 등록되어 있다.** 캐시 sync 여부가 readyz에 반영되지 않으므로, readyz로 standby의 warm 상태를 판단할 수 없습니다.

### 2.3 Failover 타임라인

```text
리더 사망
  → lease 만료 대기: leaseDuration ~15s
  → 획득 경쟁: retryPeriod jitter ~2.4s
  → informer 초기 sync: 전체 CR에 Add 이벤트 발생
  → reconcile burst: CR 수 ÷ 유효 처리율
```

reconcile burst가 변수입니다. 앱 100개면 기본 QPS 20 기준 ~10초지만, Completed CR 4만 건이 쌓여 있으면 **33분**이 됩니다(4.3절).

## 3. Failover 시 작업 처리 동작

### 3.1 시나리오별 정리

| 상황 | 동작 | 결과 |
| --- | --- | --- |
| 이미 Running 중인 앱 | driver/executor pod는 리더와 무관하게 계속 실행 | 영향 없음, status 갱신만 일시 정지 |
| 공백 구간에 제출된 앱 | CR은 etcd에 저장, status는 빈 채(New) 방치 | 새 리더가 큐 소화 후 정상 제출 — **지연**만 발생 |
| submit 직후 status 기록 전 리더 사망 | at-least-once 재제출 race | deterministic driver pod 이름으로 `AlreadyExists` 방어, 드물게 FAILED 마킹 |

### 3.2 순서 보장은 없다

새 리더의 workqueue에는 기존 Running 앱과 신규 New 앱이 **구분 없이 한꺼번에** 들어가고, `--controller-threads`(기본 10, `MaxConcurrentReconciles`)개의 worker가 병렬로 처리합니다. 우선순위 큐가 아니므로 순서는 비결정적이고, 같은 key(같은 앱)에 대한 직렬화만 보장됩니다.

다만 실질적으로 신규 앱이 늦어 보이는 효과는 있습니다. Running 앱 reconcile은 status 동기화 수준의 가벼운 작업인 반면, New 앱은 spark-submit(JVM fork) 실행이라 무겁습니다. worker 10개를 놓고 경쟁하면 신규 제출이 수십 초 밀릴 수 있습니다.

!!! tip "실험 포인트"
    앱 수백 개를 걸어두고 리더 pod를 죽인 뒤, 신규 앱의 `creationTimestamp` → `lastSubmissionAttemptTime` 간격을 측정하면 failover 지연을 정량화할 수 있습니다.

## 4. 프로덕션 이슈 3건의 근본 원인

### 4.1 "10분 침묵 후 갑자기 처리" — 전역 토큰버킷

**증상** — 신규 SparkApplication이 제출됐는데 10분간 아무 일도 없다가 갑자기 처리됨. `workqueue_depth` 는 정상.

**원인** — CR 생성(Create) 이벤트조차 `AddRateLimited` 로 큐잉되는데, rate limiter의 토큰버킷(`--workqueue-ratelimiter-bucket-qps` 기본 50 / `bucket-size` 기본 500)이 **컨트롤러 전역 공유**입니다. executor pod가 분당 수백 개씩 churn하면 pod 이벤트가 버킷을 소진하고, 신규 CR은 delaying queue에서 토큰이 찰 때까지 대기합니다.

!!! important "핵심 함정"
    delaying queue에 있는 항목은 `workqueue_depth` 에 **잡히지 않습니다.** 겉보기에 큐는 비어 있는데 처리만 안 되는, 지표상 보이지 않는 지연입니다.

**대응**

```text
--workqueue-ratelimiter-bucket-qps: 200
--workqueue-ratelimiter-bucket-size: 5000
```

### 4.2 "failover 후 20분 정지" — CFS throttling

**증상** — 리더 교체 후 20분간 신규 작업 처리 지연. worker 10개 전부 점유 상태.

**진단 경로** — YuniKorn `STOPPED_BY_RM`(shim→core, K8s 측 pod 소멸이 트리거) → workqueue depth 분석 → worker 포화 확인 → 원인 후보 압축.

**가장 유력한 원인** — failover replay 시 worker 10개가 동시에 spark-submit JVM을 fork하는데, operator pod의 CPU limit에 걸려 CFS throttling으로 전체가 극도로 느려짐. Airflow/Redis에서 이미 겪었던 것과 동일한 패턴입니다.

**진단 지표**

```text
container_cpu_cfs_throttled_periods_total{pod=~"spark-operator.*"}
histogram_quantile(0.99, rate(workqueue_work_duration_seconds_bucket[5m]))
workqueue_longest_running_processor_seconds
controller_runtime_active_workers / max_concurrent_reconciles
```

**대응** — `controller-threads` 증설이 아니라 **CPU limit 제거(requests만 유지)**. worker를 늘리면 동시 fork가 늘어 throttling이 더 심해집니다.

**보조 후보** — client-side API throttling. 로그에서 `Waited for Xs due to client-side throttling` 패턴 확인.

### 4.3 Completed CR 4만 건 — 조용한 부채

**발견** — `updateSparkApplicationStatus` 는 DeepEqual 스킵 없이 **무조건 `Status().Update()` 를 호출**합니다. Completed CR 4만 건이 쌓여 있으면 failover burst에서 각각 최소 1회의 API write가 발생하고, 기본 QPS 20 기준 40,000 ÷ 20 = 2,000초 ≈ **33분**의 burst가 됩니다. 과거 "정체불명의 20분 burst" 관측과 일치합니다.

**파생 영향** — kube-apiserver의 `MountVolume.SetUp failed` 알람과의 연관을 분석했는데, CR 누적은 근본 원인이 아니라 **부차적 증폭 요인**으로 판정했습니다. 정리 전후 정규화 지표(storage_operation 실패 수 ÷ pod 생성률)로 검증 가능합니다.

**대량 삭제 시 부작용**(dev 클러스터 검증 시 확인 항목)

- compaction 전 etcd 사이즈 일시 증가 → 사전 quota 확인, 이후 compaction + 순차 defrag
- Delete 이벤트가 같은 토큰버킷을 오염 → 삭제 속도 제한(초당 5~10건)
- driver pod에 ownerRef가 남아 있으면 GC 캐스케이드 폭풍
- OpenSearch/audit 파이프라인 스파이크 → 삭제 윈도우 동안 알람 silence

## 5. Airflow 관점: failover는 관측 불가능한 이벤트

`SparkKubernetesOperator` 는 operator pod와 직접 통신하지 않습니다. 하는 일은 두 가지뿐입니다 — ① API 서버에 CR 생성, ② CR status(또는 driver pod) 폴링/watch. Airflow 입장에서 operator는 완전히 투명한 존재입니다.

| 상황 | Airflow 감지 시점 | 결과 |
| --- | --- | --- |
| Running 중 failover | 감지 못 함 | 정상 진행 |
| 공백 중 제출 | 감지 못 함 | 지연 후 정상 |
| webhook 동반 사망 | CR 생성 즉시 (admission 거부) | 해당 task만 즉시 실패 |
| 재제출 race로 FAILED 마킹 | 다음 폴링(≤ poke_interval, 기본 60s) | 해당 task만 실패 |
| 큐 소화 장기화 | sensor/execution timeout | timeout 걸린 task 실패 |

**유일한 즉시 감지 경로는 webhook 동반 사망입니다.** webhook은 leader election 대상이 아닌 별도 deployment인데, replica 1개가 controller와 같은 노드에 있다가 노드 장애를 맞으면 `failurePolicy: Fail` 기준으로 CR 생성 자체가 거부됩니다.

**방어책** — Airflow `retries` + `webhook.replicas: 2+` + controller/webhook podAntiAffinity. 이 세 가지면 leader failover는 Airflow에서 사실상 관측 불가능한 이벤트가 됩니다.

## 6. 튜닝 가이드

### 6.1 controller-threads: 노드 규모는 판단 기준이 아니다

worker는 goroutine(`MaxConcurrentReconciles`)이므로 클러스터가 190노드든 10노드든 무관합니다. 기준은 **① 동시 앱 수와 제출 빈도, ② reconcile 1건당 소요 시간(특히 spark-submit), ③ workqueue 대기 시간**입니다.

**판단 공식 (Little's Law)**

```text
필요 worker 수 ≈ 제출률(apps/sec) × 평균 reconcile 시간(sec) + 여유분
```

**증설 신호** — "큐 대기 시간이 실제 작업 시간보다 길다"

```text
# queue_duration ≫ work_duration 지속 → worker 증설
histogram_quantile(0.95, rate(workqueue_queue_duration_seconds_bucket{name=~".*spark.*"}[5m]))
histogram_quantile(0.95, rate(workqueue_work_duration_seconds_bucket{name=~".*spark.*"}[5m]))
workqueue_depth{name=~".*spark.*"}
```

10 → 20 → 30으로 단계 증설하면서 `container_cpu_cfs_throttled_periods_total` 을 같이 봅니다. queue_duration p95가 1초 아래로 내려오는 지점이 적정값입니다.

### 6.2 worker만 늘리면 안 되는 이유 — 같이 조정할 상한 3개

1. **Workqueue rate limiter** — worker 50개여도 재큐 속도가 여기서 잘리면 무의미합니다.
2. **API 클라이언트 QPS** — `-kube-api-qps` / `-kube-api-burst`. worker 증가에 비례해 status update와 pod 조회가 늘고, 여기가 막히면 reconcile 자체가 느려져 오히려 큐가 쌓입니다.
3. **Operator pod 리소스** — spark-submit마다 JVM이 뜨므로 동시 submit 수 × (수백 MB + CPU 스파이크)만큼 필요합니다. limit이 2core/2Gi에 worker 30이면 CFS throttling 재현.

### 6.3 권장 값 요약

| 파라미터 | 기본값 | 권장 | 근거 |
| --- | --- | --- | --- |
| `--kube-api-qps` / `burst` | 20 / 30 | 100 / 200 | failover burst 단축 (4만 CR × QPS 20 = 33분) |
| `--workqueue-ratelimiter-bucket-qps` / `size` | 50 / 500 | 200 / 5000 | executor churn과 버킷 공유 완화 |
| `--controller-threads` | 10 | 실측 후 단계 증설 | queue vs work duration 비교 |
| CPU limit | (환경별) | **제거**, requests만 | CFS throttling 방지 |
| `webhook.replicas` | 1 | 2+ (podAntiAffinity) | 유일한 즉시 실패 경로 차단 |
| `spec.timeToLiveSeconds` | 없음 | 신규 제출분에 설정 | CR 누적 원천 차단 |

## 7. CR 정리 자동화 (CronJob)

`spec.timeToLiveSeconds: 86400` 을 CR에 넣으면 operator가 자동 삭제하지만, Airflow 템플릿 전체 수정이 필요하고 FAILED까지 같이 지워집니다. 제어권 측면에서 CronJob 방식이 낫습니다.

**설계 결정 사항**

- 기준 시각은 `metadata.creationTimestamp` 가 아니라 **`status.terminationTime`** — 장시간 실행 job이 종료 직후 지워지는 것 방지
- bitnami/kubectl 이미지에 jq가 없으므로 `go-template` 출력 사용
- `-wait=false` 로 finalizer 대기 블로킹 방지
- **`DELETE_INTERVAL_SEC` 로 삭제 간 sleep** — Delete 이벤트도 operator의 같은 토큰버킷에 들어가므로 대량 삭제가 신규 앱 처리를 밀어낼 수 있음 (4.1절과 동일 메커니즘)
- `DRY_RUN` 변수로 첫 실행 검증, 매일 02:00 스케줄

핵심 로직(전체 매니페스트는 RBAC 포함 별도 관리):

```yaml
env:
  - name: RETENTION_HOURS
    value: "24"
  - name: TARGET_STATES
    value: "COMPLETED"        # 필요시 "COMPLETED FAILED"
  - name: DRY_RUN
    value: "false"
  - name: DELETE_INTERVAL_SEC
    value: "1"                # operator workqueue 보호
```

## 8. 부록 A — 통신 토폴로지

Operator ↔ YuniKorn 직접 통신은 없습니다. 모든 상호작용은 kube-apiserver 허브를 경유합니다.

```text
Airflow ──(CR 생성/폴링)──▶ kube-apiserver ◀──(watch/reconcile)── Spark Operator
                                 ▲
                                 └──(watch/bind)── YuniKorn
```

이 구조 때문에 리더 공백 중에도 CR 생성이 성공하고(2.3절), YuniKorn placeholder pod(tg- prefix, gracePeriod=0은 YUNIKORN-2771에 따른 YuniKorn 자체 설계) lifecycle churn이 `MountVolume.SetUp failed` 알람을 증폭시킵니다. 이 알람은 watch 기반 비동기 아키텍처의 예상된 race이지 버그가 아닙니다.

## 9. 부록 B — Sidekick sidecar 종료 문제

driver Completed 후 Sidekick sidecar가 안 죽는 문제. `spec.driver.sidecars` 는 pod의 `spec.containers` 로 들어가 main 종료 후에도 K8s가 죽여주지 않습니다.

- **1차 해결** — native sidecar 패턴 — `initContainers` + `restartPolicy: Always` (spark-operator PR #2022, K8s 1.29+ SidecarContainers). driver 종료 시 kubelet이 SIGTERM 자동 전송.
- **최종 아키텍처** — Sidekick을 **standalone DaemonSet** 으로 분리 + Service `internalTrafficPolicy: Local`. 같은 노드의 Sidekick pod로만 라우팅해 data locality를 유지하면서 pod lifecycle 결합을 제거. 상세: [DirectPV · Sidekick 배치 방식](../poc/04-directpv-sidekick.md)

## 10. 부록 C — 진단 쿼리 치트시트

**과거 failover 시각 발굴 (OpenSearch)**

```text
kubernetes.namespace_name:"spark-operator" AND
(log:"successfully acquired lease" OR log:"leader election lost")
```

**client-side throttling 규모 확정 (OpenSearch PPL)** — `Waited for Xs due to client-side throttling` 패턴을 시간당 집계 — 시간당 수백 건이거나 max_wait 수 초면 QPS 부족 확정.

**리더 교체 시각 앵커 (Prometheus)**

```text
leader_election_master_status{name="spark-operator-lock"}
# pod별 0↔1 전환 = 리더 교체 시각
# 이후 increase(controller_runtime_reconcile_total[...])이 앱 수 도달까지 = 실측 burst 시간
```

**API 사용률 clipping 확인**

```text
max_over_time(sum(rate(rest_client_requests_total{namespace="spark-operator"}[5m]))[30d:5m])
# 피크가 20 근처에서 평평하게 잘려 있으면 throttling 시각적 증거
```

!!! note
    전제 — operator metrics endpoint(`--metrics-port` 8080)가 PrometheusAgent scrape 대상에 포함되어 있어야 `workqueue_*`, `rest_client_*` 가 수집됩니다.

## 11. 결론

Spark Operator HA 운영에서 배운 것을 한 문장으로 압축하면 — **failover를 두려워할 게 아니라, failover가 드러내는 평상시의 부채(CR 누적, 토큰버킷 공유, CPU limit)를 관리해야 한다.** 리더 교체는 잘 설계된 30초짜리 이벤트고, 20~33분짜리 장애로 만든 건 전부 운영 상태였습니다.

**액션 아이템 체크리스트**

- [ ] Completed CR 정리 CronJob 배포 (dry-run → 삭제 속도 제한)
- [ ] 신규 제출 템플릿에 `timeToLiveSeconds` 추가
- [ ] operator CPU limit 제거, requests 상향
- [ ] `-kube-api-qps/burst`, workqueue 버킷 상향
- [ ] `webhook.replicas: 2` + podAntiAffinity
- [ ] failover 리허설로 `creationTimestamp → lastSubmissionAttemptTime` 실측
