# 아키텍처 결정 기록

나중에 "왜 이렇게 했지?"에 답하기 위한 메모. 결정 당시의 맥락과 기각한 대안을 보존한다.

---

## 001 — Surrogate PK는 UUID v7 사용

**날짜:** 2026-05-07

**결정:** 자연 키가 없는 테이블(`crawl_jobs`, `video_user_tags`, `clip_user_tags`)의 PK를 `UUID PRIMARY KEY DEFAULT uuidv7()`로 통일한다. 자연 키가 있는 테이블(`streamers.channel_id`, `videos.video_no`, `clips.clip_uid`)은 CHZZK API 원본 식별자를 그대로 사용한다.

**이유:**
- UUID v7은 Unix 타임스탬프 prefix → B-Tree 인덱스 삽입이 시간순으로 정렬되어 페이지 분할(page split) 감소
- INSERT 전에 ID를 미리 알 수 있어 로그 추적이 편함
- PG18이 `uuidv7()` 내장 지원 예정이라 외부 확장 불필요

> **확인 필요:** 컨테이너 기동 후 `SELECT uuidv7();` 실행. 실패 시 `CREATE EXTENSION IF NOT EXISTS pg_uuidv7;` 추가.

**기각한 대안:**
- **BIGSERIAL** — 단순하고 가독성 좋지만, 수평 확장 시 단일 시퀀스 병목 + INSERT 전 ID 미리 알 수 없음. 나중에 바꾸면 마이그레이션 비용이 크다.
- **UUID v4** — 완전 랜덤이라 B-Tree 인덱스 핫스팟 발생. v7 대비 이점 없음.
- **ULID** — v7과 특성이 비슷하지만 PostgreSQL 네이티브 타입이 아니라 확장 필요. v7이 있으면 굳이 쓸 이유 없음.

---

## 002 — 사용자 태그를 콘텐츠 타입별로 분리

**날짜:** 2026-05-07

**결정:** `user_tags` 단일 테이블 대신 `video_user_tags`와 `clip_user_tags`를 별도 테이블로 분리한다.

**이유:**
- 단일 테이블로 만들면 `content_id VARCHAR`로는 `videos.video_no`나 `clips.clip_uid` 중 어느 쪽과도 FK를 걸 수 없어 DB 참조 무결성이 없다.
- 분리하면 FK + `ON DELETE CASCADE`로 부모 삭제 시 태그가 자동 정리된다.
- API 엔드포인트 구조(`/api/videos/{no}/tags`, `/api/clips/{uid}/tags`)와도 자연스럽게 대응된다.

**기각한 대안:**
- **다형 참조 (content_type + content_id VARCHAR)** — 테이블 1개로 단순하지만 FK 불가. DB가 보장해야 할 무결성을 애플리케이션 코드로 구현해야 하는 구조는 피하고 싶었다.
- **PostgreSQL table inheritance** — FK 제약이 자식 테이블에 전파되지 않아 결국 같은 문제.

---

## 003 — 네이버 OAuth 단독, 로컬 로그인 없음

**날짜:** 2026-05-07

**결정:** 네이버 OAuth 2.0만 사용한다. AT(단명 JWT, DB 미저장)와 RT(Redis, key: `refresh:{naver_id}`, TTL 14일) 패턴으로 구현한다.

**이유:**
- 서비스 대상이 치지직 시청자 → 네이버 계정을 이미 갖고 있어 가입 마찰이 없다.
- 비밀번호를 저장하지 않으니 자격증명 유출 위험이 없다.
- 로컬 로그인까지 구현하면 비밀번호 해싱, 재설정 이메일, 계정 연동 로직이 추가되어 혼자 하기엔 복잡도가 너무 커진다.
- RT를 Redis에 두는 이유: `is_active=false` 처리(계정 정지) 시 RT를 즉시 무효화해야 하므로 순수 Stateless JWT로는 불가능하다.

**기각한 대안:**
- **로컬 로그인 병행** — 네이버 계정 없는 사용자를 위한 배려지만, 서비스 특성상 해당 사용자 비율이 낮고 구현 비용이 크다.
- **JWT Stateless (RT 없이)** — Redis 불필요하지만 계정 정지 즉시 반영이 불가능하다.
