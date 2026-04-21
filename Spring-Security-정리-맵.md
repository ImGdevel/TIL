# Spring Security 정리 맵 v1

이 문서는 `Spring Security` 관련 `TIL` 문서를 어떤 대표 문서로 묶어 `wiki`에 승격할지 정리한 작업용 맵이다.

핵심 원칙은 단순하다.

- `Tech Blog` 문서는 `wiki` 대표 문서의 직접 원문 후보로 본다.
- `Spring Security` 폴더 문서는 학습 원본과 세부 설명 소스로 유지한다.
- 루트 문서는 병합 또는 재배치 대상으로 본다.
- 겹치는 문서는 삭제하지 않고 먼저 대표/보조 관계를 확정한다.

## 1. 위키 대표 페이지 매핑

| 위키 대표 페이지 | 대표 원문 | 보조 원문 | 처리 방식 | 메모 |
| --- | --- | --- | --- | --- |
| `Spring Security Servlet 아키텍처` | `Tech Blog/Spring Security 아키텍처 이해하기.md` | `Spring Security/Spring Security 아키텍처 이해하기.md`, `Spring Security/Spring_Security_필터_체인.md` | 대표 문서 1개로 승격 | 첫 번째 위키 문서로 가장 적합 |
| `Spring Security 인증 아키텍처` | `Tech Blog/Spring Security 인증 아키텍처 이해하기.md` | `Spring Security/Spring Security 인증 아키텍처 이해하기.md` | 대표 문서 1개로 승격 | Servlet 아키텍처 다음 순서로 작성 |
| `Spring Security Filter Chain 예외 처리` | `Tech Blog/Spring Security Filter Chain 예외 처리 전략.md` | `Spring Security/Spring Security Filter Chain 예외 처리 전략.md`, `Spring Security/Spring_Security_필터_체인.md` | 대표 문서 1개로 승격 | 예외 흐름과 엔트리 포인트를 여기로 수렴 |
| `Spring Security CORS` | `Spring Security/CORS.md` | `Spring-Security-CORS.md` | 병합 후 승격 | 개념 문서 + 문제 추적 사례 문서 조합 |
| `Spring Security CSRF` | `Spring Security/CSRF 공격과 방어 전략.md` | 없음 | 단독 승격 | 품질이 충분하면 바로 위키화 가능 |
| `Spring Security 커스텀 필터 설계` | `Spring Security/GenericFilterBean vs OncePerRequestFilter.md` | `Spring Security/Spring_Security_필터_체인.md` | 재구성 후 승격 | 필터 위치, 중복 실행, 베이스 클래스 선택을 묶음 |

## 2. 파일별 처리 결정

### 2.1 대표 원문으로 사용하는 문서

- `Tech Blog/Spring Security 아키텍처 이해하기.md`
- `Tech Blog/Spring Security 인증 아키텍처 이해하기.md`
- `Tech Blog/Spring Security Filter Chain 예외 처리 전략.md`
- `Spring Security/CORS.md`
- `Spring Security/CSRF 공격과 방어 전략.md`
- `Spring Security/GenericFilterBean vs OncePerRequestFilter.md`

### 2.2 보조 원문으로 남기는 문서

- `Spring Security/Spring Security 아키텍처 이해하기.md`
- `Spring Security/Spring Security 인증 아키텍처 이해하기.md`
- `Spring Security/Spring Security Filter Chain 예외 처리 전략.md`
- `Spring-Security-CORS.md`

### 2.3 분해해서 사용하는 문서

- `Spring Security/Spring_Security_필터_체인.md`

이 문서는 현재 형태 그대로 `wiki`에 올리기보다 아래 세 군데로 나눠 쓰는 편이 낫다.

- `Servlet 아키텍처`
- `Filter Chain 예외 처리`
- `커스텀 필터 설계`

## 3. 병합 규칙

### 3.1 아키텍처 3종

- `Spring Security 아키텍처 이해하기`
- `Spring Security 인증 아키텍처 이해하기`
- `Spring Security Filter Chain 예외 처리 전략`

이 세 문서는 `Tech Blog` 버전을 대표 원문으로 삼는다.
동일 제목의 `Spring Security` 폴더 문서는 더 러프한 학습 기록으로 보고, `wiki` 작성 시 세부 설명이나 예시를 보강하는 소스로 사용한다.

### 3.2 CORS 묶음

- `Spring Security/CORS.md`
- `Spring-Security-CORS.md`

두 문서의 역할은 다르다.

- `Spring Security/CORS.md`: 개념과 설정 중심의 대표 문서
- `Spring-Security-CORS.md`: 필터 체인 관점에서 원인 추적한 사례 문서

따라서 삭제보다는 하나의 `wiki` 문서 안에서 `개념 + 사례` 구조로 합치는 편이 좋다.

### 3.3 필터 체인 / 커스텀 필터 묶음

- `Spring Security/Spring_Security_필터_체인.md`
- `Spring Security/GenericFilterBean vs OncePerRequestFilter.md`
- `Spring Security/Spring Security Filter Chain 예외 처리 전략.md`

이 세 문서는 모두 `필터 체인`을 축으로 연결된다.
다만 독자의 질문은 서로 다르다.

- 구조가 궁금한가
- 예외 처리가 궁금한가
- 커스텀 필터를 어디에 넣을지가 궁금한가

따라서 `wiki`에서는 하나의 긴 문서보다 세 갈래 대표 문서로 나누는 편이 낫다.

## 4. 실제 작성 순서

`wiki` 작성은 아래 순서로 진행하는 것이 가장 자연스럽다.

1. `Spring Security Servlet 아키텍처`
2. `Spring Security 인증 아키텍처`
3. `Spring Security Filter Chain 예외 처리`
4. `Spring Security CORS`
5. `Spring Security CSRF`
6. `Spring Security 커스텀 필터 설계`

이 순서를 택한 이유는 앞 문서가 뒤 문서의 전제 개념이 되기 때문이다.

## 5. 완료 기준

아래 조건을 만족하면 `Spring Security` 1차 배치는 끝난 것으로 본다.

- 위키 대표 문서 6개의 제목이 확정되어 있다.
- 각 문서의 대표 원문과 보조 원문이 정해져 있다.
- `Spring_Security_필터_체인.md`의 분해 방향이 정해져 있다.
- 다음 작업에서 바로 첫 위키 문서를 작성할 수 있다.
