# changbin-yoon.github.io

> **이 저장소는 포트폴리오(Portfolio)를 위한 개인 프로젝트입니다.**
> 진행한 프로젝트, 업무 경험, 기술 스택을 정리해 웹사이트로 공개하는 것이 목적입니다.

[![CI](https://github.com/changbin-yoon/changbin-yoon.github.io/actions/workflows/ci.yml/badge.svg)](https://github.com/changbin-yoon/changbin-yoon.github.io/actions/workflows/ci.yml)
[![Deploy](https://github.com/changbin-yoon/changbin-yoon.github.io/actions/workflows/deploy.yml/badge.svg)](https://github.com/changbin-yoon/changbin-yoon.github.io/actions/workflows/deploy.yml)

## 사이트 주소

https://changbin-yoon.github.io/portfolio/

## 기술 구성

| 항목 | 사용 기술 |
| --- | --- |
| 정적 사이트 생성기 | [MkDocs](https://www.mkdocs.org/) |
| 테마 | [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) |
| CI | GitHub Actions (`mkdocs build --strict`) |
| CD | GitHub Actions → GitHub Pages |

## 디렉터리 구조

```
.
├── .github/workflows/
│   ├── ci.yml          # PR·기능 브랜치 빌드 검증
│   └── deploy.yml      # main 브랜치 → GitHub Pages 배포
├── docs/
│   ├── index.md            # 홈 (핵심 요약·기술 스택)
│   ├── about/              # 소개 (프로필·학력·보유 기술)
│   ├── experience/         # 경력 (미리비트·커미조아)
│   ├── projects/           # 대표 프로젝트 7건
│   ├── work/               # 업무내역 — 조사 기록 25건
│   │   ├── index.md            # 분류 요약
│   │   ├── troubleshooting/    # 장애·증상 대응 9건
│   │   ├── poc/                # 도입·구성 검토 5건
│   │   └── verification/       # 동작·성능·비교 검증 11건
│   └── contact/            # 연락처
├── mkdocs.yml          # 사이트 설정
└── requirements.txt    # Python 의존성
```

## 로컬에서 실행하기

Python 3.12 이상이 필요합니다.

```bash
# 가상환경 생성 및 활성화
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

# 의존성 설치
pip install -r requirements.txt

# 개발 서버 실행 (http://127.0.0.1:8000)
mkdocs serve

# 배포와 동일한 조건으로 빌드 검증
mkdocs build --strict
```

## 배포 방식

`main` 브랜치에 push하면 `deploy.yml` 워크플로가 사이트를 빌드해 GitHub Pages로 배포합니다.
그 외 브랜치와 Pull Request는 `ci.yml`에서 `--strict` 빌드로 검증만 수행합니다.

### 최초 1회 설정

GitHub 저장소의 **Settings → Pages → Build and deployment → Source** 를
**GitHub Actions** 로 변경해야 배포가 동작합니다.

## 내용 추가하기

1. `docs/` 아래에 Markdown 파일을 추가합니다.
2. `mkdocs.yml`의 `nav`에 해당 경로를 등록합니다.
3. `main`에 push하면 자동으로 배포됩니다.
