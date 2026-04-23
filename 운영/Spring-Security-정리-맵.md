# Spring Security 정리 맵 v2

이 문서는 `Spring Security` 관련 `TIL` 원문이 현재 어떤 `wiki` 대표 문서로 정리되었는지 추적하기 위한 운영 맵이다.

현재 상태는 “계획 단계”가 아니라 “1차 승격 완료 후 추적 단계”다.

## 1. 현재 대표 문서 목록

현재 `TIL.wiki`의 Spring Security 대표 문서는 아래와 같다.

- `spring-security-servlet-architecture.md`
- `spring-security-authentication-architecture.md`
- `spring-security-filter-chain-exception-handling.md`
- `spring-security-cors.md`
- `spring-security-cors-troubleshooting.md`
- `spring-security-csrf.md`
- `spring-security-custom-filter-design.md`
- `spring-filter-base-classes.md`
- `spring-security-jwt-filter-positioning.md`

비교안 문서는 아래 두 개를 유지한다.

- `spring-security-servlet-architecture-kafka-style.md`
- `spring-security-servlet-architecture-blog-style.md`

## 2. 원문 -> 대표 문서 매핑

| 원문 | 현재 역할 | 연결된 wiki 문서 |
| --- | --- | --- |
| `Tech Blog/Spring Security 아키텍처 이해하기.md` | 대표 원문 | `spring-security-servlet-architecture.md` |
| `Spring Security/Spring Security 아키텍처 이해하기.md` | 보조 원문 | `spring-security-servlet-architecture.md` |
| `Tech Blog/Spring Security 인증 아키텍처 이해하기.md` | 대표 원문 | `spring-security-authentication-architecture.md` |
| `Spring Security/Spring Security 인증 아키텍처 이해하기.md` | 보조 원문 | `spring-security-authentication-architecture.md` |
| `Tech Blog/Spring Security Filter Chain 예외 처리 전략.md` | 대표 원문 | `spring-security-filter-chain-exception-handling.md` |
| `Spring Security/Spring Security Filter Chain 예외 처리 전략.md` | 보조 원문 | `spring-security-filter-chain-exception-handling.md` |
| `Spring Security/CORS.md` | 개념 원문 | `spring-security-cors.md` |
| `Spring Security/CORS-원인-추적.md` | 추적 원문 | `spring-security-cors-troubleshooting.md`, `spring-security-cors.md` |
| `Spring Security/CSRF 공격과 방어 전략.md` | 대표 원문 | `spring-security-csrf.md` |
| `Spring Security/GenericFilterBean vs OncePerRequestFilter.md` | 대표 원문 | `spring-filter-base-classes.md`, `spring-security-custom-filter-design.md` |
| `Spring Security/Spring_Security_필터_체인.md` | 분해 소스 | `spring-security-servlet-architecture.md`, `spring-security-custom-filter-design.md`, `spring-security-jwt-filter-positioning.md`, `spring-security-filter-chain-exception-handling.md` |

## 3. 문서 역할 분리

현재 Spring Security 묶음은 아래 역할로 나뉜다.

### 3.1 구조 문서

- `spring-security-servlet-architecture.md`
- `spring-security-authentication-architecture.md`

### 3.2 트러블슈팅 / 흐름 문서

- `spring-security-filter-chain-exception-handling.md`
- `spring-security-cors-troubleshooting.md`
- `spring-security-jwt-filter-positioning.md`

### 3.3 개념 / 설계 문서

- `spring-security-cors.md`
- `spring-security-csrf.md`
- `spring-security-custom-filter-design.md`
- `spring-filter-base-classes.md`

## 4. `Spring_Security_필터_체인.md` 처리 결과

이 원문은 단일 위키 문서로 승격하지 않았다.
현재는 아래 대표 문서들의 분해 소스로 본다.

- `spring-security-servlet-architecture.md`
- `spring-security-filter-chain-exception-handling.md`
- `spring-security-custom-filter-design.md`
- `spring-security-jwt-filter-positioning.md`

즉, 이 문서는 “분해 완료된 원문” 상태다.

## 5. CORS 묶음 처리 결과

기존에는 `CORS.md`와 `CORS-원인-추적.md`를 하나의 문서로 합칠지 검토했다.
현재는 역할이 다르므로 아래처럼 분리 운영한다.

- `spring-security-cors.md`: 개념, 구조, 설정 중심
- `spring-security-cors-troubleshooting.md`: 원인 추적, 증상별 점검 중심

이 구성이 더 읽기 쉽고, 실제 사용 상황도 분리된다.

## 6. 완료 상태

현재 Spring Security 묶음은 아래 기준으로 1차 완료 상태다.

- 핵심 구조 문서 작성 완료
- 개념 문서 작성 완료
- 필터 설계 문서 작성 완료
- 트러블슈팅 문서 작성 완료
- 비교안 2종 유지 완료

따라서 현재 남은 일은 새 대표 문서 추가보다, 원문과 위키 간 연결 표시를 강화하는 운영 작업에 가깝다.
