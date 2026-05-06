# 치즈보드 인프라 설계서

**프로젝트:** 치즈보드 (CheeseBoard)  
**작성일:** 2026-04-23  
**최종 수정:** 2026-05-07  
**상태:** Living Document — 로컬 환경 운영 중, OCI 배포 전  
**인프라:** Oracle Cloud Infrastructure (Always Free Tier)

---

## 0. 로컬 개발 환경 (Docker Compose)

운영 환경(OCI)에 올리기 전 로컬에서 개발할 때는 `cheeseboard-infra/` 레포의 Docker Compose를 사용.

**레포:** https://github.com/cheeseboard-dev/cheeseboard-infra

### 실행 방법

```bash
cd cheeseboard-infra
cp .env.example .env      # .env 열어서 비밀번호 설정
docker compose up -d      # PostgreSQL 16 컨테이너 실행
docker compose ps         # State: healthy 확인
```

### 로컬 서비스 포트

| 서비스 | 로컬 포트 | 컨테이너 포트 |
|---|---|---|
| PostgreSQL 18 | 5432 | 5432 |
| Redis 7 | 6379 | 6379 |

> Elasticsearch는 백엔드 개발 단계에 docker-compose.yml에 추가 예정.

---

## 1. 전체 아키텍처

```
인터넷
  │
  ▼
[프론트 서버] ──── nginx (80/443) ──── React 정적 파일
  │ API 요청
  ▼
[앱 서버] ─────── Spring Boot (8080) ── 검색/조회 API
         └─────── FastAPI     (8000) ── 스크래퍼 API
  │ DB/ES 접근
  ▼
[DB 서버] ──────── PostgreSQL (5432) ── 원본 데이터
          └──────── Elasticsearch (9200) ── 검색 인덱스
                           ▲
                   FastAPI가 수집한 데이터를 CHZZK API에서 가져와 여기에 저장
```

**데이터 흐름:**
```
CHZZK API
    │
    ▼ (수집)
FastAPI 스크래퍼 (앱 서버)
    ├──▶ PostgreSQL (DB 서버)  : 원본 메타데이터 보존
    └──▶ Elasticsearch (DB 서버): 검색 인덱스 구축
                  │
                  ▼ (검색 쿼리)
           Spring Boot (앱 서버)
                  │
                  ▼ (JSON API)
          React 프론트엔드 (프론트 서버)
```

---

## 2. 서버 구성 (Oracle Cloud Free Tier)

Oracle Cloud Always Free Tier 한도:
- **ARM A1 Flex:** 최대 4 OCPU + 24 GB RAM (합산, 여러 VM으로 분할 가능)
- **AMD Micro:** 2개 (각 1/8 OCPU, 1 GB RAM)
- **Block Volume:** 총 200 GB

### 서버 1 — DB 서버 (A1 ARM)

| 항목 | 값 |
|---|---|
| Shape | VM.Standard.A1.Flex |
| OCPU | 2 |
| RAM | 12 GB |
| Boot Volume | 100 GB |
| OS | Ubuntu 22.04 (ARM) |
| 역할 | PostgreSQL 16 + Elasticsearch 8 |
| 네트워크 | Private Subnet (공개 IP 없음) |

**설치 소프트웨어:**

| 소프트웨어 | 포트 | 접근 범위 |
|---|---|---|
| PostgreSQL 16 | 5432 | 앱 서버 내부 IP만 허용 |
| Elasticsearch 8 | 9200, 9300 | 앱 서버 내부 IP만 허용 |

> DB 서버는 외부 인터넷에 직접 노출하지 않음. 앱 서버에서만 내부 네트워크로 접근.

---

### 서버 2 — 앱 서버 (A1 ARM)

| 항목 | 값 |
|---|---|
| Shape | VM.Standard.A1.Flex |
| OCPU | 2 |
| RAM | 12 GB |
| Boot Volume | 50 GB |
| OS | Ubuntu 22.04 (ARM) |
| 역할 | Spring Boot API + FastAPI 스크래퍼 + Redis |
| 네트워크 | Public Subnet (공개 IP 있음) |

**설치 소프트웨어:**

| 소프트웨어 | 포트 | 접근 범위 |
|---|---|---|
| Spring Boot (JDK 21) | 8080 | 프론트 서버 IP + 로컬 테스트용 |
| FastAPI (Python 3.11) | 8000 | 내부 전용 (스케줄러 자체 호출) |
| Redis 7 | 6379 | localhost 전용 (외부 노출 안 함) |
| nginx (리버스 프록시) | 80 | 전체 공개 |

**Redis 역할 (단일 인스턴스, 네임스페이스로 구분):**

| 역할 | Key 패턴 | TTL | 주체 |
|---|---|---|---|
| Refresh Token | `refresh:{userId}` | 14일 | Spring Boot |
| API 응답 캐싱 | `cache:clips:popular`, `cache:search:{query}` | 5~10분 | Spring Boot |
| 크롤 중복 방지 | `crawl_lock:{clipUid}` | 1시간 | FastAPI |
| 유저 크롤 쿨다운 | `crawl_cooldown:{channelId}` | 30분 | Spring Boot |

> nginx가 `/api/*` 요청을 Spring Boot(8080)으로 프록시. FastAPI는 내부 크론 스케줄러로만 사용 (외부 노출 불필요).

---

### 서버 3 — 프론트 서버 (AMD Micro)

| 항목 | 값 |
|---|---|
| Shape | VM.Standard.E2.1.Micro |
| OCPU | 1/8 |
| RAM | 1 GB |
| Boot Volume | 50 GB |
| OS | Ubuntu 22.04 |
| 역할 | nginx로 React 빌드 정적 파일 서빙 |
| 네트워크 | Public Subnet (공개 IP 있음) |

**설치 소프트웨어:**

| 소프트웨어 | 포트 | 접근 범위 |
|---|---|---|
| nginx | 80, 443 | 전체 공개 |

> React 앱은 로컬에서 `npm run build` 후 `dist/` 폴더만 서버에 배포. 1 GB RAM으로 빌드하면 메모리 부족이므로 로컬 빌드 필수.

---

## 3. 네트워크 설계 (OCI VCN)

### VCN 구성

| 항목 | 값 |
|---|---|
| VCN CIDR | 10.0.0.0/16 |
| Public Subnet | 10.0.1.0/24 (앱 서버, 프론트 서버) |
| Private Subnet | 10.0.2.0/24 (DB 서버) |

### Security List (방화벽 규칙)

**Public Subnet Ingress (인바운드):**

| 포트 | 프로토콜 | 소스 | 용도 |
|---|---|---|---|
| 22 | TCP | 내 IP | SSH 접속 |
| 80 | TCP | 0.0.0.0/0 | HTTP (앱/프론트) |
| 443 | TCP | 0.0.0.0/0 | HTTPS (앱/프론트) |
| 8080 | TCP | 10.0.1.0/24 | Spring Boot (서버 간 통신) |

**Private Subnet Ingress (인바운드):**

| 포트 | 프로토콜 | 소스 | 용도 |
|---|---|---|---|
| 22 | TCP | 내 IP | SSH 접속 |
| 5432 | TCP | 10.0.1.0/24 | PostgreSQL (앱 서버 → DB) |
| 9200 | TCP | 10.0.1.0/24 | Elasticsearch (앱 서버 → DB) |

---

## 4. 서버별 설치 가이드

### 4.1 DB 서버 설치

```bash
# PostgreSQL 16
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16

# DB 및 사용자 생성
sudo -u postgres psql -c "CREATE USER cheeseboard WITH PASSWORD 'your_password';"
sudo -u postgres psql -c "CREATE DATABASE cheeseboard OWNER cheeseboard;"

# 앱 서버 IP에서 접근 허용 (postgresql.conf)
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/16/main/postgresql.conf

# pg_hba.conf에 앱 서버 IP 추가
echo "host cheeseboard cheeseboard 10.0.1.0/24 md5" | sudo tee -a /etc/postgresql/16/main/pg_hba.conf
sudo systemctl restart postgresql
```

```bash
# Elasticsearch 8 (ARM 지원)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install -y elasticsearch

# JVM 힙 메모리 설정 (12GB 중 4GB 할당)
sudo sed -i 's/-Xms1g/-Xms4g/' /etc/elasticsearch/jvm.options
sudo sed -i 's/-Xmx1g/-Xmx4g/' /etc/elasticsearch/jvm.options

# 내부 네트워크만 바인딩
echo "network.host: 10.0.2.x  # DB 서버 내부 IP로 교체" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
echo "xpack.security.enabled: false" | sudo tee -a /etc/elasticsearch/elasticsearch.yml

sudo systemctl enable elasticsearch && sudo systemctl start elasticsearch
```

---

### 4.2 앱 서버 설치

```bash
# JDK 21 (Spring Boot용)
sudo apt install -y openjdk-21-jdk

# Python 3.11 + venv (FastAPI용)
sudo apt install -y python3.11 python3.11-venv python3-pip
python3.11 -m venv /opt/cheeseboard/venv
source /opt/cheeseboard/venv/bin/activate
pip install fastapi uvicorn httpx pydantic asyncpg sqlalchemy[asyncio] elasticsearch

# Redis 7
sudo apt install -y redis-server
# localhost만 허용 (기본값 확인)
grep "^bind" /etc/redis/redis.conf   # "bind 127.0.0.1 -::1" 이어야 함
sudo systemctl enable redis-server && sudo systemctl start redis-server

# nginx 설치
sudo apt install -y nginx
```

**nginx 리버스 프록시 설정** (`/etc/nginx/sites-available/cheeseboard`):

```nginx
server {
    listen 80;
    server_name your-app-server-ip;

    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

### 4.3 프론트 서버 설치

```bash
# nginx만 설치 (Node.js 빌드는 로컬에서)
sudo apt install -y nginx

# React 빌드 파일 배포 경로
sudo mkdir -p /var/www/cheeseboard
```

**nginx 정적 파일 서빙** (`/etc/nginx/sites-available/cheeseboard`):

```nginx
server {
    listen 80;
    server_name your-front-server-ip;
    root /var/www/cheeseboard;
    index index.html;

    # React Router 새로고침 대응
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 앱 서버 API 프록시 (CORS 우회)
    location /api/ {
        proxy_pass http://앱서버_내부_IP:8080;
        proxy_set_header Host $host;
    }
}
```

---

## 5. 환경 변수 템플릿

### FastAPI 크롤러 (`.env`)

```env
# DB 연결
DB_HOST=10.0.2.x          # DB 서버 내부 IP
DB_PORT=5432
DB_NAME=cheeseboard
DB_USER=cheeseboard
DB_PASSWORD=your_password

# Elasticsearch
ES_HOST=http://10.0.2.x:9200

# Redis (크롤 갱신 중복 방지용 분산 락)
REDIS_HOST=localhost
REDIS_PORT=6379

# 수집 설정
CRAWL_INTERVAL_HOURS=6     # 배치 수집 주기
CRAWL_CONCURRENCY=3        # 동시 요청 수 (semaphore)
INITIAL_STREAMERS_CSV=data/streamers.csv
```

### Spring Boot (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://10.0.2.x:5432/cheeseboard
    username: cheeseboard
    password: your_password
  elasticsearch:
    uris: http://10.0.2.x:9200
  data:
    redis:
      host: localhost
      port: 6379

server:
  port: 8080
```

---

## 6. DB 스키마 (PostgreSQL DDL)

```sql
-- 스트리머
CREATE TABLE streamers (
    channel_id      VARCHAR(32) PRIMARY KEY,
    channel_name    VARCHAR(100) NOT NULL,
    profile_image_url TEXT,
    follower_count  INTEGER,
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- VOD
CREATE TABLE videos (
    video_no        BIGINT PRIMARY KEY,
    channel_id      VARCHAR(32) REFERENCES streamers(channel_id),
    title           TEXT NOT NULL,
    category        VARCHAR(100),
    tags            TEXT[],
    read_count      INTEGER DEFAULT 0,
    duration        INTEGER,
    published_at    TIMESTAMP,
    thumbnail_url   TEXT
);

-- 클립
CREATE TABLE clips (
    clip_uid        VARCHAR(100) PRIMARY KEY,
    channel_id      VARCHAR(32) REFERENCES streamers(channel_id),
    title           TEXT NOT NULL,
    read_count      INTEGER DEFAULT 0,
    duration        INTEGER,
    created_at      TIMESTAMP,
    thumbnail_url   TEXT,
    origin_video_no BIGINT REFERENCES videos(video_no)
);

-- 검색 성능용 인덱스
CREATE INDEX idx_videos_channel ON videos(channel_id);
CREATE INDEX idx_videos_read_count ON videos(read_count DESC);
CREATE INDEX idx_clips_channel ON clips(channel_id);
CREATE INDEX idx_clips_read_count ON clips(read_count DESC);
```

---

## 7. OCI VM 생성 순서

```
1. OCI 콘솔 로그인
   ↓
2. Networking > VCN 생성 (10.0.0.0/16)
   ├── Public Subnet (10.0.1.0/24)
   └── Private Subnet (10.0.2.0/24)
   ↓
3. Security List 규칙 설정 (4절 참고)
   ↓
4. Compute > Instances > Create Instance
   ├── DB 서버: A1 Flex, 2 OCPU, 12 GB, Private Subnet
   ├── 앱 서버: A1 Flex, 2 OCPU, 12 GB, Public Subnet
   └── 프론트 서버: E2.1.Micro, 1 GB, Public Subnet
   ↓
5. SSH 키 등록 (각 VM 생성 시)
   ↓
6. 소프트웨어 설치 (4절 가이드 순서대로)
   ↓
7. 크롤러 초기 수집 실행 (streamers.csv 100개)
```

---

## 8. 리소스 사용량 예상

| 서버 | 프로세스 | RAM 예상 |
|---|---|---|
| DB 서버 (12 GB) | PostgreSQL | ~1–2 GB |
| | Elasticsearch | ~4–6 GB |
| | 여유 | ~4–7 GB ✓ |
| 앱 서버 (12 GB) | Spring Boot | ~512 MB–1.5 GB |
| | FastAPI + uvicorn | ~200–500 MB |
| | Redis 7 | ~100–300 MB |
| | 여유 | ~10 GB ✓ |
| 프론트 서버 (1 GB) | nginx | ~50 MB |
| | 여유 | ~950 MB ✓ |

> 초기 배치 수집(100개 채널) 시 FastAPI의 메모리 피크가 잠깐 올라가지만 앱 서버 12 GB 내에서 여유롭게 처리 가능.
