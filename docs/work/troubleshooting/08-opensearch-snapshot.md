---
title: 08. OpenSearch 스냅샷 리포지토리 운영
---

# 🗃️ 연동 시스템 — OpenSearch 스냅샷 리포지토리 운영

!!! abstract "개요"
    **목적** — MinIO를 스냅샷 리포지토리로 쓰는 OpenSearch에서 스냅샷이 계속 쌓이기만 하는 원인과 안전한 정리 방법 확인

    **결론 한 줄** — 정리는 반드시 `DELETE _snapshot` API로만 해야 하며, **버킷에서 오브젝트를 직접 지우면 리포지토리 전체가 복구 불가**가 된다

## 1. 왜 안 줄어드는가

### 원인 1 — 보존 정책 부재 (가장 흔함)

OpenSearch는 스냅샷을 자동으로 지우지 않습니다. 생성만 cron이나 SM 정책 creation으로 걸어두고 deletion 조건이 없으면 무한히 쌓입니다.

```text
GET _plugins/_sm/policies
GET _plugins/_sm/policies/<policy_name>/_explain
```

SM 정책은 **자기가 만든 스냅샷만 관리**합니다. 외부 스크립트나 수동으로 만든 것은 정책 대상이 아니므로 별도 정리가 필요합니다.

### 원인 2 — 증분 구조

스냅샷은 증분이라 삭제해도 다른 스냅샷이 참조 중인 세그먼트 파일은 남습니다. 오래된 것 몇 개 지웠는데 용량이 거의 안 줄었다면 이 때문입니다. 반면 장점도 있습니다 — **오래된 것부터 지워도 최신 스냅샷은 안전**합니다.

### 원인 3 — MinIO 버킷 versioning

versioning이 켜져 있으면 OpenSearch가 오브젝트를 지워도 delete marker만 생기고 이전 버전이 용량을 점유합니다.

```bash
mc version info <alias>/opensearch-snapshots
mc ilm rule ls <alias>/opensearch-snapshots

# noncurrent version 즉시 만료 룰 추가
mc ilm rule add <alias>/opensearch-snapshots --noncurrent-expire-days 1
```

## 2. 정리 절차

### 2.1 현황 파악

```text
GET _snapshot
GET _cat/snapshots/<repo>?v&s=end_epoch
GET _snapshot/<repo>/_current      # 진행 중(IN_PROGRESS) 확인 — 건드리지 말 것
```

### 2.2 삭제

이름에 날짜 패턴이 있으면 와일드카드로 한 번에:

```text
DELETE _snapshot/<repo>/snapshot-2026.05*
```

개수가 많으면 최신 N개만 남기는 스크립트로:

```bash
REPO="my-repo"
KEEP=30
HOST="https://opensearch:9200"

curl -sk -u admin:$PASS "$HOST/_cat/snapshots/$REPO?h=id&s=end_epoch" \
  | head -n -$KEEP \
  | while read snap; do
      echo "deleting $snap"
      curl -sk -u admin:$PASS -X DELETE "$HOST/_snapshot/$REPO/$snap"
    done
```

**제약 조건**

- 삭제는 한 번에 하나씩 순차 처리됩니다. 병렬로 날리면 `concurrent_snapshot_execution_exception`.
- 삭제 중에는 해당 리포지토리에 신규 스냅샷 생성이 블록됩니다 → 생성 cron 시간대를 피합니다.

### 2.3 재발 방지 — SM 정책

```json
POST _plugins/_sm/policies/snapshot-policy
{
  "creation": {
    "schedule": { "cron": { "expression": "0 2 * * *", "timezone": "Asia/Seoul" } }
  },
  "deletion": {
    "schedule": { "cron": { "expression": "0 4 * * *", "timezone": "Asia/Seoul" } },
    "condition": { "max_age": "14d", "min_count": 7 }
  },
  "snapshot_config": { "repository": "my-repo", "indices": "*" }
}
```

- 기존에 creation만 있는 정책이 있다면 새로 만들지 말고 그 정책에 deletion 블록을 추가합니다.
- creation과 deletion cron은 겹치지 않게 잡습니다.

## 3. 경고

!!! danger "MinIO 버킷에서 오브젝트를 직접 지우면 안 된다"
    리포지토리 메타데이터가 깨져서 **남은 스냅샷 전체가 복구 불가**가 될 수 있습니다. 용량이 급해도 반드시 API를 통합니다.

부가 사항 — 스냅샷이 수백 개 이상 쌓이면 리포지토리 메타데이터가 커져서 생성 · 삭제 자체가 느려집니다. 보존 정책은 용량뿐 아니라 성능 문제이기도 합니다.

## 4. 정리

| 항목 | 결과 |
| --- | --- |
| 자동 삭제 | 없음 — SM deletion 블록 명시 필요 |
| 삭제 방식 | `DELETE _snapshot/<repo>/<name>` 순차 처리만 |
| 용량 미감소 원인 | 증분 세그먼트 공유 또는 버킷 versioning |
| 정책 관리 범위 | SM이 생성한 스냅샷만 |
| 금지 항목 | 버킷에서 오브젝트 직접 삭제 |
