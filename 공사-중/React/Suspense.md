# Suspense

## 정의

`<Suspense>`란 자식 요소를 로드하기 전까지 화면에 대체 UI Fallback를 보여주는 React 컴포넌트

## 의의

1. 비동기 의존성으로 인해 아직 렌더링할 수 없는 UI의 렌더링 타이밍을
   선언적으로 통제할 수 있게 해준다.
2. UX적으로 하나의 의미 단위인 UI의 완성 여부 판단을
   트리 단위로 끌어올려 일관된 시점에 렌더링할 수 있게 한다.
3. Suspense 하위 컴포넌트는
   비동기 의존성이 해결된 상태에서 렌더링됨을 가정할 수 있다. (React Query의 경우 data?. 옵셔널 체이닝을 무한 반복하지 않아도 됨.)

## Suspense를 활성화 하는 조건

React 공식 문서에 따르면 Suspense는 아래와 같은 조건에 해당할 경우 지연을 감지할 수 있다고 한다.

- Relay와 Next.js 같이 Suspense가 가능한 프레임워크를 사용한 데이터 가져오기.
- lazy를 활용한 지연 로딩 컴포넌트.
- use를 사용해서 캐시된 Promise 값 읽기.

[아직 이해 못한 부분]

- React Query와 위 세 가지 케이스와의 관계.
- Suspense는 렌더 중의 모든 Promise를 감지하는가?
  - 아님. Suspense와 연결된(=Suspense-enabled) 경로에서만 그 suspend를 인정
    - 'Suspense와 연결된 경로'가 무엇인가?
- Suspense가 렌더 중의 아무 Promise나 처리하지 않는 이유
  - 아무 Promise나 처리했을 때 생기는 문제?
    - 케이스 1. 매 렌더마다 새 Promise를 throw 하는 경우 (무한 Suspense)
      ```ts
      function UserProfile({ id }: { id: string }) {
        // ❌ 렌더마다 새 Promise 생성
        throw fetch(`/api/users/${id}`).then((r) => r.json());
      }
      ```
    - 왜 React가 막아야 하나? => 두 가지 이유 모두 이해 못함. Suspense의 구체적인 매커니즘을 몰라서 그러는 듯.
      - React는 이 Promise가 “이전과 같은 작업인지” 알 수 없음
      - 캐시 없음 → 동일성 보장 실패

## (생각) 언제 사용하는 것이 좋은가?

위 Suspense의 의의를 따져보았을 때, 2번은 해당되는 경우가 특정적이지만 1, 3번은 기본적으로 이점이 되는 것 같다.

하지만, Suspense의 하위 컴포넌트에서 아무 Promise나 처리해주는 것이 아닌, Suspense를 활성화 시키는 조건에 맞는 비동기 처리(ex React Query의 useSuspenseQuery 사용)를 해야 하기 때문에 해당 컴포넌트에 Suspense 종속성이 생긴다는 단점이 있다. 재사용성이 떨어지고 파악해야 할 의존성이 생기는 것이다.

또한, 어느 위치에 Suspense 경계가 존재하는지 파악하기 어려워질 수도 있다.

이렇게 두 가지 단점이 있는데, 다음과 같이 풀어낼 수 있을 것 같다.

첫 번째, Suspense 종속성이 생겨나는 문제는 [toss의 라이브러리인 suspensive의 `<SuspenseQuery>`](https://suspensive.org/ko/docs/react-query/SuspenseQuery)와 같은 인터페이스를 통해 해결할 수 있다.

```tsx
import { SuspenseQuery } from '@suspensive/react-query';
import { Suspense, ErrorBoundary } from '@suspensive/react';
import { PostListItem, UserProfile } from '~/components';

const PostsPage = ({ userId }) => (
  <ErrorBoundary fallback={({ error }) => <>{error.message}</>}>
    <Suspense fallback={'loading...'}>
      <SuspenseQuery {...userQueryOptions(userId)}>
        {({ data: user }) => <UserProfile key={user.id} {...user} />}
      </SuspenseQuery>
      <SuspenseQuery {...postsQueryOptions(userId)} select={(posts) => posts.filter(({ isPublic }) => isPublic)}>
        {({ data: posts }) => posts.map((post) => <PostListItem key={post.id} {...post} />)}
      </SuspenseQuery>
    </Suspense>
  </ErrorBoundary>
);
```

해당 라이브러리 문서에서 가져온 예시인데 위와 같이 Suspense 의존성을 컴포넌트에서 분리하여 별개의 레이어로 관리한다.

```tsx
import type { User } from './api';

export const UserProfile = (props: User) => {
  return (
    <div style={{ padding: '16px' }}>
      <h1>{props.name}</h1>
      <p>{props.email}</p>
      <p>{props.phone}</p>
    </div>
  );
};
```

하위 컴포넌트 내부 구현을 살펴보면 Suspense에 대한 의존성이 전혀 없다.

하지만 이 패턴은 결국 데이터 fetch의 책임을 분리해내는 것이기도 한데 이것에 대한 장단점도 있을 것이어서 무조건적인 해답은 아닌 것 같다.

또한, Suspense 종속성이 생기는 것이 무조건적인 단점일까? 라고 하면 그것도 아니다.
Suspense는 React의 일반적인 렌더 패턴을 벗어나는 장치다.
그렇기 때문에 오히려 종속성이 있는 컴포넌트라는 것이 명확하게 드러나는 게 좋을 수 있다.

만약 재사용성이 중요한 컴포넌트라면 위와 같은 인터페이스를 사용하여 Suspense 종속성을 별개의 레이어로 표현하면 된다.

<br/>

두 번째 단점인 Suspense 경계가 존재하는지 어디에 예상하기 어려워질 수 있다는 것은 Suspense의 사용 의도가 명확한 코드를 작성하면 된다.

Suspense를 통해 여러 비동기 의존성을 한 번에 처리할 수 있지만 이를 무턱대고 묶으면 UX가 저하된다.

묶어서 처리해야 하는 명확한 의미 단위일 경우에만 묶어서 처리한다면 Suspense의 위치를 예상하기 어렵지 않을 것이다.

### 결론

- 잘 쓰면 써서 나쁠 것 없고 이점이 꽤 있다. (위 Suspense의 의의 참고)
- 의미 단위로 잘 묶어서 사용한다. 특별히 묶을 의미 단위가 없다면 비동기 상태 하나와 Suspense 하나의 1:1 대응이 관리하기 좋다.
- Suspense를 사용할 비동기 처리는 그 의존성을 표시할 수밖에 없고 그러는 것이 좋다. 만약, 컴포넌트 재사용성이 필요하다면 그 비동기 처리 자체를 분리해내는 방법을 사용한다. (위 suspensive 라이브러리 예시 참고)
