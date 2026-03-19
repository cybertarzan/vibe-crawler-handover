# 외부 서비스 연동

---

## Incross i-flow API

프로젝트 생성 시 광고주/캠페인/매체 정보를 조회하는 Incross 사 연동 API.

| 항목 | Dev | Prod |
|------|-----|------|
| Base URL | `https://dev.i-flow.kr/request/api` | `https://i-flow.kr/request/api` |
| 연결 모드 | local: `tunnel`, dev/prod: `direct` | `direct` |

- 활성화 조건: `is_api_transmission_enabled = True` + Workspace `data_api_url` 설정
- **Adapter**: `incross_api_adapter.py` (직접) / `incross_api_tunnel_adapter.py` (SSH 터널)
- **용도**: `get_advertisers()`, `get_campaigns()`, `get_media()`

---

## 데이터 업로드 API (후처리)

크롤링 완료 후 수집된 데이터를 i-flow에 업로드하는 2단계 API.

| 단계 | API | 설명 |
|------|-----|------|
| 1차 | `analyze` | CSV 파일 분석 (컬럼 매핑, 유효성 검증) |
| 2차 | `upload` | 분석 완료된 데이터를 i-flow에 업로드 |

- **Adapter**: `data_upload_adapter.py` (직접) / `data_upload_tunnel_adapter.py` (SSH 터널)
- **결과 저장**: `post_crawling_api_results` 테이블에 단계별 기록

---

## Excel 매체 분석기 API

크롤링으로 수집한 CSV/Excel 파일의 컬럼을 자동 분석하는 서비스.

- **Adapter**: `excel_media_analyzer_adapter.py`
- **용도**: 수집 파일의 컬럼 구조 파악, 매핑 자동화

---

## OpenAI API

- **용도**:
    - AI Auto Fix — 크롤링 실패 분석 및 스크립트 수정
    - 입력정보 추가 — 크롤링 대상 사이트 분석 및 입력 항목 자동 생성
    - VLM (Vision Language Model) — 스크린샷 기반 화면 분석
- **Adapter**: `open_ai_vision_adapter.py`
- **키 관리**: `.env` 파일의 `OPENAI_API_KEY`
- **타임아웃**: 1800초 (30분)

---

## 스크립트 다운로드 API

원격 저장소에서 크롤링 스크립트를 다운로드하는 서비스.

- **Adapter**: `script_download_adapter.py`
- **용도**: 프로젝트 생성/업데이트 시 스크립트 파일 동기화

---

## Remote Executor (워크플로우)

워크플로우 내 Python 스크립트를 원격 서버에서 실행하는 서비스.

- **Adapter**: `remote_executor_adapter.py`
- **용도**: 워크플로우 노드에서 스크립트 원격 실행

---

## Mail 알림 API

- **URL**: `.env` 파일의 `MAIL_API_URL`
- **Adapter**: `email_notification_adapter.py`
- **트리거**: Job 완료/실패 시 알림 수신자에게 메일 발송

---

## 환경변수 요약

| 변수 | 용도 |
|------|------|
| `OPENAI_API_KEY` | OpenAI API 인증 |
| `MAIL_API_URL` | 메일 발송 API 엔드포인트 |
| `INCROSS_API_URL` | i-flow API Base URL |
| `SSH_TUNNEL_*` | 로컬 개발용 SSH 터널 설정 |
