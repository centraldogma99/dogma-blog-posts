---
title: Subpath Import로 테스트 모킹 혁신하기
description: vi.mock()과 resolve.alias를 넘어선 Node.js 표준 기반 모킹 전략
date: 2025-08-09
draft: true
tag:
  - Node.js
  - Testing
  - Vitest
  - TypeScript
---

(SUbpath import 문서 링크 필요)
테스트 코드를 작성하다 보면 외부 의존성을 모킹해야 하는 순간이 반드시 찾아옵니다. 데이터베이스 접근, API 호출과 같은 것이나, `axios`와 같은 외부 라이브러리를 모킹해야 하는 경우도 있습니다.

아래와 같은 `MyPage` 컴포넌트가 있고, 그 컴포넌트가 값을 잘 표시하는지를 테스트해야 하는 상황이라고 가정해 봅시다.
API를 호출하고, 응답값을 화면에 표시하는 간단한 예시입니다.

```tsx
interface UserData {
  name: string,
  email: string
}

// src/api/client.ts
export async function fetchUserData(userId: string): UserData {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  return data as UserData;
}
export async function updateUser() {...}
export async function deleteUser() {...}

// src/page/MyPage.tsx
export const MyPage = () => {
  const [data, setData] = useState<UserData | null>(null)
  
  useEffect(() => {
    fetchUserData().then((data) => {
      setData(data)
    })
  }, [])

  return (
    <h1>{data?.name}</h1>
    <h2>{data?.email}</h2>
  )
};
```

보통은 MSW(Mock Service Worker)를 사용하여 요청을 가로채거나, 모듈 자체를 모킹하기 위해 `vi.mock()`, 또는 번들러의 `resolve.alias` 설정을 사용해 이 문제를 해결합니다. 하지만 이 방법들에는 각각의 한계가 있습니다.

이 글에서는 Node.js의 **Subpath Import**를 활용해 더 우아하고 강력한 모킹 전략을 구현하는 방법을 소개합니다.

## 왜 Subpath Import인가?

제가 Subpath Import를 통한 모킹을 시작하게 된 계기는 API 요청 모킹 때문이었습니다. 처음에는 MSW를 사용했지만, 단순히 API 응답을 모킹하는 용도로는 너무 low-level이었습니다. 네트워크 레벨에서 동작하기 때문에 설정이 복잡하고, 실제로는 함수 레벨의 모킹만 필요한 경우가 대부분이었죠.

또한 type-safe한 모킹이 불가능하여 개발자 경험(DX)이 좋지 않을 뿐더러, 파편화의 가능성도 생기게 됩니다.

```typescript
// MSW의 복잡함: 단순 API 모킹에는 과한 설정
const server = setupServer(
  rest.get('/api/user/:id', (req, res, ctx) => {
    return res(ctx.json({ 
      name: 'John',
      email: 103840 // 잘못된 값(string 이어야 함)이지만 에러를 발생시키지 않음
    }));
  })
);

// 서버 생명주기 관리 필요
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

번들러 설정을 통한 모킹은 설정 자체는 간단하지만, 설정이 파편화되는 문제가 있습니다.
당연히 type-safe 모킹도 불가능하구요.

```javascript
// vite.config.js
export default {
  resolve: {
    alias: process.env.NODE_ENV === 'test' 
      ? { '#api/client.ts': 'src/api/client.mock.ts' }
      : {}
  }
};

// webpack.config.js - 또 다른 설정
// jest.config.js - 또 또 다른 설정...
```

Vitest의 `vi.mock` 역시 여전히 몇 가지 불편함이 있습니다.
이 방법은 type-safe한 모킹이 가능하긴 하지만, 모듈의 일부만 모킹하려 해도 너무 많은 코드를 작성해야 하고, mock 코드의 호이스팅 등 여러 가지 고려해야 할 사항이 좀더 많다는 문제가 있습니다.

```typescript
const mockData = {
  name: 'John',
  email: 'john@example.com'
}

// 모듈에서 하나의 함수만 모킹하고 싶어도 매번 이렇게 긴 코드를 작성해야 함
vi.mock(import('#api/client.ts'), async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    // vi.mock 은 hoist되기 때문에, 아래와 같이 변수를 참조할 경우 모킹이 제대로 되지 않음.
    // 문서 https://vitest.dev/api/vi.html#vi-mock 참조
    fetchUserData: () => mockData
  };
});
```

## Subpath Import + vi.fn() 하이브리드 전략

Subpath import의 조건부 모듈 해석 기능과 Vitest의 동적 모킹을 결합하면, 두 방식의 장점을 모두 활용할 수 있습니다. **한 번 설정하면 모든 테스트에서 자동으로 적용**되며, type-safe한 모킹도 가능합니다.

### 1단계: package.json 설정

```json
{
  "imports": {
    "#api/client.ts": {
      "test": "./src/api/client.mock.ts",
      "default": "./src/api/client.ts"
    },
    "#*": "./src/*"
  }
}
```

### 2단계: Mock 파일 구성

실제 함수 대신 `vi.fn()`으로 mock 파일을 구성합니다:

```typescript
// src/api/client.mock.ts
import { vi } from 'vitest';
import * as actual from './client.ts';

// 원래 모듈(client.ts)에서 export하는 모든 함수들을 여기에서도 동일하게 export해야 합니다.
export const fetchUser = vi.fn(actual.fetchUser);
export const updateUser = vi.fn(actual.updateUser);
export const deleteUser = vi.fn(actual.deleteUser);

// 기본 모킹 설정 (선택사항)
fetchUser.mockResolvedValue({ 
  email: 'asdf@asd.asd', 
  name: 'Test User'
});
```

### 3단계: 테스트에서 활용

이제 테스트에서 자유롭게 모킹 동작을 제어할 수 있습니다:

```tsx
// MyPage.test.tsx
import { it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { MyPage } from '#pages/MyPage.tsx';
// mock 파일을 import 하는 것이 맞습니다. 실수가 아닙니다!
import { fetchUser } from '#api/client.mock.ts';

const setup = () => {
  render(<MyPage />);
}

const mockData: UserData = {
  email: 'johndoe@gmail.com',
  name: 'John Doe',
}

it('이름과 이메일이 잘 표시됨', async () => {
  fetchUser.mockResolvedValue(mockData)
  setup();

  // h1 안에 이름이 표시되는지 확인
  expect(await screen.findByRole('heading', {
    level: 1,
  })).toHaveTextContent(mockData.name);
  // h2 안에 이메일이 표시되는지 확인
  expect(await screen.findByRole('heading', {
    level: 2,
  })).toHaveTextContent(mockData.email);
});
```

## 장점과 고려사항

### 장점

1. **설정의 일회성**: package.json에 한 번만 설정하면 모든 테스트에서 자동 적용
2. **중앙 집중식 관리**: 모든 모킹 경로를 한 곳에서 관리
3. **호이스팅 문제 해결**: `vi.mock()` 호이스팅 순서를 신경쓸 필요 없음
4. **타입 안정성**: TypeScript의 타입 체크 완벽 지원
5. **동적 제어**: 각 테스트에서 `mockImplementation`, `mockResolvedValue` 등 자유롭게 사용

### 고려사항

1. **Mock 디렉토리 구조 유지**: 실제 모듈 구조와 동일한 mock 디렉토리 필요
2. **파일 확장자 명시**: import 시 `.ts`, `.tsx` 등 확장자 필수
3. **초기 설정**: 프로젝트 시작 시 mock 구조 설계 필요

## 결론

Subpath Import와 `vi.fn()`을 결합한 하이브리드 모킹 전략은 기존 방식의 장점을 모두 취하면서 단점을 보완합니다.

- **일회성 설정**: 한 번 설정하면 모든 테스트에서 자동 적용
- **구조적 일관성**: Subpath Import로 모킹 경로를 중앙 관리
- **동적 유연성**: `vi.fn()`으로 테스트별 맞춤 동작 구현
- **개발자 경험**: 호이스팅 걱정 없이 직관적인 import 사용
- **유지보수성**: 명확한 mock 구조와 재사용 가능한 패턴

처음에는 MSW의 복잡한 설정과 `vi.mock()`의 반복적인 선언에서 벗어나고 싶어서 시작했지만, 결과적으로 전체 테스트 전략을 개선하는 방법을 발견하게 되었습니다. 특히 대규모 프로젝트에서 복잡한 의존성을 체계적으로 관리해야 할 때, 이 접근 방식은 테스트 코드의 품질과 유지보수성을 크게 향상시킬 수 있습니다.

다음 프로젝트에서 테스트 전략을 수립할 때, Subpath Import를 활용한 모킹을 고려해보시는 건 어떨까요? MSW의 복잡함과 `vi.mock()`의 반복적인 설정에서 벗어나, Node.js 표준 기능을 활용한 더 나은 테스트 경험이 여러분을 기다리고 있습니다.

resolve.conditions에 vitest인 경우 설정을 추가해야 할 수도 있음.
