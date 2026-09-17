---
title: 03. Kafka 이벤트 파이프라인
---

# 📨 이벤트/파이프라인 — Kafka 알림 설정과 적재 구현

!!! abstract "개요"
    **목적** — 버킷에 오브젝트가 들어오면 Kafka로 알림을 받고, 그걸 다시 스토리지에 적재하는 파이프라인을 구성

    **범위** — ① AIStor 버킷 prefix 이벤트 알림 설정 ② Kafka → S3 put 구현과 멱등성 설계

    **결론 한 줄** — 오브젝트 키를 `partition-startOffset-endOffset` 로 잡으면 재처리에도 같은 키에 덮어써져 effectively exactly-once가 된다

## 1. 이벤트 알림 설정 (2단계)

### 1.1 서버 레벨 — Kafka 타겟 등록

```bash
mc admin config set <alias> notify_kafka:1 \
  brokers="kafka-0.kafka.svc.cluster.local:9092,kafka-1.kafka.svc.cluster.local:9092" \
  topic="minio-events" \
  queue_dir="/data/events" \
  queue_limit="10000"

# 적용 후 재기동 필요
mc admin service restart <alias>

# 확인
mc admin config get <alias> notify_kafka
```

- `notify_kafka:1` 의 `1` 은 타겟 식별자입니다. 여러 타겟을 등록하려면 `:2`, `:3` 으로 늘립니다.
- **`queue_dir` 가 중요합니다.** 대상이 오프라인일 때 이벤트를 로컬에 큐잉해둡니다. 설정해두면 Kafka나 MinIO 재시작 순간의 이벤트가 유실되지 않고 복구 후 재전송됩니다.
- SASL / TLS가 필요하면 `sasl=on sasl_username=... sasl_password=... tls=on` 파라미터를 추가합니다.
- 에어갭 환경이므로 브로커 주소는 내부 DNS 또는 ClusterIP를 씁니다.

### 1.2 버킷 레벨 — 이벤트 규칙 추가

```bash
mc event add <alias>/mybucket arn:minio:sqs::1:kafka \
  --event put,delete \
  --prefix "logs/2026/" \
  --suffix ".parquet"

mc event ls <alias>/mybucket
mc event rm <alias>/mybucket arn:minio:sqs::1:kafka
```

!!! warning "주의"
    동일 이벤트 타입에 대해 prefix가 서로 겹치면 규칙 등록이 실패합니다. `logs/` 와 `logs/2026/` 를 둘 다 put 이벤트로 걸 수 없습니다.

생성되는 메시지는 S3 표준 알림 JSON 포맷이라 `s3.object.key` 를 그대로 다운스트림(StarRocks Routine Load, Spark 파이프라인 등)에서 쓸 수 있습니다.

## 2. Kafka → S3 적재 구현

### 2.1 컨슈머 설정

```yaml
spring:
  kafka:
    bootstrap-servers: kafka-0.kafka:9092
    consumer:
      group-id: s3-sink-group
      enable-auto-commit: false
      max-poll-records: 5000
      fetch-min-bytes: 1048576      # 1MB 모일 때까지 대기
      fetch-max-wait-ms: 5000
    listener:
      type: batch
      ack-mode: manual
```

핵심은 **배치 수신 + 수동 offset commit** 입니다. 메시지 1건당 put하면 small file 문제와 API 호출 폭증이 같이 옵니다.

### 2.2 S3 클라이언트 — MinIO 필수 설정

```java
S3Client.builder()
    .endpointOverride(URI.create("https://aistor.internal:9000"))
    .region(Region.US_EAST_1)          // MinIO는 값 자체는 무관
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create(accessKey, secretKey)))
    .forcePathStyle(true)              // MinIO 필수
    .build();
```

`forcePathStyle(true)` 가 빠지면 버킷명이 호스트명으로 붙어 DNS 해석에 실패합니다. StarRocks의 `enable_path_style_access`, Spark의 `fs.s3a.path.style.access` 와 같은 이유입니다.

### 2.3 멱등 키 설계

```java
var first = records.get(0);
var last  = records.get(records.size() - 1);
String key = String.format("raw/events/dt=%s/p%d-%d-%d.jsonl.gz",
    LocalDate.now(), first.partition(), first.offset(), last.offset());

s3.putObject(...);
ack.acknowledge();   // put 성공 후에만 commit → at-least-once
```

put 성공 → ack 순서로 at-least-once가 보장되고, 키를 파티션 · 오프셋 범위로 만들면 rebalance 후 재처리도 같은 키에 덮어써져 중복이 사라집니다. Kafka Connect S3 Sink가 exactly-once를 구현하는 방식과 동일합니다.

## 3. 테스트에서 드러난 주의점

| 항목 | 내용 |
| --- | --- |
| small file | 메시지당 put 시 오브젝트 수 폭증 → 반드시 배치로 묶음 |
| flush 정책 | poll 단위 flush는 저트래픽 토픽에서 파일이 잘게 쪼개짐 |
| 시간 / 크기 버퍼링 | 도입 시 flush 완료 전까지 ack를 미뤄야 해 offset 추적 로직이 복잡해짐 |
| rebalance | `onPartitionsRevoked` 에서 버퍼 flush / 폐기 처리 안 하면 중복 · 유실 |
| prefix 중복 | 동일 이벤트 타입에 겹치는 prefix 등록 시 에러 |

버퍼링 복잡도가 올라가는 시점이 Connect나 Spark로 넘어갈 신호입니다.

## 4. 대안 비교

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| 커스텀 컨슈머 | 제어권 최대, 키 설계 자유 | 버퍼링 · rebalance 로직 직접 구현 |
| Kafka Connect S3 Sink | 표준화, exactly-once 내장 | 에어갭은 커넥터 반입 필요 (Aiven 배포판은 Apache 2.0) |
| Spark Structured Streaming | Iceberg 적재 · compaction DAG 재사용 | 클러스터 자원 상시 점유 |

용도가 **원본 아카이빙**이면 커스텀 컨슈머가 가볍고, **Iceberg 적재 전 랜딩존**이면 이미 운영 중인 Spark Streaming 파이프라인에 합치는 편이 orphan file 관리 · compaction 측면에서 유리했습니다.

## 5. 정리

| 항목 | 결과 |
| --- | --- |
| 알림 설정 단계 | 서버 타겟 등록 → 버킷 규칙 추가 |
| 내구성 | `queue_dir` 설정 시 대상 오프라인 중 이벤트 보존 |
| MinIO 필수 옵션 | path-style access 강제 |
| exactly-once | 오프셋 범위 기반 멱등 키로 달성 |
| 상태 관리 | rebalance 핸들러 미구현 시 중복 / 유실 발생 |
