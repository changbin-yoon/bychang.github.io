---
title: 01. 인증/IAM — LDAP 연동 Access Key
---

# 🔐 인증/IAM — LDAP 연동 Access Key 발급과 권한 전파 검증

!!! abstract "개요"
    **목적** — LDAP 연동 환경에서 발급한 Access Key가 권한 변경 시 어떻게 평가되고, 변경이 언제 전파되는지 확인

    **배경** — 권한 변경 → 원복 후에도 간헐적 Access Denied가 발생해 동작 원리를 검증

    **결론 한 줄** — 권한은 요청마다 LDAP을 조회해 평가되지 않고 **노드별 in-memory IAM 캐시**로 평가되며, 노드 간 갱신 시점이 달라 **과도기 불일치 구간이 존재**한다

## 1. 확인하려는 것

- LDAP 계정으로 발급한 Access Key(service account)의 권한 평가 경로
- LDAP에서 권한을 바꾸면 클러스터에 언제 반영되는지
- 즉시 반영이 필요할 때 쓸 수 있는 수단과 그 영향 범위

## 2. 권한 평가가 일어나는 경로

요청 한 건이 처리되는 순서입니다.

1. **서명 검증** — access key / secret key로 각 노드가 로컬에서 처리. **LDAP에 가지 않는다.**
2. **권한 평가** — 해당 key의 parent user(LDAP DN)에 매핑된 policy 기준. 매핑은 `.minio.sys` 하위 IAM 저장소에 있고, **각 노드가 메모리에 캐싱**해두고 쓴다.
3. **캐시 갱신** — 주기적 전체 리로드 + 매핑 변경 이벤트 발생 시 peer notification.
4. **LDAP sweep** — 백그라운드로 parent DN 존재 여부와 그룹 멤버십을 재확인.

!!! important
    권한 평가의 기준은 LDAP도, 백엔드 저장소도 아닌 **요청을 받은 노드의 메모리**입니다.
    LB 뒤에서 노드마다 캐시 상태가 다르면 **같은 키의 같은 요청이 200과 403으로 갈립니다.**

## 3. 리프레시 주기 — 설정 가능한가

`mc admin config` 에서 찾을 수 없어 upstream 소스를 확인했습니다.

- `cmd/globals.go` — `globalRefreshIAMInterval = 10 * time.Minute` 상수로 정의되며 기동 시 `globalIAMSys.Init()` 에 그대로 전달
- `cmd/iam.go` periodic routine — 전 노드 동시 리프레시로 부하가 몰리는 것을 피하기 위해 **base interval의 0.5~1.5배 사이 랜덤 시점**에 수행

결과적으로 **노드별 실효 주기는 5~15분**이고, LDAP 유저 purge / 그룹 멤버십 sweep은 **시간당 1회**입니다.

!!! warning "AIStor 차이 주의 — 미확인 항목"
    위는 upstream(AGPL) 소스 기준입니다. AIStor 문서에는 OIDC 그룹 멤버십 리프레시가 30~90분 사이로 변동한다는 식으로 upstream과 다른 주기가 명시된 부분이 있습니다.
    LDAP policy mapping 캐시 주기가 AIStor에서 설정 가능해졌는지는 공개 문서로 확인되지 않으므로 **SUBNET 문의가 필요**합니다.

## 4. 진단 절차

```bash
# 1. Access Key의 parent user DN 확인
mc admin user svcacct info <alias> <accesskey>

# 2. DN - policy 매핑이 변경 전과 "정확히 같은 문자열"인지 비교
#    대소문자, 공백, cn= vs CN= 까지 확인
mc idp ldap policy entities <alias>

# 3. 실패 시점에 어느 노드가 403을 주는지 추적
#    특정 노드만 거부하면 캐시 불일치
mc admin trace <alias> --errors
```

!!! tip "놓치기 쉬운 부차 원인 — DN 문자열 불일치"
    권한을 원복하는 과정에서 그룹 / 유저 DN의 **대소문자 · 공백 · 표기가 미세하게 달라지면** policy 매핑 lookup이 조용히 실패합니다. sweep 타이밍과 겹쳐 캐시 문제처럼 보이므로 반드시 함께 확인합니다.

관측은 `minio_cluster_iam_*` 계열(마지막 sync 경과 시간, sync 소요 시간, 성공 / 실패 카운트)을 **노드별로 나란히** 보는 게 가장 빠릅니다. 리프레시가 수 초 이상 걸리면 서버 로그에 `IAM refresh took` 경고도 남습니다.

## 5. 즉시 전파 방법

주기를 줄일 수 없으니, **매핑 변경 이벤트를 인위적으로 발생시켜 peer notification을 타게 하는** 방식을 씁니다.

```bash
mc idp ldap policy detach ALIAS policyname --user="uid=xxx,ou=...,dc=..." && \
mc idp ldap policy attach ALIAS policyname --user="uid=xxx,ou=...,dc=..."
```

**간격을 두지 않는 게 핵심입니다.**

1. detach된 순간부터 해당 DN의 모든 access key가 **전 노드에서 거부**됩니다. detach도 즉시 전파되므로 간격을 벌릴수록 실서비스 거부 시간만 길어집니다.
2. 목적은 "시간을 기다리는 것"이 아니라 **"갱신 이벤트를 발생시키는 것"** 입니다. 일부 노드에 notification이 유실돼도 주기적 리로드에서 최종 상태(attach)로 수렴합니다.

두 명령을 이어 붙이면 거부 구간이 **1초 미만**이라 S3 클라이언트 retry 범위에서 흡수됩니다. 실행 후 `mc idp ldap policy entities` 로 정확한 DN 문자열로 붙었는지 확인합니다.

### 5.1 대안 — `mc admin service restart` 의 영향 범위

확실하지만 비용이 큽니다. 동작을 정확히 알고 써야 합니다.

- **rolling이 아니다.** alias가 가리키는 deployment의 **전 노드가 거의 동시에** 재시작됩니다. 바이너리가 스스로 프로세스를 교체(re-exec)하므로 컨테이너는 안 죽지만 **수 초간 신규 요청을 못 받습니다.**
- in-memory 상태가 전부 초기화됩니다 — IAM 캐시, bucket metadata 캐시, 진행 중 connection.
- 대규모에서는 기동 시 erasure set 초기화 · format 확인으로 노드당 수 초~수십 초 소요.
- **in-flight 요청은 끊깁니다.** 진행 중이던 멀티파트 업로드의 전송 중 part는 재시도 필요.
- 고사용률 클러스터라면 재기동 직후 scanner 재개 부하도 감안.

대형 Spark 잡이 상시로 도는 환경이라면 **Iceberg commit 직전 단계를 피하는 게 안전**합니다. commit 자체는 원자적이지만 쓰기 실패로 task retry가 대량 발생하면 잡 지연으로 번집니다.

## 6. 정리

| 항목 | 확인 결과 |
| --- | --- |
| 권한 평가 기준 | 노드별 in-memory IAM 캐시 (요청당 LDAP 조회 아님) |
| 기본 리프레시 주기 | 10분 상수, 노드별 랜덤으로 실효 5~15분 |
| 설정 변경 가능 여부 | upstream은 불가 (재빌드 필요), AIStor는 미확인 |
| 즉시 전파 수단 | policy detach 후 즉시 attach (간격 없이 연속) |
| 사이드 이펙트 | detach 구간 중 전면 거부 (1초 미만) |

**운영에 반영할 것**

1. LDAP 권한 변경 / 원복 절차에 **detach 후 attach를 표준 단계로 포함**한다.
2. 변경 전후 `mc idp ldap policy entities` 출력을 diff해 **DN 문자열을 검증**한다.
3. "일부 노드만 한동안 stale"은 **설계상 정상**이므로, 권한 변경 직후 단발성 403 알람에는 dedup / grace를 둔다.
4. 노드별 IAM 리프레시 대시보드를 상시 노출해 다음엔 **stale 노드를 바로 특정**한다.
