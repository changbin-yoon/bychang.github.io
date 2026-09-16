---
title: 트러블슈팅
---

# 트러블슈팅 · 조사 기록

운영·검증 과정에서 조사하고 해결한 내용을 영역별로 정리한 문서 모음입니다.
각 문서는 **배경/질문 → 조사·검증 내용 → 결론·조치** 순서로 작성했습니다.

<div class="grid cards" markdown>

-   :material-kubernetes: **[Kubernetes Infra](kubernetes-infra/index.md)**

    ---

    클러스터·네트워크·스케줄러 계층. Cilium BGP/ClusterMesh, YuniKorn, API 서버 부하 등 10건.

-   :material-cog-outline: **[Kubernetes Service](kubernetes-service/index.md)**

    ---

    워크로드 계층. Spark Operator HA, Airflow 3.x, CNPG, n8n 자동화 등 5건.

-   :material-database: **[Storage · AIStor / MinIO](storage-aistor/index.md)**

    ---

    객체 스토리지 PoC. IAM, 용량/성능, 이벤트 파이프라인, DirectPV 등 8건.

-   :material-file-chart: **[검증 보고서](reports/index.md)**

    ---

    카탈로그·메타스토어 비교 검증, Spark Operator 운영 분석 보고서.

</div>

## 운영 이슈 요약

주요 장애·성능 이슈의 원인과 조치는 [이슈 해결 요약](summary.md)에 정리했습니다.
