---
title: 메타데이터(HMS) 관리·모니터링
---

# 메타데이터(HMS) 관리, 모니터링

| 항목 | 내용 |
| --- | --- |
| 기간 | 2025 ~ 2026 |
| 역할 | HMS 구축·운영, 감사 체계 개발, 장애 대응 |
| 핵심 기술 | Hive Metastore, Oracle DB, Iceberg, OpenSearch, Grafana |

## Hive Metastore 구축·운영

- **Oracle DB 기반** HMS 구축 및 운영 — 버전 검증 보고서: [Oracle 21c/26ai HMS 스키마 검증](../work/verification/10-oracle-hms-schema.md)
- **Hive-Metastore Hook** 적용
- AIStor 연동

## 감사(Audit) 대시보드 개발

**OpenSearch 로그 기반 Grafana 대시보드**를 개발해 메타데이터 변경을 추적할 수 있게 했습니다.

- DB / EventType별 **CRUD 이벤트 모니터링**
- 위험 이벤트 하이라이트

## Iceberg 메타데이터 정리

정리 스크립트를 운영해 메타데이터 증가를 관리했습니다.

```text
rewrite_data_files
rewrite_manifests
expire_snapshots
```

## 장애 대응

- **HMS Oracle DB 장애 대응 (2026.05)** 및 장애 보고서 작성 — 상세: [업무내역 요약](../work/index.md)
