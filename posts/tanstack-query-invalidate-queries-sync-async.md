---
title: TanStack Query의 invalidateQueries는 언제 await이 필요할까?
subtitle: invalidateQueries의 동기적 stale 표시와 비동기 refetch 동작을 이해하고, await을 올바르게 사용하는 방법
date: 2025-01-14
draft: true
tag:
  - tanstack-query
  - react-query
  - cache-management
---

## TL;DR

- `invalidateQueries`는 **await 없이 호출해도 캐시를 즉시 stale로 표시**합니다
- await은 캐시 무효화가 아닌 **refetch 완료를 기다리는 것**입니다
- 구독자가 없으면 refetch 자체가 일어나지 않아 await이 무의미합니다
- **대부분의 경우 await은 불필요**합니다 - React Query가 알아서 UI를 업데이트하기 때문입니다

## 들어가며

TanStack Query를 사용하다 보면 이런 코드를 자주 보게 됩니다:

```tsx
// 어떤 개발자의 코드
await queryClient.invalidateQueries({ queryKey: ['todos'] })

// 다른 개발자의 코드  
queryClient.invalidateQueries({ queryKey: ['todos'] })
```

같은 `invalidateQueries`인데 왜 어떤 사람은 `await`을 붙이고, 어떤 사람은 붙이지 않을까요? 

더 혼란스러운 건, **둘 다 제대로 동작한다**는 점입니다. 

이 글에서는 많은 개발자들이 모르고 있는 `invalidateQueries`의 동작 방식을 알아보겠습니다.

## invalidateQueries가 하는 두 가지 일

`invalidateQueries`는 비동기 함수지만, 사실 두 가지 작업을 수행합니다:

1. **즉시 실행**: 캐시를 stale(오래된 상태)로 표시
2. **비동기 실행**: 캐시를 구독하고 있는 곳이 있으면 refetch

이를 간단히 표현하면:

```typescript
// invalidateQueries의 동작 방식 (개념적 설명)
async function invalidateQueries(queryKey) {
  // 1단계: 즉시 실행 - 캐시를 "오래됨"으로 표시
  markAsStale(queryKey)  // 이건 동기적으로 바로 실행됨!
  
  // 2단계: 비동기 실행 - 필요한 경우에만 새로 fetch
  if (이_데이터를_구독하는_곳이_있는가()) {
    await refetch(queryKey)  // 이것만 비동기
  }
}
```

## 핵심: await 붙여도, 안 붙여도 캐시는 즉시 무효화

많은 개발자들이 착각하는 부분이 바로 이것입니다:

> "await을 안 붙이면 캐시 무효화가 나중에 일어날 거야"

**아닙니다!** await 여부와 관계없이 캐시는 **즉시** stale로 표시됩니다.

```tsx
// await 없는 경우
queryClient.invalidateQueries({ queryKey: ['todos'] })
// 👆 이 줄이 실행되는 순간 캐시는 이미 stale!

// await 있는 경우  
await queryClient.invalidateQueries({ queryKey: ['todos'] })
// 👆 캐시가 stale로 표시되는 타이밍은 위와 동일!
// await은 단지 refetch가 완료될 때까지 기다릴 뿐
```

## 흔한 오해들

### 오해 1: "await을 안 붙이면 무효화가 안 된다"

```tsx
queryClient.invalidateQueries({ queryKey: ['todos'] })
// 👆 await 없어도 캐시는 즉시 stale 상태가 됩니다
```

이미 stale로 표시되었기 때문에, 다음번에 이 데이터를 사용하는 컴포넌트가 마운트되었을 때 자동으로 새로운 데이터를 가져옵니다.

### 오해 2: "await을 붙이면 항상 서버 요청을 기다린다"

```tsx
// 아무도 이 데이터를 구독하고 있지 않다면?
await queryClient.invalidateQueries({ queryKey: ['unused-data'] })
// 👆 refetch가 일어나지 않아서 즉시 완료됨
```

useQuery 등으로 해당 데이터를 구독하는 컴포넌트가 없다면, refetch 자체가 일어나지 않습니다. await을 붙여도 기다릴 게 없어서 바로 다음 줄로 넘어갑니다.

### 오해 3: "mutation 후에는 항상 await을 붙여야 한다"

```tsx
const { mutate } = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // await 없어도 괜찮습니다
    queryClient.invalidateQueries({ queryKey: ['todos'] })
    // React Query가 알아서 UI를 업데이트합니다
  }
})
```

대부분의 경우 await이 필요 없습니다. useQuery를 사용하는 컴포넌트들이 알아서 새 데이터를 가져와서 화면을 업데이트하기 때문입니다.

## 정리

`invalidateQueries`에 대해 꼭 기억해야 할 세 가지:

1. **캐시 무효화는 항상 즉시 일어난다** - await 붙이든 안 붙이든 상관없음
2. **await은 refetch를 기다리는 것** - 캐시 무효화를 기다리는 게 아님  
3. **대부분의 경우 await은 불필요** - React Query가 알아서 UI를 업데이트함

결론: await을 붙일지 말지 고민된다면, **안 붙이는 게 맞을 확률이 높습니다**. 정말로 refetch 완료를 기다려야 하는 특별한 이유가 있을 때만 await을 사용하세요.

## 참고 자료

- [TanStack Query - invalidateQueries 공식 문서](https://tanstack.com/query/latest/docs/reference/QueryClient#queryclientinvalidatequeries)
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)