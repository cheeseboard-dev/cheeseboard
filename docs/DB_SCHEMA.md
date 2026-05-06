# DB 스키마

**프로젝트:** 치즈보드 (CheeseBoard)
**작성일:** 2026-04-22
**최종 수정:** 2026-05-07
**상태:** Living Document — init.sql과 항상 동기화 유지

> 아키텍처 결정 근거: [ADR-0001 UUID v7](adr/0001-uuid-v7-primary-keys.md) · [ADR-0002 User Tags 분리](adr/0002-split-user-tags-by-content-type.md) · [ADR-0003 Naver OAuth](adr/0003-naver-oauth-only.md)
**DB:** PostgreSQL

---

## 테이블 구성

### streamers (스트리머)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| channel_id | VARCHAR(32) PK | CHZZK 채널 UUID |
| channel_name | VARCHAR(100) | 채널명 |
| profile_image_url | TEXT | 프로필 이미지 URL |
| follower_count | INTEGER | 팔로워 수 |
| updated_at | TIMESTAMP | 마지막 수집 시각 |

### videos (VOD)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| video_no | BIGINT PK | CHZZK 영상 번호 |
| channel_id | VARCHAR(32) FK | streamers.channel_id 참조 |
| title | TEXT | 영상 제목 |
| category | VARCHAR(100) | 카테고리 |
| tags | TEXT[] | 태그 목록 |
| read_count | INTEGER | 조회수 |
| duration | INTEGER | 영상 길이 (초) |
| published_at | TIMESTAMP | 업로드 시각 |
| thumbnail_url | TEXT | 썸네일 URL |

### clips (클립)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| clip_uid | VARCHAR(100) PK | CHZZK 클립 UID |
| channel_id | VARCHAR(32) FK | streamers.channel_id 참조 |
| title | TEXT | 클립 제목 |
| read_count | INTEGER | 조회수 |
| duration | INTEGER | 클립 길이 (초) |
| created_at | TIMESTAMP | 생성 시각 |
| thumbnail_url | TEXT | 썸네일 URL |
| origin_video_no | BIGINT (nullable) | 원본 VOD 참조 (없을 수 있음) |

### users (사용자)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| naver_id | VARCHAR(100) PK | 네이버 고유 사용자 ID |
| nickname | VARCHAR(100) | 네이버 닉네임 |
| profile_image_url | TEXT | 프로필 이미지 URL |
| created_at | TIMESTAMP | 최초 가입 시각 |
| last_login_at | TIMESTAMP | 마지막 로그인 시각 |

### user_tags (사용자 기여 태그)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | BIGSERIAL PK | 자동 증가 ID |
| content_type | VARCHAR(5) | `'vod'` 또는 `'clip'` |
| content_id | VARCHAR(100) | video_no 또는 clip_uid |
| tag | TEXT | 태그 텍스트 |
| naver_id | VARCHAR(100) FK | users.naver_id 참조 |
| created_at | TIMESTAMP | 태그 추가 시각 |

**제약:** `(content_type, content_id, tag, naver_id)` UNIQUE — 동일 사용자가 같은 영상에 같은 태그 중복 추가 불가
