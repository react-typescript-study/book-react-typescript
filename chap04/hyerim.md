# 4장 타입 확장하기·좁히기

## 4.1 타입 확장하기

### 타입 확장 방법

- `interface`는 `extends` 키워드로, `type`은 교차 타입 연산자 `&`로 기존 타입을 확장할 수 있다.
- 기존 인터페이스에 특정 대상만 필요한 프로퍼티를 선택 속성으로 추가할 수도 있지만, 공통 인터페이스는 그대로 유지하고 요구사항별 인터페이스를 만들어 확장하는 방법도 있다.
- 이렇게 하면 공통 타입에 불필요한 선택 속성이 늘어나는 것을 막고, 각 타입이 실제로 요구하는 프로퍼티를 더 분명하게 표현할 수 있다.

### 유니온 타입

- 유니온 타입 `A | B`는 값의 집합 관점에서 합집합으로 볼 수 있다.
- 여기서 합집합이라는 말은 `A`의 모든 프로퍼티와 `B`의 모든 프로퍼티를 합친다는 뜻이 아니다. `A` 타입의 값과 `B` 타입의 값을 모두 허용한다는 뜻이다.
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

type BaedalStep = CookingStep | DeliveryStep;

const cooking: BaedalStep = {
  orderId: "ORDER_1",
  price: 15000,
}; // CookingStep 타입의 값이므로 대입 가능

const delivery: BaedalStep = {
  orderId: "ORDER_2",
  time: 20,
  distance: "3km",
}; // DeliveryStep 타입의 값이므로 대입 가능

function getDeliveryDistance(step: CookingStep | DeliveryStep) {
  step.orderId; // 가능: 두 타입에 모두 존재함
  return step.distance;
  // 오류: `CookingStep`에는 `distance` 프로퍼티가 없음
}
```

- `step`은 `CookingStep`이거나 `DeliveryStep`이지, **두 타입을 동시에 만족한다고 보장할 수 없다.**
- 따라서 두 타입의 공통 프로퍼티인 `orderId`에는 바로 접근할 수 있지만, `DeliveryStep`에만 있는 `distance`를 사용하려면 **먼저 타입을 좁혀야 한다.**
- 즉 **대입할 수 있는 값의 범위**와 **안전하게 접근할 수 있는 프로퍼티의 범위**를 구분해야 한다.
  - 값의 관점: `CookingStep`의 값 또는 `DeliveryStep`의 값을 받을 수 있으므로 합집합이다.
  - 프로퍼티 접근의 관점: 실제 값이 어느 타입인지 모르므로 두 타입에 공통으로 존재하는 프로퍼티만 안전하게 사용할 수 있다.
- 공통 프로퍼티에만 접근할 수 있다는 사실이 유니온을 교집합으로 만드는 것은 아니다. 합집합에 포함된 어떤 값이 들어와도 오류가 발생하지 않게 하기 위한 제약이다.

### 교차 타입

- 교차 타입 `A & B`는 여러 타입을 모두 만족하는 타입이다.
- 객체 타입을 교차하면 각 객체 타입의 프로퍼티를 모두 가진 하나의 타입이 만들어진다.
- 공통 타입에 옵셔널 프로퍼티를 추가하는 대신, 특정 타입에만 필요한 프로퍼티를 교차 타입으로 결합해 필수 속성으로 추가할 수 있다.

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

#### 교차하는 타입이 서로 호환되지 않는 경우

- `BaedalProgress`는 `CookingStep`과 `DeliveryStep`을 모두 만족해야 하므로 두 타입의 모든 프로퍼티를 가진다.
- 같은 이름의 프로퍼티를 **서로 호환되지 않는 타입으로 선언한 객체 타입을 교차**하면 해당 프로퍼티의 타입은 `never`가 된다.

```ts
type DeliveryTip = {
  tip: number;
};

type Filter = DeliveryTip & {
  tip: string;
};

// Filter의 tip: number & string, 즉 never
```

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

- 원시 타입을 추론할 때는 `typeof`를 사용하고, 인스턴스화된 객체 타입을 판별할 때는 `instanceof`를 사용한다.
- 인스턴스화란 클래스를 바탕으로 `new` 키워드를 사용해 실제 객체인 인스턴스를 만드는 것을 말한다.
- `instanceof`는 객체가 특정 클래스에서 생성된 인스턴스인지 확인한다. 타입스크립트는 이 결과를 바탕으로 객체를 해당 클래스 타입으로 좁히므로, 클래스에 선언된 프로퍼티와 메서드에 접근할 수 있다.

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

### is를 활용한 사용자 정의 타입 가드

- 타입 명제(type predicate)는 함수의 반환 타입을 `A is B` 형식으로 작성한다. `A`는 함수의 매개변수 이름이고, `B`는 좁히려는 타입이다.
- 타입 명제도 런타임에는 불리언 값을 반환하지만, **일반적인 `boolean`과 달리 반환값이 `true`일 때 매개변수 `A`를 타입 `B`로 좁혀야 한다는 정보까지 타입스크립트에 알려준다.**

```ts
function isDeliveryStep(
  step: CookingStep | DeliveryStep,
): step is DeliveryStep {
  return "distance" in step;
}

function getDeliveryDistance(step: CookingStep | DeliveryStep) {
  if (isDeliveryStep(step)) {
    return step.distance;
    // step은 DeliveryStep으로 좁혀짐
  }

  return undefined;
}
```

- 사용자 정의 타입 가드는 다음과 같은 경우에 유용하다.
  - API 응답처럼 타입의 범위가 넓거나 확실하지 않은 값을 검사할 때
  - 배열에서 특정 타입의 원소만 필터링할 때
  - 복잡한 타입 판별 로직을 별도 함수로 분리할 때
  - 검사 후 특정 타입의 프로퍼티나 메서드를 안전하게 사용해야 할 때
- 검사와 사용이 같은 위치에 있어 타입스크립트가 조건문을 직접 분석하여 타입을 좁힐 수 있다면 사용자 정의 타입 가드를 만들 필요가 없다.

## 4.3 타입 좁히기 - 식별할 수 있는 유니온(Discriminated Unions)

- 식별할 수 있는 유니온은 각 타입에 서로 구분할 수 있는 판별자 프로퍼티를 추가한 유니온 타입이다.
- 판별자를 사용하면 구조적으로 비슷한 타입들의 포함 관계를 제거하고, 판별자의 값에 따라 각 타입을 명확하게 구분할 수 있다.
- 판별자 프로퍼티는 유닛 타입으로 선언해야 정상적으로 타입을 좁힐 수 있다.
  - 유닛 타입은 `null`, `undefined`, 문자열·숫자 리터럴 타입, `true`처럼 정확히 하나의 값을 나타내는 타입이다.
  - `void`, `string`, `number`처럼 여러 값을 포함할 수 있는 타입은 판별자를 위한 유닛 타입으로 사용할 수 없다.
