---
title: TanStack Query의 invalidateQueries는 언제 await이 필요할까?
subtitle: invalidateQueries의 동기적 stale 표시와 비동기 refetch 동작을 이해하고, await을 올바르게 사용하는 방법
date: 2025-08-15
draft: true
tag:
  - tanstack-query
  - react-query
---

## TL;DR

- `invalidateQueries`는 **await 없이 호출해도 캐시를 즉시 stale로 표시**합니다
- await은 캐시 무효화가 아닌 **refetch 완료를 기다리는 것**입니다
- 구독자가 없으면 refetch 자체가 일어나지 않아 await이 무의미합니다
- **대부분의 경우 await은 불필요**하며, 정말 필요하다면 **`prefetchQuery`가 더 명확한 대안**입니다

## 들어가며

TanStack Query를 사용하다 보면 이런 코드를 자주 보게 됩니다:

```tsx
// 어떤 개발자의 코드
await queryClient.invalidateQueries(queryOptions.getTodos)

// 다른 개발자의 코드  
queryClient.invalidateQueries(queryOptions.getTodos)
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
  if (hasActiveObserver()) {
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
queryClient.invalidateQueries(queryOptions.getTodos)
// 👆 이 줄이 실행되는 순간 캐시는 이미 stale!

// await 있는 경우  
await queryClient.invalidateQueries(queryOptions.getTodos)
// 👆 캐시가 stale로 표시되는 타이밍은 위와 동일!
// await은 단지 refetch가 완료될 때까지 기다릴 뿐
```

## 흔한 오해들

### 오해 1: "await을 안 붙이면 무효화가 안 된다"

```tsx
queryClient.invalidateQueries(queryOptions.getTodos)
// 👆 await 없어도 캐시는 즉시 stale 상태가 됩니다
```

이미 stale로 표시되었기 때문에, 다음번에 이 데이터를 사용하는 컴포넌트가 마운트되었을 때 자동으로 새로운 데이터를 가져옵니다.

### 오해 2: "await을 붙이면 항상 서버 요청을 기다린다"

```tsx
// 아무도 이 데이터를 구독하고 있지 않다면?
await queryClient.invalidateQueries({ queryKey: ['unused-query'] })
// 👆 refetch가 일어나지 않아서 즉시 완료됨
```

useQuery 등으로 해당 데이터를 구독하는 컴포넌트가 없다면, refetch 자체가 일어나지 않습니다. await을 붙여도 기다릴 게 없어서 바로 다음 줄로 넘어갑니다.

### 오해 3: "mutation 후에는 항상 await을 붙여야 한다"

```tsx
const { mutate } = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // await 없어도 괜찮습니다
    queryClient.invalidateQueries(queryOptions.getTodos)
    // React Query가 알아서 UI를 업데이트합니다
  }
})
```

**대부분의 경우 await이 필요 없습니다.** useQuery를 사용하는 컴포넌트들이 알아서 새 데이터를 가져와서 화면을 업데이트하기 때문입니다.

하지만 refetch 완료를 기다려야 할 때가 있습니다. 예를 들어 프로필 업데이트 후 대시보드로 이동하는 상황을 보겠습니다:

```tsx
// 커스텀 훅 - 프로필 업데이트
function useUpdateProfile({ onSuccess }) {
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: updateProfile,
    onSuccess: () => {
      // await 없이 바로 페이지 이동
      queryClient.invalidateQueries(queryOptions.getProfile)
      onSuccess()
    }
  })
}

// 커스텀 훅 사용
const { mutate } = useUpdateProfile({ 
  onSuccess: () => { navigate('/dashboard') }
})

// 대시보드 컴포넌트
function Dashboard() {
  // 대시보드에서 user 데이터를 사용
  const { data: user } = useQuery(queryOptions.getProfile)
  
  // 문제: 아직 refetch가 진행 중이라 stale 데이터가 보일 수 있음
  return <div>Welcome, {user.name}!</div>
}
```

이런 문제들이 발생합니다:
- 대시보드로 이동했는데 아직 이전 프로필 정보가 표시됨
- 잠시 후 refetch가 완료되면 갑자기 새 프로필로 변경됨
- Suspense 사용 시 fallback으로 빠질 수도 있음

그렇다고 await을 붙이면?

```tsx
function useUpdateProfile() {
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: updateProfile,
    onSuccess: async () => {
      await queryClient.invalidateQueries(queryOptions.getProfile)
      navigate('/dashboard')  // refetch 완료까지 대기
    }
  })
}
```

`useUpdateProfile` 커스텀 훅은, 기다리고 싶지 않을 때도 무조건 기다려야 하는 **유연하지 못한 구조**가 됩니다.

이러한 상황에서는 `prefetchQuery`를 사용하는 것이 더 명확한 대안입니다.

**왜 prefetchQuery가 더 나은가?**

- `invalidateQueries` + `await`의 문제점: 
  - 의도가 불명확합니다 ("캐시를 무효화하고 싶은 건지, 새 데이터를 기다리고 싶은 건지?")
  - 구독자 유무에 의존적입니다 (구독자가 없으면 await이 무의미)
  - 코드를 읽는 사람이 "왜 await을 붙였지?"라고 고민하게 만듭니다

- `prefetchQuery`의 장점:
  - "데이터를 미리 가져온다"는 의도가 명확합니다
  - 구독자 유무와 관계없이, stale하다면 항상 데이터를 fetch합니다
  - 코드의 의도를 정확히 전달합니다


```tsx
function useUpdateProfile() {
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: updateProfile,
    onSuccess: async () => {
      // 1. 캐시만 무효화 (await 없음) - "기존 데이터는 오래됐어"
      queryClient.invalidateQueries(queryOptions.getProfile)
      // 2. 최신 데이터를 명시적으로 가져오기 - "새 데이터가 필요해"
      await queryClient.prefetchQuery(queryOptions.getProfile)
      // 3. 캐시가 최신 데이터로 채워져 있으므로 안전하게 페이지 이동
      // Suspense fallback이나 화면 깜빡임 없음
      navigate('/dashboard')
    }
  })
}
```

## 정리

`invalidateQueries`의 핵심 동작 원리:

1. **캐시 무효화는 항상 즉시 일어난다** - await 붙이든 안 붙이든 상관없음
2. **await은 refetch를 기다리는 것** - 캐시 무효화를 기다리는 게 아님  
3. **구독자가 없으면 refetch 자체가 발생하지 않음** - await이 무의미해짐

최종 권장사항:
- await 없이 `invalidateQueries` 사용을 추천
  - await을 붙일 경우 불필요하게 refetch를 기다리게 되거나, refetch가 일어나지 않는데 일어난다는 오해를 불러일으킬 수 있음
- 페이지 이동 시, 다음 페이지에서 최신 데이터가 필요하다면 이동 전에 `prefetchQuery`로 명시적으로 가져오기
- 낙관적 업데이트(Optimistic update)를 활용하면 await 필요성을 더 줄일 수 있음
