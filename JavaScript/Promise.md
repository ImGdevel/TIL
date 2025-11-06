# Promise: 콜백 지옥에서 벗어나기

## 들어가며

"이 데이터가 언제 올지 모르는데, 어떻게 처리하지?"

웹 개발을 하다 보면 항상 마주치는 고민입니다. 서버에서 데이터를 가져오거나, 파일을 읽거나, 타이머가 끝나기를 기다리는 등 시간이 걸리는 작업들이 있죠. 이런 **비동기 작업**을 어떻게 효율적으로 처리할 수 있을까요?

초창기 JavaScript에서는 **콜백 함수**로 이 문제를 해결했습니다. 하지만 콜백이 중첩되면서 악명 높은 **콜백 지옥(Callback Hell)**이 탄생했죠.

이 문제를 우아하게 해결하기 위해 등장한 것이 바로 **Promise**입니다. Promise는 단순히 콜백의 대안이 아니라, 비동기 프로그래밍의 패러다임을 바꾼 핵심 개념입니다.

## Promise가 왜 필요할까요?

### 콜백 함수의 문제점

초창기에는 비동기 작업을 처리하기 위해 **콜백 함수**를 사용했습니다. 함수를 인자로 넘겨서 작업이 완료되면 그 함수를 호출하는 방식이었죠.

```javascript
// 콜백 함수를 사용한 비동기 처리
getUserData(userId, function(error, user) {
  if (error) {
    console.error('사용자 조회 실패:', error);
    return;
  }

  getOrders(user.id, function(error, orders) {
    if (error) {
      console.error('주문 조회 실패:', error);
      return;
    }

    getOrderDetails(orders[0].id, function(error, details) {
      if (error) {
        console.error('주문 상세 조회 실패:', error);
        return;
      }

      console.log('주문 상세:', details);
      // 콜백 지옥 (Callback Hell) 발생!
    });
  });
});
```

이 코드의 문제점은:

1. **가독성 저하**: 중첩이 깊어질수록 코드가 오른쪽으로 계속 들여쓰기되어 읽기 어려워집니다
2. **에러 처리 중복**: 각 콜백마다 에러 처리를 반복해야 합니다
3. **유지보수 어려움**: 로직 변경이나 순서 조정이 매우 어렵습니다

### Promise의 등장

이런 문제를 해결하기 위해 등장한 것이 **Promise**입니다.

```javascript
// Promise를 사용한 비동기 처리
getUserData(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log('주문 상세:', details))
  .catch(error => console.error('에러 발생:', error));
```

같은 로직인데 훨씬 읽기 쉽고, 에러 처리도 한 곳에서 할 수 있습니다!

## Promise란 무엇인가?

Promise는 **비동기 작업의 성공 또는 실패 결과를 처리하기 위한 객체**입니다.

쉽게 말해, 시간이 걸리는 작업(예: API 호출해서 데이터 가져오기)을 시작했을 때, 그 작업이 당장 완료되지는 않지만 나중에 결과(성공 또는 실패)를 **꼭 알려주겠다고 약속하는 객체**입니다.

### 카페 진동벨 비유

카페에서 음료를 주문하면 점원이 진동벨을 줍니다. "음료 준비되면 벨이 울릴 거예요!" 이게 바로 Promise입니다.

- **주문을 받음** = Promise 객체 생성 (Pending)
- **음료가 완성됨** = 작업 성공 (Fulfilled)
- **재료가 떨어짐** = 작업 실패 (Rejected)

진동벨을 받은 손님은 자리로 돌아가서 다른 일을 할 수 있습니다. 음료가 준비될 때까지 카운터 앞에서 기다릴 필요가 없죠. 이것이 바로 **비동기 처리**의 핵심입니다.

## Promise의 세 가지 상태

Promise 객체는 항상 세 가지 상태 중 하나에 있습니다:

### 1. Pending (대기)

```javascript
const promise = new Promise((resolve, reject) => {
  // 이 시점에서 Promise는 Pending 상태
  console.log('작업 시작!');
});
```

- 약속을 한 직후의 상태입니다
- 아직 결과가 나오지 않고 기다리는 중입니다
- 비동기 작업이 진행 중인 상태입니다
- `new Promise`로 객체가 생성되는 시점부터 대기 상태로 들어갑니다
- 카페 비유: 주문을 넣고 진동벨을 받아서 음료가 만들어지기를 기다리는 상태

### 2. Fulfilled (이행)

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve({ name: 'Kim', age: 25 }); // Fulfilled 상태로 변경
  }, 1000);
});
```

- 약속이 성공적으로 지켜진 상태입니다
- 비동기 작업이 성공적으로 완료되어서 결과값을 반환한 상태입니다
- `resolve` 함수가 호출되면 이행 상태로 바뀌고, `resolve` 함수에 전달된 값이 Promise의 최종 결과값이 됩니다
- 서버에서 사용자 데이터를 성공적으로 가져온 경우입니다
- 카페 비유: 음료가 완성되어서 진동벨이 울리는 상태

### 3. Rejected (거부)

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject(new Error('네트워크 오류')); // Rejected 상태로 변경
  }, 1000);
});
```

- 약속이 지켜지지 못한 상태입니다
- 비동기 작업이 실패하고 오류가 발생한 상태입니다
- `reject` 함수가 호출되면 거부 상태로 바뀝니다
- 네트워크 오류로 데이터를 가져오지 못한 경우입니다
- 카페 비유: 재료가 떨어져서 음료를 만들지 못하는 상태

### 상태 전이의 중요한 규칙

**Fulfilled나 Rejected 상태가 되면 그 상태는 절대 변하지 않습니다.**

약속이 한번 지켜지거나 깨지면 그걸로 끝입니다. 이를 **불변성(Immutability)**이라고 하며, 비동기 코드의 안정성을 보장합니다.

```javascript
const promise = new Promise((resolve, reject) => {
  resolve('성공!');
  reject(new Error('실패!')); // 무시됨! 이미 resolve 됨
  resolve('다시 성공!'); // 무시됨! 이미 한 번 settle 됨
});

promise.then(value => console.log(value)); // '성공!' 출력
```

## Promise 생성과 사용

### Promise 생성하기

Promise는 `new Promise` 생성자를 통해 만듭니다:

```javascript
const myPromise = new Promise((resolve, reject) => {
  // 비동기 작업 수행
  // 성공하면 resolve() 호출
  // 실패하면 reject() 호출
});
```

생성자 함수는 두 개의 인자를 받습니다:
- **resolve**: 작업이 성공했을 때 호출하는 함수
- **reject**: 작업이 실패했을 때 호출하는 함수

### 실제 예제: 사용자 정보 가져오기

```javascript
const getUserNameById = (id) => {
  return new Promise((resolve, reject) => {
    console.log('서버에 사용자 정보 요청 중... (Pending)');

    // 1초 후에 결과 반환 (API 호출 시뮬레이션)
    setTimeout(() => {
      if (id === 1) {
        // 성공 케이스
        resolve({ name: 'Kim', email: 'kim@example.com' });
      } else {
        // 실패 케이스
        reject(new Error('사용자를 찾을 수 없습니다'));
      }
    }, 1000);
  });
};

// Promise 사용
getUserNameById(1)
  .then((user) => {
    console.log('성공:', user);
    // 출력: 성공: { name: 'Kim', email: 'kim@example.com' }
  })
  .catch((error) => {
    console.log('실패:', error.message);
  });

getUserNameById(999)
  .then((user) => {
    console.log('성공:', user);
  })
  .catch((error) => {
    console.log('실패:', error.message);
    // 출력: 실패: 사용자를 찾을 수 없습니다
  });
```

### resolve와 reject 함수

**resolve 함수:**
- 비동기 작업이 성공했을 때 호출하는 함수입니다
- 호출해야만 Fulfilled 상태로 변경됩니다
- 전달된 값이 `.then()`으로 넘어갑니다

**reject 함수:**
- 비동기 작업이 실패했을 때 호출하는 함수입니다
- 호출해야만 Rejected 상태로 변경됩니다
- 전달된 에러가 `.catch()`로 넘어갑니다

## Promise를 사용하는 이유

### 1. 명확한 에러 처리

콜백 방식에서는 함수마다 에러 처리를 따로 해야 했습니다:

```javascript
// 콜백 방식: 각 단계마다 에러 처리 필요
getUserData(userId, function(error, user) {
  if (error) {
    console.error(error);
    return;
  }

  getOrders(user.id, function(error, orders) {
    if (error) {
      console.error(error);
      return;
    }

    getOrderDetails(orders[0].id, function(error, details) {
      if (error) {
        console.error(error);
        return;
      }

      console.log(details);
    });
  });
});
```

하지만 Promise는 `then()`이 몇 개든 마지막 `.catch()`에서 한 번에 처리할 수 있어서 명확하고 깔끔합니다:

```javascript
// Promise 방식: 마지막에 한 번만 처리
getUserData(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(error => console.error(error)); // 모든 에러를 여기서 처리
```

### 2. 가독성 향상

콜백 지옥에서 벗어나 코드를 순차적으로 읽을 수 있습니다. 코드가 수직으로 플랫하게 작성되어 로직의 흐름을 쉽게 파악할 수 있습니다.

### 3. 체이닝 가능

Promise는 `.then()`을 계속 연결할 수 있어서 여러 비동기 작업을 순차적으로 처리하기 쉽습니다.

### 4. 조합 가능

`Promise.all`, `Promise.race` 등의 메서드를 통해 여러 Promise를 효율적으로 조합할 수 있습니다.

## Promise Chain (프로미스 체이닝)

Promise의 강력한 기능 중 하나는 **체이닝(Chaining)**입니다. `.then()`을 계속 연결해서 비동기 작업을 순차적으로 처리할 수 있습니다.

`then()`이 **무엇을 반환하느냐**에 따라서 다음 `then()`의 동작이 결정됩니다:

### 1. 일반 값을 반환하는 경우

다음 `then()`이 그 값을 즉시 인자로 받아서 처리합니다.

```javascript
Promise.resolve(1)
  .then(value => {
    console.log(value); // 1
    return value + 1; // 일반 값 반환
  })
  .then(value => {
    console.log(value); // 2 (즉시 받음)
    return value + 1;
  })
  .then(value => {
    console.log(value); // 3
  });

// 출력:
// 1
// 2
// 3
```

### 2. Promise를 반환하는 경우

다음 `then()`은 그 Promise가 완료될 때까지 기다렸다가, 그 Promise의 결과값으로 받습니다.

```javascript
Promise.resolve(1)
  .then(value => {
    console.log(value); // 1
    return new Promise(resolve => {
      setTimeout(() => {
        console.log('1초 대기 중...');
        resolve(value + 1);
      }, 1000);
    });
  })
  .then(value => {
    console.log(value); // 2 (1초 후에 받음)
    return value + 1;
  })
  .then(value => {
    console.log(value); // 3
  });

// 출력:
// 1
// 1초 대기 중...
// 2
// 3
```

**이 2번 규칙이 비동기 작업을 순차적으로 연결하는 핵심입니다!**

### 실제 사용 예제: 순차적 API 호출

```javascript
// 사용자 정보 조회 → 주문 목록 조회 → 첫 주문 상세 조회
getUserById(1)
  .then(user => {
    console.log('사용자 조회 완료:', user.name);
    return getOrdersByUser(user.id); // Promise 반환
  })
  .then(orders => {
    console.log('주문 목록 조회 완료:', orders.length, '개');
    return getOrderDetails(orders[0].id); // Promise 반환
  })
  .then(details => {
    console.log('주문 상세 조회 완료:', details);
  })
  .catch(error => {
    console.error('어디선가 에러 발생:', error);
  });
```

### 올바른 체이닝 방식 vs 잘못된 방식

Promise 체인은 **수직으로 플랫하게** 작성해야 합니다:

```javascript
// ✅ 올바른 방식: 플랫한 체인
getUserById(1)
  .then(user => getOrdersByUser(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(error => console.error(error));

// ❌ 잘못된 방식: 중첩된 체인 (콜백 지옥과 비슷)
getUserById(1)
  .then(user => {
    getOrdersByUser(user.id)
      .then(orders => {
        getOrderDetails(orders[0].id)
          .then(details => console.log(details));
      });
  });
```

잘못된 방식은 Promise를 사용하는 의미가 없어집니다. 콜백 지옥과 같은 구조가 되어버리죠!

### finally(): 성공/실패 관계없이 실행

```javascript
fetch('/api/data')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error(error))
  .finally(() => {
    console.log('로딩 스피너 숨기기');
    // 성공하든 실패하든 항상 실행됨
  });
```

## 여러 개의 Promise 다루기

실무에서는 여러 비동기 작업을 동시에 처리해야 하는 경우가 많습니다. Promise는 이런 경우를 위한 유틸리티 메서드를 제공합니다.

각 메서드는 **성공/실패 시점의 판단이 다릅니다**.

### Promise.all - 모두 성공해야 할 때

```javascript
const promise1 = fetch('/api/popular-products');
const promise2 = fetch('/api/recommended-products');
const promise3 = fetch('/api/new-products');

Promise.all([promise1, promise2, promise3])
  .then(results => {
    console.log('모든 데이터 로드 완료:', results);
    // 페이지 렌더링
  })
  .catch(error => {
    console.error('하나라도 실패하면 여기로:', error);
  });
```

| 항목 | 설명 |
|------|------|
| **언제 성공?** | 모든 Promise가 Fulfilled 상태가 될 때 |
| **언제 실패?** | 하나라도 Rejected 되면 즉시 실패 |
| **반환값** | 모든 Promise의 결과값을 담은 배열 `[결과1, 결과2, 결과3]` |
| **언제 사용?** | 모든 API 요청이 반드시 성공해야 하는 경우 (필수 데이터) |

**실무 예시:**
쇼핑몰 메인 페이지에서 인기상품, 추천상품, 신상품 API를 모두 호출해야 하고, 각 요청은 독립적이지만 **모두 성공했을 때 UI를 한번에 렌더링**하고 싶을 때 사용합니다.

```javascript
Promise.all([
  fetchPopularProducts(),
  fetchRecommendedProducts(),
  fetchNewProducts()
])
  .then(([popular, recommended, newProducts]) => {
    // 모든 데이터가 준비됨
    renderMainPage({ popular, recommended, newProducts });
  })
  .catch(error => {
    // 하나라도 실패하면 에러 페이지 표시
    showErrorPage();
  });
```

### Promise.allSettled - 부분 실패를 허용할 때

```javascript
const promise1 = fetch('/api/popular-products');
const promise2 = fetch('/api/recommended-products'); // 실패 가능
const promise3 = fetch('/api/new-products');

Promise.allSettled([promise1, promise2, promise3])
  .then(results => {
    results.forEach(result => {
      if (result.status === 'fulfilled') {
        console.log('성공:', result.value);
        // 데이터 렌더링
      } else {
        console.log('실패:', result.reason);
        // 폴백 UI 표시
      }
    });
  });
```

| 항목 | 설명 |
|------|------|
| **언제 성공?** | 모든 Promise가 Settled 상태(Fulfilled 또는 Rejected)가 될 때 |
| **언제 실패?** | 없음 (항상 성공) |
| **반환값** | 각 Promise의 상태와 결과/이유를 담은 객체 배열 |
| **언제 사용?** | 여러 API 요청 중 일부가 실패해도 괜찮을 때 |

**반환값 구조:**
```javascript
[
  { status: 'fulfilled', value: 결과값 },
  { status: 'rejected', reason: 에러 }
]
```

**실무 예시:**
신상품 API가 일시적으로 오류가 나도 인기상품, 추천상품은 보여주고 싶을 때. 결과 배열을 순회하면서 `status`가 'fulfilled'인 데이터는 렌더링하고, 'rejected'인 데이터는 폴백 UI를 보여줄 수 있습니다.

```javascript
Promise.allSettled([
  fetchPopularProducts(),
  fetchRecommendedProducts(),
  fetchNewProducts()
])
  .then(results => {
    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        renderSection(index, result.value);
      } else {
        renderErrorSection(index, '데이터를 불러올 수 없습니다');
      }
    });
  });
```

### Promise.any - 하나라도 성공하면 될 때

```javascript
const server1 = fetch('https://mirror1.example.com/file.zip');
const server2 = fetch('https://mirror2.example.com/file.zip');
const server3 = fetch('https://mirror3.example.com/file.zip');

Promise.any([server1, server2, server3])
  .then(result => {
    console.log('가장 빠른 서버의 응답:', result);
  })
  .catch(error => {
    console.error('모든 서버가 실패했습니다:', error);
  });
```

| 항목 | 설명 |
|------|------|
| **언제 성공?** | 여러 Promise 중 하나라도 Fulfilled 상태로 넘어가면 성공 |
| **언제 실패?** | 모든 Promise가 Rejected 되면 실패 |
| **반환값** | 가장 먼저 Fulfilled 된 Promise의 결과값 |
| **언제 사용?** | 여러 대안 중 하나만 성공해도 되는 경우 |

**실무 예시:**
- 여러 미러 서버 중 가장 빠른 서버에서 파일 다운로드
- 여러 API 엔드포인트 중 하나라도 응답하면 되는 경우

### Promise.race - 가장 빠른 응답을 원할 때

```javascript
const dataPromise = fetch('/api/data');
const timeoutPromise = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('시간 초과')), 5000)
);

Promise.race([dataPromise, timeoutPromise])
  .then(data => console.log('데이터 받음:', data))
  .catch(error => console.error('실패:', error.message));
```

| 항목 | 설명 |
|------|------|
| **언제 완료?** | 가장 먼저 완료(성공 또는 실패)되는 Promise의 결과를 따름 |
| **반환값** | 가장 먼저 완료된 Promise의 결과 또는 에러 |
| **언제 사용?** | 타임아웃 구현, 가장 빠른 응답을 받고 싶을 때 |

**실무 예시:**
- API 호출에 타임아웃 설정
- 여러 경로 중 가장 빠른 응답 사용

### Promise 메서드 비교 표

| 메서드 | 성공 조건 | 실패 조건 | 반환값 | 주요 사용처 |
|--------|-----------|-----------|--------|-------------|
| `Promise.all` | 모두 성공 | 하나라도 실패 | 모든 결과의 배열 | 필수 데이터 모두 로드 |
| `Promise.allSettled` | 모두 완료 | 없음 | 상태+결과 객체 배열 | 부분 실패 허용 |
| `Promise.any` | 하나라도 성공 | 모두 실패 | 첫 성공 결과 | 미러 서버, 대안 |
| `Promise.race` | 첫 완료 | 첫 실패 | 첫 완료 결과 | 타임아웃, 속도 경쟁 |

## Promise와 Event Loop

Promise는 **Microtask Queue**를 사용합니다. Event Loop 관점에서 Promise의 `then()`, `catch()`, `finally()` 콜백은 모두 Microtask Queue에 들어갑니다.

```javascript
console.log('1'); // 동기 코드

setTimeout(() => {
  console.log('2'); // Macrotask Queue
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3'); // Microtask Queue
  });

console.log('4'); // 동기 코드

// 출력:
// 1
// 4
// 3 (Microtask가 먼저!)
// 2
```

**이것이 Promise가 `setTimeout`보다 항상 먼저 실행되는 이유입니다!**

Event Loop는 다음 순서로 작업을 처리합니다:
1. 동기 코드 (Call Stack)
2. Microtask Queue의 모든 작업 (Promise)
3. Macrotask Queue에서 하나 (setTimeout)
4. 2번으로 돌아가기

자세한 내용은 [Event Loop와 비동기 처리](./Event-Loop와-비동기-처리.md) 문서를 참고하세요.

## 실무 패턴과 Best Practices

### 1. Promise를 반환하는 함수 만들기

```javascript
// ✅ 좋은 예: Promise를 반환하는 함수
function fetchUserData(userId) {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json());
}

// 사용
fetchUserData(1)
  .then(user => console.log(user))
  .catch(error => console.error(error));
```

### 2. 에러 처리를 항상 포함하기

```javascript
// ❌ 나쁜 예: catch 없음
getUserData(1)
  .then(user => console.log(user));

// ✅ 좋은 예: catch 포함
getUserData(1)
  .then(user => console.log(user))
  .catch(error => console.error(error));
```

### 3. Promise 체인에서 값 전달하기

```javascript
getUserById(1)
  .then(user => {
    console.log('사용자:', user);
    return user; // 다음 then으로 전달!
  })
  .then(user => {
    // user를 받을 수 있음
    return getOrdersByUser(user.id);
  })
  .then(orders => {
    console.log('주문:', orders);
  });
```

### 4. 병렬 처리가 가능하면 Promise.all 사용

```javascript
// ❌ 나쁜 예: 순차적으로 실행 (느림)
const user = await getUserById(1);
const products = await getProducts();
const categories = await getCategories();
// 총 시간: 각 요청 시간의 합

// ✅ 좋은 예: 병렬로 실행 (빠름)
const [user, products, categories] = await Promise.all([
  getUserById(1),
  getProducts(),
  getCategories()
]);
// 총 시간: 가장 느린 요청의 시간
```

## 자주 하는 실수와 해결법

### 실수 1: Promise를 반환하지 않기

```javascript
// ❌ 잘못된 코드
function getUserData(id) {
  fetch(`/api/users/${id}`)
    .then(response => response.json());
  // return이 없음!
}

getUserData(1)
  .then(user => console.log(user)); // undefined!

// ✅ 올바른 코드
function getUserData(id) {
  return fetch(`/api/users/${id}`)
    .then(response => response.json());
}
```

### 실수 2: then 안에서 Promise를 중첩하기

```javascript
// ❌ 잘못된 코드 (콜백 지옥과 비슷)
getUserById(1)
  .then(user => {
    getOrdersByUser(user.id)
      .then(orders => {
        console.log(orders);
      });
  });

// ✅ 올바른 코드 (플랫하게)
getUserById(1)
  .then(user => getOrdersByUser(user.id))
  .then(orders => console.log(orders));
```

### 실수 3: catch를 빠뜨리기

```javascript
// ❌ 잘못된 코드
Promise.resolve()
  .then(() => {
    throw new Error('에러 발생!');
  })
  .then(() => {
    console.log('이 코드는 실행되지 않음');
  });
// Unhandled Promise Rejection!

// ✅ 올바른 코드
Promise.resolve()
  .then(() => {
    throw new Error('에러 발생!');
  })
  .then(() => {
    console.log('이 코드는 실행되지 않음');
  })
  .catch(error => {
    console.error('에러 처리:', error.message);
  });
```

### 실수 4: Promise 생성자에서 비동기 작업을 제대로 처리하지 않기

```javascript
// ❌ 잘못된 코드
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = Math.random() > 0.5;
    // resolve나 reject를 호출하지 않음!
  }, 1000);
});

// ✅ 올바른 코드
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = Math.random() > 0.5;
    if (success) {
      resolve('성공!');
    } else {
      reject(new Error('실패!'));
    }
  }, 1000);
});
```

## 핵심 정리

Promise를 이해하기 위한 핵심을 정리하면:

| 개념 | 설명 |
|------|------|
| **Promise란?** | 비동기 작업의 성공/실패 결과를 처리하기 위한 객체 |
| **세 가지 상태** | Pending (대기), Fulfilled (이행), Rejected (거부) |
| **상태 불변성** | 한번 settled(이행 또는 거부)되면 상태가 변하지 않음 |
| **resolve 함수** | 작업 성공 시 호출, Fulfilled 상태로 변경 |
| **reject 함수** | 작업 실패 시 호출, Rejected 상태로 변경 |
| **then()** | 성공 시 실행할 콜백 등록 |
| **catch()** | 실패 시 실행할 콜백 등록 |
| **체이닝** | then()을 연결하여 순차적 비동기 처리 |
| **Microtask Queue** | Promise 콜백은 Microtask Queue에서 처리됨 |

**Promise를 사용하는 이유:**
1. 콜백 지옥에서 벗어남
2. 명확한 에러 처리 (.catch)
3. 가독성 향상 (플랫한 체인)
4. 여러 Promise 조합 가능 (Promise.all 등)

**주의할 점:**
- Promise를 반환하는 함수는 반드시 `return` 사용
- `.catch()`로 에러 처리를 빠뜨리지 말기
- Promise 체인은 중첩하지 말고 플랫하게 작성
- 병렬 처리 가능한 경우 `Promise.all` 사용

## 다음 단계: async/await

Promise는 콜백보다 훨씬 낫지만, 여전히 `.then()` 체인이 길어지면 복잡할 수 있습니다. 이를 더욱 간결하게 만든 것이 **async/await** 문법입니다.

```javascript
// Promise 체인
getUserById(1)
  .then(user => getOrdersByUser(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(error => console.error(error));

// async/await로 동일한 로직
async function getOrderInfo() {
  try {
    const user = await getUserById(1);
    const orders = await getOrdersByUser(user.id);
    const details = await getOrderDetails(orders[0].id);
    console.log(details);
  } catch (error) {
    console.error(error);
  }
}
```

async/await는 Promise를 **더 동기 코드처럼 읽히게** 만든 문법적 설탕(Syntactic Sugar)입니다. 내부적으로는 여전히 Promise를 사용하므로, Promise를 먼저 이해하는 것이 중요합니다!

## 참고 자료

- [MDN Web Docs - Promise](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [JavaScript.info - Promises, async/await](https://javascript.info/async)
- [MDN Web Docs - Using Promises](https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Using_promises)
- [You Don't Know JS: Async & Performance](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/async%20%26%20performance/README.md)

---

*작성일: 2025-11-06*
