---
title: 스토리지 도입·평가·이관
---

# 스토리지 도입, 평가, 이관

| 항목 | 내용 |
| --- | --- |
| 기간 | 2024 ~ 2026 |
| 역할 | 스토리지 성능 검증, 도입 선정, 객체 스토리지 이관 |
| 핵심 기술 | Isilon, Ceph(Rook), MinIO/AIStor, DirectPV, S3A, fio |

## 스토리지 성능 테스트 및 선정

- **fio** 기반 성능 측정 — Isilon(SSD / HDD), Ceph(RBD / CephFS)
- **RWO / RWX 2가지 모드**로 순차 / 랜덤 / IOPS 측정·비교

!!! success "결과"
    운영 스토리지를 **Isilon 기반으로 선정**하고, 측정 결과를 근거로 StorageClass를 구성했습니다.

## Ceph

- Rook-Ceph 설치 및 테스트, **External Ceph 연동**
- CSI-RBD / CSI-CephFS 드라이버, StorageClass 생성

## MinIO / AIStor (객체 스토리지)

- 구축 및 버전 업그레이드
- **HDFS → MinIO 데이터 이관**
- DirectPV 디스크 구성
- **AirGapped 설치 런북·가이드 작성**
- 시스템별(Dev / Prd) 권한 정책(MinIO Policy) 구성 및 운영

## HDFS → 객체 스토리지 이관

- **S3A** 연동, **jceks Credential 암호화** 적용
- Sample Data 이관 및 PoC, 성능 테스트 진행
