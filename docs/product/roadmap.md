# 로드맵 (ROADMAP)

**목적:** Phase 1/2 작업 진척을 한 페이지에서 추적한다. 작업이 끝날 때마다 체크박스를 갱신하고, 막힌 지점은 메모로 남긴다.
**최종 갱신:** 2026-05-16

---

## 현재 레포 상태

| 레포 | 상태 |
|---|---|
| `cheeseboard-infra` | Postgres 18 + Redis 컴포즈 운영. `init.sql` 제거 완료, Flyway가 DB 스키마의 진실 공급원. |
| `cheeseboard-back` | Spring Boot 3.4 + JDK 21. Flyway V1/V2/V3, JPA, Security, 네이버 OAuth 로그인 골격, JWT 쿠키 인증, `/api/auth/me`, `/api/auth/logout` 구현. 검색/클립 도메인 미착수. |
| `cheeseboard-crawler` | FastAPI + PostgreSQL 연동 완료. API Key 보호, CHZZK API 수집, 스트리머 관리, 크롤 잡/재시도, 단건 refresh hook, APScheduler 정기 잡 구현. Elasticsearch 연동 미구현. |
| `cheeseboard-front` | Vite+React+TS + React Router + TanStack Query/Zustand + API 클라이언트 골격. `HomePage`/`SearchPage`는 mock 데이터 기반 UI 구현 상태. |

> 레포별 상세 진척은 [CLAUDE.md](../CLAUDE.md)의 "레포지토리 구성" 표가 아니라 이 문서를 우선 참고한다.

---

## Phase 1 — MVP

서비스가 "검색되는 치지직 VOD/클립 탐색 사이트"로 동작하기 위한 최소 기능. 비로그인도 핵심 기능 사용 가능.

### 디자인
- [x] 와이어프레임 1단계: 페이지 스위처 + 메인 + 헤더 로그인 버튼
- [x] 와이어프레임 2단계: 검색 결과 페이지 (필터 바, VOD/클립 뱃지)
- [ ] 컬러 팔레트/간격 토큰화 — 와이어프레임의 CSS 변수를 Tailwind config 또는 디자인 토큰으로 정리

### 프론트 (Phase 1 기능 구현)
- [ ] `HomePage.tsx` — 인기 클립 피드 그리드 UI는 mock 데이터로 구현. 실제 `usePopularClips()` 연결 + 무한스크롤 남음 ([FR-01](./functional-spec.md#fr-01-인기-클립-피드))
- [ ] `SearchPage.tsx` — 검색 결과/정렬/타입/스트리머 필터 UI는 mock 데이터로 구현. 실제 `useSearch()` 연결 + 무한스크롤 남음 ([FR-02~05](./functional-spec.md))
- [x] 영상 카드 컴포넌트 — VOD/클립 뱃지 포함, 썸네일·제목·메타·길이
- [x] 클립 재생 모달 + VOD 새 탭 이동 ([FR-06](./functional-spec.md#fr-06-영상-재생))
- [ ] 검색창 자동완성 (스트리머명, 300ms debounce) ([FR-08](FUNCTIONAL_SPEC.md#fr-08-검색-자동완성))
- [ ] 헤더 로그인 버튼 + 네이버 OAuth 진입 ([FR-10](./functional-spec.md#fr-10-네이버-로그인)) — 버튼 UI만 있음, OAuth 이동 연결 남음
- [ ] 로그인 후 헤더 프로필 표시/로그아웃 드롭다운
- [ ] 사용자 태그 기여 UI — 카드/모달 내 태그 칩 + 추가 입력 ([FR-12](FUNCTIONAL_SPEC.md#fr-12-사용자-태그-기여))

### 백엔드 (Phase 1 기능 구현)
- [ ] 클립/VOD 도메인 + 인기 클립 API (`GET /api/clips/popular`)
- [ ] 검색 API (`GET /api/search`) — Elasticsearch + nori 연동
- [ ] 스트리머 목록 API (`GET /api/streamers`)
- [ ] 자동완성 API (`GET /api/search/suggest`)
- [ ] 사용자 태그 CRUD API + 비속어 필터링
- [ ] Redis 캐싱 — INFRA.md 네임스페이스 규칙대로

### 크롤러
- [x] CHZZK VOD/클립 수집 + PostgreSQL 적재
- [ ] Elasticsearch 인덱싱 동기화 (적재 시점 또는 별도 sync 잡)
- [x] 정기 크롤 스케줄링 (`hot_clips_poll`, `latest_videos_poll`, `channel_clips_incremental`, `weekly_reconciliation`)

### 인프라
- [x] `cheeseboard-infra/init.sql` 삭제
- [ ] Elasticsearch 컨테이너 추가 (현재 Postgres + Redis만)
- [ ] OCI 운영 인프라 — [INFRA.md](INFRA.md) 설계대로 배포

---

## Phase 2 — MVP 이후

### 디자인
- [ ] 와이어프레임 3단계: 주제 목록(`/topics`) + 주제 상세(`/topics/{slug}`)
- [ ] 와이어프레임 4단계: 관리자 페이지(`/admin/*`) — 대시보드 / 주제 CRUD / 태그 모더레이션
- [ ] 콜라주 커버 모형 — 4/3/2/1/0장 영상일 때 그리드 패턴 시각화

### 명세 정합 (구현 전)
- [ ] [database-schema.md](../architecture/database-schema.md)에 `topics`, `topic_items` 테이블 추가
- [ ] Flyway 마이그레이션 `V3__topics.sql`
- [ ] 관리자 인증 모델 확정 — `users.role` vs 별도 `admin_users` 화이트리스트 ([FR-16](FUNCTIONAL_SPEC.md#fr-16-관리자-페이지-phase-2))
- [ ] [api/spec.md](../api/spec.md)에 토픽 조회/큐레이션 엔드포인트 추가

### 프론트 (Phase 2 기능 구현)
- [ ] 메인 페이지 히어로 캐러셀 (자동 순회 + 좌우 화살표/닷, 0개일 때 미노출) ([FR-15](./functional-spec.md#fr-15-주제별-모아보기-phase-2))
- [ ] 콜라주 커버 컴포넌트 — `videos.thumbnail_url`/`clips.thumbnail_url` CSS 그리드 합성
- [ ] `TopicsPage.tsx` (주제 목록) + `TopicDetailPage.tsx` (주제 상세)
- [ ] `AdminPage.tsx` + 하위 라우트 + 화이트리스트 접근 제어 ([FR-16](./functional-spec.md#fr-16-관리자-페이지-phase-2))
- [ ] 라우팅 추가 (`/topics`, `/topics/:slug`, `/admin/*`)

### 백엔드 (Phase 2 기능 구현)
- [ ] 토픽 조회 API (`GET /api/topics`, `GET /api/topics/{slug}`)
- [ ] 토픽 관리 API (관리자 전용 CRUD) + 영상 큐레이션
- [ ] 사용자 태그 모더레이션 API (관리자 전용)
- [ ] 외부 영상 연동 ([FR-13](./requirements.md)) — 범위 미정
- [ ] 스트리머 자동 확장 ([FR-14](./requirements.md)) — 실시간 방송 상위 N명 주기적 편입

---

## 작업 우선순위 메모

- **즉시**: Phase 1 백엔드/프론트 핵심 기능 (검색 → 인기 피드 API 연결 → 로그인 버튼 OAuth 연결 → 태그 기여)
- **Phase 1 완료 후**: 와이어프레임 3·4단계 → Phase 2 명세 정합(DB/API) → Phase 2 구현
- **언제든**: 컬러 토큰화, ROADMAP 갱신, ADR 작성 (큰 의사결정 발생 시)

---

## 갱신 규칙

- 작업이 끝나면 해당 체크박스를 `[x]`로 바꾸고 "최종 갱신" 날짜를 갱신한다.
- 새 작업이 식별되면 적절한 Phase / 영역에 추가한다.
- 큰 의사결정은 [decisions/index.md](../decisions/index.md)에 ADR로, 일별 작업은 `work-logs/`에 기록한다. 이 문서는 "체크리스트"만 유지한다.
