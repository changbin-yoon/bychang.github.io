---
title: T-Hadoop 레거시 운영
---

# T-Hadoop (Vanilla Hadoop) 레거시 운영

| 항목 | 내용 |
| --- | --- |
| 기간 | 2021 ~ 2025 |
| 역할 | 클러스터 일상 운영, HW 교체 절차 수립·수행, 이슈 대응 |
| 핵심 기술 | Hadoop(HDFS, YARN), Ceph, Hive, Trino |

## 클러스터 운영

- Hadoop 클러스터 **일일 점검**
- **월간 리소스 사용량 보고서** 작성 (모니터링 툴 데이터 취합)

## 노후화 서버 교체

Bigdata Cluster 노후 서버 교체를 위해 **HW Part 온·오프라인 교체 절차서를 수립하고 직접 수행**했습니다.

- 대상: OS Disk / SSD / HDD / Memory / NIC
- Ceph OSD 교체

## 이슈 대응

- hive-server2 이슈
- UDP Network 오류
- Hadoop → Ceph 이관 테스트
- Trino Graceful Shutdown 테스트
