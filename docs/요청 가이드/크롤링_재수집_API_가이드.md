# 크롤링 재수집(Refetch) API 연동 가이드

본 문서는 Vibe Crawling Platform 백엔드의 **재수집(Refetch) API**를 외부 시스템이나 다른 서비스에서 직접 호출하여 연동하기 위한 가이드입니다.

---

## 1. API 개요

재수집(Refetch) API는 기존에 등록된 크롤링 스크립트를 코드를 수정하지 않고 **특정 날짜 범위(`sdate`, `edate`)를 지정하여 다시 실행**할 때 사용합니다. 호출 시 백엔드에서는 새로운 크롤링 세션(Session)을 생성하고 비동기적으로 스크립트를 실행합니다.

- **URL Path**: `/api/v1/jobs/{job_id}/refetch`
- **HTTP Method**: `POST`
- **Content-Type**: `application/json`

---

## 2. API 상세 스펙

### 2.1. Request Path Variable
| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| `job_id` | String(UUID) | **Yes** | 크롤링 대상이 되는 Job의 고유 ID |

### 2.2. Request Query Parameter
| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| `workspace_id` | String | No | 멀티 워크스페이스 환경일 경우 지정하는 워크스페이스 ID (필요 시 포함) |

### 2.3. Request Body (JSON)
재수집할 대상 데이터의 시작 날짜와 종료 날짜를 지정합니다. 형태는 자유로우나 크롤러 스크립트가 파싱 가능한 형태의 날짜 문자열(예: `YYYY-MM-DD` 또는 `YYYYMMDD`)을 권장합니다.
| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| `sdate` | String | No | 수집 시작 일자 (예: `"2023-10-01"`) |
| `edate` | String | No | 수집 종료 일자 (예: `"2023-10-31"`) |

**요청 (Request) 예시:**
```bash
curl -X POST "https://{API_HOST}/api/v1/jobs/123e4567-e89b-12d3-a456-426614174000/refetch?workspace_id=my-workspace" \
    -H "Content-Type: application/json" \
    -d '{
        "sdate": "2023-10-01",
        "edate": "2023-10-02"
    }'
```

### 2.4. Response
요청이 정상적으로 접수되면, 생성된 세션 ID와 작업의 현재 상태를 포함한 JSON이 반환됩니다. (크롤링 자체는 백그라운드에서 비동기적으로 수행됩니다.)

**HTTP Status Response:**
- `200 OK`: 재수집 요청 성공
- `404 Not Found`: 존재하지 않는 `job_id`
- `500 Internal Server Error`: 서버 내부 오류 발생

**응답 (Response) 파라미터:**
| Name | Type | Description |
| ---- | ---- | ----------- |
| `message` | String | 처리 결과 메시지 |
| `session_id` | String(UUID) | 신규로 생성된 크롤링 세션(실행 단위)의 고유 ID |
| `job_id` | String(UUID) | 요청한 Job ID |
| `status` | String | 세션의 현재 상태 (예: `"running"`, `"pending"` 등) |

**응답 (Response) 예시:**
```json
{
    "message": "Refetch가 성공적으로 시작되었습니다.",
    "session_id": "987fcdeb-51a2-43d7-9012-3456789abcde",
    "job_id": "123e4567-e89b-12d3-a456-426614174000",
    "status": "running"
}
```

---

## 3. 세션 상태 조회 API

재수집 요청으로 발급된 `session_id`를 사용하여 크롤링의 현재 상태 및 최종 결과를 조회하는 API입니다.

- **URL Path**: `/api/v1/sessions/{session_id}`
- **HTTP Method**: `GET`

### 3.1. Request Path Variable
| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| `session_id` | String(UUID) | **Yes** | 상태를 확인할 대상 세션 ID (`job_id`가 아닌 `session_id`) |

### 3.2. Response 예시

실행 진행 상황을 나타내는 상태(`status`)와 최종 데이터 수집 성공 여부(`success`)를 반환합니다. 
`status` 값이 `completed` 또는 `failed`, `cancelled` 중 하나일 때 실행이 모두 종료된 것입니다.

```json
{
    "id": "987fcdeb-51a2-43d7-9012-3456789abcde",
    "job_id": "123e4567-e89b-12d3-a456-426614174000",
    "status": "completed",
    "config": {
        "trigger_source": "refetch",
        "sdate": "2023-10-01",
        "edate": "2023-10-02"
    },
    "created_at": "2023-10-05T10:00:00.000000",
    "started_at": "2023-10-05T10:00:05.123456",
    "completed_at": "2023-10-05T10:05:00.987654",
    "updated_at": "2023-10-05T10:05:00.987654",
    "execution_time": 295.8,
    "success": true,
    "error_log_file_path": null,
    "session_summary": null,
    "ai_auto_fix_status": null
}
```
