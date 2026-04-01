# Claude Code v2.1.88 — 소스 코드 분석

> **면책 조항**: 이 저장소의 모든 소스 코드는 **Anthropic 및 Claude**의 지적 재산입니다. 이 저장소는 오직 기술 연구, 학습, 그리고 관심 있는 사람들 간의 교육적 교류만을 위해 제공됩니다. **상업적 이용은 엄격히 금지됩니다.** 어떠한 개인, 조직, 기관도 이 내용을 상업적 목적, 영리 활동, 불법 행위, 또는 그 밖의 무단 용도로 사용할 수 없습니다. 본 저장소의 어떤 내용이 귀하의 법적 권리, 지적 재산권 또는 기타 이익을 침해한다고 판단되면 연락해 주시기 바랍니다. 확인 후 즉시 삭제하겠습니다.

> npm 패키지 `@anthropic-ai/claude-code` 버전 **2.1.88**에서 추출했습니다.
> 배포된 패키지는 단일 번들 `cli.js`(~12MB)만 포함합니다. 이 저장소의 `src/` 디렉터리에는 npm tarball에서 추출한 **비번들 TypeScript 소스**가 들어 있습니다.

**언어**: [English](README.md) | [中文](README_CN.md) | **한국어**

---

## 목차

- [심층 분석 보고서 (`docs/`)](#심층-분석-보고서-docs) — 텔레메트리, 코드명, 위장 모드, 원격 제어, 향후 로드맵
- [누락된 모듈 안내](#누락된-모듈-안내-108개-모듈) — npm 패키지에 포함되지 않은 feature gate 모듈 108개
- [아키텍처 개요](#아키텍처-개요) — Entry → Query Engine → Tools/Services/State
- [도구 시스템 및 권한](#도구-시스템-아키텍처) — 40개 이상의 도구, 권한 흐름, 서브 에이전트
- [12단계 점진적 하네스 메커니즘](#12단계-점진적-하네스-메커니즘) — Claude Code가 에이전트 루프 위에 운영용 기능을 어떻게 적층하는지
- [빌드 노트](#빌드-노트) — 이 소스가 바로 컴파일되지 않는 이유

---

## 심층 분석 보고서 (`docs/`)

역컴파일한 v2.1.88을 바탕으로 작성한 소스 코드 분석 보고서입니다. 다국어(EN/ZH/KR)로 제공합니다.

```
docs/
├── en/                                        # English
│   ├── [01-telemetry-and-privacy.md]          # Telemetry & Privacy — 무엇을 수집하는지, 왜 opt-out 할 수 없는지
│   ├── [02-hidden-features-and-codenames.md]  # 코드명(Capybara/Tengu/Numbat), feature flag, 내부/외부 차이
│   ├── [03-undercover-mode.md]                # Undercover Mode — 오픈소스 저장소에서 AI 작성 흔적 감추기
│   ├── [04-remote-control-and-killswitches.md]# Remote Control — 관리 설정, 킬스위치, 모델 override
│   └── [05-future-roadmap.md]                 # Future Roadmap — Numbat, KAIROS, 음성 모드, 미공개 도구
│
├── zh/                                        # 中文
│   ├── [01-遥测与隐私分析.md]                    # 遥测与隐私 — 收集了什么，为什么无法退出
│   ├── [02-隐藏功能与模型代号.md]                # 隐藏功能 — 模型代号，feature flag，内外用户差异
│   ├── [03-卧底模式分析.md]                     # 卧底模式 — 在开源项目中隐藏 AI 身份
│   ├── [04-远程控制与紧急开关.md]                # 远程控制 — 托管设置，紧急开关，模型覆盖
│   └── [05-未来路线图.md]                       # 未来路线图 — Numbat，KAIROS，语音模式，未上线工具
│
└── kr/                                        # 한국어
    ├── [01-telemetry-and-privacy.md]          # 텔레메트리와 프라이버시 — 무엇을 수집하는지, 왜 끌 수 없는지
    ├── [02-hidden-features-and-codenames.md]  # 숨겨진 기능과 모델 코드명 — 코드명, 플래그, 내부/외부 차이
    ├── [03-undercover-mode.md]                # 위장 모드 분석 — 공개 저장소에서 AI 작성 사실 숨기기
    ├── [04-remote-control-and-killswitches.md]# 원격 제어와 킬스위치 — 관리 설정, 긴급 차단, 모델 override
    └── [05-future-roadmap.md]                 # 미래 로드맵 — Numbat, KAIROS, 음성 모드, 미공개 도구
```

> 아래 표의 링크를 사용해 각 전체 보고서를 열 수 있습니다.

| # | 주제 | 핵심 발견 | 링크 |
|---|------|-----------|------|
| 01 | **텔레메트리와 프라이버시** | 분석 이벤트는 두 경로(1P → Anthropic, Datadog)로 전송됩니다. 모든 이벤트에 환경 지문, 프로세스 메트릭, 저장소 해시가 포함됩니다. 1차 로깅에는 **UI에서 노출된 opt-out 수단이 없습니다.** `OTEL_LOG_TOOL_DETAILS=1`을 설정하면 도구 입력 전체가 기록됩니다. | [EN](docs/en/01-telemetry-and-privacy.md) · [KR](docs/kr/01-telemetry-and-privacy.md) |
| 02 | **숨겨진 기능과 코드명** | 동물 코드명(Capybara v8, Tengu, Fennec→Opus 4.6, 다음은 **Numbat**)이 확인됩니다. feature flag는 목적을 숨기기 위해 `tengu_frond_boric` 같은 무작위 단어 조합을 사용합니다. 내부 사용자는 더 나은 프롬프트, 검증 에이전트, effort anchor를 받습니다. 숨겨진 명령어로 `/btw`, `/stickers`가 있습니다. | [EN](docs/en/02-hidden-features-and-codenames.md) · [KR](docs/kr/02-hidden-features-and-codenames.md) |
| 03 | **위장 모드** | Anthropic 직원은 공개 저장소에서 자동으로 위장 모드에 들어갑니다. 모델은 *"정체를 들키지 마라"* 는 지시를 받고 AI 흔적을 모두 제거하며, 커밋도 "인간 개발자처럼" 작성합니다. **강제로 끄는 옵션은 없습니다.** 이는 오픈소스 커뮤니티의 투명성 문제를 제기합니다. | [EN](docs/en/03-undercover-mode.md) · [KR](docs/kr/03-undercover-mode.md) |
| 04 | **원격 제어** | `/api/claude_code/settings`를 1시간마다 폴링합니다. 위험한 변경은 차단 대화상자를 띄우며 **거부하면 앱이 종료됩니다.** 권한 우회, fast mode, voice mode, analytics sink 등을 포함해 6개 이상의 킬스위치가 존재합니다. GrowthBook 플래그는 사용자 동의 없이 동작을 바꿀 수 있습니다. | [EN](docs/en/04-remote-control-and-killswitches.md) · [KR](docs/kr/04-remote-control-and-killswitches.md) |
| 05 | **미래 로드맵** | **Numbat** 코드명이 확인되었습니다. Opus 4.7 / Sonnet 4.8이 개발 중입니다. **KAIROS**는 `<tick>` 하트비트, 푸시 알림, PR 구독을 갖춘 완전 자율 에이전트 모드입니다. 음성 모드(push-to-talk)는 구현되어 있지만 게이트로 막혀 있습니다. 미출시 도구도 17개 확인되었습니다. | [EN](docs/en/05-future-roadmap.md) · [KR](docs/kr/05-future-roadmap.md) |

---

## 누락된 모듈 안내 (108개 모듈)

> **이 소스는 완전하지 않습니다.** `feature()` 게이트 분기에서 참조되는 모듈 108개는 npm 패키지에 **포함되어 있지 않습니다.**
> 이 모듈들은 Anthropic 내부 모노레포에만 존재하며, 컴파일 시점에 dead-code-elimination으로 제거되었습니다.
> 따라서 `cli.js`, `sdk-tools.d.ts`, 또는 공개된 어떤 산출물에서도 **복구할 수 없습니다.**

### Anthropic 내부 코드 (~70개 모듈, 미공개)

이 모듈들은 npm 패키지 어디에도 소스 파일이 없습니다. Anthropic 내부 인프라 전용입니다.

<details>
<summary>전체 목록 펼치기</summary>

| 모듈 | 용도 | Feature Gate |
|------|------|--------------|
| `daemon/main.js` | 백그라운드 데몬 감독자 | `DAEMON` |
| `daemon/workerRegistry.js` | 데몬 워커 레지스트리 | `DAEMON` |
| `proactive/index.js` | 사전 알림 시스템 | `PROACTIVE` |
| `contextCollapse/index.js` | 컨텍스트 collapse 서비스(실험적) | `CONTEXT_COLLAPSE` |
| `contextCollapse/operations.js` | collapse 작업 | `CONTEXT_COLLAPSE` |
| `contextCollapse/persist.js` | collapse 영속화 | `CONTEXT_COLLAPSE` |
| `skillSearch/featureCheck.js` | 원격 스킬 기능 체크 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/remoteSkillLoader.js` | 원격 스킬 로더 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/remoteSkillState.js` | 원격 스킬 상태 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/telemetry.js` | 스킬 검색 텔레메트리 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/localSearch.js` | 로컬 스킬 검색 | `EXPERIMENTAL_SKILL_SEARCH` |
| `skillSearch/prefetch.js` | 스킬 프리패치 | `EXPERIMENTAL_SKILL_SEARCH` |
| `coordinator/workerAgent.js` | 멀티 에이전트 coordinator 워커 | `COORDINATOR_MODE` |
| `bridge/peerSessions.js` | 브리지 peer 세션 관리 | `BRIDGE_MODE` |
| `assistant/index.js` | Kairos assistant 모드 | `KAIROS` |
| `assistant/AssistantSessionChooser.js` | assistant 세션 선택기 | `KAIROS` |
| `compact/reactiveCompact.js` | 반응형 컨텍스트 압축 | `CACHED_MICROCOMPACT` |
| `compact/snipCompact.js` | snip 기반 압축 | `HISTORY_SNIP` |
| `compact/snipProjection.js` | snip projection | `HISTORY_SNIP` |
| `compact/cachedMCConfig.js` | cached micro-compact 설정 | `CACHED_MICROCOMPACT` |
| `sessionTranscript/sessionTranscript.js` | 세션 전사 서비스 | `TRANSCRIPT_CLASSIFIER` |
| `commands/agents-platform/index.js` | 내부 agents 플랫폼 | `ant` (internal) |
| `commands/assistant/index.js` | assistant 명령어 | `KAIROS` |
| `commands/buddy/index.js` | buddy 시스템 알림 | `BUDDY` |
| `commands/fork/index.js` | fork subagent 명령어 | `FORK_SUBAGENT` |
| `commands/peers/index.js` | 멀티 peer 명령어 | `BRIDGE_MODE` |
| `commands/proactive.js` | proactive 명령어 | `PROACTIVE` |
| `commands/remoteControlServer/index.js` | 원격 제어 서버 | `DAEMON` + `BRIDGE_MODE` |
| `commands/subscribe-pr.js` | GitHub PR 구독 | `KAIROS_GITHUB_WEBHOOKS` |
| `commands/torch.js` | 내부 디버그 도구 | `TORCH` |
| `commands/workflows/index.js` | 워크플로 명령어 | `WORKFLOW_SCRIPTS` |
| `jobs/classifier.js` | 내부 작업 분류기 | `TEMPLATES` |
| `memdir/memoryShapeTelemetry.js` | 메모리 형태 텔레메트리 | `MEMORY_SHAPE_TELEMETRY` |
| `services/sessionTranscript/sessionTranscript.js` | 세션 전사 | `TRANSCRIPT_CLASSIFIER` |
| `tasks/LocalWorkflowTask/LocalWorkflowTask.js` | 로컬 워크플로 태스크 | `WORKFLOW_SCRIPTS` |
| `protectedNamespace.js` | 내부 네임스페이스 가드 | `ant` (internal) |
| `protectedNamespace.js` (envUtils) | 보호 네임스페이스 런타임 | `ant` (internal) |
| `coreTypes.generated.js` | 생성된 core type | `ant` (internal) |
| `devtools.js` | 내부 개발 도구 | `ant` (internal) |
| `attributionHooks.js` | 내부 attribution 훅 | `COMMIT_ATTRIBUTION` |
| `systemThemeWatcher.js` | 시스템 테마 감시기 | `AUTO_THEME` |
| `udsClient.js` / `udsMessaging.js` | UDS 메시징 클라이언트 | `UDS_INBOX` |
| `systemThemeWatcher.js` | 테마 감시기 | `AUTO_THEME` |

</details>

### Feature Gate 도구 (~20개 모듈, 번들에서 DCE 처리)

이 도구들은 `sdk-tools.d.ts`에는 타입 시그니처가 있지만 실제 구현은 컴파일 시 제거되었습니다.

<details>
<summary>전체 목록 펼치기</summary>

| 도구 | 용도 | Feature Gate |
|------|------|--------------|
| `REPLTool` | 인터랙티브 REPL (VM sandbox) | `ant` (internal) |
| `SnipTool` | 컨텍스트 snipping | `HISTORY_SNIP` |
| `SleepTool` | 에이전트 루프 내 sleep/delay | `PROACTIVE` / `KAIROS` |
| `MonitorTool` | MCP 모니터링 | `MONITOR_TOOL` |
| `OverflowTestTool` | overflow 테스트 | `OVERFLOW_TEST_TOOL` |
| `WorkflowTool` | 워크플로 실행 | `WORKFLOW_SCRIPTS` |
| `WebBrowserTool` | 브라우저 자동화 | `WEB_BROWSER_TOOL` |
| `TerminalCaptureTool` | 터미널 캡처 | `TERMINAL_PANEL` |
| `TungstenTool` | 내부 성능 모니터링 | `ant` (internal) |
| `VerifyPlanExecutionTool` | 계획 검증 | `CLAUDE_CODE_VERIFY_PLAN` |
| `SendUserFileTool` | 사용자에게 파일 보내기 | `KAIROS` |
| `SubscribePRTool` | GitHub PR 구독 | `KAIROS_GITHUB_WEBHOOKS` |
| `SuggestBackgroundPRTool` | 백그라운드 PR 제안 | `KAIROS` |
| `PushNotificationTool` | 푸시 알림 | `KAIROS` |
| `CtxInspectTool` | 컨텍스트 검사 | `CONTEXT_COLLAPSE` |
| `ListPeersTool` | 활성 peer 목록 | `UDS_INBOX` |
| `DiscoverSkillsTool` | 스킬 탐색 | `EXPERIMENTAL_SKILL_SEARCH` |

</details>

### 텍스트/프롬프트 자산 (~6개 파일)

이 파일들은 내부 프롬프트 템플릿과 문서이며 공개되지 않았습니다.

<details>
<summary>펼치기</summary>

| 파일 | 용도 |
|------|------|
| `yolo-classifier-prompts/auto_mode_system_prompt.txt` | 분류기용 auto-mode 시스템 프롬프트 |
| `yolo-classifier-prompts/permissions_anthropic.txt` | Anthropic 내부 권한 프롬프트 |
| `yolo-classifier-prompts/permissions_external.txt` | 외부 사용자 권한 프롬프트 |
| `verify/SKILL.md` | 검증 스킬 문서 |
| `verify/examples/cli.md` | CLI 검증 예시 |
| `verify/examples/server.md` | 서버 검증 예시 |

</details>

### 왜 누락되었는가

```
  Anthropic Internal Monorepo              Published npm Package
  ──────────────────────────               ─────────────────────
  feature('DAEMON') → true    ──build──→   feature('DAEMON') → false
  ↓                                         ↓
  daemon/main.js  ← INCLUDED    ──bundle─→  daemon/main.js  ← DELETED (DCE)
  tools/REPLTool  ← INCLUDED    ──bundle─→  tools/REPLTool  ← DELETED (DCE)
  proactive/      ← INCLUDED    ──bundle─→  (src/에는 참조만 있고 실제 파일은 없음)
  ```

  Bun의 `feature()`는 **컴파일 타임 intrinsic**입니다.
  - Anthropic 내부 빌드에서는 `true`를 반환하므로 코드가 번들에 유지됩니다.
  - 공개 빌드에서는 `false`를 반환하므로 코드가 dead-code-elimination 됩니다.
  - 따라서 이 108개 모듈은 공개 산출물 어디에도 존재하지 않습니다.

---

## 저작권 및 면책 조항

```
Copyright (c) Anthropic. All rights reserved.

All source code in this repository is the intellectual property of Anthropic and Claude.
This repository is provided strictly for technical research and educational purposes.
Commercial use is strictly prohibited.

If you are the copyright owner and believe this repository infringes your rights,
please contact the repository owner for immediate removal.
```

---

## 통계

| 항목 | 개수 |
|------|------|
| 소스 파일 (.ts/.tsx) | ~1,884 |
| 코드 라인 수 | ~512,664 |
| 가장 큰 단일 파일 | `query.ts` (~785KB) |
| 내장 도구 | ~40+ |
| 슬래시 명령어 | ~80+ |
| 의존성 (node_modules) | ~192 packages |
| 런타임 | Bun (Node.js >= 18 번들로 컴파일) |

---

## 에이전트 패턴

```
                    핵심 루프
                    =========

    User --> messages[] --> Claude API --> response
                                          |
                                stop_reason == "tool_use"?
                               /                          \
                             yes                           no
                              |                             |
                        execute tools                    return text
                        append tool_result
                        loop back -----------------> messages[]


    이것이 최소한의 에이전트 루프입니다. Claude Code는 이 루프 위에
    권한, 스트리밍, 동시성, 압축, 서브 에이전트, 영속성, MCP를 더해
    운영 환경용 하네스를 구성합니다.
```

---

## 디렉터리 참조

```
src/
├── main.tsx                 # REPL 부트스트랩, 4,683줄
├── QueryEngine.ts           # SDK/headless query 라이프사이클 엔진
├── query.ts                 # 메인 에이전트 루프 (785KB, 가장 큰 파일)
├── Tool.ts                  # Tool 인터페이스 + buildTool 팩토리
├── Task.ts                  # Task 타입, ID, 상태 기반
├── tools.ts                 # Tool 레지스트리, 프리셋, 필터링
├── commands.ts              # 슬래시 명령 정의
├── context.ts               # 사용자 입력 컨텍스트
├── cost-tracker.ts          # API 비용 누적
├── setup.ts                 # 최초 실행 설정 플로우
│
├── bridge/                  # Claude Desktop / 원격 브리지
│   ├── bridgeMain.ts        #   세션 라이프사이클 관리자
│   ├── bridgeApi.ts         #   HTTP 클라이언트
│   ├── bridgeConfig.ts      #   연결 설정
│   ├── bridgeMessaging.ts   #   메시지 중계
│   ├── sessionRunner.ts     #   프로세스 실행
│   ├── jwtUtils.ts          #   JWT 갱신
│   ├── workSecret.ts        #   인증 토큰
│   └── capacityWake.ts      #   용량 기반 깨우기
│
├── cli/                     # CLI 인프라
│   ├── handlers/            #   명령 핸들러
│   └── transports/          #   I/O 전송 계층 (stdio, structured)
│
├── commands/                # 약 80개의 슬래시 명령
│   ├── agents/              #   에이전트 관리
│   ├── compact/             #   컨텍스트 압축
│   ├── config/              #   설정 관리
│   ├── help/                #   도움말 표시
│   ├── login/               #   인증
│   ├── mcp/                 #   MCP 서버 관리
│   ├── memory/              #   메모리 시스템
│   ├── plan/                #   계획 모드
│   ├── resume/              #   세션 재개
│   ├── review/              #   코드 리뷰
│   └── ...                  #   그 외 70개 이상 명령
│
├── components/              # React/Ink 터미널 UI
│   ├── design-system/       #   재사용 가능한 UI 기본 요소
│   ├── messages/            #   메시지 렌더링
│   ├── permissions/         #   권한 대화상자
│   ├── PromptInput/         #   입력 필드 + 제안
│   ├── LogoV2/              #   브랜딩 + 웰컴 화면
│   ├── Settings/            #   설정 패널
│   ├── Spinner.tsx          #   로딩 인디케이터
│   └── ...                  #   40개 이상의 컴포넌트 그룹
│
├── entrypoints/             # 애플리케이션 엔트리 포인트
│   ├── cli.tsx              #   CLI 메인 (버전, 도움말, daemon)
│   ├── sdk/                 #   에이전트 SDK (타입, 세션)
│   └── mcp.ts               #   MCP 서버 엔트리
│
├── hooks/                   # React 훅
│   ├── useCanUseTool.tsx    #   권한 검사
│   ├── useReplBridge.tsx    #   브리지 연결
│   ├── notifs/              #   알림 훅
│   └── toolPermission/      #   도구 권한 핸들러
│
├── services/                # 비즈니스 로직 계층
│   ├── api/                 #   Claude API 클라이언트
│   │   ├── claude.ts        #     스트리밍 API 호출
│   │   ├── errors.ts        #     오류 분류
│   │   └── withRetry.ts     #     재시도 로직
│   ├── analytics/           #   텔레메트리 + GrowthBook
│   ├── compact/             #   컨텍스트 압축
│   ├── mcp/                 #   MCP 연결 관리
│   ├── tools/               #   도구 실행 엔진
│   │   ├── StreamingToolExecutor.ts  # 병렬 도구 실행기
│   │   └── toolOrchestration.ts      # 배치 오케스트레이션
│   ├── plugins/             #   플러그인 로더
│   └── settingsSync/        #   기기 간 설정 동기화
│
├── state/                   # 애플리케이션 상태
│   ├── AppStateStore.ts     #   스토어 정의
│   └── AppState.tsx         #   React provider + hooks
│
├── tasks/                   # Task 구현
│   ├── LocalShellTask/      #   Bash 명령 실행
│   ├── LocalAgentTask/      #   서브 에이전트 실행
│   ├── RemoteAgentTask/     #   브리지를 통한 원격 에이전트
│   ├── InProcessTeammateTask/ # 프로세스 내 팀메이트
│   └── DreamTask/           #   백그라운드 사고
│
├── tools/                   # 40개 이상의 도구 구현
│   ├── AgentTool/           #   서브 에이전트 spawn + fork
│   ├── BashTool/            #   셸 명령 실행
│   ├── FileReadTool/        #   파일 읽기 (PDF, 이미지 등)
│   ├── FileEditTool/        #   문자열 치환 편집
│   ├── FileWriteTool/       #   전체 파일 생성
│   ├── GlobTool/            #   파일 패턴 검색
│   ├── GrepTool/            #   내용 검색 (ripgrep)
│   ├── WebFetchTool/        #   HTTP 가져오기
│   ├── WebSearchTool/       #   웹 검색
│   ├── MCPTool/             #   MCP 도구 래퍼
│   ├── SkillTool/           #   스킬 호출
│   ├── AskUserQuestionTool/ #   사용자 상호작용
│   └── ...                  #   그 외 30개 이상의 도구
│
├── types/                   # 타입 정의
│   ├── message.ts           #   메시지 discriminated union
│   ├── permissions.ts       #   권한 타입
│   ├── tools.ts             #   도구 진행 상태 타입
│   └── ids.ts               #   브랜디드 ID 타입
│
├── utils/                   # 유틸리티 (가장 큰 디렉터리)
│   ├── permissions/         #   권한 규칙 엔진
│   ├── messages/            #   메시지 포매팅
│   ├── model/               #   모델 선택 로직
│   ├── settings/            #   설정 관리
│   ├── sandbox/             #   샌드박스 런타임 어댑터
│   ├── hooks/               #   훅 실행
│   ├── memory/              #   메모리 시스템 유틸
│   ├── git/                 #   Git 작업
│   ├── github/              #   GitHub API
│   ├── bash/                #   Bash 실행 헬퍼
│   ├── swarm/               #   멀티 에이전트 swarm
│   ├── telemetry/           #   텔레메트리 보고
│   └── ...                  #   그 외 30개 이상의 유틸 그룹
│
└── vendor/                  # 네이티브 모듈 소스 스텁
    ├── audio-capture-src/   #   오디오 입력
    ├── image-processor-src/ #   이미지 처리
    ├── modifiers-napi-src/  #   네이티브 modifier
    └── url-handler-src/     #   URL 처리
```

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ENTRY LAYER                                 │
│  cli.tsx ──> main.tsx ──> REPL.tsx (interactive)                   │
│                     └──> QueryEngine.ts (headless/SDK)              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       QUERY ENGINE                                  │
│  submitMessage(prompt) ──> AsyncGenerator<SDKMessage>               │
│    │                                                                │
│    ├── fetchSystemPromptParts()    ──> 시스템 프롬프트 조립          │
│    ├── processUserInput()          ──> /commands 처리               │
│    ├── query()                     ──> 메인 에이전트 루프           │
│    │     ├── StreamingToolExecutor ──> 병렬 도구 실행               │
│    │     ├── autoCompact()         ──> 컨텍스트 압축               │
│    │     └── runTools()            ──> 도구 오케스트레이션          │
│    └── yield SDKMessage            ──> 소비자에게 스트리밍          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
┌──────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│   TOOL SYSTEM    │ │  SERVICE LAYER  │ │   STATE LAYER    │
│                  │ │                 │ │                  │
│ Tool Interface   │ │ api/claude.ts   │ │ AppState Store   │
│  ├─ call()       │ │  API client     │ │  ├─ permissions  │
│  ├─ validate()   │ │ compact/        │ │  ├─ fileHistory  │
│  ├─ checkPerms() │ │  auto-compact   │ │  ├─ agents       │
│  ├─ render()     │ │ mcp/            │ │  └─ fastMode     │
│  └─ prompt()     │ │  MCP protocol   │ │                  │
│                  │ │ analytics/      │ │ React Context    │
│ 40+ Built-in:    │ │  telemetry      │ │  ├─ useAppState  │
│  ├─ BashTool     │ │ tools/          │ │  └─ useSetState  │
│  ├─ FileRead     │ │  executor       │ │                  │
│  ├─ FileEdit     │ │ plugins/        │ └──────────────────┘
│  ├─ Glob/Grep    │ │  loader         │
│  ├─ AgentTool    │ │ settingsSync/   │
│  ├─ WebFetch     │ │  cross-device   │
│  └─ MCPTool      │ │ oauth/          │
│                  │ │  auth flow      │
└──────────────────┘ └─────────────────┘
              │                │
              ▼                ▼
┌──────────────────┐ ┌─────────────────┐
│   TASK SYSTEM    │ │   BRIDGE LAYER  │
│                  │ │                 │
│ Task Types:      │ │ bridgeMain.ts   │
│  ├─ local_bash   │ │  session mgmt   │
│  ├─ local_agent  │ │ bridgeApi.ts    │
│  ├─ remote_agent │ │  HTTP client    │
│  ├─ in_process   │ │ workSecret.ts   │
│  ├─ dream        │ │  auth tokens    │
│  └─ workflow     │ │ sessionRunner   │
│                  │ │  process spawn  │
│ ID: prefix+8chr  │ └─────────────────┘
│  b=bash a=agent  │
│  r=remote t=team │
└──────────────────┘
```

---

## 데이터 흐름: 단일 질의 라이프사이클

```
 USER INPUT (prompt / slash command)
     │
     ▼
 processUserInput()                ← /commands 파싱, UserMessage 생성
     │
     ▼
 fetchSystemPromptParts()          ← tools → 프롬프트 섹션, CLAUDE.md 메모리
     │
     ▼
 recordTranscript()                ← 사용자 메시지를 디스크(JSONL)에 저장
     │
     ▼
 ┌─→ normalizeMessagesForAPI()     ← UI 전용 필드 제거, 필요 시 압축
 │   │
 │   ▼
 │   Claude API (streaming)        ← tools + system prompt와 함께 POST /v1/messages
 │   │
 │   ▼
 │   stream events                 ← message_start → content_block_delta → message_stop
 │   │
 │   ├─ text block ──────────────→ 소비자(SDK / REPL)에게 전달
 │   │
 │   └─ tool_use block?
 │       │
 │       ▼
 │   StreamingToolExecutor         ← 동시 실행 가능 / 직렬 실행 분할
 │       │
 │       ▼
 │   canUseTool()                  ← 권한 검사 (hooks + rules + UI prompt)
 │       │
 │       ├─ DENY ────────────────→ tool_result(error) 추가 후 루프 지속
 │       │
 │       └─ ALLOW
 │           │
 │           ▼
 │       tool.call()               ← 도구 실행 (Bash, Read, Edit 등)
 │           │
 │           ▼
 │       append tool_result        ← messages[]에 추가, recordTranscript()
 │           │
 └─────────┘                      ← API 호출로 다시 루프
     │
     ▼ (stop_reason != "tool_use")
 yield result message              ← 최종 텍스트, usage, cost, session_id
```

---

## 도구 시스템 아키텍처

```
                    TOOL INTERFACE
                    ==============

    buildTool(definition) ──> Tool<Input, Output, Progress>

    모든 도구는 다음을 구현합니다:
    ┌────────────────────────────────────────────────────────┐
    │  LIFECYCLE                                             │
    │  ├── validateInput()      → 잘못된 인자를 조기 거부    │
    │  ├── checkPermissions()   → 도구별 권한 검사           │
    │  └── call()               → 실행 후 결과 반환          │
    │                                                        │
    │  CAPABILITIES                                          │
    │  ├── isEnabled()          → feature gate 검사          │
    │  ├── isConcurrencySafe()  → 병렬 실행 가능 여부        │
    │  ├── isReadOnly()         → 부작용이 없는지            │
    │  ├── isDestructive()      → 되돌릴 수 없는 작업인지    │
    │  └── interruptBehavior()  → 사용자 입력 시 취소/차단   │
    │                                                        │
    │  RENDERING (React/Ink)                                 │
    │  ├── renderToolUseMessage()     → 입력 표시            │
    │  ├── renderToolResultMessage()  → 출력 표시            │
    │  ├── renderToolUseProgressMessage() → spinner/status   │
    │  └── renderGroupedToolUse()     → 병렬 도구 그룹       │
    │                                                        │
    │  AI FACING                                             │
    │  ├── prompt()             → LLM용 도구 설명           │
    │  ├── description()        → 동적 설명                 │
    │  └── mapToolResultToAPI() → API 응답 형식으로 변환    │
    └────────────────────────────────────────────────────────┘
```

### 전체 도구 목록

```
    FILE OPERATIONS          SEARCH & DISCOVERY        EXECUTION
    ═════════════════        ══════════════════════     ══════════
    FileReadTool             GlobTool                  BashTool
    FileEditTool             GrepTool                  PowerShellTool
    FileWriteTool            ToolSearchTool
    NotebookEditTool                                   INTERACTION
                                                       ═══════════
    WEB & NETWORK           AGENT / TASK              AskUserQuestionTool
    ════════════════        ══════════════════        BriefTool
    WebFetchTool             AgentTool
    WebSearchTool            SendMessageTool           PLANNING & WORKFLOW
                             TeamCreateTool            ════════════════════
    MCP PROTOCOL             TeamDeleteTool            EnterPlanModeTool
    ══════════════           TaskCreateTool            ExitPlanModeTool
    MCPTool                  TaskGetTool               EnterWorktreeTool
    ListMcpResourcesTool     TaskUpdateTool            ExitWorktreeTool
    ReadMcpResourceTool      TaskListTool              TodoWriteTool
                             TaskStopTool
                             TaskOutputTool            SYSTEM
                                                       ════════
                             SKILLS & EXTENSIONS       ConfigTool
                             ═════════════════════     SkillTool
                             SkillTool                 ScheduleCronTool
                             LSPTool                   SleepTool
                                                       TungstenTool
```

---

## 권한 시스템

```
    TOOL CALL REQUEST
          │
          ▼
    ┌─ validateInput() ──────────────────────────────────┐
    │  어떤 권한 검사보다 먼저 잘못된 입력을 거부        │
    └────────────────────┬───────────────────────────────┘
                         │
                         ▼
    ┌─ PreToolUse Hooks ─────────────────────────────────┐
    │  사용자 정의 셸 명령(settings.json hooks)          │
    │  승인, 거부, 입력 수정이 가능                      │
    └────────────────────┬───────────────────────────────┘
                         │
                         ▼
    ┌─ Permission Rules ─────────────────────────────────┐
    │  alwaysAllowRules: 도구 이름/패턴 일치 → 자동 허용 │
    │  alwaysDenyRules:  도구 이름/패턴 일치 → 거부      │
    │  alwaysAskRules:   도구 이름/패턴 일치 → 질의      │
    │  출처: 설정, CLI 인자, 세션 결정                   │
    └────────────────────┬───────────────────────────────┘
                         │
                    규칙이 없으면?
                         │
                         ▼
    ┌─ Interactive Prompt ───────────────────────────────┐
    │  사용자는 도구 이름과 입력을 확인                   │
    │  옵션: Allow Once / Allow Always / Deny            │
    └────────────────────┬───────────────────────────────┘
                         │
                         ▼
    ┌─ checkPermissions() ───────────────────────────────┐
    │  도구별 로직(예: 경로 샌드박싱)                     │
    └────────────────────┬───────────────────────────────┘
                         │
                    APPROVED → tool.call()
```

---

## 서브 에이전트 및 멀티 에이전트 아키텍처

```
                        MAIN AGENT
                        ==========
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌──────────────┐ ┌──────────┐ ┌──────────────┐
     │  FORK AGENT  │ │ REMOTE   │ │ IN-PROCESS   │
     │              │ │ AGENT    │ │ TEAMMATE     │
     │ Fork process │ │ Bridge   │ │ Same process │
     │ Shared cache │ │ session  │ │ Async context│
     │ Fresh msgs[] │ │ Isolated │ │ Shared state │
     └──────────────┘ └──────────┘ └──────────────┘

    생성 모드:
    ├─ default    → 같은 대화를 공유하는 in-process
    ├─ fork       → 자식 프로세스, 새로운 messages[], 공유 파일 캐시
    ├─ worktree   → 격리된 git worktree + fork
    └─ remote     → Claude Code Remote / container로 브리지

    COMMUNICATION:
    ├─ SendMessageTool     → 에이전트 간 메시지
    ├─ TaskCreate/Update   → 공유 작업 보드
    └─ TeamCreate/Delete   → 팀 라이프사이클 관리

    SWARM MODE (feature-gated):
    ┌─────────────────────────────────────────────┐
    │  Lead Agent                                 │
    │    ├── Teammate A ──> Task 1 선점           │
    │    ├── Teammate B ──> Task 2 선점           │
    │    └── Teammate C ──> Task 3 선점           │
    │                                             │
    │  Shared: task board, message inbox          │
    │  Isolated: messages[], file cache, cwd      │
    └─────────────────────────────────────────────┘
```

---

## 컨텍스트 관리 (Compact 시스템)

```
    CONTEXT WINDOW BUDGET
    ═════════════════════

    ┌─────────────────────────────────────────────────────┐
    │  시스템 프롬프트 (tools, permissions, CLAUDE.md)    │
    │  ══════════════════════════════════════════════      │
    │                                                     │
    │  대화 히스토리                                      │
    │  ┌─────────────────────────────────────────────┐    │
    │  │ [오래된 메시지의 압축 요약]                  │    │
    │  │ ═══════════════════════════════════════════  │    │
    │  │ [compact_boundary marker]                   │    │
    │  │ ─────────────────────────────────────────── │    │
    │  │ [최근 메시지 — 완전 보존]                   │    │
    │  │ user → assistant → tool_use → tool_result  │    │
    │  └─────────────────────────────────────────────┘    │
    │                                                     │
    │  현재 턴 (user + assistant response)                │
    └─────────────────────────────────────────────────────┘

    세 가지 압축 전략:
    ├─ autoCompact     → 토큰 수가 임계값을 넘으면 동작
    │                     compact API 호출로 오래된 메시지를 요약
    ├─ snipCompact     → zombie 메시지와 오래된 marker 제거
    │                     (HISTORY_SNIP feature flag)
    └─ contextCollapse → 효율을 위해 컨텍스트를 재구성
                         (CONTEXT_COLLAPSE feature flag)

    압축 흐름:
    messages[] ──> getMessagesAfterCompactBoundary()
                        │
                        ▼
                  오래된 메시지 ──> Claude API (요약) ──> compact summary
                        │
                        ▼
                  [summary] + [compact_boundary] + [recent messages]
```

---

## MCP (Model Context Protocol) 통합

```
    ┌─────────────────────────────────────────────────────────┐
    │                  MCP ARCHITECTURE                        │
    │                                                         │
    │  MCPConnectionManager.tsx                               │
    │    ├── Server Discovery (settings.json의 설정)          │
    │    │     ├── stdio  → 자식 프로세스 실행                │
    │    │     ├── sse    → HTTP EventSource                  │
    │    │     ├── http   → Streamable HTTP                   │
    │    │     ├── ws     → WebSocket                         │
    │    │     └── sdk    → 프로세스 내 transport             │
    │    │                                                    │
    │    ├── Client Lifecycle                                  │
    │    │     ├── connect → initialize → list tools          │
    │    │     ├── MCPTool 래퍼를 통한 tool call              │
    │    │     └── disconnect / reconnect with backoff        │
    │    │                                                    │
    │    ├── Authentication                                   │
    │    │     ├── OAuth 2.0 flow (McpOAuthConfig)            │
    │    │     ├── Cross-App Access (XAA / SEP-990)           │
    │    │     └── 헤더 기반 API key                          │
    │    │                                                    │
    │    └── Tool Registration                                │
    │          ├── mcp__<server>__<tool> 네이밍 규칙          │
    │          ├── MCP 서버에서 동적 스키마 로드              │
    │          ├── Claude Code로 권한 전달                    │
    │          └── 리소스 목록 (ListMcpResourcesTool)         │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
```

---

## 브리지 계층 (Claude Desktop / Remote)

```
    Claude Desktop / Web / Cowork          Claude Code CLI
    ══════════════════════════            ═════════════════

    ┌───────────────────┐                 ┌──────────────────┐
    │  Bridge Client    │  ←─ HTTP ──→   │  bridgeMain.ts   │
    │  (Desktop App)    │                 │                  │
    └───────────────────┘                 │  Session Manager │
                                          │  ├── spawn CLI   │
    PROTOCOL:                             │  ├── poll status │
    ├─ JWT authentication                 │  ├── relay msgs  │
    ├─ Work secret exchange               │  └── capacityWake│
    ├─ Session lifecycle                  │                  │
    │  ├── create                         │  Backoff:        │
    │  ├── run                            │  ├─ conn: 2s→2m  │
    │  └─ stop                            │  └─ gen: 500ms→30s│
    └─ Token refresh scheduler            └──────────────────┘
```

---

## 세션 영속성

```
    SESSION STORAGE
    ══════════════

    ~/.claude/projects/<hash>/sessions/
    └── <session-id>.jsonl           ← append-only 로그
        ├── {"type":"user",...}
        ├── {"type":"assistant",...}
        ├── {"type":"progress",...}
        └── {"type":"system","subtype":"compact_boundary",...}

    재개 흐름:
    getLastSessionLog() ──> JSONL 파싱 ──> messages[] 재구성
         │
         ├── --continue     → 현재 cwd의 마지막 세션
         ├── --resume <id>  → 특정 세션
         └── --fork-session → 새 ID 생성, 히스토리 복사

    영속화 전략:
    ├─ 사용자 메시지  → await write (충돌 복구를 위해 blocking)
    ├─ assistant 메시지 → fire-and-forget (순서 보장 큐)
    ├─ 진행 상태      → inline write (다음 질의에서 dedup)
    └─ Flush          → 결과 전달 시 / cowork eager flush
```

---

## Feature Flag 시스템

```
    DEAD CODE ELIMINATION (Bun compile-time)
    ══════════════════════════════════════════

    feature('FLAG_NAME')  ──→  true  → 번들에 포함
                           ──→  false → 번들에서 제거

    FLAGS (소스에서 확인된 것):
    ├─ COORDINATOR_MODE      → 멀티 에이전트 coordinator
    ├─ HISTORY_SNIP          → 공격적인 히스토리 trimming
    ├─ CONTEXT_COLLAPSE      → 컨텍스트 재구성
    ├─ DAEMON                → 백그라운드 데몬 워커
    ├─ AGENT_TRIGGERS        → cron/원격 트리거
    ├─ AGENT_TRIGGERS_REMOTE → 원격 트리거 지원
    ├─ MONITOR_TOOL          → MCP 모니터링 도구
    ├─ WEB_BROWSER_TOOL      → 브라우저 자동화
    ├─ VOICE_MODE            → 음성 입력/출력
    ├─ TEMPLATES             → 작업 분류기
    ├─ EXPERIMENTAL_SKILL_SEARCH → 스킬 탐색
    ├─ KAIROS                → 푸시 알림, 파일 전송
    ├─ PROACTIVE             → sleep tool, proactive 동작
    ├─ OVERFLOW_TEST_TOOL    → 테스트 도구
    ├─ TERMINAL_PANEL        → 터미널 캡처
    ├─ WORKFLOW_SCRIPTS      → 워크플로 도구
    ├─ CHICAGO_MCP           → 컴퓨터 사용 MCP
    ├─ DUMP_SYSTEM_PROMPT    → 프롬프트 추출 (ant 전용)
    ├─ UDS_INBOX             → peer 탐색
    ├─ ABLATION_BASELINE     → 실험 ablation
    └─ UPGRADE_NOTICE        → 업그레이드 알림

    런타임 게이트:
    ├─ process.env.USER_TYPE === 'ant'  → Anthropic 내부 기능
    └─ GrowthBook feature flags         → 런타임 A/B 실험
```

---

## 상태 관리

```
    ┌──────────────────────────────────────────────────────────┐
    │                  AppState Store                           │
    │                                                          │
    │  AppState {                                              │
    │    toolPermissionContext: {                              │
    │      mode: PermissionMode,           ← default/plan/etc │
    │      additionalWorkingDirectories,                        │
    │      alwaysAllowRules,               ← 자동 승인         │
    │      alwaysDenyRules,                ← 자동 거부         │
    │      alwaysAskRules,                 ← 항상 질의         │
    │      isBypassPermissionsModeAvailable                    │
    │    },                                                    │
    │    fileHistory: FileHistoryState,    ← undo 스냅샷       │
    │    attribution: AttributionState,    ← commit 추적       │
    │    verbose: boolean,                                     │
    │    mainLoopModel: string,           ← 활성 모델          │
    │    fastMode: FastModeState,                              │
    │    speculation: SpeculationState                         │
    │  }                                                       │
    │                                                          │
    │  React 통합:                                             │
    │  ├── AppStateProvider   → createContext로 store 생성     │
    │  ├── useAppState(sel)   → selector 기반 구독             │
    │  └── useSetAppState()   → immer 스타일 updater 함수      │
    └──────────────────────────────────────────────────────────┘
```

---

## 12단계 점진적 하네스 메커니즘

이 소스 코드는 운영 환경의 AI 에이전트 하네스가 기본 루프 외에 필요로 하는 12개의 계층적 메커니즘을 보여줍니다. 각 단계는 이전 단계 위에 쌓입니다.

```
    s01  THE LOOP             "루프 하나와 Bash만 있어도 시작할 수 있다"
         query.ts: Claude API를 호출하고,
         stop_reason을 확인하고, 도구를 실행하고, 결과를 붙이는 while-true 루프.

    s02  TOOL DISPATCH        "도구 추가 = 핸들러 하나 추가"
         Tool.ts + tools.ts: 모든 도구가 dispatch 맵에 등록됩니다.
         루프는 그대로 유지됩니다. buildTool() 팩토리가
         안전한 기본값을 제공합니다.

    s03  PLANNING             "계획 없는 에이전트는 표류한다"
         EnterPlanModeTool/ExitPlanModeTool + TodoWriteTool:
         먼저 단계를 나열한 뒤 실행합니다. 완료율이 두 배로 높아집니다.

    s04  SUB-AGENTS           "큰 작업은 쪼개고, 하위 작업은 깨끗한 컨텍스트로"
         AgentTool + forkSubagent.ts: 각 자식은 새로운 messages[]를 받아
         메인 대화를 깔끔하게 유지합니다.

    s05  KNOWLEDGE ON DEMAND  "필요할 때 지식을 불러온다"
         SkillTool + memdir/: 시스템 프롬프트가 아니라 tool_result로 주입합니다.
         CLAUDE.md 파일은 디렉터리별로 지연 로드됩니다.

    s06  CONTEXT COMPRESSION  "컨텍스트는 가득 찬다. 공간을 만들어라"
         services/compact/: 3단계 전략:
         autoCompact (요약) + snipCompact (잘라내기) + contextCollapse

    s07  PERSISTENT TASKS     "큰 목표 → 작은 작업 → 디스크"
         TaskCreate/Update/Get/List: 상태 추적, 의존성,
         영속화를 갖춘 파일 기반 태스크 그래프.

    s08  BACKGROUND TASKS     "느린 작업은 백그라운드에서, 에이전트는 계속 생각한다"
         DreamTask + LocalShellTask: 데몬 스레드가 명령을 실행하고,
         완료 시 알림을 주입합니다.

    s09  AGENT TEAMS          "혼자 하기엔 크면 팀메이트에게 위임"
         TeamCreate/Delete + InProcessTeammateTask: 비동기 메일박스를 가진
         영속 팀메이트.

    s10  TEAM PROTOCOLS       "공유된 소통 규칙"
         SendMessageTool: 하나의 요청-응답 패턴으로
         에이전트 간 모든 협상을 처리합니다.

    s11  AUTONOMOUS AGENTS    "팀메이트가 스스로 작업을 찾아 선점한다"
         coordinator/coordinatorMode.ts: idle cycle + auto-claim,
         리드가 일일이 과제를 배정할 필요가 없습니다.

    s12  WORKTREE ISOLATION   "각자 자기 디렉터리에서 작업한다"
         EnterWorktreeTool/ExitWorktreeTool: task는 목표를 관리하고,
         worktree는 디렉터리를 관리하며, 둘은 ID로 연결됩니다.
```

---

## 핵심 설계 패턴

| 패턴 | 위치 | 목적 |
|------|------|------|
| **AsyncGenerator streaming** | `QueryEngine`, `query()` | API에서 소비자까지 전체 체인을 스트리밍 |
| **Builder + Factory** | `buildTool()` | 도구 정의에 안전한 기본값 제공 |
| **Branded Types** | `SystemPrompt`, `asSystemPrompt()` | 문자열/배열 혼동 방지 |
| **Feature Flags + DCE** | `feature()` from `bun:bundle` | 컴파일 타임 dead code elimination |
| **Discriminated Unions** | `Message` types | 타입 안전한 메시지 처리 |
| **Observer + State Machine** | `StreamingToolExecutor` | 도구 실행 라이프사이클 추적 |
| **Snapshot State** | `FileHistoryState` | 파일 작업의 undo/redo |
| **Ring Buffer** | 오류 로그 | 긴 세션에서 메모리 사용량 제한 |
| **Fire-and-Forget Write** | `recordTranscript()` | 순서를 유지하는 비차단 영속화 |
| **Lazy Schema** | `lazySchema()` | 성능을 위해 Zod 스키마 평가 지연 |
| **Context Isolation** | `AsyncLocalStorage` | 공유 프로세스 내 에이전트별 컨텍스트 |

---

## 빌드 노트

이 소스는 이 저장소만으로는 **바로 컴파일할 수 없습니다**.

- `tsconfig.json`, 빌드 스크립트, Bun 번들러 설정이 누락되어 있습니다.
- `feature()` 호출은 Bun의 컴파일 타임 intrinsic이며 번들링 중에 해석됩니다.
- `MACRO.VERSION`은 빌드 시점에 주입됩니다.
- `process.env.USER_TYPE === 'ant'` 구간은 Anthropic 내부 전용입니다.
- 컴파일된 `cli.js`는 Node.js >= 18만 있으면 실행 가능한 독립형 12MB 번들입니다.
- 소스맵(`cli.js.map`, 60MB)은 디버깅을 위해 이 소스 파일들로 매핑됩니다.

**빌드 지침과 우회 방법은 [QUICKSTART_KR.md](QUICKSTART_KR.md)를 참고하세요.**

---

## 라이선스

이 저장소의 모든 소스 코드는 **Anthropic 및 Claude**의 저작권 보호를 받습니다. 이 저장소는 기술 연구 및 교육 목적으로만 제공됩니다. 전체 라이선스 조항은 원본 npm 패키지를 참고하세요.
