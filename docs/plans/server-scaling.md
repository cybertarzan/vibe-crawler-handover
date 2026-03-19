# 크롤링 서버 증량 계획서

> 작성일: 2026-03-18 KST  
> 대상: 운영(Prod) 환경  
> 목적: 크롤링 처리 용량 확장 및 배치 선점 문제 해결

---

## 1. 현재 아키텍처 분석

### 1.1 서버 구성 (단일 EC2)

```
┌─────────────────────────────────────────────────────┐
│  EC2 인스턴스 (96GB 디스크)                           │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │ vibe-backend │  │ vibe-frontend│                  │
│  │ (API+스케줄러)│  │ (Nginx+Vue)  │                  │
│  │ Port 8080    │  │ Port 80/443  │                  │
│  └──────┬───────┘  └──────────────┘                  │
│         │                                            │
│         │ docker.sock (형제 컨테이너)                   │
│         ▼                                            │
│  ┌─────────────────────────────────┐                 │
│  │ crawler-{session_id} 컨테이너들   │                 │
│  │ (Playwright 기반, 세션별 1개)     │                 │
│  └─────────────────────────────────┘                 │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │ PostgreSQL   │  │ system_storage│                  │
│  │ Port 5432    │  │ (공유 볼륨)    │                  │
│  └──────────────┘  └──────────────┘                  │
└─────────────────────────────────────────────────────┘
```

### 1.2 현재 배치(스케줄러) 동작 방식

| 구성 요소 | 현재 방식 | 문제점 |
|---|---|---|
| **스케줄러** | APScheduler (인메모리, 단일 프로세스) | 서버 2대 이상이면 **중복 실행** 발생 |
| **PENDING 세션 픽업** | 15초 간격 폴링 → `localhost:6767` API 호출 | 자기 자신만 호출 가능, **타 서버로 분배 불가** |
| **동시 실행 제한** | `max_concurrent_jobs` (DB 기반 카운트) | DB 레벨 카운트는 공유 가능하나, **슬롯 선점 경합** 있음 |
| **Job 락** | `acquire_lock()` — SQL UPDATE WHERE status≠RUNNING | **DB 레벨 Atomic**이므로 멀티서버에서도 안전 ✅ |
| **크롤러 실행** | Docker 형제 컨테이너 (`docker.sock` 마운트) | 해당 호스트의 Docker만 제어 가능 |
| **스토리지** | `/app/system_storage` (호스트 디렉토리 마운트) | **각 서버 로컬 디스크에 저장**, 서버 간 공유 안 됨 |

### 1.3 현재 병목 지점

| 병목 | 현재 수치 | 영향 |
|---|---|---|
| 동시 크롤링 수 | `max_concurrent_jobs = 3` | 프로젝트 20+개 시 대기 시간 증가 |
| 디스크 | 96GB (일당 4~16GB 소모) | 보관 기간 10일 이상 불가 |
| CPU/메모리 | Playwright + Chromium 세션당 1~2GB RAM | 동시 3개 = 최소 6GB RAM 필요 |

---

## 2. 증량 방안 비교

### 방안 A: 단일 서버 스케일업 (Scale-Up)

```
기존 EC2 → 더 큰 인스턴스 + 더 큰 디스크
```

| 항목 | 변경 |
|---|---|
| 인스턴스 | t3.medium → t3.xlarge (4vCPU, 16GB RAM) |
| 디스크 | 96GB → 200GB+ EBS |
| `max_concurrent_jobs` | 3 → 5~8 |
| 코드 변경 | **없음** |

- **장점**: 코드 수정 0, 즉시 적용 가능
- **단점**: 수직 확장 한계, 단일 장애점(SPOF) 해결 안 됨

### 방안 B: Worker 서버 분리 — Scale-Out (권장)

현재는 **"사장 + 직원"이 한 사무실에서 모든 일을 같이 하는** 구조입니다:

```
[현재: 1대가 다 함]
사장(스케줄러): "5시에 크롤링 해야지!" → 직접 크롤러도 실행
                                     → 직접 API 응답도 처리
                                     → 직접 DB 관리도 수행
```

**방안 B**는 "사장"과 "직원"의 역할을 분리하는 것입니다:

```
[방안 B: 역할 분리]
Master 서버(사장): "5시다! → Worker야 이거 해!"
    ↓ HTTP 호출
Worker 서버(직원): "네! 크롤러 돌리겠습니다" → Docker 컨테이너 실행
```

- **Master(사장)**: 스케줄 관리 + API 응답만 담당 (크롤러 직접 안 돌림)
- **Worker(직원)**: 크롤러만 전담 실행 (스케줄러 없음)
- Worker를 **여러 대 추가**하면 → 직원이 늘어나면 → 동시에 더 많은 크롤링 가능
- **사장이 1명**이므로 → 배치 선점 문제가 **발생하지 않음** ✅

아키텍처 다이어그램:

```
┌───────────────────┐     ┌───────────────────┐
│  Master 서버       │     │  Worker 서버 (N대)  │
│ (API + 스케줄러)    │     │ (크롤러 전용)        │
│                   │     │                    │
│ ┌───────────────┐ │     │ ┌────────────────┐ │
│ │ Backend API   │─┼─────┼▶│ Worker Agent   │ │
│ │ + APScheduler │ │     │ │ (크롤러 실행)    │ │
│ └───────────────┘ │     │ └────────────────┘ │
│ ┌───────────────┐ │     │ ┌────────────────┐ │
│ │ PostgreSQL    │◀┼─────┼─│ DB 접근 (직접)   │ │
│ └───────────────┘ │     │ └────────────────┘ │
│ ┌───────────────┐ │     │ ┌────────────────┐ │
│ │ EFS/NFS       │◀┼─────┼─│ 공유 스토리지     │ │
│ └───────────────┘ │     │ └────────────────┘ │
└───────────────────┘     └───────────────────┘
```

- **장점**: 수평 확장 가능, Master 장애 시에도 실행 중인 크롤링은 계속
- **단점**: 코드 변경 필요, 공유 스토리지 비용

### 방안 C: 멀티 Master (Active-Active) — Scale-Out

방안 B와 달리 **두 서버가 둘 다 "사장" 역할**을 합니다:

```
[방안 C: 둘 다 사장]
서버A(사장): "5시다! 크롤링 하자!"  ──┐
                                    ├── 같은 일을 동시에 하려고 함!
서버B(사장): "5시다! 크롤링 하자!"  ──┘
```

사장이 2명이니까 둘 다 같은 PENDING 세션을 **동시에 가져가려는 경쟁**이 발생합니다.
이게 바로 **배치 선점 문제**이며, 반드시 해결해야 방안 C를 사용할 수 있습니다.

```
┌───────────────┐  ┌───────────────┐
│ Master A      │  │ Master B      │
│ API+스케줄러   │  │ API+스케줄러   │
│ +크롤러       │  │ +크롤러       │
└───────┬───────┘  └───────┬───────┘
        │    공유 DB/EFS    │
        └────────┬─────────┘
           ┌─────┴─────┐
           │ PostgreSQL │
           │ + EFS/NFS  │
           └───────────┘
```

- **장점**: 고가용성(HA), 부하 분산
- **단점**: 배치 선점 문제 해결이 **필수**, 코드 변경 범위 큼

---

## 3. 배치 선점 문제 상세 분석

### 3.1 문제 정의

**배치 선점 문제란?**  
서버가 2대 이상일 때, 각 서버가 "내가 이 일을 하겠다!"고 **동시에 손을 들어서** 같은 작업을 중복 실행하는 문제입니다.

쉬운 비유로 설명하면:

```
[1대일 때 — 문제 없음]
사장 1명: "대기열에 3건 있네? 1번부터 순서대로 처리하자"
→ 경쟁자가 없으니 중복 실행 불가능

[2대일 때 — 문제 발생!]
사장A: "대기열에 3건 있네! 1번 내가 한다!" ──┐
                                           ├── 1번을 둘 다 처리해버림!
사장B: "대기열에 3건 있네! 1번 내가 한다!" ──┘
```

현재 APScheduler는 각 서버에서 **독립적으로** 15초마다 PENDING 세션을 조회하므로:

```
[서버A] 15초마다: PENDING 세션 조회 → 3개 발견 → 3개 다 실행 시도
[서버B] 15초마다: PENDING 세션 조회 → 3개 발견 → 3개 다 실행 시도
                                            ↓
                                     동일 세션 중복 실행!
```

### 3.2 현재 방어선과 한계

| 방어선 | 위치 | 효과 |
|---|---|---|
| `acquire_lock()` | `job_repository_adapter.py:346` | Job 레벨에서 Atomic UPDATE → **같은 Job의 중복 실행은 방지됨** ✅ |
| `Job당 1개 세션 제한` | `check_and_execute_pending_sessions_service.py:66-73` | RUNNING 세션 있으면 스킵 → **같은 Job 내 세션 중복은 방지** ✅ |
| `max_concurrent 카운트` | DB `count_running_jobs()` | 전체 시스템 동시 실행 제한 → **DB 공유 시 유효** ✅ |
| **PENDING→RUNNING 세션 전이** | API를 통해 처리 | **localhost 호출이라 자기 서버에서만 실행** ❌ |

### 3.3 핵심 문제: 세션 선점 경쟁 (Race Condition)

현재 코드에서 문제가 발생하는 정확한 지점:

```python
# check_and_execute_pending_sessions_service.py 현재 로직

# ① 15초마다 실행 — 서버A, B 모두 같은 PENDING 목록을 봄
pending_sessions = await session_repo.find_pending_sessions_oldest_per_job(limit=available_slots)

# ② for문으로 하나씩 실행 — 서버A가 세션-1을 잡으려는 순간 B도 잡으려 함
for session in pending_sessions:
    await crawling_executor.execute_crawling(session.job_id, pending_session_id=str(session.id))

# ③ 결과: 같은 세션인데 크롤러 컨테이너가 2개 뜸! 💥
```

`acquire_lock()`이 Job 레벨에서 방어하지만, **같은 세션이 두 서버에서 중복으로 크롤러 컨테이너를 띄우는 순간이 존재**합니다.

### 3.4 해결 방법: "먼저 손 든 사람이 가져감" (Atomic UPDATE)

DB의 Atomic UPDATE를 사용하면 **동시에 손을 들어도 딱 1명만 성공**합니다:

```sql
-- 서버A가 먼저 이 쿼리를 실행하면:
UPDATE sessions SET status='RUNNING', worker='서버A'
WHERE id='세션-1' AND status='PENDING'
-- → rowcount=1 (성공! 세션-1은 서버A가 선점)

-- 서버B가 직후에 동일한 쿼리를 실행하면:
UPDATE sessions SET status='RUNNING', worker='서버B'
WHERE id='세션-1' AND status='PENDING'
-- → rowcount=0 (이미 RUNNING이라 WHERE 조건 불일치 → 실패, 건너뜀)
```

이 패턴은 이미 프로젝트에서 `acquire_lock()`(Job 레벨)으로 사용 중이며,
이것을 **세션 레벨에도 동일하게 적용**하는 것이 `acquire_session_lock()` 입니다.

---

## 4. 권장 증량 방안: 방안 B (Worker 분리)

### 4.1 아키텍처

```
┌──────────────────────────────────────────────────────┐
│  Master EC2 (기존 서버)                                │
│                                                       │
│  ┌─────────────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Backend API     │  │ Frontend │  │ PostgreSQL   │ │
│  │ + APScheduler   │  │          │  │              │ │
│  │ (스케줄러 전담)   │  │          │  │              │ │
│  └────────┬────────┘  └──────────┘  └──────┬───────┘ │
│           │                                │         │
│           │ ①PENDING 세션 생성              │         │
│           │ ②Worker에 실행 요청             │         │
│           ▼                                │         │
│  ┌─────────────────┐                       │         │
│  │ EFS Mount       │◄─────────────────────┐│         │
│  │ /system_storage │                      ││         │
│  └─────────────────┘                      ││         │
└───────────────────────────────────────────┼┼─────────┘
                                            ││
            ┌───────────────────────────────┼┼─────────┐
            │  Worker EC2-1                 ││         │
            │                               ││         │
            │  ┌──────────────────┐         ││         │
            │  │ Worker Agent     │─────────┘│         │
            │  │ (FastAPI Lite)   │          │         │
            │  │ Port 6767        │          │         │
            │  └────────┬────────┘          │         │
            │           │ docker.sock        │         │
            │           ▼                    │         │
            │  ┌──────────────────┐          │         │
            │  │ crawler-{sid}    │          │         │
            │  │ 컨테이너들         │          │         │
            │  └──────────────────┘          │         │
            │  ┌──────────────────┐          │         │
            │  │ EFS Mount        │──────────┘         │
            │  │ /system_storage  │                    │
            │  └──────────────────┘                    │
            └──────────────────────────────────────────┘
```

### 4.2 필요한 인프라 변경

| 항목 | 현재 | 변경 후 |
|---|---|---|
| **EC2** | 1대 (All-in-One) | Master 1대 + Worker N대 |
| **스토리지** | 호스트 로컬 디스크 | **AWS EFS** (NFS 공유 볼륨) |
| **DB** | Docker Compose 내 PostgreSQL | **AWS RDS PostgreSQL** 또는 기존 유지 (Worker에서 TCP 접속) |
| **네트워크** | 단일 호스트 | **VPC 내 Private Subnet** (Master↔Worker 통신) |

### 4.3 코드 변경 사항

#### 변경 1: 세션 선점 락 추가 (배치 선점 해결)

```python
# crawling_session_repository_adapter.py — 새 메서드 추가
async def acquire_session_lock(self, session_id: UUID, worker_id: str) -> bool:
    """
    PENDING 세션을 Atomic하게 선점 (SELECT FOR UPDATE SKIP LOCKED)
    """
    stmt = (
        update(CrawlingSessionModel)
        .where(
            and_(
                CrawlingSessionModel.id == str(session_id),
                CrawlingSessionModel.status == CrawlingStatus.PENDING.value
            )
        )
        .values(
            status=CrawlingStatus.RUNNING.value,
            process_identifier=worker_id,
            updated_at=datetime.utcnow()
        )
    )
    result = await self.db.execute(stmt)
    await self.db.commit()
    return result.rowcount > 0
```

핵심 원리: 기존 `acquire_lock()`과 동일한 **Atomic UPDATE** 패턴을 세션 레벨에 적용합니다.

#### 변경 2: CrawlingExecutorAdapter — Worker 라우팅

```python
# crawling_executor_adapter.py 수정
class CrawlingExecutorAdapter(CrawlingExecutorPort):
    async def execute_crawling(self, job_id: UUID, pending_session_id: str = None):
        # 기존: localhost:6767 호출
        # 변경: Worker 풀에서 가용 Worker 선택 후 호출
        worker_url = await self._select_available_worker()
        url = f"http://{worker_url}/api/v1/jobs/{str(job_id)}/execute"
        ...
```

#### 변경 3: Worker Agent 서비스 (신규)

Worker EC2에서 실행되는 경량 FastAPI 서버:
- 크롤링 실행 API만 제공 (`/api/v1/jobs/{id}/execute`)
- 스케줄러 **없음** (Master만 스케줄러 보유)
- DB 직접 접속 (세션 상태 업데이트)
- Docker 소켓으로 크롤러 컨테이너 실행

#### 변경 4: Docker Compose 분리

```yaml
# docker-compose.master.yml — Master 전용
services:
  backend:
    environment:
      ROLE: master        # 스케줄러 활성화
      WORKER_URLS: "worker1:6767,worker2:6767"

# docker-compose.worker.yml — Worker 전용
services:
  worker:
    environment:
      ROLE: worker        # 스케줄러 비활성화
      DATABASE_URL: postgresql+asyncpg://...@master-ip:5432/...
    volumes:
      - /efs/system_storage:/app/system_storage
      - /var/run/docker.sock:/var/run/docker.sock
```

---

## 5. 배치 선점 해결 상세 설계

### 5.1 세션 실행 흐름 (변경 후)

```
[Master] APScheduler (15초 간격)
    │
    ▼
[Master] CheckAndExecutePendingSessionsService
    │
    ├── 1. DB에서 PENDING 세션 목록 조회
    ├── 2. 세션별로 acquire_session_lock() 호출 (Atomic UPDATE)
    │       └── 성공하면 해당 세션 선점
    ├── 3. 선점된 세션 → Worker 라우팅
    │       └── Round-Robin 또는 부하 기반 선택
    └── 4. Worker API 호출 (HTTP)
            └── Worker가 Docker 컨테이너 실행
```

### 5.2 중복 방지 계층

| 계층 | 메커니즘 | 방어 대상 |
|---|---|---|
| **L1: 세션 선점** | `acquire_session_lock()` Atomic UPDATE | 동일 세션 중복 실행 |
| **L2: Job 락** | `acquire_lock()` (기존) | 동일 Job 중복 실행 |
| **L3: 동시 실행 제한** | `max_concurrent_jobs` DB 카운트 | 전체 시스템 과부하 |
| **L4: Job당 1세션** | RUNNING 세션 체크 (기존) | Job 내 세션 중복 |

### 5.3 Worker 장애 대응

| 상황 | 감지 | 처리 |
|---|---|---|
| Worker API 응답 없음 | Master HTTP 타임아웃 (10초) | 다른 Worker로 재시도 |
| Worker 크래시 (세션 실행 중) | `sync_job_status_runner` (30초) | DB에서 RUNNING인데 컨테이너 없음 → FAILED |
| Worker 디스크 풀 | Worker 헬스체크 실패 | 풀에서 제외, EFS이므로 Master에서도 데이터 접근 가능 |

---

## 6. AWS 인프라 구성

### 6.1 리소스 목록

| 리소스 | 스펙 | 용도 | 예상 월 비용 |
|---|---|---|---|
| **Master EC2** | t3.medium (2vCPU, 4GB) | API + 스케줄러 + DB | 기존 유지 |
| **Worker EC2 × 1** | t3.large (2vCPU, 8GB) | 크롤러 실행 전용 | ~$60 |
| **AWS EFS** | General Purpose | 공유 스토리지 | ~$30/100GB |
| **VPC/SG** | Private Subnet | 내부 통신 | 무료 |

### 6.2 디스크 용량 해결

| 항목 | 현재 | EFS 전환 후 |
|---|---|---|
| 용량 제한 | 96GB (EBS 고정) | **무제한** (사용량 기반 과금) |
| 서버 간 공유 | ❌ 불가 | ✅ NFS 마운트 |
| 자동 백업 | 수동 | EFS Lifecycle 정책 |
| 비용 | EBS $10/100GB | EFS $30/100GB (IA 클래스: $2.5/100GB) |

### 6.3 네트워크 구성

```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24)
│   └── Master EC2 (EIP: 기존 IP)
│       └── SG: 80,443(외부), 8080(외부), 5432(Private만)
│
├── Private Subnet (10.0.2.0/24)
│   └── Worker EC2-1
│       └── SG: 6767(Master만), 5432→Master
│
└── EFS Mount Target
    └── SG: 2049(NFS, VPC 내부만)
```

---

## 7. 단계별 실행 계획

### Phase 1: 준비 (코드 변경 없이 즉시 가능)

- [ ] Master EC2 인스턴스 스케일업 (t3.medium → t3.large)
- [ ] EBS 디스크 증량 (96GB → 200GB)
- [ ] `max_concurrent_jobs` 조정 (3 → 5)
- [ ] 소요: 1일, 코드 변경 없음

### Phase 2: 공유 스토리지 전환

- [ ] AWS EFS 생성 + Master에 마운트
- [ ] 기존 system_storage 데이터 EFS로 마이그레이션
- [ ] Docker Compose `volumes` EFS 경로로 변경
- [ ] 소요: 2~3일

### Phase 3: 세션 선점 락 구현

- [ ] `acquire_session_lock()` 메서드 구현
- [ ] `CheckAndExecutePendingSessionsService`에 선점 로직 적용
- [ ] 유닛테스트 작성 (경쟁 조건 시뮬레이션)
- [ ] 소요: 2~3일

### Phase 4: Worker 서버 배포

- [ ] Worker EC2 프로비저닝 (Private Subnet)
- [ ] Worker Agent 서비스 구현 (크롤링 실행 API만)
- [ ] `CrawlingExecutorAdapter` Worker 라우팅 로직
- [ ] Master↔Worker 통신 테스트
- [ ] 소요: 3~5일

### Phase 5: 운영 전환 및 검증

- [ ] 스테이징 환경에서 멀티 서버 통합 테스트
- [ ] 배치 선점 경합 테스트 (동시 다발 PENDING 생성)
- [ ] Worker 장애 시뮬레이션 (kill 후 복구 확인)
- [ ] 운영 환경 롤아웃
- [ ] 소요: 2~3일

---

## 8. 빠른 효과를 위한 우선 적용 (Phase 1만)

Phase 2~5는 개발 기간이 필요하므로, **즉시 효과**가 필요하면 Phase 1만 먼저 적용합니다:

```bash
# EC2 인스턴스 타입 변경 (다운타임 5분)
aws ec2 stop-instances --instance-ids i-xxxxx
aws ec2 modify-instance-attribute --instance-id i-xxxxx --instance-type t3.xlarge
aws ec2 start-instances --instance-ids i-xxxxx

# EBS 볼륨 확장 (무중단)
aws ec2 modify-volume --volume-id vol-xxxxx --size 200
# EC2 내부에서:
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

이것만으로도 동시 크롤링 수를 3 → 8개까지 늘릴 수 있고, 디스크 여유도 확보됩니다.

---

## 9. 리스크 및 고려사항

| 리스크 | 영향 | 완화 방안 |
|---|---|---|
| EFS 네트워크 레이턴시 | 대용량 스크린샷 저장 속도 저하 | EFS Provisioned Throughput 또는 세션별 로컬 캐시 후 비동기 동기화 |
| Worker 장애 시 세션 유실 | RUNNING 세션이 FAILED로 전환되지 않을 수 있음 | 기존 `sync_job_status_runner`로 감지 (30초 주기) |
| RDS 비용 증가 | 외부 DB 사용 시 월 $15~50 추가 | 초기에는 Master의 PostgreSQL Docker를 Worker에서 TCP 접속으로 공유 |
| 보안 | Worker가 Master DB에 직접 접속 | VPC Security Group으로 Worker IP만 허용 |
