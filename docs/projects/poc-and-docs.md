---
title: 신기술 조사 · PoC · 문서화
---

# 신기술 조사, PoC, 문서화

| 항목 | 내용 |
| --- | --- |
| 기간 | 2024 ~ 2026 |
| 역할 | 신기술 비교 검증, 정책 적용, 가이드 문서화 및 사내 발표 |
| 핵심 기술 | Gravitino, Apache Polaris, Iceberg, OPA/Gatekeeper, Cilium |

## PoC

- **Gravitino / Polaris Iceberg 카탈로그 PoC**
- **Gatekeeper / OPA 정책 적용** (Trino)
- **Sealed Secrets** 도입
- **Cilium v1.18.12 → v1.19.6 업그레이드 영향도 검증**
    - Pod 간 통신이 일시적으로 중단될 수 있으나, CNI 복구 시점에 에러는 해소됨
    - 핵심 쟁점은 **허용 가능한 downtime 산정** — downtime이 길수록 Job 지연이 누적되어 처리량 부하가 증가

## 문서화 / 발표

- **Lakehouse 공청회 발표자료** 작성
- **AIStor 설치 런북 · 가이드북** 제작
- **Kubernetes 클러스터 사용 가이드** 작성
