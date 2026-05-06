# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

**치즈보드 (CheeseBoard)** — 치지직(CHZZK, Naver 스트리밍 플랫폼) VOD와 클립을 스트리머·키워드·정렬 기준으로 탐색하는 통합 검색 서비스.

## 레포지토리 구성

| 레포 | 역할 | 상태 |
|---|---|---|
| `cheeseboard-front` | React + TypeScript + Vite | 미착수 |
| `cheeseboard-back` | Spring Boot + PostgreSQL + Elasticsearch | 미착수 |
| `cheeseboard-crawler` | Python + FastAPI 스크래퍼 | 스크립트 단계 |
| `cheeseboard-infra` | Docker Compose, 인프라 설정 | 로컬 운영 중 |

GitHub 조직: https://github.com/cheeseboard-dev

## 로컬 개발 환경

```bash
cd cheeseboard-infra
cp .env.example .env   # 비밀번호 설정 후
docker compose up -d   # PostgreSQL 16 → localhost:5432
```

## 데이터 흐름

```
CHZZK API → FastAPI 스크래퍼 → PostgreSQL (원본) + Elasticsearch (검색 인덱스)
                                         ↓
                              Spring Boot API → React 프론트
```

## 문서 목차

| 문서 | 내용 |
|---|---|
| [PROJECT_OVERVIEW.md](docs/PROJECT_OVERVIEW.md) | 프로젝트 목적, 배경, 전체 개요 |
| [REQUIREMENTS.md](docs/REQUIREMENTS.md) | 기능/비기능 요구사항 정의 |
| [FUNCTIONAL_SPEC.md](docs/FUNCTIONAL_SPEC.md) | 기능 명세, 화면별 동작 정의 |
| [API_SPEC.md](docs/API_SPEC.md) | REST API 엔드포인트 명세 |
| [DB_SCHEMA.md](docs/DB_SCHEMA.md) | PostgreSQL 테이블 스키마 DDL |
| [INFRA.md](docs/INFRA.md) | 로컬 Docker 환경 + OCI 운영 인프라 설계 |
| [SCRAPER_GUIDE.md](docs/SCRAPER_GUIDE.md) | 크롤러 수집 전략, 실행 방법, API 패턴 |
| [design/wireframe.html](docs/design/wireframe.html) | UI 와이어프레임 |
| [DECISIONS.md](docs/DECISIONS.md) | 아키텍처 결정 기록 (왜 이렇게 했나) |

## 코드 작업 시 체크리스트

### 크롤러 (cheeseboard-crawler)
- [ ] 외부 API 호출은 세마포어(`CRAWL_CONCURRENCY`)로 동시성 제한했는가?
- [ ] 크롤 작업 시작/종료/실패를 `crawl_jobs` 테이블에 기록했는가?
- [ ] 페이지네이션이 명시된 패턴을 따르는가? (VOD: `?page=N`, 클립: `?clipUID={cursor}`)
- [ ] CHZZK API 응답 파싱 시 nullable 필드를 안전하게 처리하는가?

### DB 스키마 변경
- [ ] 새 surrogate PK는 `UUID PRIMARY KEY DEFAULT uuidv7()`을 쓰는가?
- [ ] 새 FK에 `ON DELETE` 동작이 명시되어 있는가?
- [ ] `init.sql`과 `docs/DB_SCHEMA.md`가 동기화되어 있는가?
- [ ] 아키텍처 결정에 해당하면 ADR을 먼저 작성했는가?

### Spring Boot API
- [ ] Phase 1 기능인가, Phase 2인가? Phase 2는 `users` 테이블 의존 없이 동작해야 한다.
- [ ] 캐시 키 패턴이 INFRA.md의 Redis 네임스페이스 규칙과 일치하는가?
