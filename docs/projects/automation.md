---
title: 배포 자동화 · 커스텀 개발
---

# 배포 자동화, 커스텀 개발

| 항목 | 내용 |
| --- | --- |
| 기간 | 2026 |
| 역할 | 오케스트레이션 체계 구축, 런타임 이미지 개발, 운영 자동화 |
| 핵심 기술 | Airflow, Spark 3.5, Iceberg, Spark Operator, Prometheus |

## Airflow 오케스트레이션 체계 구축

- Airflow **커스텀 이미지 / Helm 차트** 구성
- **DataOps 공용 DAG 파이프라인 개발**
    - Kafka → Iceberg 적재
    - Spark Operator 연동
    - 메트릭 컨슈머
    - DB / logfile 클린업

## DataLake Spark Runtime 이미지 개발

- **Spark 3.5.7 + Iceberg 카탈로그 + 메트릭 + Spark Connect**
- 서비스별 Spark Application(CR) **배포 템플릿** 작성

## 운영 자동화

- Spark WorkDir 정리 **CronJob**
- 팀별 ServiceAccount 기반 **Kubeconfig 생성**
- **Spark Operator 모니터링** — Prometheus 메트릭, 알람 규칙 개발. 소스 레벨 운영 분석: [Kubeflow Spark Operator 운영 분석](../work/verification/08-kubeflow-spark-operator-report.md)
