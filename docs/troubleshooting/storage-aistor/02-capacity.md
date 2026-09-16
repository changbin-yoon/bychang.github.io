---
title: 02. 용량/성능 — 고사용률 구간 동작
---

# 📊 용량/성능 — 고사용률 구간 동작과 진단 지표

!!! abstract "개요"
    **목적** — 드라이브 사용률이 권장치(70%)를 크게 넘은 구간에서 무슨 일이 일어나는지, 성능 저하가 제품 동작인지 부수효과인지 구분

    **대상** — 9PB / 약 40억 object / 사용률 94%

    **결론 한 줄** — 90%에서 의도적으로 속도를 낮추는 스로틀링은 **없다**. 저하는 XFS 단편화 · scanner · heal의 부수효과이며, 쓰기는 임계값에서 "완만한 저하"가 아니라 **507로 하드 거부**된다

## 1. 확인하려는 것

- 90% 지점에 제품 차원의 성능 제한이 걸리는가
- 94% 구간에서 관측된 CPU 2배 상승과 응답 지연의 실제 원인
- 네트워크 사용량이 상시 높은 구간의 발원지(빈 pool vs 찬 pool)

## 2. 임계값 동작 — 스로틀링이 아니라 하드 거부

테스트 결과, **90%라는 숫자에 연동된 성능 스로틀링은 존재하지 않았습니다.** 대신 드라이브별 최소 여유 공간을 둘러싼 별도 기준이 있고, 그 기준에 걸리면 점진적 저하가 아니라 **쓰기가 거부**됩니다.

| 구분 | 동작 |
| --- | --- |
| 권장 최대 사용률 | **70%** (장애 시 parity 상향 여유를 위한 헤드룸) |
| 쓰기 거부 | HTTP 507 / `XMinioStorageFull` |
| pool 제외 조건 | 드라이브 사용률 99% 초과 또는 free inode 1000 미만 |
| 90%에서의 스로틀링 | **없음** |

여기서 "헤드룸"은 드라이브가 죽었을 때 남은 조각으로 데이터를 재구성해 다시 써넣을 공간을 뜻합니다. **꽉 채우면 장애 때 복구할 자리가 없습니다.**

## 3. CPU 상승의 인과 체인

우선순위대로 네 가지를 검토했습니다.

1. **XFS free-space 단편화** — 거의 가득 찬 상태에서 extent 할당자가 연속 공간을 찾느라 오버헤드가 커집니다. **가장 유력.**
2. **백그라운드 scanner** — object 40억 규모면 거의 쉬지 않고 도는 수준입니다. 각 erasure set 멤버의 `xl.meta` 를 읽으며 data usage 계산 + ILM 평가 + heal scan을 수행합니다.
3. **heal / rebalance** — near-full pool은 부분 쓰기 실패로 outdated shard가 생기기 쉽고, auto-heal이 노드 간 재구성 트래픽을 지속적으로 만듭니다.
4. **Go GC 압력** — 메타데이터 증가에 따른 부수적 요인.

!!! important "goroutine 급증은 원인이 아니라 결과"
    `minio_system_process_go_routine_total` 급증은 **백프레셔 지표**로 읽어야 합니다.
    디스크 I/O가 느려지면 goroutine이 더 오래 blocked 상태로 머물고 → 누적되면서 스케줄러와 GC가 더 많은 스택을 다루게 돼 CPU가 오르고 → 큐잉된 요청이 latency로 나타납니다.

## 4. 네트워크 사용량 — 어느 pool이 범인인가

다중 pool 환경에서는 트래픽 발원지가 **비어 있는 pool이냐 찬 pool이냐**에 따라 원인이 갈립니다.

- **빈 pool이 뜨거움** → **weighted write.** 확장 후 자동 rebalance를 하지 않고 신규 쓰기를 여유 공간 비율(해당 pool 여유 / 전체 여유)로 분배하므로, 기존 pool이 차면 신규 ingest가 여유 pool로 몰립니다. 찬 pool이 99% / inode 컷오프에 걸리면 쓰기 100%가 전환됩니다.
- **찬 pool이 뜨거움** → heal 압력 또는 scanner 바닥 트래픽.

핵심은 대부분의 drive / erasure-set 메트릭에 **`pool_index` 라벨이 붙어 있어** pool 단위 집계가 가능하다는 점입니다.

```text
# pool별 사용률 / 임계값 근접 여부
minio_system_drive_used_bytes
minio_system_drive_free_bytes
minio_system_drive_free_inodes   # 1000 inode 컷오프 근접 확인

# 노드별 트래픽 (server → pool 매핑 필요)
rate(minio_node_if_rx_bytes[5m])

# scanner / heal 비중 분리
minio_node_scanner_objects_scanned
minio_heal_objects_total
```

보조 명령:

```bash
mc admin info <alias>            # pool/노드별 capacity·상태
mc admin scanner info <alias>    # scanner 진행 상황
mc admin rebalance status <alias>
mc admin trace <alias> --call heal
mc support profile <alias>       # goroutine 프로파일 → go tool pprof
```

## 5. 진단 플로우

1. 노드에서 `mpstat` 로 **iowait 비중** 확인 — iowait가 높으면 디스크 병목, sys가 높으면 GC / 스케줄러 의심
2. XFS 단편화 수치 확인
3. `mc admin scanner info` 로 scanner 사이클 진행 상황 확인
4. heal / rebalance 진행 여부 확인 (진행 중이면 그게 **베이스라인 부하**)
5. `mc support profile` + `go tool pprof` 로 goroutine이 **어디서** blocked인지 분류 — disk I/O vs lock vs internode call
6. 5xx / 429 비율과 `mc admin trace` 로 느린 API 실시간 확인

## 6. 정리

| 항목 | 확인 결과 |
| --- | --- |
| 90% 스로틀링 | 없음 — 임계값에서 507 하드 거부 |
| 권장 사용률 | 70% (parity 상향 헤드룸) |
| CPU 상승 주원인 | XFS 단편화 + scanner + heal → I/O 지연 |
| goroutine 급증 | 백프레셔 증상 (원인 아님) |
| pool 트래픽 판별 | `pool_index` 라벨로 집계 → 빈/찬 pool 구분 |

**운영에 반영할 것**

1. 사용률 알람을 90%가 아니라 **70% 기준으로 내려** 잡고, 확장 리드타임을 확보한다.
2. **free inode 지표를 용량 지표와 같은 비중으로** 감시한다 (컷오프 조건).
3. goroutine 수를 단독 알람으로 쓰지 않고 **iowait · latency와 묶어서** 판단한다.
4. pool 확장 후에는 weighted write로 신규 pool에 부하가 집중되므로, **확장 직후 네트워크 상승은 정상 동작**으로 본다.
