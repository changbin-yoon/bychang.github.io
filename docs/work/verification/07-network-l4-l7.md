---
title: 07. 트래픽 분산 경로 검증 (L4 vs L7)
---

# 🌐 네트워크 — 트래픽 분산 경로 검증 (L4 vs L7)

!!! abstract "개요"
    **목적** — 128노드 클러스터에서 특정 5노드만 네트워크가 급증하는 현상의 원인 경로를 특정

    **경로** — client → BGP LB → (nginx ingress) → MinIO

    **결론 한 줄** — 진입 경로의 분배 문제가 아니라 **목적지 측 요인**(pod 배치 또는 데이터 소유권)이 원인이며, S3 대용량 트래픽을 L7로 통과시키는 구성은 그 자체가 별도 개선 대상

## 1. 확인하려는 것

- BGP ECMP 해싱이 트래픽을 고르게 나누는가
- Service의 traffic policy 설정이 쏠림을 만드는가
- S3 API 트래픽을 nginx L7으로 보내는 게 적절한가

## 2. 구성과 경로

!!! important "ECMP의 성질"
    ECMP는 stateless하게 5-tuple 해시로 넥스트홉을 고릅니다. 즉 **연결 개수를 나누지 바이트를 나누지 않습니다.**
    적은 수의 무거운 연결(= S3 대용량 전송)이라면 **불균형은 정상**입니다.

**정책 확인 결과**

| 설정 | 값 | 해석 |
| --- | --- | --- |
| `internalTrafficPolicy` | Cluster | 로컬 우선 아님 → 로컬리티발 쏠림 배제 |
| `externalTrafficPolicy` | 미지정(=Cluster) | 모든 노드가 VIP 광고 → 정책발 쏠림 배제 |

정책이 쏠림을 만들지 않는다는 게 확인되면서 원인이 **목적지 쪽으로 좁혀졌습니다.**

!!! warning "Cluster 정책의 부작용"
    정책을 배제하는 대신 **분석을 흐리게 합니다.** SNAT + 포워딩 때문에
    ① 진입 노드와 처리 노드 양쪽에서 트래픽이 **이중 카운팅**되고,
    ② 클라이언트 IP가 노드 IP로 바뀝니다.

## 3. 가설 소거 순서

명령 하나로 대부분이 갈립니다.

```bash
# 1순위 — MinIO pod가 그 5노드에만 있는가?
kubectl get pods -l app=minio -o wide

# ingress controller 배치와 겹치는가
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx <controller-svc> -o jsonpath="{.spec.externalTrafficPolicy}"

# 플로우 실시간 확인
hubble observe
```

| 순위 | 가설 | 판별 조건 |
| --- | --- | --- |
| 1 | MinIO pod가 그 5노드에만 존재 | pod 배치 = 뜨거운 노드 → 정상, "용량 충분한가"로 문제 재정의 |
| 2 | hot object / erasure set 쏠림 | 요청 수는 고른데 **바이트만** 쏠림 |
| 3 | healing / rebalance / decommission | 클라이언트 트래픽과 무관하게 상시 높음 |
| 4 | externalTrafficPolicy: Local | 미지정 확인 → **배제됨** |
| 5 | ECMP 해시 쏠림 | 두 경로가 같은 5노드로 겹치므로 단독 원인 가능성 낮음 |

관련 메트릭:

```text
rate(minio_node_if_rx_bytes[5m])                  # by (server) — 어느 서버가 뜨거운가
rate(minio_bucket_traffic_received_bytes[5m])     # by (bucket) — 특정 버킷 쏠림
minio_s3_requests_inflight_total
```

## 4. L4 vs L7 — 구조적 검토 사항

S3 트래픽은 오브젝트 본문이 큽니다. 그래서 **nginx 부하는 독립 변수가 아니라 MinIO 트래픽의 거울**입니다. MinIO가 5노드에 쏠려 뜨거우면 그걸 프록시하는 nginx도 같은 규모로 뜨겁습니다.

다만 그것과 별개로, 대용량 S3 스트림을 L7 http 프록시로 통과시키는 구성은 **버퍼링 · 헤더 처리 · TLS 종단 오버헤드**가 큽니다. MinIO 권장도 S3 API는 **L4 로드밸런싱**(또는 nginx `stream` 모듈 TCP 패스스루)입니다.

확인해야 할 것 — **S3 API가 정말 nginx L7을 거치는가, 아니면 콘솔 UI만 nginx고 데이터는 LB 직결인가.** 전자라면 5노드 쏠림 해결과 무관하게 **아키텍처를 L4 패스스루로 바꾸는 게 더 근본적인 개선**입니다.

## 5. 부가 — 클라이언트 커넥션 설정

| 클라이언트 | 설정 | 비고 |
| --- | --- | --- |
| Hadoop S3A | `fs.s3a.connection.maximum` | 버전별 기본값 편차 큼 (수십~100 수준) |
| Trino | `s3.max-connections` | 네이티브 S3 파일시스템 기준 기본 500 |

또 하나 확인한 점 — **Spark가 IPv6 우선이고 MinIO가 IPv4면 DNS AAAA 우선 조회 단계부터 에러**가 납니다. 혼재 환경에서는 JVM의 주소 계열 우선순위를 명시적으로 맞춰야 합니다.

## 6. 정리

| 항목 | 확인 결과 |
| --- | --- |
| ECMP 분산 | 연결 단위 해싱 — 소수의 무거운 연결은 불균형이 정상 |
| traffic policy | 둘 다 Cluster → 정책발 쏠림 아니지만 SNAT로 분석은 흐려짐 |
| 유력 원인 | MinIO pod 배치 또는 hot object / erasure set 소유권 |
| L7 프록시 | S3 데이터 경로라면 L4 패스스루로 전환 권장 |

**운영에 반영할 것**

1. "특정 노드 네트워크 급증" 알람을 받으면 경로보다 **pod 배치부터** 확인한다.
2. **요청 수와 바이트를 분리해서** 본다. 둘의 분포가 다르면 hot object다.
3. S3 API 경로에서 **L7 프록시를 걷어내는 것을 중기 과제**로 둔다.
