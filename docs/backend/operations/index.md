# 백엔드 — 운영 가이드

---

## 인프라 정보

### 서버 구성

| 환경 | 호스트 | 접근 방법 |
|------|--------|----------|
| **Dev/Prod** | EC2 (배포 시 동적 할당) | `ssh -i heesun.pem ubuntu@<IP>` |
| **로컬 개발** | localhost | 직접 실행 |

### Docker Compose 서비스

| 컨테이너 | 이미지 | 포트 | 역할 |
|----------|--------|:----:|------|
| `vibe-crawler-db` | `postgres:15-alpine` | 5432 | 메인 DB |
| `vibe-crawler-migration` | `vibe-migration:latest` | - | Alembic 마이그레이션 (1회 실행) |
| `vibe-crawler-backend` | `vibe-backend:latest` | 6767→8080 | API 서버 + 스케줄러 |

### AWS 정보

| 항목 | 값 |
|------|-----|
| AWS Account ID | `850205788233` |
| Region | `ap-northeast-2` (서울) |
| ECR | `vibe-backend`, `vibe-migration` |
| SSH Key | `heesun.pem` |


