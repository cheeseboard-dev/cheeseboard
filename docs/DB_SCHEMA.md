# DB 스키마

**프로젝트:** 치즈보드 (CheeseBoard)
**작성일:** 2026-04-22
**최종 수정:** 2026-05-07
**상태:** Living Document — `cheeseboard-infra/init.sql`과 항상 동기화 유지

> 아키텍처 결정 근거: [DECISIONS.md](DECISIONS.md)

---

## 테이블 구성

### streamers (스트리머)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| channel_id | VARCHAR(32) PK | CHZZK 채널 UUID |
| channel_name | VARCHAR(100) NOT NULL | 채널명 |
| profile_image_url | TEXT | 프로필 이미지 URL |
| follower_count | INTEGER | 팔로워 수 |
| updated_at | TIMESTAMP NOT NULL DEFAULT NOW() | 마지막 수집 시각 |
| last_crawled_at | TIMESTAMP | 마지막 크롤 시각 |
| last_refreshed_at | TIMESTAMP | 마지막 갱신 시각 |
| is_active | BOOLEAN NOT NULL DEFAULT TRUE | FALSE: 수집 대상에서 제외 |

---

### videos (VOD)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| video_no | BIGINT PK | CHZZK 영상 번호 |
| video_id | VARCHAR(40) NOT NULL UNIQUE | CHZZK UUID (클립 연결용) |
| channel_id | VARCHAR(32) FK → streamers | 채널 |
| title | TEXT NOT NULL | 영상 제목 |
| category | VARCHAR(100) | 카테고리 |
| tags | TEXT[] | 태그 목록 |
| read_count | INTEGER NOT NULL DEFAULT 0 | 조회수 |
| duration | INTEGER | 영상 길이 (초) |
| published_at | TIMESTAMP | 업로드 시각 |
| thumbnail_url | TEXT | 썸네일 URL |
| last_refreshed_at | TIMESTAMP | 마지막 갱신 시각 |

---

### clips (클립)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| clip_uid | VARCHAR(20) PK | CHZZK 클립 UID |
| channel_id | VARCHAR(32) FK → streamers | 채널 |
| origin_video_id | VARCHAR(40) FK → videos.video_id (nullable) | 원본 VOD |
| title | TEXT NOT NULL | 클립 제목 |
| read_count | INTEGER NOT NULL DEFAULT 0 | 조회수 |
| duration | INTEGER | 클립 길이 (초) |
| created_at | TIMESTAMP | 생성 시각 |
| thumbnail_url | TEXT | 썸네일 URL |
| last_refreshed_at | TIMESTAMP | 마지막 갱신 시각 |

---

### users (사용자) — Phase 2

네이버 OAuth 전용. 로컬 로그인 없음. ([DECISIONS.md #003](DECISIONS.md) 참고)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| naver_id | VARCHAR(100) PK | 네이버 고유 식별자 |
| nickname | VARCHAR(100) NOT NULL | 닉네임 |
| profile_image_url | TEXT | 프로필 이미지 URL |
| email | VARCHAR(200) | 네이버 OAuth 제공 시에만 저장 |
| is_active | BOOLEAN NOT NULL DEFAULT TRUE | FALSE: 이용 정지 |
| created_at | TIMESTAMP NOT NULL DEFAULT NOW() | 최초 가입 시각 |
| last_login_at | TIMESTAMP | 마지막 로그인 시각 |

---

### video_user_tags (VOD 사용자 기여 태그) — Phase 2

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK DEFAULT uuidv7() | — |
| video_no | BIGINT NOT NULL FK → videos ON DELETE CASCADE | 대상 VOD |
| tag | TEXT NOT NULL | 태그 텍스트 |
| naver_id | VARCHAR(100) NOT NULL FK → users | 기여한 사용자 |
| created_at | TIMESTAMP NOT NULL DEFAULT NOW() | 태그 추가 시각 |

**제약:** `UNIQUE (video_no, tag, naver_id)`

---

### clip_user_tags (클립 사용자 기여 태그) — Phase 2

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK DEFAULT uuidv7() | — |
| clip_uid | VARCHAR(20) NOT NULL FK → clips ON DELETE CASCADE | 대상 클립 |
| tag | TEXT NOT NULL | 태그 텍스트 |
| naver_id | VARCHAR(100) NOT NULL FK → users | 기여한 사용자 |
| created_at | TIMESTAMP NOT NULL DEFAULT NOW() | 태그 추가 시각 |

**제약:** `UNIQUE (clip_uid, tag, naver_id)`

---

### crawl_jobs (크롤 작업 이력)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK DEFAULT uuidv7() | — |
| job_type | VARCHAR(20) NOT NULL | `initial` / `incremental` / `full` / `user_triggered` |
| started_at | TIMESTAMP NOT NULL DEFAULT NOW() | 시작 시각 |
| finished_at | TIMESTAMP | 종료 시각 |
| status | VARCHAR(10) NOT NULL DEFAULT 'running' | `running` / `done` / `failed` |
| total_streamers | INTEGER | 대상 스트리머 수 |
| success_count | INTEGER NOT NULL DEFAULT 0 | 성공 수 |
| failed_count | INTEGER NOT NULL DEFAULT 0 | 실패 수 |
| triggered_by | VARCHAR(20) | `scheduler` / `user` / `admin` |
| error_msg | TEXT | 실패 시 오류 메시지 |

---

## 인덱스

| 인덱스 | 테이블 | 컬럼 |
|---|---|---|
| idx_videos_channel | videos | channel_id |
| idx_videos_published_at | videos | published_at DESC |
| idx_videos_read_count | videos | read_count DESC |
| idx_clips_channel | clips | channel_id |
| idx_clips_created_at | clips | created_at DESC |
| idx_clips_read_count | clips | read_count DESC |
| idx_clips_origin_video | clips | origin_video_id |
| idx_video_user_tags_video | video_user_tags | video_no |
| idx_video_user_tags_user | video_user_tags | naver_id |
| idx_clip_user_tags_clip | clip_user_tags | clip_uid |
| idx_clip_user_tags_user | clip_user_tags | naver_id |
| idx_crawl_jobs_status | crawl_jobs | status, started_at DESC |
