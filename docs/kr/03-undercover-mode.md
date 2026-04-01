# 위장 모드 분석

> Claude Code v2.1.88 역컴파일 소스 코드 분석 기반.

## 위장 모드란 무엇인가?

위장 모드는 Anthropic 직원이 공개/오픈소스 저장소에 기여할 때 동작하는 안전장치입니다. 활성화되면 AI 관련 attribution을 모두 제거하고, 모델이 자신의 기여를 인간 개발자가 쓴 것처럼 보이도록 지시합니다.

출처: `src/utils/undercover.ts`

## 활성화 로직

```typescript
// src/utils/undercover.ts:28-37
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true
    // Auto: active unless positively confirmed internal repo
    return getRepoClassCached() !== 'internal'
  }
  return false
}
```

핵심 속성:
- **내부 전용**: Anthropic 직원(`USER_TYPE === 'ant'`)에게만 활성화됩니다.
- **기본값 ON**: 내부 허용 목록에 있는 저장소가 아닌 모든 곳에서 기본 활성화됩니다.
- **강제 OFF 불가**: "강제로 끄는 기능은 없다. 모델 코드명 유출 방지를 위한 장치다"
- **외부 빌드**: 번들러에 의해 dead-code-elimination 되므로 실제 외부 빌드에서는 실행되지 않습니다.

## 모델에 주어지는 프롬프트

```typescript
// src/utils/undercover.ts:39-69
export function getUndercoverInstructions(): string {
  return `## UNDERCOVER MODE — CRITICAL

You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository. Your commit
messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal
information. Do not blow your cover.

NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
- Internal repo or project names (e.g., claude-cli-internal, anthropics/…)
- Internal tooling, Slack channels, or short links (e.g., go/cc, #claude-code-…)
- The phrase "Claude Code" or any mention that you are an AI
- Any hint of what model or version you are
- Co-Authored-By lines or any other attribution

Write commit messages as a human developer would — describe only what the code
change does.

GOOD:
- "Fix race condition in file watcher initialization"
- "Add support for custom key bindings"

BAD (never write these):
- "Fix bug found while testing with Claude Capybara"
- "1-shotted by claude-opus-4-6"
- "Generated with Claude Code"
- "Co-Authored-By: Claude Opus 4.6 <…>"`
}
```

## Attribution 시스템

attribution 시스템(`src/utils/attribution.ts`, `src/utils/commitAttribution.ts`)은 위장 모드를 보완합니다.

```typescript
// src/utils/attribution.ts:70-72
// @[MODEL LAUNCH]: Update the hardcoded fallback model name below
// (guards against codename leaks).
// For external repos, fall back to "Claude Opus 4.6" for unrecognized models.
```

```typescript
// src/utils/model/model.ts:386-392
function maskModelCodename(baseName: string): string {
  // e.g. capybara-v2-fast → cap*****-v2-fast
  const [codename = '', ...rest] = baseName.split('-')
  const masked = codename.slice(0, 3) + '*'.repeat(Math.max(0, codename.length - 3))
  return [masked, ...rest].join('-')
}
```

## 함의

### 오픈소스에 대한 영향

Anthropic 직원이 Claude Code를 사용해 오픈소스 프로젝트에 기여할 때:
1. 코드는 AI가 작성하지만 커밋은 사람이 작성한 것처럼 보입니다.
2. `Co-Authored-By: Claude` 같은 attribution이 남지 않습니다.
3. `Generated with Claude Code` 같은 표식도 없습니다.
4. 프로젝트 유지관리자와 커뮤니티는 AI 생성 기여를 식별할 수 없습니다.
5. 이는 AI 기여에 대한 오픈소스 투명성 규범과 충돌할 수 있습니다.

### Anthropic 보호 관점

명시된 주된 목적은 다음의 우발적 유출을 막는 것입니다.
- 내부 모델 코드명(경쟁 정보)
- 미출시 버전 번호(시장 타이밍)
- 내부 인프라 세부 사항(보안)

### 윤리적 고려 사항

"Do not blow your cover"라는 표현은 AI를 잠입 요원처럼 취급합니다. 공개 코드 기여에서 AI 작성 사실을 의도적으로 숨기는 것은 다음과 같은 질문을 제기합니다.
- 오픈소스 커뮤니티의 투명성
- 프로젝트 기여 가이드라인 준수 여부
- 영업 비밀 보호와 기만 사이의 경계
