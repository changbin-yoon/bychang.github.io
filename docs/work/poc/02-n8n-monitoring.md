---
title: 02. n8n 기반 K8s 모니터링 자동화
---

# 🤖 n8n 기반 Kubernetes 모니터링 자동화

!!! abstract "요약"
    Prometheus 메트릭을 n8n 멀티에이전트 워크플로우로 수집·분석해 이메일 리포트로 발송하는 시스템을 구축한 기록.

## 아키텍처

다섯 개 AI 에이전트 구조입니다. 4개 영역 전문가(infra / kubernetes / devops / service)가 병렬 분기하고, 총괄 관리자 에이전트가 4개 보고서를 종합해 최종 메일을 생성합니다.

```text
Schedule Trigger → [4개 병렬 분기]
  각 분기: PromQL 빌더(Code) → HTTP Request(Prometheus) → 이상탐지(Code) → AI Agent
→ Merge(Append, 4-input 동기화 게이트)
→ Chief Manager Agent(종합) → HTML 렌더러(Code) → Send Email
```

## 핵심 설계 결정

**1) 숫자는 AI가 아니라 고정 스크립트가 담당한다**

HTML 렌더링은 원시 메트릭값만으로 결정론적 Code 노드가 수행하고, **AI 출력에서 숫자를 가져오지 않습니다** — hallucination으로 숫자가 왜곡되는 것을 구조적으로 차단하기 위함입니다.

**2) node-exporter instance(IP:port) ↔ 노드명 매핑**

`node_uname_info` 와 `kube_node_role` 을 `label_replace` 로 조인해서 노드명을 하드코딩 없이 마스터 / 워커로 구분합니다.

**3) 1시간 윈도우 서브쿼리**

모든 메트릭을 `avg_over_time` / `max_over_time` + `[1h:5m]` 스텝으로 단일 요약값으로 산출합니다.

**4) 서비스 도메인 우선 설계**

Spark 실패 driver 탐지를 오퍼레이터 네이티브 메트릭이 아니라 **pod 라벨(`label_spark_role="driver"`) 기준**으로 구현했습니다 — 오퍼레이터 버전에 덜 종속적이라 더 안정적입니다.

## n8n 노드 모델에서 주의할 점 — Merge vs 일반 Code 노드

4개 분기를 일반 Code 노드에 직접 연결하면 안 됩니다. Code 노드는 도착하는 input마다 **1번씩 따로** 실행되어 4번 실행됩니다(동기화 안 됨).

반드시 **Merge 노드(Append 모드, 4-input)** 를 써야 모든 분기를 기다렸다가 하나로 묶을 수 있습니다. 반대로 단일 Schedule Trigger에서 4개 분기로 펼칠 때는 별도 노드 없이 출력 점에서 연결선 4개만 그으면 됩니다.

## 외부(클러스터 밖) 배포 버전

n8n을 Docker(Ubuntu)로 클러스터 외부에 띄우는 대안도 적용했습니다. 이 경우 PromQL 대신 **kubectl 기반 수집**으로 전환합니다.

- kubectl / jq가 포함된 커스텀 이미지 빌드 필요 (n8n 2.x는 distroless 기반이라 multi-stage build 필요)
- Execute Command 노드는 n8n 2.x에서 기본 비활성화되어 있으며 `NODES_EXCLUDE=[]` 로 활성화해야 함

## 결론

핵심 원칙은 **"숫자는 Prometheus / Code가 책임지고, AI는 서술만, 그래프 · HTML은 고정 스크립트가 그린다"** 입니다 — 보고 신뢰성을 위해 AI의 역할을 의도적으로 좁힌 구조입니다.
