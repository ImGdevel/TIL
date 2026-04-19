# [객체지향] Association / Aggregation / Composition

## 한 줄 요약

> <u>`Association`은 “알고 있다/참조한다” 수준의 관계이고, `Aggregation`은 약한 포함(has-a), `Composition`은 생명주기가 강하게 묶인 강한 포함(whole-part)이다.</u>

현업 코드에서는 UML 용어를 엄격히 구분하기보다, “소유/생명주기”를 명확히 하는 게 더 중요하다.

핵심 키워드: `has-a`, `ownership`, `lifecycle`, `whole-part`, `UML`

<br /><br />

---

## 1. Association(연관)

> <u>한 객체가 다른 객체를 참조/사용한다.</u>

- 생명주기 소유를 의미하지는 않는다.

<br /><br />

---

## 2. Aggregation(집합)

> <u>전체-부분 관계지만, 부분의 생명주기는 독립적일 수 있다.</u>

- “차가 바퀴를 가진다”처럼 생각하지만,
  바퀴가 차와 생명주기를 완전히 공유한다고 단정하긴 어렵다.

<br /><br />

---

## 3. Composition(합성)

> <u>부분이 전체에 강하게 소유되고, 생명주기가 함께 간다.</u>

- 전체가 파괴되면 부분도 함께 사라지는 관계로 본다.

