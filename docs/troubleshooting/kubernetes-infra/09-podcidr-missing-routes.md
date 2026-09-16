---
title: 09. 190노드 중 3노드 PodCIDR 경로 누락
---

# 🔎 네트워크 — 190노드 중 3노드 PodCIDR 경로 누락: 원인 후보와 공개 사례

!!! abstract "조사 기준 2026-09-13 · 원인 미확정"
    **관측** — 190개 출발 노드 중 3개에서 원격 Pod 대상 접근 시 `No route to host`. Cilium 1.18.4 → 1.19.7 롤링 업그레이드와 재기동 후에도 지속.
    아래는 재현 가능한 **가설과 공개 사례**입니다.

!!! warning "가장 중요한 선결 확인"
    기존 내부 기록 [07번](07-clustermesh-podcidr.md), [08번](08-geneve-routing.md)은 해당 190노드 환경을 **Geneve 터널 + DSR** 로 적었습니다.
    이 기록이 현재 환경에도 맞다면 **원격 PodCIDR의 Linux 경로가 없는 것만으로는 장애 원인이라고 결론 낼 수 없습니다.** native routing 전제 설명은 조건부로 읽어야 합니다.

## 구성 구조

**Native routing을 사용하는 경우** ClusterMesh는 원격 엔드포인트 · 노드 메타데이터를 동기화하고, 각 노드의 Cilium BGP Control Plane은 자기 노드에 할당된 PodCIDR을 피어인 Leaf 스위치에 광고합니다. Leaf / Spine은 BGP로 학습한 경로를 사용합니다. **Cilium BGP가 학습한 원격 prefix를 각 노드의 Linux FIB에 자동 설치하는 구조는 아닙니다.**

같은 L2의 노드 간에는 `autoDirectNodeRoutes=true` 가 대상 노드 IP를 다음 홉으로 하는 원격 PodCIDR 경로를 설치할 수 있습니다. `directRoutingSkipUnreachable=true` 면 다른 L2에 있는 노드에 대해서는 이 직접 경로를 생략합니다. 이 경우 출발 노드가 패킷을 스위치로 보내는 유효한 경로를 별도로 가져야 하며, 스위치와 반환 경로에도 양쪽 PodCIDR이 알려져 있어야 합니다.

## 1. 먼저 확인할 사실

- **문제 3대와 정상 1대에서 실제 적용값 비교** — `routing-mode`, `tunnel-protocol`, `auto-direct-node-routes`, `direct-routing-skip-unreachable`, `ipv4-native-routing-cidr`, `bpf-lb-mode`, `bpf-lb-dsr-dispatch`. Helm values뿐 아니라 각 agent의 `cilium-dbg config` 와 로그를 확인합니다. 노드별 CiliumNodeConfig override 및 이미지 digest도 비교합니다.
- **오류가 나는 정확한 최종 목적지 Pod IP를 고정**합니다. Global Service는 백엔드가 달라질 수 있으므로 서비스 VIP만 검사하면 원인을 놓칩니다.
- 정상 노드의 "경로"가 무엇인지 `ip -d route show table all` 로 확인합니다. `via 대상노드` 직접 경로인지, `via 스위치` 인 정적 / 라우팅 데몬 경로인지, 단순 default route인지 **출처를 구분**합니다.

## 2. 모드별 핵심 분기

### A. routingMode=tunnel, tunnelProtocol=geneve

공식 문서상 터널 모드에서는 노드 IP 간 연결과 **Geneve UDP 6081**이 필요하고, 언더레이가 원격 PodCIDR을 알 필요는 없습니다. 원격 PodCIDR이 커널 FIB에 없다는 관찰만으로 `autoDirectNodeRoutes` 실패라고 할 수 없습니다.

이 경우 우선 **원격 Pod→노드 매핑(ipcache), ClusterMesh 동기화, 터널 전송, 서비스 백엔드 선택, DSR dispatch** 를 조사합니다. DSR의 `opt` dispatch는 터널 모드와 호환되지 않고 `geneve` dispatch는 호환됩니다.

다만 **ipcache 누락이나 DSR 설정 불일치를 현재 원인으로 확정한 증거는 아직 없습니다.**

### B. routingMode=native

`autoDirectNodeRoutes` 는 같은 L2의 원격 노드 PodCIDR에 대해 직접 경로를 설치합니다. `directRoutingSkipUnreachable=true` 면 다른 L2 세그먼트의 직접 경로를 건너뜁니다.

BGP Control Plane은 스위치에 PodCIDR을 광고하지만 **노드 FIB를 직접 채우지 않습니다.** 따라서 노드에 개별 경로가 없다면 default / 별도 라우팅 테이블을 통해 스위치로 가는지 확인합니다.

## 3. 직접 PodCIDR 경로가 3개 출발 노드에서만 빠질 수 있는 경우

1. **L2 판정 차이** — 문제 노드에서 대상 노드 IP가 직접 연결되지 않고 `via <gateway>` 로 조회됩니다. NIC, VLAN, 주소 / 넷마스크, policy rule 차이 또는 Cilium이 선택한 노드 IP 차이가 원인일 수 있습니다. `directRoutingSkipUnreachable=true` 라면 **생략이 의도된 동작**일 수 있습니다.
2. **대상 정보 누락 / 불일치** — 해당 agent가 원격 CiliumNode의 노드 IP · PodCIDR을 충분히 받지 못했거나, 대상의 PodCIDR 할당 정보가 비어 있습니다. 특히 신규 pool / CIDR, ClusterMesh 동기화 상태를 확인합니다. *(후보이며 현재 환경에서 입증되지 않음)*
3. **경로 설치 실패 / 충돌** — 같은 prefix의 기존 경로, 잘못된 다음 홉, kernel netlink 오류 등. agent 로그에서 `direct route`, `directly reachable`, `failed to enable direct routes`, `File exists` 를 찾습니다.
4. **경로가 설치된 뒤 삭제됨** — NetworkManager / systemd-networkd / FRR / 자동화 / 수동 변경 또는 Cilium 재조정 문제. 정상 경로의 `proto` 와 journal · 설정관리 기록, `ip monitor route` 로 확인합니다. 재기동 후에도 없으면 단순 일시적 삭제보다는 **반복되는 로컬 상태 / 설치 조건**을 우선 의심합니다.
5. **노드별 설정 차이** — DaemonSet 템플릿은 같아도 CiliumNodeConfig override, agent 버전 · 이미지 digest, 부팅된 커널, host 경로 / 규칙은 다를 수 있습니다. 커널의 `ip_forward` · `rp_filter` 는 주로 전달 / 드롭 증상에 영향을 주므로 **경로 자체가 없는 원인의 첫 후보는 아닙니다.**
6. **1.19 회귀** — 동일 입력 · 네트워크 상태 · 설정에서 1.18.4는 경로를 설치했고 1.19.7은 세 노드에서 반복 실패했다는 증거가 모이면 Cilium 버그로 분류할 근거가 됩니다. **현재 조사한 공개 자료에서는 이 정확한 1.19.7 패턴을 확인하지 못했습니다.**

## 4. 공개 발생 사례 — 현재 건과의 관련성

| 사례 | 내용 | 관련성 |
| --- | --- | --- |
| [#24777](https://github.com/cilium/cilium/issues/24777) | Hetzner 혼합 환경에서 대상 노드 IP로 가는 경로에 gateway가 있어 Cilium이 직접 PodCIDR 경로를 거부. 유지보수자도 **동일 L2가 필수**라고 설명 | 3개 노드의 대상 노드 IP 조회가 다를 때 **직접 관련** |
| [#31843](https://github.com/cilium/cilium/issues/31843) | ClusterMesh + auto-direct-node-routes 업그레이드 후 원격 노드를 gateway로 설치하려다 agent 실패 (1.14→1.15). 로그에 `must be directly reachable` | **버전도 증상도 달라 1.19.7 회귀 증거는 아님.** 비교할 오류 문자열 제공 |
| [#31124](https://github.com/cilium/cilium/issues/31124) | 여러 zone에서 `autoDirectNodeRoutes` 가 다른 서브넷 노드 때문에 실패 | skip 옵션 / 토폴로지 판정의 **배경 사례** |
| [#26663](https://github.com/cilium/cilium/pull/26663) | Multi-Pool · ENI · Azure 계열의 CiliumNode PodCIDR 파싱 버그 수정 | **IPAM 모드가 해당될 때만 관련** |
| [#41811](https://github.com/cilium/cilium/issues/41811) | 라우팅 모드 변경 후 기존 Cilium 경로 정리 실패 | 경로가 **추가되지 않은** 현상과는 반대 방향, 참고 수준 |
| [#45777](https://github.com/cilium/cilium/issues/45777) | Cilium CLI 0.19.x ClusterMesh 연결 정보 오기록 (Geneve 터널 환경) | direct route 누락 증거는 아님. ClusterMesh 상태가 다를 때만 참고 |

## 5. 최소 비교 수집 — 정상 1대 vs 문제 3대

아래는 **읽기 전용** 확인입니다. 동일한 대상 Pod IP와 대상 노드 IP를 사용합니다.

```bash
# 노드 호스트에서
ip -d route show table all
ip rule show
ip route get <대상_노드_IP>
ip route get <대상_Pod_IP>
ip route get <대상_Pod_IP> from <출발_Pod_IP> iif <출발_Pod의_host측_veth>
ip addr show

# Cilium Pod에서 (해당 노드에 뜬 Pod를 각각 선택)
kubectl -n kube-system exec <cilium-pod> -- cilium-dbg status --verbose
kubectl -n kube-system exec <cilium-pod> -- cilium-dbg config
kubectl -n kube-system exec <cilium-pod> -- cilium-dbg bpf ipcache list
kubectl -n kube-system logs <cilium-pod> --since=24h

# 컨트롤 플레인
kubectl get ciliumnodes.cilium.io <대상-노드> -o yaml
kubectl get ciliumnodeconfigs.cilium.io -A
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

`ip route get` 은 호스트 커널 경로의 증거이지만 **Cilium eBPF 터널 경로 전체를 대변하지 않습니다.** `No route to host` 문자열도 경로 부재 외에 ICMP unreachable · 이웃 탐색 실패 · 거부 경로 등으로 발생할 수 있습니다. Geneve 모드라면 정상 / 문제 노드의 ipcache, 터널 맵과 원격 노드 IP 도달성, 포트 6081, Hubble / drop 이유를 함께 비교합니다.

## 6. 판정 기준

| 상황 | 판정 |
| --- | --- |
| 터널 모드 + 원격 PodCIDR 경로 없음 | **정상일 수 있다.** 누락 경로를 원인으로 확정하지 않는다 |
| Native + 문제 노드만 대상 노드 IP가 gateway 경유 | L2 skip 또는 네트워크 / 인터페이스 차이를 먼저 해결 |
| Native + 동일 L2 · 동일 설정 · 동일 CiliumNode 정보인데 설치 실패 로그 | 오류 원인 확인 후 Cilium 이슈 또는 환경 충돌로 분류 |
| Native + 설치 시도도 없고 재시작 후 계속 빠짐 | agent의 노드 정보 동기화, override, 1.19.7 회귀 의심 → 재현 자료 수집 |
| 수동 route add 후 통신 성공 | 커널 경로 사용 **가능성의 단서일 뿐**, 원래 Cilium이 그 경로를 설치해야 했다는 증거는 아니다 |

## 7. 아직 필요한 증거

실제 `routingMode` / DSR dispatch, 오류 발생 직전 선택된 최종 Pod IP, 문제 / 정상 노드의 위 비교 출력, 해당 CIDR의 route `proto`, Cilium agent 설치 오류 로그, 스위치의 대상 PodCIDR BGP 수신 경로.

**이 자료 없이는 원인을 특정하거나 1.19.7 버그라고 확정할 수 없습니다.**

## 참고 문서

- [Cilium Routing](https://docs.cilium.io/en/stable/network/concepts/routing/)
- [BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/)
- [DSR dispatch 지원표](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
