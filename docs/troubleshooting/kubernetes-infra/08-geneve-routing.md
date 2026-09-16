---
title: 08. Geneve 터널에서 커널 라우팅이 필요해지는 순간
---

# 🌐 네트워크 — Geneve 터널에서 커널 라우팅이 필요해지는 순간 (RIB/FIB, ipcache 폴백)

!!! abstract "요약"
    Geneve 터널 + DSR 환경에서 `No route to host` 를 수동 `route add` 로 넘긴 뒤, **"이게 원래 필요한 일이 맞나"** 를 파고든 기록.
    결론적으로 터널 모드에서는 원격 PodCIDR이 커널 FIB에 없어도 정상이었고, **수동 route는 증상을 가린 것이지 원인을 고친 게 아니었다.**

## RIB와 FIB는 다른 것이다

먼저 용어를 분리해야 나머지가 보입니다.

| | RIB | FIB |
| --- | --- | --- |
| 정체 | Routing Information Base — 프로토콜이 학습한 **경로 후보** 모음 | Forwarding Information Base — 커널이 **실제 전달에 쓰는** 테이블 |
| 어디 있나 | BGP 데몬(Cilium GoBGP, FRR, BIRD) 내부 | 리눅스 커널 내부 |
| 보는 법 | `cilium bgp routes available` | `ip route show` |

**RIB에 경로가 있다고 자동으로 FIB에 들어가지 않습니다.** 그 사이를 이어주는 동작이 바로 `route add` 입니다.

## Cilium BGP는 받은 경로를 FIB에 넣지 않는다

공식 문서가 명시적으로 밝힙니다.

> BGP Control Plane은 데이터패스를 프로그래밍하지 않으므로, 클러스터 내부 도달성 확보 용도로 사용해서는 안 된다.

즉 Cilium의 BGP 스피커는 **광고(advertise) 전용**입니다. 피어로부터 받은 경로는 RIB에만 머물고 커널로 내려가지 않습니다.

다만 "Cilium이 route add를 아예 안 한다"는 서술은 틀립니다. 정확히는 이렇게 갈립니다.

| 대상 | Cilium이 FIB에 넣는가 |
| --- | --- |
| 로컬 노드에 붙은 파드 (veth, cilium_host) | 넣는다 — netlink로 직접 설치 |
| Native routing의 per-node 경로 (`autoDirectNodeRoutes`) | 설정에 따라 넣는다 |
| **BGP로 수신한 원격 경로** | **넣지 않는다** |

커뮤니티에서 `injectReceivedRoutes` 필드를 추가하자는 제안(cilium/cilium#31091)이 있었으나, 연결된 PR이나 브랜치 없이 닫혔습니다. 현재 v2 CRD 스펙(`CiliumBGPClusterConfig`, `CiliumBGPAdvertisement`, `CiliumBGPPeerConfig`)에도 해당 필드가 없습니다. **helm 값으로 켤 수 있는 옵션이 아닙니다.**

## 그런데 우리는 Geneve 터널 모드다

여기서 앞의 논의 전제가 무너졌습니다.

!!! important "터널 모드에서는 원격 PodCIDR이 커널 FIB에 없어도 정상이다"
    파드→파드 트래픽은 eBPF가 `ipcache` BPF 맵을 보고 "이 파드 IP는 어느 노드 뒤에 있다"를 판단해 그 노드 IP로 **캡슐화**해 보냅니다.
    커널 FIB는 캡슐화된 패킷의 **바깥쪽 목적지(노드 IP)** 만 라우팅하면 되고, 안쪽 파드 IP는 알 필요가 없습니다.

따라서 "다른 노드는 route가 자동으로 된 것 같다"는 관찰은, 사실 라우팅 테이블에 뭔가 자동으로 채워진 게 아니라 **애초에 PodCIDR 단위 커널 라우팅이 필요 없는 정상 상태**였을 가능성이 높습니다.

즉 질문을 바꿔야 합니다.

> ~~"왜 이 노드만 라우팅 정보가 없었나"~~
> **"왜 이 노드만 캡슐화 경로를 못 타고 커널 라우팅으로 폴백했나"** 가 맞는 질문이다.

## 캡슐화 경로를 못 타는 경우

### 1. DSR dispatch 설정 불일치

공식 문서에 따르면 DSR의 기본 dispatch 방식(IP 옵션)은 **Native-Routing 전용**입니다.

> DSR with Geneve 방식은 라우팅 모드가 Geneve 캡슐화든 Native-Routing이든 상관없이 사용할 수 있다. 반면 기본 dispatch 방식은 Native-Routing으로 배포되어야 하며 캡슐화 모드에서는 동작하지 않는다.

`routingMode: tunnel` + `dsrDispatch: opt` 조합이면 응답 경로가 캡슐화를 건너뛰고 **생짜 커널 라우팅으로 나갈 수 있습니다.**

### 2. nativeRoutingCIDR에 원격 대역이 포함된 경우

클러스터 간 트래픽만 별도로 native routing으로 빠지도록 구성했다면(WAN 구간 MTU · 오버헤드 회피 목적), 클러스터 간은 순수 라우팅이 되어 **진짜로 PodCIDR 단위 커널 라우팅이 필요**해집니다. 이 경우에만 BGP 전파 문제가 진짜 원인이 됩니다.

### 3. ipcache 동기화 지연 — "특정 노드만"을 가장 잘 설명

BPF 캡슐화가 동작하려면 각 노드의 ipcache에 "파드 IP → 노드 IP" 매핑이 있어야 합니다. ClusterMesh는 이걸 원격 etcd 동기화로 채웁니다.

특정 노드의 agent만 watch가 끊기거나 지연되면 **그 노드만 캡슐화 불가 → 커널 폴백 → EHOSTUNREACH**.

증상(특정 노드만, 신규 대역만, 간헐적)과 가장 잘 맞는 설명입니다.

### 4. agent 재시작 타이밍

agent는 **기동 시점의 ClusterMesh 상태를 스냅샷처럼** 반영합니다. 신규 PodCIDR 할당 전후로 재시작 이력이 갈리면 노드별 반영 여부가 달라집니다.

## 업그레이드 롤아웃이 노드별 편차를 만들 수 있다

"업그레이드로 재배포됐으면 자동으로 동기화됐어야 하는 거 아닌가"라는 질문에 대한 답입니다.

재시작은 **그 순간의 스냅샷**을 다시 받아오는 것이지, 앞으로도 최신 상태를 유지해주겠다는 보장이 아닙니다. 더구나 DaemonSet 롤링 업그레이드는 190노드를 동시에 재시작하지 않고 순차적으로 돌기 때문에, 신규 PodCIDR 할당이 롤아웃 도중에 일어났다면 이렇게 갈립니다.

- 할당 **이후에** 재시작 차례가 온 노드 → 신규 대역까지 받아옴 → "자동으로 된 것처럼" 보임
- 할당 **이전에** 재시작을 마친 노드 → 그 시점 상태로 멈춤 → 신규 대역 누락 → **증상 재현**

무작위처럼 보이는 노드별 편차가 사실은 **롤아웃 순서와 신규 대역 할당 타이밍의 교차점**이었을 수 있습니다.

## 진단 순서

| # | 확인 대상 | 명령 |
| --- | --- | --- |
| 1 | DSR dispatch / 라우팅 모드 정합성 | `cilium config view \| grep -E 'dsr\|routing-mode\|tunnel'` |
| 2 | nativeRoutingCIDR에 원격 대역 포함 여부 | `cilium config view \| grep native-routing-cidr` |
| 3 | ipcache 엔트리 유무 (문제 / 정상 노드 비교) | `cilium bpf ipcache list \| grep <원격 pod IP>` |
| 4 | ClusterMesh 연결 상태 | `cilium-dbg status --all-clusters` |
| 5 | agent 재시작 시각 vs 신규 대역 할당 시각 | `kubectl -n kube-system get pods -l k8s-app=cilium -o wide` |
| 6 | 기존 경로의 출처 추정 | `ip -d route show <CIDR>` — proto 필드 확인 |

## helm values 확인과 변경

원본 values.yaml보다 **실제 적용된 값**을 보는 게 정확합니다. helm upgrade가 누적되면 둘이 달라질 수 있습니다.

```bash
helm get values cilium -n kube-system --all
cilium config view | grep -E 'dsr|routing-mode|native-routing-cidr|tunnel'
kubectl -n kube-system get cm cilium-config -o yaml | grep -E 'dsr|routing-mode|native-routing-cidr'
```

관련 설정 항목:

```yaml
routingMode: tunnel          # native | tunnel
tunnelProtocol: geneve       # vxlan | geneve

loadBalancer:
  mode: dsr                  # snat | dsr | hybrid
  dsrDispatch: geneve        # opt | ipip | geneve
  # opt 는 routingMode: native 에서만 동작

ipv4:
  nativeRoutingCIDR: ""      # 터널 순수모드면 보통 비어있어야 정상
```

변경 시 `--reuse-values` 를 반드시 붙입니다. 안 붙이면 기존 커스텀 값이 기본값으로 리셋됩니다.

```bash
helm upgrade cilium cilium/cilium -n kube-system --reuse-values \
  --set loadBalancer.dsrDispatch=geneve
kubectl -n kube-system rollout restart ds/cilium
```

190노드 전체에 영향을 주는 변경이므로 **카나리 검증 후 전체 반영**을 권합니다.

## 강제 재동기화 방법

ipcache 동기화가 원인이라면 **agent 재시작이 사실상 유일한 방법**입니다.

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide | grep <문제노드>
kubectl -n kube-system delete pod <해당 cilium 파드>
```

!!! danger "주의"
    - **노드 단위로만** 합니다. `rollout restart ds/cilium` 은 190노드가 동시에 원격 etcd에 재수립을 시도하게 됩니다.
    - **clustermesh-apiserver는 재시작하지 않습니다.** 과거 버전에서 반대편 클러스터의 모든 agent가 엔드포인트 업데이트를 못 받는 이슈가 있었습니다. 블래스트 레이디어스가 너무 큽니다.

## 수동 route add 흔적 추적

bash history에 안 남았을 때 확인 순서입니다.

| 방법 | 명령 | 비고 |
| --- | --- | --- |
| sudo 로그 | `grep route /var/log/auth.log*` 또는 `journalctl _COMM=sudo \| grep -i route` | history와 별개로 남음 — **가장 유력** |
| 열린 세션 | `who`, `w` | history는 세션 종료 시 flush됨 |
| HISTFILE 위치 | `echo $HISTFILE`, `echo $HISTCONTROL` | root vs 원래 계정 둘 다 확인 |
| process accounting | `lastcomm ip` | psacct / acct가 사전 활성화된 경우만 |
| 경로 출처 추정 | `ip -d route show <CIDR>` | proto 값으로 간접 판단 |
| 영구화 파일 | netplan / network-scripts / systemd-networkd 수정 시각 | 재부팅 대비로 반영했다면 |
| 자동화 로그 | Ansible / AWX / Jenkins 실행 이력 | 노드가 아니라 컨트롤 노드 쪽 |

`ip -d route show` 의 proto 값 해석:

- `proto boot` — `ip route add` 를 옵션 없이 수동 실행했을 때의 기본값. **수동 조치 정황 증거**
- `proto static` — 명시적 static 지정 또는 설정관리 도구가 설치
- `proto bgp` / `proto zebra` — FRR / BIRD 등 라우팅 데몬이 설치 (사람이 아님)
- `proto kernel` — 인터페이스 설정 시 커널이 자동 생성

앞으로를 위해서는 **auditd 규칙을 프로비저닝(Kubespray 플레이북)에 포함**시키는 게 맞습니다.

```bash
auditctl -a always,exit -F arch=b64 -S execve -F path=/usr/sbin/ip -k route_changes
# 영구 적용은 /etc/audit/rules.d/ 에 규칙 파일로 저장
# 이후 ausearch -k route_changes 로 추적
```

## 결론

- **RIB와 FIB는 다르다.** Cilium BGP는 RIB까지만 채우고 FIB로 내려보내는 기능은 없다. helm 옵션으로 켜지지 않는다.
- **다만 우리는 Geneve 터널 모드이므로 원래 PodCIDR 단위 커널 라우팅이 필요 없다.** BGP 전파 문제로 결론을 내리기 전에 이 전제를 먼저 확인해야 한다.
- 수동 `route add` 는 **폴백 경로를 메운 것**이지 정상 경로를 복구한 것이 아니다. 증상은 사라졌지만 원인은 그대로다.
- **재시작은 그 순간의 스냅샷일 뿐** 지속적 동기화 보장이 아니다. 롤링 업그레이드 순서가 노드별 편차를 만들 수 있다.
- 진단은 **설정(dsrDispatch, nativeRoutingCIDR) → 상태(ipcache, ClusterMesh) → 타이밍(agent 재시작 시각)** 순서로 좁힌다.
