---
title: Kubernetes Infra
---

# 🏗️ [Kubernetes Infra] 클러스터·네트워크·스케줄러 운영 기록

베어메탈 Kubernetes 클러스터(약 190노드, 5 마스터 + ~185 워커, 노드당 96코어 / 1TB, Kubespray 배포, 에어갭 환경)의 **인프라 계층**에서 조사·트러블슈팅한 내용을 유형별로 정리한 문서 모음입니다. 워크로드(Spark / Airflow / CNPG 등) 관점의 내용은 [Kubernetes Service](../kubernetes-service/index.md) 문서로 분리했습니다.

!!! info "대상 환경"
    베어메탈 Kubernetes, Kubespray, Cilium CNI(BGP Control Plane + ClusterMesh), YuniKorn Gang Scheduler, Airgapped, MultiCluster Network, CI/CD(Jenkins / ArgoCD), Nexus, Helm, Kustomize

**정리 기준** — 각 문서는 배경/질문 → 조사·검증 내용 → 결론·조치 순서

## 문서 목록

| # | 유형 | 다루는 내용 |
| --- | --- | --- |
| [01](01-cluster-tooling.md) | 클러스터 구축·운영 도구 | Kubespray/AWX 오케스트레이션, Node Tuning Operator, RBAC, OIDC 연동, kubelet 경로 이관 |
| [02](02-cilium-bgp-clustermesh.md) | 네트워크 — Cilium BGP/ClusterMesh | BGP Control Plane 구성, ClusterMesh 연결·검증, native routing 판별 |
| [03](03-cilium-upgrade.md) | 네트워크 — Cilium 버전 업그레이드 | 1.18→1.19 무중단 롤링, breaking change 대응, K8s 버전 동시 계획 |
| [04](04-yunikorn-gang-scheduling.md) | 스케줄러 — YuniKorn Gang Scheduling | Placeholder pod 생명주기, conntrack 포화, Admission Controller |
| [05](05-apiserver-watch-race.md) | API 서버 부하 & Watch 레이스 | Volume Mount 실패 레이스 컨디션, APF, workqueue 튜닝 |
| [06](06-prometheus.md) | 모니터링 — Prometheus | RBAC, Agent 모드, apiserver 메트릭/인증서 만료 진단 |
| [07](07-clustermesh-podcidr.md) | 네트워크 — ClusterMesh | 신규 PodCIDR 라우팅 누락과 No route to host |
| [08](08-geneve-routing.md) | 네트워크 — Geneve 터널 | 커널 라우팅이 필요해지는 순간 (RIB/FIB, ipcache 폴백) |
| [09](09-podcidr-missing-routes.md) | 네트워크 — 경로 누락 | 190노드 중 3노드 PodCIDR 경로 누락: 원인 후보와 공개 사례 |

## 공통 전제

- 클러스터는 Kubespray로 구축되었으며 **kube-proxy replacement(Cilium)** 를 사용합니다.
- **에어갭 환경**이므로 이미지 / 차트는 Nexus 내부 미러를 경유합니다.
- 각 문서의 수치·버전은 조사 시점 기준이며, 이후 업그레이드로 달라질 수 있습니다.
