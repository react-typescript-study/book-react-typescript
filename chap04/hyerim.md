# 4장 타입 확장하기·좁히기

## 4.1 타입 확장하기

### 타입 확장 방법

- `interface`는 `extends` 키워드로, `type`은 교차 타입 연산자 `&`로 기존 타입을 확장할 수 있다.
- 기존 인터페이스에 특정 대상만 필요한 프로퍼티를 선택 속성으로 추가할 수도 있지만, 공통 인터페이스는 그대로 유지하고 요구사항별 인터페이스를 만들어 확장하는 방법도 있다.
- 이렇게 하면 공통 타입에 불필요한 선택 속성이 늘어나는 것을 막고, 각 타입이 실제로 요구하는 프로퍼티를 더 분명하게 표현할 수 있다.

### 유니온 타입

- 유니온 타입 `A | B`는 값의 집합 관점에서 합집합으로 볼 수 있다. 즉 값은 `A` 또는 `B` 중 하나의 타입에 해당한다.
- 하지만 유니온 타입의 값이 구체적으로 어느 타입인지 확인하기 전에는, 유니온에 포함된 모든 타입이 공통으로 가지는 프로퍼티에만 접근할 수 있다.

```ts
interface CookingStep {
  orderId: string;
  price: number;
}

interface DeliveryStep {
  orderId: string;
  time: number;
  distance: string;
}

function getDeliveryDistance(step: CookingStep | DeliveryStep) {
  return step.distance;
  // 오류: `CookingStep`에는 `distance` 프로퍼티가 없음
}
```

- `step`은 `CookingStep`이거나 `DeliveryStep`이지, 두 타입을 동시에 만족한다고 보장할 수 없다.
- 따라서 두 타입의 공통 프로퍼티인 `orderId`에는 바로 접근할 수 있지만, `DeliveryStep`에만 있는 `distance`를 사용하려면 먼저 타입을 좁혀야 한다.

### 교차 타입

- 교차 타입 `A & B`는 여러 타입을 모두 만족하는 타입이다.
- 객체 타입을 교차하면 각 객체 타입의 프로퍼티를 모두 가진 하나의 타입이 만들어진다.

```ts
interface CookingStep {
  orderId: string;
  price: number;
}

interface DeliveryStep {
  orderId: string;
  time: number;
  distance: string;
}

type BaedalProgress = CookingStep & DeliveryStep;
// {
//   orderId: string;
//   price: number;
//   time: number;
//   distance: string;
// }
```

- `BaedalProgress`는 `CookingStep`과 `DeliveryStep`을 모두 만족해야 하므로 두 타입의 모든 프로퍼티를 가진다.
- 같은 이름의 프로퍼티를 서로 호환되지 않는 타입으로 선언한 객체 타입을 교차하면, 해당 프로퍼티도 두 타입을 모두 만족해야 하므로 `never`가 된다.

```ts
type DeliveryTip = {
  tip: number;
};

type Filter = DeliveryTip & {
  tip: string;
};

// Filter의 tip: number & string, 즉 never
```

#### 교차하는 타입이 서로 호환되지 않는 경우

```ts
type IdType = string | number;
type Numeric = number | boolean;

type Universal = IdType & Numeric; // number
```

- `Universal`은 `IdType`과 `Numeric`을 모두 만족하는 값만 포함한다.
  - `string`이면서 `number`인 값은 없다.
  - `string`이면서 `boolean`인 값은 없다.
  - `number`이면서 `number`인 값은 존재한다.
  - `number`이면서 `boolean`인 값은 없다.
- 따라서 두 유니온에 공통으로 포함된 `number`만 남아 `Universal`은 `number` 타입이 된다.

## 4.2 타입 좁히기 - 타입 가드

- 타입스크립트의 타입 정보는 컴파일 과정에서 제거되므로 런타임에는 존재하지 않는다.
- 따라서 타입 자체를 런타임 조건으로 사용할 수는 없다. 타입을 좁히려면 컴파일 후에도 자바스크립트 코드로 남는 방법이 필요하다.
- `typeof`, `instanceof`, `in`과 같은 자바스크립트 연산자를 사용하면 런타임에 실제 값을 검사하면서 타입스크립트가 타입을 더 구체적으로 추론하도록 만들 수 있다. 이를 타입 가드라고 한다.

### typeof와 instanceof

- 원시 타입을 확인할 때는 주로 `typeof`를 사용한다.
- 클래스를 통해 생성된 인스턴스의 타입을 판별할 때는 `instanceof`를 사용한다.

> TODO: 객체에 `typeof`를 사용하면 대부분 `"object"`가 나오는 이유와, `instanceof`가 `Array` 같은 구체적인 객체 타입을 어떻게 판별하는지 알아보기

### in 연산자를 활용한 객체의 속성 유무 구분

- `in` 연산자는 객체에 특정 프로퍼티가 존재하는지 확인하고 `true` 또는 `false`를 반환한다.
- 여러 객체 타입으로 이루어진 유니온 타입에서는 각 타입에만 존재하는 프로퍼티를 `in`으로 검사하여 조건에 따라 타입을 좁힐 수 있다.

```ts
function getDeliveryDistance(step: CookingStep | DeliveryStep) {
  if ("distance" in step) {
    return step.distance;
    // 이 블록에서 step은 DeliveryStep으로 좁혀짐
  }

  return undefined;
}
```

## 4.3 타입 좁히기 - 식별할 수 있는 유니온(Discriminated Unions)

- 식별할 수 있는 유니온은 각 타입에 서로 구분할 수 있는 판별자 프로퍼티를 추가한 유니온 타입이다.
- 판별자를 사용하면 구조적으로 비슷한 타입들의 포함 관계를 제거하고, 판별자의 값에 따라 각 타입을 명확하게 구분할 수 있다.
- 판별자 프로퍼티는 유닛 타입으로 선언해야 정상적으로 타입을 좁힐 수 있다.
  - 유닛 타입은 `null`, `undefined`, 문자열·숫자 리터럴 타입, `true`처럼 정확히 하나의 값을 나타내는 타입이다.
  - `void`, `string`, `number`처럼 여러 값을 포함할 수 있는 타입은 판별자를 위한 유닛 타입으로 사용할 수 없다.
