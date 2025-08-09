---
title: Subpath Import로 테스트 모킹 혁신하기
subtitle: vi.mock()과 resolve.alias를 넘어선 Node.js 표준 기반 모킹 전략
date: 2025-08-09
draft: false
tag:
  - Node.js
  - Testing
  - Vitest
  - TypeScript
---

테스트 코드를 작성하다 보면 외부 의존성을 모킹해야 하는 순간이 반드시 찾아옵니다. 데이터베이스 접근, API 호출과 같은 것이나, `axios`와 같은 외부 라이브러리를 모킹해야 하는 경우도 있습니다.

```typescript
// 이런 코드를 테스트하려면?
export async function fetchUserData(userId: string) {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  await logger.info(`User ${userId} fetched`);
  return data;
}
```

대부분의 개발자들은 MSW(Mock Service Worker)를 사용하여 요청을 가로채거나, 모듈 자체를 모킹하기 위해 `vi.mock()`, 또는 번들러의 `resolve.alias` 설정을 사용해 이 문제를 해결합니다. 하지만 이 방법들에는 각각의 한계가 있습니다.

이 글에서는 Node.js의 **Subpath Import**를 활용해 더 우아하고 강력한 모킹 전략을 구현하는 방법을 소개합니다. 특히 **설정의 중앙화**와 **자동 적용**이 핵심입니다.

## 왜 Subpath Import인가?

제가 Subpath Import를 통한 모킹을 시작하게 된 계기는 API 요청 모킹 때문이었습니다. 처음에는 MSW를 사용했지만, 단순히 API 응답을 모킹하는 용도로는 너무 low-level이었습니다. 네트워크 레벨에서 동작하기 때문에 설정이 복잡하고, 실제로는 함수 레벨의 모킹만 필요한 경우가 대부분이었죠.

```typescript
// MSW의 복잡함: 단순 API 모킹에는 과한 설정
const server = setupServer(
  rest.get('/api/user/:id', (req, res, ctx) => {
    return res(ctx.json({ 
      name: 'John',
      email: 'john@example.com'
    }));
  })
);

// 서버 생명주기 관리 필요
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

이런 복잡성을 피하고 더 간단하면서도 강력한 모킹 방법을 찾다가 Subpath Import를 발견하게 되었습니다.

## 기존 모킹 방식의 한계

### MSW의 한계

MSW는 네트워크 레벨에서 동작하는 강력한 도구지만, 몇 가지 한계가 있습니다:

1. **과도한 설정**: 단순 함수 모킹에 비해 설정이 복잡
2. **네트워크 레벨 동작**: 실제 HTTP 요청을 가로채기 때문에 오버헤드 존재
3. **디버깅 어려움**: 네트워크 레이어에서 동작하므로 문제 추적이 어려움

```typescript
// MSW는 실제 네트워크 요청을 가로채는 방식
const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(ctx.json({ data: 'mock' }));
  })
);

// 모든 테스트 파일에서 반복되는 보일러플레이트
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### vi.mock()의 제약

Vitest의 `vi.mock()`은 타입 안정성을 지원하지만, 여전히 몇 가지 불편함이 있습니다:

```typescript
// vi.mock()은 타입 안정성을 지원하지만...
vi.mock('./api/client', async (importOriginal) => {
  const actual = await importOriginal<typeof import('./api/client')>();
  return {
    ...actual,
    fetchUser: vi.fn() // 매번 원본 타입을 명시해야 함
  };
});

// 매 테스트 파일마다 반복
vi.mock('./api/client');
vi.mock('./utils/logger');
vi.mock('./services/database');

// ESM 환경에서의 호이스팅 이슈
import { fetchData } from './api/client'; // vi.mock보다 먼저 실행될 수 있음
```

가장 큰 문제는 **모든 테스트 파일에서 반복적으로 mock 설정을 해야 한다**는 점입니다. 또한 호이스팅 때문에 import 순서를 신경써야 하는 번거로움도 있습니다.

### resolve.alias의 파편화

번들러 설정을 통한 모킹은 설정이 파편화되는 문제가 있습니다:

```javascript
// vite.config.js
export default {
  resolve: {
    alias: process.env.NODE_ENV === 'test' 
      ? { './api': './api.mock' }
      : {}
  }
};

// webpack.config.js - 또 다른 설정
// jest.config.js - 또 또 다른 설정...
```

## Subpath Import + vi.fn() 하이브리드 전략

Subpath import의 조건부 모듈 해석 기능과 Vitest의 동적 모킹을 결합하면, 두 방식의 장점을 모두 활용할 수 있습니다. **한 번 설정하면 모든 테스트에서 자동으로 적용**되는 것이 가장 큰 장점입니다.

### 1단계: package.json 설정

```json
{
  "imports": {
    "#api/*": {
      "test": "./src/__mocks__/api/*",
      "default": "./src/api/*"
    },
    "#services/*": {
      "test": "./src/__mocks__/services/*",
      "default": "./src/services/*"
    },
    "#logger": {
      "test": "./src/__mocks__/logger.ts",
      "development": "./src/utils/logger-dev.ts",
      "production": "./src/utils/logger-prod.ts"
    }
  }
}
```

### 2단계: Mock 파일 구성

실제 함수 대신 `vi.fn()`으로 mock 파일을 구성합니다:

```typescript
// src/__mocks__/api/client.ts
import { vi } from 'vitest';
import type { UserData, ApiResponse } from '#api/client.ts';

// 타입 안정성 보장
export const fetchUser = vi.fn<[string], Promise<UserData>>();
export const updateUser = vi.fn<[string, Partial<UserData>], Promise<ApiResponse>>();
export const deleteUser = vi.fn<[string], Promise<void>>();

// 기본 동작 설정 (선택사항)
fetchUser.mockResolvedValue({ 
  id: 'default-id', 
  name: 'Test User'
});
```

```typescript
// src/__mocks__/services/database.ts
import { vi } from 'vitest';

export const query = vi.fn();
export const transaction = vi.fn();
export const connect = vi.fn().mockResolvedValue(true);
export const disconnect = vi.fn();
```

### 3단계: 테스트에서 활용

이제 테스트에서 자유롭게 모킹 동작을 제어할 수 있습니다:

```typescript
// user.service.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { getUserProfile } from './user.service.ts';
import { fetchUser } from '#api/client.ts'; // 자동으로 mock 버전 로드
import { query } from '#services/database.ts';

describe('UserService', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('캐시가 있으면 DB를 조회하지 않음', async () => {
    // 테스트별 맞춤 설정
    fetchUser.mockResolvedValueOnce({ 
      id: '123', 
      name: 'Cached User',
      cached: true 
    });

    const result = await getUserProfile('123');
    
    expect(fetchUser).toHaveBeenCalledWith('123');
    expect(query).not.toHaveBeenCalled(); // DB 호출 없음
    expect(result.cached).toBe(true);
  });

  it('네트워크 에러 시 DB 폴백', async () => {
    // 첫 번째 호출은 실패, 두 번째는 성공
    fetchUser.mockRejectedValueOnce(new Error('Network Error'));
    query.mockResolvedValueOnce({ 
      rows: [{ id: '123', name: 'DB User' }] 
    });

    const result = await getUserProfile('123');
    
    expect(fetchUser).toHaveBeenCalledTimes(1);
    expect(query).toHaveBeenCalledWith(
      'SELECT * FROM users WHERE id = $1', 
      ['123']
    );
    expect(result.name).toBe('DB User');
  });

  it('연속 호출 시나리오 테스트', async () => {
    // mockImplementation으로 복잡한 로직 구현
    let callCount = 0;
    fetchUser.mockImplementation(async (id) => {
      callCount++;
      if (callCount === 1) {
        return { id, name: 'First Call', version: 1 };
      }
      return { id, name: 'Updated', version: 2 };
    });

    const first = await getUserProfile('123');
    const second = await getUserProfile('123');
    
    expect(first.version).toBe(1);
    expect(second.version).toBe(2);
    expect(fetchUser).toHaveBeenCalledTimes(2);
  });
});
```

## 실전 활용 패턴

### 패턴 1: 환경별 Mock 구성

개발/테스트/프로덕션 환경에 따라 다른 구현체를 자동으로 로드:

```json
{
  "imports": {
    "#payment/*": {
      "test": "./src/__mocks__/payment/*",
      "development": "./src/services/payment-sandbox/*",
      "production": "./src/services/payment/*"
    }
  }
}
```

```typescript
// 코드는 동일, 환경에 따라 다른 구현 사용
import { processPayment } from '#payment/processor.ts';

// 테스트: mock 함수
// 개발: Stripe 샌드박스
// 프로덕션: 실제 Stripe API
```

### 패턴 2: 부분 모킹

일부 함수만 모킹하고 나머지는 실제 구현 사용:

```typescript
// src/__mocks__/utils/crypto.ts
export * from '../../utils/crypto.ts'; // 실제 구현 re-export
import { vi } from 'vitest';

// 특정 함수만 모킹
export const generateToken = vi.fn().mockReturnValue('mock-token-123');
```

### 패턴 3: 전역 Mock 상태 관리

여러 테스트에서 공유하는 mock 상태:

```typescript
// src/__mocks__/services/auth.ts
import { vi } from 'vitest';

interface MockAuthState {
  isAuthenticated: boolean;
  currentUser: User | null;
}

// 전역 mock 상태
const mockState: MockAuthState = {
  isAuthenticated: false,
  currentUser: null
};

export const login = vi.fn().mockImplementation(async (credentials) => {
  mockState.isAuthenticated = true;
  mockState.currentUser = { id: '1', email: credentials.email };
  return mockState.currentUser;
});

export const logout = vi.fn().mockImplementation(() => {
  mockState.isAuthenticated = false;
  mockState.currentUser = null;
});

export const getCurrentUser = vi.fn().mockImplementation(() => {
  return mockState.currentUser;
});

// 테스트 헬퍼
export const resetAuthState = () => {
  mockState.isAuthenticated = false;
  mockState.currentUser = null;
};
```

## 장점과 고려사항

### 장점

1. **설정의 일회성**: package.json에 한 번만 설정하면 모든 테스트에서 자동 적용
2. **중앙 집중식 관리**: 모든 모킹 경로를 한 곳에서 관리
3. **호이스팅 문제 해결**: `vi.mock()` 호이스팅 순서를 신경쓸 필요 없음
4. **환경별 자동 전환**: 테스트/개발/프로덕션 환경에 따라 자동으로 적절한 모듈 로드
5. **타입 안정성**: TypeScript의 타입 체크 완벽 지원
6. **동적 제어**: 각 테스트에서 `mockImplementation`, `mockResolvedValue` 등 자유롭게 사용

### 고려사항

1. **Mock 디렉토리 구조 유지**: 실제 모듈 구조와 동일한 mock 디렉토리 필요
2. **파일 확장자 명시**: import 시 `.ts`, `.tsx` 등 확장자 필수
3. **초기 설정**: 프로젝트 시작 시 mock 구조 설계 필요

## 마이그레이션 전략

기존 `vi.mock()` 사용 코드에서 점진적으로 마이그레이션:

```typescript
// 1단계: 자주 모킹하는 모듈부터 시작
{
  "imports": {
    "#api/*": {
      "test": "./src/__mocks__/api/*",
      "default": "./src/api/*"
    }
  }
}

// 2단계: 기존 vi.mock() 코드를 mock 파일로 이동
// Before (각 테스트 파일에서)
vi.mock('./api/client', () => ({
  fetchData: vi.fn()
}));

// After (src/__mocks__/api/client.ts)
import { vi } from 'vitest';
export const fetchData = vi.fn();

// 3단계: import 경로 변경
// import { fetchData } from './api/client';
import { fetchData } from '#api/client.ts';
```

## 실제 프로젝트 적용 예시

복잡한 의존성을 가진 실제 서비스 테스트:

```typescript
// src/services/order.service.ts
import { fetchUser } from '#api/user.ts';
import { processPayment } from '#payment/processor.ts';
import { sendEmail } from '#notifications/email.ts';
import { query, transaction } from '#services/database.ts';
import logger from '#logger';

export async function createOrder(userId: string, items: CartItem[]) {
  try {
    const user = await fetchUser(userId);
    
    const result = await transaction(async (client) => {
      const order = await client.query(
        'INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING *',
        [userId, calculateTotal(items)]
      );
      
      const payment = await processPayment({
        amount: order.rows[0].total,
        customerId: user.stripeId
      });
      
      await client.query(
        'UPDATE orders SET payment_id = $1 WHERE id = $2',
        [payment.id, order.rows[0].id]
      );
      
      return { order: order.rows[0], payment };
    });
    
    await sendEmail(user.email, 'order-confirmation', result);
    await logger.info(`Order created: ${result.order.id}`);
    
    return result;
  } catch (error) {
    await logger.error('Order creation failed', error);
    throw error;
  }
}
```

테스트 코드:

```typescript
// src/services/order.service.test.ts
import { createOrder } from './order.service.ts';
import { fetchUser } from '#api/user.ts';
import { processPayment } from '#payment/processor.ts';
import { sendEmail } from '#notifications/email.ts';
import { transaction } from '#services/database.ts';
import logger from '#logger';

describe('Order Service', () => {
  it('성공적인 주문 생성 플로우', async () => {
    // 모든 mock이 자동으로 로드됨
    fetchUser.mockResolvedValue({ 
      id: 'user-1', 
      email: 'test@example.com',
      stripeId: 'cus_123' 
    });
    
    transaction.mockImplementation(async (callback) => {
      const mockClient = {
        query: vi.fn()
          .mockResolvedValueOnce({ 
            rows: [{ id: 'order-1', total: 100 }] 
          })
          .mockResolvedValueOnce({ rows: [] })
      };
      return callback(mockClient);
    });
    
    processPayment.mockResolvedValue({ 
      id: 'pay_123', 
      status: 'succeeded' 
    });
    
    const result = await createOrder('user-1', [
      { productId: 'prod-1', quantity: 2, price: 50 }
    ]);
    
    expect(result.order.id).toBe('order-1');
    expect(result.payment.status).toBe('succeeded');
    expect(sendEmail).toHaveBeenCalledWith(
      'test@example.com',
      'order-confirmation',
      expect.any(Object)
    );
    expect(logger.info).toHaveBeenCalledWith('Order created: order-1');
  });
});
```

## 결론

Subpath Import와 `vi.fn()`을 결합한 하이브리드 모킹 전략은 기존 방식의 장점을 모두 취하면서 단점을 보완합니다.

- **일회성 설정**: 한 번 설정하면 모든 테스트에서 자동 적용
- **구조적 일관성**: Subpath Import로 모킹 경로를 중앙 관리
- **동적 유연성**: `vi.fn()`으로 테스트별 맞춤 동작 구현
- **개발자 경험**: 호이스팅 걱정 없이 직관적인 import 사용
- **유지보수성**: 명확한 mock 구조와 재사용 가능한 패턴

처음에는 MSW의 복잡한 설정과 `vi.mock()`의 반복적인 선언에서 벗어나고 싶어서 시작했지만, 결과적으로 전체 테스트 전략을 개선하는 방법을 발견하게 되었습니다. 특히 대규모 프로젝트에서 복잡한 의존성을 체계적으로 관리해야 할 때, 이 접근 방식은 테스트 코드의 품질과 유지보수성을 크게 향상시킬 수 있습니다.

다음 프로젝트에서 테스트 전략을 수립할 때, Subpath Import를 활용한 모킹을 고려해보시는 건 어떨까요? MSW의 복잡함과 `vi.mock()`의 반복적인 설정에서 벗어나, Node.js 표준 기능을 활용한 더 나은 테스트 경험이 여러분을 기다리고 있습니다.