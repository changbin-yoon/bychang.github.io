---
title: 04. CNPG & Cilium 업그레이드 영향
---

# 🐘 CNPG & Cilium 업그레이드 영향 — probe fencing

!!! abstract "요약"
    Cilium 롤링 업그레이드 중 CNPG(CloudNativePG) 파드가 재시작된 문제를 진단하고 대응한 기록.
    자세한 Cilium 업그레이드 절차는 [Kubernetes Infra 03번 문서](../kubernetes-infra/03-cilium-upgrade.md)를 참조.

## 문제 현상

이전 Cilium 업그레이드 진행 중 **CNPG pod이 예상치 못하게 재시작**되었습니다.

## 원인

CNPG instance manager의 liveness probe가 **API 서버 isolation check** 를 포함합니다.

kube-proxy replacement(Cilium) 환경에서는 ClusterIP 해석이 Cilium eBPF에 전적으로 의존하는데, **Cilium agent가 롤링 업그레이드로 재시작되는 짧은 순간** 이 isolation check가 실패하고, kubelet이 이를 장애로 판단해 컨테이너를 재시작합니다.

즉 문제는 CNPG 자체가 아니라, **Cilium agent 재시작이라는 정상 이벤트가 CNPG의 엄격한 isolation probe와 충돌**한 것입니다.

## 대응 — probe 완화 또는 fencing

Cilium 업그레이드 런북에 포함한 CNPG 전용 조치입니다.

1. **D-1 사전 조치** — CNPG probe 완화(타임아웃 / 임계치 여유 확보) 또는 CNPG 노드에 대한 fencing 절차 준비
2. **롤링 당일** — CNPG 노드가 롤링 대상이 되기 전에 fencing / switchover를 먼저 수행하고, 통과 후 롤링 항목을 해제
3. Cilium과 CNPG 양쪽 Deployment / StatefulSet이 에이전트와 버전 일치하는지 롤링 진행 중 확인

## 결론

kube-proxy replacement 환경에서는 **CNI 재시작이 단순히 네트워크만이 아니라 ClusterIP 해석 경로 자체에 영향을 준다**는 점이 핵심입니다. isolation check가 들어간 probe를 쓰는 서비스(CNPG 등)는 CNI 롤링 시 별도 보호 조치가 필요합니다.
