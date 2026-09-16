---
title: 04. YuniKorn Gang Scheduling과 conntrack/CNI 부하
---

# 🧩 스케줄러 — YuniKorn Gang Scheduling과 conntrack/CNI 부하

!!! abstract "요약"
    YuniKorn의 Placeholder Pod 생명주기가 Cilium conntrack 맵 포화와 Airflow / Redis 타임아웃까지 이어지는 인과관계를 추적한 기록.

## 문제의 발단

분당 수십~수백 개의 pod가 뜨고 사라지는 **고churn 워크로드**(gang scheduling의 taskgroup placeholder 포함) 환경에서 네트워크 드랍이 발생했습니다. `CT: Map insertion failed` 에러가 확인됐습니다.

## 인과관계 체인

```text
YuniKorn gang scheduling → placeholder(pause) pod 대량 생성/삭제
  → Cilium conntrack(CT) 맵 포화
  → CT insertion 실패 → 패킷 드랍
  → Airflow가 쓰는 git-sync, Redis 컴포넌트에서 간헐적 타임아웃
```

핵심 발견 하나 — **placeholder pod의 네트워크 설정은 실제 pod spec과 비대칭으로 따로 수정할 수 없습니다.** YuniKorn의 gang swap 제약 때문입니다. 즉 placeholder만 골라서 완화 설정을 넣는 방법은 없습니다.

## 근본 원인 — `terminationGracePeriodSeconds: 0`

**Executor(taskgroup pod)의 grace period가 0으로 설정되어 있었던 것**이 가장 직접적인 원인 후보였습니다. grace period 0은 SIGTERM 직후 곧바로 SIGKILL이 오는 것과 같아서, graceful shutdown이나 decommission 시간이 전혀 없습니다.

Spark dynamic allocation에서 executor를 줄일 때 `spark.decommission.enabled` 가 **기본 false** 라는 점이 중요한 단서였습니다 — 켜지 않으면 SIGPWR 기반 우아한 마이그레이션 없이 executor가 그냥 죽습니다. 이 경우 driver와의 RPC / shuffle fetch 연결이 **FIN 교환 없이 끊기고, 그대로 conntrack 누수(stale entry)** 가 됩니다.

decommission을 켜더라도 shuffle 마이그레이션이 grace period(기본 30초) 안에 안 끝나면 결국 SIGKILL(137)로 강제 종료되어 의미가 사라집니다.

!!! tip "판별 지점"
    driver 로그의 다음 한 줄:
    ```text
    Remove reason statistics: (gracefully decommissioned: N, unexpectedly exited: N)
    ```
    `unexpectedly exited` 가 높으면 **conntrack 누수의 직접 증거**입니다.

## Placeholder Pod와 Volume Mount 레이스 (YUNIKORN-2771)

별개로 확인된 사항입니다. placeholder pod의 `gracePeriod=0` 은 **YuniKorn 자체의 설계**(YUNIKORN-2771)이지 사용자가 켠 설정이 아닙니다. 따라서 `terminationGracePeriodSeconds` 튜닝으로 **사용자 측에서 완화할 수 없습니다.**

처음엔 사용자 설정으로 오판했다가 소스 / JIRA 확인 후 정정한 부분입니다.

## 조치 우선순위

1. **Spark 측** — `spark.decommission.enabled` + `spark.storage.decommission.*` 계열을 명시적으로 켜고, executor grace period를 마이그레이션이 끝날 만큼 확보 (SparkApplication CR의 executor spec에 preStop hook 포함)
2. **NetworkPolicy** — conntrack 엔트리에 정책 집행 상태를 묶어두므로, cluster-wide NetworkPolicy를 쓰는 경우 문제가 배가된다는 점 인지
3. **CT GC 튜닝** — GC 부작용(CPU 스파이크, CNI 삭제 지연 최대 10초+, 활성 연결까지 회수될 위험)도 함께 고려

## 결론

conntrack 누수는 **네트워크 계층(Cilium) 증상으로 보이지만, 근본 원인은 워크로드 계층(Spark decommission 미설정 + grace period 0)** 에 있는 전형적 사례입니다.

인프라 증상과 서비스 원인이 분리되는 이 패턴이 반복적으로 나타납니다 — [Spark × YuniKorn 작업 지연](05-spark-yunikorn-delay.md) 문서와 함께 볼 것.
