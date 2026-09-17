---
title: 09. DirectPV 공유로 인한 Erasure Set 쓰기 차단
---

# 🚧 용량/스토리지 — DirectPV 드라이브 공유로 인한 Erasure Set 쓰기 차단

!!! danger "한 줄 요약"
    DirectPV로 프로비저닝한 로그 · 메트릭 볼륨이 객체 스토리지 볼륨과 동일 XFS 파일시스템을 공유하면서, 특정 드라이브가 포화되어 해당 드라이브가 속한 Erasure Set으로 배정되는 객체만 HTTP 507로 차단되었다.

| 항목 | 내용 |
| --- | --- |
| 분류 | 스토리지 / 용량 관리 (Capacity Management) |
| 현상 | 객체 스토리지 쓰기 요청의 일부가 HTTP 507 반환 |
| 영향도 | 부분 장애 (Partial Outage). 데이터 유실 없음 |
| 근본 원인 | 단일 파일시스템을 다중 워크로드가 공유하여 발생한 용량 회계 불일치 |
| 조치 | 워크로드별 물리 드라이브 분리 (Access Tier 기반 격리) |
| 검증 방식 | minio / directpv / linux 커널 소스 직접 확인 |

## 1. 용어 정의

| 용어 | 설명 |
| --- | --- |
| **Erasure Coding** (소거 부호) | 데이터를 데이터 블록과 패리티 블록으로 나눠 저장하는 방식. 일부 블록이 소실돼도 나머지로 원본을 복원한다. 본 환경은 EC 2+2 (데이터 2 + 패리티 2) |
| **Erasure Set** (소거 집합) | 하나의 EC 연산이 적용되는 드라이브 묶음. EC 2+2이면 4장이 1개 Set. 객체 하나는 반드시 하나의 Set 안에서만 분산 저장된다 |
| **Server Pool** (서버 풀) | 여러 Erasure Set을 묶은 상위 단위. 용량 확장 시 Pool을 추가하는 방식으로 스케일아웃한다 |
| **Write Quorum** (쓰기 정족수) | 쓰기를 성공으로 판정하기 위한 최소 성공 드라이브 수. EC 2+2에서는 3 |
| **Read Quorum** (읽기 정족수) | 읽기에 필요한 최소 드라이브 수. EC 2+2에서는 2 |
| **Project Quota** (프로젝트 쿼터) | XFS의 디렉터리 단위 용량 제한 기능. 디렉터리 트리에 프로젝트 ID를 부여하고 사용량 상한을 설정한다 |
| **statfs()** | 파일시스템의 전체 용량과 여유 공간을 조회하는 POSIX 시스템 콜. df 명령과 애플리케이션의 용량 판단이 모두 이 값에 의존한다 |
| **ENOSPC** | 디스크 공간 부족을 나타내는 POSIX 오류 코드 |
| **Noisy Neighbor** | 동일 물리 자원을 공유하는 다른 워크로드가 자원을 소비하여 성능 · 가용성에 영향을 주는 현상 |
| **Degraded Write** (열화 쓰기) | 정상 개수보다 적은 블록만 기록하여 이중화 수준이 낮아진 상태로 쓰기를 완료하는 것 |

## 2. 현상

1. `mc admin info` 조회 시 특정 드라이브 1장의 사용률만 100%, 동일 노드의 나머지 드라이브는 70~80% 수준
2. S3 `PUT` 요청 중 일부가 `HTTP 507 Insufficient Storage` 반환
3. 동일 요청을 재시도해도 동일하게 실패
4. 반면 클러스터 전체 여유 공간은 충분한 것으로 표시됨

전체 용량이 충분한데 일부 요청만 실패한다는 점, 재시도가 무의미하다는 점이 일반적인 용량 부족 상황과 달랐습니다.

## 3. 영향

- 단일 객체 단위 실패율은 낮으나, 다수 파일을 기록하는 배치 작업(Spark Job 등)은 파일 하나만 실패해도 전체가 중단되므로 **작업 단위 실패율은 사실상 100%**
- 데이터 유실 및 정합성 훼손 없음. 쓰기가 사전에 거부되므로 부분 기록된 객체가 생성되지 않음
- 읽기 연산에는 영향 없음

## 4. 환경

| 구성 요소 | 내용 |
| --- | --- |
| 노드 | 베어메탈 8대 |
| 드라이브 | 노드당 1.7TB SSD 13장 (총 104장) |
| 스토리지 프로비저닝 | DirectPV (CSI 드라이버) |
| Erasure Coding | EC 2+2 (Set 크기 4) |
| Erasure Set 수 | 104 ÷ 4 = 26 |
| Server Pool 수 | 1 |
| 가용 용량 | 88.4TB (Raw 176.8TB의 50%) |

!!! warning "추가 구성 사항"
    OpenSearch 및 Prometheus 데이터 저장을 위해 동일한 DirectPV StorageClass로 200Gi PV 1개와 250Gi PV 4개를 프로비저닝한 상태였습니다. 이 구성이 원인이었습니다.

## 5. 원인 분석

### 5.1 가설 수립

> 객체 스토리지가 인접 워크로드의 공간 사용을 인지하지 못한 상태에서, 특정 드라이브가 포화되어 해당 드라이브가 속한 Erasure Set에만 쓰기 실패가 발생한다.

이 가설을 세 계층(CSI 프로비저너 → 커널 파일시스템 → 애플리케이션)으로 나누어 순차 검증했습니다.

### 5.2 스토리지 계층 구조 확인

DirectPV는 PV 하나가 드라이브 하나를 전용하는 구조가 아닙니다. 드라이브를 XFS로 포맷한 뒤 Project Quota 옵션으로 마운트합니다.

```go
// directpv: pkg/xfs/mount_linux.go:35
sys.Mount(device, target, "xfs", []string{"noatime"}, "prjquota")
```

개별 볼륨은 해당 파일시스템 하위의 디렉터리로 생성되며, PVC 요청 용량이 Project Quota의 Hard Limit으로 설정됩니다.

```go
// directpv: pkg/csi/node/server.go:248
quota := xfs.Quota{
    HardLimit: uint64(requiredBytes),   // PVC 요청 용량
}

// directpv: pkg/xfs/quota_linux.go:145
fsx.fsXProjID  = projectID
fsx.fsXXFlags |= uint32(flagProjectInherit)   // FS_XFLAG_PROJINHERIT
```

즉 드라이브 1장이 파일시스템 1개이며, 그 위에 객체 스토리지 볼륨과 로그 · 메트릭 볼륨이 동일 파일시스템을 공유하는 구조입니다.

### 5.3 용량 정보 전달 경로 분석

결론은 인지 불가이며, 다만 감소한 여유 공간 자체는 관측된다는 점을 구분해야 합니다.

**(1) CSI 계층의 용량 회계는 애플리케이션에 전달되지 않는다**

DirectPV는 `DirectPVDrive` 커스텀 리소스에 `FreeCapacity`, `AllocatedCapacity` 필드를 유지하며, 신규 PV 요청 시 여유 용량이 가장 큰 드라이브를 선택합니다.

```go
// directpv: pkg/csi/controller/utils.go
case drive.Status.FreeCapacity > maxFreeCapacity:
    maxFreeCapacity = drive.Status.FreeCapacity
    maxFreeCapacityDrives = []types.Drive{drive}
```

그러나 이는 CSI 드라이버 내부의 논리적 회계 정보이며, MinIO는 해당 커스텀 리소스를 조회하지 않습니다. 두 컴포넌트 간 용량 정보를 교환하는 인터페이스가 존재하지 않습니다.

**(2) 애플리케이션은 statfs 결과에만 의존한다**

```go
// minio: internal/disk/stat_linux.go
s := syscall.Statfs_t{}
err = syscall.Statfs(path, &s)

reservedBlocks := s.Bfree - s.Bavail
info = Info{
    Total: uint64(s.Frsize) * (s.Blocks - reservedBlocks),
    Free:  uint64(s.Frsize) * s.Bavail,
}
info.Used = info.Total - info.Free
```

**(3) 커널이 Project Quota 기준으로 결과를 보정한다**

```c
/* linux: fs/xfs/xfs_qm_bhv.c -- xfs_fill_statvfs_from_dquot() */
limit = blkres->softlimit ? blkres->softlimit : blkres->hardlimit;
if (limit) {
    uint64_t remaining = 0;
    if (limit > blkres->reserved)
        remaining = limit - blkres->reserved;

    statp->f_blocks = min(statp->f_blocks, limit);
    statp->f_bfree  = min(statp->f_bfree, remaining);
}
```

!!! important "핵심은 min() 연산이다"
    파일시스템 실제 여유 공간과 Quota 잔여량 중 작은 값을 반환합니다. fs/xfs/xfs_super.c 에서 f_bavail = f_bfree 대입이 해당 호출 이후에 수행되므로, MinIO가 참조하는 Bavail 에도 동일하게 반영됩니다.

| 필드 | 동작 |
| --- | --- |
| Total | Quota Hard Limit이 드라이브 용량과 동일하므로 변화 없음 |
| Free | 인접 워크로드가 소비한 만큼 감소 |
| Used | Total - Free 로 계산되므로 **인접 워크로드의 사용량이 자기 사용량으로 계상됨** |

따라서 `mc admin info` 에서 특정 드라이브의 사용률만 높게 표시되는 것은 오류가 아니라 정상 동작입니다. 애플리케이션은 자신이 해당 공간을 소비한 것으로 인식합니다.

부수적으로, 애플리케이션이 공간 부족을 사전에 인지하므로 쓰기 도중 ENOSPC가 발생하여 부분 기록된 객체가 남는 상황은 방지됩니다.

### 5.4 쓰기 거부 경로 분석

**(1) 객체는 이름 해시로 Erasure Set에 배정된다**

```go
// minio: cmd/erasure-sets.go
func (s *erasureSets) getHashedSetIndex(input string) int {
    return hashKey(s.distributionAlgo, input, len(s.sets), s.deploymentID)
}
```

객체 키의 해시값을 Set 개수로 나눈 나머지로 배정됩니다. 본 환경에서는 26개 중 하나가 결정되며, 동일 키는 항상 동일 Set으로 배정됩니다.

**(2) 용량 검사는 배정된 Set에 한정된다**

```go
// minio: cmd/erasure-server-pool.go:434
storageInfos[index] = getDiskInfos(ctx, pool.getHashedSet(object).getDisks()...)

// :448
if avail, err := hasSpaceFor(zinfo, size); err != nil || !avail {
    serverPools[i] = poolAvailableSpace{Index: i}   // 해당 Pool을 후보에서 제외
    continue
}
```

**(3) Set 내 드라이브 중 1장이라도 조건 미달이면 거부된다**

```go
// minio: cmd/object-api-utils.go:1242
func hasSpaceFor(di []*DiskInfo, size int64) (bool, error) {
    // We multiply the size by 2 to account for erasure coding.
    size *= 2
    if size < 0 {
        size = diskAssumeUnknownSize   // 1 << 30 = 1GiB
    }
    ...
    perDisk := size / int64(nDisks)
    for _, disk := range di {
        if disk == nil || disk.Total == 0 { continue }
        if int64(disk.Free) <= perDisk {
            return false, nil
        }
    }
```

임계값 산정 시 주의할 점이 두 가지 있습니다.

- 요청 크기를 EC 오버헤드 반영을 위해 2배로 보정한 뒤 드라이브 수로 나눕니다. 4장 Set 기준 실제 임계값은 2S ÷ 4 = S/2 입니다.
- Content-Length 가 없는 스트리밍 업로드는 크기를 1GiB로 가정합니다. 이 경우 드라이브당 256MiB의 여유가 필요합니다.

**(4) 후보 Pool이 없으면 507로 변환된다**

```go
// minio: cmd/erasure-server-pool.go:652
idx = z.getAvailablePoolIdx(ctx, bucket, object, size)
if idx < 0 {
    return -1, toObjectErr(errDiskFull)
}

// cmd/object-api-errors.go:61
case errDiskFull.Error():
    return StorageFull{}

// cmd/api-errors.go:1300
ErrStorageFull: {
    Code:           "XMinioStorageFull",
    HTTPStatusCode: http.StatusInsufficientStorage,   // 507
},
```

### 5.5 실패 분포 분석

Set 단위로 검사가 이뤄지므로 실패는 전체가 아닌 일부에만 발생합니다. 배정이 해시 기반이므로 실패는 무작위가 아니라 객체 키에 대해 결정론적입니다.

- 동일 객체 키는 재시도해도 반복 실패
- 다른 키는 정상 처리
- 워크로드 전체 실패율은 영향받은 Set 수를 전체 Set 수로 나눈 값에 수렴

PV 5개가 서로 다른 드라이브에 배치된 경우 최대 5개 Set이 영향을 받아 약 19%의 쓰기가 실패합니다. 다만 다수 파일을 기록하는 배치 작업은 파일 하나의 실패로 전체가 중단되므로, 작업 단위로는 상시 실패로 관측됩니다.

## 6. 설계 배경 검토

분석 과정에서 제기된 의문 두 가지. 소스 확인 결과 모두 의도된 설계임이 확인되었습니다.

### 6.1 포화된 드라이브를 제외하고 나머지 3장에 기록할 수 없는가

기술적으로는 가능합니다. EC 2+2의 Write Quorum은 3이며, 쓰기 경로는 Quorum까지의 실패를 허용합니다.

```go
// minio: cmd/erasure-metadata.go:557
writeQuorum := dataBlocks
if dataBlocks == parityBlocks {
    writeQuorum++          // 2+2 → 3
}

// cmd/erasure-object.go:1183
n, erasureErr := erasure.Encode(ctx, data, writers, buffer, writeQuorum)
```

그럼에도 사전 차단하는 이유는 드라이브 포화가 일시적 장애가 아니기 때문입니다. 드라이브 오프라인은 복구 시 Heal 과정에서 누락 블록이 재생성되므로 Degraded Write가 합리적인 절충입니다. 반면 포화 상태는 자동 해소되지 않으므로 다음 문제가 누적됩니다.

1. EC 2+2에서 블록 3개만 기록된 객체는 Read Quorum(2)까지 여유가 1장뿐입니다. 생성 시점부터 단일 장애 지점에 노출됩니다.
2. Heal이 완료될 수 없습니다. 누락 블록을 기록할 공간이 없기 때문입니다.
3. 해당 Set으로 배정되는 모든 후속 쓰기가 동일 상태로 누적됩니다.

사전 차단은 복구 불가능한 이중화 부채를 만들지 않기 위한 설계 판단으로 해석됩니다.

### 6.2 포화된 Set을 건너뛰고 다른 Set에 기록할 수 없는가

구조적으로 불가능합니다. erasure-sets.go 의 모든 객체 연산은 다음 형태입니다.

```go
// minio: cmd/erasure-sets.go
set := s.getHashedSet(object)
return set.PutObject(ctx, bucket, object, data, opts)
```

Set 선택 함수에 용량이 인자로 전달되지 않으며, 후보 목록이나 대체 경로가 존재하지 않습니다.

이유는 객체 위치를 저장하지 않고 계산하기 때문입니다. MinIO에는 객체와 Set의 매핑을 기록하는 메타데이터 인덱스가 없습니다. GetObjectInfo 역시 동일하게 getHashedSet(object) 를 한 번 호출할 뿐입니다. 쓰기 시 Set 우회를 허용하면 읽기 시 객체 위치를 특정할 수 없어 전체 Set을 순회해야 합니다. 해시 함수가 인덱스 역할을 수행하므로 배치가 결정론적이어야 조회가 성립합니다.

### 6.3 우회는 Server Pool 단위에서만 지원된다

동일한 우회 동작이 상위 계층에는 구현되어 있습니다. Pool이 복수인 경우 Pool 0의 해당 Set이 포화되면 Pool 1로 우회하고, 읽기 시에는 모든 Pool을 병렬 조회하여 객체를 탐색합니다. Pool은 일반적으로 1~3개이므로 전수 조회가 가능하지만, Set은 26개이므로 동일 방식을 적용할 수 없습니다.

| 단위 | 개수 | 위치 결정 방식 | 포화 시 동작 |
| --- | --- | --- | --- |
| Server Pool | 1~3 | 읽기 시 전수 조회 | 다른 Pool로 우회 |
| Erasure Set | 26 | 객체 키 해시로 계산 | 우회 불가, 507 반환 |

본 환경은 Server Pool이 1개이므로 우회 경로가 존재하지 않습니다.

### 6.4 완충 로직 미적용 구간

사용률이 높은 Pool을 후순위로 배치하는 로직이 존재하나, 단일 Pool 환경에서는 동작하지 않습니다.

```go
// minio: cmd/erasure-server-pool.go:362
func (p serverPoolsAvailableSpace) FilterMaxUsed(maxUsed int) {
    if len(p) <= 1 {
        // Nothing to do.
        return
    }
```

또한 .minio.sys 내부 버킷에 대한 쓰기는 hasSpaceFor 검사를 수행하지 않습니다. 사용자 객체는 507로 사전 차단되지만, 내부 메타데이터 쓰기는 커널 수준의 ENOSPC를 직접 반환받습니다. IAM 캐시 갱신이나 버킷 메타데이터 갱신의 산발적 실패로 관측될 수 있습니다.

## 7. 진단 절차

세 계층이 서로 다른 값을 참조하므로, 계층별 값을 대조하면 원인을 확정할 수 있습니다.

```bash
# 1단계. 드라이브별 볼륨 배치 확인 -- 공유 드라이브 식별
kubectl directpv list drives --output wide
kubectl directpv list volumes --all -o wide \
  | awk "{print \$NF}" | sort | uniq -c | sort -rn

# 2단계. 애플리케이션이 인식 중인 드라이브별 여유 공간
mc admin info <alias> --json \
  | jq ".info.servers[].drives[] | {endpoint, availableSpace, totalSpace}"

# 3단계. 커널이 반환하는 실제 값 확인
xfs_quota -x -c "report -p -h" /var/lib/directpv/mnt/<fsuuid>
df -h /var/lib/directpv/mnt/<fsuuid>
```

!!! success "판정 기준"
    3단계에서 프로젝트별 사용량 합계와 df 의 Used 값이 일치하는 반면, 2단계의 availableSpace 만 현저히 작다면 본 사례에 해당합니다.

mc admin heal 은 본 사례에 효과가 없습니다. 데이터 정합성 문제가 아니라 용량 부족 문제이기 때문입니다.

## 8. 조치

### 8.1 워크로드별 드라이브 분리 (권장)

DirectPV는 Access Tier 레이블과 StorageClass 파라미터를 매칭하여 드라이브 풀을 분리할 수 있습니다.

```go
// directpv: pkg/csi/controller/utils.go
case string(directpvtypes.AccessTierLabelKey):
    accessTiers, _ := directpvtypes.StringsToAccessTiers(value)
    if len(accessTiers) > 0 && drive.GetAccessTier() != accessTiers[0] {
        return false
    }
```

노드당 13장 중 1장을 로그 · 메트릭 전용 Tier로 분리하고, 객체 스토리지는 나머지 12장만 사용하도록 구성합니다.

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| 객체 스토리지 드라이브 | 104장 | 96장 |
| Erasure Set 수 | 26 | 24 |
| 가용 용량 | 88.4TB | 81.6TB |

가용 용량 6.8TB를 감수하고 장애 요인을 제거하는 방식입니다. 부수적으로 I/O 경합도 해소됩니다. Prometheus의 WAL 기록과 OpenSearch의 세그먼트 병합이 EC 블록 기록과 동일 SSD 큐를 공유하던 상황이 함께 정리됩니다.

기존 볼륨은 kubectl directpv move 로 동일 노드 내 다른 드라이브로 이전 가능하나, 로그 · 메트릭 데이터의 경우 PVC 재생성 후 재적재가 일반적으로 더 신속합니다.

### 8.2 임시 조치

즉시 분리가 어려운 경우, 인접 워크로드의 보존 기간을 단축하여 포화 드라이브의 여유 공간을 확보하면 해당 Set의 쓰기가 재개됩니다. 근본 해결이 아니므로 재발합니다.

## 9. 재발 방지

- [ ] **드라이브 단위 모니터링** — 클러스터 전체 사용률 지표로는 본 장애를 탐지할 수 없다. minio_node_drive_free_bytes 를 드라이브 단위로 수집하고, Set 하나가 차단되기 전에 경보하도록 임계값을 설정한다
- [ ] **StorageClass 분리 표준화** — 객체 스토리지 전용 StorageClass와 일반 워크로드용 StorageClass를 분리하여, 동일 드라이브에 이종 워크로드가 배치되는 경로를 원천 차단한다
- [ ] **EC 구성 재검토** — EC 2+2는 저장 효율 50%이며 Set당 드라이브 1장 손실만 허용한다. 노드당 드라이브 수를 조정하여 Set 크기 16, 패리티 4 구성이 가능하다면 저장 효율 75%를 확보할 수 있다

## 10. 정리

- DirectPV의 Project Quota는 용량 상한만 분리하며, 파일시스템 자체는 공유됩니다.
- 객체 스토리지는 인접 볼륨의 존재를 인지하지 못하나, 커널의 min() 보정으로 감소한 여유 공간은 관측합니다. 결과적으로 타 워크로드의 사용량을 자기 사용량으로 계상합니다.
- 용량 검사는 getHashedSet 으로 결정된 Erasure Set 단위로 수행되며, 임계값은 S/2 입니다.
- Set 배정은 객체 키 해시로 결정되므로 다른 Set으로 우회할 수 없습니다. 해시가 인덱스를 대체하는 설계이기 때문입니다.
- 우회는 Server Pool 단위에서만 지원되며, 단일 Pool 환경에서는 적용되지 않습니다.

!!! tip
    Project Quota를 적용하면 워크로드가 격리된다고 판단하기 쉽습니다. 격리되는 것은 용량 상한이며, 가시성은 격리되지 않습니다.

## 참고

- [minio/minio](https://github.com/minio/minio) — cmd/erasure-sets.go, cmd/erasure-server-pool.go, cmd/object-api-utils.go, cmd/erasure-metadata.go, internal/disk/stat_linux.go
- [minio/directpv](https://github.com/minio/directpv) — pkg/xfs/, pkg/csi/controller/utils.go, pkg/csi/node/server.go
- [torvalds/linux](https://github.com/torvalds/linux) — fs/xfs/xfs_qm_bhv.c, fs/xfs/xfs_super.c
