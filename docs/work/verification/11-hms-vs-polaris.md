---
title: HMS(Oracle) vs Apache Polaris(REST) 카탈로그 비교 검증
---

# 🧊 Hive Metastore(Oracle) vs Apache Polaris(REST) — Iceberg 카탈로그 비교 검증 보고서

!!! abstract "개요"
    2026-09-11 · Trino 483 · Apache Iceberg 1.11.0 · 동일 MinIO 오브젝트 스토리지 위에서 HMS(Oracle 26ai) 백엔드와 Polaris 1.7.0(PostgreSQL 17) REST 카탈로그를 **동일 스키마 / 동일 데이터(약 1,000만 건)** 로 구성해 성능·기능·운영 특성을 비교

## 1. 테스트 목적

| # | 검증 항목 |
| --- | --- |
| 1 | 동일한 Iceberg 테이블을 HMS 카탈로그와 Polaris REST 카탈로그에 각각 만들고 약 1천만 건을 적재 |
| 2 | 카탈로그 종류가 **쓰기 / 읽기 / DDL / 커밋 지연** 에 주는 영향 측정 |
| 3 | 두 카탈로그의 **View 관리 방식** 차이 규명 |
| 4 | Polaris에 HMS의 Iceberg 테이블을 **external catalog** 로 등록했을 때의 **핸들링 범위** 확인 |

---

## 2. 아키텍처

> 그림 1. 테스트 아키텍처 — 동일한 MinIO 오브젝트 스토리지 위에 HMS(oracle26)와 Polaris(oracle21) 두 카탈로그를 나란히 놓고, Trino 483 한 대가 hcatalog · pcatalog · hcat_ext 세 카탈로그로 양쪽을 모두 바라본다.

### 2-1. 구성 요소

| 계층 | hcatalog (HMS) | pcatalog (Polaris) |
| --- | --- | --- |
| 카탈로그 구현 | Hive Metastore 3.1.3 (thrift) | Apache Polaris 1.7.0 (Iceberg REST) |
| 카탈로그 저장소 | Oracle AI Database 26ai Free (`freepdb1`) | PostgreSQL 17 |
| 배포 위치 | oracle26 / 10.10.105.27 (Docker) | oracle21 / 10.10.105.4 (Docker) |
| 인증 | 없음 (plain thrift) | OAuth2 client_credentials (`root`) |
| 오브젝트 스토리지 | MinIO `s3a://hcatalog/warehouse` | MinIO `s3://pcatalog/warehouse` |
| Trino 커넥터 | `iceberg`  • `iceberg.catalog.type=hive_metastore` | `iceberg`  • `iceberg.catalog.type=rest` |

!!! note
    Trino는 `fs.native-s3` 로 MinIO를 **클러스터 내부 주소**(`minio.minio-verify.svc:9000`)로, HMS/Polaris는 **NodePort**(`10.10.105.9:32000`)로 같은 MinIO를 바라본다. 같은 버킷이므로 메타데이터에 기록되는 경로는 스킴/버킷만 남고 엔드포인트는 클라이언트 설정으로 분리된다.

---

## 3. 배포 결과 (1~4단계)

| 단계 | 확인 내용 | 결과 |
| --- | --- | --- |
| 1 | oracle26 에 Oracle + Hive Metastore | Oracle 26ai Free는 **가동 중이었고**, HMS 스키마(74 테이블 / SCHEMA_VERSION 3.1.0)도 이미 초기화되어 있었음. **HMS 서비스(thrift)는 떠 있지 않아 신규 기동** |
| 2 | oracle21 에 PostgreSQL + Polaris | 둘 다 **미배포 상태** → PostgreSQL 17 + Polaris 1.7.0 신규 배포, realm `POLARIS` bootstrap |
| 3 | 불필요 서비스 정리 | `oracle-xe-21c`(21c XE)는 이번 테스트에 불필요 → **중지 상태 유지**. `openldap`은 타 검증용으로 보여 **임의 중지하지 않음** (확인 요청 항목) |
| 4 | 쿠버네티스 접근 | `cluster-mesh2.kubeconfig` 로 접속 성공 (control-plane 1 + worker 3, v1.32.1) |

### 3-1. 기동된 컨테이너

```plain text
oracle26 (10.10.105.27)
  hive-metastore     apache/hive:3.1.3                     Up   (:9083)
  oracle-free-26ai   container-registry.oracle.com/...     Up   (:1521)

oracle21 (10.10.105.4)
  polaris            apache/polaris:1.7.0                  Up   (:8181, :8182)
  polaris-postgres   postgres:17                           Up   (:5432)
  openldap           osixia/openldap:1.5.0                 Up   (이번 테스트와 무관)
  oracle-xe-21c      gvenzl/oracle-xe:21-slim              Exited (중지 유지)
```

---

## 4. MinIO 버킷 및 경로 매핑 (5~6단계)

버킷 2개를 새로 만들고, 벤치마크 전용 서비스 계정(`icebergbench`)을 발급해 Trino / HMS / Polaris가 모두 같은 키로 접근하도록 했다 (MinIO root 키는 사용하지 않음).

| 카탈로그 | 버킷 | 스키마(namespace) | 실제 매핑된 위치 |
| --- | --- | --- | --- |
| `hcatalog` | `hcatalog` | `bench` | `s3a://hcatalog/warehouse/bench` (HMS `DBS.DB_LOCATION_URI`) |
| `pcatalog` | `pcatalog` | `bench` | `s3://pcatalog/warehouse/bench/` (Polaris namespace property `location`) |

```sql
-- HMS: 위치를 명시적으로 지정
CREATE SCHEMA hcatalog.bench WITH (location = 's3a://hcatalog/warehouse/bench');
-- Polaris: 카탈로그의 default-base-location 아래로 자동 배치
CREATE SCHEMA pcatalog.bench;
```

적재 완료 후 두 버킷의 사용량은 **거의 동일**했다 — 같은 Iceberg 포맷/같은 데이터이므로 당연한 결과이며, 카탈로그 종류는 오브젝트 레이아웃에 영향을 주지 않는다는 확인이기도 하다.

| 버킷 | 용량 | 오브젝트 수 |
| --- | --- | --- |
| `hcatalog` | 498 MiB | 414 |
| `pcatalog` | 498 MiB | 413 |

---

## 5. 테스트 데이터 (8단계)

```sql
CREATE TABLE <catalog>.bench.lineitem_bench (
  l_orderkey bigint, l_partkey bigint, l_suppkey bigint, l_linenumber integer,
  l_quantity double, l_extendedprice double, l_discount double, l_tax double,
  l_returnflag varchar(1), l_linestatus varchar(1),
  l_shipdate date, l_commitdate date, l_receiptdate date,
  l_shipinstruct varchar(25), l_shipmode varchar(10), l_comment varchar(44)
) WITH (format = 'PARQUET');

INSERT INTO <catalog>.bench.lineitem_bench
SELECT ... FROM tpch.sf10.lineitem WHERE l_orderkey <= 10000000;   -- 9,999,388 행
```

- 소스는 Trino `tpch` 커넥터 sf10 (결정적 생성) → 두 카탈로그에 **완전히 동일한 데이터**
- 적재 결과: 9,999,388 행 / 10 data file / 약 259 MB (Parquet)
- 편향 제거를 위해 **전체 시나리오를 2회** 수행 (round 1: HMS→Polaris, round 2: Polaris→HMS 순서 반전)

---

## 6. 성능 측정 결과 (9단계)

측정값은 클라이언트 체감 시간이 아니라 **Trino 코디네이터의 서버 사이드 통계**(`/v1/query` 의 `elapsedTime`)를 사용했다. CLI JVM 기동 시간 등 클라이언트 오버헤드가 섞이지 않는다.

> 그림 2. 성능 지표 한눈에 비교 — 읽기는 ±6% 이내로 사실상 동등하고, 차이가 나는 구간은 전부 카탈로그 메타데이터를 쓰거나 목록을 읽는 구간이다. 막대는 두 값 중 큰 쪽을 100%로 정규화했다.

### 6-1. 종합 비교 (ms, 낮을수록 좋음)

| 측정 항목 | HMS r1 | Polaris r1 | HMS r2 | Polaris r2 | HMS 평균 | Polaris 평균 | Polaris/HMS |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CREATE TABLE (16컬럼) | 211.3 | 181.2 | 148.6 | 143.5 | 179.9 | 162.3 | 0.90 |
| CREATE TABLE (3컬럼) | 155.9 | 167.7 | 134.9 | 135.5 | 145.4 | 151.6 | 1.04 |
| CREATE TABLE x20 평균 | 171.4 | 155.6 | 126.7 | 126.1 | 149.1 | 140.8 | 0.94 |
| SHOW TABLES | 166.4 | 89.5 | 71.2 | 70.2 | 118.8 | 79.8 | 0.67 |
| information_schema.columns | 284.8 | 220.6 | 127.4 | 134.8 | 206.1 | 177.7 | 0.86 |
| DROP TABLE x20 평균 | 97.3 | 59.9 | 68.8 | 48.6 | 83.0 | 54.3 | **0.65** |
| **INSERT 9,999,388행** | 15,930 | 14,340 | 15,250 | 14,090 | **15,590** | **14,215** | **0.91** |
| SELECT count(*) | 208.2 | 107.0 | 61.6 | 100.0 | 134.9 | 103.5 | 0.77 |
| TPC-H Q1 형태 집계 | 1,340 | 953.7 | 673.2 | 968.2 | 1,006.6 | 961.0 | 0.95 |
| 선택적 필터 스캔 | 418.4 | 355.1 | 280.2 | 298.7 | 349.3 | 326.9 | 0.94 |
| 포인트 조회 | 520.2 | 236.4 | 117.7 | 194.3 | 319.0 | 215.4 | 0.68 |
| `$files` 메타데이터 | 107.6 | 94.4 | 81.8 | 64.9 | 94.7 | 79.7 | 0.84 |
| `$snapshots` 메타데이터 | 68.6 | 87.2 | 39.7 | 54.4 | 54.1 | 70.8 | 1.31 |
| **1k행 INSERT x30 평균** | 424.4 | 361.6 | 377.7 | 334.1 | **401.0** | **347.9** | **0.87** |
| OPTIMIZE | 813.5 | 553.6 | 514.8 | 566.3 | 664.1 | 560.0 | 0.84 |

### 6-2. 대량 적재 처리량

| 대상 | 소요 | CPU 시간 | 기록량 | 행/초 | MB/s | Peak Mem |
| --- | --- | --- | --- | --- | --- | --- |
| HMS r1 | 15.93 s | 127 s | 258.7 MB | 627,708 | 16.2 | 455 MB |
| HMS r2 | 15.25 s | 119 s | 259.2 MB | 655,698 | 17.0 | 476 MB |
| Polaris r1 | 14.34 s | 109 s | 259.0 MB | 697,307 | 18.1 | 439 MB |
| Polaris r2 | 14.09 s | 108 s | 259.3 MB | 709,680 | 18.4 | 438 MB |

### 6-3. 소량 커밋 지연 (1,000행 INSERT x 30회 — 카탈로그 왕복이 지배적인 구간)

| 대상 | avg | p50 | p95 | min | max |
| --- | --- | --- | --- | --- | --- |
| HMS r1 | 424 ms | 404 | 572 | 354 | 659 |
| HMS r2 | 378 ms | 384 | 431 | 308 | 520 |
| Polaris r1 | 362 ms | 370 | 414 | 308 | 426 |
| Polaris r2 | 334 ms | 342 | 383 | 264 | 393 |

### 6-4. 워밍업 후 반복 읽기 (3회, 카탈로그 교차 실행)

| 쿼리 | HMS avg | HMS min | Polaris avg | Polaris min | Polaris/HMS |
| --- | --- | --- | --- | --- | --- |
| count(*) | 71.7 | 48.3 | 76.0 | 72.9 | 1.06 |
| TPC-H Q1 집계 | 535.6 | 518.9 | 551.0 | 521.7 | 1.03 |
| 선택적 필터 | 265.8 | 244.3 | 273.4 | 245.5 | 1.03 |
| 포인트 조회 | 193.3 | 187.1 | 173.7 | 150.3 | 0.90 |
| SHOW TABLES | 52.9 | 44.9 | 66.6 | 64.8 | 1.26 |
| `$files` | 63.4 | 57.4 | 67.5 | 64.1 | 1.06 |

### 6-5. 동시 커밋 (8 writer 동시 INSERT, 6 라운드)

| 라운드 | 페이로드 | HMS 실패 | Polaris 실패 |
| --- | --- | --- | --- |
| 1 | 2,000행 x8 | 0/8 | **2/8** |
| 2 | 2,000행 x8 | 0/8 | 0/8 |
| 3 | 2,000행 x8 | 0/8 | 0/8 |
| 4~6 | 1행 x8 | 0/8 | 0/8 |
| **합계** |  | **0/48** | **2/48** |

Polaris 실패 시 오류:

```plain text
Failed to commit the transaction during insert: Commit failed:
Requirement failed: branch main has changed: expected id 3731011747399293110 != 3171147169018615876
```

Polaris 서버 로그상으로는 내부적으로 `Tasks` 재시도(100ms 백오프)를 수행한 뒤 최종적으로 `HTTP 409` 를 반환했고, Trino REST 카탈로그 클라이언트는 이를 그대로 사용자 오류로 올렸다.

### 6-6. 성능 해석

- **읽기 성능은 사실상 동일하다.** 워밍업 후 반복 읽기 편차는 ±6% 이내다. 카탈로그는 쿼리당 1~2회의 `loadTable` 호출에만 관여하고 실제 스캔은 동일한 Iceberg 메타데이터 + 동일한 MinIO 오브젝트를 읽기 때문이다. **카탈로그 선택은 쿼리 성능 결정 요인이 아니다.**
- **쓰기/DDL은 Polaris가 일관되게 소폭 우위**다 (커밋 -13%, DROP -35%). HMS 경로는 `thrift → JDBC → Oracle` 왕복이 필요하고 테이블 메타데이터가 `TBLS / SDS / COLUMNS_V2 / TABLE_PARAMS / SERDES` 로 정규화되어 다중 DML이 발생하는 반면, Polaris는 단일 `entities` 테이블에 대한 PostgreSQL 트랜잭션 1건으로 끝난다.
- **대량 적재 9% 차이는 카탈로그 때문이 아니다.** 15초 중 커밋은 수백 ms에 불과하다. CPU 시간(127s vs 108s)도 함께 차이 나는 것으로 보아 실행 시점의 클러스터 상태 차이가 더 크다. 즉 **1천만 건 규모 배치 적재에서 카탈로그 선택의 영향은 무시할 수준**이다.
- **차이가 커지는 구간은 "작은 커밋을 자주" 하는 워크로드**다. 커밋당 약 53ms 차이는 스트리밍 마이크로배치처럼 분당 수백 커밋이 발생하는 환경에서 누적된다.
- 동시 커밋에서 Polaris 쪽 1회 실패는 **재현성이 낮은 간헐적 현상**(6라운드 중 1회)이지만, 실패 시 **재시도 책임이 클라이언트로 넘어온다**는 점은 설계 시 고려해야 한다.

---

## 7. View 관리 방식 비교 (10단계)

동일한 View를 양쪽에 만들고, 내부 저장 형태를 **Oracle 테이블 / Polaris REST API / MinIO 오브젝트** 수준까지 직접 열어 확인했다.

> 그림 3. View 저장 형태 비교 — HMS는 뷰 정의 본문을 base64로 감싸 Oracle CLOB에 넣고 덮어쓰기 때문에 이력이 남지 않는다. Polaris는 Iceberg View Spec JSON을 MinIO에 쓰고 카탈로그는 포인터만 가지며, CREATE OR REPLACE 때마다 versions[]와 version-log[]가 누적된다.

### 7-1. 실측된 차이

| 항목 | HMS View | Polaris(Iceberg) View |
| --- | --- | --- |
| 표준 | Presto/Trino 독자 포맷 | **Apache Iceberg View Spec v1** |
| 저장 위치 | Oracle `TBLS.VIEW_ORIGINAL_TEXT` (CLOB) | MinIO `metadata/0000N-*.gz.metadata.json` |
| 카탈로그가 갖는 것 | 정의 **본문 전체** | `metadata-location` **포인터만** |
| SQL 인코딩 | `/* Presto View: base64 JSON */` | 평문 JSON 내 `representations[].sql` |
| 컬럼 스키마 | `COLUMNS_V2` 에는 `dummy string` 1개, 실제 스키마는 base64 안에 | `schemas[]` 에 **필드 ID·타입까지 정식 기술** |
| 다중 엔진(dialect) | 불가 (Trino 전용) | `representations[].dialect` 로 엔진별 SQL 병기 가능 |
| 버전 관리 | **없음** — 같은 행을 덮어씀 (`TBL_ID` 고정, `CREATE_TIME` 불변) | **있음** — `CREATE OR REPLACE` 시 `versions[]` / `version-log[]` 누적 |
| 오브젝트 스토리지 사용 | 사용하지 않음 | 버전마다 새 metadata JSON 생성 |
| Materialized View | **지원** | **미지원** |

### 7-2. 검증 근거

**HMS** — Oracle에서 직접 조회한 결과:

```plain text
TBL_ID 46  v_returnflag_summary  VIRTUAL_VIEW
VIEW_ORIGINAL_TEXT = /* Presto View: eyJvcmlnaW5hbFNxbCI6IlNFTEVDVFxuICBsX3JldHVybmZsYWc... */

TABLE_PARAMS:  presto_view=true | trino_created_by=Trino Iceberg connector
               trino_version=483 | comment=Presto View
COLUMNS_V2  :  dummy / string          ← 실제 컬럼은 base64 JSON 안에만 존재
```

`CREATE OR REPLACE VIEW` 를 2회 실행해도 `TBL_ID=46`, `CREATE_TIME=1789130536` 이 그대로였다 → **이전 정의는 남지 않는다.**

**Polaris** — `GET /v1/pcatalog/namespaces/bench/views/v_returnflag_summary`:

```plain text
view-uuid        : 7c14b0b5-84d3-4c55-ad0b-9d1283be6d23
format-version   : 1
current-version  : 2
versions         : 2      (v1 schema 0 / v2 schema 1, 둘 다 dialect=trino)
version-log      : [{ts:...543922, v:1}, {ts:...612870, v:2}]
metadata-location: s3://pcatalog/warehouse/bench/v_returnflag_summary-.../metadata/00001-....gz.metadata.json
```

**Materialized View** — 양쪽 동일 DDL로 시도:

```plain text
hcatalog : CREATE MATERIALIZED VIEW → 성공
           HMS에는 TBL_TYPE=VIRTUAL_VIEW 이면서 TABLE_PARAMS 에 metadata_location 이 붙는
           하이브리드 형태로 저장(= 내부 Iceberg 스토리지 테이블), REFRESH 시 실제 데이터 적재
           comment=Presto Materialized View / presto_view=true

pcatalog : CREATE MATERIALIZED VIEW → 실패
           "createMaterializedView is not supported for Iceberg REST catalog"
```

### 7-3. 운영 관점 시사점

- **엔진 호환성이 중요하면 Polaris.** HMS View는 base64로 감싼 Trino 독자 포맷이라 Spark/Flink에서 읽을 수 없고, 반대로 Hive가 만든 View를 Trino가 못 읽는 문제도 같은 뿌리다. Iceberg View는 스펙 표준이라 다른 엔진이 `representations` 에 자기 dialect를 추가할 수 있다.
- **View 변경 이력·롤백이 필요하면 Polaris.** HMS는 덮어쓰기라 "어제 정의"가 남지 않는다.
- **Materialized View가 필요하면 현재로서는 HMS만 가능하다** (Trino 483 기준). Polaris 쪽은 MV가 필요한 워크로드를 **Iceberg 테이블 + 스케줄드 INSERT OVERWRITE** 로 대체해야 한다.
- **백업 대상이 다르다.** HMS는 View 정의가 Oracle 안에만 있으므로 **RDBMS 백업이 곧 View 백업**이다. Polaris는 포인터만 DB에 있고 본문은 오브젝트 스토리지에 있으므로 **양쪽을 함께 백업**해야 한다.

### 7-4. 교차 카탈로그 VIEW 실측 — Presto View ↔ Iceberg View

hcatalog 에는 **Presto View** 로 pcatalog 의 테이블을, pcatalog 에는 **Iceberg View** 로 external catalog(`hcat_ext`)에 등록된 hcatalog 테이블을 바라보게 만들어, 같은 "교차 참조"가 두 뷰 형식에서 어떻게 다르게 동작하는지 확인했다.

```sql
-- hcatalog 에 Presto View : pcatalog 테이블을 참조
CREATE VIEW hcatalog.bench.v_cross_to_p AS
  SELECT grp, count(*) AS cnt, sum(amt) AS sum_amt
  FROM pcatalog.bench.vsrc_p GROUP BY grp;

-- pcatalog 에 Iceberg View : external catalog(hcat_ext)에 등록된 hcatalog 테이블을 참조
CREATE VIEW pcatalog.bench.v_cross_to_h AS
  SELECT grp, count(*) AS cnt, sum(amt) AS sum_amt
  FROM hcat_ext.bench.vsrc_h GROUP BY grp;
```

> 그림 4. 교차 카탈로그 VIEW — 두 뷰 모두 조회할 때마다 저장된 SQL을 재분석해 원본을 다시 읽는다. 차이는 뷰 형식이 아니라 중간에 static facade external catalog 가 끼느냐에서 생긴다.

#### 항목별 실측 비교

| 항목 | hcatalog · Presto View → pcatalog | pcatalog · Iceberg View → hcat_ext |
| --- | --- | --- |
| 생성 · 조회 | ✅ 정상, 4개 그룹 결과 동일 | ✅ 정상, 4개 그룹 결과 동일 |
| 정의가 저장되는 곳 | Oracle `TBLS.VIEW_ORIGINAL_TEXT` (CLOB, base64) | MinIO `metadata/00000-*.gz.metadata.json` (PostgreSQL은 포인터만) |
| 저장된 SQL | `FROM pcatalog.bench.vsrc_p` — 카탈로그명 그대로 | `FROM hcat_ext.bench.vsrc_h` — 카탈로그명 그대로 |
| 컬럼 스키마 | base64 JSON 안 `columns[4]` (`COLUMNS_V2` 는 dummy 1개) | `schemas[4]` — 필드 ID·타입 정식 기술 |
| 기본 실행 주체 | `owner=trino`, `runAsInvoker=false` — DEFINER | `properties.trino.run-as-owner=trino` — DEFINER |
| `SECURITY INVOKER` | ✅ 지원 — `runAsInvoker=true` 로 기록 | ✅ 지원 — `properties` 자체가 기록되지 않음 |
| **원본 변경 반영** | ✅ **즉시** — 100행 INSERT 후 1,000 → 1,100 | ❌ **반영 안 됨** — 원본은 1,100인데 뷰는 1,000 |
| 동기화 방법 | 불필요 | hms_ext 에서 de-register → 새 metadata-location 으로 재등록 |
| 참조 소실 시 | `Failed analyzing stored view ...: Table ... does not exist` | 동일 메시지 |
| 소실 후 뷰 객체 | 남아있음 — `SHOW CREATE VIEW` · `information_schema.views` 에 계속 노출 | 동일 |
| 목록 관리 | HMS `TBLS` 한 테이블에 `TBL_TYPE=VIRTUAL_VIEW` 로 섞임 | `/tables` 와 `/views` 엔드포인트 분리 (이름 공간은 공유) |
| 2단 체이닝 | ✅ hcatalog View → pcatalog View → hcat_ext 테이블 3단 교차도 정상 동작 | ← 동일 |
| 뷰 경유 오버헤드 | analysis +10.5 ~ 12.8 ms | analysis +11.6 ~ 11.8 ms |
| external catalog 안에 뷰 생성 | ❌ 불가 — `Failed to create view` | ❌ 불가 — hcat_ext 에는 뷰를 만들 수 없다 |

!!! tip "느린 쪽은 Iceberg View 가 아니다"
    두 뷰 모두 조회할 때마다 저장된 SQL을 꺼내 재분석하고 원본을 다시 읽는다. 다만 Iceberg View 쪽 경로에만 스냅샷이 고정된 static facade external catalog 가 끼어 있을 뿐이다. pcatalog 내부 테이블을 보는 Iceberg View 는 Presto View 와 똑같이 즉시 반영된다.

#### ⚠️ 교차 카탈로그 뷰 사용 시 주의사항

| # | 주의점 | 근거 · 대응 |
| --- | --- | --- |
| 1 | **뷰 본문에 Trino 카탈로그 이름이 문자열로 박힌다** | `pcatalog` · `hcat_ext` 는 Trino 설정 파일의 이름일 뿐 카탈로그가 아는 이름이 아니다. 카탈로그명을 바꾸거나 다른 Trino 클러스터에서 같은 뷰를 열면 깨진다. Iceberg View 도 `default-catalog` 가 비어 있어 **표준 스펙만으로는 해석 불가** — 뷰의 이식성은 Iceberg View 라도 보장되지 않는다 |
| 2 | **external catalog 를 경유하는 뷰는 조용히 틀린 값을 낸다** | 에러 없이 옛 데이터를 정상인 것처럼 반환한다(1,000 vs 1,100). 운영하려면 de-register/재등록 파이프라인을 스케줄링하고, 뷰에 "기준 시각" 컬럼을 달아 소비자가 신선도를 판단할 수 있게 해야 한다 |
| 3 | **기본이 DEFINER — 교차 뷰는 권한 우회 통로가 된다** | 두 뷰 모두 소유자 권한으로 실행된다. hcatalog 권한만 가진 사용자가 hcatalog 뷰를 통해 pcatalog 데이터를 읽게 된다. 의도한 공유가 아니라면 `SECURITY INVOKER` 로 생성해야 한다 (양쪽 모두 지원) |
| 4 | **Polaris RBAC 는 Trino 최종 사용자를 보지 못한다** | Trino 카탈로그는 `oauth2.credential=root:...` 로 고정된 단일 서비스 계정으로 접속한다. 즉 Polaris 의 principal/catalog role 로는 교차 뷰 접근을 통제할 수 없고, **Trino 측 접근제어(OPA 등)로 막아야 한다** |
| 5 | **참조 무결성이 없다 — dangling view 가 남는다** | 원본을 DROP 하거나 de-register 해도 뷰는 그대로 남고 조회 시점에만 실패한다. 카탈로그를 넘나들면 소유 팀이 달라 더 흔하다 — 전체 뷰에 대해 주기적으로 `EXPLAIN` 을 돌리는 유효성 점검을 권장 |
| 6 | **external catalog 위에는 뷰를 올릴 수 없다** | `hcat_ext` 에 `CREATE VIEW` 는 `Failed to create view`. 따라서 교차 뷰는 반드시 INTERNAL 카탈로그(pcatalog)에 만들어야 하고, 그 뷰는 external 의 고정 스냅샷을 본다 |
| 7 | **교차 MV 는 데이터 소재를 옮긴다** | Polaris(REST)는 MV 자체가 불가. HMS 에서 pcatalog 를 보는 MV 를 만들면 실체화된 데이터가 **hcatalog 버킷에 복제**된다 — 데이터 소재·보존 정책 위반 가능성을 먼저 확인할 것 |
| 8 | 이름 충돌은 안전하게 막힌다 | Polaris 는 `/tables` 와 `/views` 가 분리되어 있지만 이름 공간은 공유한다. REST 를 직접 호출해도 `409 View with same name already exists` 로 차단되며, Trino 도 `Table already exists` 로 막는다 |

---

## 8. External Catalog 검증 (11단계)

### 8-1. 시도한 두 가지 경로

> 그림 5. External Catalog 등록 경로와 핸들링 범위 — ① Hive 페더레이션은 공식 이미지에 구현체가 없어 406으로 막히고, 정적 등록(register)만 동작한다. ② 읽기는 전부 되지만 쓰기는 전부 차단되며 namespace 관리만 예외적으로 허용된다. ③ 등록된 것은 그 시점의 metadata-location 하나뿐이라 원본 HMS의 커밋이 전파되지 않는다.

### 8-2. 경로 A — Catalog Federation (실패, 원인 규명 완료)

| 시도 | 결과 |
| --- | --- |
| `connectionType=HIVE` (기본 설정) | `500 Unsupported connection type: HIVE` |
| `connectionType=HADOOP` (기본 설정) | `500 Unsupported connection type: HADOOP` |
| 서버 설정으로 `ENABLE_CATALOG_FEDERATION=true`, `SUPPORTED_CATALOG_CONNECTION_TYPES` 적용 후 | 카탈로그 **생성은 201 성공** |
| 생성된 페더레이션 카탈로그로 namespace 조회 | `406 External catalog factory for type 'HIVE' is unavailable.` |

!!! warning
    **공식 Docker 이미지(`apache/polaris:1.7.0`)에는 HMS 페더레이션 구현체가 번들되어 있지 않다.** 모델·설정 레이어는 존재하지만 런타임 팩토리가 없다. 실제로 쓰려면 Hive 번들을 포함해 직접 빌드하거나, HMS 앞에 Iceberg REST 어댑터를 두고 `connectionType=ICEBERG_REST` 로 연결해야 한다. (인증 타입은 `BEARER` / `OAUTH` 만 유효하며 `IMPLICIT` · `SIGV4` · 미지정은 거부됨)

### 8-3. 경로 B — Static Facade External Catalog (성공)

```bash
# 1) EXTERNAL 카탈로그 생성 (connectionConfigInfo 없음 = static facade)
POST /api/management/v1/catalogs
  {"catalog":{"name":"hms_ext","type":"EXTERNAL",
   "properties":{"default-base-location":"s3://hcatalog/warehouse"},
   "storageConfigInfo":{"storageType":"S3","allowedLocations":["s3://hcatalog/warehouse"],
     "endpoint":"http://10.10.105.9:32000","pathStyleAccess":true,"stsUnavailable":true}}}

# 2) namespace 생성 후 HMS가 가진 metadata_location 을 그대로 등록
POST /api/catalog/v1/hms_ext/namespaces/bench/register
  {"name":"lineitem_bench_s3a",
   "metadata-location":"s3a://hcatalog/warehouse/bench/lineitem_bench-.../metadata/00001-....metadata.json"}
```

**핸들링 범위 — Trino의 `hcat_ext` 카탈로그로 직접 시험한 결과**

| 연산 | 결과 | 서버 메시지 |
| --- | --- | --- |
| `SELECT count(*)` | ✅ 정상 (9,999,388) | — |
| `SELECT ... GROUP BY` | ✅ 정상 | — |
| `$snapshots` / `$files` 메타데이터 조회 | ✅ 정상 | — |
| `INSERT` | ❌ 차단 | `Cannot update table on static-facade external catalogs.` |
| `DELETE` | ❌ 차단 | `Cannot update table on static-facade external catalogs.` |
| `ALTER TABLE ADD COLUMN` | ❌ 차단 | `Cannot update table on static-facade external catalogs.` |
| `CREATE TABLE` | ❌ 차단 | `Failed to create transaction` |
| `CREATE VIEW` | ❌ 차단 | `Failed to create view` |
| `DROP TABLE` | ❌ 차단 | `Failed to drop table` |
| **`CREATE SCHEMA` / `DROP SCHEMA`** | ⚠️ **허용됨** | namespace 관리는 external 카탈로그에서도 가능 |
| `ALTER TABLE EXECUTE optimize` | ⚠️ 부분 | 재작성 대상 0건이면 성공(커밋 없음), 재작성이 필요하면 커밋 단계에서 차단 |

**가장 중요한 특성 — 스냅샷이 고정된다**

```plain text
HMS(hcatalog) 측에서 5,000행 추가 커밋
  ├─ hcatalog.bench.commit_bench   count = 35,000   snapshot = 8961762130070745814
  └─ hcat_ext.bench.commit_bench   count = 30,000   snapshot =  781882561524379211   ← 갱신 안 됨
```

등록된 것은 **그 시점의 `metadata-location` 문자열 하나**일 뿐이며, Polaris는 HMS를 추적하지 않는다. 동기화하려면 **de-register 후 재등록**해야 한다 (재등록만 하면 `409 Table already exists`).

```plain text
POST .../register   (동일 이름)                  → 409 AlreadyExistsException
DELETE .../tables/commit_bench  (purge 없이)     → 204  ← 카탈로그 등록만 해제, 데이터는 보존
POST .../register   (새 metadata-location)       → 200  → count = 35,000 으로 정상 반영
```

### 8-4. 부수적으로 확인된 Polaris의 제약

| 동작 | 내용 |
| --- | --- |
| **스토리지 위치 중복 금지** | 이미 다른 카탈로그가 점유한 경로로 카탈로그를 만들면 `Cannot create Catalog ... locations overlaps with an existing catalog` |
| **s3 ↔ s3a 정규화** | `allowedLocations` 에 `s3://` 만 넣어도 `s3a://` metadata-location 등록이 통과하고, 같은 테이블을 두 스킴으로 중복 등록하면 **동일 위치로 판정해 403** |
| **카탈로그 삭제 선행조건** | 카탈로그 롤이 남아 있으면 삭제 불가 (`cannot be dropped, it is not empty`) — 롤 먼저 제거 |
| **DROP 시 purge 기본 차단** | `DROP TABLE` 은 기본적으로 `403 Unable to purge entity`. `polaris.config.drop-with-purge.enabled=true` 를 카탈로그 속성으로 켜야 Trino의 `DROP TABLE`(purgeRequested=true)이 동작 |
| **프로덕션 준비성 검사** | 1.7.0은 `ALLOW_INSECURE_STORAGE_TYPES` 같은 위험 플래그가 켜져 있으면 **기동 자체를 거부**한다. S3 호환 스토리지에 STS가 없으면 `storageConfigInfo.stsUnavailable=true` 를 써야 한다 |

---

## 9. 메타데이터 저장소 구조 비교

| 항목 | HMS / Oracle 26ai | Polaris / PostgreSQL 17 |
| --- | --- | --- |
| 스키마 테이블 수 | **74** | **8** |
| 데이터 모델 | 정규화 (`TBLS`, `SDS`, `COLUMNS_V2`, `TABLE_PARAMS`, `SERDES`, `DBS` ...) | 범용 엔티티 (`entities`, `grant_records`, `events`, `*_metrics_report` ...) |
| 현재 저장량 | 4.31 MB (segment) | 8.2 MB (database) |
| 테이블 1개 등록 시 쓰기 | TBLS 1 + SDS 1 + SERDES 1 + COLUMNS_V2 N + TABLE_PARAMS M | `entities` 1행 |
| 권한 모델 | 없음 (외부 시스템에 위임) | `grant_records` 기반 RBAC (catalog role ↔ principal role) |
| 관측성 | 없음 | `scan_metrics_report`, `commit_metrics_report` 테이블 내장 |

현재 규모(테이블 7개)에서 실제 행 수는 `TBLS 7 / SDS 7 / COLUMNS_V2 42 / TABLE_PARAMS 60 / SERDES 7` 대 `entities 20 / grant_records 12` 이다. **테이블당 메타데이터 행 수가 한 자릿수 배 차이**나며, 이것이 6-1의 DDL·커밋 지연 차이로 그대로 나타난다.

---

## 10. 결론 및 권고

### 10-1. 결론

1. **쿼리 성능 관점에서 두 카탈로그는 동등하다.** 반복 읽기 편차 ±6% 이내. 카탈로그는 스캔 경로에 개입하지 않으므로 "어떤 카탈로그가 더 빠른가"는 선택 기준이 되기 어렵다.
2. **쓰기 메타데이터 경로는 Polaris가 일관되게 빠르다** (커밋 -13%, DROP -35%). 정규화된 HMS 스키마 대비 단일 엔티티 테이블 구조의 이점이며, **소량 고빈도 커밋 워크로드에서 차이가 누적**된다.
3. **1천만 건 배치 적재에서는 카탈로그 차이가 무의미하다** (15.6s vs 14.2s, 커밋은 그중 수백 ms).
4. **View는 두 시스템이 근본적으로 다른 물건을 만든다.** HMS는 Trino 독자 base64 정의를 RDBMS에, Polaris는 Iceberg View Spec JSON을 오브젝트 스토리지에 저장하며 **버전 이력을 남긴다**. 단, **Materialized View는 현재 HMS만 지원**한다.
5. **Polaris의 external catalog는 "정적 파사드"다.** 읽기는 되지만 모든 쓰기가 차단되고, 원본 HMS의 변경을 추적하지 않는다. 진짜 페더레이션(HIVE connectionType)은 공식 이미지에 구현체가 없어 사용할 수 없다.

### 10-2. 도입 시나리오별 권고

| 상황 | 권고 |
| --- | --- |
| Trino 단일 엔진 + MV 필요 | **HMS 유지** — MV는 현재 HMS에서만 동작 |
| Spark/Flink/Trino 멀티 엔진 | **Polaris** — Iceberg View Spec으로 View까지 공유 가능 |
| 스트리밍/마이크로배치 (분당 수백 커밋) | **Polaris** — 커밋당 ~53ms 우위가 누적. 단 409 재시도 전략 필수 |
| 멀티테넌트 · 세밀한 권한 통제 | **Polaris** — catalog/principal role 기반 RBAC 내장 |
| 기존 HMS 자산을 Polaris로 점진 이관 | **정적 등록(register)은 읽기 전용 미러로만.** 양쪽 동시 쓰기는 불가하며 컷오버 시점에 de-register/재등록 절차 필요 |

### 10-3. 운영상 주의점

- Polaris `DROP TABLE` 은 카탈로그 속성 `polaris.config.drop-with-purge.enabled=true` 없이는 403이다.
- Polaris 1.7.0은 프로덕션 준비성 검사가 강제라 위험 플래그로는 기동되지 않는다. S3 호환 스토리지는 `stsUnavailable: true` 를 써야 한다.
- HMS는 s3a 스킴으로 `CREATE SCHEMA` 시 실제 디렉터리를 만들기 때문에 `hadoop-aws` + `aws-java-sdk-bundle` 을 클래스패스에 넣어야 한다 (이미지의 `/opt/hadoop/share/hadoop/tools/lib` 에 존재).
- Polaris는 카탈로그 간 스토리지 경로 중복을 막으므로 버킷/프리픽스 설계를 먼저 확정해야 한다.

---

## 11. 부록

### 11-1. 재현 자산

모든 배포 스크립트·벤치마크 SQL·원시 측정 결과는 `C:\project\ycb-brain\iceberg-bench\` 에 보관되어 있다.

```plain text
iceberg-bench\
├─ k8s\trino.yaml                     Trino 483 (coordinator 1 + worker 3) + 카탈로그 3종
├─ k8s\minio-setup-job.yaml           버킷 hcatalog/pcatalog 생성 + 서비스 계정 발급
├─ k8s\minio-inspect-job.yaml         버킷 용량/오브젝트 통계
├─ oracle26\hive-site.xml             HMS 설정 (Oracle JDBC + s3a)
├─ oracle26\run-hms.sh                HMS 기동
├─ oracle21\run-postgres.sh           PostgreSQL 17
├─ oracle21\bootstrap-polaris.sh      realm bootstrap
├─ oracle21\run-polaris.sh            Polaris 기동 (federation 설정 포함)
├─ oracle21\create-pcatalog.sh        pcatalog 생성
├─ oracle21\grant-pcatalog.sh         권한 부여
├─ oracle21\enable-drop-purge.sh      DROP purge 허용
├─ oracle21\register-hms-external.sh  HMS 테이블 external 등록
├─ oracle21\refresh-external.sh       de-register + 재등록
├─ sql\gen_bench_sql.sh               벤치마크 SQL 생성기
├─ sql\bench_*.sql                    실제 실행된 벤치마크 스크립트
├─ parse_stats.js                     Trino /v1/query 통계 파서
└─ results\                           원시 결과 (bench_queries.csv, bench_summary.json 등)
```

### 11-2. 접속 정보

| 대상 | 주소 |
| --- | --- |
| Trino Web UI | `http://any-node:32085` (NodePort) |
| MinIO API / Console | `http://any-node:32000` / `:32001` |
| Polaris REST | `http://10.10.105.4:8181/api/catalog` |
| Polaris Management | `http://10.10.105.4:8181/api/management/v1` |
| Hive Metastore | `thrift://10.10.105.27:9083` |

### 11-3. 측정 방법 주의

- 모든 지연 수치는 Trino 코디네이터 `/v1/query` 의 `queryStats.elapsedTime` (서버 사이드)이다.
- 순서 편향 제거를 위해 라운드 1은 HMS→Polaris, 라운드 2는 Polaris→HMS 순으로 실행했다.
- 반복 읽기 측정은 두 카탈로그를 1회씩 번갈아 3세트 실행했다.
- 동시성 테스트는 8개의 독립 Trino 세션을 동시에 띄워 같은 테이블에 INSERT 했다.

### 11-4. 핵심 설정 원문

이 문서만으로 동일한 환경을 재현할 수 있도록 핵심 설정을 그대로 옮긴다.

**① Hive Metastore — `oracle26:~/hive-metastore/conf/hive-site.xml`**

```xml
<configuration>
  <!-- Metastore RDBMS : Oracle 26ai Free (freepdb1) -->
  <property><name>javax.jdo.option.ConnectionURL</name><value>jdbc:oracle:thin:@//10.10.105.27:1521/freepdb1</value></property>
  <property><name>javax.jdo.option.ConnectionDriverName</name><value>oracle.jdbc.driver.OracleDriver</value></property>
  <property><name>javax.jdo.option.ConnectionUserName</name><value>hive</value></property>
  <property><name>javax.jdo.option.ConnectionPassword</name><value>HiveMeta_2026x</value></property>
  <property><name>hive.metastore.schema.verification</name><value>true</value></property>
  <property><name>datanucleus.schema.autoCreateAll</name><value>false</value></property>

  <!-- Thrift service -->
  <property><name>hive.metastore.uris</name><value>thrift://0.0.0.0:9083</value></property>
  <property><name>metastore.thrift.port</name><value>9083</value></property>
  <property><name>hive.metastore.event.db.notification.api.auth</name><value>false</value></property>
  <property><name>hive.compactor.initiator.on</name><value>false</value></property>
  <property><name>hive.metastore.housekeeping.threads.on</name><value>false</value></property>

  <!-- Warehouse on MinIO bucket hcatalog -->
  <property><name>hive.metastore.warehouse.dir</name><value>s3a://hcatalog/warehouse</value></property>
  <property><name>metastore.warehouse.dir</name><value>s3a://hcatalog/warehouse</value></property>

  <!-- S3A -> MinIO (NodePort) -->
  <property><name>fs.s3a.endpoint</name><value>http://10.10.105.9:32000</value></property>
  <property><name>fs.s3a.access.key</name><value>icebergbench</value></property>
  <property><name>fs.s3a.secret.key</name><value>IcebergBench_2026_MinioKey</value></property>
  <property><name>fs.s3a.path.style.access</name><value>true</value></property>
  <property><name>fs.s3a.connection.ssl.enabled</name><value>false</value></property>
  <property><name>fs.s3a.impl</name><value>org.apache.hadoop.fs.s3a.S3AFileSystem</value></property>
  <property><name>fs.s3.impl</name><value>org.apache.hadoop.fs.s3a.S3AFileSystem</value></property>
</configuration>
```

**② HMS 기동 — `run-hms.sh`** (기존 스키마 위에 서비스만 올리므로 `IS_RESUME=true` 로 initSchema 생략)

```bash
docker run -d --name hive-metastore --restart unless-stopped \
  -p 9083:9083 \
  -e SERVICE_NAME=metastore -e DB_DRIVER=oracle -e IS_RESUME=true \
  -e HIVE_CUSTOM_CONF_DIR=/hive_custom_conf -e SERVICE_OPTS="-Xmx2G" \
  -v /home/ubuntu/hive-metastore/conf:/hive_custom_conf:ro \
  -v /home/ubuntu/hive-metastore/lib/ojdbc8.jar:/opt/hive/lib/ojdbc8.jar:ro \
  -v /home/ubuntu/hive-metastore/auxlib/hadoop-aws-3.1.0.jar:/opt/hive/lib/hadoop-aws-3.1.0.jar:ro \
  -v /home/ubuntu/hive-metastore/auxlib/aws-java-sdk-bundle-1.11.271.jar:/opt/hive/lib/aws-java-sdk-bundle-1.11.271.jar:ro \
  apache/hive:3.1.3
```

**③ Polaris 기동 — `run-polaris.sh`**

```bash
docker run -d --name polaris --restart unless-stopped --network polaris-net \
  -p 8181:8181 -p 8182:8182 \
  -e POLARIS_PERSISTENCE_TYPE=relational-jdbc \
  -e QUARKUS_DATASOURCE_DB_KIND=postgresql \
  -e QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://polaris-postgres:5432/polaris \
  -e QUARKUS_DATASOURCE_USERNAME=polaris -e QUARKUS_DATASOURCE_PASSWORD=Polaris_2026x \
  -e POLARIS_REALM_CONTEXT_REALMS=POLARIS \
  -e AWS_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=icebergbench -e AWS_SECRET_ACCESS_KEY=IcebergBench_2026_MinioKey \
  -e QUARKUS_HTTP_HOST=0.0.0.0 \
  -v /home/ubuntu/polaris-conf/application.properties:/deployments/config/application.properties:ro \
  apache/polaris:1.7.0

# bootstrap (최초 1회)
docker run --rm --network polaris-net <동일한 DATASOURCE 환경변수> \
  apache/polaris-admin-tool:1.7.0 bootstrap \
  --realm=POLARIS --credential=POLARIS,root,Polaris_Root_2026x
```

```javascript
# /home/ubuntu/polaris-conf/application.properties
polaris.features."ENABLE_CATALOG_FEDERATION"=true
polaris.features."SUPPORTED_CATALOG_CONNECTION_TYPES"=["ICEBERG_REST","HIVE","HADOOP"]
```

**④ Trino 카탈로그 3종 — `k8s/trino.yaml` 의 ConfigMap `trino-catalog`**

```javascript
# hcatalog.properties  — Iceberg on Hive Metastore
connector.name=iceberg
iceberg.catalog.type=hive_metastore
hive.metastore.uri=thrift://10.10.105.27:9083
hive.metastore.thrift.client.read-timeout=60s
iceberg.file-format=PARQUET
iceberg.register-table-procedure.enabled=true
fs.native-s3.enabled=true
s3.endpoint=http://minio.minio-verify.svc.cluster.local:9000
s3.region=us-east-1
s3.path-style-access=true
s3.aws-access-key=icebergbench
s3.aws-secret-key=IcebergBench_2026_MinioKey

# pcatalog.properties  — Iceberg on Polaris REST (INTERNAL)
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://10.10.105.4:8181/api/catalog
iceberg.rest-catalog.warehouse=pcatalog
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.credential=root:Polaris_Root_2026x
iceberg.rest-catalog.oauth2.scope=PRINCIPAL_ROLE:ALL
iceberg.rest-catalog.vended-credentials-enabled=false
iceberg.file-format=PARQUET
fs.native-s3.enabled=true
s3.endpoint=http://minio.minio-verify.svc.cluster.local:9000
s3.region=us-east-1
s3.path-style-access=true
s3.aws-access-key=icebergbench
s3.aws-secret-key=IcebergBench_2026_MinioKey

# hcat_ext.properties  — Polaris EXTERNAL 카탈로그(hms_ext) 연결
#   pcatalog 과 동일하고 warehouse 만 다르다
iceberg.rest-catalog.warehouse=hms_ext
```

### 11-5. 첨부 파일

그림 5장은 각 절에 인라인으로 들어있고, 아래 zip 에 원본 PNG와 SVG 소스(HTML)까지 모두 들어 있다.

**① 전체 자산 번들** — 배포 스크립트 · 벤치마크 SQL · 원시 결과 · 그림 원본/소스 전부 (2.0 MB) — `iceberg-bench-assets.zip`: iceberg-bench/ (k8s, oracle21, oracle26, sql, results, REPORT.md, CONFIRM-ME.md, parse_stats.js) + images/ (PNG 5장 + src/*.html)

**② 원시 측정 데이터**

- `bench_queries.csv` — 벤치마크 쿼리 376건의 Trino 서버 사이드 통계 (elapsed / cpu / analysis / planning / 입출력 바이트 / peak memory)
- `bench_summary.json` — 항목별 집계 결과 (6장 표의 원본 수치)
- `view_perf.json` — 뷰 경유 vs 직접 조회 성능 측정 원본 (7-4절)

---

## 12. 확인 요청 사항

작업 중 제가 판단해서 진행한 것, 요청에 명시되지 않았지만 짚고 넘어가야 할 것, 그리고 결정을 넘겨드려야 하는 것을 정리했다.

### 12-1. 제가 판단해서 그대로 진행한 것

| # | 항목 | 내용과 근거 |
| --- | --- | --- |
| A-1 | **"oracle21" 서버 지칭 — 호스트명이 실제로는 `oracle19`** | 10.10.105.4 로 해석했다. 이 서버에 21c XE 컨테이너(`oracle-xe-21c`)가 있고 이전 보고서에서 "서버 A(21c XE)"로 기록된 장비기 때문. 다만 이 서버의 실제 hostname 은 `oracle19` 이다 (10.10.105.27 은 hostname·역할 모두 `oracle26` 으로 일치). **호스트명을 정정할지 문서 표기를 맞출지 결정 필요** |
| A-2 | HMS 는 "스키마만 있고 서비스는 없는" 상태였다 | oracle26 에 Oracle 과 HMS 스키마(74 테이블)만 있었고 thrift 서비스는 떠 있지 않았다. 기존 스키마를 재초기화하지 않고(`IS_RESUME=true`) 그 위에 서비스만 기동했다 |
| A-3 | HMS 에 S3A 라이브러리를 추가로 넣음 | Trino 가 `CREATE SCHEMA ... location='s3a://...'` 를 호출하면 HMS 가 직접 디렉터리를 만들기 때문에 `hadoop-aws-3.1.0.jar`  • `aws-java-sdk-bundle-1.11.271.jar` 를 `/opt/hive/lib` 로 마운트했다 (이미지 내 `/opt/hadoop/share/hadoop/tools/lib` 에서 추출) |
| A-4 | Polaris 기동 플래그 변경 | `SKIP_CREDENTIAL_SUBSCOPING_INDIRECTION` / `ALLOW_INSECURE_STORAGE_TYPES` 로 시도했으나 **1.7.0 은 이 플래그가 켜져 있으면 기동을 거부**한다. 서버가 안내한 대체 방식인 `storageConfigInfo.stsUnavailable=true` (카탈로그 단위)로 전환 |
| A-5 | Polaris `DROP TABLE` purge 허용 | Polaris 는 purge 를 동반한 DROP 을 기본 403으로 막는다. Trino 는 항상 `purgeRequested=true` 로 호출하므로 벤치마크 진행을 위해 `pcatalog` 속성에 `polaris.config.drop-with-purge.enabled=true` 를 설정했다. **운영 정책상 이게 맞는지 확인 필요** (기본 차단이 오히려 안전장치일 수 있음) |
| A-6 | Polaris 카탈로그 페더레이션 기능 플래그 활성화 | HMS 페더레이션 시도를 위해 `ENABLE_CATALOG_FEDERATION=true` 와 `SUPPORTED_CATALOG_CONNECTION_TYPES` 를 켰다. 결과적으로 HIVE 구현체가 없어 사용 불가였고, 설정은 무해하여 그대로 두었다. 원복은 `run-polaris.sh` 의 `-v ...application.properties` 마운트만 제거하면 된다 |
| A-7 | Trino 는 기존 것을 쓰지 않고 신규 배포 | `trino-verify` 에 Trino **480** 이 이미 있었지만, "최신 버전 하나 띄워서"라는 요청과 기존 검증 환경 보호를 위해 `iceberg-bench` 네임스페이스에 **483** 을 별도 배포했다. 기존 환경은 손대지 않았다 |
| A-8 | **성능 측정 방식** | CLI 왕복 시간이 아니라 Trino 코디네이터의 서버 사이드 통계(`/v1/query` → `elapsedTime`)를 사용했고, 실행 순서 편향을 없애려고 전체 시나리오를 2회(순서 반전) 돌렸다. **1회만 돌렸다면 "Polaris가 훨씬 빠르다"는 잘못된 결론이 나왔다** (1회차 hcatalog 는 콜드 스타트 영향으로 최대 2배 느리게 측정됨) |

### 12-2. 결정이 필요한 것

!!! success "쿠버네티스는 건드리지 않는 것으로 확정되었습니다"
    `trino-verify` · `juicefs` · `minio-verify` 의 기존 리소스는 모두 그대로 둔 상태입니다. 아래는 호스트 측·정책 측으로 남은 항목입니다.

| # | 결정 항목 | 현황과 선택지 |
| --- | --- | --- |
| B-1 | oracle21 호스트의 `openldap` 중지 여부 | 5일째 가동 중이며 이번 테스트와 무관하다. 다른 인증 검증용으로 보여 **임의로 내리지 않았다**. 정리 여부 알려주시면 바로 처리합니다 |
| B-2 | oracle21 의 `oracle-xe-21c` 컨테이너·이미지 삭제 여부 | 이미 **Exited** 상태로 멈춰 있다. 이미지까지 지우면 약 2.58 GB 확보 가능 |
| B-3 | **자격증명이 전부 평문이다** | 검증용 구성이라 그대로 두었다. 운영 전환 시 반드시 정리 필요 — 아래 12-3 표 참고 |
| B-4 | 벤치마크 데이터 보존 여부 | 버킷 `hcatalog` 498 MiB / `pcatalog` 498 MiB, 테이블 다수, Polaris 카탈로그 2개(`pcatalog`, `hms_ext`), Trino 네임스페이스 `iceberg-bench`(4 파드, 48Gi 요청)가 남아 있다. 정리 원하시면 처리합니다 |
| B-5 | **Materialized View 미지원이 의사결정에 영향을 주는지** | Polaris(REST 카탈로그)는 Trino 483 기준 MV 를 만들 수 없다. 현재 또는 향후 MV 계획이 있다면 이게 가장 큰 제약이며, 대안은 `Iceberg 테이블 + 스케줄드 INSERT OVERWRITE` 이다 |

### 12-3. 운영 전환 시 반드시 정리해야 할 보안 항목

| 항목 | 현재 (검증용) | 운영 시 권고 |
| --- | --- | --- |
| MinIO 키 (`icebergbench`) | Trino ConfigMap · HMS `hive-site.xml` 에 평문 | k8s Secret + `${ENV:...}` 참조 |
| MinIO 서비스 계정 권한 | **root 권한을 그대로 상속**(svcacct) | 버킷 2개로 제한한 정책 부여 |
| Polaris `root` 크레덴셜 | Trino ConfigMap 에 평문 | 전용 principal 생성 + 최소 권한 role |
| Oracle `hive` 비밀번호 | `hive-site.xml` 에 평문 | Hadoop Credential Provider 또는 Secret |
| Hive Metastore | **인증 없음**, 9083 호스트 포트 공개 | 네트워크 제한 또는 SASL/Kerberos |
| Trino | 인증·TLS 없음, NodePort 32085 공개 | 기존 `trino-verify` 처럼 LDAP + OPA 적용 |
| 교차 카탈로그 뷰 | 기본 DEFINER — 권한 우회 가능 (7-4절 참고) | `SECURITY INVOKER`  • Trino 측 접근제어 |

### 12-4. 요청에 없었지만 짚어둘 점 · 추가 제안

| # | 내용 |
| --- | --- |
| C-1 | **이 결과를 "Hive vs Iceberg" 로 읽으면 안 된다.** 양쪽 모두 Iceberg 테이블이고 다른 건 카탈로그뿐이다. 그래서 읽기 성능이 같게 나왔다. 한쪽을 **Hive 포맷 테이블**(파티션을 HMS 가 직접 관리)로 놓고 비교했다면 결과가 완전히 달라진다 — 파티션이 수천 개면 HMS 가 전부 RDBMS 에서 긁어와야 해서 플래닝이 수 초로 늘어난다. **필요하면 이 비교도 추가로 돌릴 수 있다** |
| C-2 | 이번 테이블은 **비파티션**이다. Iceberg 는 파티션 정보를 카탈로그가 아니라 매니페스트에 두므로 차이가 거의 없을 것으로 예상되지만, 그 "예상"을 실측으로 확인해두면 C-1 논점을 데이터로 못 박을 수 있다 |
| C-3 | **HMS 페더레이션을 정말 하고 싶다면 우회로가 있다.** 공식 이미지에 HIVE 커넥션 구현체가 없다는 게 확인됐으므로 — (1) HMS 앞에 Iceberg REST 어댑터를 두고 `connectionType=ICEBERG_REST` 로 페더레이션 (가장 현실적, 원하시면 바로 검증 가능), (2) Hive 번들 포함해 직접 빌드, (3) 정적 등록만 쓰고 읽기 전용 미러로 운영 |
| C-4 | 동시 커밋 실패 1건은 더 파볼 여지가 있다. 6라운드 중 1라운드에서 Polaris 쪽 8개 중 2개가 `409 CommitFailed` 로 실패했다. 스트리밍 도입을 고려한다면 **writer 수를 16~32로 올려 실패율 곱선을 그려보는 것**을 권한다 (Iceberg 테이블 속성 `commit.retry.num-retries` 조정 효과도 함께) |
| C-5 | 메타스토어 백엔드 Oracle 이 **Free 에디션**이다 — CPU 2코어 / RAM 2GB / 사용자 데이터 12GB 제한. 현재 hive 스키마는 4.31 MB 라 여유롭지만 **테이블 수만 개 규모의 운영 메타스토어**로 가면 이 제한이 병목이 된다. 이번 HMS 커밋 지연도 이 제한된 Oracle 기준값임을 감안해야 한다 |
| C-6 | Trino 리소스 설정이 임시다. 노드 메모리가 빠듯해 컨테이너 12Gi / `query.max-memory-per-node=6GB` 로 낮춰 잡았다. 다만 **두 카탈로그의 상대 비교에는 영향이 없다** (같은 클러스터에서 교차 실행했으므로) |
