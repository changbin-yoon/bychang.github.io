---
title: Kubernetes Service
---

# ⚙️ [Kubernetes Service] 워크로드 운영 기록

Kubernetes 위에서 도는 데이터 플랫폼 워크로드 — Spark Operator, YuniKorn 연동, Airflow, CNPG, n8n — 관련 질문과 조사 내용을 유형별로 정리한 문서 모음입니다. 클러스터 자체(네트워크·스케줄러·API 서버 등) 관점은 [Kubernetes Infra](../kubernetes-infra/index.md) 문서로 분리했습니다.

!!! info "대상 환경"
    Kubeflow Spark Operator v2, YuniKorn Gang Scheduler, Airflow 3.0.6, CloudNativePG, n8n(클러스터 외부 Docker 배포)

**정리 기준** — 각 문서는 배경/질문 → 조사·소스 분석 내용 → 결론·조치 순서

## 문서 목록

| # | 유형 | 다루는 내용 |
| --- | --- | --- |
| [01](01-spark-operator-ha.md) | Spark Operator HA & 리더 일렉션 | Failover 타임라인, 토큰 버킷 rate limiter, 완료 CR 적체 |
| [02](02-spark-yunikorn-delay.md) | Spark × YuniKorn 작업 지연 진단 | SUBMITTED→RUNNING 지연, gang scheduling 상호작용 |
| [03](03-airflow-3x.md) | Airflow 3.x 운영 | 로그 타임존, DAG 동기화, probe 튜닝, Celery/Redis |
| [04](04-cnpg-cilium.md) | CNPG & Cilium 업그레이드 영향 | Liveness probe isolation check, fencing |
| [05](05-n8n-monitoring.md) | n8n 기반 K8s 모니터링 자동화 | 멀티 에이전트 헬스 리포트 워크플로우 |

## 공통 전제

- **Airflow(SparkKubernetesOperator) → Spark Operator v2 → YuniKorn** 순으로 이어지는 배치 파이프라인이 기본 구조입니다.
- 워크로드의 약 **60%가 Spark, 30%가 Trino** 입니다.
- 각 문서의 수치·버전은 조사 시점 기준입니다.
