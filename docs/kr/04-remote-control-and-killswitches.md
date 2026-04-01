# 원격 제어와 킬스위치

> Claude Code v2.1.88 역컴파일 소스 코드 분석 기반.

## 개요

Claude Code는 Anthropic(및 엔터프라이즈 관리자)이 사용자 명시적 동의 없이 동작을 바꿀 수 있게 하는 여러 원격 제어 메커니즘을 구현하고 있습니다.

## 1. 원격 관리 설정

### 아키텍처

자격이 되는 모든 세션은 다음에서 설정을 가져옵니다.
```
GET /api/claude_code/settings
```

출처: `src/services/remoteManagedSettings/index.ts:105-107`

### 폴링 동작

```typescript
// src/services/remoteManagedSettings/index.ts:52-54
const SETTINGS_TIMEOUT_MS = 10000
const DEFAULT_MAX_RETRIES = 5
const POLLING_INTERVAL_MS = 60 * 60 * 1000 // 1 hour
```

설정은 1시간마다 폴링되며, 실패 시 최대 5회 재시도합니다.

### 적용 대상

- Console 사용자(API key): 모두 대상
- OAuth 사용자: Enterprise/C4E 및 Team 구독자만 대상

### 수락 아니면 종료 대화상자

원격 설정에 "위험한" 변경이 포함되면 차단 대화상자가 뜹니다.

```typescript
// src/services/remoteManagedSettings/securityCheck.tsx:67-73
export function handleSecurityCheckResult(result: SecurityCheckResult): boolean {
  if (result === 'rejected') {
    gracefulShutdownSync(1)  // Exit with code 1
    return false
  }
  return true
}
```

사용자가 원격 설정을 거부하면 애플리케이션은 **강제로 종료됩니다.** 선택지는 둘뿐입니다. 원격 설정을 수락하거나, Claude Code를 종료당하거나.

### 점진적 저하(Graceful Degradation)

원격 서버에 도달할 수 없으면 디스크에 캐시된 설정을 사용합니다.

```typescript
// src/services/remoteManagedSettings/index.ts:433-436
if (cachedSettings) {
  logForDebugging('Remote settings: Using stale cache after fetch failure')
  setSessionCache(cachedSettings)
  return cachedSettings
}
```

한 번 원격 설정이 적용되면, 서버가 내려가도 그 설정은 계속 유지됩니다.

## 2. Feature Flag 킬스위치

여러 기능은 GrowthBook feature flag를 통해 원격으로 비활성화될 수 있습니다.

### 권한 우회 킬스위치

```typescript
// src/utils/permissions/bypassPermissionsKillswitch.ts
// Checks a Statsig gate to disable bypass permissions
```

사용자 동의 없이 권한 우회 기능을 끌 수 있습니다.

### Auto Mode 회로 차단기

```typescript
// src/utils/permissions/autoModeState.ts
// autoModeCircuitBroken state prevents re-entry to auto mode
```

Auto mode를 원격으로 비활성화할 수 있습니다.

### Fast Mode 킬스위치

```typescript
// src/utils/fastMode.ts
// Fetches from /api/claude_code_penguin_mode
// Can permanently disable fast mode for a user
```

### Analytics Sink 킬스위치

```typescript
// src/services/analytics/sinkKillswitch.ts:4
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'
```

모든 analytics 출력을 원격으로 중단시킬 수 있습니다.

### Agent Teams 킬스위치

```typescript
// src/utils/agentSwarmsEnabled.ts
// Requires both env var AND GrowthBook gate 'tengu_amber_flint'
```

### Voice Mode 킬스위치

```typescript
// src/voice/voiceModeEnabled.ts:21
// 'tengu_amber_quartz_disabled' — emergency off for voice mode
```

## 3. 모델 override 시스템

Anthropic은 내부 직원이 사용하는 모델을 원격으로 override 할 수 있습니다.

```typescript
// src/utils/model/antModels.ts:32-33
// @[MODEL LAUNCH]: Update tengu_ant_model_override with new ant-only models
// @[MODEL LAUNCH]: Add the codename to scripts/excluded-strings.txt
```

`tengu_ant_model_override` GrowthBook 플래그는 다음을 할 수 있습니다.
- 기본 모델 지정
- 기본 effort 수준 지정
- 시스템 프롬프트에 내용 추가
- 사용자 정의 모델 alias 정의

## 4. Penguin Mode

Fast mode 상태는 전용 엔드포인트에서 가져옵니다.

```typescript
// src/utils/fastMode.ts
// GET /api/claude_code_penguin_mode
// If API indicates disabled, permanently disabled for user
```

여러 feature flag가 fast mode 사용 가능 여부를 제어합니다.
- `tengu_penguins_off`
- `tengu_marble_sandcastle`

## 요약

| 메커니즘 | 범위 | 사용자 동의 |
|----------|------|-------------|
| 원격 관리 설정 | Enterprise/Team | 수락 아니면 종료 |
| GrowthBook feature flag | 전체 사용자 | 없음 |
| 킬스위치 | 전체 사용자 | 없음 |
| 모델 override | 내부 사용자 (ant) | 없음 |
| Fast mode 제어 | 전체 사용자 | 없음 |

원격 제어 인프라는 광범위하며, 대부분 사용자 가시성이나 동의 없이 작동합니다. 엔터프라이즈 관리자는 사용자가 재정의할 수 없는 정책을 강제할 수 있고, Anthropic은 feature flag를 통해 어떤 사용자든 원격으로 동작을 바꿀 수 있습니다.
