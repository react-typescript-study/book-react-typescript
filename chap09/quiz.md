# 9장 훅 코드 리뷰 과제

상품 검색 화면을 구현한 동료가 코드 리뷰를 요청했다. 아래 코드는 의도적으로 문제가 포함된 학습용 예제다. 정답은 포함하지 않는다.

## 요구사항

- 검색어 입력은 즉시 반응해야 한다.
- 검색 결과는 최신 검색어와 일치해야 한다.
- 검색 요청에 실패하면 오류를 표시해야 한다.
- 검색 완료 알림에는 최신 테마를 적용하되, 테마 변경만으로 다시 검색하지 않아야 한다.
- 브라우저 온라인 상태 변경을 화면에 반영해야 한다.

React 19.2 이상, TypeScript 환경을 가정한다. 기본 실행 환경은 클라이언트 렌더링이며, 상위에는 Error Boundary가 있다. `notify`는 전달받은 테마로 알림을 표시하는 함수다. API는 정상 응답 시 `Product[]` 형태의 JSON을 반환한다.

## 리뷰할 코드

```tsx
import {
  Suspense,
  use,
  useEffect,
  useEffectEvent,
  useState,
  useSyncExternalStore,
  useTransition,
} from "react";

type Product = {
  id: number;
  name: string;
};

type Theme = "light" | "dark";

declare function notify(message: string, theme: Theme): void;

async function fetchProducts(query: string): Promise<Product[]> {
  const response = await fetch(
    `/api/products?q=${encodeURIComponent(query)}`
  );

  if (!response.ok) {
    throw new Error("상품을 불러오지 못했습니다.");
  }

  return response.json();
}

// [A] 온라인 상태
function subscribeOnline(onStoreChange: () => void) {
  window.addEventListener("online", onStoreChange);
  window.addEventListener("offline", onStoreChange);

  return () => {
    window.removeEventListener("online", onStoreChange);
    window.removeEventListener("offline", onStoreChange);
  };
}

function getOnlineSnapshot() {
  return { isOnline: navigator.onLine };
}

// [B] 검색 결과
function ProductList({ query }: { query: string }) {
  const products = use(fetchProducts(query));

  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}

export default function ProductSearch({ theme }: { theme: Theme }) {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  const { isOnline } = useSyncExternalStore(
    subscribeOnline,
    getOnlineSnapshot
  );

  // [C] 검색 완료 알림
  useEffect(() => {
    let ignore = false;

    fetchProducts(query)
      .then(() => {
        if (!ignore) {
          notify("검색이 완료되었습니다.", theme);
        }
      })
      .catch(() => {
        // 오류는 결과 컴포넌트에서 처리한다고 가정
      });

    return () => {
      ignore = true;
    };
  }, [query, theme]);

  // [D] 최근 검색어 저장
  const saveRecentQuery = useEffectEvent(() => {
    localStorage.setItem("recent-query", query);
    notify("검색어를 저장했습니다.", theme);
  });

  return (
    <section>
      <p>{isOnline ? "온라인" : "오프라인"}</p>

      {/* [E] 검색어 입력 */}
      <input
        aria-label="상품 검색"
        value={query}
        onChange={(event) => {
          const nextQuery = event.currentTarget.value;

          startTransition(() => {
            setQuery(nextQuery);
          });
        }}
      />

      <button onClick={saveRecentQuery}>검색어 저장</button>

      {isPending && <p>검색 중…</p>}

      <Suspense fallback={<p>상품을 불러오는 중…</p>}>
        <ProductList query={query} />
      </Suspense>
    </section>
  );
}
```

## 문제

### 1. 입력과 Transition

`[E]`의 상태 업데이트 방식은 입력창 요구사항에 적합한가? 문제가 있다면 이유를 설명하고, 입력 상태와 결과 화면 업데이트를 어떻게 구성할지 제안하라.

### 2. Promise와 Suspense

`[B]`가 처음 렌더링된 뒤 요청이 완료되면 어떤 일이 일어날 수 있는가? 재렌더링 시 Promise의 정체성을 고려하고, React Query를 사용한다면 어떻게 개선할지 작성하라.

### 3. Effect의 책임과 의존성

`[C]`에서 테마가 변경되면 어떤 동작이 발생하는가? `[B]`와 함께 봤을 때 요청 구조에도 문제가 있는지 찾아라. 최신 테마와 실제로 표시할 검색 결과의 완료 알림을 모두 만족하는 개선 방향을 제안하라.

### 4. Effect Event의 사용 범위

`[D]`를 버튼의 `onClick`에 연결한 방식이 적절한가? 최신 값을 읽는다는 특성만으로 이 사용을 정당화할 수 있는지 설명하고 수정하라.

### 5. 외부 스토어의 스냅샷

`[A]`에서 온라인 상태가 그대로인데 React가 `getOnlineSnapshot()`을 여러 번 호출하면 어떤 값들이 반환되는가? React의 비교 방식과 연결해 문제를 설명하고 수정하라. 추가로 서버 렌더링까지 지원하려면 무엇이 필요한지도 작성하라.

## 답변 형식

각 문제마다 **문제 위치 → 발생 조건·증상 → 원인 → 개선 코드** 순서로 정리한다. 여러 결함이 함께 있으므로, 다른 결함에 가려진 동작도 각 문제에서 독립적으로 검토한다.
