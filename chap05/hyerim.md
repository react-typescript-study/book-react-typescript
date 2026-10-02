# 5장 타입 활용하기

## 5.1 조건부 타입

### `extends`와 제네릭을 활용한 조건부 타입

- 조건부 타입 `T extends U ? X : Y`는 `T`를 `U`에 할당할 수 있으면 `X`, 그렇지 않으면 `Y` 타입으로 결정된다.
- 제네릭과 `extends`를 함께 사용하면 제네릭으로 받을 수 있는 타입의 범위를 제한할 수 있다.
- 함수나 타입이 허용하지 않는 값을 컴파일 단계에서 발견할 수 있으므로 잘못된 값이 전달되는 실수를 줄일 수 있다.

### `extends` 조건부 타입을 활용하여 반환 타입 개선하기

- 조건부 타입은 인자의 타입에 따라 반환 타입을 다르게 표현하고 싶을 때 활용할 수 있다.
- 제네릭으로 인자의 구체적인 타입을 보존하고, 조건부 타입으로 그에 대응하는 반환 타입을 결정할 수 있다.
- 반환 타입이 실제 인자의 타입에 맞게 구체적으로 추론되므로, 함수를 사용하는 쪽에서 불필요한 타입 가드나 타입 단언을 줄일 수 있다.

```ts
type PayMethod<T> = T extends "card"
  ? { card: string }
  : { account: string };

declare function getPayMethod<T extends "card" | "account">(
  type: T,
): PayMethod<T>;

const card = getPayMethod("card");
// { card: string }으로 추론

const account = getPayMethod("account");
// { account: string }으로 추론
```

### `infer`를 활용해서 타입 추론하기

- `infer`는 조건부 타입의 `extends` 절 안에서 특정 위치의 타입을 추론하여 새로운 타입 변수로 사용할 때 쓰인다.
- 조건부 타입은 삼항 연산자와 비슷한 형태를 가지며, `extends`로 조건을 검사하고 `infer`로 필요한 타입을 추출한다.

```ts
type ElementType<T> = T extends Array<infer E> ? E : never;

type StringElement = ElementType<string[]>; // string
type NumberElement = ElementType<number[]>; // number
```

- `T`가 배열 타입이면 배열 요소의 타입을 `E`로 추론해 반환하고, 배열 타입이 아니면 `never`가 된다.

## 5.2 템플릿 리터럴 타입 활용하기

- 타입에서도 자바스크립트의 템플릿 리터럴과 비슷한 문법을 사용할 수 있다.
- 문자열 리터럴 타입을 조합하여 허용할 문자열의 형태를 구체적으로 표현할 수 있다.

```ts
type HeadingNumber = 1 | 2 | 3 | 4 | 5;
type HeaderTag = `h${HeadingNumber}`;
// "h1" | "h2" | "h3" | "h4" | "h5"
```

```ts
type Vertical = "top" | "bottom";
type Horizon = "left" | "right";
type Direction = Vertical | `${Vertical}${Capitalize<Horizon>}`;
// "top" | "bottom" | "topLeft" | "topRight" | "bottomLeft" | "bottomRight"
```

## 5.3 커스텀 유틸리티 타입 활용하기

### `PickOne` 유틸리티 타입

- 여러 프로퍼티 중 하나만 받도록 만들 때 식별할 수 있는 유니온을 사용할 수 있지만, 각 경우의 타입을 일일이 선언해야 하는 불편함이 있다.
- 선택한 하나의 프로퍼티에는 원래 타입의 값을 허용하고, 나머지 프로퍼티는 선택 속성인 `undefined`로 만들면 한 번에 하나만 받는 타입을 구현할 수 있다.

```ts
type PayMethod =
  | { account: string; card?: undefined; payMoney?: undefined }
  | { account?: undefined; card: string; payMoney?: undefined }
  | { account?: undefined; card?: undefined; payMoney: string };
```

이를 커스텀 유틸리티 타입으로 일반화하면 다음과 같다.

```ts
type PickOne<T> = {
  [P in keyof T]: Record<P, T[P]> &
    Partial<Record<Exclude<keyof T, P>, undefined>>;
}[keyof T];
```

#### `PickOne<T>` 해석하기

`PickOne<T>`은 각 프로퍼티를 하나씩 선택하는 부분과, 선택하지 않은 프로퍼티를 `undefined`로 제한하는 부분으로 나누어 생각할 수 있다.

```ts
type One<T> = {
  [P in keyof T]: Record<P, T[P]>;
}[keyof T];
```

- `[P in keyof T]`는 객체 타입 `T`의 키를 하나씩 순회한다.
- `Record<P, T[P]>`는 현재 키 `P`와 해당 키의 값 타입만 가진 객체 타입을 만든다.
- 마지막의 `[keyof T]`로 매핑된 타입의 모든 값에 접근하면, `T`의 프로퍼티를 하나씩 가진 객체들의 유니온이 된다.

```ts
type Payment = {
  card: string;
  account: string;
};

type OnePayment = One<Payment>;
// { card: string } | { account: string }
```

선택하지 않은 프로퍼티를 제한하는 부분은 다음과 같다.

```ts
type ExcludeOne<T> = {
  [P in keyof T]: Partial<
    Record<Exclude<keyof T, P>, undefined>
  >;
}[keyof T];
```

- `Exclude<keyof T, P>`는 `T`의 전체 키에서 현재 선택한 키 `P`를 제외한다.
- `Record<..., undefined>`는 남은 키들의 값 타입을 `undefined`로 만든다.
- `Partial`은 남은 키들을 선택 속성으로 만들어 아예 생략할 수도 있게 한다.
- 따라서 선택하지 않은 프로퍼티는 없거나 `undefined`여야 하며, 실제 값을 가질 수 없다.

`PickOne<T>`은 두 과정을 같은 `P`에 대해 결합한다.

```ts
type PickOne<T> = {
  [P in keyof T]: Record<P, T[P]> &
    Partial<Record<Exclude<keyof T, P>, undefined>>;
}[keyof T];
```

- 각 유니온 구성원은 하나의 프로퍼티에만 원래 값 타입을 요구한다.
- 나머지 프로퍼티는 생략하거나 `undefined`만 할당할 수 있다.

```ts
type Card = { card: string };
type Account = { account: string };
type CardOrAccount = PickOne<Card & Account>;

const pickOne1: CardOrAccount = { card: "hyundai" }; // O
const pickOne2: CardOrAccount = { account: "hana" }; // O
const pickOne3: CardOrAccount = {
  card: "hyundai",
  account: undefined,
}; // O
const pickOne4: CardOrAccount = {
  card: undefined,
  account: "hana",
}; // O
const pickOne5: CardOrAccount = {
  card: "hyundai",
  account: "hana",
}; // X
```

### `NonNullable` 타입

```ts
type NonNullable<T> = T extends null | undefined ? never : T;
```

- `NonNullable<T>`는 `T`의 유니온 구성원 중 `null`과 `undefined`를 `never`로 바꾸어 제거한다.

```ts
type NullableName = string | null | undefined;
type Name = NonNullable<NullableName>; // string
```

## 5.5 `Record`의 원시 타입 키 개선하기

### `Partial`을 활용하여 정확한 타입 표현하기

- 키의 범위가 무한한 `Record` 타입은 특정 키의 값이 반드시 존재하는 것처럼 추론될 수 있다.
- 실제 객체에서 해당 키가 없을 수 있다면 `Partial`을 사용하여 값이 `undefined`일 수 있는 상태를 타입에 반영할 수 있다.

```ts
type PartialRecord<K extends keyof any, T> = Partial<Record<K, T>>;

type User = {
  name: string;
};

const users: PartialRecord<string, User> = {};
const user = users["unknown-id"]; // User | undefined
```

- 이렇게 선언하면 객체에 존재하지 않는 키로 접근한 뒤 곧바로 값을 사용하는 실수를 방지하고, 먼저 `undefined` 여부를 확인하도록 유도할 수 있다.
