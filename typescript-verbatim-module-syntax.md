---
title: TypeScript의 verbatimModuleSyntax - import를 있는 그대로 유지하는 이유
subtitle: 현대적인 빌드 도구 체인에서 TypeScript가 코드를 "똑똑하게" 변형하지 않는 것이 왜 더 나은 선택인지 알아봅니다.
date: 2025-08-08
draft: true
tag:
  - typescript
  - vite
---

TypeScript 프로젝트를 진행하다 보면 `tsconfig.json`에서 낯선 옵션들을 마주치곤 합니다. 그중에서도 `verbatimModuleSyntax`는 이름부터 생소하죠. "import/export 구문을 있는 그대로 유지한다"는 설명을 읽어도 왜 필요한지 와닿지 않습니다.

TypeScript가 알아서 최적화해 주는 게 더 좋은 거 아닌가? 싶었지만, 좀 더 알아보니 왜 이 옵션이 필요한지 알 수 있었습니다.

## TypeScript의 "똑똑한" 최적화가 문제가 되는 이유

TypeScript는 기본적으로 우리가 작성한 import 구문을 분석해서 "이건 타입으로만 쓰이네?"라고 판단하면 컴파일 시 제거합니다.

```typescript
// 원본 코드
import { useState } from 'react';  // 실제 값
import { FC } from 'react';        // 타입으로만 사용

// TypeScript가 컴파일한 결과 (verbatimModuleSyntax: false)
import { useState } from 'react';  // FC는 자동으로 제거됨
```

언뜻 보면 효율적인 것 같지만, 여기서 문제가 시작됩니다.

### 번들러와의 충돌

Vite, esbuild, SWC 같은 현대적인 번들러들은 TypeScript 파일을 직접 처리합니다. 이들은 자체적인 최적화 전략을 가지고 있는데, TypeScript가 먼저 코드를 변형해버리면 번들러의 트리셰이킹이 제대로 작동하지 않을 수 있습니다.

```typescript
// 번들러 입장에서 보면
import { Component } from 'react';  // 값인지 타입인지 모호함

// 명시적으로 표현하면
import type { Component } from 'react';  // 확실히 제거 가능
import { Component } from 'react';       // 확실히 유지 필요
```

### 예상치 못한 side effect 제거

CSS import처럼 side effect만 있고 값은 없는 경우를 생각해보세요.

```typescript
import './styles.css';  // 값은 없지만 스타일 적용이라는 부작용이 있음

// TypeScript: "아무도 안 쓰네? 제거!" -> 결과: 스타일이 적용되지 않음
```

`verbatimModuleSyntax: true`를 사용하면 이런 실수를 방지할 수 있습니다.

## type 키워드 - 개발자와 도구 간의 명확한 계약

`verbatimModuleSyntax`를 활성화하면, TypeScript는 더 이상 "추측"하지 않습니다. 대신 개발자가 `type` 키워드로 명시적으로 표시한 것만 제거합니다.

```typescript
// ✅ 명확한 의도 표현
import type { User, UserRole } from './types';  // 타입 전용 - 컴파일 과정에서 제거됨
import { api, validateUser } from './utils';    // 런타임 값

// ❌ 모호한 표현 (verbatimModuleSyntax: true일 때 에러)
import { User } from './types';  // 타입인데 type 키워드 없음
```

이는 마치 타입스크립트의 엄격 모드처럼, 처음엔 번거롭지만 장기적으로 더 안전한 코드를 만들어줍니다.


## 개발 환경에서는 코딩 컨벤션 강제 도구

typescript 컴파일과 관rP없는 개발 환경에서는 `verbatimModuleSyntax`는 **코드 품질 관리 도구**에 가깝습니다. ESLint 규칙처럼 작동하지만, TypeScript 컴파일러 레벨에서 강제한다는 점이 다릅니다.

```typescript
// ESLint 규칙
"@typescript-eslint/consistent-type-imports": "error"

// tsconfig.json
"verbatimModuleSyntax": true

// 둘 다 같은 결과를 만듦
import type { User } from './types';  // ✅
import { User } from './types';       // ❌ 에러!
```

## 마무리

`verbatimModuleSyntax`는 TypeScript가 코드를 "똑똑하게" 변형하는 것을 막고, 개발자가 명시적으로 의도를 표현하도록 강제합니다. 처음엔 `type` 키워드를 일일이 붙이는 게 번거로울 수 있지만, 이는 장기적으로 다음과 같은 이점을 가져다줍니다.

- **예측 가능한 빌드 결과**: 작성한 그대로 출력되므로 디버깅이 쉬움
- **번들러와의 완벽한 협업**: 각 도구가 자신의 역할에 집중
- **명확한 코드 의도**: 타입과 값의 구분이 한눈에 보임

