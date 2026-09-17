---
title: 업무내역 요약
---

# 업무내역 요약

데이터 플랫폼 인프라를 운영하며 남긴 조사·검증 기록 **24건**을 성격에 따라 세 갈래로 분류했습니다.

| 분류 | 건수 | 성격 |
| --- | --- | --- |
| **[트러블슈팅](#트러블슈팅-9건)** | 9건 | 실제 발생한 장애·증상의 원인을 규명하고 조치한 기록 |
| **[PoC](#poc-5건)** | 5건 | 신규 도입·구성 방식을 결정하기 위해 직접 구축하고 테스트한 기록 |
| **[검증](#검증-10건)** | 10건 | 동작 원리·성능·버전 호환성을 사전에 확인하고 근거를 남긴 기록 |

!!! info "대상 환경"
    베어메탈 Kubernetes 약 190노드(마스터 5 + 워커 ~185, 노드당 96코어 / 1TB), **에어갭** 환경.
    Kubespray, Cilium CNI(BGP Control Plane + ClusterMesh), YuniKorn Gang Scheduler, Kubeflow Spark Operator v2, Airflow 3.0.6, CloudNativePG, AIStor(MinIO), Trino, Iceberg.
    워크로드 비중은 **Spark 약 60%, Trino 약 30%**.

---

## 트러블슈팅 (9건)

증상이 먼저 발생하고 원인을 거슬러 올라간 기록입니다. 공통적으로 드러난 패턴은 **인프라 계층에서 관측된 증상의 근본 원인이 워크로드 계층에 있는 경우가 많다**는 것이었습니다.

| # | 문서 | 증상 → 원인 |
| --- | --- | --- |
| 01 | [YuniKorn Gang Scheduling과 conntrack 부하](troubleshooting/01-yunikorn-conntrack.md) | 네트워크 드랍 → Spark decommission 미설정 + grace period 0으로 인한 conntrack 누수 |
| 02 | [API 서버 부하 & Watch 레이스](troubleshooting/02-apiserver-watch-race.md) | Volume Mount 실패 알람 → watch 기반 비동기 구조가 내장한 레이스(버그 아님), 알람 룰로 관리 |
| 03 | [ClusterMesh 신규 PodCIDR 라우팅 누락](troubleshooting/03-clustermesh-podcidr.md) | 간헐적 `No route to host` → 신규 대역 prefix가 일부 노드에 미전파 |
| 04 | [190노드 중 3노드 PodCIDR 경로 누락](troubleshooting/04-podcidr-missing-routes.md) | 특정 3노드만 실패 → **원인 미확정**, 가설과 공개 사례 정리 |
| 05 | [Spark × YuniKorn 작업 지연](troubleshooting/05-spark-yunikorn-delay.md) | "submitted but not started" 경고 → gang scheduling 대기 + 이미지 pull의 정상 반영 |
| 06 | [Airflow 3.x 운영](troubleshooting/06-airflow-3x.md) | dag-processor 반복 재시작 → 증상은 같지만 원인 계층이 7가지로 갈림 |
| 07 | [CNPG & Cilium 업그레이드 영향](troubleshooting/07-cnpg-cilium.md) | CNPG 파드 재시작 → CNI 롤링 중 isolation probe 실패, fencing으로 대응 |
| 08 | [OpenSearch 스냅샷 리포지토리](troubleshooting/08-opensearch-snapshot.md) | 스냅샷 용량 미감소 → 보존 정책 부재 + 증분 구조 + 버킷 versioning |
| 09 | [DirectPV Erasure Set 쓰기 차단](troubleshooting/09-directpv-erasure-set.md) | 일부 PUT만 HTTP 507 → 드라이브 공유로 인한 용량 회계 불일치 |

### 이 분류에서 얻은 것

- **인프라 증상 ≠ 인프라 원인** — conntrack 포화(01), 507 쓰기 차단(09)은 모두 네트워크·스토리지 계층 증상이었지만 근본 원인은 워크로드 설정과 볼륨 배치에 있었습니다.
- **"간헐적"은 대개 무작위가 아니다** — 03·09는 각각 신규 대역 파드, 객체 키 해시에 대해 **결정론적으로** 실패하고 있었습니다. 대상을 고정해 테스트하는 것이 원인 규명의 첫 단계였습니다.
- **알람은 없애는 게 아니라 구분하는 것** — 02는 레이스 자체를 제거할 수 없어, "죽을 리소스의 잔여 참조"와 "살아있는 워크로드의 수렴 실패"를 가르는 알람 룰로 정리했습니다.
- 원인을 확정하지 못한 건(04)은 **미확정임을 명시**하고 검증에 필요한 자료 목록을 남겼습니다.

### 그 외 운영 이슈 대응

별도 문서로 남기지 않은 상시 운영 이슈입니다.

| 이슈 | 조치 |
| --- | --- |
| Trino 쿼리 지연 | 카탈로그 이미지 / 리소스 변경, HMS 로그와 트래픽 분석으로 원인 규명 후 튜닝 |
| HMS Oracle DB 장애 *(2026.05)* | 장애 원인 분석·복구 및 장애 보고서 작성 |
| Airflow 작업 지연 | Spark Operator / YuniKorn 스케줄러 대기 상태 점검 후 조치 |
| Spark 작업 지연 | Spark Operator Controller Down 확인 → 컨트롤러 가용성 개선 |

---

## PoC (5건)

도입 여부나 구성 방식을 정하기 위해 직접 띄워보고 비교한 기록입니다. 결론이 **"채택/제외"로 끝나는 것**이 트러블슈팅과 다른 점입니다.

| # | 문서 | 검토 대상 → 결정 |
| --- | --- | --- |
| 01 | [클러스터 구축·운영 도구](poc/01-cluster-tooling.md) | AWX / Rundeck / Terraform / Semaphore 비교 → **AWX 채택**, Node Tuning Operator와 역할 분리 |
| 02 | [n8n 기반 모니터링 자동화](poc/02-n8n-monitoring.md) | 멀티 에이전트 헬스 리포트 구축 → 숫자는 스크립트, 서술만 AI |
| 03 | [Kafka 이벤트 파이프라인](poc/03-kafka-events.md) | 버킷 이벤트 → Kafka → S3 적재 구현, 오프셋 범위 멱등 키로 exactly-once |
| 04 | [DirectPV · Sidekick 배치 방식](poc/04-directpv-sidekick.md) | 사이드카 vs DaemonSet → **독립 DaemonSet + `internalTrafficPolicy: Local` 채택** |
| 05 | [StarRocks / Spark · Iceberg · HMS 연동](poc/05-query-engine.md) | 쿼리 엔진 연동 구성 확립, 커밋 원자성 동작 확인 |

### 이 분류에서 얻은 것

- **도구는 계층이 다르면 경쟁 관계가 아니다** — 01에서 AWX(변경을 가하는 도구)와 Node Tuning Operator(변경을 유지하는 도구)는 택일 대상이 아니라 둘 다 필요하다는 결론에 도달했습니다.
- **클러스터를 관리하는 도구를 클러스터 안에 두지 않는다** — 순환 의존성을 피하기 위해 standalone Ansible 마스터를 break-glass로 유지했습니다.
- **AI의 역할을 의도적으로 좁힌다** — 02에서 숫자는 Prometheus·Code 노드가 책임지고 AI는 서술만 맡겨 hallucination을 구조적으로 차단했습니다.
- **pod 생명주기 결합을 끊는다** — 04에서 사이드카를 DaemonSet으로 분리해 잔류 문제와 데이터 로컬리티를 동시에 해결했습니다.

---

## 검증 (10건)

장애가 나기 전에, 혹은 도입 결정의 근거를 만들기 위해 **동작 원리와 수치를 확인**한 기록입니다. 소스 코드와 커널 레벨까지 확인한 건들이 여기 속합니다.

| # | 문서 | 검증한 것 |
| --- | --- | --- |
| 01 | [Cilium BGP Control Plane & ClusterMesh](verification/01-cilium-bgp-clustermesh.md) | BGP(데이터 플레인)와 ClusterMesh(컨트롤 플레인) 분리, 6계층 검증 체크리스트 |
| 02 | [Cilium 1.18 → 1.19 무중단 업그레이드](verification/02-cilium-upgrade.md) | breaking change 3건 사전 식별, 카나리 우선 롤링 절차 수립 |
| 03 | [Prometheus Agent 모드](verification/03-prometheus.md) | 로컬 TSDB 부재가 admin API·규칙 평가·메트릭 삭제에 미치는 영향 |
| 04 | [Spark Operator HA & 리더 일렉션](verification/04-spark-operator-ha.md) | 소스 레벨 failover 분석, `LeaderElectionReleaseOnCancel` 비활성 확인 |
| 05 | [LDAP 연동 Access Key 권한 전파](verification/05-ldap-iam.md) | 노드별 in-memory IAM 캐시(실효 5~15분)로 평가됨을 소스로 확인 |
| 06 | [고사용률 구간 동작](verification/06-capacity.md) | 90% 스로틀링은 **없음**, 임계값에서 507 하드 거부임을 실측 |
| 07 | [트래픽 분산 경로 (L4 vs L7)](verification/07-network-l4-l7.md) | ECMP는 연결 단위 해싱 — 소수의 무거운 연결은 불균형이 정상 |
| 08 | [Kubeflow Spark Operator 운영 분석 보고서](verification/08-kubeflow-spark-operator-report.md) | HA·workqueue 토큰버킷·CFS throttling 종합 분석 및 튜닝 가이드 |
| 09 | [Oracle 21c XE / 26ai Free HMS 스키마 검증](verification/09-oracle-hms-schema.md) | 두 버전에서 74개 테이블 구조 **0건 차이** 확인 |
| 10 | [HMS(Oracle) vs Apache Polaris(REST) 비교](verification/10-hms-vs-polaris.md) | 동일 데이터 1천만 건으로 성능·기능·운영 특성 비교 |

### 이 분류에서 얻은 것

- **문서의 서술보다 소스와 커널이 정확하다** — 04·05·08은 공식 문서에 없거나 실제와 다른 동작(플래그 설명의 ConfigMap 표기, 리프레시 주기 상수, 무조건 Status Update)을 소스에서 직접 확인했습니다.
- **업그레이드는 실행보다 사전 식별이 일이다** — 02에서 breaking change 3건(정책 기본값 변경, BGP v1 CRD 제거, 롤링 중 권한 이슈)을 미리 찾아낸 덕에 190노드 롤링을 무중단으로 넘길 수 있었습니다.
- **"없음"을 확인하는 것도 결과다** — 06에서 90% 스로틀링이 존재하지 않음을 확인한 덕에, 성능 저하의 원인을 XFS 단편화·scanner·heal 쪽으로 옮겨 찾을 수 있었습니다.
- **비교 검증은 변인을 통제해야 의미가 있다** — 10은 동일 스토리지·동일 스키마·동일 데이터 위에서 카탈로그만 바꿔 비교했기 때문에, 읽기 성능 차이가 없다는 결과를 신뢰할 수 있었습니다.
