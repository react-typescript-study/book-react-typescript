# 9장 훅

훅(Hook)은 함수 컴포넌트에서 상태, 생명주기, ref 같은 리액트 기능을 사용할 수 있게 해주는 함수다. 9장에서는 상태가 Effect의 실행으로 이어지는 흐름과, 렌더링을 일으키지 않는 ref의 특성을 중심으로 살펴본다.

## 9.1 리액트 훅

### 훅의 규칙

리액트는 훅의 호출 순서를 기준으로 여러 렌더링 사이의 상태를 연결한다. 따라서 다음 규칙을 지켜야 한다.

- 훅은 함수 컴포넌트 또는 커스텀 훅의 최상위에서 호출한다.
- 반복문, 조건문, 중첩 함수, `try-catch-finally` 블록 안에서 호출하지 않는다.
- 이벤트 핸들러나 클래스 컴포넌트에서 호출하지 않는다.
- 일반 자바스크립트 함수가 다른 훅을 호출한다면 이름을 `use`로 시작하는 커스텀 훅으로 만든다.

```tsx
function Profile({ isLoggedIn }: { isLoggedIn: boolean }) {
  // 조건문보다 먼저, 항상 같은 순서로 호출한다.
  const [user, setUser] = useState<User | null>(null);

  if (!isLoggedIn) {
    return <Login />;
  }

  return <UserInfo user={user} />;
}
```

다음 코드는 렌더링마다 훅의 호출 여부가 달라질 수 있으므로 잘못된 사용이다.

```tsx
if (isLoggedIn) {
  const [user, setUser] = useState<User | null>(null); // 잘못된 사용
}
```

예외적으로 React 19의 `use` API는 조건문과 반복문 안에서 호출할 수 있다. 다만 `use` 역시 컴포넌트나 커스텀 훅 안에서 호출해야 하며 `try-catch` 안에서는 호출할 수 없다.

### 상태와 반응성

#### `useState`

`useState`는 컴포넌트에 상태를 추가한다. 초기값만으로 타입을 충분히 추론할 수 있다면 제네릭을 생략하고, 원하는 타입을 추론할 수 없다면 타입 인수를 명시한다.

```tsx
const [count, setCount] = useState(0); // number
const [name, setName] = useState(""); // string

interface User {
  id: number;
  name: string;
}

const [user, setUser] = useState<User | null>(null);
```

빈 객체를 `User`로 단언하면 실제 값과 타입이 달라진다. 값이 아직 없다는 상태를 `null`이나 `undefined`로 정확하게 나타내는 편이 안전하다.

```tsx
const [user, setUser] = useState<User>({} as User); // 지양
const [user, setUser] = useState<User | null>(null); // 권장
```

이전 상태로부터 다음 상태를 계산한다면 함수형 업데이트를 사용한다.

```tsx
setCount((previousCount) => previousCount + 1);
```

초기값 계산 비용이 크다면 초기화 함수를 전달한다. 초기화 함수는 최초 렌더링의 초기 상태를 만들 때 사용된다.

```tsx
const [items, setItems] = useState<Item[]>(() => createInitialItems());
```

#### 의존성 배열

`useEffect`, `useLayoutEffect`, `useMemo`, `useCallback` 같은 훅은 의존성 배열을 받는다. 의존성은 단순히 “이 훅을 다시 실행할 조건”이 아니라 훅 내부에서 읽는 반응형 값의 목록이다.

반응형 값에는 다음이 포함된다.

- props와 state
- 컴포넌트 본문에서 선언한 변수
- 컴포넌트 본문에서 선언한 함수

리액트는 이전 의존성과 현재 의존성을 `Object.is`로 비교한다.

```tsx
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();

  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

의존성 배열에서 값을 임의로 빼면 Effect가 오래된 props나 state를 읽는 stale closure 문제가 생길 수 있다. 린트 규칙을 끄기보다 코드를 바꾸어 의존성을 제거해야 한다.

- 이전 state만 필요하면 함수형 업데이트를 사용한다.
- 렌더링과 무관한 값은 컴포넌트 밖으로 이동한다.
- Effect 안에서만 사용하는 객체나 함수는 Effect 안에서 선언한다.
- 정말 캐시가 필요할 때만 `useMemo`나 `useCallback`으로 참조를 안정화한다.

```tsx
// 매 렌더링마다 options가 새 객체가 되어 Effect가 다시 실행된다.
const options = { serverUrl, roomId };

useEffect(() => {
  const connection = createConnection(options);
  return () => connection.disconnect();
}, [options]);
```

```tsx
// Effect 안에서 만들면 실제 반응형 값만 의존성으로 남는다.
useEffect(() => {
  const options = { serverUrl, roomId };
  const connection = createConnection(options);
  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

#### `useMemo`와 `useCallback`

`useMemo`와 `useCallback`도 의존성 배열을 사용한다. 두 훅은 의존성이 바뀌지 않았다면 이전 렌더링에서 저장한 값이나 함수 참조를 재사용한다.

- `useMemo`: 계산 결과를 캐시한다.
- `useCallback`: 함수 자체를 캐시한다.

```tsx
const visibleProducts = useMemo(
  () => products.filter((product) => product.category === category),
  [products, category],
);

const handleSelect = useCallback((id: Product["id"]) => {
  setSelectedId(id);
}, []);
```

`useCallback`은 다음 `useMemo`와 같은 관계로 이해할 수 있다.

```tsx
useCallback(callback, dependencies);
useMemo(() => callback, dependencies);
```

`useMemo`는 계산한 값을 저장하고 `useCallback`은 실행할 함수를 저장한다. 둘 다 의존성 중 하나라도 바뀌면 새 값이나 함수를 만든다.

주요 사용 사례는 다음과 같다.

- 렌더링마다 반복되는 비용이 큰 계산을 건너뛴다.
- `memo`로 감싼 자식에게 이전과 같은 객체나 함수 참조를 전달한다.
- 다른 훅의 의존성으로 사용되는 값이나 함수의 참조를 안정화한다.

```tsx
const ProductList = memo(function ProductList({
  products,
  onSelect,
}: {
  products: Product[];
  onSelect: (id: Product["id"]) => void;
}) {
  return products.map((product) => (
    <button key={product.id} onClick={() => onSelect(product.id)}>
      {product.name}
    </button>
  ));
});
```

메모이제이션은 의미를 보장하기 위한 장치가 아니라 성능 최적화다. 계산 비용이 작거나 참조를 유지할 이유가 없다면 사용하지 않는 편이 단순하다. 특히 자식 컴포넌트가 메모이제이션되어 있지 않다면 콜백만 `useCallback`으로 감싸도 자식의 리렌더링은 막을 수 없다.

의존성은 Effect와 마찬가지로 콜백 안에서 읽는 반응형 값을 모두 포함해야 한다. 의존성을 줄이려고 필요한 값을 누락해서는 안 된다.

#### `useEffect`와 `useLayoutEffect`

두 훅 모두 컴포넌트를 외부 시스템과 동기화하지만 실행 시점이 다르다.

| 구분 | `useEffect` | `useLayoutEffect` |
| --- | --- | --- |
| 실행 시점 | 일반적으로 브라우저가 화면을 그린 뒤 | DOM 커밋 뒤, 브라우저가 그리기 전 |
| 화면 그리기 차단 | 하지 않음 | 함 |
| 대표 용도 | 네트워크 연결, 이벤트 구독, 타이머 | DOM 크기 측정 후 즉시 재배치 |

`useEffect`는 다음 순서로 동작한다.

1. 컴포넌트가 커밋되면 setup 함수가 실행된다.
2. 의존성이 바뀌면 이전 값으로 cleanup을 실행한 뒤 새 값으로 setup을 실행한다.
3. 컴포넌트가 제거되면 마지막 cleanup을 실행한다.

```tsx
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();

  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

의존성 배열 작성 방식에 따른 차이는 다음과 같다.

| 작성 방식 | 실행 시점 |
| --- | --- |
| 생략 | 모든 커밋 이후 |
| `[]` | 마운트 이후 |
| `[a, b]` | 마운트 이후와 `a` 또는 `b` 변경 이후 |

개발 환경의 Strict Mode에서는 setup과 cleanup이 대칭인지 검증하기 위해 마운트 전에 `setup → cleanup`을 한 차례 더 실행할 수 있다.

`useLayoutEffect`는 잘못된 배치가 잠깐이라도 화면에 보이면 안 되는 경우에 사용한다.

```tsx
function Tooltip() {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    const nextHeight = ref.current?.getBoundingClientRect().height ?? 0;
    setHeight(nextHeight);
  }, []);

  return <div ref={ref}>높이: {height}</div>;
}
```

`useLayoutEffect`의 코드와 그 안에서 발생한 상태 업데이트는 페인팅을 막는다. 화면이 깜빡이는 문제를 막기 위해 반드시 페인트 전에 처리해야 하는 작업이 아니라면 `useEffect`를 사용한다.

Effect가 필요하지 않은 경우도 구분해야 한다.

- props나 state로 계산할 수 있는 값은 렌더링 중 계산한다.
- 사용자 행동에 따른 로직은 이벤트 핸들러에서 실행한다.
- 외부 시스템과 동기화할 때 Effect를 사용한다.

```tsx
// 불필요한 Effect
const [fullName, setFullName] = useState("");
useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);

// 렌더링 중 계산
const fullName = `${firstName} ${lastName}`;
```

### ref와 명령형 제어

#### `useRef`

`useRef`는 렌더링 사이에 값을 유지하는 `{ current: T }` 형태의 객체를 반환한다. `ref.current`를 변경해도 리렌더링은 발생하지 않는다.

대표적인 용도는 두 가지다.

1. DOM 요소에 접근하기
2. 타이머 ID나 이전 값처럼 렌더링에 필요하지 않은 값을 보관하기

```tsx
function SearchInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>포커스</button>
    </>
  );
}
```

DOM ref는 첫 렌더링 시 아직 DOM 노드가 연결되지 않았으므로 `null`을 포함한다. 따라서 접근할 때 null 검사를 해야 한다.

```tsx
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();
```

#### ref 전달하기

부모가 자식 컴포넌트 내부의 DOM에 접근해야 할 때 ref를 전달한다. 책에서 사용하는 React 18 이하 방식에서는 `forwardRef`를 사용한다.

```tsx
interface InputProps extends React.ComponentPropsWithoutRef<"input"> {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, ...props }, ref) => (
    <label>
      {label}
      <input {...props} ref={ref} />
    </label>
  ),
);
```

```tsx
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <>
      <Input ref={inputRef} label="이름" />
      <button onClick={() => inputRef.current?.focus()}>이름 입력</button>
    </>
  );
}
```

React 19부터는 함수 컴포넌트가 `ref`를 prop으로 받을 수 있다. 기존 코드와 책의 예제를 읽을 때는 `forwardRef` 방식을 이해하고, 새 코드에서는 사용하는 React 버전과 팀의 기준에 맞춘다.

#### `useImperativeHandle`

ref를 그대로 전달하면 부모가 내부 DOM의 모든 기능에 접근한다. `useImperativeHandle`을 사용하면 부모에게 노출할 명령형 API만 제한할 수 있다.

```tsx
interface InputHandle {
  focus: () => void;
  clear: () => void;
}

interface CustomInputProps {
  label: string;
}

const CustomInput = forwardRef<InputHandle, CustomInputProps>(
  ({ label }, forwardedRef) => {
    const inputRef = useRef<HTMLInputElement>(null);

    useImperativeHandle(
      forwardedRef,
      () => ({
        focus: () => inputRef.current?.focus(),
        clear: () => {
          if (inputRef.current) inputRef.current.value = "";
        },
      }),
      [],
    );

    return <input ref={inputRef} aria-label={label} />;
  },
);
```

```tsx
const inputRef = useRef<InputHandle>(null);

<CustomInput ref={inputRef} label="검색어" />;
<button onClick={() => inputRef.current?.focus()}>검색창으로 이동</button>;
```

부모와 자식의 결합도를 높이는 API이므로 포커스, 스크롤, 텍스트 선택처럼 명령형 동작이 필요한 경우에 제한해서 사용한다. 값과 이벤트는 우선 props로 표현하는 것이 좋다.

#### `useRef`의 특성

`useState`와 `useRef`는 모두 렌더링 사이에 값을 보존하지만 값 변경이 렌더링에 미치는 영향이 다르다.

| 구분 | `useState` | `useRef` |
| --- | --- | --- |
| 렌더링 사이 값 유지 | O | O |
| 값 변경 시 리렌더링 | O | X |
| 렌더링 중 값 읽기 | 화면을 그리는 데 사용 | 원칙적으로 초기화 외에는 지양 |
| 주 용도 | 화면을 결정하는 상태 | DOM 참조, 타이머 ID, 외부 객체 |

화면에 표시되는 값을 ref에만 저장하면 값이 바뀌어도 화면이 갱신되지 않는다.

```tsx
function Counter() {
  const countRef = useRef(0);

  return (
    <button onClick={() => countRef.current++}>
      {countRef.current}
    </button>
  );
}
```

반대로 렌더링에 필요하지 않은 타이머 ID를 state에 저장하면 불필요한 렌더링이 발생한다.

```tsx
const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

function stopTimer() {
  if (timerRef.current !== null) {
    clearInterval(timerRef.current);
    timerRef.current = null;
  }
}
```

ref는 escape hatch다. 렌더링 중 `ref.current`를 임의로 읽거나 쓰면 컴포넌트가 순수 함수라는 전제가 깨질 수 있다. DOM 접근이나 외부 시스템 조작은 이벤트 핸들러 또는 Effect에서 수행한다.

## 9.2 커스텀 훅

커스텀 훅은 여러 컴포넌트가 공유하는 상태 로직을 함수로 추출한 것이다. 상태 자체를 공유하는 것이 아니라 상태를 다루는 로직을 재사용한다. 같은 커스텀 훅을 두 번 호출하면 각각 독립된 상태가 만들어진다.

### 커스텀 훅의 타입

매개변수와 반환값을 명시하면 사용하는 컴포넌트가 구현 세부 사항을 몰라도 안전하게 사용할 수 있다.

```tsx
interface UseToggleReturn {
  value: boolean;
  on: () => void;
  off: () => void;
  toggle: () => void;
}

function useToggle(initialValue = false): UseToggleReturn {
  const [value, setValue] = useState(initialValue);

  return {
    value,
    on: () => setValue(true),
    off: () => setValue(false),
    toggle: () => setValue((previous) => !previous),
  };
}
```

제네릭을 사용하면 데이터 타입을 호출부까지 전달하는 재사용 가능한 훅을 만들 수 있다.

```tsx
interface UseFetchResult<T> {
  data: T | null;
  error: Error | null;
  isLoading: boolean;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      try {
        setIsLoading(true);
        const response = await fetch(url, { signal: controller.signal });
        const result = (await response.json()) as T;
        setData(result);
      } catch (caught: unknown) {
        if (caught instanceof Error && caught.name !== "AbortError") {
          setError(caught);
        }
      } finally {
        if (!controller.signal.aborted) setIsLoading(false);
      }
    }

    void fetchData();
    return () => controller.abort();
  }, [url]);

  return { data, error, isLoading };
}
```

`as T`는 서버 응답을 런타임에 검증하지 않는다. 외부 데이터의 타입 안전성이 중요하다면 스키마 검증이나 타입 가드를 함께 사용해야 한다.

### 추가 학습

다음 API는 책의 내용을 확장해 최신 React 공식 문서를 기준으로 정리했다. `useLayoutEffect`는 실행 시점 비교가 중요해 앞의 `useEffect`와 함께 설명했다.

#### `use`

`use`는 Promise나 Context 같은 리소스의 값을 읽는 API다.

```tsx
import { Suspense, use } from "react";

interface MessageProps {
  messagePromise: Promise<string>;
}

function Message({ messagePromise }: MessageProps) {
  const message = use(messagePromise);
  return <p>{message}</p>;
}

function MessageContainer({ messagePromise }: MessageProps) {
  return (
    <Suspense fallback={<p>불러오는 중...</p>}>
      <Message messagePromise={messagePromise} />
    </Suspense>
  );
}
```

Promise가 대기 중이면 컴포넌트가 일시 중단되고 가장 가까운 `Suspense`의 fallback이 표시된다. Promise가 거부되면 가장 가까운 Error Boundary가 처리한다.

Context도 읽을 수 있다. 다른 훅과 달리 `use`는 조건문과 반복문 안에서 호출할 수 있다.

```tsx
function Button({ show }: { show: boolean }) {
  if (show) {
    const theme = use(ThemeContext);
    return <button className={theme}>확인</button>;
  }

  return null;
}
```

컴포넌트나 커스텀 훅 안에서만 호출해야 하며 `try-catch` 안에서는 호출할 수 없다. Promise는 렌더링할 때마다 새로 만들기보다 프레임워크나 서버 컴포넌트에서 만들어 전달하는 편이 안정적이다.

#### `useEffectEvent`

`useEffectEvent`는 Effect 안의 비반응형 로직을 분리하면서 항상 최신 props와 state를 읽게 한다.

```tsx
function ChatRoom({ roomId, theme }: ChatRoomProps) {
  const onConnected = useEffectEvent(() => {
    showNotification("연결됨!", theme);
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on("connected", onConnected);
    connection.connect();

    return () => connection.disconnect();
  }, [roomId]);
}
```

`theme`은 알림의 최신 모양을 결정하지만 채팅방을 다시 연결할 이유는 아니다. 따라서 연결은 반응형 Effect에 남기고 알림 로직만 Effect Event로 옮긴다.

- Effect Event는 Effect 또는 다른 Effect Event 안에서만 호출한다.
- 다른 컴포넌트나 훅으로 전달하지 않는다.
- 의존성을 숨기는 편법으로 사용하지 않는다.
- Effect Event 자체는 의존성 배열에 넣지 않는다.

#### `useTransition`

`useTransition`은 일부 상태 업데이트를 긴급하지 않은 Transition으로 표시한다. 무거운 UI를 갱신하는 동안 클릭이나 입력 같은 긴급 업데이트에 먼저 응답할 수 있다.

```tsx
type Tab = "about" | "posts" | "contact";

function TabContainer() {
  const [tab, setTab] = useState<Tab>("about");
  const [isPending, startTransition] = useTransition();

  const selectTab = (nextTab: Tab) => {
    startTransition(() => setTab(nextTab));
  };

  return (
    <>
      <button onClick={() => selectTab("posts")}>게시물</button>
      {isPending && <span>화면 전환 중...</span>}
      <TabContent tab={tab} />
    </>
  );
}
```

`startTransition`은 콜백을 나중에 실행하지 않는다. 콜백을 즉시 실행하면서 그 안의 상태 업데이트를 중단 가능한 non-blocking 업데이트로 표시한다.

- `isPending`으로 진행 중인 Transition을 표시할 수 있다.
- 제어 입력의 상태 업데이트에는 사용할 수 없다.
- `setTimeout`처럼 나중에 실행되는 업데이트는 자동으로 Transition이 되지 않는다.
- 현재는 `await` 뒤의 상태 업데이트를 다시 `startTransition`으로 감싸야 한다.

#### `useSyncExternalStore`

`useSyncExternalStore`는 React 외부에서 변경되는 저장소나 브라우저 API를 구독한다.

```tsx
function subscribe(callback: () => void) {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);

  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function getServerSnapshot() {
  return true;
}

function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

- `subscribe(callback)`: 저장소를 구독하고 구독 해제 함수를 반환한다.
- `getSnapshot()`: 현재 스냅샷을 반환한다. 저장소가 바뀌지 않았다면 같은 값을 반환해야 한다.
- `getServerSnapshot()`: 서버 렌더링과 하이드레이션에 사용할 초기 스냅샷을 반환한다.

스냅샷은 불변이어야 한다. 변경 가능한 저장소라면 데이터가 바뀐 경우에만 새 불변 스냅샷을 만들고, 바뀌지 않았다면 캐시한 값을 반환한다. 일반적인 컴포넌트 상태에는 `useState`나 `useReducer`를 우선하고, React 밖에서 바뀌는 값을 연결할 때 이 훅을 사용한다.

#### 선택 기준 요약

| 필요한 일 | 사용할 API |
| --- | --- |
| Promise나 Context 값 읽기 | `use` |
| Effect 재실행 없이 최신 값을 읽는 비반응형 로직 | `useEffectEvent` |
| 무거운 상태 업데이트를 non-blocking으로 처리 | `useTransition` |
| 페인트 전에 DOM 측정 및 재배치 | `useLayoutEffect` |
| React 외부 저장소나 브라우저 API 구독 | `useSyncExternalStore` |

## 참고 자료

- [React 공식 문서: 내장 React Hook](https://ko.react.dev/reference/react/hooks)
- [React 공식 문서: 훅의 규칙](https://ko.react.dev/reference/rules/rules-of-hooks)
- [React 공식 문서: `use`](https://ko.react.dev/reference/react/use)
- [React 공식 문서: `useEffectEvent`](https://ko.react.dev/reference/react/useEffectEvent)
- [React 공식 문서: `useTransition`](https://ko.react.dev/reference/react/useTransition)
- [React 공식 문서: `useLayoutEffect`](https://ko.react.dev/reference/react/useLayoutEffect)
- [React 공식 문서: `useSyncExternalStore`](https://ko.react.dev/reference/react/useSyncExternalStore)
- [우아한 타입스크립트 with 리액트 북 스터디](https://together-we-coding.github.io/woowahan-typescript-react/)
- [우아한 타입스크립트 스터디 정리 저장소](https://github.com/NoAnyClub/elegantTypeScript/tree/main/CH09_%ED%9B%85)
