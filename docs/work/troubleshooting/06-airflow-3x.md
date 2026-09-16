---
title: 03. Airflow 3.x 운영
---

# 🌬️ Airflow 3.x 운영 — dag-processor, git-sync, Celery/Redis

!!! abstract "요약"
    Airflow 3.0.6, Helm chart, NFS 로그, PostgreSQL 메타DB, Redis Celery broker 구성에서 여러 차례에 걸쳐 다룬 운영 이슈 모음.

## dag-processor 파드 재시작 원인 7가지

liveness probe(`airflow jobs check --job-type DagProcessorJob --hostname $(hostname)`)가 메타DB를 조회해 판단하므로, **경보는 같지만 책임 소재가 다른** 7가지 원인을 분리했습니다.

| # | 원인 | 판별 포인트 |
| --- | --- | --- |
| 1 | Liveness probe 오작동(hostname 불일치 등) | 정확히 주기적(예: 5분)으로 restart |
| 2 | OOMKilled | `Last State: Reason: OOMKilled` |
| 3 | DAG 파싱 hang / 타임아웃 (1과 맞물려 루프) | probe 임계치 초과 |
| 4 | 메타DB 커넥션 문제(풀 고갈 등) | 로그에 OperationalError / pool / timeout |
| 5 | DAG 소스 접근 실패(git-sync / PVC) | gitsync 컨테이너 로그 에러 |
| 6 | 부팅 단계 실패(CrashLoopBackOff) | RESTARTS 급증 + 부팅 로그 에러 |
| 7 | 노드 레벨 이슈(eviction / drain) | pod 재생성, events에서 구분 |

1 · 3 · 4번은 증상(probe 실패 → restart)은 같지만 **근본 원인 위치가 다릅니다**.

- **1번은 probe 자체 오작동** — standalone dag-processor에서 probe가 DagProcessorJob이 아니라 SchedulerJob을 체크하는 상속 버그 이력이 있었습니다.
- **3번은 DAG 코드 상태 자체**
- **4번은 DB 상태**

즉 probe는 방아쇠일 뿐입니다.

```bash
kubectl describe pod -n airflow <dag-processor-pod>   # Last State Reason, Events
kubectl logs -n airflow <dag-processor-pod> --previous
kubectl get events -n airflow --sort-by=.lastTimestamp | grep dag-processor
```

## git-sync 재시작과 Redis의 관계

!!! success "결론: 직접 관계 없다"
    Redis는 CeleryExecutor 전용 브로커(태스크 큐)이고 git-sync는 DAG 동기화 사이드카라, **Executor 종류와 무관하게** 동작합니다.

간접 경로는 하나뿐입니다.

```text
Redis 지연 → Worker Pod 리소스 압박 → 노드 메모리 압박 → kubelet이 사이드카 강제 종료
```

시각 / 노드가 일치해도 대개 "공통 노드 리소스 이슈"가 진짜 원인이지, Redis가 git-sync를 직접 재시작시키는 경로는 없습니다.

## Celery Worker Redis ping 타임아웃과 CFS Throttling

Celery 컨트롤 메시지는 Redis를 거쳐 왕복하므로, 타임아웃은 **worker 과부하 / Redis 지연 / 느려진 브로커 연결** 어느 쪽에서도 발생할 수 있습니다.

리소스 request 이내인데도 타임아웃이 나는 핵심 원인은 **CFS(Completely Fair Scheduler) throttling** 입니다. Linux는 CPU limit을 100ms 윈도우(quota / period) 단위로 강제하므로, 컨테이너가 윈도우 중 할당량을 다 쓰면 **노드 CPU가 놀고 있어도 강제로 멈춥니다.** 평균 지표에는 안 잡히는 sub-second 정지지만 ping 타임아웃을 유발하기에는 충분합니다.

```text
진단: /sys/fs/cgroup/cpu.stat
      container_cpu_cfs_throttled_periods_total
```

## CeleryExecutor vs KubernetesExecutor 구성 비교

| 구성요소 | KubernetesExecutor | CeleryExecutor |
| --- | --- | --- |
| Redis | 불필요 | 필수(브로커) |
| git-sync | 필요 | 필요(모든 Worker Pod에 사이드카) |
| Worker Pod | 태스크마다 동적 생성 | 상시 실행 |

## NFS 로그 저장 & Postgres 이력 테이블

NFS PVC는 Airflow에게 **로컬 POSIX 파일시스템으로 취급**되므로, 원격 로깅이 아니라 `AIRFLOW__LOGGING__BASE_LOG_FOLDER` 로 설정하는 것이 정답입니다.

`logs.persistence.enabled: true` 가 이미 모든 컴포넌트(api-server 포함)에 PVC를 자동 마운트하므로, 수동 `extraVolumeMounts` 와 겹치면 `mountPath must be unique` 에러가 납니다.

이전 시도(try_number) 로그는 Airflow 2.6+ / 3.x에서 **`task_instance_history` 테이블로 이동**하므로, 과거 시도 조회 시 이 테이블을 봐야 합니다.

## 결론

dag-processor / git-sync / Celery worker는 증상이 닮아도 **근본 원인 계층이 제각각**입니다(probe / DB / 사이드카 / 리소스 압박). 재시작 주기와 `Last State Reason` 을 먼저 보고 구분하는 것이 진단의 출발점입니다.
