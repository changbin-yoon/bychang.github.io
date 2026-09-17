---
title: 01. Cilium BGP Control Plane & ClusterMesh
---

# 🌐 네트워크 — Cilium BGP Control Plane & ClusterMesh

!!! abstract "요약"
    두 클러스터(스토리지 / 컴퓨트) 간 BGP + ClusterMesh 구성을 검증한 기록. Native routing 환경에서 종종 헷갈리는 설정값 해석 포함.

## 아키텍처 — BGP와 ClusterMesh는 다른 계층

가장 먼저 정리한 개념적 구분입니다.

- **Cilium BGP Control Plane = 데이터 플레인**
  PodCIDR을 물리 네트워크(ToR 스위치)에 광고해 **실제 패킷이 다니는 경로**를 만듭니다.
- **Cilium ClusterMesh = 컨트롤 플레인**
  클러스터 간 서비스 / 엔드포인트 아이덴티티를 동기화해 **"어디로 가야 하는지"** 를 알려줍니다.

BGP는 ClusterMesh의 전제조건이 아닙니다 — 원리적으로는 별개지만, 베어메탈 환경에서 클러스터 간 통신이 실제로 되려면 결국 라우팅(BGP)이 뒷받침돼야 하므로 사실상 함께 쓰입니다.

## 헷갈렸던 설정값 — `tunnel-protocol: vxlan` + `routing-mode: native`

!!! warning "판단 기준은 routing-mode"
    `tunnel-protocol: vxlan` 이 설정되어 있어도 `routing-mode: native` 이면 실제로는 **네이티브 라우팅**이 동작합니다.
    tunnel-protocol 값은 tunnel 모드가 아닐 때는 쓰이지 않는 **미사용 default** 일 뿐이므로, `routing-mode` 가 최종 판단 기준입니다.

## 6계층 검증 체크리스트

환경 전체가 제대로 붙어 있는지 확인하기 위해 정리한 순서입니다(설정 → 데이터 플레인 검증).

| 계층 | 확인 명령 | 확인 대상 |
| --- | --- | --- |
| 1. 설정 | `cilium config view` | routing-mode, tunnel-protocol 등 선언값 |
| 2. BGP 세션 | `cilium bgp peers` | Established 상태 |
| 3. 커널 라우팅 | `ip route get <pod-ip>` | 실제 라우팅 테이블에 반영됐는지 |
| 4. ClusterMesh 연결 | `cilium clustermesh status` | 양방향 연결 상태 |
| 5. End-to-end | `cilium connectivity test` | 실제 통신 성공 여부 |
| 6. 패킷 레벨 | 물리 NIC `tcpdump` (VXLAN이면 UDP 8472 대역 확인) | underlay 라우팅의 최종 증거 |

ToR 게이트웨이 식별은 **LLDP와 MAC OUI lookup** 으로 수행했습니다.

## 결론

- BGP(데이터 플레인)와 ClusterMesh(컨트롤 플레인)를 분리해서 이해하면 **장애 지점을 좁히기 쉽다.**
- `tunnel-protocol` 값만 보고 tunnel 모드로 오판하지 말 것 — **`routing-mode` 가 우선.**
- 설정만 확인하고 끝내지 말고 **패킷 레벨(tcpdump)까지** 봐야 "진짜 underlay를 타는지" 확정할 수 있다.
