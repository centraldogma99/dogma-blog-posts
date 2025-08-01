---
title: React Router loader와 Tanstack Query를 활용한 사용자 상태에 따른 리다이렉트 구현하기
subtitle: 
date: 2025-08-01
draft: true
tag:
  - tanstack-query
  - react-router
---

## 목차

1. [들어가며](#들어가며)
2. [기존 시스템의 문제점](#기존-시스템의-문제점)
   - [너무 엄격한 1:1 매핑](#1-너무-엄격한-11-매핑)
   - [예외 케이스의 등장](#2-예외-케이스의-등장)
3. [해결 방안: React Router Loader를 활용한 유연한 구조](#해결-방안-react-router-loader를-활용한-유연한-구조)
   - [핵심 아이디어: "진입 시점 검증"](#1-핵심-아이디어-진입-시점-검증)
   - [캐시 무효화 전략 개선](#2-캐시-무효화-전략-개선)
   - [로딩 상태 처리](#3-로딩-상태-처리)
   - [Refetch on focus 기능 보존](#4-refetch-on-focus-기능-보존)
   - [에러 처리와 로딩 처리](#5-에러-처리와-로딩-처리)
4. [결과 및 배운 점](#결과-및-배운-점)
   - [장점](#장점)
   - [트레이드오프](#트레이드오프)
5. [마무리](#마무리)

## 들어가며

웹 애플리케이션에서 사용자 상태에 따른 페이지 리다이렉션은 흔히 마주하는 요구사항입니다. 로그인하지 않은 사용자를 로그인 페이지로 보내거나, 온보딩을 완료하지 않은 사용자를 튜토리얼 페이지로 안내하는 등의 기능이 대표적인 예시죠.

저희 팀도 이러한 기능을 구현했지만, 서비스가 성장하면서 기존 구조의 한계가 드러나기 시작했습니다. 이 글에서는 저희가 겪은 문제와 이를 해결한 과정을 공유하고자 합니다.

## 기존 시스템의 문제점

이해를 돕기 위해 일반적인 웹 서비스의 사용자 플로우, '회원가입 → 온보딩 → 이메일/휴대폰 인증 → 서비스 이용'를 예시로 설명하겠습니다.

### 1. 너무 엄격한 1:1 매핑

저희의 초기 시스템은 다음과 같은 전제로 설계되었습니다:

> "하나의 사용자 상태는 하나의 페이지와 정확히 매칭된다"

예를 들어, 아래와 같은 방식입니다.
- `ONBOARDING_REQUIRED` (온보딩) → `/onboarding`
- `VERIFICATION_NEEDED` (이메일/휴대폰 인증) → `/verify`
- `ACTIVE` (서비스 이용) → `/dashboard`

이런 구조를 만든 배경에는 그보다 전에 있던 문제 때문입니다. 원래는 API 응답에 따라 `navigate('/verify')`와 같이 직접 다음 경로를 지정하는 방식이었습니다.
```tsx
const Page = () => {
  const { mutate, isPending } = useMutation( ... )
  ...
  // 폼을 제출하고 다음 페이지로 이동
  const handleSubmit = (data) => {
    mutate(data, {
      onSuccess: () => {
        openToast('제출 성공')
        navigate('/dashboard')
      }
    })
  }
  ...

  // mutate에 넘긴 onSuccess가 완료될 때까지 pending 상태가 됨
  <Button onClick={handleSubmit} isLoading={isPending}>제출</Button>
}
```

이러한 방식은 mutation 후의 다음 user status가 `/dashboard`에 해당하는 `ACTIVE`일 것이라는 가정 하에서만 정상동작한다는 문제가 있습니다. 만약 API의 변경으로 인해 서버에서 주는 다음 user status가 달라지는 경우 런타임 에러를 일으킬 것입니다.

이를 해결하기 위해 프론트엔드에서 다음 경로를 결정하는 로직을 제거하고, user status와 경로를 1:1로 매핑한 상수를 만들었습니다. 그리고 Tanstack Query를 활용해 사용자 상태를 실시간으로 구독하고, 상태가 변경되면 즉시 해당 페이지로 리다이렉션하는 구조를 구축했습니다.

**이 방식의 장점은 명확했습니다.** 개발자는 사용자 상태만 올바르게 invalidate 처리하면 됐고, 그러면 페이지 이동은 알아서 처리되었습니다. 잘못된 페이지에 머무르는 일도 없었고, 각 페이지에서 복잡한 리다이렉션 로직을 작성할 필요도 없었죠. 사용자 경험 측면에서도 일관성 있는 플로우를 보장할 수 있었습니다. 

```tsx
const USER_STATUS_ROUTE_MAP = {
  ONBOARDING_REQUIRED: '/onboarding',
  VERIFICATION_NEEDED: '/verify',
  ACTIVE: '/dashboard'
}

function AuthRedirectWrapper({ children, allowedUserStatus }) {
  const { data: userStatus } = useSuspenseQuery(queryOptions.getUserStatus);
  const navigate = useNavigate();

  return allowedUserStatus === userStatus ?
    children :
    <Navigate to={USER_STATUS_ROUTE_MAP[userStatus]} replace />;
}

// 페이지 컴포넌트
const Page = () => {
  const { mutate, isPending } = useMutation( ... )
  ...
  // 폼을 제출하고 다음 페이지로 이동
  const handleSubmit = (data) => {
    mutate(data, {
      onSuccess: () => {
        openToast('제출 성공')
        // 이 부분이 핵심: invalidate 후 refetch가 완료되면 새로운 user status에 맞는 경로로 자동으로 리다이렉트 됩니다.
        await queryClient.invalidateQueries()
      }
    })
  }
  ...

  // mutate에 넘긴 onSuccess가 완료될 때까지 pending 상태가 됨
  <Button onClick={handleSubmit} isLoading={isPending}>제출</Button>
}

// 라우트 상수. layout element 방식으로 접근 통제 로직 구현
{
  element:
    <AuthRedirectWrapper allowedUserStatus='VERIFY'>
      <Outlet />
    </AuthRedirectWrapper>
  children: [{
    path: '/verify',
    element: <VerifyPage />
  }]
}
```

### 2. 예외 케이스의 등장

하지만 새로운 기능들이 추가되면서 이 전제가 깨지기 시작했습니다.

실제 서비스에서는 사용자 상태가 동일하더라도 상황에 따라 다른 페이지로 안내해야 하는 경우들이 생겼습니다:

- **컨텍스트에 따른 분기**: 온보딩을 완료한 사용자(`ACTIVE`)라도, 특정 프로모션 페이지에서 액션을 완료했다면 대시보드(`/dashboard`)가 아닌 리워드 페이지로 이동해야 하는 경우
- **조건부 라우팅**: 같은 `VERIFICATION_NEEDED` 상태여도, 이메일 인증이 필요한 경우와 휴대폰 인증이 필요한 경우 다른 페이지로 안내해야 하는 경우
- **임시 페이지 유지**: 이벤트 페이지나 공지사항을 보고 있는 중에 백그라운드에서 상태가 변경되어도 현재 페이지에 머물러야 하는 경우

즉, **사용자 상태와 목적지가 더 이상 1:1 관계가 아니게 되었고**, 기존의 방식으로는 유연하게 대응하기 어려워졌습니다.

## 해결 방안: React Router Loader를 활용한 유연한 구조

### 1. 핵심 아이디어: "진입 시점 검증"

문제의 핵심은 **"언제 사용자 상태를 확인하고 리다이렉션을 결정할 것인가"** 였습니다.

기존 방식은 페이지가 렌더링된 후에도 계속 상태를 감시했지만, 새로운 방식은 **페이지에 진입하는 시점에만 검증**하도록 변경했습니다.

여기서 React Router의 `loader` 기능이 핵심 역할을 합니다.
`loader`는 라우트가 렌더링되기 전에 실행되는 함수로, 데이터를 미리 불러오거나 접근 권한을 검사하는 용도로 사용됩니다. 

이를 활용하면 사용자가 새 페이지로 이동하려 할 때만 상태를 확인하고, 이후에는 상태 변화를 신경 쓰지 않아도 됩니다.
```tsx
import { replace } from 'react-router-dom'

// 로더 함수
export const protectedRouteLoader =
  (queryClient: QueryClient) =>
  (allowedUserStatus: UserStatus[]) => () => {
    // fetchQuery: 캐시에 fresh한 데이터가 없을 때만 새로 fetch합니다.
    const { status } = await queryClient.fetchQuery(queryOptions.getUserStatus);

    // 이 페이지에 접근 가능한 status인지 확인
    if (!allowedUserStatus.includes(status)) {
      return replace(getFallbackPath(status));
    }

    return null; // 접근 허용
  };

// 라우트 상수
{
  path: '/verify'
  loader: protectedRouteLoader(queryClient)(['VERIFY'])
  element: <VerifyPage />
}

// 페이지 컴포넌트
const Page = () => {
  const { mutate, isPending } = useMutation( ... )
  const navigate = useNavigate()
  ...
  // 폼을 제출하고 다음 페이지로 이동
  const handleSubmit = (data) => {
    mutate(data, {
      onSuccess: () => {
        openToast('제출 성공')
        // 달라진 부분
        navigate('/dashboard')
      }
    })
  }
  ...

  <Button onClick={handleSubmit} isLoading={isPending}>제출</Button>
}
```

코드를 보고 의문이 생겼을 것입니다. '이거 맨 위에서 옛날 버전으로 소개했던 방식 아닌가?'

맞습니다. 사실 navigate를 명시적으로 호출한다는 것으로 한정하면 옛날 방식으로 회귀한다고 볼 수 있습니다. 그래서 위에서 언급했던 옛날 방식의 문제점을 그대로 안고 있습니다.

이를 해결하기 위해서는 type-safe한 네비게이션 유틸리티를 만들거나, 서버 응답에 다음 경로 정보를 포함시키는 등의 추가적인 개선이 필요합니다. 

하지만 그럼에도 불구하고 이 방식을 선택한 이유는, **개발자가 원하는 시점에 원하는 페이지로 이동시킬 수 있는 유연성**을 얻을 수 있기 때문입니다. 기존의 자동 리다이렉션 방식에서는 불가능했던 컨텍스트별 분기나 조건부 라우팅이 가능해진 것이죠.

### 2. 캐시 무효화 전략 개선

새로운 방식으로 전환하면서 한 가지 문제가 생겼습니다. 기존에는 사용자 상태를 항상 구독하고 있었기 때문에, mutation 후 `await invalidateQueries()`를 호출하면 refetch가 완료될 때까지 기다릴 수 있었습니다. 이를 통해 로딩 상태를 표시하고, 데이터가 갱신된 후 페이지를 이동시킬 수 있었죠. 처음 보여드린 코드에서 `isPending`을 로딩 여부 값으로 사용할 수 있었던 것도 그 때문입니다.

하지만 loader 방식에서는 더 이상 사용자 상태를 상시 구독하지 않습니다. 따라서 `invalidateQueries()`를 호출해도 refetch가 일어나지 않고, `await`이 의미가 없어집니다. 즉, `invalidateQueries`를 호출하는 쪽에서 새로운 user status의 fetch가 완료되었는지 어떤지를 알 수가 없다는 것입니다.

이를 해결하기 위해 Tanstack Query의 `invalidateQueries` 동작 원리를 활용했습니다. `invalidateQueries`는 비동기 함수이지만, 실제로는 다음과 같이 동작합니다: ([참조1](https://tanstack.com/query/v5/docs/reference/QueryClient#queryclientinvalidatequeries)) ([참조2](https://tkdodo.eu/blog/mastering-mutations-in-react-query#awaited-promises))
1. **즉시** 해당 쿼리를 stale 상태로 표시
2. **비동기로** 활성 구독자가 있을 경우 refetch 수행

이 특성을 활용하여, `invalidateQueries` 호출 시 `await` 없이 호출함으로써, mutation 후에 캐시를 즉시 stale로 만들기만 하고 새로운 user status의 fetch는 다음 라우트에서 진행하도록 하였습니다.

또한 중요한 개선점은 **mutation 커스텀 훅에 invalidate를 내재화**시킨 것입니다. 기존에는 각 컴포넌트에서 mutation 후 수동으로 `invalidateQueries()`를 호출해야 했고, 이를 누락하면 stale한 데이터로 인한 버그가 발생할 수 있었습니다. 하지만 새로운 방식에서는 개발자의 실수로 mutation 후에 무효화를 놓치는 문제는 발생하지 않게 됩니다.

단순히 개발자의 실수를 줄이기 위한 목적만은 아닙니다.
내재화하지 않는다면 `invalidateQueries()`를 직접 호출해야 할 텐데, 호출할 때 `await`을 할 수도 하지 않을 수도 있는 가능성을 열어주게 됩니다.
그런데 이번 변경은 user status를 `useQuery`를 통해 구독하지 않고 `queryClient`를 통해 조회하도록 하는 것이므로, `invalidateQueries()`를 호출할 시점에 user status 쿼리는 inactive 상태일 가능성이 높습니다. 따라서, `await`을 하더라도 refetch가 일어나지 않을 가능성이 높습니다.

그럼에도 불구하고, `invalidateQueries()`에 `await`이 붙어 있다면 코드를 보는 사람에게 마치 refetch가 일어나고 그것을 기다릴 것 같다는 인상을 줍니다. 뿐만 아니라, 설령 refetch가 일어난다고 하더라도, 새로운 user status가 현재 라우트에서 필요하지 않으므로 그것을 기다릴 필요가 없습니다.

하지만 커스텀 훅에 내재화를 시킬 경우, 캐시 무효화가 진행되는 시점을 정확하게 특정할 수 없다는 단점이 있습니다. 이전 방식에서는 user status의 변화 시점 즉 캐시 무효화 시점이 곧 페이지 이동 시점이었기 때문에, 페이지 이동이 어느 시점에 이루어지는지 알 수 없어 로딩 처리 등을 할 수 없기 때문에 이 단점이 치명적입니다. 하지만 새로운 방식에서는 user status의 변화와 페이지 이동 로직이 분리되었기 때문에 이러한 단점은 문제가 되지 않습니다.

```typescript
// Mutation 커스텀 훅에 invalidate 내재화
function useUpdateProfile(props: UseMutationOptions) {
  const queryClient = useQueryClient();
  
  return useMutation({
    ...props,
    mutationFn: updateProfile,
    onSuccess: (...args) => {
      // await 하지 않음 - 캐시만 stale로 표시
      queryClient.invalidateQueries(queryOptions.getUserStatus.queryKey);
      
      props?.onSuccess?.(...args);
    }
  });
}
```

### 3. 로딩 상태 처리

기존 방식에서는 `await invalidateQueries()`로 refetch가 완료될 때까지 기다리며 버튼에 로딩 상태를 표시할 수 있었습니다. 하지만 위에서 언급했듯이, loader를 사용할 때는 활용할 수 없는 방법입니다.

해결책은 React Router의 `useNavigation` 훅이었습니다. 이 훅은 다음 페이지의 loader가 실행되는 동안의 상태를 추적할 수 있게 해줍니다. 자세한 내용은 [useNavigation 문서](https://reactrouter.com/6.30.1/hooks/use-navigation)를 참조하세요.

```tsx
// 페이지 컴포넌트
const Page = () => {
  const { mutate, isPending } = useMutation( ... )
  const navigate = useNavigate()
  const navigation = useNavigation()
  ...
  // 폼을 제출하고 다음 페이지로 이동
  const handleSubmit = (data) => {
    mutate(data, {
      onSuccess: () => {
        openToast('제출 성공')
        navigate('/dashboard')
      }
    })
  }
  ...

  <Button
    onClick={handleSubmit}
    // 달라진 부분
    isLoading={navigation.state === 'loading'}
  >제출</Button>
}
```

이렇게 하면 mutation 후 페이지 이동이 시작되고, 다음 페이지의 loader에서 새로운 사용자 상태를 fetch하는 동안 자연스럽게 로딩 UI를 표시할 수 있습니다.

### 4. Refetch on focus 기능 보존

loader를 도입했을 때 또 다른 문제는, 사용자 정보를 가져올 때 `useQuery`를 사용하지 않게 되면서, 기존 `useQuery` 훅의 [window focus 시 refetch](https://tanstack.com/query/v4/docs/framework/react/guides/window-focus-refetching) 기능을 사용할 수 없게 되었다는 것입니다.
이 기능을 사용하지 않게 되면, 여러 탭에 서비스를 띄우고 사용하는 경우와 같이 현재 페이지 밖에서 user status가 변경될 경우 문제가 발생할 수 있습니다.

이를 해결하기 위해, 아래와 같은 간단한 코드를 통해 같은 기능을 구현하였습니다.

```typescript
export const StatusRefreshHandler = () => {
  const queryClient = useQueryClient();
  const revalidator = useRevalidator();

  useEffect(() => {
    const handleRefetch = () => {
      if (document.visibilityState === 'visible') {
        queryClient.invalidateQueries({
          queryKey: queryOptions.getUserStatus.queryKey
        });
        revalidator.revalidate(); // 라우트의 loader 재실행
      }
    };

    window.addEventListener('visibilitychange', handleRefetch);

    return () => {
      window.removeEventListener('visibilitychange', handleRefetch);
    };
  }, []);

  return <Outlet />;
};

// 라우트 상수
{
  element: <StatusRefreshHandler />
  children: [
    {
      path: '/verify'
      loader: protectedRouteLoader(queryClient)('VERIFY')
      element: <VerifyPage />
    },
    // ...
  ]
}
```

### 5. 에러 처리와 로딩 처리

#### 에러 처리: errorElement
[문서](https://reactrouter.com/6.30.1/route/error-element)
loader 실행 중 에러가 발생했을 때(예: 네트워크 오류, 서버 에러 등), React의 Error Boundary에서 catch되지 않습니다. 따라서 React Router의 `errorElement`를 사용해야 합니다.
또한, errorElement 내부에서는 `useRouteError`  훅을 통해 에러 객체를 가져올 수 있습니다.

```tsx
{
  path: '/verify',
  loader: protectedRouteLoader(queryClient)(['VERIFY']),
  element: <VerifyPage />,
  errorElement: <ErrorElement />
}

// ErrorBoundary 컴포넌트
function ErrorElement() {
  const error = useRouteError();
  
  return (
    <div>
      <h2>오류가 발생했습니다</h2>
      <p>{error.message}</p>
      <button onClick={() => window.location.reload()}>
        새로고침
      </button>
    </div>
  );
}
```

#### 로딩 처리: fallbackElement
[문서](https://reactrouter.com/6.30.1/routers/router-provider#fallbackelement)
다음 라우트의 loader가 실행되는 동안 기본적으로는 현재 화면에 머무르게 됩니다.(그 동안 위에서 언급한 `useNavigation` 훅을 통해 적절한 로딩 처리가 필요하겠죠.)
하지만 서비스에 최초 진입할 때는 '현재 화면'이 없으므로, 이때 보여줄 화면을 정의해야 합니다. 이를 위해 `RouterProvider`의 prop으로 넘기는 `fallbackElement`를 사용합니다.

이미 `Suspense`를 활용해서 네트워크 요청 중에 로딩 처리를 하고 있었겠지만, 로더를 사용할 때는 유효하지 않습니다. 왜냐하면 로더는 컴포넌트 렌더링이 시작되기 전에 실행되고 완료되므로, 렌더링 중에 suspend가 발생하지 않기 때문입니다.

```tsx
const router = createBrowserRouter( ... )

<RouterProvider
  fallbackElement={<Spinner />}
  router={router}
/>
```


## 결과 및 배운 점

### 장점

1. **유연성 향상**: 특정 페이지에서는 사용자 상태가 변경되어도 머물 수 있게 되었습니다.
2. **개발자 경험 개선**: mutation 훅에 invalidate가 내재화되어 실수를 방지할 수 있습니다.
3. **성능 최적화**: 불필요한 상태 구독이 사라져 리렌더링이 줄었습니다.

### 트레이드오프

1. **휴먼 에러 가능성**: 이전과 달리 개발자가 직접 네비게이션을 시키게 되므로, 백엔드 스펙 변경에 취약하고, 접근할 수 없는 경로로 이동시키는 실수를 할 가능성도 존재합니다.

## 마무리

사용자 인증과 라우팅은 웹 애플리케이션의 핵심 기능입니다. 처음에는 단순한 1:1 매핑으로 시작했지만, 서비스가 성장하면서 더 유연한 구조가 필요했습니다.

React Router의 loader와 Tanstack Query의 캐시 시스템을 조합하여, 안정성과 유연성을 모두 갖춘 시스템을 구축할 수 있었습니다. 비록 초기 설정이 복잡해졌지만, 장기적으로는 더 나은 사용자 경험과 개발자 경험을 제공할 수 있게 되었습니다.

이 글이 비슷한 고민을 하는 개발자분들께 도움이 되길 바랍니다.
