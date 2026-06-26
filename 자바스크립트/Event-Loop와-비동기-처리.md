# Event Loop로 시작하는 JavaScript 비동기 처리

> 태그: `#javascript` `#event-loop` `#비동기`<br>
> 작성일: 2025-11-05<br>
> 최종 수정일: 2026-06-25

## 정의

Event Loop는 싱글스레드인 JavaScript가 Call Stack, Web API, Task Queue(Macrotask/Microtask)를 오가며 비동기 작업의 콜백을 실행 순서대로 처리하는 메커니즘이다.

## 특징 / 상세

> 카페에서 손님이 몰려도 바리스타 한 명이 효율적으로 음료를 만들어내는 방식에 빗대어 Event Loop를 설명하는 노트. 아래는 그 비유를 유지한 채 정리한 내용이다.

### 1. Event Loop가 뭔가요?

바리스타 한 명이 운영하는 작은 카페를 떠올려보자. 손님이 한 명씩 와서 음료를 주문하면, 바리스타는 한 번에 한 가지 일만 할 수 있다. 그런데 신기하게도 손님이 많아져도 카페는 잘 돌아간다. 왜일까?

바리스타가 똑똑하게 일을 처리하기 때문이다. 오래 걸리는 작업(예: 에스프레소 추출, 우유 데우기)은 기계에 맡겨두고, 그 동안 다른 손님 주문을 받는다. 기계가 작업을 끝내면 신호를 주고, 바리스타는 그때 다시 그 작업을 마무리한다.

**Event Loop**가 정확히 이런 역할을 한다. JavaScript는 싱글스레드(바리스타 한 명)지만, 시간이 걸리는 작업(네트워크 요청, 타이머 등)을 브라우저나 Node.js 런타임(기계)에 맡기고, 그 결과가 준비되면 적절한 시점에 콜백을 실행해주는 관리자 역할을 한다.

### 2. 왜 Event Loop를 알아야 할까요?

1. **비동기 코드의 실행 순서를 예측할 수 있다** — `setTimeout`, `Promise`, `fetch` 등이 어떤 순서로 실행되는지 정확히 이해하게 됨
2. **버그를 디버깅할 수 있다** — "분명 순서대로 썼는데 왜 결과가 이상하게 나오지?"라는 상황을 해결할 수 있음
3. **성능 최적화를 할 수 있다** — 무거운 작업이 메인 스레드를 막지 않도록 설계할 수 있음
4. **면접에서 자주 나온다** — JavaScript 비동기 처리는 프론트엔드/Node.js 개발자 면접의 단골 주제

### 3. JavaScript 비동기 처리의 핵심 구성 요소

**1. Call Stack (호출 스택)**

코드가 실행되는 장소다. 함수가 호출되면 스택에 쌓이고, 함수가 끝나면 스택에서 제거된다(LIFO, Last In First Out). 바리스타가 지금 손에 들고 처리 중인 작업이라고 보면 된다.

```javascript
function a() {
  b();
}
function b() {
  console.log('b 실행');
}
a();

// Call Stack 변화:
// [a] → [a, b] → [a] → []
```

**2. Web API (브라우저/Node.js 런타임이 제공)**

`setTimeout`, `fetch`, DOM 이벤트 등 JavaScript 엔진 자체가 아니라 브라우저나 Node.js가 제공하는 기능이다. 시간이 걸리는 작업을 처리해주는 "기계" 역할이다. Call Stack을 막지 않고 백그라운드에서 작업을 진행한다.

**3. Task Queue (콜백 큐)**

Web API가 작업을 끝내면, 그 결과를 처리할 콜백 함수를 큐에 넣는다. 이 큐는 두 종류로 나뉜다.

- **Macrotask Queue**: `setTimeout`, `setInterval`, I/O, UI 렌더링 등
- **Microtask Queue**: `Promise.then/catch/finally`, `queueMicrotask` 등

**4. Event Loop**

Call Stack이 비었는지 계속 확인하면서, 비었다면 Task Queue에서 작업을 꺼내 Call Stack으로 옮겨 실행시키는 역할을 한다. 바리스타가 "지금 손에 든 일이 끝났나? 끝났으면 다음 줄 손님(또는 완료된 기계 작업)을 처리하자"라고 확인하는 과정과 같다.

### 4. Event Loop의 실행 우선순위

Event Loop는 다음 순서로 작업을 처리한다:

1. Call Stack의 동기 코드를 모두 실행
2. Call Stack이 비면, **Microtask Queue**의 모든 작업을 순서대로 실행
3. Microtask Queue가 비면, **Macrotask Queue**에서 작업을 하나만 꺼내 실행
4. 다시 2번으로 돌아가 Microtask Queue 확인
5. 반복

**핵심: Microtask가 Macrotask보다 항상 먼저 처리된다.** Macrotask 하나를 실행한 뒤에는 다음 Macrotask로 넘어가기 전에 Microtask Queue를 반드시 비운다.

### 5. 기본 동작 원리

```javascript
// 바리스타가 일을 시작
console.log('Script Start'); // 1. 동기 코드

// 바리스타가 Web API에게 0초 뒤에 로그 출력해달라고 부탁하고 일단 돌아감
setTimeout(() => {
  console.log('setTimeout'); // 4. 매크로태스크 / 베이글(일반줄 대기)
}, 0);

// 바리스타가 VIP 주문을 받습니다. Promise로 받는 건 VIP 주문
Promise.resolve().then(() => {
  console.log('Promise 1'); // 3. 마이크로태스크 / VIP 주문 1
}).then(() => {
  console.log('Promise 2'); // 3. 마이크로태스크 / VIP 주문 2
});

// 바리스타는 마저 작업을 처리한다
console.log('Script End'); // 2. 동기 코드

// 출력 결과:
// 'Script Start'
// 'Script End'
// 'Promise 1'
// 'Promise 2'
// 'setTimeout'
```

단계별로 추적하면:

1. **Call Stack 시작**: `console.log('Script Start')` 실행 → `Script Start` 출력
2. **setTimeout 만남**: `setTimeout()` 호출 → Web API에게 타이머(0초)와 콜백 함수를 위임하고 Call Stack에서 즉시 제거됨. Web API는 0초 타이머를 시작
3. **Promise 만남**: `Promise.resolve().then()` 실행 → `then()`에 등록된 콜백 함수가 Microtask Queue로 이동 (`Microtask Queue: [Promise 1 콜백]`)
4. **동기 코드 완료**: `console.log('Script End')` 실행 → `Script End` 출력. 동기 코드 실행이 모두 끝나 Call Stack이 비워짐
5. **Event Loop 시작**: Call Stack이 비었는지 확인 ✅ → Microtask Queue를 먼저 확인
6. **Microtask 실행**: Microtask Queue에서 Promise 1 콜백을 꺼내 실행 → `Promise 1` 출력. 연결된 `then()`에 등록된 Promise 2 콜백이 Microtask Queue로 이동
7. **Microtask 계속**: Microtask Queue(`[Promise 2 콜백]`)에서 콜백을 꺼내 실행 → `Promise 2` 출력. Microtask Queue가 비어짐
8. **Macrotask 실행**: Microtask Queue가 비었으므로 Macrotask Queue 확인 → Web API에서 0초 타이머가 완료되어 setTimeout 콜백이 Macrotask Queue로 이동한 상태 → 꺼내서 실행 → `setTimeout` 출력

### 6. 실무에서 알아두면 좋은 것들

**1. setTimeout(fn, 0)의 진짜 의미**

`setTimeout`의 지연 시간을 0으로 설정해도 즉시 실행되지 않는다:

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');

// 출력:
// 1
// 3
// 2
```

0초여도 Web API를 거쳐 Macrotask Queue로 가기 때문에, 동기 코드가 모두 끝난 후에 실행된다.

사용 사례: 무거운 작업을 다음 이벤트 루프 사이클로 미루기, UI 업데이트 후 작업 실행하기.

**2. Promise vs setTimeout 우선순위**

Promise는 항상 setTimeout보다 먼저 실행된다:

```javascript
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('sync');

// 출력:
// sync
// promise
// timeout
```

이유는 Promise는 Microtask Queue로, setTimeout은 Macrotask Queue로 가기 때문이다.

**3. async/await와 Event Loop**

`async/await`는 Promise의 문법적 설탕이므로, 내부적으로는 Microtask Queue를 사용한다:

```javascript
console.log('1');

async function asyncFunc() {
  console.log('2');
  await Promise.resolve();
  console.log('3'); // Microtask로 처리됨
}

asyncFunc();
console.log('4');

// 출력:
// 1
// 2
// 4
// 3
```

`await` 이후의 코드는 `.then()` 안에 있는 것처럼 동작한다.

**4. 연속된 Promise 체이닝**

Promise 체이닝이 길어지면 각 `then`이 순차적으로 Microtask Queue에 추가된다:

```javascript
Promise.resolve()
  .then(() => console.log('1'))
  .then(() => console.log('2'))
  .then(() => console.log('3'));

Promise.resolve()
  .then(() => console.log('A'))
  .then(() => console.log('B'))
  .then(() => console.log('C'));

console.log('Start');

// 출력:
// Start
// 1
// A
// 2
// B
// 3
// C
```

첫 번째 Promise의 `then`과 두 번째 Promise의 `then`이 번갈아가며 실행된다.

### 7. 자주 하는 실수와 해결법

**실수 1: setTimeout 0초면 즉시 실행될 것이라고 착각**

```javascript
// ❌ 잘못된 기대
let value = 0;
setTimeout(() => {
  value = 100;
}, 0);
console.log(value); // 0 출력 (100이 아님!)

// ✅ 올바른 방법
let value = 0;
setTimeout(() => {
  value = 100;
  console.log(value); // 100 출력
}, 0);
```

`setTimeout`은 0초여도 비동기로 동작하므로, 동기 코드가 먼저 실행된다.

**실수 2: Microtask Queue 무한 루프**

```javascript
// ❌ 위험한 코드: 무한 루프
function infiniteMicrotask() {
  Promise.resolve().then(() => {
    console.log('Microtask');
    infiniteMicrotask(); // 자기 자신을 계속 호출
  });
}

infiniteMicrotask();

setTimeout(() => {
  console.log('Macrotask'); // 절대 실행되지 않음!
}, 0);
```

Microtask Queue가 비워지지 않으면 Macrotask는 영원히 실행되지 않는다.

```javascript
// ✅ 올바른 방법: 탈출 조건 추가
function limitedMicrotask(count) {
  if (count <= 0) return;
  Promise.resolve().then(() => {
    console.log('Microtask', count);
    limitedMicrotask(count - 1);
  });
}

limitedMicrotask(5);

setTimeout(() => {
  console.log('Macrotask'); // 이제 실행됨
}, 0);
```

**실수 3: Event Loop를 블로킹하는 무거운 작업**

```javascript
// ❌ 잘못된 방법: Call Stack을 오래 점유
function heavyTask() {
  const start = Date.now();
  while (Date.now() - start < 3000) {
    // 3초 동안 아무것도 못 함
  }
  console.log('Done');
}

heavyTask(); // 화면이 3초간 멈춤
```

```javascript
// ✅ 올바른 방법: 작업을 분할
function heavyTaskChunked(iterations, chunk = 1000) {
  let count = 0;

  function processChunk() {
    const end = Math.min(count + chunk, iterations);

    for (let i = count; i < end; i++) {
      // 작업 수행
    }

    count = end;

    if (count < iterations) {
      setTimeout(processChunk, 0); // 다음 사이클로 미루기
    } else {
      console.log('Done');
    }
  }

  processChunk();
}

heavyTaskChunked(1000000); // 화면이 멈추지 않음
```

### 8. 복잡한 예제로 완전히 이해하기

모든 개념을 종합한 예제:

```javascript
console.log('1');

setTimeout(() => {
  console.log('2');
  Promise.resolve().then(() => {
    console.log('3');
  });
}, 0);

Promise.resolve().then(() => {
  console.log('4');
  setTimeout(() => {
    console.log('5');
  }, 0);
});

console.log('6');

// 출력:
// 1
// 6
// 4
// 2
// 3
// 5
```

실행 순서 분석:

1. **동기 코드 실행**: `console.log('1')` → `1` 출력. `setTimeout` → Web API로 위임, Macrotask Queue 대기. `Promise.resolve().then` → Microtask Queue로 이동. `console.log('6')` → `6` 출력
2. **Microtask Queue 처리**: `Microtask Queue: [Promise '4' 콜백]` → 실행되어 `4` 출력. 콜백 내부의 `setTimeout`이 다시 Web API로 위임되어 Macrotask Queue에 대기
3. **첫 번째 Macrotask 처리**: `Macrotask Queue: [setTimeout '2' 콜백, setTimeout '5' 콜백]` → `2` 콜백 실행되어 `2` 출력. 콜백 내부의 `Promise.resolve().then`이 Microtask Queue로 이동
4. **다시 Microtask Queue 처리**: `Microtask Queue: [Promise '3' 콜백]` → 실행되어 `3` 출력
5. **두 번째 Macrotask 처리**: `Macrotask Queue: [setTimeout '5' 콜백]` → 실행되어 `5` 출력

### 9. 핵심 정리

| 개념 | 설명 |
|------|------|
| Call Stack | 코드가 실제로 실행되는 곳, LIFO 구조 |
| Web API | 브라우저/런타임이 제공하는 비동기 기능(setTimeout, fetch 등) |
| Macrotask Queue | setTimeout, setInterval, I/O 등의 콜백이 들어가는 큐 |
| Microtask Queue | Promise.then/catch/finally, queueMicrotask 콜백이 들어가는 큐 |
| Event Loop | Call Stack이 빌 때마다 Queue에서 작업을 꺼내 실행시키는 메커니즘 |
| 우선순위 | Microtask Queue를 모두 비운 뒤에만 다음 Macrotask로 넘어감 |

**다른 개념과의 연관성:**

| 연관 개념 | 관계 |
|---|---|
| Promise | `.then/.catch/.finally` 콜백이 Microtask Queue에서 처리됨 |
| async/await | 내부적으로 Promise 기반이며, `await` 이후 코드는 Microtask로 실행 |
| 블로킹/논블로킹, 동기/비동기 | Event Loop는 JavaScript가 싱글스레드이면서도 논블로킹/비동기를 구현하는 메커니즘 |
| setTimeout(fn, 0) | "즉시 실행"이 아니라 "다음 Macrotask로 미루기"를 의미 |

### 10. 마치며

- Event Loop는 JavaScript가 싱글스레드임에도 비동기 작업을 효율적으로 처리할 수 있게 해주는 핵심 메커니즘이다.
- Call Stack, Web API, Task Queue(Macrotask/Microtask)가 서로 협력하여 동작한다.
- Microtask는 항상 Macrotask보다 먼저 처리된다.
- `setTimeout(fn, 0)`은 즉시 실행을 의미하지 않는다.
- Event Loop를 이해하면 비동기 코드의 실행 순서를 예측하고, 성능 문제를 디버깅할 수 있다.

## 트레이드오프

해당 없음 — JavaScript 런타임의 동작 원리에 대한 개념 정리이며 분산 시스템 트레이드오프(일관성/가용성/지연/비용/운영부담)와는 무관. Microtask/Macrotask 우선순위에 따른 실행 순서·성능 영향은 위 특징/상세 4, 7번 항목 참고.

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 6번 "실무에서 알아두면 좋은 것들", 7번 "자주 하는 실수와 해결법"에 포함)

## 참고

원본 학습 노트(TIL)에 다음 `> 연결된 정리본` 링크들이 있었으나, 이 볼트 외부의 위키 경로라 마크다운 링크로 옮기지 않음. 필요 시 원본 TIL 위키에서 직접 확인.
- 이벤트 루프 (`../../../TIL.wiki/javascript-event-loop.md`)
- 비동기 기초 (`../../../TIL.wiki/javascript-async-fundamentals.md`)

- [MDN Web Docs - The event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
- [dev.to - JavaScript Event Loop Explained](https://dev.to/lydiahallie/javascript-visualized-event-loop-3dif)
- [YouTube - What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- [YouTube - In The Loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [HTML Living Standard - Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)

## 관련 내용

- [이벤트-루프와-비동기-처리](이벤트-루프와-비동기-처리.md)
- [Promise](Promise.md)
- [블로킹-논블로킹과-동기-비동기](블로킹-논블로킹과-동기-비동기.md)
