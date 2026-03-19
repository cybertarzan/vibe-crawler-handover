# 좀비 컨테이너 및 디스크 관리 이슈

> 최종 업데이트: 2026-03-17 14:41 (KST)
> 상태: ✅ 주요 이슈 해결 완료 (남은 과제는 하단 참조)

---

## 1. 운영 서버 디스크 분석

- **디스크 상태 (수정 전)**: 96GB 중 96GB 사용 (100% 풀)
- **디스크 상태 (수동 정리 후)**: 96GB 중 47GB 사용 (49%) — 3c042a40 10일 이전 데이터 37GB 삭제
- **주 원인**: 프로젝트 `3c042a40` = 81GB (전체의 84%)
- 28개 날짜 폴더, 날짜별 TOP: 20260316(20GB), 20260317(7.7GB), 20260219(5.6GB)

### 용량이 큰 이유

| 폴더 | 세션당 용량 | 원인 |
|---|---|---|
| action_snapshots | 2.8~6.7GB | 매 액션마다 PNG 스크린샷, 1000+장 |
| videos | 1.1~2.6GB | webm 녹화 파일 1개 |
| downloads (실제 결과물) | 15~47MB | 실제 크롤링 결과 |

---

## 2. 좀비 컨테이너 (e07185ae)

- 3월 16일 07:18 시작 → 17시간째 running
- **원인**: 디스크 풀 → 스크린샷 저장 실패 → 타임아웃 → "무시하고 계속" 로직 → 무한 루프
- 4.1GB 데이터를 남긴 채 progress 불가 상태

---

## 3. 서버 재시작/배포 시 컨테이너 관리

### 3-1. 서버 재시작 시 (`recover_zombie_jobs_service.py`)
- 서버 시작 10초 후 1회 실행
- RUNNING 세션 조회 → `process_identifier` 기준 PID/Docker 확인
- 살아있으면 무시, 죽었으면 FAILED 처리 후 Job 재시작
- **안전장치**: 생성 후 600초(10분) 이내 보호, 최근 60초 업데이트 보호

### 3-2. 배포 시 (`deploy-to-ec2.sh`)
```
docker-compose down        ← backend/frontend/db만 내림
docker system prune -a -f  ← 미사용 이미지/네트워크 정리
docker-compose up -d       ← 새 이미지로 재시작
```
- **한계**: compose 외부 크롤러 컨테이너(`crawler-xxx`)는 안 죽음

### 3-3. Docker Cleanup Service (1시간 주기, prod만)
- 24시간 이상 미사용 리소스만 정리
- 실행 중인 컨테이너는 건드리지 않음

### 감지 능력 요약

| 상황 | 감지 | 이유 |
|---|---|---|
| 컨테이너가 죽어있음 | ✅ | docker ps로 확인, FAILED 처리 |
| 컨테이너가 살아있지만 무한루프 | ❌ | docker ps에 뜨므로 "정상" 판단 |
| 배포 시 정리 | ❌ | compose 외부 컨테이너라 미포함 |

---

## 4. 해결 조치 (2026-03-17 적용)

### ✅ 4-1. 고아 컨테이너 자동 정리 추가

Docker Cleanup Service에 **역방향 검사** 로직 추가:
1. `docker ps --filter name=crawler-*`로 실행 중인 크롤러 컨테이너 목록 조회
2. 컨테이너명에서 `session_id` 추출
3. DB에서 해당 세션이 RUNNING인지 확인
4. RUNNING이 아니거나 DB에 없으면 → `docker stop` + `docker rm -f`

**변경 파일:**
- `docker_cleanup_port.py` — 메서드 2개 추가
- `docker_cleanup_adapter.py` — 구현
- `docker_cleanup_service.py` — 고아 정리 단계 추가
- `test_docker_orphan_cleanup.py` — 7개 테스트 전부 통과 ✅

### ✅ 4-2. 로그 자동 정리 경로 버그 수정

**증상**: Auto Cleanup ON + 보관 10일인데 21일 전 데이터 남아있음

**원인**: `get_old_session_directories()`에서 `self.base_path / self.RESULTS_DIR`(`system_storage/crawling_results/`)을 스캔했으나, 실제 데이터는 `self.results_base_path`(`project_root/crawling_results/`)에 저장됨 → 경로 불일치로 "삭제할 거 없음" 반환

**수정**: 1줄 변경
```diff
# file_system_adapter.py:1306
- results_dir = self.base_path / self.RESULTS_DIR
+ results_dir = self.results_base_path
```

**재발 방지**: 경로 일관성 테스트 3개 추가 (`TestPathConsistency`)
- Legacy/Workspace 구조별 저장 경로 == 정리 스캔 경로 일치 검증
- E2E: 세션 생성 후 정리 로직이 실제로 찾는지 검증

**테스트**: cleanup_old_logs 17개 전부 통과 ✅

---

## 5. 남은 과제

- [ ] 실행 시간 기반 타임아웃 (예: 4시간 이상 running이면 강제 종료)
- [ ] 취소/실패 세션의 대용량 데이터(action_snapshots, videos) 자동 삭제
- [ ] 크롤러 에러 핸들링 개선 (스크린샷 실패 시 무한루프 방지)
- [x] ~~좀비 세션 FAILED 전환 시 프로세스 kill 누락~~ → 4-3에서 해결
- [x] ~~OS 레벨 좀비 프로세스(defunct) 자동 수거~~ → 4-4에서 해결

---

## ✅ 4-3. 좀비 세션 FAILED 시 프로세스 Kill 로직 추가

**증상**: DB에서 세션이 FAILED로 변경되었지만 실제 Python/Chromium 프로세스가 종료되지 않아 메모리 누수 발생
- 예: `4e051756` 세션 — DB `failed` 상태인데 프로세스가 5시간째 메모리 1.1GB 차지

**원인**: `_mark_session_as_failed()` (recover)와 `_detect_and_handle_zombie_sessions()` (sync)에서 DB 상태만 변경, 프로세스 kill 미실행

**수정**:
- `recover_zombie_jobs_service.py` — `_mark_session_as_failed()`에 `_kill_process()` 호출 추가
- `sync_job_status_service.py` — 좀비 세션 FAILED 전환 시 `_kill_process()` 호출 추가
- Local(PID) → `os.kill(SIGTERM)`, Docker → `docker stop + rm`

**테스트**: recover_zombie 16개 전부 통과 ✅

---

## ✅ 4-4. OS 레벨 좀비 프로세스(defunct) 자동 수거

**증상**: 운영 서버에서 `[chrome] <defunct>` 좀비 프로세스 발견 (PID 414802, 부모=uvicorn)
- 기존 `_kill_orphan_processes()`는 살아있는 프로세스 대상 → 이미 죽은 좀비에 효과 없음

**원인**: 로컬 모드 크롤링 시 Python 프로세스 먼저 종료 → Chrome이 uvicorn의 양자로 이관 → uvicorn이 `waitpid()` 미호출 → 좀비(Z 상태)로 잔류

**수정**:
- `sync_job_status_service.py` — `_reap_zombie_children()` 메서드 추가
  - `/proc/[PID]/stat`에서 상태=Z + 부모PID=현재프로세스 필터링
  - `os.waitpid(pid, os.WNOHANG)`으로 안전 수거 (asyncio 충돌 없음)
  - macOS(/proc 없는 환경) 자동 스킵
- `sync()` 메서드에서 30초 주기 자동 호출

**테스트**: test_reap_zombie 6개 전부 통과 ✅