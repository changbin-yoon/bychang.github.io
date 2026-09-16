---
title: Oracle 21c XE/26ai Free 기반 HMS 스키마 검증
---

# 🗄️ Oracle 21c XE / 26ai Free 기반 Hive Metastore 스키마 검증 보고서

!!! abstract "개요"
    2026-09-02, 두 대의 Ubuntu 서버에 Oracle Database를 Docker로 구성하고, Hive 3.1.3 메타스토어 스키마 초기화(`schematool -initSchema`) 및 PUBLIC SYNONYM 작업이 Oracle 버전에 관계없이 동일하게 동작하는지 검증

## 1. 테스트 환경

| 항목 | 서버 A (10.10.105.4) | 서버 B (10.10.105.27) |
| --- | --- | --- |
| OS | Ubuntu 24.04.4 LTS (Noble Numbat) | Ubuntu 24.04.4 LTS (Noble Numbat) |
| 리소스 | 32GB RAM, 154GB Disk | 32GB RAM, 154GB Disk |
| Docker | 29.7.2 (공식 apt repo로 설치) | 29.7.2 (공식 apt repo로 설치) |
| Oracle 이미지 | `gvenzl/oracle-xe:21-slim` | `container-registry.oracle.com/database/free:latest` |
| Oracle 컨테이너명 | `oracle-xe-21c` | `oracle-free-26ai` |
| Oracle 버전(배너) | Oracle Database 21c Express Edition Release 21.0.0.0.0 | Oracle AI Database 26ai Free Release 23.26.3.0.0 |
| PDB 서비스명 | `xepdb1` | `freepdb1` |
| 포트 | 1521 (DB), 5500 (EM Express) | 1521 (DB), 5500 (EM Express) |
| Hive 이미지 | `apache/hive:3.1.3` | `apache/hive:3.1.3` |
| Hive 실행 방식 | 1회성 컨테이너 (`--network host`), schematool 전용 실행 | 1회성 컨테이너 (`--network host`), schematool 전용 실행 |
| Oracle JDBC 드라이버 | `ojdbc8-21.9.0.0.jar` (Maven Central) | `ojdbc8-21.9.0.0.jar` (Maven Central) |
| Hive 메타스토어 DB 계정 | `hive` | `hive` |

!!! info
    원래 목표는 서버 A에 Oracle 19c XE 설치였으나, 19c XE는 사전 빌드된 공식/커뮤니티 Docker 이미지가 존재하지 않아(직접 RPM 다운로드 + 빌드 필요) 21c XE로 대체 확인했습니다. 서버 B는 최신 무료 버전인 26ai Free로 진행했습니다.

### 1-1. 아키텍처 구성

Hive schematool이 JDBC로 Oracle PDB에 접속해 메타스토어 74개 테이블을 생성하고, SYSDBA가 PUBLIC SYNONYM을 얹어 다른 계정이 스키마 접두사 없이 조회할 수 있게 만드는 흐름입니다. 이 구조는 서버 A(21c XE) · 서버 B(26ai Free) 양쪽에 동일하게 적용됩니다.

Hive 컨테이너는 상시 실행되는 서비스가 아니라, `schematool` 실행 한 번을 위해 떴다가 종료되는 **1회성 컨테이너**이며 `--network host` 로 붙어 `localhost:1521` 의 Oracle에 JDBC로 접속하는 구조입니다. 실제 스키마 · 데이터는 전부 Oracle 컨테이너(`hive` 계정) 안에 영구히 남아 있습니다.

## 2. 수행한 작업 절차

### 2-1. 인프라 준비

1. 두 서버 SSH 접속 확인(`ubuntu` 계정, 공개키 인증) 및 passwordless sudo 확인
2. Docker CE 공식 apt 저장소 등록 후 `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` 설치, `ubuntu` 계정을 `docker` 그룹에 추가
3. 접속 시점에 두 서버 모두 Oracle 컨테이너가 이미 25시간째 가동 중인 상태로 발견됨 (사전 설치 확인)

### 2-2. Oracle 상태 확인

- `docker inspect` 로 각 컨테이너의 `Config.Env` 를 조회해 `ORACLE_HOME` 경로(`/opt/oracle/product/21c/dbhomeXE` vs `/opt/oracle/product/26ai/dbhomeFree`)로 버전 특정
- `sqlplus ... as sysdba` 로 `v$pdbs`, `v$services` 조회 → PDB 서비스명 확정 (`xepdb1`, `freepdb1`)

### 2-3. Hive 메타스토어용 Oracle 계정 생성

각 PDB에 `hive` 계정을 생성하고 필요한 권한을 부여했습니다.

```sql
CREATE USER hive IDENTIFIED BY "HiveMeta_2026x";
GRANT CONNECT, RESOURCE TO hive;
GRANT CREATE TABLE, CREATE SEQUENCE, CREATE TRIGGER, CREATE VIEW TO hive;
GRANT UNLIMITED TABLESPACE TO hive;
```

(SYS 비밀번호는 컨테이너 환경변수 `ORACLE_PASSWORD`/`ORACLE_PWD` 에서 읽어와 화면에 노출하지 않고 사용)

### 2-4. Hive Metastore 스키마 초기화 (init-schema)

1. `apache/hive:3.1.3` 이미지 확인 → Java 8, `$HIVE_HOME/scripts/metastore/upgrade/oracle/` 에 Oracle용 DDL 스크립트 다수 존재(19c/21c/26ai 공통으로 Oracle 백엔드 정식 지원)
2. Oracle JDBC 드라이버(`ojdbc8-21.9.0.0.jar`, Maven Central)를 서버에 다운로드
3. `hive-site.xml` 작성 (`javax.jdo.option.ConnectionURL=jdbc:oracle:thin:@//localhost:1521/<pdb>` 등)
4. 1회성 컨테이너로 schematool 실행

```bash
docker run --rm --network host \
  -v ~/hive-metastore/lib/ojdbc8.jar:/opt/hive/lib/ojdbc8.jar \
  -v ~/hive-metastore/conf/hive-site.xml:/opt/hive/conf/hive-site.xml \
  --entrypoint /opt/hive/bin/schematool \
  apache/hive:3.1.3 -dbType oracle -initSchema -verbose
```

두 서버 모두 `Initialization script completed` / `schemaTool completed` 로 정상 종료됐습니다.

### 2-5. 스키마 검증 및 비교

- `hive` 계정으로 재접속해 `user_tables`, `VERSION` 테이블 확인 → 양쪽 모두 **74개 테이블**, `VERSION.SCHEMA_VERSION = 3.1.0`
- `user_tables`, `user_tab_columns`, `user_indexes`, `user_ind_columns`, `user_constraints`, `user_sequences` 를 덤프해 `diff` 로 전체 비교

### 2-6. PUBLIC SYNONYM 작업

1. `hive` 계정에서 `user_tables` 기준으로 `CREATE OR REPLACE PUBLIC SYNONYM <table> FOR hive.<table>;` DDL 74건 생성
2. `CREATE PUBLIC SYNONYM` 권한이 없는 `hive` 계정 대신 SYSDBA로 DDL 실행
3. `all_synonyms` 에서 `owner='PUBLIC' AND table_owner='HIVE'` 로 개수 / 누락 여부 검증

## 3. 결과

| 검증 항목 | 서버 A (21c XE) | 서버 B (26ai Free) |
| --- | --- | --- |
| schematool initSchema | 성공 | 성공 |
| 생성된 테이블 수 | 74 | 74 |
| VERSION.SCHEMA_VERSION | 3.1.0 | 3.1.0 |
| 테이블/컬럼 정의 diff | 기준 | **0건 차이** (완전 일치) |
| 인덱스/제약조건 구조 | 기준 | **0건 차이** (완전 일치) |
| PUBLIC SYNONYM 생성 | 74/74 성공 | 74/74 성공 |
| PUBLIC SYNONYM 검증(`all_synonyms`) | 74건, 누락 0 | 74건, 누락 0 |

유일하게 다른 값은 `user_indexes`/`user_constraints` 의 **시스템 자동 생성 객체명**(`SYS_C008326` vs `SYS_C008677`, `SYS_IL0000075984...` 등)뿐이며, 이는 각 Oracle 인스턴스가 독립적으로 부여한 내부 object ID 시퀀스 차이일 뿐 구조적 차이가 아닙니다.

## 4. Oracle 21c XE ↔ 26ai Free 버전 차이점

### 4-1. 이번 작업에서 실제로 확인된 차이

| 구분 | 21c XE | 26ai Free |
| --- | --- | --- |
| 버전 배너 | `Oracle Database 21c Express Edition Release 21.0.0.0.0` | `Oracle AI Database 26ai Free Release 23.26.3.0.0` (내부적으로 23ai 계열의 23.26.3 릴리스) |
| ORACLE_HOME 경로 | `/opt/oracle/product/21c/dbhomeXE` | `/opt/oracle/product/26ai/dbhomeFree` |
| ORACLE_SID | `XE` | `FREE` |
| 기본 PDB 서비스명 | `xepdb1` | `freepdb1` |
| 공식 Docker 이미지 | **없음** (Oracle Container Registry 미제공, 커뮤니티 이미지 `gvenzl/oracle-xe` 로 대체) | **있음** (`container-registry.oracle.com/database/free`, 로그인 / 라이선스 동의 후 pull) |
| 컨테이너 관리자 비밀번호 env 변수명 | `ORACLE_PASSWORD` (gvenzl 이미지 기준) | `ORACLE_PWD` (Oracle 공식 이미지 기준) — Oracle 버전 차이라기보다 이미지 제작 주체(커뮤니티 vs 공식) 차이 |
| 제품 브랜딩 | Oracle **Database** | Oracle **AI Database** (23ai부터 AI 기능 강조를 위해 브랜드명 자체가 변경됨) |

### 4-2. 이번 작업 범위 밖의 공식 스펙 차이 (참고용)

| 항목 | 21c XE | 26ai(23ai 계열) Free |
| --- | --- | --- |
| CPU 제한 | 2 코어 | 2 코어 (동일) |
| RAM 제한 | 2 GB | 2 GB (동일) |
| 사용자 데이터 제한 | 12 GB | 12 GB (동일) |
| PDB 개수 제한 | 제한 있음(버전별 상이) | 최대 16개 |
| 신규 기능 | - | JSON Relational Duality, SQL Domains, AI Vector Search, True Cache 등 23ai 이후 신규 기능 다수 포함 |
| 라이선스 | 무료(XE 프로그램) | 무료(Free 프로그램), 21c XE 단종 후 후속 프로그램으로 전환 |

!!! warning
    4-2 표는 이번 세션에서 직접 실측한 값이 아니라 Oracle 공식 문서 / 기술 블로그 기반 조사 결과이므로 실제 운영 적용 전 최신 공식 문서 재확인을 권장합니다.

## 5. 공통점 (버전 무관하게 동일하게 동작한 부분)

- **Hive 메타스토어 스키마 자체** — `hive-schema-3.1.0.oracle.sql` 기반 DDL이 두 버전에서 완전히 동일한 74개 테이블 / 컬럼 / 인덱스 / 제약조건 구조로 생성됨
- **JDBC 호환성** — 동일한 `ojdbc8-21.9.0.0.jar` 드라이버로 두 버전 모두 문제없이 연결 · DDL 실행 가능 (thin 드라이버의 하위 / 상위 호환성 확인)
- **계정 / 권한 부여 절차** — `CONNECT, RESOURCE, CREATE TABLE/SEQUENCE/TRIGGER/VIEW, UNLIMITED TABLESPACE` 권한 세트로 동일하게 충분
- **PUBLIC SYNONYM 생성 절차 및 권한 요구사항** — 두 버전 모두 일반 계정은 `CREATE PUBLIC SYNONYM` 시스템 권한이 기본적으로 없어 SYSDBA 실행이 필요했고, DDL 문법도 완전히 동일하게 통과
- **schematool 동작** — `-dbType oracle -initSchema` 옵션 및 로그 메시지(`Initialization script completed`, `schemaTool completed`) 형식이 동일

## 6. 결론

Oracle 21c XE와 26ai Free 사이에는 배포 방식(공식 이미지 유무), 내부 경로 / 서비스명 네이밍, 브랜딩 등의 차이는 있으나, **Hive 3.1.3 메타스토어 스키마 초기화와 PUBLIC SYNONYM 작업 관점에서는 절차 · SQL · 권한 요구사항이 완전히 동일**하며 두 버전 모두 문제없이 재현 가능함을 확인했습니다.

## 7. 참고 자료

- [Oracle AI Database Free 26ai Released — ORACLE-BASE Blog](https://oracle-base.com/blog/2025/10/15/oracle-ai-database-26ai-released/)
- [Oracle AI Database 26ai Get Started](https://docs.oracle.com/en/database/oracle/oracle-database/26/)
- [Oracle AI Database Free Licensing Restrictions (26ai)](https://docs.oracle.com/en/database/oracle/oracle-database/26/xeinl/licensing-restrictions.html)
- [Oracle Database 21c XE Licensing Restrictions](https://docs.oracle.com/en/database/oracle/oracle-database/21/xeinl/licensing-restrictions.html)
- [gvenzl/oracle-xe Docker Hub](https://hub.docker.com/r/gvenzl/oracle-xe)
- [Oracle Database container images (GitHub)](https://github.com/oracle/docker-images/blob/main/OracleDatabase/SingleInstance/README.md)
