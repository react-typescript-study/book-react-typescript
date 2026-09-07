# 2장 타입 퀴즈

## 1. `NaN`과 `Infinity`

자바스크립트와 타입스크립트에서 `NaN`과 `Infinity`는 각각 어떤 타입일까요?

<details>
<summary>정답</summary>

둘 다 `number` 타입이다.

```ts
typeof NaN; // "number"
typeof Infinity; // "number"
```

</details>

## 2. `null` 또는 `undefined` 확인하기

다음 조건을 짧게 표현하면 어떻게 될까요?

```ts
value === null || value === undefined;
```

<details>
<summary>정답</summary>

```ts
value == null;
```

느슨한 동등 비교에서 `null`은 `null`과 `undefined`에만 일치한다. 다만 프로젝트의 린트 규칙에 따라 `==` 사용이 금지될 수 있다.

</details>

## 3. 구조적 서브타이핑

다음 중 타입 오류가 발생하지 않는 코드는 무엇일까요?

```ts
interface Pet {
  name: string;
}

interface Cat {
  name: string;
  age: number;
}

const pet: Pet = { name: "Mong" };
const cat: Cat = { name: "Zag", age: 2 };
```

```ts
// A
const first: Pet = cat;

// B
const second: Cat = pet;
```

<details>
<summary>정답</summary>

`A`만 가능하다.

`Cat`은 `Pet`에 필요한 `name`을 가지고 있으므로 `Pet`에 할당할 수 있다. 반대로 `Pet`에는 `Cat`에 필요한 `age`가 없으므로 `Cat`에 할당할 수 없다.

</details>

## 4. `typeof`와 `instanceof`

`typeof`와 `instanceof`는 각각 무엇을 확인하며, 언제 사용하는 것이 좋을까요?

<details>
<summary>정답</summary>

- `typeof`는 값의 원시 타입을 문자열로 확인할 때 주로 사용한다.
- `instanceof`는 생성자의 `prototype`이 객체의 프로토타입 체인에 존재하는지 확인한다.
- 문자열이나 숫자 같은 원시 값은 `typeof`, `Error`, `Date` 또는 클래스 인스턴스 같은 객체는 `instanceof`로 좁히는 것이 일반적이다.

```ts
typeof "hello" === "string";
error instanceof Error;
```

</details>

## 5. 타입 단언의 효과

`value as string`을 사용하면 런타임에 `value`가 실제 문자열로 변환될까요?

<details>
<summary>정답</summary>

아니다. 타입 단언은 컴파일러에게 개발자가 판단한 타입을 알려줄 뿐 값을 변환하거나 검증하지 않는다. 런타임 안전성이 필요하면 별도의 타입 검사가 필요하다.

</details>

## 6. `enum`과 유니온 타입

유니온 타입과 달리 `enum`을 런타임에 순회하거나 값 검증에 활용할 수 있는 이유는 무엇일까요?

<details>
<summary>정답</summary>

유니온 타입은 컴파일 과정에서 제거되는 타입이지만, 일반 `enum`은 컴파일 후에도 자바스크립트 객체 형태의 값으로 남기 때문이다.

</details>

## 7. `{}` 타입

`{}` 타입은 프로퍼티가 하나도 없는 객체만을 의미할까요?

<details>
<summary>정답</summary>

아니다. 타입스크립트에서 `{}`는 "프로퍼티가 없는 객체"를 나타내는 정확한 객체 형태가 아니라, `null`과 `undefined`를 제외한 모든 값을 허용하는 타입이다. 따라서 프로퍼티가 있는 객체도 `{}`에 할당할 수 있다.

```ts
const value: {} = { name: "Mong" }; // 가능
const count: {} = 1; // 가능
```

다만 `{}` 타입 자체에는 `name` 같은 프로퍼티가 선언되어 있지 않으므로, `value.name`처럼 해당 프로퍼티에 접근할 수는 없다. 이는 실제 값에 키가 들어가면 안 된다는 뜻이 아니라, 컴파일러가 `{}`라는 타입 정보만으로는 그 키의 존재를 보장할 수 없다는 뜻이다.

</details>
