# Quick Start — 소스에서 빌드하기

> **요약**: 완전한 재빌드를 하려면 **Node.js가 아니라 Bun**이 필요합니다. 이유는 `feature()`, `MACRO`, `bun:bundle` 같은 컴파일 타임 intrinsic을 사용하기 때문입니다. esbuild로도 최선을 다한 빌드는 가능하지만 약 95% 수준이며, feature gate로 숨겨진 약 108개 모듈은 수동 보정이 필요합니다.

## 옵션 A: 미리 빌드된 CLI 실행하기 (권장)

npm 패키지에는 이미 컴파일된 `cli.js`가 들어 있습니다.

```bash
cd /path/to/parent/            # package.json과 cli.js가 있는 위치
node cli.js --version           # → 2.1.88 (Claude Code)
node cli.js -p "Hello Claude"   # 비대화형 모드

# 또는 전역 설치:
npm install -g .
claude --version
```

**인증 필요**: `ANTHROPIC_API_KEY`를 설정하거나 `node cli.js login`을 실행하세요.

## 옵션 B: 소스에서 빌드하기 (Best Effort)

### 사전 요구 사항

```bash
node --version   # >= 18
npm --version    # >= 9
```

### 단계

```bash
cd claude-code-2.1.88/

# 1. 빌드 의존성 설치
npm install --save-dev esbuild

# 2. 빌드 스크립트 실행
node scripts/build.mjs

# 3. 성공하면 출력물 실행
node dist/cli.js --version
```

### 빌드 스크립트가 하는 일

| 단계 | 동작 |
|------|------|
| **1. Copy** | `src/` → `build-src/` 복사 (원본은 그대로 유지) |
| **2. Transform** | `feature('X')` → `false` 로 변환 (dead code elimination 유도) |
| **2b. Transform** | `MACRO.VERSION` → `'2.1.88'` 로 치환 (컴파일 타임 버전 주입) |
| **2c. Transform** | `import from 'bun:bundle'` → stub import 로 교체 |
| **3. Entry** | MACRO 전역값을 주입하는 래퍼 생성 |
| **4. Bundle** | 누락된 모듈에 대한 반복적 stub 생성과 함께 esbuild 수행 |

### 알려진 문제

이 소스 코드는 esbuild로 완전히 재현할 수 없는 **Bun 컴파일 타임 intrinsic**을 사용합니다.

1. **`bun:bundle`의 `feature('FLAG')`** — Bun은 이를 컴파일 타임에 `true`/`false`로 해석하고 죽은 분기를 제거합니다. 여기서는 `false`로 치환하지만, esbuild는 그 분기 안의 `require()`도 여전히 해석합니다.

2. **`MACRO.X`** — Bun의 `--define`은 이를 컴파일 타임에 치환합니다. 여기서는 문자열 치환으로 처리하는데, 대부분은 동작하지만 복잡한 표현식에서는 일부 edge case를 놓칠 수 있습니다.

3. **누락된 108개 모듈** — 데몬, bridge assistant, context collapse 등 feature gate 내부 모듈이 공개 소스에는 없습니다. Bun은 보통 이를 dead-code-elimination으로 제거하지만, esbuild는 `require()` 호출이 문법적으로 남아 있어 제거하지 못합니다.

4. **`bun:ffi`** — 네이티브 프록시 지원에 쓰이며, 여기서는 stub 처리됩니다.

5. **생성된 파일에서의 TypeScript `import type`** — 일부 생성된 타입 파일이 공개 소스에 포함되어 있지 않습니다.

### 남은 문제를 고치려면

```bash
# 1. 아직 무엇이 빠졌는지 확인:
npx esbuild build-src/entry.ts --bundle --platform=node \
  --packages=external --external:'bun:*' \
  --log-level=error --log-limit=0 --outfile=/dev/null 2>&1 | \
  grep "Could not resolve" | sort -u

# 2. 누락된 각 모듈에 대해 build-src/src/ 아래에 stub 생성:
#    JS/TS: 빈 함수 export 파일 생성
#    텍스트: 빈 파일 생성

# 3. 다시 실행:
node scripts/build.mjs
```

## 옵션 C: Bun으로 빌드하기 (완전 재빌드 — 내부 접근 필요)

```bash
# Bun 설치
curl -fsSL https://bun.sh/install | bash

# 실제 빌드는 Bun 번들러와 컴파일 타임 feature flag를 사용합니다:
# bun build src/entrypoints/cli.tsx \
#   --define:feature='(flag) => flag === "SOME_FLAG"' \
#   --define:MACRO.VERSION='"2.1.88"' \
#   --target=bun \
#   --outfile=dist/cli.js

# 하지만 내부 빌드 설정은 공개 패키지에 포함되어 있지 않습니다.
# Anthropic 내부 저장소에 접근할 수 있어야 합니다.
```

## 프로젝트 구조

```
claude-code-2.1.88/
├── src/                  # 원본 TypeScript 소스 (1,884 files, 512K LOC)
├── stubs/                # Bun 컴파일 타임 intrinsic용 빌드 stub
│   ├── bun-bundle.ts     #   feature() stub → 항상 false 반환
│   ├── macros.ts         #   MACRO 버전 상수
│   └── global.d.ts       #   전역 타입 선언
├── scripts/
│   └── build.mjs         # esbuild 기반 빌드 스크립트
├── node_modules/         # 192개 npm 의존성
├── vendor/               # 네이티브 모듈 소스 stub
├── build-src/            # 빌드 스크립트가 생성하는 변환 복사본
└── dist/                 # 빌드 출력물 (빌드 스크립트가 생성)
```
