---
title: 05. 구성 컴포넌트 — DirectPV·Sidekick
---

# 🧱 구성 컴포넌트 — DirectPV · Sidekick 배치 방식

!!! abstract "개요"
    **목적** — 스토리지 주변 컴포넌트(DirectPV, Sidekick)가 데이터 경로에 어떤 영향을 주는지 확인하고 배치 방식을 결정

    **결론 한 줄** — DirectPV는 마운트 이후 데이터 I/O 경로 밖에 있어 pod 재시작이 read/write를 끊지 않으며, Sidekick은 사이드카 대신 DaemonSet + `internalTrafficPolicy: Local` 이 깔끔했다

## 1. DirectPV — 역할과 경계

DirectPV는 DAS(직결 디스크)를 XFS로 포맷해 PersistentVolume으로 공급하는 **CSI 드라이버**입니다.

### 1.1 구성

| 요소 | 형태 | 역할 |
| --- | --- | --- |
| CSI Controller | Deployment (기본 replica 3) | 볼륨 프로비저닝 조율 |
| Node Server | DaemonSet | 드라이브 발견 · 포맷 · 마운트 |
| StorageClass | `directpv-min-io` | `WaitForFirstConsumer` 바인딩 |

### 1.2 드라이브 초기화 절차

```bash
kubectl directpv discover          # 드라이브 스캔 → drives.yaml 생성
# drives.yaml 검토 — 제외할 드라이브 확인 필수
kubectl directpv init drives.yaml  # XFS 포맷 + 마운트
kubectl directpv list drives
```

### 1.3 핵심 확인 사항

**DirectPV는 실제 데이터 I/O 경로에 있지 않습니다.** 프로비저닝과 관리만 담당하고, 볼륨이 한번 마운트되면 MinIO pod는 로컬 디스크에 직접 접근합니다. 그래서:

- DirectPV pod 재시작 → MinIO read/write 중단 없음
- DirectPV로 인한 성능 레이턴시 가산 없음
- 예외는 마운트 자체가 유실되는 경우뿐

이 사실 덕에 DirectPV 업그레이드 / 재기동을 서비스 중단 없이 진행할 수 있다고 판단했습니다.

### 1.4 버전 라이프사이클 — 반입 전 확인

커뮤니티 OSS 브랜치는 순차적으로 유지보수 모드로 전환되어 왔고(v4.0.x는 2025-01-01, v4.1.x는 2026-01-01), 엔터프라이즈 라인은 AIStor 브랜드로 날짜 기반 버저닝을 씁니다. 에어갭 미러에 반입할 버전을 고를 때 확인해야 합니다.

## 2. Sidekick — 배치 방식 결정

### 2.1 문제 상황

Spark driver가 Completed가 된 뒤에도 Sidekick 사이드카 컨테이너가 종료되지 않아 pod가 잔류했습니다. 원인은 단순합니다 — `spec.containers` 에 들어간 일반 사이드카는 메인 컨테이너가 끝나도 Kubernetes가 자동으로 종료시키지 않습니다.

### 2.2 1차 해결 — 네이티브 사이드카

```yaml
initContainers:
  - name: sidekick
    image: quay.io/minio/sidekick:v7.0.1
    restartPolicy: Always   # 이 필드가 네이티브 사이드카로 만든다
```

`initContainers` + `restartPolicy: Always` 조합이면 driver 종료 시 Kubernetes가 사이드카에 SIGTERM을 보냅니다. 동작은 확인됐지만, 잡마다 Sidekick 인스턴스가 뜨는 구조라는 점은 그대로입니다.

### 2.3 최종 구성 — 독립 DaemonSet

Sidekick을 pod 밖으로 빼서 노드당 1개로 운영하고, Service로 붙였습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-sidekick
spec:
  internalTrafficPolicy: Local   # 같은 노드 pod로만 라우팅
  ports:
    - port: 9000
```

```text
spark.hadoop.fs.s3a.endpoint=http://minio-sidekick.minio.svc.cluster.local:9000
```

- driver / executor spec에서 `sidecars` 블록을 전부 제거합니다.
- `internalTrafficPolicy: Local` 덕분에 hostPort나 hostNetwork 없이도 데이터 로컬리티가 유지됩니다.
- MinIO와 Spark 노드가 분리된 구성이라면 `nodeSelector` 로 executor 노드에만 배포합니다.

### 2.4 헷갈리는 두 엔드포인트

| 옵션 | 의미 |
| --- | --- |
| `--health-path` | Sidekick이 **백엔드 MinIO 노드**를 프로브할 경로 |
| `/v1/health` | **Sidekick 자신의** 헬스 엔드포인트 (k8s probe용) |

### 2.5 주의사항

- 롤링 업데이트 중 해당 노드에 pod가 잠시 없으면 `internalTrafficPolicy: Local` 특성상 트래픽이 실패합니다. `maxUnavailable` 과 헬스체크를 보수적으로 잡습니다.
- 백엔드 엔드포인트를 ellipses 문법으로 정확히 지정해야 합니다.

## 3. 정리

| 항목 | 확인 결과 |
| --- | --- |
| DirectPV 위치 | 데이터 I/O 경로 밖 (프로비저닝 전담) |
| DirectPV 재시작 영향 | MinIO read/write 무중단 |
| Sidekick 사이드카 | driver 종료 후 잔류 — 네이티브 사이드카로 해결 가능 |
| 최종 선택 | 독립 DaemonSet + internalTrafficPolicy: Local |
| 데이터 로컬리티 | hostPort/hostNetwork 없이 유지됨 |
