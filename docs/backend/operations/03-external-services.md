# 외부 서비스 연동

---

## Incross i-flow API

| 항목 | Dev | Prod |
|------|-----|------|
| Base URL | `https://dev.i-flow.kr/request/api` | `https://i-flow.kr/request/api` |
| 연결 모드 | local: `tunnel`, dev/prod: `direct` | `direct` |

- 활성화 조건: `is_api_transmission_enabled = True` + Workspace `data_api_url` 설정

## OpenAI API

- **용도**: AI Auto Fix (크롤링 실패 분석 및 스크립트 수정), 입력정보 추가 (크롤링 대상 사이트 분석 및 입력 항목 자동 생성)
- **키 관리**: `.env` 파일의 `OPENAI_API_KEY`

## Mail 알림 API

- **URL**: `MAIL_API_URL`
- **트리거**: Job 완료/실패 시 알림 수신자에게 메일 발송
