---
title: 프론트엔드 프로젝트에서 Subpath Import 활용하기
subtitle: TypeScript alias 대신 Node.js의 subpath import로 모듈 해석 통합하기
date: 2025-08-08
draft: true
tag:
  - Node.js
  - TypeScript
  - Bundler
---

프론트엔드 프로젝트가 복잡해질수록 import 경로 관리는 중요한 문제가 됩니다. 상대 경로(`../../../components/Button`)를 사용하다 보면 코드 가독성이 떨어지고 리팩토링이 어려워집니다. 이를 해결하기 위해 많은 개발자들이 TypeScript의 path alias를 사용하지만, 번들러마다 별도 설정이 필요하다는 문제가 있습니다. 이 글에서는 Node.js의 **subpath import** 기능을 활용해 이러한 문제를 해결하는 방법을 소개합니다.

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

    // "#*": "./src/*" 로도 가능
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
    alias: {
      '@components': '/src/components'
    }
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

### 3. Node.js 네이티브 지원

Subpath import는 **Node.js 12.19.0**부터 지원되는 공식 기능입니다. 이는 단순한 컨벤션이나 서드파티 도구가 아닌, 플랫폼 차원의 표준입니다.

이것이 왜 중요할까요? 우리가 사용하는 대부분의 프론트엔드 도구들은 Node.js 위에서 동작합니다. Webpack, Vite, Rollup 같은 번들러들도, TypeScript 컴파일러도, 심지어 ESLint나 Prettier 같은 린터와 포매터도 모두 Node.js 환경에서 실행됩니다. 따라서 Node.js가 공식적으로 지원하는 기능이라는 것은 이러한 도구들이 자연스럽게 이를 지원하게 된다는 의미입니다.

실제로 TypeScript의 path alias는 TypeScript 컴파일러만의 기능이기 때문에, 다른 도구들은 이를 이해하지 못합니다. 반면 subpath import는 Node.js의 모듈 해석 시스템에 내장되어 있어, Node.js 위에서 동작하는 대부분의 도구가 추가 설정 없이도 이를 이해할 수 있습니다. 이는 곧 더 나은 호환성과 적은 설정 부담을 의미합니다.

또한 Node.js 팀이 직접 유지보수하는 기능이므로 장기적인 안정성이 보장됩니다. ECMAScript 모듈 시스템의 발전과 함께 지속적으로 개선되고 있으며, 새로운 JavaScript 표준이 등장하더라도 하위 호환성을 유지하면서 발전할 것입니다. 반면 서드파티 도구나 특정 번들러의 독자적인 기능은 언제든 deprecated되거나 breaking change가 발생할 수 있습니다.


## 주의사항

### 1. TS 5.4+에서 지원됨

### 2. import 문에서 확장자 명시 필요

import 경로에 파일 확장자를 반드시 명시해야 합니다:

```typescript
// ❌ 확장자 없이 사용 - 오류 발생
import Button from '#components/Button';

// ✅ 확장자 명시 - 정상 동작
import Button from '#components/Button.tsx';
```

확장자를 왜 넣어야 하는지에 대한 이유는 TypeScript 레포의 [이슈](https://github.com/microsoft/TypeScript/issues/60003#issuecomment-2364130579)를 참조하세요.