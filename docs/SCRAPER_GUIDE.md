# 치즈보드(CheeseBoard) 스크래퍼 가이드

**프로젝트:** 치즈보드 (CheeseBoard)
**최종 수정:** 2026-05-07
**상태:** Living Document — API 탐색 완료, 크롤러 구현 진행 전

---

## 1. 수집 인프라

| 항목 | 내용 |
|---|---|
| Framework | FastAPI (Python) |
| HTTP Client | `httpx` (비동기) |
| Data Modeling | `Pydantic v2` |
| 스케줄러 | APScheduler (추후 도입) |

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
| videoTitle | videos.title |
| publishDateAt | videos.published_at |
| readCount | videos.read_count |
| duration | videos.duration |
| tags | videos.tags |
| thumbnailImageUrl | videos.thumbnail_url |

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

### 4.1 기본 원칙 — "항상 N페이지, upsert"

비공식 API 특성상 응답 정렬 순서를 보장할 수 없다. 따라서:

- **워터마크(watermark) 기반 조기 종료 사용 안 함**
  - "이미 본 ID 나오면 중단" 로직은 정렬 순서가 보장될 때만 안전함
  - 비공식 API에서는 신규 항목 누락 위험이 있어 사용하지 않음
- **항상 최근 N페이지를 전부 가져와 upsert**
  - 신규 항목: INSERT
  - 기존 항목: 가변 필드(조회수, 제목 등) UPDATE
  - 페이지 수는 수집 주기와 스트리머 활동량에 따라 조정

```
스트리머별 재크롤 시:
  최근 3페이지(60개) 무조건 수집
  → DB upsert (중복 무시, read_count 등 가변 필드 덮어씀)
  → ES 인덱스 부분 업데이트
```

### 4.2 수집 주기 (스케줄)

| 구분 | 주기 | 범위 | 목적 |
|---|---|---|---|
| 증분 크롤 | 2~4시간 | 전체 스트리머, 최근 3페이지 | 신규 VOD/클립 수집, 조회수 갱신 |
| 전체 크롤 | 주 1회 (새벽) | 전체 스트리머, 전 페이지 | 누락 보정, 오래된 데이터 정합성 확보 |

> 스트리머 400명 기준 증분 크롤 1회 시 API 호출 수: 약 400 × 2(VOD+클립) × 3페이지 = 2,400건  
> 세마포어 3 + 딜레이 0.5~1.5초 적용 시 완료까지 약 20~40분 예상

### 4.3 조회 트리거 갱신 (View-triggered Refresh)

유저가 콘텐츠를 조회할 때 해당 항목을 비동기로 갱신한다.

**목적:**
- 인기 콘텐츠(조회 빈도 높음)의 조회수를 크롤 주기보다 빠르게 최신화
- 정기 크롤 사이에도 핫한 콘텐츠는 지속적으로 갱신

**동작 흐름:**

```
유저 → 클립/VOD 조회 요청
  ↓
Spring Boot: DB에서 데이터 즉시 반환 (응답 지연 없음)
  ↓ (비동기, 응답과 무관)
updated_at < now - 1시간 이면:
  → CHZZK API로 해당 항목 1건 fetch
  → DB read_count 등 갱신
  → ES 인덱스 부분 업데이트
```

**구현 포인트:**

| 항목 | 내용 |
|---|---|
| 갱신 판단 기준 | `updated_at < now - 1시간` (임계값 조정 가능) |
| 실행 방식 | 응답 반환 후 백그라운드 태스크 실행 (Spring @Async) |
| 중복 방지 | Redis `SET NX TTL`로 동일 항목 동시 갱신 방지 (분산 락) |
| 실패 처리 | 갱신 실패해도 유저 응답에 영향 없음. 로그만 기록 |

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
| 예상치 못한 응답 구조 | 경고 로그 + Slack/이메일 알림 (추후 도입) |

---

## 7. 데이터 흐름

```
CHZZK 비공식 API
      ↓
  FastAPI 스크래퍼
  ├── PostgreSQL  : upsert (원본 데이터)
  └── Elasticsearch : partial update (검색 인덱스)
```

ES 인덱스 업데이트는 항목 단위 부분 업데이트(`POST /{index}/_update`)로 수행.  
전체 재인덱싱은 주 1회 전체 크롤 시에만 수행.
