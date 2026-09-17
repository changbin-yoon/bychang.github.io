---
title: 02. API 서버 부하 & Watch 레이스 컨디션
---

# ⚡ API 서버 부하 & Watch 기반 레이스 컨디션 — Volume Mount 실패 알람

!!! abstract "요약"
    반복적으로 뜨는 `MountVolume.SetUp failed ... not found` 알람 두 종류를 근본 원인부터 알람 필터링 전략까지 소스 레벨로 검증한 기록.

## 핵심 결론 먼저

이 알람은 버그가 아니라 **watch 기반 비동기 아키텍처가 내장한 레이스 컨디션**입니다.

apiserver는 수동적 저장소일 뿐이고(YuniKorn에 "리소스 있어?"라고 묻지 않습니다), Spark Operator → YuniKorn → kubelet → driver가 각자 watch로 반응하는 구조라 **"삭제 전파"가 "마운트 시도"보다 늦게 도착하는 구간이 구조적으로 존재**합니다.

## 두 에러는 발생 지점이 다르다

| 에러 패턴 | 정체 | 취급 |
| --- | --- | --- |
| `token not found` (`tg-` prefix) | **RACE #1** — placeholder pod 삭제 레이스. Gang 충족 후 즉시 삭제되는데 kubelet은 아직 TokenRequest 재시도 중 | 노이즈 — 필터 |
| `conf-map not found`, driver 삭제됨 | **RACE #2-B** — driver 삭제 → ownerReference GC 연쇄 삭제 → 잔여 executor가 참조 | 노이즈 — 필터 |
| `conf-map not found`, **driver Running** 지속 | 수렴 실패 | **실제 장애 — 알람 필요** |
| 429 / timeout | 요청 자체가 apiserver에서 막힘(APF) | 별도 트랙 — QPS / burst 조치 |

!!! important "판별 기준"
    **에러 메시지가 아니라 대상의 운명**입니다.
    죽기로 예정된 리소스의 잔여 참조인가, 살아있는 워크로드의 수렴 실패인가.

## 8단계 이벤트 흐름

```text
SparkApplication 제출 → apiserver 저장(수동적) → YuniKorn watch 감지
  → placeholder(tg-) 일괄 생성 → kubelet DSW에 마운트 등록
  → kubelet TokenRequest → YuniKorn bind + placeholder 즉시 삭제   [RACE #1]
  → driver 기동, conf-map 생성(ownerRef=driver) → executor 생성
  → driver 종료 시 GC 연쇄로 conf-map 삭제 → 잔여 executor 참조 실패 [RACE #2-B]
```

## apiserver 부하와의 관계

`not found` 는 요청이 apiserver에 정상 도달해 **올바르게 거부된 것**이지 부하 문제가 아닙니다. 부하(throttling)라면 429 / timeout으로 나타나야 합니다.

다만 부하로 레이스 윈도우 자체가 벌어지면 비선형적으로 폭증할 수 있으므로, **그 경우에만** APF 메트릭(`apiserver_flowcontrol_rejected_requests_total`)과 kubelet client-side throttling 로그로 근거를 잡고 QPS / FlowSchema를 조정합니다.

## 알람 조치 우선순위

1. **즉시 — 필터링** — OpenSearch 모니터에서 `token not found` + `tg-` 패턴, `conf-map not found` + driver 삭제됨 패턴 제외. bucket-level monitor로 동일 pod 반복(stale state) 감지는 유지
2. **보완 알람** — `ContainerCreating` 5분+ 정체 알람으로 **진짜 마운트 장애**를 별도 커버
3. **발생량 감소(선택)** — `gangSchedulingStyle=Soft`, `placeholderTimeoutInSeconds` 실측 기반 상향, executor `terminationGracePeriodSeconds: 10` (conntrack 이슈와 동시 완화 — [01번 문서](01-yunikorn-conntrack.md) 참고)
4. **부하 튜닝(근거 확보 시에만)** — kubelet `kubeAPIQPS` 상향 → apiserver inflight → Spark 트래픽 FlowSchema 격리 순

## apiserver 인증서 만료 알람 (별개 사례)

Prometheus Agent 모드 환경(`PrometheusAgent` CR)에서 `apiserver_client_certificate_expiration_seconds_*` 관련 알람을 진단한 기록입니다.

- 이 히스토그램은 **성공적으로 인증된** client cert의 남은 유효기간만 기록합니다. 이미 만료된 cert는 TLS handshake 단계에서 거부되어 **이 메트릭 자체가 증가하지 않습니다.**
- `sum(apiserver_client_certificate_expiration_seconds_bucket{le="86400"})` 가 0보다 크면 **만료 임박 cert로 지금도 인증이 이뤄지고 있다는 직접 증거** — 대개 갱신하지 않은 kubeconfig가 원인입니다.
- `kubeadm certs renew` 는 컨트롤 플레인 leaf cert만 갱신하고 **개인 kubeconfig는 자동 갱신하지 않습니다.** `admin.conf` 복사본이나 `kubeadm kubeconfig user` 로 발급한 개인 kubeconfig는 별도 재발급이 필요합니다.

## 결론

watch 기반 아키텍처에서 이런 레이스는 **없애는 대상이 아니라**, "죽은 대상의 잔여 참조"와 "살아있는 워크로드의 수렴 실패"를 **구분하는 알람 룰로 관리**하는 게 근본 조치입니다.
