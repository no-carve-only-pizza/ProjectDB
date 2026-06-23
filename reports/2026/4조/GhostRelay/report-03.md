---
project_name: "GhostRelay"
quad_name: "4조"
members:
  - "20233051_박도현"
  - "20253311_강수빈"
  - "20261717_이정훈"
  - "20241947_장수연"
report_number: 3
date: "2026-06-23"
status: "진행 중"
cl_level: "CL1"
contributions:
  - name: "20233051_박도현"
    role: "에이전트 개발 (시스템) / 팀장"
    tasks: "2차 보고서 이후 EDR Agent 탐지 구조 고도화, 이벤트 출력 포맷 및 백엔드 연동 구조 공유, 대시보드 연동 검증 지원"
    percentage: 10
  - name: "20253311_강수빈"
    role: "공격 시나리오 설계 / 보안 검증"
    tasks: "EDR 탐지 검증용 공격 시나리오 구성, 데모 흐름 정리, 대시보드에서 확인할 이벤트 유형 검토"
    percentage: 10
  - name: "20261717_이정훈"
    role: "백엔드 / 프론트엔드 연동"
    tasks: "백엔드 API 구성 확인, Swagger 기반 엔드포인트 검토, 프론트엔드 API 호출 구조 및 Vercel 프록시 연동 작업"
    percentage: 40
  - name: "20241947_장수연"
    role: "프론트엔드 / 백엔드 연동"
    tasks: "대시보드 API 연동, 이벤트 목록 및 상세 데이터 렌더링, 배포 환경 검증 및 CORS 문제 해결"
    percentage: 40
---

# [제 3차 프로젝트 진행 보고서] GhostRelay

- **팀원:** (팀장) 20233051_박도현, (팀원) 20253311_강수빈, 20261717_이정훈, 20241947_장수연
- **활동 기간:** 2026. 05. 20. ~ 2026. 06. 23. (약 5주)

---

## 팀 전체 진행 현황

- **이번 회차 목표:** 2차 보고서에서 구현한 eBPF 기반 EDR Agent를 확장하여 탐지 범위를 넓히고, 실제 웹 대시보드에서 백엔드 API를 통해 EDR 이벤트를 조회할 수 있도록 프론트엔드와 백엔드를 연동한다.
- **현재 진행률:** 약 80% (전체 일정 대비)
- **주요 달성 사항:**
  - EDR Agent의 이벤트 출력 포맷 및 백엔드 연동 방향 정리
  - 백엔드 Swagger 문서를 통한 API 구조 확인
  - 프론트엔드 대시보드에서 백엔드 상태 확인 API(`/health`) 연동
  - 이벤트 목록 조회 API(`/api/v1/events`) 연동
  - Vercel rewrite 기반 프록시 설정으로 CORS 문제 해결
  - 실제 배포 환경에서 이벤트 데이터 수신 및 렌더링 확인
  - 공격 시나리오 기반 데모 흐름 정리
  - 향후 EDR Agent -> 백엔드 -> 대시보드 전체 파이프라인 검증을 위한 기반 마련

| 구분 | 2차 보고서 기준 | 이번 회차 결과 |
|------|----------------|----------------|
| 탐지 엔진 | R-001~R-015 구현 | 확장 탐지 및 대시보드 연동 가능 구조 정리 |
| 이벤트 출력 | stdout / NDJSON / HTTP POST 구조 | 백엔드 API 수신 구조와 연동 |
| 검증 방식 | 에이전트 단독 데모 중심 | 공격 시나리오 + 웹 대시보드 확인 흐름으로 확장 |
| 대시보드 | API 포맷 협의 예정 | 실제 배포 환경에서 이벤트 목록 렌더링 성공 |

---

## 개인별 기여 내역

| 팀원 | 역할 | 수행 작업 | 산출물 링크 | 기여도 |
|------|------|----------|------------|--------|
| 20233051_박도현 | 에이전트 개발 / 팀장 | EDR Agent 시스템 구조 고도화, 이벤트 출력 포맷 및 대시보드 연동 구조 공유, 연동 검증 지원 | [edr-agent](https://github.com/no-carve-only-pizza/edr-agent) | 10% |
| 20253311_강수빈 | 공격 시나리오 설계 / 보안 검증 | EDR 탐지 검증용 공격 시나리오 작성, 데모 흐름 구성, 대시보드에서 확인할 이벤트 유형 검토 | [edr-agent demo](https://github.com/no-carve-only-pizza/edr-agent/tree/main/demo) | 10% |
| 20261717_이정훈 | 백엔드 / 프론트엔드 연동 | 백엔드 Swagger API 확인, 이벤트 API 연동, Vercel 프록시 설정, 배포 환경 검증 | [배포 대시보드](https://edr-agent.vercel.app) | 40% |
| 20241947_장수연 | 프론트엔드 / 백엔드 연동 | 대시보드 API 호출 구조 구현, 이벤트 데이터 정규화, 이벤트 목록 렌더링 및 CORS 문제 해결 | [배포 대시보드](https://edr-agent.vercel.app) | 40% |

---

## 이번 회차 상세 진행 내용

### 1. 2차 보고서 이후 프로젝트 흐름

2차 보고서에서는 기존 CVE 분석 중심의 프로젝트를 리눅스 엔드포인트 탐지·대응 중심의 EDR Agent 개발로 전환하였다. 해당 회차에서 프로세스·파일·네트워크 이벤트 수집, R-001~R-015 탐지 룰, NDJSON 출력 포맷, HTTP POST 기반 외부 연동 구조를 구현하였다.

이번 회차에서는 이 흐름을 이어받아 다음 단계인 **웹 대시보드 연동**에 집중하였다. 기존 에이전트가 수집·탐지한 이벤트를 사람이 확인할 수 있는 형태로 제공하기 위해, 백엔드 API와 프론트엔드 대시보드 사이의 실제 데이터 흐름을 연결하였다.

전체 구조는 다음과 같다.

```text
EDR Agent
  -> NDJSON / HTTP POST 이벤트 출력
  -> 백엔드 API 저장 및 조회
  -> 프론트엔드 대시보드 렌더링
```

이를 통해 프로젝트는 단순한 터미널 기반 탐지 도구에서, 웹 콘솔을 통해 이벤트를 확인할 수 있는 EDR PoC 형태로 확장되었다.

---

### 2. 백엔드 API 구조 확인

백엔드 서버는 Swagger 문서를 통해 API 구조를 제공하였다.

```text
https://asc4.jeonghuncompy.cloud/docs
```

Swagger 문서를 통해 확인한 주요 엔드포인트는 다음과 같다.

| Method | Endpoint | 용도 |
|--------|----------|------|
| GET | `/health` | 백엔드 서버 상태 확인 |
| GET | `/api/v1/events` | EDR 이벤트 목록 조회 |
| POST | `/api/v1/events` | 단일 이벤트 저장 |
| POST | `/api/v1/events/batch` | 이벤트 배치 저장 |
| POST | `/api/v1/events/ndjson` | NDJSON 형식 이벤트 수집 |
| GET | `/api/v1/events/stream` | 이벤트 스트림 조회 |
| GET | `/api/v1/events/{event_id}` | 이벤트 상세 조회 |
| GET | `/api/v1/alerts` | 알림 목록 조회 |
| GET | `/api/v1/alerts/summary` | 알림 요약 정보 조회 |

프론트엔드에서는 우선 `/health`를 통해 백엔드 연결 상태를 확인하고, `/api/v1/events`를 통해 저장된 EDR 이벤트 목록을 조회하도록 구성하였다.

---

### 3. 프론트엔드 API 요청 구조 구현

API 요청 로직은 공통 함수로 분리하였다. 이를 통해 API base URL을 코드에 직접 고정하지 않고, 환경변수 기반으로 관리할 수 있도록 하였다.

```jsx
const API_BASE_URL =
  import.meta.env.VITE_EDR_API_BASE_URL?.replace(/\/$/, "") || "/edr-api";

async function apiRequest(path, token, options = {}) {
  const headers = new Headers(options.headers);

  if (token) {
    headers.set("Authorization", `Bearer ${token}`);
  }

  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    headers,
  });

  if (!response.ok) {
    let message = `${response.status} ${response.statusText}`;

    try {
      const body = await response.json();
      message = body.detail || message;
    } catch {
      // JSON 오류 응답이 아닐 수 있으므로 기본 메시지를 사용한다.
    }

    throw new Error(message);
  }

  return response.json();
}
```

해당 구조를 통해 이벤트 조회, 알림 조회, 이벤트 상세 확인 기능을 동일한 방식으로 확장할 수 있도록 하였다.

---

### 4. Vercel 환경변수 및 프록시 설정

배포 환경에서는 브라우저가 백엔드 서버를 직접 호출하지 않고, Vercel rewrite를 통해 프록시 경로로 접근하도록 구성하였다.

설정한 환경변수는 다음과 같다.

```text
VITE_EDR_API_BASE_URL=/edr-api
EDR_API_PROXY_TARGET=https://asc4.jeonghuncompy.cloud
```

최종 요청 흐름은 다음과 같다.

```text
프론트엔드 요청
https://edr-agent.vercel.app/edr-api/api/v1/events

Vercel rewrite

백엔드 API
https://asc4.jeonghuncompy.cloud/api/v1/events
```

이를 통해 프론트엔드 배포 도메인과 백엔드 API 서버 간의 CORS 문제를 해결하였다.

---

### 5. 이벤트 데이터 정규화 및 렌더링

백엔드에서 수신한 이벤트 목록은 `items` 배열 형태로 확인되었다. 프론트엔드에서는 해당 데이터를 대시보드에서 사용하기 쉬운 구조로 변환하기 위해 `normalizeRecord` 로직을 적용하였다.

이를 통해 백엔드 이벤트 데이터가 프론트엔드 컴포넌트에서 요구하는 형태로 변환되었으며, 실제 브라우저 화면에서 이벤트 목록이 정상적으로 표시되는 것을 확인하였다.

검증한 항목은 다음과 같다.

| 검증 항목 | 결과 |
|----------|------|
| Vercel Production 재배포 | 성공 |
| `/health` API 요청 | 성공 |
| 백엔드 상태 온라인 표시 | 성공 |
| `/api/v1/events` API 요청 | 성공 |
| 이벤트 데이터 `items` 배열 수신 | 성공 |
| 이벤트 데이터 정규화 | 성공 |
| 브라우저 화면 이벤트 렌더링 | 성공 |
| CORS 오류 해결 | 성공 |

최종 배포 주소는 다음과 같다.

```text
https://edr-agent.vercel.app
```

---

### 6. 공격 시나리오 기반 검증 준비

강수빈 팀원은 EDR Agent의 탐지 결과를 실제 대시보드에서 확인할 수 있도록 공격 시나리오 흐름을 구성하였다. 이 작업은 단순히 개별 룰이 발화하는지 확인하는 수준을 넘어서, 발표 및 최종 검증 단계에서 사용할 수 있는 재현 가능한 데모 흐름을 만드는 것을 목표로 하였다.

공격 시나리오의 주요 범위는 다음과 같다.

| 시나리오 | 관련 탐지 |
|----------|-----------|
| LD_PRELOAD 인젝션 | R-013 |
| memfd_create 기반 파일리스 공격 | R-017 |
| 백도어 포트 바인드 및 비표준 포트 연결 | R-004, R-006, R-007 |
| ptrace ATTACH 시도 | R-014 |
| DNS 터널링 / DGA / 남용 TLD | R-018 |
| 민감 경로 파일 수정 | R-001 |
| 로그 파일 삭제 | R-002 |
| `/tmp` 경로 실행 | R-003 |
| 인터프리터 인라인 페이로드 | R-005, R-010 |
| `/tmp`에서 시스템 경로로 rename | R-009 |
| 예상치 못한 setuid 실행 | R-015 |
| `/tmp` 쓰기 -> 실행 -> 아웃바운드 연결 체인 | R-020, R-021 |

이 시나리오들은 이후 EDR Agent가 백엔드로 이벤트를 전송하고, 대시보드가 이를 실시간 또는 목록 형태로 표시하는 전체 흐름을 검증하는 데 사용될 예정이다.

---

## 이슈 및 해결 방안

| 이슈 | 해결 방법 |
|------|----------|
| 배포된 프론트엔드에서 백엔드 API 직접 호출 시 CORS 문제가 발생할 수 있음 | Vercel rewrite를 이용해 `/edr-api` 프록시 경로를 구성 |
| API 주소를 코드에 직접 고정하면 배포 환경 변경에 취약함 | `VITE_EDR_API_BASE_URL`, `EDR_API_PROXY_TARGET` 환경변수로 분리 |
| 백엔드 이벤트 응답 구조와 프론트엔드 표시 구조가 다름 | `normalizeRecord`를 통해 이벤트 데이터를 화면 표시용 구조로 변환 |
| EDR Agent 단독 실행 결과와 웹 대시보드 표시 결과를 연결해 검증해야 함 | 공격 시나리오를 기반으로 Agent -> Backend -> Dashboard 전체 파이프라인 검증 계획 수립 |

---

## 다음 회차 목표

| 항목 | 내용 |
|------|------|
| 전체 파이프라인 검증 | EDR Agent에서 백엔드로 NDJSON 이벤트를 직접 전송하고 대시보드에서 조회 |
| 알림 대시보드 확장 | `/api/v1/alerts`, `/api/v1/alerts/summary` 연동 |
| 이벤트 상세 화면 | `/api/v1/events/{event_id}` 기반 상세 페이지 또는 모달 구현 |
| 필터링 및 검색 | severity, rule_id, event type, process name 기준 필터 추가 |
| 발표 데모 구성 | 공격 시나리오 실행 후 대시보드에서 탐지 이벤트 확인하는 흐름 정리 |
| Active Response 검토 | 에이전트의 kill 명령 기능을 대시보드 UI와 연결할 수 있는지 검토 |

---

## 참고 자료

- 소스코드: https://github.com/no-carve-only-pizza/edr-agent
- ProjectDB: https://github.com/no-carve-only-pizza/ProjectDB
- 배포 대시보드: https://edr-agent.vercel.app
- 백엔드 Swagger 문서: https://asc4.jeonghuncompy.cloud/docs
