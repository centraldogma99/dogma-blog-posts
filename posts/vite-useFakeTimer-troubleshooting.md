---
title: Vitest useFakeTimer와 Testing Library 호환성 문제 해결하기
subtitle: waitFor timeout, findBy 쿼리 오류, user action 동작 문제의 원인과 해결법
date: 2024-10-14
draft: false
tag:
  - vitest
  - testing-library
  - fake-timer
  - testing
---

Vitest에서 `useFakeTimer`를 사용하면 테스트 실행 시간을 제어할 수 있어 비동기 로직을 효율적으로 테스트할 수 있습니다. 하지만 testing-library와 함께 사용할 때 예상치 못한 호환성 문제들이 발생할 수 있습니다. 이 글에서는 실제로 겪었던 몇몇 문제와 그 해결 방법을 공유합니다.

## fakeTimer 사용중인 상태에서  `waitFor` 사용하면 timeout되는 문제
### 해결법
@testing-library/react에서 import한 waitFor 대신, vitest의 waitFor, 즉 `vi.waitFor`을 사용하세요.

```ts
// import { waitFor } from '@testing-library/react'
import { vi } from 'vitest'

...
  // waitFor(() => { expect( ... ) })
  vi.waitFor(() => { expect( ... ) })
```

### 왜 안돼?
fakeTimer를 사용하면, 내부적으로 `setTimeout`, `setInterval`, `clearInterval` 등 타이머 관련 함수들이 모킹됩니다. 하지만 testing-library의 waitFor 역시 내부적으로 `setTimeout` 등을 사용하기 때문에, fakeTimer에 의해 의도치 않게 모킹되어 제대로 동작하지 않게 됩니다. 그러나 vitest의 waitFor은 fakeTimer에 대응되도록 설계되어 있어 정상적으로 작동합니다.

### 왜 돼?
`vi.waitFor`에는 fakeTimer와 연관된 추가적인 동작이 있습니다. ([문서](https://vitest.dev/api/vi.html#vi-waitfor))
>If vi.useFakeTimers is used, vi.waitFor automatically calls vi.advanceTimersByTime(interval) in every check callback.

정확하지는 않지만, testing-library의 waitFor에는 해당 기능이 없거나 부족한 부분이 있어 예상한 대로 동작하지 않는 것으로 보입니다.


## fakeTimer를 사용 중일 때, `findBy()` 쿼리가 제대로 동작하지 않는 문제
원래 `findBy` 쿼리는 기본 1000ms의 timeout을 기다린 후, 요소를 찾지 못하면 에러를 발생시킵니다. 그러나 fakeTimer를 사용할 때는 기다리지 않고 바로 에러가 발생하는 현상이 있습니다. 이에 대한 해결 방법입니다.

### 해결법
테스트 실행 전에 `globalThis.jest = vi` 를 삽입하세요. `beforeEach()`에서 해도 좋지만, 저는 vitest setup 파일에 넣어 사용했습니다.
이 방법은 [vitest 레포 이슈](https://github.com/vitest-dev/vitest/issues/3117#issuecomment-1493249764)에서 찾았습니다.

```ts
import { vi } from 'vitest'

beforeEach(() => { globalThis.jest = vi })

it( ... )
```
or
```ts
// vitest.setup.ts
import { vi } from 'vitest'

globalThis.jest = vi

// ... setup 코드 ...
```

### 왜 안돼?
`findBy` 같은 쿼리는 요소를 찾는 데 시간이 걸리는데, fakeTimer는 그 시간을 인식하지 못합니다. 현실에서는 시간이 흘렀지만, fakeTimer 상에서는 시간이 멈춰있기 때문에 이로 인해 동기화가 맞지 않아 문제가 발생하는 것입니다.

testing-library는 이 문제에 대한 처리를 jest에만 해 두었고, 따라서 vitest 환경에서는 일부 기능이 정상적으로 동작하지 않는 것으로 보입니다.  ([testing-library 레포 이슈](https://github.com/testing-library/dom-testing-library/issues/1218#issuecomment-1460269287))

### 왜 돼?
jest 전역 객체에 vitest를 덮어씌움으로써, testing-library가 fakeTimer의 동기화 처리를 위해 jest의 메서드를 호출할 때 vitest의 그것이 대신 호출되어 문제 없이 동작하게 됩니다.

## fakeTimer를 사용 중일 때, user action이 제대로 동작하지 않는 문제
여기서 말하는 user action은 `@testing-library/user-event` 라이브러리를 사용해 에뮬레이트한 클릭, 입력 등의 액션을 의미합니다.

### 해결법
`userEvent.setup()` 시, 아래와 같이 `vi.advanceTimersByTime`을 인자로 넘겨주세요.
```ts
const user = userEvent.setup({
  advanceTimers: vi.advanceTimersByTime
})
```
출처: [testing-library 레포 이슈](https://github.com/testing-library/user-event/issues/1034#issuecomment-1231927291), [testing-library docs](https://testing-library.com/docs/user-event/options/#advancetimers)

### 왜 안돼?
testing-library 내부적으로 user action 사이사이에 일정한 딜레이가 있습니다. 하지만 이 딜레이가 fakeTimer에는 반영되지 않기 때문에, 실제 시간은 흘렀지만 fakeTimer 상에서는 시간이 멈춘 상태가 되어 동기화가 맞지 않아 문제가 발생합니다.

### 왜 돼?
user action 사이의 딜레이만큼 fakeTimer의 시간도 흐르도록 만들어야 의도한 대로 작동합니다. 위에서 vitest의 fakeTimer 시간을 조정하는 함수를 넘겨줌으로써, testing-library가 user action 사이의 딜레이를 처리할 때마다 해당 함수를 호출해 fakeTimer도 딜레이만큼 시간이 흐르도록 조정해 주기 때문에 문제없이 동작하게 됩니다.