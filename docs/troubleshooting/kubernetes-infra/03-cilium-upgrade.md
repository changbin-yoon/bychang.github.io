---
title: 03. Cilium 1.18 → 1.19 무중단 업그레이드
---

# 🔄 네트워크 — Cilium 1.18 → 1.19 무중단 업그레이드

!!! abstract "요약"
    190노드 규모, ClusterMesh + BGP + kube-proxy replacement 환경에서 Cilium을 무중단 롤링 업그레이드한 실행 기록.
    Kubernetes 1.33 → 1.34 → 1.35 업그레이드와 순서를 맞춰 계획.

## 업그레이드 순서 판단

**Cilium을 먼저 올리고 Kubernetes를 올리는 순서**를 선택했습니다.

근거는 다음과 같습니다. Cilium 1.19가 K8s 1.35 클라이언트 라이브러리로 테스트되어 릴리스됐고, 호환 매트릭스상 K8s 1.33은 Cilium 1.18 / 1.19 둘 다 지원하지만 1.34+는 1.19.x가 기준입니다. 따라서 현재 버전(1.33.3)에서 **Cilium부터 올리는 경로가 항상 지원 범위 안에 머뭅니다.**

Kubespray는 마이너 버전 스킵이 안 되므로 K8s는 1.33 → 1.34 → 1.35 **두 단계**로 진행합니다.

## Breaking Change — 환경에 직접 해당된 것들

### 1) `policy-default-local-cluster` 기본값 변경 (가장 중요)

Cilium 1.18에서 도입된 이 옵션이 1.19에서 **기본 `true`** 로 바뀝니다. 기존엔 NetworkPolicy가 모든 클러스터의 엔드포인트를 암묵적으로 선택했는데, 이제 로컬 클러스터로 제한됩니다.

ClusterMesh + 네트워크 정책을 함께 쓰는 두 클러스터 환경이라 breaking change에 해당했습니다.

```bash
cilium clustermesh inspect-policy-default-local-cluster --all-namespaces
# 영향받는 정책 사전 확인
```

### 2) BGP v1 CRD 완전 제거

`CiliumBGPPeeringPolicy`(v1)가 제거되어 `CiliumBGPClusterConfig` / `CiliumBGPPeerConfig` / `CiliumBGPAdvertisement`(v2)로 **사전 마이그레이션이 필요**합니다.

GitOps에 v1 매니페스트가 남아 있으면 **업그레이드가 즉시 실패**합니다.

### 3) 롤링 중 ClusterRole 권한 이슈

업그레이드 도중 ClusterRole에서 `ciliumbgppeeringpolicies` 권한이 제거되면서, 아직 구버전으로 도는 파드가 해당 리소스를 list하지 못해 **BGP 연결이 끊기는 사례**가 보고됐습니다.

190노드 규모라 롤링이 오래 걸려 **혼재 기간이 길기** 때문에, 임시로 권한을 유지하는 워크어라운드가 필요합니다.

## 실행 절차 요약

```yaml
# cilium-1.19.6-values.yaml 핵심 항목
upgradeCompatibility: "1.18"        # 신기능 자동 활성화·datapath 재구성 방지
clustermesh:
  policyDefaultLocalCluster: false  # breaking change 대응 결정
updateStrategy:
  rollingUpdate:
    maxUnavailable: 2               # 초기값, 카나리 검증 후 상향 검토
```

!!! danger "`--reuse-values` 금지"
    `--reuse-values` 는 차트 버전이 같을 때만 안전합니다. 버전이 바뀌면 **신규 기본값이 누락되어 템플릿이 깨집니다.**

**절차**

1. 이미지 사전 미러링(에어갭)
2. preflight DaemonSet
3. `helm upgrade` 직후 즉시 `rollout pause` 로 **카나리 1~2노드만 검증**
   (`cilium-dbg status --verbose`, endpoint ready, Pod IP 불변, BGP 세션, ClusterMesh 상태)
4. `rollout resume` 후 25% / 50% 지점에서 중간 점검
5. 전체 완료 후 `cilium status --wait`, 전 노드 버전 일치, `cilium connectivity test`, BGP / ClusterMesh 최종 확인

## CNPG 파드 재시작 이슈

이전 업그레이드에서 CNPG(CloudNativePG) 파드가 재시작된 사례가 있었습니다.

**원인** — CNPG instance manager의 liveness probe가 API 서버 isolation check를 포함하는데, kube-proxy replacement 환경에서는 ClusterIP 해석이 Cilium eBPF에 의존하므로 **Cilium agent 재시작 중 짧게 실패** → kubelet이 컨테이너를 재시작.

**대응** — probe 완화 또는 업그레이드 윈도우 동안 fencing. 상세: [Kubernetes Service 04번 문서](../kubernetes-service/04-cnpg-cilium.md)

## 핵심 결론

- **Pod IP 재할당은 없다** — agent 재시작 시 디스크의 로컬 상태에서 endpoint를 복원하고, 이미 로드된 eBPF 프로그램이 재시작 중에도 데이터플레인을 계속 처리하기 때문.
- 실제 다운타임 리스크는 Pod IP 변경이 아니라 **BGP 세션 단절**이며, Cilium과 Leaf 스위치 양쪽의 **Graceful Restart** 로 완화한다.
- **`--reuse-values` 금지, `upgradeCompatibility` 명시, 카나리 우선 검증**이 3대 원칙.
