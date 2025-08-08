---
title: Typescript의 verbatimModuleSyntax - import를 있는 그대로 유지하는 이유
subtitle: 현대적인 빌드 도구 체인에서 Typescript가 코드를 "똑똑하게" 변형하지 않는 것이 왜 더 나은 선택인지 알아봅니다.
date: 2025-08-08
draft: false
tag:
  - typescript
  - vite
---

## TL;DR

- Typescript는 타입으로만 사용되는 import 구문을 컴파일 과정에서 "똑똑하게" 알아서 제거합니다
- `verbatimModuleSyntax: true`는 Typescript가 import 구문을 "똑똑하게" 최적화하지 않고 `import type` 구문을 통해 명시된 구문만 제거합니다
- 타입 import는 반드시 `import type`으로 명시해야 하며, 이를 통해 번들러가 더 효율적으로 tree-shaking을 수행할 수 있습니다
- Vite, esbuild 같은 현대적인 빌드 도구를 사용한다면 이 옵션을 켜는 것이 좋습니다
- 개발 중에는 ESLint 규칙처럼 작동해 타입 import 시 `import type` 구문을 사용하지 않으면 에러를 발생시킵니다

---

Typescript 프로젝트를 진행하다 보면 `tsconfig.json`에서 낯선 옵션들을 마주치곤 합니다. 그중에서도 `verbatimModuleSyntax`는 이름부터 생소하죠. 

여기서 'verbatim'은 "한 글자도 바꾸지 않고 있는 그대로"라는 뜻의 영어 단어입니다. 즉, `verbatimModuleSyntax`는 직역하면 "모듈 구문을 있는 그대로 유지한다"는 의미인데, 이 설명만으로는 왜 필요한지 와닿지 않습니다.

Typescript가 알아서 최적화해 주는 게 더 좋은 거 아닌가? 싶었지만, 좀 더 알아보니 왜 이 옵션이 필요한지 알 수 있었습니다.

## Typescript의 "똑똑한" 최적화가 문제가 되는 이유

Typescript는 기본적으로 우리가 작성한 import 구문을 분석해서 "이건 타입으로만 쓰이네?"라고 판단하면 컴파일 시 제거합니다.

```typescript
// 원본 코드
import { useState } from 'react';  // 실제 값
import { FC } from 'react';        // 타입으로만 사용

// Typescript가 컴파일한 결과 (verbatimModuleSyntax: false)
import { useState } from 'react';  // FC는 자동으로 제거됨
```

언뜻 보면 효율적인 것 같지만, 여기서 문제가 시작됩니다.

### 번들러와의 충돌

Vite, esbuild, SWC 같은 현대적인 번들러들은 Typescript 파일을 직접 처리합니다. 이들은 자체적인 최적화 전략을 가지고 있는데, Typescript가 먼저 코드를 변형해버리면 번들러의 tree-shaking이 제대로 작동하지 않을 수 있습니다.

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
import './styles.css';  // 값은 없지만 스타일 적용이라는 side effect가 있음

// Typescript: "아무도 안 쓰네? 제거!" -> 결과: 스타일이 적용되지 않음
```

`verbatimModuleSyntax: true`를 사용하면 이런 실수를 방지할 수 있습니다.

## type 키워드 - 개발자와 도구 간의 명확한 계약

`verbatimModuleSyntax`를 활성화하면, Typescript는 더 이상 "추측"하지 않습니다. 대신 개발자가 `type` 키워드로 명시적으로 표시한 것만 제거합니다.

```typescript
// ✅ 명확한 의도 표현
import type { User, UserRole } from './types';  // 타입 전용 - 컴파일 과정에서 제거됨
import { api, validateUser } from './utils';    // 런타임 값

// ❌ 모호한 표현 (verbatimModuleSyntax: true일 때 에러)
import { User } from './types';  // 타입인데 type 키워드 없음
```

이는 마치 Typescript의 strict 모드처럼, 처음엔 번거롭지만 장기적으로 더 안전한 코드를 만들어줍니다.

## 실제 프로젝트에서의 활용

### 코드 품질 관리 도구로서의 역할

ESLint의 `@typescript-eslint/consistent-type-imports` 규칙을 사용해 타입 import를 관리할 수 있습니다. `verbatimModuleSyntax`는 이와 비슷한 역할을 하지만, Typescript 컴파일러 레벨에서 강제한다는 점이 다릅니다.

```typescript
// ESLint 규칙
"@typescript-eslint/consistent-type-imports": "error"

// tsconfig.json
{
  "compilerOptions": {
    "verbatimModuleSyntax": true
  }
}
```

### 실제 사용 예시: React 컴포넌트 작성

```typescript
// Before: 암묵적인 import
import { Component, ReactNode } from 'react';
import { UserProfile } from './types';

// After: 명시적인 import (verbatimModuleSyntax: true)
import { Component, type ReactNode } from 'react';
import type { UserProfile } from './types';
```

