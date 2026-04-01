# 미래 로드맵 — 소스 코드가 드러내는 것

> Claude Code v2.1.88 역컴파일 소스 코드 분석 기반.

## 1. 다음 모델: Numbat

다음 모델 출시를 보여주는 가장 직접적인 증거는 다음과 같습니다.

```typescript
// src/constants/prompts.ts:402
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

**Numbat** (袋食蚁兽)는 출시 예정 모델의 코드명입니다. 이 주석은 Numbat 출시 시 출력 효율 섹션이 수정될 것임을 시사하며, 이는 더 나은 네이티브 출력 제어 능력을 가질 가능성을 암시합니다.

### 향후 버전 번호

```typescript
// src/utils/undercover.ts:49
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
```

즉, **Opus 4.7**과 **Sonnet 4.8**이 개발 중입니다.

### 코드명 진화 체인

```
Fennec (耳廓狐) → Opus 4.6 → [Numbat?]
Capybara (水豚) → Sonnet v8 → [?]
Tengu (天狗) → telemetry/product prefix
```

Fennec에서 Opus로의 마이그레이션은 다음과 같이 문서화되어 있습니다.

```typescript
// src/migrations/migrateFennecToOpus.ts:7-11
// fennec-latest → opus
// fennec-latest[1m] → opus[1m]
// fennec-fast-latest → opus[1m] + fast mode
```

### MODEL LAUNCH 체크리스트

코드베이스에는 20개 이상의 `@[MODEL LAUNCH]` 마커가 있어, 모델 출시 시 무엇을 바꿔야 하는지 나열합니다.

- 기본 모델 이름 (`FRONTIER_MODEL_NAME`)
- 모델 패밀리 ID
- 지식 cutoff 날짜
- 가격 테이블
- 컨텍스트 윈도 설정
- thinking mode 지원 플래그
- 표시 이름 매핑
- 마이그레이션 스크립트

## 2. KAIROS — 자율 에이전트 모드

가장 큰 미공개 기능인 KAIROS는 Claude Code를 반응형 비서에서 선제적 자율 에이전트로 바꾸려 합니다.

### 시스템 프롬프트 (발췌)

```
// src/constants/prompts.ts:860-913

You are running autonomously.
You will receive <tick> prompts that keep you alive between turns.
If you have nothing useful to do, call SleepTool.
Bias toward action — read files, make changes, commit without asking.

## Terminal focus
- Unfocused: The user is away. Lean heavily into autonomous action.
- Focused: The user is watching. Be more collaborative.
```

### 관련 도구

| 도구 | Feature Flag | 목적 |
|------|--------------|------|
| SleepTool | KAIROS / PROACTIVE | 자율 행동 사이의 속도 조절 |
| SendUserFileTool | KAIROS | 파일을 사용자에게 선제적으로 전송 |
| PushNotificationTool | KAIROS / KAIROS_PUSH_NOTIFICATION | 사용자 기기에 푸시 알림 전송 |
| SubscribePRTool | KAIROS_GITHUB_WEBHOOKS | GitHub PR 웹훅 이벤트 구독 |
| BriefTool | KAIROS_BRIEF | 선제적 상태 업데이트 |

### 동작 특성

- `<tick>` 하트비트 프롬프트에 따라 동작
- 터미널 포커스 상태에 따라 자율성 조정
- 커밋, 푸시, 의사결정을 독립적으로 수행 가능
- 선제적 알림과 상태 업데이트 전송
- GitHub PR의 변화를 모니터링

## 3. 음성 모드

Push-to-talk 음성 입력은 이미 구현되어 있지만 `VOICE_MODE` feature flag 뒤에 숨겨져 있습니다.

```typescript
// src/voice/voiceModeEnabled.ts
// Connects to Anthropic's voice_stream WebSocket endpoint
// Uses conversation_engine backed models for speech-to-text
// Hold-to-talk: hold keybinding to record, release to submit
```

- OAuth 전용 (API key / Bedrock / Vertex 미지원)
- WebSocket 연결에 mTLS 사용
- 킬스위치: `tengu_amber_quartz_disabled`

## 4. 미공개 도구

소스에는 존재하지만 외부 사용자에게 아직 활성화되지 않은 도구들입니다.

| 도구 | Feature Flag | 설명 |
|------|--------------|------|
| **WebBrowserTool** | `WEB_BROWSER_TOOL` | 내장 브라우저 자동화 (코드명: bagel) |
| **TerminalCaptureTool** | `TERMINAL_PANEL` | 터미널 패널 캡처 및 모니터링 |
| **WorkflowTool** | `WORKFLOW_SCRIPTS` | 미리 정의된 워크플로 스크립트 실행 |
| **MonitorTool** | `MONITOR_TOOL` | 시스템/프로세스 모니터링 |
| **SnipTool** | `HISTORY_SNIP` | 대화 히스토리 절단/축약 |
| **ListPeersTool** | `UDS_INBOX` | Unix domain socket peer 탐색 |
| **RemoteTriggerTool** | `AGENT_TRIGGERS_REMOTE` | 원격 에이전트 트리거 |
| **TungstenTool** | ant-only | 내부 성능 모니터링 패널 |
| **VerifyPlanExecutionTool** | VERIFY_PLAN env | 계획 실행 검증 |
| **OverflowTestTool** | `OVERFLOW_TEST_TOOL` | 컨텍스트 overflow 테스트 |
| **SubscribePRTool** | `KAIROS_GITHUB_WEBHOOKS` | GitHub PR 웹훅 구독 |

## 5. Coordinator Mode

멀티 에이전트 조정 시스템입니다.

```typescript
// src/coordinator/coordinatorMode.ts
// Feature flag: COORDINATOR_MODE
```

공유 상태와 메시징을 기반으로 여러 에이전트가 협력해 작업을 실행할 수 있게 합니다.

## 6. Buddy 시스템 (가상 펫)

완전한 펫 동반자 시스템이 구현되어 있지만 아직 출시되지 않았습니다.

- **18종의 종**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- **5개 희귀도 단계**: Common (60%), Uncommon (25%), Rare (10%), Epic (4%), Legendary (1%)
- **7개의 모자**: crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **5개 스탯**: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- **1% shiny 확률**: 모든 종에 반짝이 변형 존재
- **결정적 생성**: user ID 해시 기반

출처: `src/buddy/`

## 7. Dream Task

백그라운드 메모리 통합 서브에이전트입니다.

```
// src/tasks/DreamTask/
// Auto-dreaming feature that works in the background
// Controlled by 'tengu_onyx_plover' feature flag
```

유휴 시간 동안 AI가 자율적으로 메모리를 처리하고 통합할 수 있게 합니다.

## 요약: 세 가지 방향성

1. **새 모델**: Numbat(다음), Opus 4.7, Sonnet 4.8 개발 중
2. **자율 에이전트**: KAIROS 모드 — 무인 동작, 선제적 행동, 푸시 알림
3. **멀티모달**: 음성 입력 준비 완료, 브라우저 도구 대기 중, 워크플로 자동화 예정

Claude Code는 **코딩 보조 도구**에서 **항상 켜져 있는 자율 개발 에이전트**로 진화하고 있습니다.
