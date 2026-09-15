# 상세 경력 및 프로젝트 이력

**미리비트 / 빅데이터 개발본부 / 책임 (2020.04 ~ 재직 중)**

---

## 1. Kubernetes 기반 차세대 데이터 플랫폼 구축 및 운영 전환

- **고객사/Domain** : SK하이닉스 / 반도체 제조(Fab) 빅데이터 · DataOps 플랫폼
- **수행 기간** : 2025.01 ~ 현재
- **담당 역할** : 인프라 아키텍트 겸 운영 리드. 아키텍처 설계부터 구축, 자동화 자산 개발, 표준화·문서화까지 주도

### 업무 내용

**클러스터 구축 및 배포 체계**

- Kubespray 기반 dev / stg / prd / storage 4개 환경의 베어메탈 Kubernetes 클러스터 신규 구축
- 설치 · 워커 노드 추가 · 업그레이드 · 해제 전 과정의 스크립트 자동화 체계 수립
- Cilium CNI, Keycloak, MinIO 등 플랫폼 공통 컴포넌트 표준 구성
- 수백 대 규모 Worker Node 확보 및 서비스별 리소스 할당 정책 수립
- ArgoCD 기반 GitOps 배포 체계 확립 — Platform(인프라) / Services 레포 분리, ApplicationSet + Kustomize overlay 구조 설계
- OPA/Gatekeeper 정책 적용, Sealed Secrets 도입으로 인가·시크릿 관리 표준화

**워크로드 런타임 및 파이프라인 자동화**

- Airflow 오케스트레이션 체계 구축 : 커스텀 이미지 및 Helm 차트 제작, DataOps 공용 DAG 파이프라인 개발(Kafka → Iceberg 적재, Spark Operator 연동, 메트릭 컨슈머, DB/로그 클린업)
- DataLake Spark Runtime 이미지 개발(Spark 3.5.7 + Iceberg 카탈로그 + 메트릭 + Spark Connect) 및 서비스별 SparkApplication(CR) 배포 템플릿 표준화
- Spark WorkDir 정리 CronJob, 팀별 ServiceAccount 기반 Kubeconfig 발급 자동화
- Spark Operator 모니터링 체계 개발(Prometheus 메트릭 수집, 알람 규칙 정의)
- 폐쇄망(Air-gapped) 설치 런북 및 클러스터 사용 가이드 작성·배포

### 성과

온프레미스 Hadoop 중심 환경을 Kubernetes 기반 플랫폼으로 전환하는 인프라 기반을 마련했습니다. GitOps와 표준 배포 템플릿을 도입해 서비스 팀이 직접 워크로드를 배포·운영할 수 있는 체계를 확립했고, Spark Operator Controller Down으로 인한 작업 지연과 Airflow 지연(Spark Operator / YuniKorn 스케줄러 대기) 등 운영 이슈를 원인 단위로 규명·조치했습니다.

### 보유/활용 Skill

Kubernetes(Kubespray, RKE2), Cilium CNI, Helm, Kustomize, ArgoCD(ApplicationSet), Spark Operator, Airflow, Spark 3.5 + Iceberg, Kafka(Strimzi), Keycloak, OPA/Gatekeeper, Sealed Secrets, Prometheus, Grafana, Ansible, Python, Shell, Linux(RHEL)

---

## 2. Lakehouse 스토리지 및 메타데이터 계층 전환

- **고객사/Domain** : SK하이닉스 / 스토리지 인프라 · 데이터 카탈로그 거버넌스
- **수행 기간** : 2024.01 ~ 현재
- **담당 역할** : 스토리지 · 메타데이터 계층 오너. 기술 평가, 구축, 이관, 장애 대응, 거버넌스 가시화를 일관 수행

### 업무 내용

**스토리지 기술 평가 및 이관**

- fio 기반 스토리지 성능 테스트 : Isilon(SSD/HDD)과 Ceph(RBD/CephFS)를 RWO/RWX 2개 모드로 순차 · 랜덤 · IOPS 측정 및 비교
- 측정 결과를 근거로 운영 스토리지를 Isilon 기반으로 선정하고, 이에 맞춘 StorageClass 구성
- Rook-Ceph 설치·검증, External Ceph 연동, CSI-RBD / CSI-CephFS 드라이버 및 StorageClass 구성
- MinIO / AIStor 객체 스토리지 구축 및 버전 업그레이드, DirectPV 디스크 구성
- HDFS → MinIO 데이터 이관(S3A, jceks Credential 암호화), 샘플 데이터 이관 PoC 및 성능 테스트
- 시스템(Dev/Prd)별 MinIO Policy 권한 정책 수립·운영, LDAP 연동 Access Key 발급 및 권한 전파 검증
- AIStor Air-gapped 설치 런북 · 가이드북 제작, OpenSearch 스냅샷 리포지토리 연동

**메타데이터(HMS) 구축 및 감사**

- Oracle DB 기반 Hive Metastore 구축·운영, Hive Metastore Hook 적용, AIStor 연동
- OpenSearch 로그 기반 HMS 감사(Audit) Grafana 대시보드 개발 — DB/EventType별 CRUD 이벤트 가시화 및 위험 이벤트 하이라이트
- Iceberg 메타데이터 정리 스크립트 운영(rewrite_data_files, rewrite_manifests, expire_snapshots)
- Oracle 21c XE / 26ai Free 기반 HMS 스키마 검증, Hive Metastore(Oracle) vs Apache Polaris(REST) Iceberg 카탈로그 비교 검증, Gravitino PoC 수행
- Lakehouse 전환 방향성에 대한 사내 공청회 발표자료 작성 및 발표

### 성과

정량 벤치마크 결과를 근거로 스토리지 기술을 선정해 의사결정 근거를 확보했고, HDFS 중심 저장 계층을 객체 스토리지 기반으로 이관했습니다. 2026.05 HMS Oracle DB 장애 시 원인 분석 · 복구 및 장애 보고서 작성을 주도했으며, DirectPV 드라이브 공유로 인한 Erasure Set 쓰기 차단 이슈를 분석·조치했습니다. 감사 대시보드 도입으로 메타데이터 변경 이력의 사후 추적이 가능해졌습니다.

### 보유/활용 Skill

MinIO/AIStor(DirectPV, Policy, Airgapped), Ceph(Rook, RBD/CephFS, External Cluster), Isilon(CSI-NFS), HDFS, S3A, Hadoop Credential(jceks), Hive Metastore, Oracle DB, Apache Iceberg, Polaris, Gravitino, LDAP, OpenSearch, Grafana, fio, SQL, Python

---

## 3. Query Engine 도입 · 운영 및 성능 검증

- **고객사/Domain** : SK하이닉스 / 분석 쿼리 엔진
- **수행 기간** : 2024.01 ~ 현재
- **담당 역할** : 쿼리 엔진 운영 담당 및 기술 평가 주관. 도입 후보 엔진의 정량 비교와 선정 근거 제시

### 업무 내용

- 서비스별 Trino 클러스터 구축·운영 : Hive / Iceberg / PostgreSQL 카탈로그 연동, LDAP 인증 + OPA 인가 + Group Provider 구성, Kafka Event Listener, JMX Exporter, ServiceMonitor, Cilium Ingress 적용
- Trino 버전 고도화(408 → 442 → 475 → 482) 및 업그레이드 검증
- 쿼리 지연 이슈 대응 : 카탈로그 이미지 · 리소스 변경, HMS 로그 및 트래픽 분석을 통한 원인 규명과 튜닝
- StarRocks 도입 검토 및 kube-starrocks Operator 성능 평가, Apache Ignite(V2/V3) 클러스터 PoC(K8s Discovery, cluster-init, StatefulSet)
- 성능 · 부하 테스트 주관 : JMeter 기반 21종 쿼리 성능 측정, Locust 기반 Trino / Spark Operator 부하 테스트, HDFS vs MinIO vs Ceph 스토리지별 쿼리 성능 비교
- Trino Graceful Shutdown 검증 등 무중단 운영 절차 확립

### 성과

온프레미스 대비 성능을 정량 벤치마킹해 차세대 플랫폼 전환의 기술적 타당성을 입증했습니다. 인증·인가·모니터링이 통합된 멀티테넌트 쿼리 서비스 운영 체계를 확립했습니다.

### 보유/활용 Skill

Trino, StarRocks, Apache Ignite, Apache Spark, Apache Iceberg, LDAP, OPA/Gatekeeper, JMX, JMeter, Locust, Prometheus, Grafana, Cilium Ingress

---

## 4. T-Hadoop 대규모 빅데이터 클러스터 운영 및 분석 파이프라인 고도화

- **고객사/Domain** : SK하이닉스 / 온프레미스 빅데이터(Vanilla Hadoop), 반도체 공정 이상탐지(FDC)
- **수행 기간** : 2021.06 ~ 2025.12
- **담당 역할** : 클러스터 운영 담당. 정기 점검 · 리포팅, HW 교체 절차 표준화, 장애 대응, 파이프라인 전환 실무 수행

### 업무 내용

**클러스터 운영 및 안정화**

- 1,000대 이상 노드 규모의 베어메탈 Hadoop 클러스터 안정 운영 및 일일 점검
- HDFS, YARN 등 Hadoop Ecosystem 전반 심층 트러블슈팅 및 고가용성 유지
- Trino, Spark, Airflow 기반 빅데이터 플랫폼 운영 · 최적화
- SKT 메타트론 기반 모니터링 시스템 운영 지원(이천 / 청주 / 우시 사업장 연계)
- 모니터링 툴 데이터를 취합한 월간 리소스 사용량 보고서 작성
- 노후화 서버 교체 : HW Part 온 · 오프라인 교체 절차서 수립 및 수행(OS Disk, SSD, HDD, Memory, NIC), Ceph OSD 교체
- 이슈 대응 : HiveServer2 장애, UDP 네트워크 오류, Hadoop → Ceph 이관 테스트

**FDC 분석 모델 파이프라인 고도화 (2021.06 ~ 2022.01)**

- V1(Hadoop MapReduce 기반) : Apache Azkaban으로 일(Daily) 배치 스케줄링 및 안정 운영
- V2(Spark 기반) : Apache Airflow를 도입해 기존 MR 작업을 Spark 기반 시간(Hourly) 단위 작업으로 전환 · 고도화

### 성과

대규모 트래픽 · 데이터 처리 환경에서 장애 가시성을 확보하고 무중단 데이터 서비스 환경을 유지했습니다. 서버 교체 절차를 문서화해 재현 가능한 운영 표준으로 정착시켰고, FDC 파이프라인 현대화로 분석 주기를 일 단위에서 시간 단위로 단축해 모델 예측 안정성 향상에 기여했습니다.

### 보유/활용 Skill

Hadoop(HDFS, YARN, MapReduce), Hive, Trino, Spark, Airflow, Azkaban, Ceph, Zabbix, Metatron, Linux(RHEL), Shell, Python, SQL
