---
title: 05. StarRocks / Spark · Iceberg · HMS 연동
---

# 🔍 쿼리 엔진 — StarRocks / Spark · Iceberg · HMS 연동

!!! abstract "개요"
    **목적** — StarRocks / Spark가 HMS + MinIO 조합의 Iceberg 테이블을 읽고 쓰는 구성을 맞추고, 커밋 동작을 확인

    **결론 한 줄** — HMS는 "어느 메타파일이 현재 유효한가"를 가리키는 포인터만 가지고, 실제 트랜잭션 상태는 전부 MinIO의 메타파일 체인으로 표현된다

## 1. 역할 분담

| 구분 | HMS | MinIO |
| --- | --- | --- |
| 저장 내용 | `metadata_location` 포인터만 | 실제 데이터 + 모든 메타파일 |
| Commit 시점 | 포인터 CAS 업데이트 | 파일은 미리 기록 완료 |
| 트랜잭션 보장 | 포인터 교체로 원자성 | Immutable 파일 + Append Only |

INSERT 시 파일은 항상 **하위 파일부터** 씁니다 — data parquet → manifest → manifest list(snapshot) → metadata.json → 마지막으로 HMS 포인터 교체. 이 마지막 단계 전까지는 롤백 가능합니다.

동시성은 HMS 포인터의 Compare-And-Swap으로 처리됩니다. 두 트랜잭션이 같은 베이스를 읽고 동시에 커밋하면 한쪽은 CAS 실패 → 재시도로 수렴합니다(OCC).

## 2. StarRocks External Catalog 설정

```sql
CREATE EXTERNAL CATALOG iceberg_catalog
PROPERTIES (
  "type"                            = "iceberg",
  "iceberg.catalog.type"            = "hive",
  "hive.metastore.uris"             = "thrift://<hms-host>:9083",
  "aws.s3.use_instance_profile"     = "false",
  "aws.s3.access_key"               = "<ACCESS_KEY>",
  "aws.s3.secret_key"               = "<SECRET_KEY>",
  "aws.s3.endpoint"                 = "http://minio-svc:9000",
  "aws.s3.enable_path_style_access" = "true"     -- MinIO 필수
);

SHOW CATALOGS;
SET CATALOG iceberg_catalog;
SHOW DATABASES;

-- HMS 동기화가 안 될 때 수동 갱신
REFRESH EXTERNAL TABLE my_database.my_table;
```

### BE 로컬 스토리지의 용도

External Catalog 구성에서 BE 디스크는 데이터 저장소가 **아닙니다.** 세 가지 용도로 쓰입니다.

1. 로그
2. Spill to disk — 메모리 부족 시 정렬 · 조인 · 집계 중간 결과
3. **Local Cache** — S3에서 읽은 Parquet을 로컬에 캐싱. 기본 비활성이라 be.conf에서 명시적으로 켜야 함

즉 "데이터를 담는 공간"이 아니라 **"S3를 빠르게 읽기 위한 캐시 공간"**으로 사이즈를 잡아야 합니다.

## 3. Spark · S3A 설정

```text
# 필수 5줄
spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem
spark.hadoop.fs.s3a.endpoint=http://minio-svc:9000
spark.hadoop.fs.s3a.path.style.access=true
spark.hadoop.fs.s3a.access.key=<ACCESS_KEY>
spark.hadoop.fs.s3a.secret.key=<SECRET_KEY>

# 성능
spark.hadoop.fs.s3a.multipart.size=128M
spark.hadoop.fs.s3a.multipart.threshold=128M
spark.hadoop.fs.s3a.connection.maximum=100
spark.hadoop.fs.s3a.fast.upload=true
spark.hadoop.fs.s3a.fast.upload.buffer=bytebuffer
```

키를 직접 박지 않고 Secret → 환경변수 provider로 주입하는 방식을 씁니다.

```yaml
# SparkApplication CR
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom: { secretKeyRef: { name: s3-credentials, key: access-key } }
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom: { secretKeyRef: { name: s3-credentials, key: secret-key } }
```

```text
spark.hadoop.fs.s3a.aws.credentials.provider=com.amazonaws.auth.EnvironmentVariableCredentialsProvider
```

경로는 `s3://` 가 아니라 **`s3a://`** 를 씁니다.

## 4. 체크리스트

| 단계 | 확인 항목 |
| --- | --- |
| 네트워크 | BE/executor → S3(9000) 아웃바운드 허용 |
| 네트워크 | FE → HMS Thrift 9083 허용 |
| 인증 | Access Key에 `s3:GetObject`, `s3:ListBucket` 권한 |
| HMS | 테이블 location이 올바른 스킴으로 등록됐는지 |
| 클라이언트 | path-style access 옵션 적용 여부 |
| 드라이버 | DBeaver 접속 시 MySQL 드라이버 8.x (5.x는 일부 타입 오류) |

## 5. 정리

| 항목 | 확인 결과 |
| --- | --- |
| 커밋 원자성 | HMS 포인터 CAS 교체 시점 |
| 동시 커밋 | 한쪽 CAS 실패 → 재시도로 수렴 (OCC) |
| MinIO 필수 옵션 | path-style access (StarRocks/Spark 공통) |
| BE 디스크 용도 | 로그 + spill + S3 read 캐시 (기본 비활성) |
| 경로 스킴 | `s3a://` 사용 |
