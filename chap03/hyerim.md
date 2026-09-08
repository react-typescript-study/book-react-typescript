# 3장 고급 타입

## 3.1 타입스크립트만의 독자적 타입 시스템

### unknown

- `unknown`에는 어떤 타입의 값이든 할당할 수 있다.
- 반대로 `unknown` 타입의 값은 `any`나 `unknown`을 제외한 다른 타입의 변수에 바로 할당할 수 없다.

```ts
let unknownValue: unknown;

unknownValue = 100;
unknownValue = "hello world";
unknownValue = () => console.log("unknown function");

const someValue1: any = unknownValue; // O
const someValue2: number = unknownValue; // X
const someValue3: string = unknownValue; // X
```

- `unknown` 타입의 값은 함수 호출, 프로퍼티 접근, 연산 등을 바로 수행할 수 없다.

```ts
const unknownFunction: unknown = () => console.log("hello");

unknownFunction(); // 컴파일 에러: 'unknownFunction' is of type 'unknown'
```

- 위 코드는 실행한 뒤 런타임 에러가 발생하는 것이 아니라, 타입을 좁히기 전에는 컴파일 단계에서 호출 자체를 막는다.
- 사용하려면 타입 가드 등을 통해 타입을 먼저 확인해야 한다.

```ts
if (typeof unknownFunction === "function") {
  unknownFunction(); // O
}
```

#### unknown과 any의 차이

- `any`는 타입 검사를 사실상 비활성화하므로 프로퍼티 접근이나 함수 호출을 허용한다. 잘못된 사용도 컴파일을 통과하여 런타임 에러로 이어질 수 있다.
- `unknown`도 모든 값을 받을 수 있지만, 사용하기 전에 타입을 확인하도록 강제한다.
- 따라서 외부 API 응답처럼 아직 타입을 확신할 수 없는 값에는 `any`보다 `unknown`이 안전하다.
- 두 타입의 차이는 호이스팅과는 관련이 없고, 타입 호환성과 값의 사용 가능 여부에 있다.

### void

- `void`는 주로 함수가 의미 있는 반환값을 제공하지 않는다는 것을 나타낸다.

```ts
function logMessage(message: string): void {
  console.log(message);
}
```

- `void`와 `undefined`는 같은 의미가 아니다.
  - `undefined` 반환 타입은 함수가 실제로 `undefined`를 반환해야 한다는 의미다.
  - `void` 반환 타입은 호출하는 쪽에서 함수의 반환값을 사용하지 않겠다는 의미에 가깝다.
- `strictNullChecks`가 활성화된 일반적인 환경에서 `void` 타입 변수에는 `undefined`만 할당할 수 있다. `null`은 할당할 수 없다.
- 함수의 반환값이 없음을 표현하려고 `null`이나 `undefined`를 직접 사용하기보다 `void`를 사용하면 함수의 의도가 더 잘 드러난다.

### 튜플

- 튜플은 각 인덱스에 들어갈 타입과 원소 개수를 지정한 배열 타입이다.
- 원소의 위치마다 의미와 타입이 정해져 있을 때 사용한다.

```ts
let tuple: [number] = [1];

tuple = [1, 2]; // X: 원소 개수가 다름
tuple = ["1"]; // X: 원소 타입이 다름

const detail: [number, string, boolean] = [1, "string", false]; // O
```

#### useState도 튜플일까?

- `useState`의 반환값도 현재 상태와 상태 변경 함수로 구성된 튜플이다.
- 일반 배열로 추론되면 각 원소가 모두 같은 유니온 타입을 가질 수 있지만, 튜플은 인덱스별 타입을 보장한다.

```ts
const [count, setCount] = useState(0);
// count: number
// setCount: Dispatch<SetStateAction<number>>
```

### enum

- `enum`은 관련된 상수들을 하나의 이름 아래에 묶어 관리할 때 사용한다.
- 회사 코드에서도 문자열 enum을 이용해 상태값을 관리하는 방식을 많이 사용한다.

```ts
enum ItemStatus {
  DELIVERY_HOLD = "DELIVERY_HOLD",
  DELIVERY_READY = "DELIVERY_READY",
  DELIVERING = "DELIVERING",
  DELIVERED = "DELIVERED",
}

const checkItemAvailable = (itemStatus: ItemStatus) => {
  switch (itemStatus) {
    case ItemStatus.DELIVERY_HOLD:
    case ItemStatus.DELIVERY_READY:
    case ItemStatus.DELIVERING:
      return false;
    case ItemStatus.DELIVERED:
      return true;
  }
};
```

- 숫자 enum은 값에서 이름을 찾는 역방향 매핑을 지원하지만 문자열 enum은 지원하지 않는다.
- 문자열 enum은 각 멤버의 값을 명시하므로 숫자 enum보다 값의 의미가 분명하고, 의도하지 않은 값을 방지하기 쉽다.
- `const enum`은 일반 enum처럼 런타임 객체를 만들지 않고 사용하는 위치에 실제 값을 삽입한다. 따라서 역방향 접근이나 런타임 순회가 불가능하며, 빌드 환경에 따라 사용이 제한될 수 있다.

#### IIFE란?

- IIFE(Immediately Invoked Function Expression)는 정의하자마자 즉시 실행하는 함수 표현식이다.

```js
(function () {
  console.log("즉시 실행");
})();
```

- 일반 enum을 자바스크립트로 변환하면 enum 객체를 만들고 값을 채우기 위한 IIFE 형태의 코드가 생성될 수 있다.
- 즉 enum은 타입 정보로만 존재하는 것이 아니라 컴파일 후에도 런타임 코드가 남는다. enum 사용을 지양하는 팀에서는 이러한 특징을 이유 중 하나로 들기도 한다.

## 3.2 타입 조합

### 인덱스 시그니처

- 객체의 프로퍼티 이름은 미리 알 수 없지만 프로퍼티 값의 타입은 알고 있을 때 사용한다.

```ts
interface StringMap {
  [key: string]: string;
}

const messages: StringMap = {
  success: "성공",
  error: "실패",
};
```

- 모든 프로퍼티가 인덱스 시그니처에 선언된 값 타입을 만족해야 하므로, 키를 동적으로 추가하는 객체에 유용하다.

### 맵드 타입

- 기존 타입의 키를 순회하면서 새로운 타입을 만든다.
- `readonly`와 `?`를 붙이거나 `-`를 사용해 제거할 수 있다.

```ts
type ReadonlyUser<T> = {
  readonly [K in keyof T]: T[K];
};

type RequiredUser<T> = {
  [K in keyof T]-?: T[K];
};
```

## 3.3 제네릭 사용법

### 제네릭

- 내부에서 사용할 타입을 미리 고정하지 않고 타입 변수로 비워둔 뒤, 실제 사용 시점에 타입을 결정하는 방식이다.
- `any`처럼 타입 검사를 포기하는 것이 아니라, 전달된 타입 정보를 유지하면서 여러 타입에 재사용할 수 있다.

```ts
function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}

const firstNumber = getFirst<number>([1, 2, 3]);
const firstString = getFirst(["a", "b"]); // T는 string으로 추론됨
```

- 제네릭을 사용하면 입력 타입과 반환 타입의 관계를 보존할 수 있다.
