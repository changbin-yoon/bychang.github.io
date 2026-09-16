---
title: 홈
---

# 윤창빈 · Data Platform Infra Engineer

> 온프레미스 하둡 빅데이터 클러스터의 구축·운영 경험을 바탕으로,
> **Kubernetes 기반 차세대 데이터 플랫폼(Lakehouse / DataOps) 전환**을 수행하고 있는 인프라 엔지니어입니다.

스토리지, 쿼리 엔진, 메타데이터를 아우르는 인프라 전반을 직접 구축·검증했고, ArgoCD 기반 GitOps와 Airflow / Spark Operator를 통한 배포·운영 자동화 체계를 확립했습니다. 반도체 제조(Fab) 데이터의 성능·가용성 확보를 위한 성능 테스트와 장애 대응이 강점입니다.

## 핵심 요약

- **T-Hadoop(온프레미스 빅데이터) → K8s 기반 Lakehouse / DataOps 전환** 프로젝트에 참여해 스토리지, 쿼리 엔진, 메타데이터, 배포 자동화 전반을 담당
- **Hadoop 클러스터 수백 대 운영·서버 교체·장애 대응**, 신규 K8s 데이터플랫폼(dev/stg/prd/storage) Kubespray 구축 및 **ArgoCD 기반 GitOps 배포 체계 확립**
- Storage(Isilon / Ceph / MinIO·AIStor), 쿼리 엔진(Trino / Spark / StarRocks), 메타스토어(HMS / Polaris), Kafka 등 **비교·도입 평가(PoC)와 성능 테스트 다수 수행**

## 둘러보기

<div class="grid cards" markdown>

-   :material-account-circle: **[소개](about.md)**

    ---

    프로필, 학력, 보유 기술 상세.

-   :material-briefcase: **[경력](experience.md)**

    ---

    미리비트(2020.04~현재), 커미조아(2017.05~2019.10).

-   :material-folder-multiple: **[프로젝트](projects/index.md)**

    ---

    대표 프로젝트 7건의 배경·역할·결과.

-   :material-alert-decagram: **[이슈 해결](troubleshooting.md)**

    ---

    장애·성능 이슈의 원인 규명과 조치 사례.

</div>

## 직무 요약

| 영역 | 주요 기술 |
| --- | --- |
| 인프라 / 클러스터 | Kubernetes(Kubespray, RKE2), Cilium, Helm, Kustomize, ArgoCD(GitOps), Linux(RHEL) |
| 데이터 플랫폼 | Trino, Apache Spark, Hive/HMS, StarRocks, Apache Ignite, Kafka, Airflow |
| 스토리지 | Ceph(Rook, Baremetal), Isilon(CSI-NFS), MinIO/AIStor(DirectPV), HDFS, S3A |
| 데이터 카탈로그 / 포맷 | Iceberg, HMS, Gravitino, Polaris, ORC, Parquet |
| 보안 / 인가 | OPA/Gatekeeper, LDAP, Sealed Secrets, Hadoop jceks |
| 모니터링 | Prometheus, Grafana, JMX, OpenSearch, Zabbix, Metatron |
| 언어 / 도구 | Shell, Python, SQL, Ansible, fio / JMeter / Locust |
