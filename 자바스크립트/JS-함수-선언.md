# JavaScript 함수 선언 방식 비교

> 태그: `#javascript` `#함수` `#this`<br>
> 작성일: 2025-11-04<br>
> 최종 수정일: 2026-06-25

## 정의

자바스크립트는 함수를 선언하는 세 가지 방식(함수 선언문, 함수 표현식, 화살표 함수)을 제공하며, 이 방식들은 호이스팅 동작과 `this` 바인딩 방식이 서로 다르다.

## 특징 / 상세

### 1. 일급 함수(First-class Function)

자바스크립트는 함수를 값처럼 다룰 수 있는 **일급 함수** 특성을 가진다. 함수를 변수에 할당하거나, 다른 함수의 인자로 넘기거나, 함수에서 반환할 수 있다.

```javascript
// 함수를 변수에 할당
const greet = function(name) {
  return `Hello, ${name}`;
};

// 함수를 인자로 전달
function executeFunc(func, value) {
  return func(value);
}

console.log(executeFunc(greet, 'Kim')); // 출력: Hello, Kim

// 함수를 반환
function makeMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = makeMultiplier(2);
console.log(double(5)); // 출력: 10
```

다른 언어와 비교하면:

- **C언어**: 함수 포인터라는 개념으로 존재하지만 제한적
- **Java**: 람다 표현식을 사용하긴 하지만, 자바스크립트만큼 언어의 근간에 녹아있지는 않음
- **JavaScript**: 함수가 일급 객체로서 언어의 핵심 특성

### 2. 세 가지 선언 방식

별도의 라이브러리 설치 없이 브라우저 콘솔(개발자 도구 → Console 탭)이나 Node.js 환경에서 바로 실행할 수 있다.

Node.js에서 테스트:

```bash
node
```

VS Code 등에서 파일로 실행:

```bash
node filename.js
```

**함수 선언문 (Function Declaration)** — `function` 키워드로 함수를 정의하는 가장 전통적인 방법:

```javascript
function sum(a, b) {
  return a + b;
}

console.log(sum(3, 5)); // 출력: 8
```

**함수 표현식 (Function Expression)** — 함수를 변수에 할당하는 방식:

```javascript
const sum = function(a, b) {
  return a + b;
};

console.log(sum(3, 5)); // 출력: 8
```

**화살표 함수 (Arrow Function)** — ES6에서 도입된 간결한 문법:

```javascript
const sum = (a, b) => a + b;

console.log(sum(3, 5)); // 출력: 8
```

표현식이 여러 줄이거나 `return`이 필요한 경우:

```javascript
const sum = (a, b) => {
  const result = a + b;
  return result;
};
```

### 3. 호이스팅(Hoisting) 동작 차이

**함수 선언문의 호이스팅** — 자바스크립트 엔진이 코드를 실행하기 전에 함수 선언문을 스코프의 최상단으로 끌어올리는 것처럼 동작한다. 실제로 코드가 물리적으로 이동하는 것은 아니고, 엔진이 그렇게 인식한다고 보면 된다:

```javascript
console.log(greet('Kim')); // 출력: Hello, Kim

function greet(name) {
  return `Hello, ${name}`;
}

// 순서를 바꿔도 정상적으로 동작
```

**함수 표현식과 TDZ (Temporal Dead Zone)** — `const`나 `let`으로 선언된 변수는 호이스팅되기는 하지만, 초기화되기 전까지 TDZ에 빠진다. 그래서 변수가 선언되기 전에 접근하려고 하면 에러가 발생한다:

```javascript
console.log(greet('Kim')); // ReferenceError: Cannot access 'greet' before initialization

const greet = function(name) {
  return `Hello, ${name}`;
};
```

변수 선언 후에는 정상 동작:

```javascript
const greet = function(name) {
  return `Hello, ${name}`;
};

console.log(greet('Kim')); // 출력: Hello, Kim
```

**화살표 함수의 호이스팅** — 함수 표현식과 동일하게 동작한다. 선언 전에 사용할 수 없다:

```javascript
console.log(sum(3, 5)); // ReferenceError

const sum = (a, b) => a + b;
```

### 4. this 바인딩의 차이

**일반 함수의 동적 this** — 함수 선언문과 함수 표현식은 호출 방식에 따라 `this`가 동적으로 결정된다:

```javascript
const obj = {
  name: 'JavaScript',
  greet: function() {
    console.log(`Hello, ${this.name}`);
  }
};

obj.greet(); // 출력: Hello, JavaScript

const greetFunc = obj.greet;
greetFunc(); // 출력: Hello, undefined (전역 객체의 name)
```

**화살표 함수의 정적 this** — 화살표 함수는 자신만의 `this`를 가지지 않는다. 함수가 선언된 시점의 외부 스코프, 즉 상위 스코프의 `this`를 그대로 물려받는다:

```javascript
const obj = {
  name: 'JavaScript',
  greet: function() {
    const arrowFunc = () => {
      console.log(`Hello, ${this.name}`);
    };
    arrowFunc();
  }
};

obj.greet(); // 출력: Hello, JavaScript
```

이 특징 때문에 콜백 함수나 객체 메서드 내부에서 함수를 사용할 때 화살표 함수가 유용하게 쓰인다.

### 5. 화살표 함수의 제약사항

화살표 함수는 간결하고 `this` 동작이 특별하지만, 일반 함수로 할 수 있는 모든 것을 대체할 수는 없다.

**생성자 함수로 사용 불가:**

```javascript
// ❌ TypeError: Arrow functions cannot be used as constructors
const Person = (name) => {
  this.name = name;
};

const person = new Person('Kim'); // TypeError
```

**arguments 객체 사용 불가:**

```javascript
// 일반 함수
function regularFunc() {
  console.log(arguments); // [1, 2, 3]
}
regularFunc(1, 2, 3);

// 화살표 함수
const arrowFunc = () => {
  console.log(arguments); // ReferenceError
};
arrowFunc(1, 2, 3);
```

대신 나머지 매개변수(`...args`)를 사용한다:

```javascript
const arrowFunc = (...args) => {
  console.log(args); // [1, 2, 3]
};
arrowFunc(1, 2, 3);
```

**클래스 관련 기능 미지원** — 화살표 함수는 `super`, `new.target` 같은 클래스 관련 기능을 지원하지 않는다.

### 6. 실무에서 자주 하는 실수

**var 사용으로 인한 예측 불가능한 동작:**

```javascript
console.log(typeof greet); // 출력: undefined (호이스팅됨)

var greet = function(name) {
  return `Hello, ${name}`;
};
```

`var`는 호이스팅되면서 `undefined`로 초기화되기 때문에, 에러 대신 예상치 못한 동작이 발생할 수 있다. 항상 `const`나 `let`을 사용한다.

**호이스팅을 잘못 이해하는 경우:**

```javascript
// ❌ 함수 표현식은 호이스팅되지 않음
console.log(sum(1, 2)); // ReferenceError

const sum = (a, b) => a + b;

// ✅ 함수 선언문 사용 또는 순서 변경
function sum(a, b) {
  return a + b;
}

console.log(sum(1, 2)); // 정상 동작
```

**화살표 함수를 객체 메서드로 사용:**

```javascript
// ❌ this가 예상과 다르게 동작
const user = {
  name: 'Kim',
  greet: () => {
    console.log(`Hello, ${this.name}`); // this는 상위 스코프를 가리킴
  }
};

user.greet(); // 출력: Hello, undefined

// ✅ 올바른 방법
const userCorrect = {
  name: 'Kim',
  greet: function() {
    console.log(`Hello, ${this.name}`);
  }
};

userCorrect.greet(); // 출력: Hello, Kim
```

같은 문제를 다른 코드 예시로 보면:

```javascript
// ❌ this가 예상과 다르게 동작
const calculator = {
  value: 0,
  add: (num) => {
    this.value += num; // this는 상위 스코프를 가리킴
  }
};

// ✅ 일반 함수 또는 메서드 축약 문법 사용
const calculatorCorrect = {
  value: 0,
  add(num) {
    this.value += num;
  }
};
```

**메서드 내부 콜백에서는 화살표 함수가 유용** — 객체 메서드 안에서 콜백을 사용할 때, 화살표 함수를 쓰면 `this`가 자동으로 유지된다:

```javascript
const timer = {
  seconds: 0,
  start: function() {
    setInterval(() => {
      this.seconds++; // 화살표 함수 덕분에 timer 객체를 가리킴
      console.log(this.seconds);
    }, 1000);
  }
};

timer.start(); // 1초마다 카운트 증가
```

일반 함수를 사용하면 `this`가 전역 객체를 가리키므로 에러가 발생한다.

**var로 인한 클로저 문제:**

```javascript
// ❌ var는 함수 스코프이고 호이스팅됨
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i); // 모두 3 출력
  }, 1000);
}

// ✅ let 사용 (블록 스코프)
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i); // 0, 1, 2 출력
  }, 1000);
}

// ✅ 화살표 함수와 let 조합
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
```

**콜백 함수에서는 화살표 함수가 편리** — 배열 메서드에서 화살표 함수를 사용하면 코드가 간결해진다:

```javascript
// 일반 함수
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(function(num) {
  return num * 2;
});

// 화살표 함수
const doubledArrow = numbers.map(num => num * 2);

console.log(doubledArrow); // 출력: [2, 4, 6, 8, 10]
```

### 7. 함수와 React 성능 최적화

함수를 어디에 어떻게 선언하느냐에 따라 React 같은 프레임워크에서 성능에 영향을 줄 수 있다. 리액트 컴포넌트는 자신의 `state`나 부모로부터 받은 `props`가 변경될 때 다시 렌더링되는데, 컴포넌트 내부에서 화살표 함수를 이벤트 핸들러로 지정하면 렌더링마다 새로운 함수가 생성된다.

```jsx
function MyComponent({ onSomeEvent }) {
  // ...
  return <button onClick={() => console.log('Clicked')}>Click me</button>
}
```

자식 컴포넌트가 불필요한 리렌더링을 막기 위해 `React.memo`로 메모이제이션되어 있다면, 매 렌더링마다 새로운 함수를 prop으로 전달받으므로 메모이제이션이 의도한 대로 동작하지 않는다:

```jsx
// 자식 컴포넌트
const Button = React.memo(({ onClick, children }) => {
  console.log('Button 렌더링됨');
  return <button onClick={onClick}>{children}</button>
});

// 부모 컴포넌트
function Parent() {
  const [count, setCount] = useState(0);

  // ❌ 매 렌더링마다 새로운 함수 생성
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <Button onClick={() => console.log('Clicked')}>
        Click me
      </Button>
    </div>
  );
}
```

`useCallback`을 사용하면 의존성 배열의 값이 변경되지 않는 한 이전에 만들었던 함수를 그대로 재사용해, 자식 컴포넌트에 항상 동일한 참조를 전달할 수 있다:

```jsx
import { useState, useCallback } from 'react';

const Button = React.memo(({ onClick, children }) => {
  console.log('Button 렌더링됨');
  return <button onClick={onClick}>{children}</button>
});

function Parent() {
  const [count, setCount] = useState(0);

  // ✅ useCallback으로 함수 메모이제이션
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // 의존성 배열이 비어있으므로 컴포넌트 생명주기 동안 동일한 함수 유지

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <Button onClick={handleClick}>Click me</Button>
    </div>
  );
}
```

함수가 외부 변수에 의존한다면 의존성 배열에 명시해야 한다:

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(2);

  // multiplier가 변경될 때만 새로운 함수 생성
  const handleClick = useCallback(() => {
    console.log('Result:', count * multiplier);
  }, [count, multiplier]); // count나 multiplier가 변경되면 함수 재생성

  return (
    <div>
      <p>Count: {count}</p>
      <p>Multiplier: {multiplier}</p>
      <Button onClick={handleClick}>Calculate</Button>
    </div>
  );
}
```

모든 함수에 `useCallback`을 쓸 필요는 없다. 과도한 메모이제이션은 코드를 복잡하게 만들고 메모리를 더 사용하게 만들 수 있다.

**useCallback을 사용해야 할 때:**
- 자식 컴포넌트가 `React.memo`로 최적화되어 있을 때
- 함수가 다른 훅의 의존성 배열에 포함될 때
- 무거운 계산이나 네트워크 요청을 트리거하는 함수일 때

**useCallback이 불필요한 경우:**
- 자식 컴포넌트가 메모이제이션되지 않았을 때
- 간단한 이벤트 핸들러일 때
- 성능 문제가 실제로 측정되지 않았을 때

### 8. 선언 방식 비교 표 및 선택 가이드

| 구분 | 호이스팅 | this 바인딩 | 생성자 사용 | arguments | 특징 |
|------|----------|-------------|-------------|-----------|------|
| 함수 선언문 | 가능 | 동적 | 가능 | 있음 | 어디서든 호출 가능, 가독성 좋음 |
| 함수 표현식 | 불가 (TDZ) | 동적 | 가능 | 있음 | 변수처럼 전달 가능, 조건부 정의 |
| 화살표 함수 | 불가 (TDZ) | 상위 스코프 고정 | 불가 | 없음 | 간결, 콜백용 최적, this 유지 |

**사용 상황별 추천:**

- 객체 메서드를 정의하거나 생성자 함수를 만들 때 → 일반 함수(함수 선언문 or 함수 표현식) 또는 클래스 문법
- 콜백 함수처럼 상위 스코프 this를 유지하면서 간단한 로직을 수행할 때 → 화살표 함수
- 최상위 레벨에서 독립적인 유틸리티 함수를 정의할 때 → 함수 선언문
- 조건부로 함수를 정의하거나 IIFE를 만들 때 → 함수 표현식

세부적으로는:

- **함수 선언문**: 최상위 레벨에서 독립적인 함수를 정의할 때, 함수 위치와 상관없이 호출하고 싶을 때, 코드 가독성을 높이고 싶을 때
  ```javascript
  function calculateTotal(price, quantity) {
    return price * quantity;
  }
  ```
- **함수 표현식**: 조건부로 함수를 정의할 때, 즉시 실행 함수(IIFE)를 만들 때, 함수를 명시적으로 변수에 할당하고 싶을 때
  ```javascript
  const calculateTax = isEligible ?
    function(amount) { return amount * 0.1; } :
    function(amount) { return 0; };
  ```
- **화살표 함수**: 콜백 함수를 작성할 때, 배열 메서드(`map`, `filter`, `reduce` 등)를 사용할 때, 상위 스코프의 `this`를 유지해야 할 때, 간단한 로직을 간결하게 표현하고 싶을 때
  ```javascript
  const numbers = [1, 2, 3, 4, 5];
  const evens = numbers.filter(num => num % 2 === 0);
  ```
  단, 객체 메서드를 정의할 때, 생성자 함수가 필요할 때, 동적 `this`가 필요할 때는 피한다.

## 트레이드오프

해당 없음 — 언어 문법 선택의 문제이며 분산 시스템 트레이드오프(일관성/가용성/지연/비용/운영부담)와는 무관. 호이스팅·this 바인딩·React 리렌더링 관점의 트레이드오프는 위 특징/상세 4, 7번 항목 참고.

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 6번 "실무에서 자주 하는 실수", 7번 "함수와 React 성능 최적화"에 포함)

## 참고

원본 학습 노트(TIL)에 `> 연결된 정리본: 함수 선언과 this` 링크(`../../../TIL.wiki/javascript-function-declarations-and-this.md`)가 있었으나, 이 볼트 외부의 위키 경로라 마크다운 링크로 옮기지 않음. 필요 시 원본 TIL 위키에서 직접 확인.

- [MDN - Functions](https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Functions)
- [MDN - Arrow functions](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
- [MDN - Hoisting](https://developer.mozilla.org/ko/docs/Glossary/Hoisting)
- [JavaScript.info - Functions](https://javascript.info/function-basics)
- [JavaScript.info - Arrow functions basics](https://javascript.info/arrow-functions-basics)
- [React - useCallback Hook](https://react.dev/reference/react/useCallback)
- [React - React.memo](https://react.dev/reference/react/memo)

## 관련 내용

- [Promise](Promise.md)
- [Event-Loop와-비동기-처리](Event-Loop와-비동기-처리.md)
- [이벤트-루프와-비동기-처리](이벤트-루프와-비동기-처리.md)
