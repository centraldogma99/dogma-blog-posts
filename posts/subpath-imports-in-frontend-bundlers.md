---
title: 프론트엔드 프로젝트에서 Subpath Import 활용하기
description: TypeScript alias 대신 Node.js의 subpath import로 모듈 해석 통합하기
date: 2025-08-08
draft: false
tag:
  - Node.js
  - TypeScript
  - Bundler
---

프론트엔드 프로젝트의 규모가 커지면 우리는 어김없이 이런 import 문을 마주하게 됩니다.

```typescript
import BigButton from '../../../../common/components/Button';
```

끝없이 이어지는 `../` 릴레이는 코드의 가독성을 해칠 뿐만 아니라, 수정하기도 번거롭습니다.
많은 경우 이 문제를 해결하기 위해 TypeScript의 paths alias를 사용하지만, 이는 또 다른 문제를 낳습니다.

바로 TypeScript, Webpack, Vite, Jest... 우리가 사용하는 모든 도구에게 이 별칭을 하나하나 알려줘야 한다는 점입니다. 설정은 파편화되고, 잠재적인 오류의 원인이 됩니다.

이 글에서는 이 모든 문제를 Node.js의 표준 기능인 **Subpath Import**를 통해 '단 하나의 설정'으로 해결하는 방법을 소개하며 Subpath Import의 강력한 기능도 추가로 설명해 드리고자 합니다.

> 참고: 이 글은 Typescript + 번들러(vite, webpack 등)를 활용하는 일반적인 프론트엔드 환경을 가정합니다.

## Subpath Import란?

Subpath import는 Node.js가 제공하는 모듈 해석 기능으로, `package.json`의 `imports` 필드를 통해 프로젝트 내부 모듈에 대한 별칭(alias)을 정의할 수 있습니다. `#`으로 시작하는 특별한 import 경로를 사용합니다.

```javascript
// 기존 상대 경로
import Button from '../../../components/Button.tsx';

// Subpath import 사용
import Button from '#components/Button.tsx';
```

## 설정 방법

`package.json`에 `imports` 필드를 추가하여 subpath import를 구성할 수 있습니다.

```json
{
  "name": "my-app",
  "imports": {
    "#components/*": "./src/components/*",
    "#utils/*": "./src/utils/*",
    "#hooks/*": "./src/hooks/*",
    "#styles/*": "./src/styles/*"
    // ...
  }
}
```

`tsconfig.json`에서도 `moduleResolution` 설정을 해야 합니다.

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    // 필수는 아님, 하지만 설정하지 않을 경우
    // 파일 확장자를 'ts', 'tsx'가 아닌 'js', 'jsx'로 사용해야 함
    "allowImportingTsExtensions": true 
  }
}
```

## Subpath Import의 핵심 장점

### 1. 단일 원천(Single Source of Truth)으로 설정 간소화

TypeScript의 path alias를 사용할 때의 가장 큰 문제점은 **설정의 중복**입니다. TypeScript 컴파일러가 경로를 이해하도록 `tsconfig.json`에 설정하더라도, 실제 번들링 과정에서는 번들러가 이를 이해하지 못합니다.

#### 기존 방식의 문제점

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@components/*": ["./src/components/*"]
    }
  }
}
```

```javascript
// webpack.config.js
module.exports = {
  resolve: {
    alias: {
      '@components': path.resolve(__dirname, 'src/components')
    }
  }
};
```

```javascript
// vite.config.js
export default {
  resolve: {
    alias: [
      { find: '@', replacement: path.resolve(__dirname, 'src') },
      {
        find: '@components',
        replacement: path.resolve(__dirname, 'src/components'),
      },
    ]
  }
};
```

보시다시피 동일한 경로 매핑을 TypeScript, Webpack, Vite 등 각각의 도구에 중복으로 설정해야 합니다. 이는 다음과 같은 문제를 야기합니다:

- 설정 불일치로 인한 빌드 오류
- 새로운 경로 추가 시 여러 파일 수정 필요
- 번들러 변경 시 설정 마이그레이션 필요

이러한 문제를 해결해 주기 위해 `tsc-alias` 와 같은 라이브러리가 있기는 하지만, 별도의 의존성을 설치해야 하므로 완전한 해결책은 아닙니다.

#### Subpath Import로 해결

Subpath import는 Node.js 표준 기능이므로, 대부분의 현대적인 번들러들이 자동으로 지원합니다:

```json
// package.json - 이것 하나만 설정하면 됨!
{
  "imports": {
    "#components/*": "./src/components/*"
  }
}
```

Webpack 5, Vite, Rollup, esbuild 등 주요 번들러들은 `package.json`의 `imports` 필드를 자동으로 읽어 처리합니다. 별도의 번들러 설정이 필요 없습니다!

### 2. 실행 환경별 조건부 모듈 해석

Subpath import의 정말 강력한 기능 중 하나입니다. 바로 **실행 환경에 따라 다른 모듈을 로드할 수 있다**는 것입니다.

#### 환경별 분기 설정

```json
{
  "imports": {
    "#config": {
      "development": "./src/config/dev.ts",
      "production": "./src/config/prod.ts",
      "default": "./src/config/default.ts"
    },
    "#api/*": {
      "browser": "./src/api/client/*",
      "node": "./src/api/server/*",
      "default": "./src/api/universal/*"
    },
    "#polyfills": {
      "node": "./src/polyfills/node.ts",
      "browser": "./src/polyfills/browser.ts"
    }
  }
}
```

#### 실제 활용 예시

##### 환경에 따른 선택적 폴리필 적용
```typescript
// 이 코드는 환경에 따라 다른 모듈을 로드합니다
import config from '#config';
import { fetchData } from '#api/data.ts';
import '#polyfills';

// 브라우저에서는: ./src/api/client/data.ts
// Node.js에서는: ./src/api/server/data.ts
// 를 자동으로 로드
```

##### 테스트 환경에서 mock 모듈 자동 로드
```json
{
  "imports": {
    "#logger": {
      "production": "./src/utils/logger-prod.ts",
      "default": "./src/utils/logger-dev.ts"
    }
  }
}
```

##### 개발 환경과 운영 환경에서 동작 분기
```javascript
// 개발 환경: 상세한 로깅
// 프로덕션: 최소한의 로깅만
import logger from '#logger';

logger.debug('This will only appear in development');
```

### 3. 플랫폼 표준이 주는 안정성

Subpath import는 **Node.js 12.19.0**부터 지원되는 공식 기능입니다. 이는 단순한 컨벤션이나 서드파티 도구가 아닌, 플랫폼 차원의 표준입니다.

이것이 왜 중요할까요? 우리가 사용하는 대부분의 프론트엔드 도구들은 Node.js 위에서 동작합니다. Webpack, Vite, Rollup 같은 번들러들도, TypeScript 컴파일러도, 심지어 ESLint나 Prettier 같은 린터와 포매터도 모두 Node.js 환경에서 실행됩니다. 따라서 Node.js가 공식적으로 지원하는 기능이라는 것은 이러한 도구들이 자연스럽게 이를 지원하게 된다는 의미입니다.

실제로 TypeScript의 path alias는 TypeScript 컴파일러만의 기능이기 때문에, 다른 도구들은 이를 이해하지 못합니다. 반면 subpath import는 Node.js의 모듈 해석 시스템에 내장되어 있어, Node.js 위에서 동작하는 대부분의 도구가 이를 이해할 수 있습니다. 이는 곧 더 나은 호환성과 적은 설정 부담을 의미합니다.

또한 Node.js 팀이 직접 유지보수하는 기능이므로 장기적인 안정성이 보장됩니다. ECMAScript 모듈 시스템의 발전과 함께 지속적으로 개선되고 있으며, 새로운 JavaScript 표준이 등장하더라도 하위 호환성을 유지하면서 발전할 것입니다. 반면 서드파티 도구나 특정 번들러의 독자적인 기능은 언제든 deprecated되거나 breaking change가 발생할 수 있습니다.


## 주의사항

### 1. TS 5.4+에서 지원됨

### 2. import 문에서 확장자 명시 필요

import 경로에 파일 확장자를 반드시 명시해야 합니다:

```typescript
// ❌ 확장자 없이 사용 - 오류 발생
import Button from '#components/Button';
// ❌ barrel pattern(index.js) - 오류 발생
import foobar from '#components/foobar';

// ✅ 확장자 명시 - 정상 동작
import Button from '#components/Button.tsx';
// ✅ index.ts와 같은 index 파일명 명시 - 정상 동작
import foobar from '#components/foobar/index.ts';
```

확장자를 왜 넣어야 하는지에 대한 이유는 TypeScript 레포의 [이슈](https://github.com/microsoft/TypeScript/issues/60003#issuecomment-2364130579)를 참조하세요.

## TypeScript Path Alias에서 Subpath Import로 마이그레이션하기

기존 프로젝트에서 TypeScript path alias를 사용하고 있다면, 다음 단계를 따라 subpath import로 마이그레이션할 수 있습니다.

### 1단계: 현재 Path Alias 설정 확인

먼저 `tsconfig.json`에서 현재 사용 중인 path alias를 확인합니다:

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@hooks/*": ["src/hooks/*"],
      "@services/*": ["src/services/*"],
      "@/*": ["src/*"]
    }
  }
}
```

### 2단계: package.json에 imports 필드 추가

TypeScript path alias를 subpath import로 변환합니다. `@` 대신 `#`을 사용하고, 경로를 `./`로 시작하도록 수정합니다:

```json
// package.json
{
  "imports": {
    "#components/*": "./src/components/*",
    "#utils/*": "./src/utils/*",
    "#hooks/*": "./src/hooks/*",
    "#services/*": "./src/services/*",
    "#*": "./src/*"
  }
}
```

### 3단계: TypeScript 설정 업데이트

`tsconfig.json`을 수정하여 subpath import를 지원하도록 설정합니다:

```json
// tsconfig.json
{
  "compilerOptions": {
    // path alias 관련 설정 제거
    // "baseUrl": ".",
    // "paths": { ... }
    
    // subpath import 지원 설정 추가
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true
  }
}
```

### 4단계: 번들러 설정 정리

기존 번들러 설정에서 alias 관련 설정을 제거할 수 있습니다:

```javascript
// vite.config.js
export default {
  resolve: {
    // 이제 불필요한 alias 설정 제거
    // alias: [
    //   { find: '@', replacement: path.resolve(__dirname, 'src') },
    //   {
    //     find: '@components',
    //     replacement: path.resolve(__dirname, 'src/components'),
    //   },
    // ]
  }
};
```

Vite가 아닌 다른 번들러에서도 비슷한 작업을 통해 제거할 수 있습니다.

### 5단계: Import 문 업데이트

프로젝트 전체의 import 문을 업데이트합니다. 주요 변경사항:
- `@` → `#`으로 변경
- 파일 확장자 추가 (`.ts`, `.tsx`, `.js`, `.jsx`)

```typescript
// 변경 전
import Button from '@components/Button';
import { useAuth } from '@hooks/useAuth';
import { apiClient } from '@services/api';

// 변경 후
import Button from '#components/Button.tsx';
import { useAuth } from '#hooks/useAuth.ts';
import { apiClient } from '#services/api/index.ts';
```
