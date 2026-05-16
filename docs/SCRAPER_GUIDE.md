# 치즈보드(CheeseBoard) 스크래퍼 가이드

**프로젝트:** 치즈보드 (CheeseBoard)
**최종 수정:** 2026-05-16
**상태:** Living Document — 크롤러 구현 완료 (ES 연동 미구현)

---

## 1. 수집 인프라

| 항목 | 내용 |
|---|---|
| Framework | FastAPI (Python) |
| HTTP Client | `httpx` (비동기) |
| Data Modeling | `Pydantic v2` |
| 스케줄러 | APScheduler (`app/services/scheduler.py`) |

---

## 2. API 사용 방침

치즈보드는 Naver/CHZZK의 **비공식 내부 API**를 사용한다.

```
Base URL: https://api.chzzk.naver.com
```

**비공식 API 사용의 함의:**

| 항목 | 내용 |
|---|---|
| 정렬 순서 보장 없음 | 응답 항목이 항상 최신순으로 온다고 가정할 수 없음 |
| 응답 스키마 변경 가능 | 언제든 필드 추가·삭제·형식 변경 가능 |
| 차단 가능성 | 요청 패턴에 따라 IP 차단 또는 API 폐쇄 가능 |
| 공식 지원 없음 | 오류 발생 시 원인 파악이 어려움 |

이에 따라 스크래퍼 레이어는 **백엔드와 분리**하여, API 변경 시 스크래퍼만 수정하면 되도록 유지한다.

---

## 3. 수집 대상 API

### 3.1 스트리머 채널 정보

```
GET https://api.chzzk.naver.com/service/v1/channels/{channelId}
```

| 수집 필드 | DB 컬럼 |
|---|---|
| channelId | streamers.channel_id |
| channelName | streamers.channel_name |
| profileImageUrl | streamers.profile_image_url |
| followerCount | streamers.follower_count |

### 3.2 VOD (다시보기)

```
GET https://api.chzzk.naver.com/service/v1/channels/{channelId}/videos
```

| 수집 필드 | DB 컬럼 |
|---|---|
| videoNo | videos.video_no |
| videoId | videos.video_id |
| videoTitle | videos.title |
| publishDateAt | videos.published_at |
| readCount | videos.read_count |
| duration | videos.duration |
| tags | videos.tags |
| thumbnailImageUrl | videos.thumbnail_url |

> `videoId`는 클립의 `origin_video_id`와 연결되는 UUID 형태 식별자. `video_no`(숫자 PK)와 별도로 관리된다.

### 3.3 클립

```
GET https://api.chzzk.naver.com/service/v1/channels/{channelId}/clips
```

| 수집 필드 | DB 컬럼 |
|---|---|
| clipUID | clips.clip_uid |
| clipTitle | clips.title |
| createdDate | clips.created_at |
| readCount | clips.read_count |
| duration | clips.duration |
| thumbnailImageUrl | clips.thumbnail_url |

---

## 4. 수집 전략

### 4.1 기본 원칙 — "잡 목적별 분리"

크롤링 잡은 **목적**에 따라 4개로 분리된다. 한 잡이 모든 일을 떠맡지 않는다.

| 카테고리 | 무엇을 책임지는가 |
|---|---|
| **Discovery (발견)** | 새 클립/영상을 빨리 DB에 들이기 |
| **Ranking accuracy (정확성)** | 메인 페이지에 노출되는 인기 콘텐츠의 read_count 신선도 |
| **Reconciliation (정합성)** | 누락·정렬 흔들림 보정, 깊은 페이지까지 채우기 |
| **On-demand (수동)** | 운영자 트리거, 사용자 조회 트리거 (View-triggered Refresh) |

각 잡은 컷오프와 페이지 캡으로 강도를 다르게 조절한다.

- **컷오프**: `since` 시간 컷오프 또는 **DB watermark** (채널별 `MAX(published_at)` / `MAX(created_at)`). 그 이전 항목이 나오면 페이지 순회 중단.
- **`read_count ≥ 100` 필터** (클립만): 노이즈 제거. 한 번도 100을 못 넘긴 클립은 메인 페이지 영역 밖이므로 DB에서 제외.
- **upsert**: 모든 항목은 INSERT ON CONFLICT UPDATE.

### 4.2 수집 주기 (스케줄)

| 잡 id | 카테고리 | 주기 | API / 범위 | 컷오프·필터 |
|---|---|---|---|---|
| `hot_clips_poll` | Ranking | 1시간 | `/home/recommended/clips?filterType=WITHIN_1_DAY&orderType=POPULAR` cursor 3페이지 (~90개) | `read_count ≥ 100` |
| `latest_videos_poll` | Discovery | 1시간 | `/home/videos?sortType=LATEST` cursor 2페이지 (~60개) | 없음 (영상은 생성 빈도 낮아 전체 수용) |
| `channel_clips_incremental` | Discovery | 3시간 | 활성 채널 ~500 × 클립 1페이지 | 채널별 `MAX(created_at)` watermark + `read_count ≥ 100` |
| `weekly_reconciliation` | Reconciliation | 일 03:00 KST | 활성 채널 전체 × VOD 10페이지 / 클립 5페이지 | 페이지 캡 적용 |

> 평균 API 호출: 분당 약 3회 (CHZZK 안전선 분당 7~13 대비 여유). 일요일 03:00 weekly 피크 시만 분당 20회 가까이 솟음.

**채널 단위 영상 incremental은 폐기.** 영상은 `latest_videos_poll`이 글로벌 LATEST 피드로 신규를 잡고, 누적 인기 갱신은 weekly가 책임진다.

### 4.3 자동 채널 발견

`hot_clips_poll`과 `latest_videos_poll` 응답에는 각 항목의 `ownerChannel` / `channel` 정보가 포함된다. DB에 없는 채널이면 **자동으로 `is_active=true` stub upsert** → 다음 incremental에 자동 포함된다. 이미 등록된 채널의 follower_count 등은 덮어쓰지 않는다.

라이브 발견(`/lives`)은 보조 수단으로 남되 정기 스케줄에서 제거됨 (수동 트리거 또는 운영 시점에 필요 시 활성).

### 4.4 View-triggered Refresh — 미구현

> **현재 상태:** 미구현. Spring Boot API 개발 시 호출자(클라이언트) 구현. 크롤러 측 hook은 `POST /videos/{video_no}/refresh`, `POST /clips/{clip_uid}/refresh`로 이미 준비됨.

유저가 콘텐츠를 조회할 때 Spring Boot가 비동기로 크롤러의 refresh 엔드포인트를 호출 → 단건 즉시 갱신. 핫 콘텐츠 정확성을 정기 잡 주기보다 빠르게 보강.

| 항목 | 내용 |
|---|---|
| 갱신 판단 기준 | `updated_at < now - 1시간` (임계값 조정 가능) |
| 실행 방식 | 응답 반환 후 백그라운드 태스크 (Spring `@Async`) |
| 중복 방지 | Redis `SET NX TTL` |
| 실패 처리 | 유저 응답에 영향 없음. 로그만 |

### 4.5 잡 중복 방지

같은 `job_type`의 `status=running` 잡이 이미 있으면:
- API 트리거: `409 CRAWL_JOB_CONFLICT`
- 스케줄러 트리거: warn 로그 + skip (다음 tick까지 대기)

---

## 5. 요청 제어

| 항목 | 값 | 이유 |
|---|---|---|
| 동시 요청 수 | 최대 3 (Semaphore) | 비공식 API 차단 방지 |
| 요청 간 딜레이 | 0.5 ~ 1.5초 (랜덤) | 일정 패턴 회피 |
| 타임아웃 | 10초 | 응답 지연 시 무한 대기 방지 |
| 재시도 | 최대 3회, 지수 백오프 | 일시적 오류 대응 |

---

## 6. 오류 및 스키마 변경 감지

비공식 API는 언제든 응답 구조가 바뀔 수 있다. 파싱 오류는 조용히 무시하지 않고 즉시 감지한다.

| 오류 유형 | 처리 방식 |
|---|---|
| 필수 필드 누락 (Pydantic ValidationError) | 해당 항목 스킵 + 경고 로그 |
| HTTP 4xx (차단, 인증 오류) | 크롤 중단 + 알람 |
| HTTP 5xx (CHZZK 서버 오류) | 재시도 후 실패 시 해당 스트리머 스킵 + 로그 |
| 예상치 못한 응답 구조 | 경고 로그 + Slack/이메일 알림 (미구현, 추후 도입 예정) |

---

## 7. 관리 API

크롤러 서버가 제공하는 운영/관리용 엔드포인트. `Base URL: /api/v1`

### 스트리머 관리

| 메서드 | 경로 | 설명 |
|---|---|---|
| `POST` | `/streamers` | channel_id로 CHZZK 조회 후 DB 등록 |
| `POST` | `/streamers/bulk` | softc JSON 포맷(`uuid`, `name`, `followers`) 배열로 일괄 등록 |
| `GET` | `/streamers?active_only=` | 등록된 스트리머 목록. `active_only=true` 시 활성 채널만 |
| `GET` | `/streamers/{channel_id}/stats` | 영상/클립 개수, 최근 항목 시각, 마지막 크롤 시각 |
| `POST` | `/streamers/{channel_id}/refresh` | 채널 정보(이름·프로필·팔로워)만 즉시 동기 갱신 (영상/클립 제외) |
| `PATCH` | `/streamers/{channel_id}` | `is_active` 토글 (수집 대상 포함/제외) |

### 크롤 작업

| 메서드 | 경로 | 파라미터 | 설명 |
|---|---|---|---|
| `POST` | `/crawl/channel/{id}` | `since`, `mode`, `max_video_pages`, `max_clip_pages` | 단건 채널 동기 크롤 |
| `POST` | `/crawl/bulk` | `scope=full\|videos\|clips\|streamers_only` | 채널 목록 비동기 일괄 크롤. scope로 수집 범위 제어 |
| `POST` | `/crawl/live` | `min_viewers`, `since`, `mode` | 실시간 방송 채널 자동 발견·크롤 |
| `GET` | `/crawl/jobs?limit=` | — | 크롤 이력 목록 (기본 10건, 최대 100) |
| `GET` | `/crawl/jobs/{job_id}` | — | 작업 진행 상태 단건 조회 |
| `POST` | `/crawl/jobs/{job_id}/retry` | — | 해당 잡의 `failed_channels`만 새 잡으로 재크롤 |

### 단건 콘텐츠 갱신

| 메서드 | 경로 | 설명 |
|---|---|---|
| `POST` | `/videos/{video_no}/refresh` | CHZZK에서 단일 영상 정보 재조회 후 upsert |
| `POST` | `/clips/{clip_uid}/refresh` | CHZZK에서 단일 클립 정보 재조회 후 upsert |

Spring Boot의 View-triggered Refresh(SCRAPER_GUIDE 4.4) 구현 시 호출되는 백엔드 훅. 운영 시 단건 디버깅용으로도 사용.

### 고아 잡 처리

앱 기동 시 `status=running` 상태로 남은 잡을 자동으로 `status=failed`로 정리한다.  
서버 재시작 전에 실행 중이던 크롤 작업이 미완료 상태로 영구히 남는 것을 방지한다.

---

## 8. 데이터 흐름

```
CHZZK 비공식 API
      ↓
  FastAPI 스크래퍼
  ├── PostgreSQL       : upsert (원본 데이터) ← 구현 완료
  └── Elasticsearch    : partial update (검색 인덱스) ← 미구현
```

ES 인덱스 업데이트는 항목 단위 부분 업데이트(`POST /{index}/_update`)로 수행 예정.  
전체 재인덱싱은 주 1회 전체 크롤 시에만 수행 예정.
