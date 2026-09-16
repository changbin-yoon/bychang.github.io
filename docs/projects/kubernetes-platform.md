---
title: Kubernetes 데이터플랫폼 구축·운영
---

# Kubernetes 기반 신규 데이터플랫폼 클러스터 구축·운영

| 항목 | 내용 |
| --- | --- |
| 기간 | 2026 |
| 역할 | 클러스터 구축, 공통 컴포넌트 구성, GitOps 배포 체계 설계 |
| 핵심 기술 | Kubespray, Cilium, Keycloak, MinIO, ArgoCD, Kustomize |

## 수행 내용

- **Kubespray 기반 4개 환경 K8s 클러스터 구축** — dev / stg / prd / storage
- 설치 / 워커 추가 / 업그레이드 / 해제 **스크립트 자동화**
- **플랫폼 공통 컴포넌트 구성** — Cilium CNI, Keycloak, MinIO 등
- 수백 대 Worker Node 확보 및 **리소스 할당 정책 수립**

## ArgoCD 기반 GitOps 배포 체계 확립

- Platform(인프라) 레포와 Services 레포를 **분리**
- **ApplicationSet + Kustomize overlay** 구조로 환경별 배포 일원화
