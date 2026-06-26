# SOLID

> 태그: `#oop` `#design-principle` `#solid`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`SOLID`는 변경에 강한 설계를 위한 5가지 원칙(SRP, OCP, LSP, ISP, DIP)이며, 핵심은 결합도를 낮추고 확장 가능성을 높이는 데 있다.

## 특징 / 상세

핵심 키워드: `SRP`, `OCP`, `LSP`, `ISP`, `DIP`

원칙은 정답 템플릿이 아니라, 설계가 깨질 때 "왜 깨졌는지"를 설명해주는 언어다.

### 1. SRP (Single Responsibility Principle)

> 변경 이유가 하나가 되도록 책임을 나눈다.

### 2. OCP (Open-Closed Principle)

> 확장에는 열려 있고, 변경에는 닫혀 있도록 설계한다.

### 3. LSP (Liskov Substitution Principle)

> 하위 타입은 상위 타입을 대체해도 프로그램 의미가 유지되어야 한다.

### 4. ISP (Interface Segregation Principle)

> 필요하지 않은 메서드에 의존하게 만들지 말고, 인터페이스를 쪼갠다.

### 5. DIP (Dependency Inversion Principle)

> 구체가 아니라 추상에 의존하고, 고수준 정책이 저수준 구현에 끌려가지 않게 한다.

### 6. 실전 포인트

> SOLID는 따로따로가 아니라 같이 묶여서 작동할 때 효과가 크다.

- SRP와 ISP는 클래스/인터페이스 크기를 줄인다.
- OCP와 DIP는 교체 가능성을 높인다.
- LSP는 상속을 쓸 때 설계가 무너지지 않게 막아준다.

## 트레이드오프

해당 없음

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 6장 "실전 포인트"에 포함)

## 참고

해당 없음 — 원본 TIL 노트에 출처가 명시되어 있지 않음.

## 관련 내용

- [Composition-over-Inheritance](Composition-over-Inheritance.md)
- [Dependency-Injection](Dependency-Injection.md)
