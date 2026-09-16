---
title: 01. 클러스터 구축·운영 도구
---

# 🔧 클러스터 구축·운영 도구 — Kubespray/AWX, Node Tuning Operator, OIDC

!!! abstract "요약"
    190노드(5 마스터 + ~185 워커) 규모에서 반복적인 노드 설정 변경을 어떻게 통제 가능하게 만들 것인가에 대한 조사 기록.

## 배경

Kubespray + Ansible로 클러스터를 구축했지만, **190개 동일 스펙 노드에 대한 설정 변경**(OS 튜닝, sysctl, kubeadm 변경 등)을 매번 수동 Ansible 실행으로 처리하는 게 지속 불가능하다는 문제의식에서 시작했습니다.

## 오케스트레이션 UI 비교 — AWX vs Rundeck vs Terraform vs Semaphore

세 도구가 사실 **서로 다른 계층**에서 동작한다는 점이 핵심 결론이었습니다.

| 도구 | 계층 | 판단 |
| --- | --- | --- |
| **AWX** | Ansible 실행을 감싸는 컨트롤 플레인 UI (RBAC / 감사 포함) | Kubernetes Operator로 배포, **채택** |
| Rundeck | 범용 잡 오케스트레이터 | Ansible 중심 워크로드와 불일치, 제외 |
| Terraform / OpenTofu | 인프라 프로비저닝(VM 등) | 베어메탈 전용 환경이라 적용 대상 자체가 없음, 제외 |
| Semaphore | 경량 Ansible / Kubespray 웹 UI | AWX 개발 중단(2024.07) 이후 대안으로 재검토, Docker Compose로 가볍게 운영 가능 |

!!! warning "순환 의존성 주의"
    AWX를 관리 대상 클러스터 위에 올리면 **"자기 자신을 관리하는 도구가 관리 대상과 함께 죽는"** 순환 의존성이 생깁니다. 기존 standalone Ansible 마스터를 **bastion break-glass** 로 유지하는 구조를 권장합니다.

190노드 규모에서의 Ansible 성능 튜닝 포인트:

- `forks 50~100`
- pipelining, ControlPersist
- fact caching (jsonfile / Redis)
- 에어갭 대응 — Kubespray의 ansible-core 버전에 고정한 커스텀 Execution Environment 빌드

## 두 번째 축 — Node Tuning Operator (선언적 드리프트 교정)

WebUI만으로는 절반만 해결된다는 게 중요한 지점입니다. Ansible push는 **"그 순간의 상태"만** 맞추고, 이후 누군가 수동으로 노드를 건드리면 drift가 생겨도 감지되지 않습니다.

| 계층 | 도구 | 대상 | 방식 |
| --- | --- | --- | --- |
| **Tier 1** | Ansible / Kubespray | 클러스터 라이프사이클, base OS 설치 | 명령형(push) |
| **Tier 2** | Node Tuning Operator + TuneD profile | sysctl, hugepages, NUMA balancing, CPU Manager static policy, Topology Manager | 선언형(Kubernetes-native), 185개 워커 전체에 self-healing 방식으로 지속 강제 |

두 계층을 분리한 이유는 **Tier 1은 "바꾸는 행위", Tier 2는 "바뀐 상태를 계속 유지시키는 행위"** 로 책임이 다르기 때문입니다.

## Headlamp + Keycloak OIDC 연동

Kubernetes 대시보드(Headlamp)에 사내 Keycloak SSO를 붙인 설정 기록입니다.

1. **Keycloak** — Client 생성(`confidential`), redirect URI 등록, RBAC 연동을 위한 `groups` Mapper 추가 (Group Membership, Token Claim Name = `groups`)
2. **kube-apiserver** — 플래그 추가
   ```text
   --oidc-issuer-url
   --oidc-client-id
   --oidc-username-claim=preferred_username
   --oidc-groups-claim=groups
   --oidc-username-prefix / --oidc-groups-prefix
   ```
3. **Headlamp Helm values** — `config.oidc` 블록 신규 추가 (clientSecret은 별도 K8s Secret으로 분리 — `clientSecretName` / `clientSecretKey` 사용 권장), `ingress.enabled` 등 기존 default 값 변경
4. **RBAC** — OIDC 그룹 / 사용자 prefix와 ClusterRoleBinding의 subject name prefix를 정확히 일치시켜야 함(`oidc:` 등)

## kubelet rootDir 변경 (Kubespray)

kubelet의 데이터 디렉터리를 옮길 때 **Kubespray 변수 변경만으로는 부족**하고, 기존 노드의 실제 데이터 이관(볼륨 마운트 경로, 기존 컨테이너 상태)까지 함께 고려해야 합니다.

신규 노드는 변수 설정만으로 충분하지만, **기존 운영 노드는 rolling 절차가 필요**합니다.

## 결론

- 컨트롤 플레인 UI(AWX / Semaphore)는 **변경을 가하는 도구**, Node Tuning Operator는 **변경을 유지시키는 도구** 로 역할이 다르며 둘 다 필요하다.
- **클러스터를 관리하는 도구 자체를 클러스터 안에 두지 않는다**(순환 의존성 방지).
- OIDC 연동은 apiserver 플래그 · IdP 설정 · Helm values · RBAC subject **네 곳의 prefix / claim 일치가 전부**다.
