---
title: 02. Spark × YuniKorn 작업 지연 진단
---

# ⏱️ Spark × YuniKorn 작업 지연 진단 — SUBMITTED→RUNNING

!!! abstract "요약"
    Airflow 로그에 반복적으로 뜨는 "Spark job submitted but not yet started" 경고를 추적해 SparkApplication 생명주기 전체를 매핑한 기록.

## 경고의 정체

`airflow.providers.cncf.kubernetes.operators.custom_object_launcher.CustomObjectLauncher` 가 SparkApplication CR이 SUBMITTED에서 RUNNING으로 넘어가기 전까지 폴링하며 찍는 **정상 동작**이었습니다(13초 / 10초 주기).

Spark Operator의 `workqueueRateLimiter`(bucketQPS 50, bucketSize 500) 설정은 해당 환경에 적절해서 원인이 아니라고 판단했습니다.

## SparkApplication 생명주기 구간별 분해

Driver Pod 기동에 실측 약 **26초**가 걸렸고, 전체 지연의 주범은 다음 두 구간으로 판명됐습니다.

1. **YuniKorn 스케줄링 구간** — gang scheduling이 taskGroup(minMember) 충족을 기다리는 시간
2. **컨테이너 이미지 pull 구간** — 이미지 크기 / 레지스트리 위치에 직접 영향

## YuniKorn Admission Controller vs Scheduler

둘은 역할이 완전히 다른 컴포넌트입니다.

| 컴포넌트 | 역할 |
| --- | --- |
| Admission Controller | Mutating / Validating Webhook — schedulerName 주입, 큐 검증, annotation 보강 |
| Scheduler | 실제 리소스 할당 & 노드 bind |

Admission Controller가 없으면 pod에 YuniKorn schedulerName이 주입되지 않아 **기본 스케줄러가 사용되거나 큐 검증이 생략**됩니다.

## YuniKorn 스케줄러 스케일링에 대한 오해

YuniKorn 스케줄러를 2 인스턴스로 늘리는 건 성능 향상이 아닙니다 — **싱글 프로세스 인메모리 상태 아키텍처**라 지원되지 않습니다. **Active-Standby HA**가 유일한 다중 replica 사용 케이스입니다.

## 결론

"submitted but not started" 경고는 대부분 버그가 아니라 **gang scheduling 대기와 이미지 pull 시간이 정상적으로 반영된 것**입니다.

경고가 비정상적으로 길어진다면 이 두 구간이 아닌 다른 원인(리소스 부족, 큐 설정)을 의심하고, `/ws/v1/queues`, `/ws/v1/apps` REST 엔드포인트로 큐 상태를 직접 확인해야 합니다.
