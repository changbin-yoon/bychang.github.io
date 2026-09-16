---
title: Query Engine 운영 및 성능 검증
---

# Query Engine 운영 및 성능 검증

| 항목 | 내용 |
| --- | --- |
| 기간 | 2024 ~ 2026 |
| 역할 | Trino 클러스터 구축·운영, 신규 엔진 도입 평가, 성능 테스트 설계 |
| 핵심 기술 | Trino, StarRocks, Apache Ignite, JMeter, Locust |

## Trino 클러스터 구축·운영

- **서비스별 Trino Cluster 배포**
- Hive / Iceberg / PostgreSQL **카탈로그 연동**
- **LDAP 인증 + OPA 인가 + Group-Provider** 구성
- Kafka listener, jmx-exporter, ServiceMonitor, **Cilium Ingress** 적용

## 버전 고도화 및 이슈 대응

- Trino 버전 고도화 — **408 → 442 → 475 → 482**
- 지연 원인 분석, 카탈로그 / 리소스 튜닝

## 신규 엔진 도입 검토

- **StarRocks** 도입 검토 및 성능 평가 — kube-starrocks operator 테스트
- **Apache Ignite (V2 / V3)** 클러스터 PoC — K8s discovery, cluster-init, StatefulSet

## 성능 테스트

- **JMeter** Query 성능 테스트 (21개 쿼리)
- **Locust** 부하 테스트 (Trino / Spark Operator)
- **HDFS vs MinIO vs Ceph** 비교 테스트
