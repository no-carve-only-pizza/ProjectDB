---
project_name: "GhostRelay"
quad_name: "4조"
members:
  - "20233051_박도현"
  - "20253311_강수빈"
  - "20261717_이정훈"
  - "20241947_장수연"
report_number: 6
date: "2026-08-04"
status: "진행 중"
cl_level: "CL1"
contributions:
  - name: "20233051_박도현"
    role: "에이전트 개발 (시스템) / 팀장"
    tasks: "백엔드·프론트 최종본 수령 및 아키텍처 문서 검토, 8월 발표 준비 착수"
    percentage: 25
  - name: "20253311_강수빈"
    role: "공격 시나리오 설계 / 보안 검증"
    tasks: "라이브 대시보드 탐지 결과 검토, 8월 발표 준비 착수"
    percentage: 25
  - name: "20261717_이정훈"
    role: "백엔드 / API 연동"
    tasks: "FastAPI 백엔드 최종 구현(수집·저장·SSE 실시간·인증), 프론트 연동 트러블슈팅"
    percentage: 25
  - name: "20241947_장수연"
    role: "프론트엔드 / 대시보드 UX"
    tasks: "React 대시보드 최종 구현, Cloudflare Pages 배포 및 환경변수·캐시 트러블슈팅"
    percentage: 25
---

# [제 6차 프로젝트 진행 보고서] GhostRelay

- **팀원:** (팀장) 20233051_박도현, (팀원) 20253311_강수빈, 20261717_이정훈, 20241947_장수연
- **활동 기간:** 2026. 07. 19. ~ 2026. 08. 04. (약 2.5주)

---

## 팀 전체 진행 현황

- **이번 회차 목표:** 5차 보고서에서 확정한 분담대로 백엔드·프론트엔드 연동을 완성하고, 8월 발표 준비를 시작한다.
- **현재 진행률:** 약 95% (구현 사실상 완료, 발표 준비 단계 진입)
- **주요 달성 사항:**
  - **백엔드 + 프론트엔드 최종본을 완성해 전달** — 구현 산출물 기준 사실상 최종 버전
  - `edr_backend`(FastAPI + SQLite WAL) ↔ React 대시보드 연동 아키텍처 문서(`ARCHITECTURE.md`) 인계
  - 라이브 배포 환경(`ebpf-agent.com` ↔ `asc4.jeonghuncompy.cloud`)에서 실시간 이벤트·알림 동작을 스크린샷으로 확인
  - Cloudflare Pages 배포 특이사항(프록시 미지원)으로 인한 연동 오류를 원인 규명 후 해결
  - 이후 작업은 **발표 준비**로 전환

| 구분          | 5차 보고서 기준         | 이번 회차 결과                                           |
| ----------- | ----------------- | -------------------------------------------------- |
| 벤치마크        | 박도현 담당, 항목·방법 미정  | 발표 준비 단계에서 진행 예정(다음 회차)                            |
| 프론트·데모 flow | 이정훈·장수연이 완성 예정    | **완성 및 전달 완료** — Agent→Backend→Dashboard 흐름 라이브 확인 |
| 발표자료        | 이정훈·장수연이 초안 준비 예정 | 데모 화면·아키텍처 자료 확보, 슬라이드화는 다음 단계                     |
| 발표 준비       | 도현·수빈이 후속 진행      | **착수**                                             |

---

## 개인별 기여 내역

| 팀원 | 역할 | 수행 작업 | 산출물 링크 | 기여도 |
|------|------|----------|------------|--------|
| 20233051_박도현 | 에이전트 개발 / 팀장 | 백엔드·프론트 최종본 및 아키텍처 문서 수령·검토, 발표 준비 착수 | [edr-agent](https://github.com/no-carve-only-pizza/edr-agent) | 25% |
| 20253311_강수빈 | 공격 시나리오 설계 / 보안 검증 | 라이브 대시보드 탐지 결과 검토, 발표 준비 착수 | [edr-agent demo](https://github.com/no-carve-only-pizza/edr-agent/tree/main/demo) | 25% |
| 20261717_이정훈 | 백엔드 / API 연동 | FastAPI 수집·저장·SSE 실시간 스트림·인증 최종 구현, 프론트 연동 트러블슈팅 | [백엔드 Swagger 문서](https://asc4.jeonghuncompy.cloud/docs) | 25% |
| 20241947_장수연 | 프론트엔드 / 대시보드 UX | React 19/Vite 대시보드 최종 구현, Cloudflare Pages 배포·환경변수·캐시 이슈 해결 | [배포 대시보드](https://ebpf-agent.com) | 25% |

---

## 이번 회차 상세 진행 내용

### 1. 백엔드·프론트엔드 최종 전달

이정훈·장수연이 5차 보고서에서 합의한 "데모 flow 완성" 목표를 마무리하고, 백엔드·프론트엔드 최종본과 아키텍처 문서(`ARCHITECTURE.md`)를 팀에 공유했다. 핵심 구성은 다음과 같다.

| 층 | 기술 | 역할 |
|---|---|---|
| 에이전트 | C++ / eBPF (libbpf) | 커널 이벤트 수집·탐지, NDJSON 전송 |
| 백엔드 | Python 3.11+ / FastAPI / SQLite(WAL) | 수집·검증·저장·조회·실시간 배포·인증 |
| 프론트엔드 | React 19 / Vite | 조회·필터·실시간 시각화, Cloudflare Pages 배포 |

- 이벤트는 `type` 필드로 구분되는 판별 유니온으로 총 13개 타입(`exec`, `file_write`, `net_connect`, `dns`, `ptrace`, `memfd`, `anomaly`, `correlation` 등)을 지원한다.
- 백엔드는 `POST /ingest`(에이전트, 관대 모드)와 `POST /api/v1/events/ndjson`(API, 엄격 모드)로 수집 경로를 이원화해, 한 줄의 스키마 위반이 배치 전체를 막는 poison-pill 문제를 방지했다.
- `GET /api/v1/events/stream`(SSE)으로 대시보드에 실시간 이벤트를 반영하며, Bearer 토큰 + 스코프(ingest/read/admin) 인증과 레이트리밋·본문 크기 제한으로 하드닝했다.

### 2. 라이브 데모 확인

배포된 대시보드(`ebpf-agent.com`)에서 실제 수집 백엔드(`asc4.jeonghuncompy.cloud`)로부터 이벤트를 조회·시각화하는 것을 확인했다.

- 전체 이벤트 299건, 알림 이벤트 24건, 최다 발생 타입 `exec`
- 심각도별 Critical 14 / High 23 / Medium 11 / Low 1
- 상세 패널에서 `net_connect` 이벤트가 룰 `R-025`("알려진 C2 서버 IP 연결", critical)로 탐지되는 것을 확인 — Agent 탐지 → Backend 저장 → Dashboard 시각화까지 전체 파이프라인이 한 번에 이어짐을 실증

(스크린샷: `assets/dashboard_live_demo.png`)

### 3. Cloudflare Pages 배포 트러블슈팅

프론트엔드 배포 과정에서 발생한 문제와 원인은 다음과 같다.

- **증상:** 배포 환경에서 API 요청이 `Unexpected token '<'` 오류로 실패
- **원인:** 초기에는 `/edr-api → 백엔드` 프록시를 `vercel.json`으로 설정했으나, 실제 배포처는 **Cloudflare Pages**라 `vercel.json` 설정을 인식하지 못함. 그 결과 `/edr-api/*` 요청이 백엔드 대신 SPA HTML로 응답됨
- **해결:** 프로덕션 빌드는 프록시를 거치지 않고 **백엔드(asc4)로 직접 요청**하도록 변경하고, 백엔드가 `ebpf-agent.com` 오리진에 CORS를 허용하도록 설정. 로컬 개발 환경은 기존 Vite 프록시를 그대로 유지
- 추가로 Pages 환경변수(`VITE_EDR_API_BASE_URL`) 반영 여부를 두고 한동안 혼선이 있었으나, 실제로는 설정에 문제가 없었고 **브라우저 hard refresh가 되지 않아 이전 빌드가 캐시된 상태로 보이는 것**이 원인이었다. 캐시 갱신 후 정상 동작을 확인했다.

---

## 이슈 및 해결 방안

| 이슈 | 해결 방법 |
|------|----------|
| Cloudflare Pages가 `vercel.json` 프록시 설정을 지원하지 않아 API 요청이 SPA HTML로 응답됨 | 프로덕션 빌드는 백엔드로 직접 요청 + 백엔드 CORS 허용으로 전환 |
| 환경변수 문제로 오인해 트러블슈팅 시간 소요 | 원인은 브라우저 캐시(hard refresh 미적용)였음 — 배포 후 캐시 무효화 점검을 배포 체크리스트에 추가 검토 |
| 벤치마크(실제 비교) 항목·방법이 아직 구체화되지 않음 | 다음 회차에서 도현이 시나리오별 탐지 여부·오탐/미탐 표를 우선 정리 |
| 발표까지 남은 기간이 짧아 슬라이드·리허설 일정이 촉박 | 데모 flow가 이미 확보되어 있으므로 슬라이드 제작과 벤치마크 정리를 병행 진행 |

---

## 다음 회차 목표

| 담당 | 목표 |
|------|------|
| 박도현 | 벤치마크(시나리오별 탐지 여부, 오탐/미탐) 정리 및 발표 슬라이드 초안 작성 |
| 강수빈 | 공격 시나리오·MITRE ATT&CK 매핑 정리, 발표 스크립트 초안 |
| 이정훈 · 장수연 | 배포 안정성 유지, 발표 중 라이브 데모 리허설 지원 |
| 공통 | 8월 발표 최종 리허설 1회 이상 |

---

## 참고 자료

- 소스코드: https://github.com/no-carve-only-pizza/edr-agent
- ProjectDB: https://github.com/no-carve-only-pizza/ProjectDB
- 배포 대시보드: https://ebpf-agent.com
- 백엔드 Swagger 문서: https://asc4.jeonghuncompy.cloud/docs
- 아키텍처 문서: `ARCHITECTURE.md`
- 프론트엔드 저장소: `COMPY07/edr-agent-frontend`
- MITRE ATT&CK Linux Matrix: https://attack.mitre.org/matrices/enterprise/linux/
