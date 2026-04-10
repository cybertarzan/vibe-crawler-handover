# Vibe Crawling Platform 인수인계 문서

> 최종 업데이트: 2026-03-19

## 📋 문서 구성

### Backend

| 문서 | 내용 |
|---|---|
| [프로젝트 개요](backend/01-project-overview.md) | 기술 스택, 디렉토리 구조, 핵심 개념 |
| [아키텍처 및 설계](backend/02-architecture-and-design.md) | 헥사고날 아키텍처, 모듈 구조, 데이터 흐름 |
| [운영 가이드](backend/03-operations-guide.md) | 배포, 모니터링, 트러블슈팅 |
| [이슈 및 히스토리](backend/04-issues-and-history.md) | 알려진 이슈, 변경 이력 |

### Frontend

| 문서 | 내용 |
|---|---|
| [프로젝트 개요](frontend/01-project-overview.md) | 기술 스택, 디렉토리 구조, 핵심 개념 |
| [아키텍처 및 설계](frontend/02-architecture-and-design.md) | 컴포넌트 구조, 상태 관리, API 연동 |
| [운영 가이드](frontend/03-operations-guide.md) | 배포, 환경 설정 |
| [이슈 및 히스토리](frontend/04-issues-and-history.md) | 알려진 이슈, 변경 이력 |

### 계획서

| 문서 | 내용 |
|---|---|
| [크롤링 서버 증량](plans/server-scaling.md) | 서버 증량 계획, 인프라 구성 |
| [워크스페이스 이관 (MINDK→INCRO)](plans/workspace-migration.md) | 프로젝트 이관 절차 및 결과 |

### 이슈

| 문서 | 내용 |
|---|---|
| [BUG-002 PID 저장실패 / 좀비세션](issues/bug-002-pid-zombie.md) | PID 저장 실패로 인한 좀비 세션 이슈 |
| [로그 자동 정리 검증](issues/log-cleanup-checklist.md) | 로그 자동 정리 체크리스트 및 검증 결과 |
| [운영서버 스토리지 분석](issues/storage-analysis.md) | 운영서버 디스크 사용량 분석 |
| [좀비 컨테이너 및 디스크 관리](issues/zombie-container-disk.md) | 좀비 컨테이너 정리 및 디스크 관리 |

### 견적서

| 문서 | 내용 |
|---|---|
| [[ISS-003] 크롤러-워커 통합 견적서](estimates/ISS-003_크롤러-워커분리/ISS-003_최종견적서.md) | Vibe Crawler 시스템 성능 최적화를 위한 워커 분배 아키텍처 도입 견적 및 기능 명세 |
