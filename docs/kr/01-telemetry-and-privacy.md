# 텔레메트리와 프라이버시 분석

> Claude Code v2.1.88 역컴파일 소스 코드 분석 기반.

## 개요

Claude Code는 광범위한 환경 정보와 사용 메타데이터를 수집하는 2단계 분석 파이프라인을 구현하고 있습니다. 키로깅이나 소스 코드 유출의 증거는 없지만, 수집 범위가 넓고 완전히 opt-out 할 수 없다는 점은 정당한 프라이버시 우려를 낳습니다.

## 데이터 파이프라인 아키텍처

### 퍼스트파티 로깅 (1P)

- **엔드포인트**: `https://api.anthropic.com/api/event_logging/batch`
- **프로토콜**: Protocol Buffers 기반 OpenTelemetry
- **배치 크기**: 배치당 최대 200개 이벤트, 10초마다 flush
- **재시도**: 이차(backoff) 재시도, 최대 8회, 내구성을 위해 디스크에 저장
- **저장 위치**: 실패한 이벤트는 `~/.claude/telemetry/`에 저장

출처: `src/services/analytics/firstPartyEventLoggingExporter.ts`

### 서드파티 로깅 (Datadog)

- **엔드포인트**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **범위**: 사전 승인된 64개 이벤트 타입으로 제한
- **토큰**: `pubbbf48e6d78dae54bceaa4acf463299bf`

출처: `src/services/analytics/datadog.ts`

## 무엇을 수집하는가

### 환경 지문

모든 이벤트에는 다음 메타데이터가 포함됩니다 (`src/services/analytics/metadata.ts:417-452`).

```
- platform, platformRaw, arch, nodeVersion
- terminal type
- 설치된 패키지 매니저 및 런타임
- CI/CD 감지, GitHub Actions 메타데이터
- WSL 버전, Linux 배포판, 커널 버전
- VCS(버전 관리 시스템) 유형
- Claude Code 버전 및 빌드 시각
- 배포 환경
```

### 프로세스 메트릭 (`metadata.ts:457-467`)

```
- uptime, rss, heapTotal, heapUsed
- CPU 사용량 및 퍼센트
- 메모리 배열과 external allocation
```

### 사용자 추적 (`metadata.ts:472-496`)

```
- 사용 중인 모델
- session ID, user ID, device ID
- account UUID, organization UUID
- 구독 등급 (max, pro, enterprise, team)
- 저장소 원격 URL 해시 (SHA256, 앞 16자)
- agent type, team name, parent session ID
```

### 도구 입력 로깅

기본적으로 도구 입력은 잘려서 기록됩니다.

```
- 문자열: 512자에서 잘리고, 표시 시에는 128자 + 생략 부호
- JSON: 4,096자로 제한
- 배열: 최대 20개 항목
- 중첩 객체: 최대 2단계 깊이
```

출처: `metadata.ts:236-241`

하지만 `OTEL_LOG_TOOL_DETAILS=1`이 설정되면 **도구 입력 전체가 기록됩니다.**

출처: `metadata.ts:86-88`

### 파일 확장자 추적

`rm, mv, cp, touch, mkdir, chmod, chown, cat, head, tail, sort, stat, diff, wc, grep, rg, sed`가 포함된 Bash 명령은 파일 인자의 확장자를 추출해 기록합니다.

출처: `metadata.ts:340-412`

## Opt-Out 문제

Anthropic 직접 API 사용자의 경우, 퍼스트파티 로깅 파이프라인은 **비활성화할 수 없습니다.**

```typescript
// src/services/analytics/firstPartyEventLogger.ts:141-144
export function is1PEventLoggingEnabled(): boolean {
  return !isAnalyticsDisabled()
}
```

`isAnalyticsDisabled()`가 true를 반환하는 경우는 다음뿐입니다.
- 테스트 환경
- 서드파티 클라우드 제공자(Bedrock, Vertex)
- 글로벌 텔레메트리 opt-out (설정 UI에는 노출되지 않음)

즉, 퍼스트파티 이벤트 로깅을 끄는 **사용자 노출 설정은 존재하지 않습니다.**

## GrowthBook A/B 테스트

사용자는 명시적 동의 없이 GrowthBook 실험 그룹에 배정됩니다. 이때 시스템은 다음 사용자 속성을 전송합니다.

```
- id, sessionId, deviceID
- platform, organizationUUID, subscriptionType
```

출처: `src/services/analytics/growthbook.ts`

## 핵심 요약

1. **수집량**: 세션당 수백 개의 이벤트가 수집됩니다.
2. **Opt-out 부재**: 직접 API 사용자는 퍼스트파티 로깅을 끌 수 없습니다.
3. **영속성**: 실패한 이벤트는 디스크에 저장되고 공격적으로 재시도됩니다.
4. **서드파티 공유**: 데이터는 Datadog로도 전달됩니다.
5. **도구 상세 백도어**: `OTEL_LOG_TOOL_DETAILS=1`은 전체 입력 로깅을 활성화합니다.
6. **저장소 지문 채취**: 저장소 URL은 해시 처리되어 서버 측 상관 분석에 사용됩니다.
