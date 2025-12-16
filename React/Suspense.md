# Suspense

`<Suspense>`란 자식 요소를 로드하기 전까지 화면에 대체 UI Fallback를 보여주는 React 컴포넌트

## 의의

비동기 의존성으로 인해 아직 렌더링할 수 없는 UI의 렌더링 타이밍을
선언적으로 통제할 수 있게 해준다.

- Suspense 사용 X

```tsx
if (isLoading) {
  return <Spinner />;
}

return <Profile data={data} />;
```

- Suspense 사용 O

```tsx
<Suspense fallback={<Spinner />}>
  <Profile />
</Suspense>
```
