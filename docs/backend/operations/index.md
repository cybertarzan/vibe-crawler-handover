# 백엔드 — 운영 가이드

---

## 시스템 접속 정보

### 서버 운영 구성

| 환경 | 접속 도메인 (URL) | 상세 설명 |
|------|--------|----------|
| **운영 (Prod)** | [http://crawler.i-flow.kr](http://crawler.i-flow.kr) | 실 구동중인 운영 서버 |
| **개발 (Dev)** | [http://180.210.80.180](http://180.210.80.180) | 테스트 및 신규 기능 배포용 개발 서버 |
| **로컬 (Local)** | `localhost` | 로컬 개발자 환경 |

---

## Docker Compose 운영 서비스

배포 시 `docker-compose.prod.yml` 에 의해 코어 인프라로 구동되는 핵심 컨테이너 구조입니다.

| 컨테이너 명칭 | 내부 포트 | 역할 및 특징 |
|----------|:----:|------|
| `vibe-crawler-db` | 5432 | 메인 PostgreSQL 데이터베이스 (외부 DB 사용 시 제외 가능) |
| `vibe-crawler-backend`  | 8080 | 플랫폼 API 서버 제공 및 크롤링 스케줄링 관리 주체 |
| `vibe-crawler-frontend` | 80 | 사용자 접근용 UI 뷰어 및 다운로드 포털 웹 서버 |


