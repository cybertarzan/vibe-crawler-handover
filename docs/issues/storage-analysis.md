# 운영서버 스토리지 분석 리포트

> 분석일: 2026-03-17 15:22 KST

---

## 1. 디스크 현황

| 항목 | 값 |
|---|---|
| 전체 디스크 | 96GB |
| 사용 중 | 51GB (53%) |
| 여유 | 46GB |
| `vibe-deployment/` | 37GB |

---

## 2. system_storage 디렉토리별 용량

| 디렉토리 | 용량 | 자동 정리 | 상태 |
|---|---|---|---|
| `workspaces/` | **36GB** | ✅ 10일 보관 | 정상 (세션 로그 자동 삭제) |
| `workflow_logs/` | 62MB | ❌ **미정리** | ⚠️ 1/20~3/17 96개 계속 누적 |
| `mdc/` | 26MB | ❌ **미정리** | ⚠️ data/videos 폴더 |
| `projects/` (Legacy) | 20MB | ❌ **미정리** | ⚠️ 6개 프로젝트 스크립트 잔존 |
| `hotfixes/` | 68KB | ❌ | 3개 파일 (유지 필요) |
| `sessions/` (Legacy) | 36KB | ❌ **미정리** | ⚠️ 4개 에러 로그 잔존 |
| `enhanced_scripts/` | 8KB | - | 비어있음 |
| `crawling_results/` (Legacy) | 4KB | - | 비어있음 |

---

## 3. 워크스페이스 상세

| 워크스페이스 | 용량 | 프로젝트 수 |
|---|---|---|
| MINDK (`22222222-...`) | **33GB** | 6개 |
| INCRO (`11111111-...`) | 3.5GB | 16개 |
| `legacy_archive` | 424KB | 0개 |

---

## 4. Docker 리소스

| 항목 | 용량 | 회수 가능 |
|---|---|---|
| 이미지(총 4개) | 10.5GB | **8GB** (미사용 2개: 크롤러 이미지) |
| Build Cache | 7.6GB | **3.6GB** |
| Local Volumes | 50MB | **50MB** (전부 미사용) |
| 컨테이너 | 34MB | 0 |

### 미사용 크롤러 이미지

```
vibe-crawler-774d5129-...:latest  4GB   ← 미사용
vibe-crawler-e07185ae-...:latest  4GB   ← 미사용 (이전 좀비 세션)
```

---

## 5. 정리되지 않는 파일 목록

### ⚠️ 5-1. workflow_logs (62MB, 96개)

- **경로**: `system_storage/workflow_logs/`
- **내용**: 워크플로우 실행 JSON 로그 (세션별 1개씩 생성)
- **범위**: 2025-12-30 ~ 2026-03-17
- **문제**: 자동 정리 로직 없음, 무한 누적
- **권장**: `cleanup_old_logs_runner`와 같은 보관 기간 적용 (10일)

### ⚠️ 5-2. projects/ (Legacy, 20MB)

- **경로**: `system_storage/projects/`
- **내용**: 과거 구조에서 생성된 스크립트 6개 프로젝트분
- **문제**: 현재는 `workspaces/*/projects/*/scripts/`에 저장 → 이중 보관
- **권장**: workspace 내 동일 프로젝트에 스크립트 존재 확인 후 삭제

### ⚠️ 5-3. sessions/ (Legacy, 36KB)

- **경로**: `system_storage/sessions/`
- **내용**: 과거 세션 에러 로그 4개
- **문제**: 더 이상 여기에 기록 안 함
- **권장**: 삭제 가능

### ⚠️ 5-4. mdc/ (26MB)

- **경로**: `system_storage/mdc/`
- **내용**: MDC 관련 data/videos
- **문제**: 자동 정리 없음
- **권장**: 사용 빈도 확인 후 보관 정책 결정

### ⚠️ 5-5. Docker 미사용 리소스 (약 11.6GB)

| 리소스 | 회수 가능 용량 |
|---|---|
| 미사용 크롤러 이미지 2개 | ~8GB |
| Build Cache | ~3.6GB |
| Local Volumes (미사용) | ~50MB |
| **합계** | **~11.6GB** |

- **권장**: `docker system prune -a` 또는 `docker image prune -a`

### ⚠️ 5-6. legacy_archive 워크스페이스 (424KB)

- **경로**: `workspaces/legacy_archive/`
- **문제**: 프로젝트 0개, 용도 불명
- **권장**: 삭제 가능

---

## 6. 즉시 회수 가능 용량

| 대상 | 용량 |
|---|---|
| Docker 미사용(이미지+캐시+볼륨) | ~11.6GB |
| Legacy sessions/ | 36KB |
| Legacy projects/ (확인 후) | 20MB |
| legacy_archive/ | 424KB |
| **합계** | **~11.6GB** |

## 7. 장기 개선 필요

| 대상 | 현재 | 권장 |
|---|---|---|
| `workflow_logs/` | 무한 누적 | 자동 정리 로직 추가 |
| `mdc/` | 무한 누적 | 보관 정책 수립 |
| Docker 이미지 | 크롤러 이미지 수동 삭제만 | 빌드 후 자동 정리 |
