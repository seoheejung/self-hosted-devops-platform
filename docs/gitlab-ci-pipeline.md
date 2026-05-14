# GitLab CI/CD Pipeline

> GitLab Runner 기반으로 Build, Test, Docker 검증, Deploy 흐름을 구성하고 GitLab Repository 변경 사항이 Pipeline으로 실행되는 구조를 검증한다.

---

## 목적

Phase 4에서는 GitLab Runner가 Job을 정상 수신하고 실행하는지 확인했다.

Phase 5에서는 GitLab Runner를 기반으로 실제 CI/CD Pipeline 구조를 구성한다.

```text
GitLab Repository
 └─ Git Push / Merge
        ↓

GitLab Pipeline
 ├─ Build
 ├─ Test
 ├─ Docker Build
 └─ Deploy
        ↓

Mini PC
 └─ Docker Compose 기반 서비스 반영
```

---

## 전제 조건

| 항목 | 기준 |
| --- | --- |
| GitLab Web UI | `http://172.30.1.69:8080` 접근 가능 |
| GitLab Container | `healthy` 상태 |
| GitLab Runner | `Online / Idle` 상태 |
| Runner Tags | `docker`, `mini-pc`, `wsl2` |
| Executor | Docker Executor |
| Docker Desktop | 실행 중 |
| WSL2 Ubuntu | 실행 중 |

---

## Pipeline 설계 기준

> 초기 Pipeline은 실제 운영 배포보다 검증 중심으로 구성한다.

```
validate
 ├─ 필수 파일 존재 여부 확인
 └─ Docker Compose config 검증

test
 └─ Runner Job 실행 검증

docker-build
 └─ Docker Image build 구조 검토

deploy
 └─ Mini PC 배포 명령 실행 범위 검토
```

---

## Stage 구성

```
stages:
  - validate
  - test
  - docker-build
  - deploy
```

| Stage | 역할 |
| --- | --- |
| `validate` | Repository 구조와 Compose 설정 검증 |
| `test` | 기본 테스트 및 Runner 실행 검증 |
| `docker-build` | Docker Image build 가능 여부 검증 |
| `deploy` | Mini PC 서비스 반영 |

---

## 1. Validate Job 구성

### 목적

Repository 구조와 Docker Compose 설정이 정상인지 확인한다.

### 검증 항목

- 필수 문서 존재 여부
- GitLab Compose 파일 존재 여부
- GitLab Runner 문서 존재 여부
- Docker Compose 설정 문법 검증

### 검증 대상
```
README.md
docs/gitlab-runner.md
infra/compose/docker-compose.gitlab.yml
```

### 구성 기준

| 항목        | 내용                                        |
| --------- | ----------------------------------------- |
| Stage     | `validate`                                |
| 목적        | Repository 구조와 Compose 설정 검증              |
| 검증 대상     | README, docs, GitLab Compose 파일           |
| Docker 검증 | `docker version`, `docker compose config` |
| 실패 처리     | 검증 실패 시 Pipeline 중단                       |


### 판단

Validate Job은 실제 배포 전에 반드시 통과해야 하는 사전 검증 단계다.

Docker Compose 설정 오류가 있는 상태에서 Deploy Job이 실행되면 기존 GitLab Container에 영향을 줄 수 있으므로, Deploy 전에 Compose config 검증을 수행한다.

---

## 2. Test Job 구성

### 목적

Pipeline 안에서 기본 테스트 Job을 실행하여 Runner tag 매칭과 Job 실행 흐름을 확인한다.

### 검증 항목
- Runner tag 매칭
- Job 로그 출력
- Job 상태 `Passed`

### 판단

현재 Repository에는 별도 애플리케이션 테스트 코드가 없다.

그래서 Test Job은 애플리케이션 단위 테스트가 아니라 GitLab Runner 기반 Job 실행 흐름 검증으로 사용한다.

---

## 3. Docker Build Job 구성

### 목적

Docker Image build가 필요한 서비스에 대해 Image build 단계를 구성할 수 있는지 검토한다.

### 적용 기준

현재 GitLab 서버는 `gitlab/gitlab-ce:latest` 이미지를 사용하므로 직접 build하지 않는다.

직접 build할 Dockerfile이 생기면 `docker-build` stage에 별도 Job을 추가한다.

| 대상                       | 판단                    |
| ------------------------ | --------------------- |
| GitLab Omnibus Container | 직접 build하지 않음         |
| Nginx Custom Image       | Reverse Proxy 단계에서 검토 |
| Monitoring Exporter      | Monitoring 단계에서 검토    |
| 테스트용 App                 | Pipeline 검증용으로 추가 가능  |

---

## 4. Deploy Job 구성

### 목적

GitLab Pipeline에서 Mini PC의 Docker Compose 기반 배포 명령을 실행할 수 있는 구조를 검토한다.

### 배포 방식 후보

| 방식 | 설명 | 판단 |
| --- | --- | --- |
| Runner가 직접 Docker Compose 실행 | Runner가 Mini PC WSL2의 Docker daemon 사용 | 현재 구조에 적합 |
| SSH Deploy | Runner가 Mini PC로 SSH 접속 | 현재 불필요 |
| GitHub Actions Deploy 유지 | GitLab 서버 Container 자체 배포 | 부트스트랩용으로 유지 |

### 현재 선택

GitLab Runner는 Mini PC WSL2 Ubuntu 내부에서 실행된다.

그래서 Deploy Job은 Runner가 직접 Docker Compose 명령을 실행하는 방식으로 설계한다.

```
GitLab Pipeline Deploy Job
 → GitLab Runner
 → Docker Compose 실행
 → Mini PC 서비스 반영
```

### 주의

GitLab 서버 자체를 GitLab Pipeline으로 재배포하는 구조는 순환 의존성이 생길 수 있다.

```
GitLab Pipeline
 → GitLab Runner
 → GitLab Container 재배포
 → GitLab Pipeline 실행 환경 영향
```

따라서 현재 Phase 5에서는 GitLab 서버 자체 재배포보다 테스트 서비스 또는 설정 검증 중심으로 Deploy Job 범위를 정한다.

실제 `docker compose up -d` 기반 배포는 다음 조건을 만족한 뒤 적용한다.

- 배포 대상 서비스가 명확함
- `docker compose up -d` 실행 시 기존 GitLab Container에 영향이 없음
- 실패 시 수동 복구 절차가 있음
- Rollback 기준이 문서화되어 있음

---

## Docker Socket 사용 기준

GitLab Runner Container는 Host Docker socket을 마운트한다.

```
-v /var/run/docker.sock:/var/run/docker.sock
```

다만 CI Job Container 안에서 Docker 명령을 실행하려면 `config.toml`의 Docker volumes에도 socket 전달이 필요하다.

```
[runners.docker]
  volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
```

설정 변경 후 Runner Container를 재시작한다.

```
docker restart gitlab-runner
```

### 판단

Docker Compose 검증이나 Docker Build Job을 CI Job에서 실행하려면 Docker socket 전달이 필요하다.

Docker socket mount는 Host Docker daemon 접근 권한을 제공하므로 신뢰 가능한 내부 Repository에서만 사용한다.

---

## `.gitlab-ci.yml` 구성 기준

실제 Pipeline 코드는 Repository 루트의 `.gitlab-ci.yml`에 작성한다.

### 구성 목적

Phase 5의 목표는 GitLab Repository 변경 사항을 Mini PC 서비스 배포까지 연결할 수 있는 Pipeline 기반을 확보하는 것이다.

다만 현재 단계에서 바로 실제 배포를 실행하면 다음 문제가 생길 수 있다.

- GitLab 서버 자체를 Pipeline으로 재배포할 경우 순환 의존성이 생길 수 있음
- Docker socket 권한 설정이 완전히 검증되지 않은 상태에서 Host Docker daemon에 접근하게 됨
- Compose 설정 오류가 있을 경우 기존 GitLab Container에 영향을 줄 수 있음
- 아직 직접 build할 애플리케이션 Dockerfile이 없으므로 Docker Image Build Job을 강제로 만들 필요가 없음

그래서 이번 Phase 5에서는 다음 항목을 함께 검증한다.

1. GitLab Runner가 실제 Repository의 Pipeline Job을 정상 실행하는지 확인한다.
2. CI Job Container 안에서 Docker 명령을 실행할 수 있는지 확인한다.
3. Docker Compose config 검증이 가능한지 확인한다.
4. Deploy Job 실행 범위를 확인한다.

이 검증이 통과하면 이후 단계에서 실제 `docker compose up -d` 기반 Deploy Job을 안전하게 추가할 수 있다.

### 구성 기준

| 항목               | 내용                           |
| ---------------- | ---------------------------- |
| Pipeline 파일      | `.gitlab-ci.yml`             |
| Runner tag       | `docker`, `mini-pc`, `wsl2`  |
| Validate Job     | Docker Compose config 검증     |
| Test Job         | Runner Job 실행 검증             |
| Docker Build Job | 직접 build 대상이 생긴 뒤 추가         |
| Deploy Job       | 실제 배포 전 실행 가능 범위 확인          |
| Docker 명령 사용     | Docker socket 전달 설정 필요       |

---

## Docker Compose 검증 기준

### 목적

GitLab Runner Job 안에서 Docker 명령을 실행할 수 있는지 확인하고, Compose 설정 파일의 문법 오류를 사전에 검증한다.

### 주요 검증
```
docker version
docker compose -f infra/compose/docker-compose.gitlab.yml config
```

### 판단

Docker Compose 검증은 실제 배포 전에 반드시 통과해야 하는 검증 단계다.

이 단계에서 실패하면 Deploy Job으로 진행하지 않는다.

---

## Deploy Job 구성안

Deploy Job은 실제 배포 명령을 바로 실행하지 않고, 먼저 상태 확인 또는 readiness check 중심으로 구성한다.

### 목적

GitLab Runner가 Mini PC 내부에서 배포 단계 Job을 실행할 수 있는지 검증한다.

### 구성 기준

| 항목 | 내용 |
| --- | --- |
| Stage | `deploy` |
| 초기 동작 | 배포 명령 직접 실행 대신 상태 확인 |
| 실제 배포 | 배포 대상 서비스가 명확해진 뒤 적용 |
| 실행 위치 | Mini PC WSL2 Ubuntu의 GitLab Runner |
| 주의 대상 | GitLab 서버 자체 재배포 |

### 판단

현재 Phase 5에서는 실제 Deploy Job 대신 배포 단계 실행 가능 여부를 확인한다.

이 Job은 `docker compose up -d`를 실행하지 않고, Runner가 배포 단계 Job을 정상 수신하고 실행할 수 있는지만 확인한다.

실제 Deploy Job은 배포 대상 서비스와 복구 기준이 정리된 뒤 추가한다.

---

## 검증 기준

### Pipeline 기준

- Pipeline 생성됨
- Runner가 Job 수신
- Job 로그 정상 출력
- Pipeline `Passed` 확인

### Runner 기준

- Runner 상태 `Online`
- Runner tag 매칭 정상
- Job stuck 상태 없음

### Docker 기준

- Docker socket 사용 여부 결정
- Docker Compose config 검증 가능
- Docker command permission 오류 없음

### Deploy 기준

- Deploy Job이 의도한 명령만 실행
- 기존 GitLab Container에 영향 없음
- 실패 시 수동 복구 가능

---