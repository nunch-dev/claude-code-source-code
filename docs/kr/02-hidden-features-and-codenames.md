# 숨겨진 기능과 모델 코드명

> Claude Code v2.1.88 역컴파일 소스 코드 분석 기반.

## 모델 코드명 시스템

Anthropic은 내부 모델 코드명으로 **동물 이름**을 사용합니다. 이 코드명은 외부 빌드로 새어나가지 않도록 매우 강하게 보호됩니다.

### 알려진 코드명

| 코드명 | 역할 | 근거 |
|--------|------|------|
| **Tengu** | 제품/텔레메트리 접두사, 또는 모델일 가능성 | 250개 이상의 analytics 이벤트와 feature flag 전부에 `tengu_*` 접두사 사용 |
| **Capybara** | Sonnet 계열 모델, 현재 v8 | `capybara-v2-fast[1m]`, v8 행동 문제 대응용 프롬프트 패치 |
| **Fennec** | Opus 4.6의 이전 이름 | 마이그레이션: `fennec-latest` → `opus` |
| **Numbat**  | 다음 모델 출시명 | 주석: "numbat를 출시하면 이 섹션을 제거하라" |

### 코드명 보호

`undercover` 모드는 보호 대상 코드명을 명시적으로 나열합니다.

```typescript
// src/utils/undercover.ts:48-49
NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
```

빌드 시스템은 `scripts/excluded-strings.txt`를 이용해 유출된 코드명을 검사합니다. buddy 시스템 종 이름 하나는 canary를 피하기 위해 `String.fromCharCode()`로 구성됩니다.

```typescript
// src/buddy/types.ts:10-13
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value keeps
// the literal out of the bundle while the check stays armed for the actual codename.
```

그 충돌하는 종 이름은 **capybara**입니다. 애완동물 종 이름이면서 모델 코드명이기도 합니다.

### Capybara v8 행동 문제

소스 코드는 Capybara v8의 구체적인 행동 문제를 드러냅니다.

1. **stop sequence 오탐지** (`<functions>`가 프롬프트 끝에 있을 때 약 10%)
   - 출처: `src/utils/messages.ts:2141`

2. **빈 tool_result가 출력 0건을 유발**
   - 출처: `src/utils/toolResultStorage.ts:281`

3. **과도한 주석 작성** — 별도의 anti-comment 프롬프트 패치 필요
   - 출처: `src/constants/prompts.ts:204`

4. **높은 false-claim 비율**: v8은 29~30%, v4는 16.7%
   - 출처: `src/constants/prompts.ts:237`

5. **검증 부족** — "thoroughness counterweight" 필요
   - 출처: `src/constants/prompts.ts:210`

## Feature Flag 네이밍 규칙

모든 feature flag는 목적을 숨기기 위해 무작위 단어쌍과 함께 `tengu_` 접두사를 사용합니다.

| 플래그 | 목적 |
|--------|------|
| `tengu_onyx_plover` | Auto Dream (백그라운드 메모리 통합) |
| `tengu_coral_fern` | memdir 기능 |
| `tengu_moth_copse` | 또 다른 memdir 스위치 |
| `tengu_herring_clock` | Team memory |
| `tengu_passport_quail` | Path 기능 |
| `tengu_slate_thimble` | 또 다른 memdir 스위치 |
| `tengu_sedge_lantern` | Away Summary |
| `tengu_frond_boric` | Analytics kill switch |
| `tengu_amber_quartz_disabled` | Voice mode kill switch |
| `tengu_amber_flint` | Agent teams |
| `tengu_hive_evidence` | Verification agent |

이 무작위 단어 패턴(형용사/재료 + 자연/사물)은 외부 관찰자가 플래그 이름만 보고 기능 목적을 추론하지 못하게 만듭니다.

## 내부 사용자와 외부 사용자의 차이

Anthropic 직원(`USER_TYPE === 'ant'`)은 눈에 띄게 더 좋은 대우를 받습니다.

### 프롬프트 차이 (`src/constants/prompts.ts`)

| 항목 | 외부 사용자 | 내부 사용자 (ant) |
|------|-------------|-------------------|
| 출력 스타일 | "Be extra concise" | "Err on the side of more explanation" |
| false-claim 완화 | 없음 | Capybara v8 전용 패치 |
| 숫자 길이 anchor | 없음 | "도구 사이 ≤25단어, 최종 ≤100단어" |
| 검증 에이전트 | 없음 | 사소하지 않은 변경에 필수 |
| 주석 가이드 | 일반적 | 과도한 주석 억제 전용 프롬프트 |
| 선제 교정 | 없음 | "사용자가 오해했으면 그렇게 말하라" |

### 도구 접근 권한

내부 사용자는 외부에 없는 도구에 접근할 수 있습니다.
- `REPLTool` — REPL 모드
- `SuggestBackgroundPRTool` — 백그라운드 PR 제안
- `TungstenTool` — 성능 모니터링 패널
- `VerifyPlanExecutionTool` — 계획 실행 검증
- 에이전트 중첩 (에이전트가 또 다른 에이전트를 생성)

## 숨겨진 명령어

| 명령어 | 상태 | 설명 |
|--------|------|------|
| `/btw` | Active | 흐름을 끊지 않고 옆가지 질문하기 |
| `/stickers` | Active | Claude Code 스티커 주문하기 (브라우저 열기) |
| `/thinkback` | Active | 2025 Year in Review |
| `/effort` | Active | 모델 effort 수준 설정 |
| `/good-claude` | Stub | 숨겨진 placeholder |
| `/bughunter` | Stub | 숨겨진 placeholder |
