# Docker

> 태그: `#infra` `#docker` `#container`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

Docker는 애플리케이션과 그 실행에 필요한 라이브러리·설정·런타임 환경을 하나의 이미지(Image)로 묶고, 이를 호스트 OS 커널을 공유하는 격리된 컨테이너(Container)로 실행함으로써 "내 PC에서는 되는데?" 문제를 줄이고 환경 일관성을 보장하는 컨테이너 플랫폼이다.

## 특징 / 상세

### 1. Docker란 무엇인가?

- 애플리케이션과 그 실행에 필요한 **라이브러리·설정·런타임 환경**을
  하나의 **이미지(Image)** 로 묶고, 이를 **컨테이너(Container)** 로 실행하는 플랫폼
- 특징
  - 가볍다: VM처럼 OS 전체를 띄우지 않고, **호스트 OS 커널을 공유**
  - 이식성: 어디서 실행해도 동일한 환경 (개발·테스트·운영 일관성)
  - 배포 용이: 이미지 한 개로 여러 서버에 동일하게 배포 가능

### 2. 핵심 개념 정리

- **Image**
  - "실행 파일 + 의존성 + 설정"이 포함된 **불변(Immutable) 템플릿**
  - `Dockerfile`을 기반으로 빌드 (`docker build`)

- **Container**
  - 이미지를 실제로 실행한 **격리된 프로세스**
  - 상태를 가지며, 종료하면 기본적으로 메모리/파일 시스템 변화는 사라짐 (볼륨 제외)

- **Dockerfile**
  - 이미지를 어떻게 만들지 정의하는 스크립트
  - 예: 베이스 이미지, 복사 파일, 의존성 설치, 실행 커맨드 등

- **Registry**
  - 이미지를 저장하고 공유하는 저장소 (Docker Hub, GitHub Container Registry, ECR 등)

- **Volume**
  - 컨테이너 외부에 데이터를 저장하기 위한 공간
  - 컨테이너를 지워도 데이터는 유지

### 3. 기본 명령어

#### 3.1 이미지 관련

- 이미지 목록 확인:
  `docker images`
- 이미지 가져오기(pull):
  `docker pull nginx:latest`
- 이미지 삭제:
  `docker rmi 이미지ID`

#### 3.2 컨테이너 관련

- 컨테이너 실행(앞에서 보기):
  `docker run --name my-nginx -p 8080:80 nginx:latest`
- 백그라운드 실행:
  `docker run -d --name my-nginx -p 8080:80 nginx:latest`
- 실행 중 컨테이너 목록:
  `docker ps`
- 모두 포함 목록(종료 포함):
  `docker ps -a`
- 로그 보기:
  `docker logs -f my-nginx`
- 컨테이너 접속:
  `docker exec -it my-nginx /bin/bash`
- 컨테이너 중지/시작/삭제:
  `docker stop my-nginx`
  `docker start my-nginx`
  `docker rm my-nginx`

#### 3.3 Dockerfile로 이미지 빌드

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- 빌드: `docker build -t my-app:1.0 .`
- 실행: `docker run -d -p 8080:8080 --name my-app my-app:1.0`

### 4. Dockerfile 작성 시 팁

- **레이어 최소화**
  - `RUN` 명령을 적절히 합쳐 레이어 수를 줄이고, 캐시 재활용 극대화
- **빌드 스테이지와 런타임 스테이지 분리(Multi-stage build)**
  - 빌드용 이미지(예: Gradle, Maven)와 실행용 이미지(경량 JRE/앱 전용)를 분리해 최종 이미지 크기 감소
- **.dockerignore 활용**
  - 불필요한 파일을 이미지 빌드 컨텍스트에서 제외 (`.git`, `node_modules`, build 결과물 등)
- **환경변수/비밀 값 관리**
  - 비밀번호/키를 이미지에 직접 넣지 말고, 환경 변수 + 외부 시크릿 관리 시스템 사용

### 5. 데이터 관리 – Volume과 Bind Mount

- **Volume**
  - Docker가 관리하는 데이터 디렉터리, 컨테이너와 독립적인 저장소
  - 예: `docker run -v my-volume:/var/lib/mysql mysql:8`

- **Bind Mount**
  - 호스트의 특정 디렉터리를 컨테이너에 그대로 마운트
  - 예: `docker run -v $(pwd)/logs:/app/logs my-app:1.0`

사용 팁:

- DB 데이터, 업로드 파일 등은 **볼륨/마운트로 외부에 저장**해 컨테이너 재배포 시 데이터 유지
- 코드 핫리로드(개발 환경)는 Bind Mount로 소스 디렉터리를 연결해서 사용

### 6. 네트워크와 포트

- 기본적으로 컨테이너는 **자기 고유의 네트워크 네임스페이스**를 가진다.

주요 개념:

- `-p 호스트포트:컨테이너포트`
  - 외부에서 컨테이너에 접근할 수 있도록 포트 매핑
- Docker 네트워크
  - `bridge`: 기본 브리지 네트워크 (컨테이너 간 내부 통신)
  - `host`: 호스트 네트워크를 그대로 사용하는 모드 (Linux 한정)
  - `none`: 네트워크 없음
- 커스텀 네트워크 생성
  - `docker network create my-net`
  - `docker run --network my-net ...`

### 7. 자주 겪는 문제와 원인

#### 7.1 "컨테이너 지우니 데이터가 다 날아갔다"

- 원인
  - 컨테이너 내부 파일 시스템에 직접 저장 (볼륨/마운트 사용 안 함)
- 해결
  - 데이터는 **반드시 Volume 또는 Bind Mount**에 저장

#### 7.2 "로컬에서는 되는데 컨테이너에서는 안 된다"

- 원인 예시
  - 의존 프로그램/라이브러리 설치 누락
  - 환경 변수/설정 파일이 이미지에 반영되지 않음
  - 네트워크/포트 설정 문제
- 해결
  - Dockerfile에 **실행에 필요한 모든 의존성 명시**
  - 환경변수는 `-e` 또는 `env_file`로 주입
  - 로그를 `docker logs`로 확인해 에러 메시지 기반으로 추적

#### 7.3 "포트는 열었는데 접속이 안 된다"

- 원인
  - 앱이 `localhost` 또는 `127.0.0.1`에만 바인딩
  - 포트 매핑(`-p`) 누락 또는 잘못된 매핑
- 해결
  - 애플리케이션을 `0.0.0.0`에 바인딩
  - `docker ps`로 실제 포트 매핑 확인

#### 7.4 이미지가 너무 크다

- 원인
  - 불필요한 파일 포함, 빌드 도구까지 한 이미지에 모두 포함
- 해결
  - **경량 베이스 이미지 사용** (예: `alpine`)
  - Multi-stage build로 빌드용/실행용 분리
  - `.dockerignore`로 불필요한 파일 제외

### 8. Docker Compose 간단 소개

- 여러 컨테이너(서비스)를 하나의 YAML 파일로 정의하고, 한 번에 올렸다 내릴 수 있게 해주는 도구

예시 (`docker-compose.yml`):

```yaml
version: "3.8"
services:
  app:
    image: my-app:1.0
    ports:
      - "8080:8080"
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

- 실행: `docker compose up -d`
- 종료: `docker compose down`

### 9. 요약

- Docker는 **애플리케이션 실행 환경을 이미지로 고정하고, 이를 컨테이너로 실행하는 기술**이다.
- 기본 개념(이미지, 컨테이너, Dockerfile, 볼륨, 네트워크)을 이해하면
  "내 PC에서는 되는데…" 문제를 크게 줄이고, **일관된 배포·운영 환경**을 만들 수 있다.
- 데이터는 볼륨/마운트로, 트래픽은 포트 매핑/네트워크로,
  복수 서비스는 Docker Compose로 관리하는 패턴을 익히면 실무 적용이 훨씬 수월하다.

## 트레이드오프

| 항목 | 내용 |
|---|---|
| 일관성 | 이미지가 불변 템플릿이므로 개발/테스트/운영 환경 동일성 보장. 단 베이스 이미지 태그를 `latest`로 두면 빌드 시점마다 내용이 달라져 일관성 깨짐 위험 |
| 가용성 | 컨테이너 자체는 단일 장애점(호스트 다운 시 전체 중단) — Compose 단일 호스트 구성은 오케스트레이션(K8s 등) 없이는 고가용성 보장 안 됨 |
| 지연 | 호스트 OS 커널 공유로 VM 대비 기동/오버헤드 적음. 네트워크는 기본 `bridge` 모드에서 NAT 오버헤드 존재, `host` 모드는 오버헤드 없지만 격리 약화 |
| 비용 | 경량 베이스 이미지(alpine)·Multi-stage build로 이미지 크기를 줄이면 레지스트리 저장/전송 비용 절감 |
| 운영부담 | 데이터 영속성을 위해 Volume/Bind Mount 관리 필요, 네트워크/포트 매핑 설정 실수가 흔한 장애 원인(7번 항목 참고) — Compose 이상 규모에서는 별도 오케스트레이션 도구 학습 부담 발생 |

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 7번 "자주 겪는 문제와 원인" 참고)

## 참고

- 원본 학습 노트(TIL)에 `> 연결된 정리본: Docker 기초` 링크(`../../../TIL.wiki/docker-basics-images-containers-volumes.md`)가 있었으나, 이 볼트 외부의 위키 경로라 마크다운 링크로 옮기지 않음. 필요 시 원본 TIL 위키에서 직접 확인.

## 관련 내용

- (관련 노트 없음)
