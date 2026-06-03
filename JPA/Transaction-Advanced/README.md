# JPA 트랜잭션 고급 전략

> Spring `@Transactional`과 JPA/Hibernate를 기준으로, 실무에서 자주 만나는 트랜잭션 문제와 해결 전략을 개별 문서로 정리한다.

## 문서 구성

| 문서 | 다루는 질문 |
| --- | --- |
| [트랜잭션 경계와 전파 전략](./01-transaction-boundary-and-propagation.md) | 트랜잭션은 어디서 시작하고 어디서 끝내야 하는가? `REQUIRED`, `REQUIRES_NEW`, `NESTED`는 언제 쓰는가? |
| [동시성 제어와 락 전략](./02-locking-and-concurrency.md) | 낙관적 락, 비관적 락, 조건부 update 중 무엇을 선택해야 하는가? |
| [외부 시스템과 트랜잭션 일관성](./03-external-system-consistency.md) | 결제, 메시지, 알림 같은 외부 시스템과 DB 변경을 어떻게 일관되게 다룰 것인가? |
| [조회 트랜잭션, flush, Lazy Loading](./04-read-flush-lazy-loading.md) | `readOnly`, flush 시점, OSIV, Lazy Loading 문제를 어떻게 다룰 것인가? |
| [배치, 데드락, 긴 트랜잭션](./05-batch-deadlock-long-transaction.md) | 대량 처리와 락 경합을 어떻게 줄일 것인가? |
| [@Transactional 실무 함정](./06-transactional-pitfalls.md) | self-invocation, rollback-only, checked exception 같은 함정을 어떻게 피할 것인가? |
| [실무 안티패턴 체크리스트](./07-practical-antipattern-checklist.md) | 실제 장애로 이어지기 쉬운 트랜잭션 실수는 무엇인가? |

## 핵심 판단 기준

JPA 트랜잭션 전략은 보통 다음 질문으로 결정한다.

1. 이 작업은 하나의 DB 트랜잭션으로 원자적이어야 하는가?
2. 동시에 같은 데이터를 수정할 때 충돌을 감지하고 재시도할 것인가, 먼저 잠글 것인가?
3. 외부 시스템과의 일관성은 즉시 일관성인가, 최종 일관성인가?
4. 실패 이력, 이벤트, 로그는 메인 트랜잭션과 같이 롤백되어도 되는가?
5. 트랜잭션이 DB 커넥션, row lock, 영속성 컨텍스트를 얼마나 오래 붙잡는가?

## 참고 문서

- [Spring Framework Reference - Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/declarative/annotations.html)
- [Spring Framework Reference - Transaction Propagation](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/declarative/tx-propagation.html)
- [Hibernate ORM User Guide - Locking](https://github.com/hibernate/hibernate-orm/blob/main/documentation/src/main/asciidoc/userguide/chapters/locking/Locking.adoc)
