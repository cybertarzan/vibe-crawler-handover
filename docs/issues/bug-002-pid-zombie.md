# [BUG-002] PID 저장 실패로 인한 크롤링 세션 좀비화 및 프로세스 잔류 이슈

## 개요

| 항목 | 내용 |
|---|---|
| **세션** | `e8584b2e-afae-4be7-84e7-ca2a95d1fc12` |
| **Job** | `8463771b-d7b4-4e98-9097-5abb13c4c4d0` |
| **발생 시각** | 2026-03-18 07:00 KST |
| **증상** | DB에서는 FAILED 상태이나, 크롤러 프로세스(PID 140628)와 Chromium이 여전히 실행 중 (CPU 28.8%) |
| **상태** | ✅ 해결 완료 (프로세스 자연 종료 + 코드 수정 반영) |

---

## 근본 원인

**PID 저장 실패 → 좀비 감지기 오판 → 프로세스 미종료**

1. **PID 콜백 DB 세션 문제**: `_pid_callback`이 외부 `session_repo`의 DB 세션을 공유하여 커밋이 안 됨 → `process_identifier = NULL`
2. **좀비 감지기 오판**: 유예 5분 초과 + PID NULL + 결과 파일 없음 → FAILED 처리 (실제로는 정상 실행 중)
3. **프로세스 미종료**: `process_identifier = NULL`이면 kill 조건문이 스킵됨 → 좀비 프로세스 방치

---

## 타임라인

| 시간 (KST) | 이벤트 |
|---|---|
| 07:00:00 | ✅ 스케줄러가 PENDING 세션 생성 |
| 07:00:12 | ✅ 크롤러 프로세스 시작 (PID 140628), `process.communicate()` 블로킹 대기 |
| 07:01:12 | ❌ PENDING 스케줄러가 동일 세션에 중복 실행 시도 → API 60초 타임아웃 |
| 07:05:26 | ⚠️ 좀비 감지기가 PID 없음 + 유예 초과로 FAILED 처리 |

---

## 핵심 버그

### 버그 #1: PID 저장 실패
- **위치**: `start_crawling_service.py` L408-421
- **원인**: `_pid_callback` 클로저가 외부 DB 세션 참조 → 블로킹 중 커밋 안 됨
- **수정**: 독립 `AsyncSessionLocal()` DB 세션으로 PID 저장 + 즉시 커밋

### 버그 #2: FAILED 처리 시 프로세스 미종료
- **위치**: `sync_job_status_service.py` L279-281
- **원인**: `if session.process_identifier:` 조건이 NULL이면 항상 스킵
- **수정**: `_kill_orphan_processes()` 메서드 추가 — `pgrep -f session_id`로 고아 프로세스 검색 후 SIGTERM

---

## 수정 결과

| 파일 | 수정 내용 |
|---|---|
| `start_crawling_service.py` | `_pid_callback`에서 독립 DB 세션 사용 |
| `start_crawling_service.py` | ✅ `_execute_pending_session`에 `_pid_callback` 추가 (스케줄 실행 PID 누락 수정) |
| `sync_job_status_service.py` | `_kill_orphan_processes()` 메서드 추가 |

- **테스트**: 221개 전부 통과 ✅ (PID 타이밍 테스트 11개 포함)
- **좀비 프로세스**: 자연 종료 확인 (07:49 시점 컨테이너 내 uvicorn만 존재)

---

## 추가 수정 (2026-03-19)

### 버그 #3: 스케줄 실행 경로 PID 콜백 누락
- **위치**: `start_crawling_service.py` `_execute_pending_session` L1520
- **원인**: `_run_crawling_background`에만 `_pid_callback`이 있고, `_execute_pending_session`에는 누락
- **영향**: 스케줄 실행 세션은 `process_identifier=NULL` → 유예 5분 초과 시 좀비 오판 → SIGTERM
- **수정**: `_execute_pending_session`에도 동일한 독립 DB 세션 `_pid_callback` 추가
- **검증**: `test_pid_save_timing.py` 11개 테스트로 비대칭 검증 완료

