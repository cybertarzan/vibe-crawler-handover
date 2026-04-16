# 시스템 모니터링 및 로그

---

운영 중인 시스템의 상태를 확인하고 장애 발생 시 로그를 추적하는 표준 방법론입니다.

## 1. 운영 컨테이너 로그 모니터링

`docker-compose.yml`이 존재하는 동일한 디렉토리에서 각 서비스의 실시간 화면 출력(STDOUT) 로그를 추적할 수 있습니다.

### 백엔드 (Backend) 로그
```bash
# 실시간 백엔드 로그 꼬리물기 (-f 옵션)
docker compose logs -f backend

# 에러 로그만 필터링하여 확인 (장애 상황 시)
docker compose logs -f backend | grep -i "error"
```

### 프론트엔드 (Frontend) 로그
```bash
# 실시간 프론트엔드 로그 (접속 불량, 렌더링 에러 확인 시)
docker compose logs -f frontend
```

---

## 2. 크롤링 결과물 및 파일 로그 구조

백엔드 컨테이너의 핵심 데이터(수집 결과물, 영상, 스크린샷, 세부 디버그 로그)는 영구 보존을 위해 `docker-compose.yml` 의 볼륨 바인딩 설정에 따라 호스트 마상의 `system_storage/` 에 지속적으로 저장됩니다.

```text
system_storage/
└── workspaces/
    └── {workspace_id}/
        └── projects/
            └── {project_id}/
                ├── scripts/             # 크롤러 엔진 스크립트
                └── sessions/
                    └── {YYYYMMDD}/
                        └── {session_id}/
                            ├── action_snapshots/  # 단계별 스크린샷 (디버깅용)
                            ├── videos/            # 화면 녹화 파일 (디버깅용)
                            ├── downloads/         # 수집 결과물 (CSV 엑셀 파일 등)
                            ├── crawling_result.json # 세션 최종 요약본
                            └── logs/              # 크롤러 인스턴스 전용 상세 로그 
```

> **Tip:** 화면상에 `FAILED` 로 처리된 특정 크롤링 세션의 상세한 실패 사유("결과 테이블 로딩 대기 30초 초과" 등)를 파악하고자 할 때는, `docker compose logs backend` 보다 직접 파일 시스템에 저장된 위 `logs/` 디렉토리 속 텍스트 파일을 조회하는 것이 가장 정확합니다.

---

## 3. 기본 장애 대응 및 복구 조치

1. **로그 추적:** 시스템 전체에 장애가 있는지 판단하려면 `docker compose logs -f backend`를 확인합니다. 특정 세션에만 에러가 났다면 `system_storage/` 의 해당 세션 로그 파일을 조회합니디.
2. **단순 컨테이너 재시작:** 서비스 메모리 누수나 일시적 뻗음 현상이 의심될 때
   ```bash
   docker compose restart backend
   docker compose restart frontend
   ```
3. **완전 재기동 (Cold Reboot):** 기존 네트워크나 볼륨 의존성이 꼬였을 때 사용하는 최후의 수단입니다.
   ```bash
   docker compose down
   docker compose up -d
   ```
