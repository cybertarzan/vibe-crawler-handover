# 시스템 빌드 및 배포
---

이 가이드는 CI/CD 환경에 의존하지 않고, **순수하게 Docker Compose 파일만을 작성하여 배포**하는 시스템 인수인계용 표준 절차를 안내합니다.

## 1. 사전 준비
*   운영 서버에 Docker 및 Docker Compose 가 설치되어 있어야 합니다.
*   사전에 빌드되어 레지스트리에 푸시된 이미지(`vibe-backend`, `vibe-frontend`, `vibe-migration`)가 필요합니다.
*   동일한 폴더 내에 `.env` 파일을 생성하여 필수 환경변수(DB 커넥션 등)를 입력해야 합니다.

## 2. Docker Compose 파일 작성

운영 서버의 배포를 원하는 디렉토리에 아래의 내용을 `docker-compose.yml` 파일로 저장합니다. 

> **Tip:** `<AWS_ACCOUNT_ID>` 나 `vibe-xx:latest-prod` 와 같은 이미지 주소는 실제 클라우드 레지스트리 경로 또는 로컬 빌드된 이미지명으로 직접 변경해야 합니다.

```yaml
version: "3.8"

services:
  # 1. Database Migration (해당 인프라 환경의 DB 배포 방식에 따라 필요 시 주석 해제하여 사용)
  # migration:
  #   image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/vibe-migration:latest-prod
  #   container_name: vibe-crawler-migration
  #   command: alembic -c /app/alembic.ini upgrade head
  #   environment:
  #     DATABASE_URL: ${DATABASE_URL}
  #   networks:
  #     - vibe-network
  #   restart: "no"

  # 2. Backend API
  backend:
    image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/vibe-backend:latest-prod
    container_name: vibe-crawler-backend
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: ${DATABASE_URL}
      PROJECT_ROOT: /app
      SYSTEM_STORAGE_ROOT: /app/system_storage
      HOST_SYSTEM_STORAGE_ROOT: ${HOST_SYSTEM_STORAGE_ROOT:-/home/ubuntu/vibe-deployment/system_storage}
      CORS_ORIGINS: ${CORS_ORIGINS:-*}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      PYTHONUNBUFFERED: 1
      ENV: prod
      TZ: Asia/Seoul
      MAIL_API_URL: ${MAIL_API_URL:-http://api.solu-tion.co.kr:8080/query}
      SMTP_SENDER_EMAIL: ${SMTP_SENDER_EMAIL:-hslee@mindknock.com}
      PORT: ${PORT:-8080}
      API_BASE_URL: ${API_BASE_URL:-http://backend:8080}
      BASE_URL: ${BASE_URL}
    volumes:
      - ${HOST_STORAGE_PATH:-/home/ubuntu/vibe-deployment/system_storage}:/app/system_storage
      - backend_logs:/app/logs
      - /var/run/docker.sock:/var/run/docker.sock
    # depends_on:
    #   migration:
    #     condition: service_completed_successfully
    networks:
      - vibe-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  # 3. Frontend
  frontend:
    image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/vibe-frontend:latest-prod
    container_name: vibe-crawler-frontend
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - vibe-network
    restart: unless-stopped

networks:
  vibe-network:
    driver: bridge

volumes:
  backend_logs:
    driver: local
```

## 3. 백그라운드 배포 실행

저장된 `docker-compose.yml` 파일이 위치한 폴더에서 아래 명령어를 실행하여 전체 시스템을 띄웁니다.

```bash
docker compose up -d
```

## 4. 배포 헬스체크

정상적으로 모든 컨테이너가 구동되었는지 상태를 확인하고, 백엔드 API 서버의 응답을 점검합니다.

```bash
# 전체 컨테이너 구동 상태 점검
docker compose ps

# 백엔드 API 정상 구동 확인 (Swagger)
curl -f http://localhost:8080/docs
```
