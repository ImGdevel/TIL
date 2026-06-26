# 2026-06-23 세션: JVM 런타임 데이터 영역

## 질문

JVM이 프로그램을 실행할 때 메모리를 몇 개의 영역으로 나누는지, 어떤 영역이 스레드별로 존재하고 어떤 영역이 공유되는지, StackOverflowError/OutOfMemoryError는 어디서 발생하는지.

## 결론

JVM 런타임 데이터 영역은 5개: Heap, Method Area, JVM Stack, PC Register, Native Method Stack.

- 전체 스레드 공유: Heap, Method Area
- 스레드별로 독립 생성: JVM Stack, PC Register, Native Method Stack

StackOverflowError는 JVM Stack과 Native Method Stack 양쪽에서 발생 가능 (스택 깊이 초과). OutOfMemoryError는 Heap, Method Area(Run-Time Constant Pool 포함), JVM Stack(확장 실패 시), Native Method Stack(확장 실패 시) 네 군데 전부에서 발생 가능 — "OOM = Heap"은 가장 흔한 케이스일 뿐 전체는 아니다.

추가로 확인한 것: JVMS는 "Method Area"라는 논리적 개념만 정의하고 구현 방식은 벤더 자유. HotSpot은 Java 8(JEP 122)부터 이를 네이티브 메모리 영역인 **Metaspace**로 구현해 PermGen을 대체했다. 이때 클래스 메타데이터는 네이티브 메모리(Metaspace)로, interned String과 클래스 static 변수는 **Heap**으로 이동했다 — 즉 "static 변수도 메타스페이스에 있다"는 통념은 부정확하다(값 자체는 Heap 객체, Metaspace에는 클래스 구조/메서드 코드/런타임 상수 풀 등이 위치).

## 근거 / 출처

- [JVM Specification SE21 §2.5 Run-Time Data Areas](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.5) — pc Register(2.5.1), JVM Stacks(2.5.2, StackOverflowError/OutOfMemoryError 명시), Heap(2.5.3, OutOfMemoryError 명시), Method Area(2.5.4, OutOfMemoryError 명시), Run-Time Constant Pool(2.5.5, Method Area에서 클래스 단위로 할당), Native Method Stacks(2.5.6, StackOverflowError/OutOfMemoryError 명시) 원문 확인.
- [JEP 122: Remove the Permanent Generation](https://openjdk.org/jeps/122) — PermGen→Metaspace 전환, 클래스 메타데이터는 네이티브 메모리, interned String/static 변수는 Heap으로 이동.

## 검증 상태

- [x] 1차 출처 확인됨 (JVMS SE21 §2.5 원문, JEP 122 원문)
- [ ] 코드/실측으로 직접 검증함
- [ ] 미검증 항목 없음

## 모르는 것 / 후속 과제

- Run-Time Constant Pool의 구체적 내부 구조(상수 타입별 표현)는 미확인 — JVMS §4.4 후속 학습 필요.
- 스택 프레임(2.6 Frames: 지역변수 배열, 오퍼랜드 스택, 동적 링킹)은 다음 세션에서 다룰 것.

## 승격된 지식 노트

- [런타임-데이터-영역](../topics/jvm/런타임-데이터-영역.md)
