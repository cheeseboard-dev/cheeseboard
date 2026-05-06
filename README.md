# CheeseBoard

치지직(CHZZK) VOD·클립을 스트리머·키워드·정렬 기준으로 탐색하는 통합 검색 서비스.

## 레포지토리 구성

| 레포 | 역할 | 스택 |
|---|---|---|
| [cheeseboard-front](https://github.com/cheeseboard-dev/cheeseboard-front) | 프론트엔드 | React + TypeScript + Vite |
| [cheeseboard-back](https://github.com/cheeseboard-dev/cheeseboard-back) | 백엔드 API | Spring Boot + PostgreSQL + Elasticsearch |
| [cheeseboard-crawler](https://github.com/cheeseboard-dev/cheeseboard-crawler) | 크롤러 | Python + FastAPI |
| [cheeseboard-infra](https://github.com/cheeseboard-dev/cheeseboard-infra) | 인프라 | Docker Compose + OCI |

## 데이터 흐름

```
CHZZK API → FastAPI 크롤러 → PostgreSQL (원본) + Elasticsearch (검색 인덱스)
                                        ↓
                             Spring Boot API → React 프론트
```

## 문서

| 문서 | 내용 |
|---|---|
| [PROJECT_OVERVIEW.md](docs/PROJECT_OVERVIEW.md) | 프로젝트 목적·배경 |
| [REQUIREMENTS.md](docs/REQUIREMENTS.md) | 기능/비기능 요구사항 |
| [FUNCTIONAL_SPEC.md](docs/FUNCTIONAL_SPEC.md) | 기능 명세, 화면별 동작 |
| [API_SPEC.md](docs/API_SPEC.md) | REST API 엔드포인트 명세 |
| [DB_SCHEMA.md](docs/DB_SCHEMA.md) | PostgreSQL 스키마 DDL |
| [INFRA.md](docs/INFRA.md) | 인프라 설계 (로컬 Docker + OCI) |
| [SCRAPER_GUIDE.md](docs/SCRAPER_GUIDE.md) | 크롤러 수집 전략 |
| [DECISIONS.md](docs/DECISIONS.md) | 아키텍처 결정 기록 |
