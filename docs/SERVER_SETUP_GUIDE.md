# 서버 설정 가이드 — 백서버 & 크롤러

> 이 문서는 cheeseboard-back(Spring Boot)과 cheeseboard-crawler(FastAPI) 두 서버가  
> 어떻게 구성되어 있고, 왜 그렇게 만들어졌는지를 코드 기준으로 정리한다.  
> 서버 개발이 처음인 사람도 흐름을 잡을 수 있도록 배경 설명을 충분히 포함했다.

---

## 목차

1. [전체 구조 한눈에 보기](#1-전체-구조-한눈에-보기)
2. [인프라 — PostgreSQL & Redis (cheeseboard-infra)](#2-인프라--postgresql--redis-cheeseboard-infra)
3. [크롤러 서버 — FastAPI (cheeseboard-crawler)](#3-크롤러-서버--fastapi-cheeseboard-crawler)
4. [백서버 — Spring Boot (cheeseboard-back)](#4-백서버--spring-boot-cheeseboard-back)
5. [두 서버가 서로 어떻게 연결되는가](#5-두-서버가-서로-어떻게-연결되는가)
6. [로컬에서 실행하는 방법](#6-로컬에서-실행하는-방법)

---

## 1. 전체 구조 한눈에 보기

```
┌─────────────────────────────────────────────────────────────┐
│                         외부 세계                            │
│           치지직(CHZZK) API  ←  크롤러가 주기적으로 호출     │
└───────────────────────┬─────────────────────────────────────┘
                        │ VOD·클립 메타데이터 수집
                        ▼
┌───────────────────────────────────┐
│   cheeseboard-crawler (FastAPI)   │  포트 8000
│   - CHZZK API 호출                │
│   - 수집한 데이터를 DB에 저장      │
└───────────────────┬───────────────┘
                    │ INSERT / UPSERT
                    ▼
┌───────────────────────────────────┐
│   cheeseboard-infra               │
│   - PostgreSQL 18  (포트 5432)    │  ← 원본 데이터 저장소
│   - Redis 7        (포트 6379)    │  ← 캐시 / 토큰 저장소
└───────────────────┬───────────────┘
                    │ SELECT
                    ▼
┌───────────────────────────────────┐
│   cheeseboard-back (Spring Boot)  │  포트 8080
│   - REST API 제공                 │
│   - 검색·정렬·필터 로직            │
└───────────────────┬───────────────┘
                    │ JSON 응답
                    ▼
           React 프론트엔드 (예정)
```

**핵심 개념:** 크롤러는 "데이터를 가져오는 역할", 백서버는 "데이터를 보여주는 역할"이다.  
두 서버는 PostgreSQL이라는 공통 데이터베이스를 통해서만 간접적으로 연결된다.  
서로 직접 HTTP 통신하지 않는다.

---

## 2. 인프라 — PostgreSQL & Redis (cheeseboard-infra)

### 왜 Docker를 쓰는가

PostgreSQL이나 Redis 같은 데이터베이스를 PC에 직접 설치하면 버전 충돌, OS별 설정 차이 등 문제가 생긴다.  
Docker를 쓰면 "이 DB는 이 버전으로, 이 설정으로" 라는 것을 파일 하나(`docker-compose.yml`)에 적어 두고  
어느 PC에서든 동일하게 실행할 수 있다.

### `docker-compose.yml` 구조

```
cheeseboard-infra/
├── docker-compose.yml   ← 컨테이너 정의
├── init.sql             ← DB 최초 실행 시 자동으로 실행되는 스키마
├── .env                 ← 비밀번호 등 민감 정보 (git에 올리지 않음)
└── .env.example         ← .env 예시 파일 (git에 올림)
```

`docker-compose.yml`에서 정의하는 컨테이너는 두 개다.

| 컨테이너       | 이미지            | 포트   | 역할                       |
| ---------- | -------------- | ---- | ------------------------ |
| `postgres` | postgres:18    | 5432 | VOD·클립 원본 데이터 저장         |
| `redis`    | redis:7-alpine | 6379 | 캐시, 나중에 Refresh Token 저장 |

### `init.sql` — DB 테이블이 어떻게 만들어지는가

PostgreSQL 컨테이너가 처음 시작될 때 `/docker-entrypoint-initdb.d/` 안에 있는 SQL 파일을 자동으로 실행한다.  
`init.sql`을 그 경로에 마운트해 두었기 때문에 컨테이너 최초 실행 시 테이블이 자동으로 생성된다.

**만들어지는 테이블:**

| 테이블               | 설명                                |
| ----------------- | --------------------------------- |
| `streamers`       | 스트리머 목록 (channel_id, 이름, 팔로워 수 등) |
| `videos`          | VOD 메타데이터 (제목, 태그, 조회수, 썸네일 등)    |
| `clips`           | 클립 메타데이터                          |
| `crawl_jobs`      | 크롤 작업 이력 (언제 시작해서 성공/실패했는지)       |
| `users`           | Phase 2 — 네이버 로그인 사용자             |
| `video_user_tags` | Phase 2 — 사용자가 VOD에 붙인 태그         |
| `clip_user_tags`  | Phase 2 — 사용자가 클립에 붙인 태그          |

**PK(기본키) 전략:**  
`streamers`의 PK는 `channel_id`(치지직 고유 ID, VARCHAR)이고,  
`videos`의 PK는 `video_no`(치지직 VOD 번호, BIGINT)이다.  
`crawl_jobs`, `*_user_tags` 같이 치지직 고유 ID가 없는 테이블은 `UUID v7`을 사용한다.  
UUID v7은 시간 순서대로 정렬되기 때문에 B-Tree 인덱스 성능이 일반 UUID보다 좋다. (→ `DECISIONS.md` ADR-001 참고)

---

## 3. 크롤러 서버 — FastAPI (cheeseboard-crawler)

### 왜 FastAPI인가

- Python 생태계는 HTTP 요청 라이브러리(`httpx`)가 강력하고, 비동기(`async/await`) 처리가 쉽다.
- FastAPI는 코드를 작성하면 Swagger 문서가 자동으로 생성되어 API를 바로 테스트할 수 있다.
- 크롤러는 DB에 대량 쓰기 작업을 하고 외부 API를 반복 호출하는 I/O 중심 작업이므로 비동기 방식이 적합하다.

### 디렉토리 구조

```
cheeseboard-crawler/
├── app/
│   ├── main.py               ← FastAPI 앱 진입점
│   ├── config.py             ← 환경 변수 관리
│   ├── exceptions.py         ← 커스텀 예외 정의
│   ├── exception_handlers.py ← 예외를 HTTP 응답으로 변환
│   ├── models/               ← 데이터 형태 정의 (Pydantic)
│   │   ├── channel.py        → ChannelInfo
│   │   ├── video.py          → Video
│   │   └── clip.py           → Clip
│   ├── routers/              ← API 엔드포인트 모음
│   │   ├── channels.py       → /api/v1/channels/*
│   │   └── crawl.py          → /api/v1/crawl/*
│   └── services/             ← 비즈니스 로직
│       ├── chzzk_client.py   → 치지직 API 호출 담당
│       └── crawler.py        → 크롤 작업 생성·관리
├── data/
│   ├── streamers.csv         ← 수집 대상 스트리머 100+명 목록
│   └── target_names.txt      ← 스트리머 이름 목록
├── pyproject.toml            ← 프로젝트 메타·의존성 정의
└── requirements.txt          ← pip install용 의존성 목록
```

### `app/main.py` — 앱의 시작과 끝

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await chzzk_client.start()   # 앱 시작 시: HTTP 클라이언트 초기화
    yield
    await chzzk_client.stop()    # 앱 종료 시: 연결 정리

app = FastAPI(
    title="CheeseBoard Crawler",
    lifespan=lifespan,
)
```

`lifespan`은 앱이 켜질 때와 꺼질 때 실행할 코드를 정의한다.  
HTTP 클라이언트(`httpx.AsyncClient`)를 앱 시작 시 한 번만 만들어 재사용하는 것이 핵심이다.  
요청마다 새로 만들면 느리고 자원 낭비가 심하다.

### `app/config.py` — 환경 변수 관리

```python
class Settings(BaseSettings):
    chzzk_base_url: str = "https://api.chzzk.naver.com/service/v1"
    max_concurrent_requests: int = 3   # 동시 요청 제한
    retry_count: int = 3               # 실패 시 재시도 횟수
    request_timeout: float = 10.0      # 요청 타임아웃 (초)
```

`BaseSettings`를 쓰면 `.env` 파일의 값을 자동으로 읽어서 채워준다.  
코드에 비밀번호나 URL을 직접 쓰지 않아도 되므로 git에 올려도 안전하다.

### `app/models/` — 데이터 형태 정의 (Pydantic)

Pydantic 모델은 두 가지 역할을 한다.

1. 치지직 API 응답에서 필요한 필드만 골라서 Python 객체로 변환한다.
2. FastAPI가 이 모델을 보고 자동으로 API 문서(Swagger)를 생성한다.

예시 — `app/models/video.py`:

```python
class Video(BaseModel):
    video_no: int          # VOD 번호 (PK)
    video_id: str          # VOD UUID
    title: str             # 제목
    tags: List[str] = []   # 태그 (없으면 빈 리스트)
    read_count: int = 0    # 조회수 (없으면 0)
    duration: int = 0      # 길이(초)
    published_at: Optional[str] = None  # 없을 수도 있으니 Optional
```

`Optional`과 `= 0` 같은 기본값을 꼼꼼히 지정하는 이유:  
치지직 API는 공식 문서가 없는 비공식 API라 응답에서 특정 필드가 빠지거나 null로 오는 경우가 있다.  
기본값이 없으면 파싱 도중 에러가 나서 크롤이 통째로 실패한다.

### `app/services/chzzk_client.py` — 치지직 API 호출의 핵심

이 파일이 크롤러의 심장부다. 치지직 API에 실제로 HTTP 요청을 보내고 응답을 파싱한다.

**동시 요청 제한 (세마포어)**

```python
self._semaphore = asyncio.Semaphore(settings.max_concurrent_requests)

async with self._semaphore:
    response = await self._client.get(url)
```

세마포어는 "동시에 최대 3개의 요청만 허용한다"는 잠금 장치다.  
이게 없으면 수백 개의 스트리머를 동시에 요청하다가 치지직 서버가 IP를 차단할 수 있다.

**랜덤 딜레이**

```python
await asyncio.sleep(random.uniform(0.5, 1.5))
```

요청 사이에 0.5~1.5초의 랜덤한 대기 시간을 넣는다.  
"로봇처럼 정확히 1초마다 요청한다"는 패턴보다 "사람처럼 불규칙하게 요청한다"는 패턴이  
봇 탐지 시스템을 덜 자극한다.

**재시도 로직 (지수 백오프)**

```python
for attempt in range(settings.retry_count):
    try:
        ...
    except Exception:
        wait = 2 ** attempt  # 1초, 2초, 4초 순서로 대기
        await asyncio.sleep(wait)
```

요청이 실패하면 바로 재시도하지 않고, 기다리는 시간을 지수적으로 늘린다.  
서버가 일시적으로 바쁜 경우 연달아 요청하면 오히려 더 오래 기다려야 한다.

### `app/services/crawler.py` — 크롤 작업 관리

```python
@dataclass
class CrawlJob:
    job_id: str       # 작업 고유 ID (UUID 앞 8자리)
    status: str       # "running" 또는 "done"
    total: int        # 처리할 스트리머 수
    processed: int    # 완료된 스트리머 수
    failed: int       # 실패한 스트리머 수
    started_at: ...
    finished_at: ...
```

크롤은 시간이 오래 걸리기 때문에 "백그라운드 작업"으로 실행된다.  
API 요청을 받으면 즉시 `job_id`를 반환하고, 크롤은 뒤에서 계속 돌아간다.  
사용자는 `job_id`로 진행 상태를 나중에 조회할 수 있다.

### `app/routers/` — API 엔드포인트

**channels.py** — 단순 조회 (치지직 API를 중계)

```
GET /api/v1/channels/{channel_id}           → 스트리머 정보
GET /api/v1/channels/{channel_id}/videos    → VOD 목록
GET /api/v1/channels/{channel_id}/clips     → 클립 목록
```

이 엔드포인트들은 치지직 API를 그대로 중계한다. 결과를 DB에 저장하지 않는다.  
개발 중에 "이 스트리머 데이터가 제대로 오는지" 확인하는 용도로 사용한다.

**crawl.py** — 실제 크롤 및 저장

```
POST /api/v1/crawl/channel/{channel_id}     → 단일 스트리머 크롤 + DB 저장
POST /api/v1/crawl/bulk                     → 여러 스트리머 백그라운드 크롤
GET  /api/v1/crawl/jobs/{job_id}            → 크롤 작업 진행 상태 조회
```

### `app/exceptions.py` & `app/exception_handlers.py` — 에러 처리

```
CheeseBoardException (기반 예외)
├── ChannelNotFoundException     → HTTP 404
├── CrawlJobNotFoundException    → HTTP 404
├── ChzzkAPIException            → HTTP 502 (외부 API 실패)
└── InvalidRequestException      → HTTP 400
```

예외를 계층 구조로 정의하면 에러 종류에 따라 적절한 HTTP 상태 코드를 자동으로 반환할 수 있다.  
`exception_handlers.py`가 예외를 잡아서 JSON 형태의 HTTP 응답으로 변환한다.

### 탐색 스크립트들 (루트 레벨 `.py` 파일들)

이 파일들은 FastAPI 앱의 일부가 아니라, 개발 과정에서 필요에 따라 직접 실행하는 스크립트다.

| 파일                          | 역할                                                  |
| --------------------------- | --------------------------------------------------- |
| `api_probe.py`              | 치지직 API 응답 구조 탐색 (필드명, 페이지네이션 방식 확인)                |
| `chzzk_modular_scraper.py`  | StreamerManager·VODManager·ClipManager로 분리된 초기 스크래퍼 |
| `chzzk_pro_scraper.py`      | 최근 N일 데이터만 수집하는 증분 스크래퍼                             |
| `get_streamer_uuid_bulk.py` | Selenium으로 치지직 검색 → 스트리머 channel_id 수집 후 CSV 저장     |

`get_streamer_uuid_bulk.py`는 `data/target_names.txt`에 있는 스트리머 이름들을 치지직에서 검색해서  
각각의 `channel_id`(UUID 형태)를 가져와 `data/streamers.csv`에 저장한다.  
이렇게 만들어진 CSV가 벌크 크롤의 대상 목록이 된다.

---

## 4. 백서버 — Spring Boot (cheeseboard-back)

### 왜 Spring Boot인가

- Java 생태계는 Elasticsearch 연동, 캐싱(Redis), DB 연결 등의 라이브러리가 성숙하다.
- 대규모 REST API 서버를 만들 때 구조를 잡아주는 틀(프레임워크)이 잘 갖춰져 있다.
- springdoc-openapi를 쓰면 Swagger UI가 자동 생성된다.

### 디렉토리 구조

```
cheeseboard-back/
├── build.gradle                    ← 빌드 설정 및 의존성 목록
├── settings.gradle                 ← 프로젝트 이름 설정
├── gradlew / gradlew.bat           ← Gradle 래퍼 (Java 빌드 도구)
└── src/
    ├── main/
    │   ├── java/com/cheeseboard/
    │   │   └── CheeseBoardApplication.java   ← 메인 클래스 (서버 진입점)
    │   └── resources/
    │       └── application.yml              ← 서버 설정 파일
    └── test/
        ├── java/com/cheeseboard/
        │   └── CheeseBoardApplicationTests.java
        └── resources/
            └── application.yml
```

현재는 스켈레톤(뼈대) 상태다. 메인 클래스와 기본 설정만 있고 실제 API 코드는 아직 작성 전이다.

### `build.gradle` — 의존성 관리

Maven의 `pom.xml`과 같은 역할로, "이 프로젝트에 어떤 라이브러리가 필요한가"를 정의한다.

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // REST API 서버 기능 전반

    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8'
    // Swagger UI 자동 생성

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    // getter/setter 같은 반복 코드를 어노테이션 하나로 자동 생성
}
```

추후 추가될 의존성 (아직 추가 안 됨):

- `spring-boot-starter-data-jpa` — PostgreSQL ORM
- `spring-boot-starter-data-elasticsearch` — Elasticsearch 연동
- `spring-boot-starter-data-redis` — Redis 캐시
- `spring-boot-starter-security` + OAuth2 — 네이버 로그인 (Phase 2)

### `src/main/resources/application.yml` — 서버 설정

```yaml
spring:
  application:
    name: cheeseboard

server:
  port: 8080               # 서버가 실행될 포트 번호

springdoc:
  api-docs:
    path: /v3/api-docs     # OpenAPI 스펙 JSON 경로
  swagger-ui:
    path: /swagger-ui.html # Swagger UI 경로
    tags-sorter: alpha      # 태그 알파벳 순 정렬
    operations-sorter: alpha
```

### 계획된 패키지 구조 (DDD + Package by Feature)

아직 코드가 없지만, `CLAUDE.md`에 어떻게 만들어야 하는지 정의되어 있다.

```
com.cheeseboard/
├── clip/                     ← 클립 도메인
│   ├── domain/
│   │   ├── Clip.java              (JPA 엔티티 — DB 테이블 매핑)
│   │   └── ClipRepository.java    (DB 접근 인터페이스)
│   ├── application/
│   │   └── ClipService.java       (비즈니스 로직)
│   ├── infrastructure/
│   │   └── ClipRepositoryImpl.java (실제 DB 쿼리 구현)
│   └── presentation/
│       ├── ClipController.java    (REST API 엔드포인트)
│       └── dto/                   (요청/응답 데이터 형태)
├── video/                    ← VOD 도메인 (같은 구조)
├── streamer/                 ← 스트리머 도메인 (같은 구조)
├── user/                     ← 사용자 도메인 — Phase 2
└── common/
    ├── exception/            (전역 에러 핸들러)
    └── config/               (Security, Redis 설정)
```

이 구조의 핵심 원칙: **의존 방향은 항상 안쪽으로**

```
presentation → application → domain ← infrastructure
(Controller)   (Service)    (Entity)   (Repository 구현)
```

`ClipController`는 `ClipService`를 알고, `ClipService`는 `ClipRepository` 인터페이스를 안다.  
하지만 `domain`은 다른 계층을 알지 못한다. 이렇게 하면 DB를 교체하거나 API 형태를 바꿀 때  
다른 계층에 영향을 최소화할 수 있다.

---

## 5. 두 서버가 서로 어떻게 연결되는가

두 서버는 **직접 통신하지 않는다.** PostgreSQL을 통해 간접적으로 연결된다.

```
크롤러 (FastAPI)                    백서버 (Spring Boot)
     │                                      │
     │  INSERT/UPSERT                SELECT │
     └──────────────→ PostgreSQL ←──────────┘
```

**크롤러가 하는 일:**

1. 치지직 API에서 VOD·클립 메타데이터를 가져온다.
2. PostgreSQL의 `videos`, `clips`, `streamers` 테이블에 저장(UPSERT)한다.
3. 이미 있는 데이터면 업데이트, 없으면 삽입한다.

**백서버가 하는 일:**

1. PostgreSQL에서 데이터를 읽어서 React 프론트에 JSON으로 제공한다.
2. Elasticsearch를 통해 빠른 검색 기능을 제공한다. (추후 구현)
3. Redis를 통해 자주 쓰는 응답을 캐싱한다. (추후 구현)

**왜 이렇게 분리했는가:**

- 크롤러가 느리거나 장애가 생겨도 백서버는 계속 서비스할 수 있다.
- 각 서버를 독립적으로 배포·스케일링할 수 있다.
- 역할이 명확하게 분리되어 코드를 관리하기 쉽다.

---

## 6. 로컬에서 실행하는 방법

### 단계 1 — 인프라 시작 (PostgreSQL + Redis)

```bash
cd cheeseboard-infra
cp .env.example .env
# .env 파일을 열어 POSTGRES_PASSWORD를 원하는 비밀번호로 변경

docker compose up -d
docker compose ps   # postgres와 redis가 healthy 상태인지 확인
```

### 단계 2 — 크롤러 실행

```bash
cd cheeseboard-crawler

# 가상환경 생성 및 의존성 설치
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# .env 파일 생성
cp .env.example .env
# DATABASE_URL을 단계 1에서 설정한 비밀번호에 맞게 수정

# 서버 실행
uvicorn app.main:app --reload --port 8000
```

실행 후 http://localhost:8000/docs 에 접속하면 Swagger UI에서 API를 바로 테스트할 수 있다.

### 단계 3 — 백서버 실행

```bash
cd cheeseboard-back

# Windows
./gradlew.bat bootRun

# Mac/Linux
./gradlew bootRun
```

실행 후 http://localhost:8080/swagger-ui.html 에서 API 문서를 확인할 수 있다.

---

## 참고 문서

| 문서                                   | 내용                         |
| ------------------------------------ | -------------------------- |
| [SCRAPER_GUIDE.md](SCRAPER_GUIDE.md) | 크롤러 수집 전략, 페이지네이션, 스케줄 주기  |
| [DB_SCHEMA.md](DB_SCHEMA.md)         | 테이블 스키마 상세 DDL             |
| [INFRA.md](INFRA.md)                 | Docker 설정, OCI 운영 환경 구성    |
| [DECISIONS.md](DECISIONS.md)         | UUID v7 선택 이유 등 아키텍처 결정 기록 |
| [API_SPEC.md](API_SPEC.md)           | 백서버 REST API 엔드포인트 명세      |
