---
title: React Router의 loader로 사용자 상태에 따른 리다이렉트 구현하기
subtitle: React Router와 Tanstack Query를 활용하여, 훌륭한 UX와 DX를 가진 사용자 상태 리다이렉트를 구현하는 방법입니다.
date: 2025-08-01
draft: false
tag:
  - tanstack-query
  - react-router
---

여러분은 이런 경험이 있으신가요?

- "로그인하지 않은 사용자가 프로필 페이지에 접근하려고 하면 로그인 페이지로 보내야 해요."
- "회원가입은 했지만 이메일 인증을 안 한 사용자는 인증 페이지로 안내해 주세요."
- "온보딩을 완료하지 않은 신규 사용자는 튜토리얼을 먼저 보여주고 싶어요."

웹 서비스를 개발하다 보면 이런 요구사항을 정말 자주 만나게 됩니다. 사용자의 상태나 권한에 따라 접근할 수 있는 페이지를 제한하고, 적절한 곳으로 안내하는 것은 모든 웹 애플리케이션의 기본적인 기능이죠.

저는 Tanstack Query와 React Router를 활용하여 인증 시스템을 구현했었지만, 서비스가 성장하면서 예상치 못한 한계에 부딪혔었는데요, 이 글에서는 제가 겪은 문제와 해결 과정을 공유하고자 합니다.

이해를 돕기 위해 다음과 같은 일반적인 웹 서비스의 사용자 플로우를 예시로 사용합니다:

'회원가입 → 온보딩 → 이메일/휴대폰 인증 → 서비스 이용'

## 기존 시스템의 문제점

### 1. 자동 리다이렉션 시스템의 탄생 배경

처음에는 API 응답에 따라 `navigate('/verify')`와 같이 개발자가 직접 다음 경로를 지정하는 방식을 사용했습니다.
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

이 방식의 문제점은 명확했습니다:
- **백엔드 의존성**: API 변경 시 프론트엔드 코드도 함께 수정해야 함
- **런타임 에러 위험**: 잘못된 경로로 이동 시도 시 에러 발생
- **중복 로직**: 여러 컴포넌트에서 동일한 리다이렉션 로직 반복

### 2. 자동 리다이렉션 시스템의 도입

이러한 문제를 해결하기 위해 **"자동 리다이렉션 시스템"**을 도입했습니다. 핵심 아이디어는 다음과 같습니다:

> "개발자가 직접 경로를 지정하지 않고, 사용자 상태가 변경되면 시스템이 자동으로 적절한 페이지로 이동시킨다"

구체적인 구현 방식:
- 사용자 상태와 경로를 매핑한 중앙 집중형 설정
- Tanstack Query로 사용자 상태를 실시간 구독
- 상태 변경 감지 시 자동으로 매핑된 경로로 리다이렉션 

```tsx
const USER_STATUS_ROUTE_MAP = {
  ONBOARDING_REQUIRED: '/onboarding',
  VERIFICATION_NEEDED: '/verify',
  ACTIVE: '/dashboard'
}

function AuthRedirectWrapper({ children, allowedUserStatus }) {
  const { data: userStatus } = useSuspenseQuery(queryOptions.getUserStatus);
  const navigate = useNavigate();

  // 현재 상태와 접근 가능 상태를 비교하여, 맞지 않으면 리다이렉트
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
        // 이 부분이 핵심: invalidate 후 새로운 user status의 refetch가 완료되면
        // 새로운 user status에 맞는 경로로 자동으로 리다이렉트 됩니다.
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

**자동 리다이렉션의 장점:**
- ✅ **일관성**: 모든 페이지에서 동일한 리다이렉션 로직 적용
- ✅ **안전성**: 잘못된 상태로 페이지에 머무를 수 없음
- ✅ **간편함**: 개발자는 상태 변경만 신경쓰면 됨
- ✅ **중앙 관리**: 리다이렉션 규칙을 한 곳에서 관리

### 3. 자동 리다이렉션의 한계점

하지만 서비스가 성장하면서 **자동 리다이렉션이 오히려 문제가 되는 상황**들이 발생했습니다:

#### 문제 1: 컨텍스트를 무시한 강제 이동
```tsx
// 사용자가 프로모션 페이지에서 인증을 완료
// → 상태가 ACTIVE로 변경
// → 자동으로 /dashboard로 이동 (❌ 프로모션 참여 완료 페이지를 보여줘야 함)
```

#### 문제 2: 세분화된 상태 구분 불가
```tsx
// 같은 VERIFICATION_NEEDED 상태여도:
// - 이메일 인증 필요 → /verify/email
// - 휴대폰 인증 필요 → /verify/phone
// 자동 시스템은 하나의 경로만 지정 가능
```

#### 문제 3: 의도하지 않은 페이지 이탈
```tsx
// 공지사항을 읽는 중 백그라운드에서 상태 변경
// → 읽던 내용을 잃고 강제로 다른 페이지로 이동
```

**핵심 문제는 "자동화"가 "제어권 상실"을 의미했다는 점입니다.** 개발자가 특정 상황에서 리다이렉션을 막거나 다른 경로로 보내고 싶어도 시스템이 이를 허용하지 않았습니다.

## 해결 방안: 수동 제어와 자동화의 균형

### 1. 핵심 아이디어: "선택적 자동화"

해결책은 **"완전 자동"에서 "선택적 수동 제어"**로 패러다임을 전환하는 것이었습니다:

> "기본적으로는 자동으로 리다이렉션하되, 필요한 경우 개발자가 제어권을 가질 수 있게 한다"

이를 위해 두 가지 전략을 조합했습니다:
1. **진입 시점 검증**: 페이지 진입 시에만 권한을 체크 (React Router Loader 활용)
2. **명시적 네비게이션**: 개발자가 의도적으로 다음 경로를 지정 가능

### 2. React Router Loader를 활용한 구현

```tsx
import { replace } from 'react-router-dom'

// 로더 함수
export const protectedRouteLoader =
  (queryClient: QueryClient) =>
  (allowedUserStatus: UserStatus[]) => () => {
    // fetchQuery: 캐시에 fresh한 데이터가 없을 때만 새로 fetch합니다.
    const { status } = await queryClient.fetchQuery(queryOptions.getUserStatus);

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

### 3. 수동 제어의 복원

이제 개발자가 다시 네비게이션을 제어할 수 있게 되었습니다:

```tsx
const Page = () => {
  const { mutate } = useMutation( ... )
  const navigate = useNavigate()
  
  const handlePromoSubmit = (data) => {
    mutate(data, {
      onSuccess: (response) => {
        // 컨텍스트에 따른 분기가 가능
        if (response.hasReward) {
          navigate('/reward-claim')  // 리워드 페이지로
        } else {
          navigate('/dashboard')      // 일반 대시보드로
        }
      }
    })
  }
}
```

**중요한 차이점:**
- **자동 리다이렉션**: 상태가 변경되면 무조건 지정된 페이지로 이동
- **수동 제어 + Loader**: 
  - 페이지 진입 시에만 권한 체크
  - 이미 페이지에 들어온 상태라면 상태가 변경되어도 유지
  - 개발자가 명시적으로 이동시킬 때만 페이지 전환

### 4. 캐시 무효화 전략 개선

새로운 방식으로 전환하면서 한 가지 문제가 생겼습니다. 기존에는 사용자 상태를 항상 구독하고 있었기 때문에, mutation 후 `await invalidateQueries()`를 호출하면 refetch가 완료될 때까지 기다릴 수 있었습니다. 이를 통해 로딩 상태를 표시하고, 데이터가 갱신된 후 페이지를 이동시킬 수 있었죠. 처음 보여드린 코드에서 `isPending`을 로딩 여부 값으로 사용할 수 있었던 것도 그 때문입니다.

하지만 loader 방식에서는 더 이상 사용자 상태를 상시 구독하지 않습니다. 따라서 `invalidateQueries()`를 호출해도 refetch가 일어나지 않고, `await`이 의미가 없어집니다. 즉, `invalidateQueries`를 호출하는 쪽에서 새로운 user status의 fetch가 완료되었는지 어떤지를 알 수가 없다는 것입니다.

이를 해결하기 위해 Tanstack Query의 `invalidateQueries` 동작 원리를 활용했습니다. `invalidateQueries`는 비동기 함수이지만, 실제로는 다음과 같이 동작합니다: ([참조1](https://tanstack.com/query/v5/docs/reference/QueryClient#queryclientinvalidatequeries)) ([참조2](https://tkdodo.eu/blog/mastering-mutations-in-react-query#awaited-promises))
1. **즉시** 해당 쿼리를 stale 상태로 표시
2. **비동기로** 활성 구독자가 있을 경우 refetch 수행

이 특성을 활용하여, `invalidateQueries` 호출 시 `await` 없이 호출함으로써, mutation 후에 캐시를 즉시 stale로 만들기만 하고 새로운 user status의 fetch는 다음 라우트에서 진행하도록 하였습니다.

또한 중요한 개선점은 **mutation 커스텀 훅에 invalidate를 내재화**시킨 것입니다. 기존에는 각 컴포넌트에서 mutation 후 수동으로 `invalidateQueries()`를 호출해야 했고, 이를 누락하면 stale한 데이터로 인한 버그가 발생할 수 있었습니다. 하지만 새로운 방식에서는 개발자의 실수로 mutation 후에 무효화를 놓치는 문제는 발생하지 않게 됩니다.

단순히 무효화를 놓치는 실수를 줄이는 것 뿐만 아니라, 코드를 더 직관적으로 만드는 효과와, 약간의 성능 향상 효과도 있습니다.

이번 변경은 user status를 `useQuery`를 통해 구독하던 것을 제거하고 대신 `queryClient`를 통해 조회하도록 하는 것이므로, `invalidateQueries()`를 호출할 시점에 user status 쿼리는 inactive 상태일 가능성이 높습니다(구독하지 않으므로).
따라서, `await`을 하더라도 refetch가 일어나지 않을 가능성이 높습니다.
이러한 상황에서, 커스텀 훅에 내재화하지 않는다면 `invalidateQueries()`를 직접 호출해야 할 텐데, 이 때 개발자가 `await`을 붙일 수도, 붙이지 않을 수도 있는 가능성을 열어주게 됩니다.
여기서 개발자가 `await`을 붙이겠다는 선택을 한다면, `await invalidateQueries()`와 같은 코드가 만들어지는데, 이는 마치 refetch가 일어나고 그것을 기다릴 것 같다는 잘못된 인상을 줍니다.
설령 refetch가 일어난다고 하더라도, 새로운 user status가 현재 라우트에서 필요하지 않으므로 그것을 기다릴 필요가 없습니다.

즉, 캐시 무효화를 커스텀 훅에 내재화함으로써, 비-직관적이고 성능 저하 여지까지 있는 코드가 작성되는 것을 방지할 수 있는 것입니다.

이러한 캐시 무효화를 내재화시키는 것의 문제점은 캐시 무효화가 진행되는 시점을 정확하게 특정할 수 없다는 것입니다.
이전 방식에서는 user status의 변화 시점, 즉 캐시 무효화 시점이 곧 페이지 이동 시점이기 때문에, 페이지 이동이 어느 시점에 이루어지는지 알 수 없어 로딩 처리 등을 할 수 없기 때문에 이 단점이 치명적입니다.
하지만 새로운 방식에서는 user status의 변화와 페이지 이동 로직이 분리되었기 때문에 이러한 단점은 문제가 되지 않습니다.

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

### 5. 로딩 상태 처리

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

### 6. Refetch on focus 기능 보존

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

### 7. 에러 처리와 로딩 처리

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

### 8. 트레이드오프와 보완 방안

이 방식이 처음 언급한 "수동 제어"의 문제로 돌아간 것처럼 보일 수 있습니다. 실제로 그런 면이 있습니다. 하지만 중요한 차이가 있습니다:

**기존 수동 방식의 문제:**
- 모든 페이지에서 권한 체크 로직 중복
- 잘못된 페이지에 머무를 수 있음
- 중앙 집중형 관리 불가능

**개선된 수동 제어 방식:**
- Loader에서 권한 체크 (중앙 집중형)
- 잘못된 페이지 진입 자체를 차단
- 필요한 경우에만 수동 제어

런타임 에러를 발생시킬 수 있다는 기존 문제의 개선을 위해, type-safe한 navigate 유틸리티를 만드는 것을 고려해볼 수 있습니다:

```tsx
// 타입 안전한 네비게이션 유틸리티
type UserStatusRoutes = {
  VERIFY: ['/verify', '/dashboard'];
  ACTIVE: ['/dashboard', '/profile', '/reward'];
} satisfies Record<UserStatus, string[]>

function navigateForStatus<T extends UserStatus>(
  navigate: NavigateFunction,
  status: T,
  route: UserStatusRoutes[T][number]
) {
  navigate(route);
}

const {data} = useSuspenseQuery(queryOptions.getUserStatus)

// 사용 예시: userStatus === 'ACTIVE' 일 때
navigateForStatus(navigate, data.userStatus, '/reward'); // ✅ OK
navigateForStatus(navigate, data.userStatus, '/profile'); // ❌ Type Error
```

## 결과 및 배운 점

### 장점

1. **유연성 향상**: 특정 페이지에서는 사용자 상태가 변경되어도 머물 수 있게 되었습니다.
2. **개발자 경험 개선**: mutation 훅에 invalidate가 내재화되어 실수를 방지할 수 있고, 캐시 무효화하는 곳과 새로운 user status를 가져오는 곳이 명확하게 분리되었습니다.
3. **성능 최적화**: 불필요한 상태 구독이 사라져 리렌더링이 줄었습니다.

### 트레이드오프

1. **수동 제어의 책임**: 개발자가 직접 네비게이션을 제어하므로 백엔드 스펙 변경에 대응이 필요합니다. 하지만 이는 **유연성의 대가**로, 타입 안전성 강화와 테스트를 통해 보완 가능합니다.
2. **초기 설정 복잡도**: Loader 설정과 권한 체크 로직 구성이 필요합니다. 하지만 한 번 설정하면 중앙 집중형 관리가 가능합니다.

## 마무리

핵심은 **완벽한 자동화가 항상 최선은 아니라는 것**입니다. 때로는 개발자에게 제어권을 돌려주는 것이 더 나은 선택일 수 있습니다. React Router의 loader와 Tanstack Query를 활용하여 다음을 달성했습니다:

- ✅ 페이지 진입 시점의 권한 체크 (안전성)
- ✅ 컨텍스트별 유연한 라우팅 (유연성)
- ✅ 중앙 집중형 권한 관리 (유지보수성)
- ✅ 명시적인 네비게이션 제어 (예측가능성)

이 글이 비슷한 고민을 하는 개발자분들께 도움이 되길 바랍니다.
