# 3장 고급 타입

## 3.1 타입스크립트만의 독자적 타입 시스템

### any

- `any`를 사용하면 타입스크립트가 해당 값에 대한 타입 검사를 사실상 포기한다.
- 따라서 잘못된 프로퍼티 접근이나 함수 호출도 타입 오류 없이 통과하며, 실제 값과 사용 방식이 맞지 않으면 런타임 오류가 발생한다.

```ts
const value: any = null;

value.name.toUpperCase(); // 타입 오류는 없지만 런타임에 TypeError 발생
```

```text
any 사용
→ 해당 값에 대한 타입 검사 생략
→ 잘못된 코드도 타입 오류 없이 통과
→ 런타임까지 도달
→ 실제 값과 사용 방식이 맞지 않으면 런타임 오류 발생
```

- `any`를 사용한다고 반드시 런타임 오류가 발생하는 것은 아니지만, 컴파일 단계에서 오류를 발견할 기회를 잃게 된다.
- 타입을 아직 알 수 없는 값에는 `any` 대신 `unknown`을 사용하면 타입 확인을 강제하여 이러한 실수를 줄일 수 있다.

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

- **이는 `unknown` 값의 실제 타입이 아직 확인되지 않았기 때문이다.** 값이 실제로는 문자열일 수도 있으므로, 타입을 확인하지 않은 채 `number` 변수에 할당하면 타입 안전성을 보장할 수 없다.
- 따라서 같은 이유로 `unknown` 타입의 값은 함수 호출, 프로퍼티 접근, 연산 등을 바로 수행할 수 없다. 함수인지, 어떤 프로퍼티가 있는지, 특정 연산을 지원하는 타입인지 아직 보장할 수 없기 때문이다.

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

> **호출 표현식과 실제 호출은 다르다.**
>
> 컴파일 단계에서는 `unknownFunction()`이라는 호출 표현식이 타입 검사를 받을 뿐, 함수가 실제로 호출되지는 않는다. 실제 호출은 생성된 자바스크립트가 런타임에 실행될 때 일어난다. 타입 정보는 자바스크립트로 변환되면 사라지며, 컴파일 오류가 있어도 자바스크립트를 생성하도록 설정했다면 이 예제의 함수는 런타임에 정상적으로 호출된다.
>
> `tsconfig.json`에서 `noEmitOnError`를 `false`로 설정하면 타입 오류가 있어도 자바스크립트가 생성된다.

#### unknown 사용 예시

- 외부 API 응답처럼 타입을 확신할 수 없는 값에 사용하고, 타입 가드로 실제 타입을 확인한 뒤 안전하게 처리한다.
- 서로 호환되지 않는 타입을 강제로 단언할 때 `as unknown as T`처럼 중간 타입으로 사용하기도 한다.

```ts
const value = "hello" as unknown as number;
```

- 이는 실제 값을 변환하는 것이 아니라 타입 검사를 우회하는 방식이다. 런타임의 `value`는 여전히 문자열이므로 꼭 필요한 경우에만 사용해야 한다.

### void

- **JavaScript의 `undefined`**: 함수에 `return`이 없으면 런타임에서 실제로 반환되는 값이다.
- **TypeScript의 `undefined`**: 실제 `undefined` 값의 타입이다.
- **TypeScript의 `void`**: 함수의 반환값을 사용하지 않겠다는 타입 수준의 계약이다.

```ts
function logMessage(message: string): void {
  console.log(message);
}
```

> JavaScript에 별도의 `void` 값이 있는 것은 아니다. 위 함수는 런타임에서 `undefined`를 반환하지만, TypeScript에서는 그 반환값을 사용하지 않는다는 의미로 `void`를 사용한다.

```ts
const result = logMessage("hello");
// result의 타입은 void, 런타임 값은 undefined

const undefinedValue: undefined = result;
// X: void 타입을 undefined 타입에 할당할 수 없음
```

### never

- `never`는 함수가 값을 반환할 수 없으며, **호출한 곳으로 실행 흐름도 돌아오지 않는다**는 것을 나타내는 타입이다.
  - `void` 함수는 실행을 마친 뒤 런타임에서 `undefined`를 반환하지만, **`never` 함수는 실행을 정상적으로 마치지 않는다.**
- `never`에는 가능한 값이 없으므로, `never` 타입 자신을 제외한 어떤 타입의 값도 할당할 수 없다.

```ts
let neverValue: never;

neverValue = 1; // X
neverValue = "error"; // X
```

#### never 사용 예시

- `throw`로 에러를 던지면 함수가 정상적으로 종료되지 않으므로 값을 반환한 것이 아니다.
- 무한 루프도 함수 실행이 끝나지 않아 호출한 곳으로 돌아오지 않으므로 반환 타입이 `never`가 된다.

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

- `useState`는 현재 상태와 상태 변경 함수를 항상 정해진 순서로 반환한다.
- 첫 번째 원소는 상태값, 두 번째 원소는 상태 변경 함수이며 원소의 개수도 항상 2개이므로 일반 배열보다 튜플로 표현하기 적합하다.

```ts
// useState의 반환 타입을 단순화한 형태
[S, Dispatch<SetStateAction<S>>];
```

- 초기값으로 `0`을 전달하면 `S`는 `number`로 추론된다. 따라서 첫 번째 원소는 `number`, 두 번째 원소는 `number` 상태를 변경하는 함수가 된다.

```ts
const [count, setCount] = useState(0);
// count: number
// setCount: Dispatch<SetStateAction<number>>
```

- 반환 타입을 튜플로 지정하지 않고 서로 다른 타입의 값을 배열로 반환하면, 타입스크립트는 각 원소의 위치보다 배열 전체의 원소 타입을 기준으로 추론할 수 있다.

```ts
function createState() {
  const count = 0;
  const setCount = (value: number) => value;

  return [count, setCount];
  // 반환 타입: (number | ((value: number) => number))[]
}

const [state, setState] = createState();
// 두 변수 모두 number | ((value: number) => number)로 추론됨
```

- 반면 튜플은 첫 번째 원소와 두 번째 원소의 타입을 각각 기억한다. 따라서 `useState`를 구조 분해하면 `count`는 `number`, `setCount`는 상태 변경 함수로 정확히 구분된다.

### enum

- `enum`은 관련된 상수들을 하나의 이름 아래에 묶어 관리할 때 사용한다.

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

- 숫자 enum은 각 멤버에 숫자 값을 지정한 enum이다. 첫 번째 값만 지정하면 다음 멤버부터 1씩 증가한 값이 자동으로 할당된다.
- 숫자 enum은 이름으로 값을 찾는 정방향 매핑과 값으로 이름을 찾는 역방향 매핑을 모두 지원한다.

```ts
enum NumericItemStatus {
  DELIVERY_HOLD = 1,
  DELIVERY_READY, // 2
  DELIVERING, // 3
  DELIVERED, // 4
}

// 역방향 매핑
NumericItemStatus.DELIVERY_HOLD; // 1: 이름 → 값
NumericItemStatus[1]; // "DELIVERY_HOLD": 값 → 이름
```

- 문자열 enum은 이름으로 값만 찾을 수 있으며, 값으로 이름을 찾는 역방향 매핑은 지원하지 않는다.
- 따라서 숫자 enum보다 값의 의미가 분명하고, 의도하지 않은 값을 방지하기 쉽다.

#### const enum

- `const enum`은 변수를 `const`로 선언하는 것이 아니라, `enum` 앞에 `const` 키워드를 붙이는 별도의 타입스크립트 문법이다.

```ts
const enum Direction {
  UP,
  DOWN,
}

const direction = Direction.UP;
```

- 일반 enum은 자바스크립트로 변환될 때 enum 객체를 만들지만, **`const enum`은 객체를 만들지 않고 사용한 위치에 실제 값을 직접 삽입한다.** 위 코드의 `Direction.UP`은 컴파일 후 `0`으로 대체된다.

```js
const direction = 0;
```

- 따라서 숫자 `const enum`도 런타임에 enum 객체가 없으므로 `Direction[0]`과 같은 역방향 접근이나 `Object.keys(Direction)`과 같은 순회를 할 수 없다.

#### IIFE란?

- IIFE(Immediately Invoked Function Expression)는 함수를 정의한 뒤 바로 호출하는 표현식이다. 일반 enum에서는 enum 객체를 생성하고 멤버를 채우는 초기화 작업을 즉시 실행하기 위해 사용된다.

```ts
enum Direction {
  UP,
  DOWN,
}
```

위 enum을 자바스크립트로 변환하면 다음과 같은 코드가 만들어진다.

```js
var Direction;

(function (Direction) {
  Direction[(Direction["UP"] = 0)] = "UP";
  Direction[(Direction["DOWN"] = 1)] = "DOWN";
})(Direction || (Direction = {}));
```

실행 과정은 다음과 같다.

1. `var Direction`으로 변수를 선언한다. 처음 값은 `undefined`다.
2. `Direction || (Direction = {})`를 평가한다. 아직 값이 없으므로 빈 객체 `{}`를 만들고 `Direction`에 저장한다.
3. 그 객체를 IIFE의 매개변수로 전달하고 함수를 즉시 실행한다.
4. IIFE 내부에서 `UP`, `DOWN`의 정방향·역방향 매핑을 객체에 채운다.

```js
Direction["UP"] = 0; // 이름 → 값
Direction[0] = "UP"; // 값 → 이름
```

- 객체가 먼저 완성되어 있어야 IIFE를 실행할 수 있는 것은 아니다. IIFE를 호출하는 인수 표현식에서 빈 객체를 만든 다음, 즉시 실행된 함수가 그 객체를 채운다.
- 따라서 일반 enum은 타입 정보로만 존재하지 않고, 컴파일 후에도 객체와 초기화 코드가 런타임에 남는다.
- 반면 `const enum`은 컴파일 과정에서 사용한 위치에 값이 직접 삽입되고 enum 객체는 제거된다. 초기화할 런타임 객체가 없으므로 IIFE도 생성되지 않는다.

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

### 인덱스드 액세스 타입

- 다른 타입의 특정 프로퍼티가 가지는 타입을 조회할 때 사용한다. 객체 값에 `object["key"]`로 접근하는 문법과 비슷하지만, 타입 위치에서 `Type["key"]` 형태로 작성한다.

```ts
type Example = {
  a: number;
  b: string;
  c: boolean;
};

type IndexedAccess = Example["a"]; // number
type IndexedAccess2 = Example["a" | "b"]; // number | string
type IndexedAccess3 = Example[keyof Example]; // number | string | boolean

type ExAlias = "b" | "c";
type IndexedAccess4 = Example[ExAlias]; // string | boolean
```

### 맵드 타입

- 기존 타입의 키를 순회하면서 새로운 타입을 만든다.
- `readonly`를 붙이면 모든 프로퍼티가 읽기 전용이 되고, `?`를 붙이면 모든 프로퍼티가 선택 사항이 된다.
- `-readonly`는 읽기 전용 속성을 제거하고, `-?`는 선택 속성을 제거해 필수 프로퍼티로 만든다. `-`가 키 자체를 삭제하는 것은 아니다.

```ts
type User = {
  name: string;
  age: number;
};

type ReadonlyOptional<T> = {
  readonly [K in keyof T]?: T[K];
};

type ReadonlyOptionalUser = ReadonlyOptional<User>;
// {
//   readonly name?: string;
//   readonly age?: number;
// }

type MutableRequired<T> = {
  -readonly [K in keyof T]-?: T[K];
};

type MutableRequiredUser = MutableRequired<ReadonlyOptionalUser>;
// {
//   name: string;
//   age: number;
// }
```

### 제네릭

- 내부에서 사용할 타입을 미리 고정하지 않고 타입 변수로 비워둔 뒤, 실제 사용 시점에 타입을 결정하는 방식이다.
- `any`처럼 타입 검사를 포기하는 것이 아니라, **전달된 타입 정보를 유지하면서 여러 타입에 재사용할 수 있다.**

```ts
function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}

const firstNumber = getFirst<number>([1, 2, 3]);
const firstString = getFirst(["a", "b"]); // T는 string으로 추론됨
```
