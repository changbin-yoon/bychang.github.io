---
title: Storage · AIStor / MinIO
---

# 🧪 [Storage] AIStor / MinIO PoC · 설정 및 테스트 기록

AIStor / MinIO 도입 및 검증 과정에서 수행한 설정·테스트·트러블슈팅 기록을 유형별로 정리한 문서 모음입니다.

!!! info "대상 환경"
    AIStor(MinIO 상용 포크), 베어메탈 Kubernetes, 에어갭

**정리 기준** — 각 문서는 목적 → 구성·사용법 → 테스트 시나리오 → 결과·관찰 → 결론 순서

## 문서 목록

| # | 유형 | 다루는 내용 |
| --- | --- | --- |
| [01](01-ldap-iam.md) | 인증 / IAM | LDAP 연동 Access Key 발급과 권한 전파 검증 |
| [02](02-capacity.md) | 용량 / 성능 | 고사용률 구간 동작과 진단 지표 |
| [03](03-network-l4-l7.md) | 네트워크 | 트래픽 분산 경로 검증 (L4 vs L7) |
| [04](04-kafka-events.md) | 이벤트 / 파이프라인 | Kafka 알림 설정과 적재 구현 |
| [05](05-directpv-sidekick.md) | 구성 컴포넌트 | DirectPV · Sidekick 배치 방식 |
| [06](06-opensearch-snapshot.md) | 연동 시스템 | OpenSearch 스냅샷 리포지토리 운영 |
| [07](07-query-engine.md) | 쿼리 엔진 | StarRocks / Spark · Iceberg · HMS 연동 |
| [08](08-directpv-erasure-set.md) | 용량 / 스토리지 | DirectPV 드라이브 공유로 인한 Erasure Set 쓰기 차단 |

## 공통 전제

- 모든 명령은 `mc` 클라이언트 기준이며, alias는 `<alias>`로 표기합니다.
- **에어갭 환경**이므로 외부 이미지·패키지는 내부 미러를 경유하고, 엔드포인트는 내부 DNS / ClusterIP를 사용합니다.
- upstream MinIO와 AIStor의 동작이 갈리는 지점은 각 문서에 별도로 표시했습니다.
