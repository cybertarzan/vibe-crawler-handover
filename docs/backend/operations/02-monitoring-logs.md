# 모니터링 및 로그

---

## 로그 확인

```bash
# Docker 환경
ssh -i heesun.pem ubuntu@<IP>
cd ~/vibe-deployment
docker-compose logs -f backend

# 로컬 환경
uvicorn shared.main:app --host 0.0.0.0 --port 6767 --reload
```

## 크롤링 로그 파일 구조 (Workspace 기반)

```
system_storage/
├── workspaces/
│   └── {workspace_id}/
│       └── projects/
│           └── {project_id}/
│               ├── scripts/             # 크롤링 스크립트
│               ├── hotfixes/            # 핫픽스 스크립트
│               ├── mdc/                 # MDC 관련 파일
│               └── sessions/
│                   └── {YYYYMMDD}/
│                       └── {session_id}/
│                           ├── action_snapshots/  # 액션별 PNG 스크린샷
│                           ├── videos/            # webm 녹화 파일
│                           ├── downloads/         # 수집 파일 (CSV 등)
│                           ├── crawling_result.json
│                           └── logs/              # 실행 로그
└── crawling_results/                    # Legacy 경로 (사용 안 함)
```

> 워크스페이스별 디렉토리  
> - MINDK: `workspaces/22222222-2222-2222-2222-222222222222/`  
> - INCRO: `workspaces/11111111-1111-1111-1111-111111111111/`

## 크롤러 Docker 컨테이너

크롤링 실행 시 세션별 Docker 컨테이너가 생성됩니다.

| 항목 | 값 |
|------|---|
| 컨테이너 명명 | `crawler-{session_id}` |
| 실행 방식 | 형제 컨테이너 (`/var/run/docker.sock` 마운트) |
| 브라우저 | Playwright (Firefox/Chromium) |

## 스케줄러 Runner 목록

| Runner | 주기 | 설명 |
|--------|------|------|
| `scheduled_job_runner` | Cron 기반 | 프로젝트별 Cron 스케줄에 따라 세션 생성 |
| `pending_sessions_runner` | 15초 | PENDING 세션 픽업 → 크롤러 실행 |
| `pending_jobs_runner` | 15초 | 놓친 스케줄 캐치업 |
| `sync_job_status_runner` | 30초 | 실행 중 Job 상태 동기화 (좀비 감지) |
| `cleanup_old_logs_runner` | 매일 04:00 KST | `log_retention_days` 초과 세션 삭제 |
| `docker_system_cleanup_runner` | 1시간 | 미사용 이미지/캐시 + 고아 컨테이너 정리 |
| `recover_zombie_jobs_runner` | 서버 시작 10초 후 1회 | RUNNING 비정상 Job 복구 |

## 에러 대응 절차

1. 프론트 세션 목록에서 FAILED 세션 → 로그 탭 확인
2. `docker-compose logs -f backend | grep ERROR`
3. `docker-compose exec db psql -U vibe_user -d vibe_crawling_db`
4. `docker-compose restart backend`
5. `./deploy-to-ec2.sh <IP>` (전체 재배포)
