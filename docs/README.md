# Cheeseboard 문서

이 디렉터리는 Cheeseboard 개발에 필요한 제품, 설계, 명세, 작업 가이드를 모아 둔 공간입니다.

## 문서 구조

| 디렉터리 | 역할 |
|---|---|
| [`product/`](./product/) | 서비스 목표, 요구사항, 기능 명세, 로드맵을 정리합니다. |
| [`architecture/`](./architecture/) | 시스템 구조, 인프라, 데이터베이스 설계를 설명합니다. |
| [`api/`](./api/) | API 엔드포인트와 요청/응답 명세를 정리합니다. |
| [`guides/`](./guides/) | 서버 설정, 스크래퍼 실행처럼 특정 작업을 수행하는 절차를 설명합니다. |
| [`decisions/`](./decisions/) | 주요 기술 결정과 그 이유를 기록합니다. |
| [`design/`](./design/) | 화면 설계, 와이어프레임, UX 관련 자료를 보관합니다. |

## 먼저 읽을 문서

1. [`product/overview.md`](./product/overview.md) - 서비스 개요
2. [`product/requirements.md`](./product/requirements.md) - 요구사항
3. [`product/functional-spec.md`](./product/functional-spec.md) - 기능 명세
4. [`architecture/infrastructure.md`](./architecture/infrastructure.md) - 인프라 구조
5. [`architecture/database-schema.md`](./architecture/database-schema.md) - 데이터베이스 스키마
6. [`api/spec.md`](./api/spec.md) - API 명세
7. [`guides/server-setup.md`](./guides/server-setup.md) - 서버 설정 가이드
8. [`guides/scraper.md`](./guides/scraper.md) - 스크래퍼 가이드

## 문서 종류

- 제품 문서는 서비스가 무엇을 해야 하는지와 왜 필요한지를 설명합니다.
- 아키텍처 문서는 시스템이 어떻게 구성되는지와 주요 설계 방향을 설명합니다.
- API 문서는 클라이언트와 서버가 지켜야 할 계약을 정의합니다.
- 가이드 문서는 특정 작업을 완료하기 위한 절차를 단계별로 설명합니다.
- 결정 문서는 중요한 기술 선택과 대안을 기록합니다.

## 작성 규칙

- 문서 본문은 한국어로 작성합니다.
- 파일명, 명령어, API 이름, 클래스명, 설정 키는 필요한 경우 영어 원문을 유지합니다.
- 새 문서를 추가할 때는 이 README의 문서 구조에 맞는 위치에 둡니다.
- 오래된 내용은 삭제하기보다 상태나 변경 이력을 남겨 추적할 수 있게 합니다.
