---
title: 07. ClusterMesh 신규 PodCIDR 라우팅 누락
---

# 🌐 네트워크 — ClusterMesh 신규 PodCIDR 라우팅 누락과 No route to host

!!! abstract "요약"
    컴퓨트 → 스토리지 클러스터로 글로벌 서비스를 호출할 때 간헐적으로 `No route to host` 가 발생한 건의 조사 기록.
    ClusterMesh는 정상이었고 원인은 언더레이 라우팅이었다. 커널이 어느 단계에서 이 에러를 만드는지까지 확인.

!!! warning "보완 필요"
    이후 확인 결과, 이 클러스터는 **Geneve 터널 모드 + DSR 구성**입니다. 터널 모드에서는 원격 PodCIDR이 커널 FIB에 없어도 정상이므로, 아래 BGP 전파 중심의 원인 분석은 **전제부터 재검토가 필요**합니다.
    상세는 [Geneve 터널 라우팅 검증](../verification/04-geneve-routing.md) 참고.

## 증상 — 같은 명령이 될 때도, 안 될 때도

컴퓨트 클러스터 일부 노드의 busybox 파드에서 `nc -zv <svc-url> 80` 을 실행하면 성공과 실패가 번갈아 나타났습니다. 실패 시 에러는 `No route to host`. **다른 노드에서는 100% 성공**했습니다.

글로벌 서비스라 백엔드가 여러 개이고, eBPF 로드밸런서가 특정 백엔드를 선택했을 때만 실패했습니다. 무작위가 아니라 **분산된 대상 중 일부 집합에만 결정적으로 실패**하는 상황이었습니다. 백엔드 IP를 고정해 직접 호출해보니 **신규 대역 파드에서만 실패**했습니다.

## 구성 전제 — BGP는 데이터 플레인, ClusterMesh는 컨트롤 플레인

계층 구분은 [Cilium BGP Control Plane & ClusterMesh 검증](../verification/01-cilium-bgp-clustermesh.md)에 정리한 그대로입니다. 이번 건은 그 구분이 **실제 장애로 드러난 사례**입니다.

이번 건과 직접 관련된 구성 특성은 세 가지입니다.

- 19랙 Spine-Leaf 패브릭, 랙당 노드 10대, 총 190대
- **Leaf 19대가 동일 ASN을 공유**하고, 노드 190대도 공통 localASN 사용
- ClusterMesh는 KVStoreMesh 미사용 — 각 cilium-agent가 원격 clustermesh-apiserver의 사이드카 etcd에 직접 연결

## 원인 — 신규 PodCIDR이 일부 노드 라우팅 테이블에 없었다

스토리지 클러스터에 신규 PodCIDR 대역이 할당됐고, 그 대역의 파드가 뜨기 시작했습니다.

ClusterMesh는 신규 파드를 백엔드로 **정상 인지**하고 있었습니다. `cilium-dbg service list` 의 백엔드 목록은 정상 노드와 동일했습니다. 그런데 일부 노드의 커널 라우팅 테이블에는 **그 대역으로 가는 경로가 없었습니다.**

> 컨트롤 플레인은 "어디로 가야 하는지"를 알았지만, 데이터 플레인에 "어떻게 가는지"가 없었다.

BGP로 PodCIDR을 전파하는 구조에서 **신규 대역 prefix가 일부 노드까지 전파되지 않은 것**이 근본 원인입니다. 파드가 새로 뜰 때마다 신규 대역 IP가 할당될 수 있으므로, 전파가 누락되면 **그 시점 이후 생성된 파드에 대해서만** 실패합니다. 간헐성의 정체가 이것입니다.

## 왜 "Network is unreachable"이 아니라 "No route to host"였나

같은 "경로 없음"이라도 **조회 경로에 따라 errno가 다릅니다.** 이번 건에서 가장 헷갈릴 수 있는 지점입니다.

| 조회 경로 | 커널 처리 | errno | 보이는 메시지 |
| --- | --- | --- | --- |
| output (호스트가 직접 송신) | `ip_route_output_key` 실패 | ENETUNREACH | `Network is unreachable` |
| input / forward (파드 → 호스트 경유) | `ip_route_input_slow` 의 `no_route:` 에서 RTN_UNREACHABLE 설정 | EHOSTUNREACH | `No route to host` |

파드는 별도 netns이므로 호스트 입장에서는 **포워딩**입니다. 따라서 forward 경로를 타고, FIB 미스가 RTN_UNREACHABLE로 처리되면서 `ip_error()` 가 ICMP type 3 code 1을 파드로 회신합니다. 파드의 TCP 스택이 이를 EHOSTUNREACH로 변환해 busybox가 `No route to host` 를 출력한 것입니다.

같은 노드의 호스트 셸에서 동일 IP로 테스트했다면 `Network is unreachable` 이 나왔을 것입니다. **메시지가 다르다고 별개 문제로 오판하기 쉽습니다.**

덧붙여 커널의 ICMP 에러 생성에는 rate limit(`net.ipv4.icmp_ratelimit`, 기본 1초)이 걸려 있습니다. 짧은 간격으로 반복 시도하면 어떤 요청은 즉시 `No route to host` 가, 어떤 요청은 무응답 타임아웃이 발생합니다. 재현 테스트 시 혼선을 줄 수 있는 요소입니다.

## 커널이 확인하는 순서

FIB 조회 자체는 아래 순서로 진행됩니다.

1. **policy routing rule 평가** (`ip rule`) — 우선순위 낮은 번호부터. Cilium이 추가한 룰 포함
2. 각 룰이 지목한 **테이블 조회** — 기본 룰 기준 `local`(255) → `main`(254) → `default`(253)
3. 매칭된 경로의 **타입 판정** — unicast / unreachable / blackhole / prohibit
4. 경로가 있으면 **neighbour 해석**(ARP). 여기서 실패해도 최종 errno는 EHOSTUNREACH

## 검증 체크리스트

| # | 확인 대상 | 명령 |
| --- | --- | --- |
| 1 | 백엔드 인지 여부 (컨트롤 플레인) | `cilium-dbg service list` |
| 2 | BGP 세션 · 수신 prefix | `cilium bgp peers`, `cilium bgp routes available` |
| 3 | output 경로 조회 | `ip route get <원격 pod IP>` |
| 4 | forward 경로 조회 (실제 장애 경로) | `ip route get <원격 pod IP> from <로컬 pod IP> iif <lxc 인터페이스>` |
| 5 | 정책 라우팅 전체 | `ip rule show`, `ip route show table all` |
| 6 | 커널 카운터 | `/proc/net/stat/rt_cache` 의 in_no_route, `nstat -az \| grep IcmpOutDestUnreachs` |

## 남은 질문 — 왜 특정 노드에서만인가 (미확정)

아직 확정되지 않았습니다. 확인해야 할 후보는 다음과 같습니다.

- **동일 ASN 공유로 인한 AS_PATH 루프 거부** — 우선 확인 대상. Leaf 19대가 같은 ASN을 쓰면 Leaf-A가 올린 경로가 Spine을 거쳐 Leaf-B로 내려올 때 Leaf-B는 AS_PATH에서 자기 ASN을 보고 기본 정책상 거부합니다. 노드 190대가 공통 localASN을 쓰는 것도 같은 구조입니다. 두 클러스터가 같은 localASN을 공유한다면 원격 PodCIDR은 애초에 수신될 수 없습니다. `allowas-in` 설정 유무와 그 값이 랙마다 일관된지 확인이 필요합니다.
- Leaf 또는 노드 측 import / export policy(prefix-list)가 신규 대역을 필터링
- 신규 대역 광고 시점에 해당 노드의 BGP 세션이 flap 또는 미수립 상태
- BGP max-prefix 제한 도달로 신규 prefix 거부

또 하나 확정이 필요한 전제가 있습니다. EHOSTUNREACH가 발생했다는 것은 **해당 트래픽이 default route에 매칭되지 않는 경로로 조회됐다**는 뜻입니다. default route가 살아 있고 거기 매칭됐다면 패킷은 게이트웨이로 나갔을 것이고 다른 증상이 나왔어야 합니다. pod 네트워크용 별도 라우팅 테이블이나 `ip rule` 이 관여하고 있을 가능성이 높습니다. 위 체크리스트 5번 결과에 따라 재발 방지 설계가 달라집니다.

## 조치와 재발 방지

문제 노드에 해당 PodCIDR 대역 경로를 **수동 추가해 증상은 해소**됐습니다. 다만 이는 임시방편입니다. 수동 추가한 경로는 노드 재부팅 시 소실될 수 있고, 다음에 또 다른 신규 대역이 할당되면 동일 장애가 재발합니다.

**구조적 해결책은 PodCIDR 요약(aggregate) prefix 광고**입니다. 노드별로 잘게 쪼개진 PodCIDR을 개별 광고하는 대신 클러스터 pool 전체를 포괄하는 상위 요약 prefix 하나를 광고하면, 신규 노드 증설이나 신규 대역 할당 시에도 라우팅 정보가 추가로 필요 없어집니다. 190노드 규모에서 패브릭의 prefix 수가 줄어드는 부수 효과도 있습니다.

요약 prefix로 해결되지 않을 경우의 차선책은 **ClusterMesh 구간을 터널(VXLAN) 모드로 두는 것**입니다. 노드 IP 간 도달성만 확보되면 되므로 PodCIDR 단위 언더레이 라우팅 의존이 사라집니다. 다만 캡슐화 오버헤드와 MTU 재조정이 수반됩니다.

모니터링 측면에서는 **노드별 원격 PodCIDR 경로 보유 여부를 주기 점검**하고, BGP 세션 상태와 **노드별 수신 prefix 수 편차**를 알람으로 잡으면 특정 노드만 prefix 수가 적을 때 즉시 인지할 수 있습니다.

## 결론

- ClusterMesh 백엔드 목록이 정상이라고 해서 **데이터 플레인 도달성이 보장되지 않는다.** 서비스 디스커버리 계층과 라우팅 계층은 분리해서 확인해야 한다.
- **"간헐적"이라는 증상은 대개 무작위가 아니라** 분산된 대상 중 일부 집합에만 결정적으로 실패하는 경우다. 백엔드를 고정해 테스트하는 것이 원인 규명의 첫 단계다.
- 같은 "경로 없음"도 output 경로와 forward 경로에서 **errno가 다르게 나온다.** 파드에서 본 에러와 호스트에서 본 에러가 다를 수 있다는 점을 전제로 진단해야 한다.
- **정적 라우트 추가는 임시조치다.** BGP 전파 구조 자체를 고치지 않으면 다음 신규 대역에서 그대로 재발한다.
