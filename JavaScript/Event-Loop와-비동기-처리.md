# Event Loop로 시작하는 JavaScript 비동기 처리

## 들어가며

JavaScript로 웹사이트를 사용할 때를 떠올려볼까요? 배경음악이 나오면서 화면에는 애니메이션이 움직이고, 동시에 스크롤을 내릴 수도 있고 버튼을 클릭할 수도 있습니다. 심지어 서버에서 데이터를 가져오는 동안에도 화면이 멈추지 않죠.

그런데 JavaScript는 **싱글스레드(Single Thread)** 언어라고 합니다. 싱글스레드는 한 번에 하나의 일만 처리할 수 있다는 의미인데, 어떻게 여러 일을 동시에 처리하는 것처럼 보일까요?

그 비밀은 바로 **이벤트 루프(Event Loop)**와 **런타임 환경**에 있습니다. 오늘은 JavaScript가 어떻게 비동기 작업을 처리하는지, 그 내부 동작 원리를 바리스타에 비유하여 쉽게 알아보겠습니다.

## Event Loop가 뭔가요?

Event Loop는 간단히 말해서 **카페 매니저** 역할을 합니다.

JavaScript를 혼자서 일하는 바리스타에 비유해볼까요? 바리스타는 싱글스레드여서 한 번에 하나의 일만 처리할 수 있습니다. 손님에게 주문을 받거나, 커피를 내리거나, 컵을 닦거나... 동시에 두 가지 일을 못해서 하나씩만 합니다. 여기서 "스레드"란 일을 처리하는 경로(실)를 의미하고, 그게 딱 하나뿐입니다.

하지만 이 바리스타에게는 **카페라는 환경**이 있습니다. 카페에는 전자레인지, 에스프레소 머신 같은 주방 기구와 도구, 그리고 매니저가 있습니다. 바리스타가 모든 걸 직접 하지 않고 **환경의 도움**을 받아 효율적으로 일할 수 있는 거죠.

이것이 바로 **런타임 환경(웹 브라우저, Node.js)**입니다. 런타임 환경이 JavaScript 엔진에게 비동기 처리 능력을 선물해주는 겁니다.

## 왜 Event Loop를 알아야 할까요?

**JavaScript 동작 방식 이해**
프론트엔드, 백엔드 할 것 없이 JavaScript 개발자라면 반드시 이해해야 하는 핵심 개념입니다. 근본적인 개발자 역량을 키우고 견해를 넓히는 중요한 내용입니다.

**성능 최적화**
어떤 작업이 언제 실행되는지 정확히 알아야 성능 문제를 진단하고 최적화할 수 있습니다.

**버그 디버깅**
"왜 이 코드가 먼저 실행되지 않을까?" 같은 비동기 관련 버그를 해결할 수 있습니다.

**면접 대비**
JavaScript 개발자 면접에서 가장 자주 나오는 질문 중 하나가 "Event Loop가 무엇인가요?"입니다.

## JavaScript 비동기 처리의 핵심 구성 요소

### 1. Call Stack (콜 스택)

JavaScript가 해야 할 일에 대한 **목록 메모장**입니다.

손님이 아이스 아메리카노를 주문했다면, 바리스타는 메모장에 작업 순서를 적습니다:
- ~~원두 갈기~~
- ~~에스프레소 추출~~
- ~~물 붓기~~

JavaScript가 코드를 실행하는 방식도 이와 같습니다:
- 함수가 호출되면 Call Stack에 쌓입니다(push)
- 함수 실행이 끝나면 스택에서 빠져나갑니다(pop)

**특징:**
- LIFO(Last In First Out) 구조
- 한 번에 하나의 작업만 처리 (싱글스레드)
- 스택이 비어있어야 다음 작업 가능

### 2. Web API

바리스타에게 베이글을 데워달라고 요청이 들어왔습니다. 바리스타는 베이글을 전자레인지에 넣고 3분 타이머를 맞춥니다. 버튼을 누르고 자기 자리로 돌아와서 다른 일을 계속합니다. 전자레인지는 알아서 베이글을 데우고 있죠.

여기서 **전자레인지가 바로 Web API**입니다.

Web API는 JavaScript 언어 자체의 기능이 아니라, **JavaScript가 실행되는 환경(브라우저)이 제공하는 기능들**입니다:
- `setTimeout`, `setInterval` - 타이머 작업
- `fetch` - 네트워크 요청
- DOM 이벤트 - 클릭, 마우스오버 등
- `XMLHttpRequest` - HTTP 요청

JavaScript는 이런 시간이 오래 걸리는 작업들을 Web API에게 **위임**합니다. 직접 처리하지 않고 맡기는 거죠.

### 3. Task Queue (태스크 큐)

3분이 지나서 전자레인지가 다 돌아갔습니다. 베이글이 완성되었죠. 이 베이글(완료된 작업)은 어디로 갈까요?

**Task Queue**라고 하는 대기줄에 세워놓습니다. 결과물(실행해야 할 콜백 함수)을 줄 세워서 기다리게 하는 거죠.

Task Queue는 두 가지 종류가 있습니다:

**Macrotask Queue (매크로태스크 큐) - 일반 대기줄**
- `setTimeout`, `setInterval`의 콜백 함수
- 사용자 클릭, 마우스오버의 콜백 함수
- 네트워크 요청(`fetch`의 일부) 콜백 함수
- 이 큐에 들어가는 작업을 **매크로태스크**라고 부릅니다

**Microtask Queue (마이크로태스크 큐) - VIP 대기줄**
- `Promise`의 `then()`, `catch()`, `finally()`에 등록된 콜백 함수
- `async/await`의 후속 처리
- `queueMicrotask()`로 등록된 작업
- 좀 더 빠르게 처리해야 하는 작업들
- 이 큐에 들어가는 작업을 **마이크로태스크**라고 부릅니다

### 4. Event Loop (이벤트 루프)

**카페 매니저** 같은 존재입니다.

Event Loop는 끊임없이 두 가지를 확인합니다:
1. **Call Stack이 비었는지 확인** - 비었다면 바리스타가 일을 다 마쳤다는 뜻
2. **Task Queue에 기다리는 작업이 있는지 확인** - 있다면 가장 오래 기다린 작업을 꺼내서 Call Stack에 올려줍니다

이것이 바로 Event Loop의 역할입니다. 계속 반복(Loop)해서 확인하고 작업을 전달하는 거죠.

## Event Loop의 실행 우선순위

여기서 중요한 규칙이 있습니다:

**하나의 매크로태스크를 실행한 후에는, 마이크로태스크 큐에 있는 모든 마이크로태스크를 전부 실행하고 나서야 다음 매크로태스크를 실행합니다.**

마이크로태스크 큐는 **VIP 대기줄**, 태스크 큐는 **일반 대기줄**입니다.

일반 대기줄에서 손님 하나를 처리하고 왔는데, 아직 일반 대기줄에는 손님들이 많이 남아있습니다. 그런데 그 와중에 VIP 대기줄에 손님이 확인되면 어떻게 할까요? 일반 손님이 처리될 때까지 VIP 손님을 기다리게 하지 않습니다. **VIP를 먼저 전부 처리하고**, 그 다음에 일반 대기줄을 처리합니다.

실행 순서를 정리하면:
1. 동기 코드 실행 (Call Stack)
2. 마이크로태스크 큐의 모든 작업 실행
3. 매크로태스크 큐에서 하나 실행
4. 2번으로 돌아가서 반복

## 기본 동작 원리

### 코드 실행 예시

다음 코드를 통해 실제로 어떻게 동작하는지 살펴보겠습니다:

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

### 단계별 실행 흐름

**1단계: Call Stack 시작**
```
Call Stack: console.log('Script Start')
출력: 'Script Start'
```

**2단계: setTimeout 만남**
```
Call Stack: setTimeout() 호출
→ Web API에게 타이머(0초)와 콜백함수를 위임
→ setTimeout은 바로 종료 (Call Stack에서 제거)
Web API: 0초 타이머 시작
```

**3단계: Promise 만남**
```
Call Stack: Promise.resolve().then() 실행
→ then()에 등록된 콜백함수는 Microtask Queue로 이동
Microtask Queue: [Promise 1 콜백]
```

**4단계: 동기 코드 완료**
```
Call Stack: console.log('Script End')
출력: 'Script End'
→ 동기적인 코드 실행 완료
Call Stack: 비어있음
```

**5단계: Event Loop 시작**
```
Event Loop: Call Stack이 비었는지 확인 ✅
Event Loop: Microtask Queue 먼저 확인
```

**6단계: Microtask 실행**
```
Microtask Queue에서 Promise 1 콜백 꺼냄
Call Stack: console.log('Promise 1')
출력: 'Promise 1'
→ 연결된 then()에 등록된 Promise 2 콜백이 Microtask Queue로 이동
```

**7단계: Microtask 계속**
```
Microtask Queue: [Promise 2 콜백]
Call Stack: console.log('Promise 2')
출력: 'Promise 2'
Microtask Queue: 비어있음
```

**8단계: Macrotask 실행**
```
Event Loop: Microtask Queue가 비었으므로 Macrotask Queue 확인
Web API에서 0초 타이머 완료 → setTimeout 콜백을 Macrotask Queue로 이동
Macrotask Queue에서 setTimeout 콜백 꺼냄
Call Stack: console.log('setTimeout')
출력: 'setTimeout'
```

## 실무에서 알아두면 좋은 것들

### 1. setTimeout(fn, 0)의 진짜 의미

`setTimeout`의 지연 시간을 0으로 설정해도 즉시 실행되지 않습니다:

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');

// 출력:
// 1
// 3
// 2
```

0초여도 Web API를 거쳐 Macrotask Queue로 가기 때문에, 동기 코드가 모두 끝난 후에 실행됩니다.

**사용 사례:**
- 무거운 작업을 다음 이벤트 루프 사이클로 미루기
- UI 업데이트 후 작업 실행하기

### 2. Promise vs setTimeout 우선순위

Promise는 항상 setTimeout보다 먼저 실행됩니다:

```javascript
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('sync');

// 출력:
// sync
// promise
// timeout
```

이유는 Promise는 Microtask Queue로, setTimeout은 Macrotask Queue로 가기 때문입니다.

### 3. async/await와 Event Loop

`async/await`는 Promise의 문법적 설탕이므로, 내부적으로는 Microtask Queue를 사용합니다:

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

`await` 이후의 코드는 `.then()` 안에 있는 것처럼 동작합니다.

### 4. 연속된 Promise 체이닝

Promise 체이닝이 길어지면 각 `then`이 순차적으로 Microtask Queue에 추가됩니다:

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

첫 번째 Promise의 `then`과 두 번째 Promise의 `then`이 번갈아가며 실행됩니다.

## 자주 하는 실수와 해결법

### 실수 1: setTimeout 0초면 즉시 실행될 것이라고 착각

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

`setTimeout`은 0초여도 비동기로 동작하므로, 동기 코드가 먼저 실행됩니다.

### 실수 2: Microtask Queue 무한 루프

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

Microtask Queue가 비워지지 않으면 Macrotask는 영원히 실행되지 않습니다.

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

### 실수 3: Event Loop를 블로킹하는 무거운 작업

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

## 복잡한 예제로 완전히 이해하기

모든 개념을 종합한 예제를 살펴보겠습니다:

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

### 실행 순서 분석:

**1단계: 동기 코드 실행**
```
Call Stack: console.log('1') → 출력: '1'
Call Stack: setTimeout (Web API로 위임, Macrotask Queue 대기)
Call Stack: Promise.resolve().then (Microtask Queue로 이동)
Call Stack: console.log('6') → 출력: '6'
```

**2단계: Microtask Queue 처리**
```
Microtask Queue: [Promise '4' 콜백]
Call Stack: console.log('4') → 출력: '4'
Call Stack: setTimeout (Web API로 위임, Macrotask Queue 대기)
```

**3단계: 첫 번째 Macrotask 처리**
```
Macrotask Queue: [setTimeout '2' 콜백, setTimeout '5' 콜백]
Call Stack: console.log('2') → 출력: '2'
Call Stack: Promise.resolve().then (Microtask Queue로 이동)
```

**4단계: 다시 Microtask Queue 처리**
```
Microtask Queue: [Promise '3' 콜백]
Call Stack: console.log('3') → 출력: '3'
```

**5단계: 두 번째 Macrotask 처리**
```
Macrotask Queue: [setTimeout '5' 콜백]
Call Stack: console.log('5') → 출력: '5'
```

## 핵심 정리

Event Loop를 이해하기 위한 핵심을 정리하면:

| 개념 | 설명 |
|------|------|
| Call Stack | JavaScript가 실행하는 작업 목록 (LIFO) |
| Web API | 브라우저가 제공하는 비동기 기능 (타이머, fetch 등) |
| Macrotask Queue | setTimeout, 이벤트 콜백 등의 대기줄 (일반) |
| Microtask Queue | Promise, async/await의 대기줄 (VIP) |
| Event Loop | 큐에서 작업을 꺼내 Call Stack으로 전달하는 관리자 |

**실행 순서:**
1. 동기 코드 (Call Stack)
2. Microtask Queue의 모든 작업
3. Macrotask Queue에서 하나
4. 2번으로 돌아가기

## 다른 개념과의 연관성

이전에 학습한 **블로킹/논블로킹**과 **동기/비동기** 개념과 연결해보면:

| 개념 | 관점 | Event Loop와의 관계 |
|------|------|-------------------|
| 블로킹 vs 논블로킹 | 제어권의 이동 | Call Stack을 점유하는가? |
| 동기 vs 비동기 | 작업 순서와 결과 처리 | Queue를 거치는가? |
| Event Loop | 작업 스케줄링 | 어떤 순서로 실행할 것인가? |

JavaScript는 **비동기/논블로킹** 모델을 통해 효율적으로 처리하고, 그것을 가능하게 만드는 것이 **Event Loop**의 역할입니다.

## 마치며

Event Loop는 JavaScript의 심장과도 같습니다. 싱글스레드 언어인 JavaScript가 마치 멀티태스킹을 하는 것처럼 보이게 만드는 마법의 비밀이죠.

실무에서 가장 중요한 건:
- **JavaScript 엔진 자체는 싱글스레드지만, 런타임 환경이 비동기 처리 능력을 제공한다**
- **Event Loop는 항상 Microtask Queue를 Macrotask Queue보다 먼저 처리한다**
- **setTimeout(fn, 0)도 비동기이므로 동기 코드 이후에 실행된다**
- **무거운 작업은 Call Stack을 오래 점유하지 않도록 분할해야 한다**
- **Microtask Queue를 무한으로 채우면 Macrotask는 영원히 실행되지 않는다**

바리스타 비유를 떠올리며 코드를 읽어보세요. 어떤 작업이 Call Stack에 있는지, 어떤 작업이 Queue에서 기다리는지, Event Loop가 어떻게 움직이는지 상상하다 보면 JavaScript의 비동기 동작이 자연스럽게 이해될 겁니다.

이제 "왜 이 코드가 나중에 실행되지?"라는 질문에 명확하게 답할 수 있을 거예요!

## 참고 자료

- [MDN Web Docs - Event Loop](https://developer.mozilla.org/ko/docs/Web/JavaScript/EventLoop)
- [JavaScript Visualized: Event Loop](https://dev.to/lydiahallie/javascript-visualized-event-loop-3dif)
- [Jake Archibald: In The Loop - JSConf.Asia](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [Philip Roberts: What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- [HTML Standard - Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)

---

*작성일: 2025-11-05*
