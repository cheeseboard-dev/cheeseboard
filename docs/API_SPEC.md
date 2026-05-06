# API 명세 (임시 초안)

**프로젝트:** 치즈보드 (CheeseBoard)
**작성일:** 2026-04-22
**상태:** 임시 초안 — 추후 상세 설계 예정
**Base URL:** `/api`

---

## 공통

### 응답 형식

```json
{
  "data": {},
  "error": null
}
```

### 에러 응답

```json
{
  "data": null,
  "error": {
    "code": "NOT_FOUND",
    "message": "해당 리소스를 찾을 수 없습니다."
  }
}
```

---

## 엔드포인트

### GET /api/clips/popular

인기 클립 피드 (메인 페이지)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| size | integer | N | 20 | 한 번에 가져올 개수 |
| cursor | string | N | - | 무한스크롤 커서 (이전 응답의 nextCursor) |

**Response**

```json
{
  "data": {
    "items": [
      {
        "clipUid": "abc123",
        "title": "클립 제목",
        "channelId": "0b33823ac81de48d5b78a38cdbc0ab94",
        "channelName": "울프",
        "profileImageUrl": "https://...",
        "readCount": 12345,
        "duration": 60,
        "createdAt": "2026-04-15T10:00:00",
        "thumbnailUrl": "https://..."
      }
    ],
    "nextCursor": "eyJpZCI6MTIzfQ==",
    "hasMore": true
  }
}
```

---

### GET /api/search

키워드 검색 (VOD + 클립 통합)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| q | string | Y | - | 검색 키워드 |
| type | string | N | all | 콘텐츠 타입: `all` \| `vod` \| `clip` |
| sort | string | N | popular | 정렬: `popular` \| `latest` |
| channelId | string | N | - | 스트리머 필터 |
| size | integer | N | 20 | 한 번에 가져올 개수 |
| cursor | string | N | - | 무한스크롤 커서 |

**Response**

```json
{
  "data": {
    "items": [
      {
        "type": "clip",
        "clipUid": "abc123",
        "title": "클립 제목",
        "channelId": "0b33823ac81de48d5b78a38cdbc0ab94",
        "channelName": "울프",
        "profileImageUrl": "https://...",
        "readCount": 12345,
        "duration": 60,
        "createdAt": "2026-04-15T10:00:00",
        "thumbnailUrl": "https://..."
      },
      {
        "type": "vod",
        "videoNo": 12744988,
        "title": "VOD 제목",
        "channelId": "0b33823ac81de48d5b78a38cdbc0ab94",
        "channelName": "울프",
        "profileImageUrl": "https://...",
        "readCount": 3651,
        "duration": 20676,
        "publishedAt": "2026-04-15T22:27:40",
        "thumbnailUrl": "https://..."
      }
    ],
    "nextCursor": "eyJpZCI6MTIzfQ==",
    "hasMore": true
  }
}
```

---

### GET /api/search/suggest

검색창 스트리머명 자동완성

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| q | string | Y | - | 입력 중인 검색어 (최소 1자) |
| size | integer | N | 5 | 반환할 최대 후보 수 |

**구현 방식:** ES `prefix query` on `channelName.keyword`

**Response**

```json
{
  "data": {
    "suggestions": [
      {
        "channelId": "0b33823ac81de48d5b78a38cdbc0ab94",
        "channelName": "울프",
        "profileImageUrl": "https://..."
      },
      {
        "channelId": "...",
        "channelName": "울챔스",
        "profileImageUrl": "https://..."
      }
    ]
  }
}
```

> **프론트 처리:** 입력 후 300ms debounce 적용. 2자 미만 입력 시 요청 안 함.

---

### GET /api/streamers

스트리머 목록 (필터 드롭다운용)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| q | string | N | - | 스트리머명 검색 |
| size | integer | N | 50 | 가져올 개수 |

**Response**

```json
{
  "data": {
    "items": [
      {
        "channelId": "0b33823ac81de48d5b78a38cdbc0ab94",
        "channelName": "울프",
        "profileImageUrl": "https://..."
      }
    ]
  }
}
```

---

### GET /api/auth/naver

네이버 OAuth 로그인 시작. 네이버 인증 페이지로 리다이렉트.

---

### GET /api/auth/naver/callback

네이버 OAuth 콜백. 인증 완료 후 JWT를 HttpOnly 쿠키로 발급하고 원래 페이지로 리다이렉트.

---

### GET /api/auth/me

현재 로그인 사용자 정보 조회 (로그인 상태 확인용)

**Response**

```json
{
  "data": {
    "naverId": "abc123",
    "nickname": "울프팬",
    "profileImageUrl": "https://..."
  }
}
```

> 비로그인 시 `401 Unauthorized`

---

### DELETE /api/auth/logout

로그아웃. JWT 쿠키 삭제.

---

### POST /api/videos/{videoNo}/tags

VOD에 사용자 태그 추가 (로그인 필요)

**Path Parameters:** `videoNo` — 대상 VOD 번호

**Request Body**

```json
{ "tag": "롤챔스" }
```

**Response**

```json
{
  "data": { "tag": "롤챔스", "contentType": "vod", "contentId": "12744988" }
}
```

> 중복 태그 시 `409 Conflict` / 태그 개수 초과(20개) 시 `400 Bad Request`

---

### DELETE /api/videos/{videoNo}/tags/{tag}

VOD 사용자 태그 삭제 (본인이 추가한 태그만 삭제 가능)

---

### POST /api/clips/{clipUid}/tags

클립에 사용자 태그 추가 (로그인 필요). 요청/응답 형식은 VOD와 동일.

---

### DELETE /api/clips/{clipUid}/tags/{tag}

클립 사용자 태그 삭제 (본인이 추가한 태그만 삭제 가능)
