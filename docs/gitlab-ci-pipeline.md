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

### 구성 기준

| 항목 | 내용 |
| --- | --- |
| Stage | `validate` |
| 목적 | Repository 구조와 Compose 설정 검증 |
| 검증 대상 | README, docs, GitLab Compose 파일 |
| 추가 검증 | Docker Compose config 문법 확인 |
| 적용 시점 | 기본 Pipeline 구성 이후 |

### 판단

초기 Validate Job은 필수 파일 존재 여부 확인부터 시작한다.

Docker Compose config 검증은 Docker socket 전달 설정이 확정된 뒤 추가한다.

---

## 2. Test Job 구성

### 목적

Pipeline 안에서 기본 테스트 Job을 실행하여 Runner tag 매칭과 Job 실행 흐름을 확인한다.

### 검증 항목

- Runner tag 매칭
- Job 로그 출력
- Job 상태 `Passed`

---

## 3. Docker Build Job 구성

### 목적

Docker Image build가 필요한 서비스에 대해 Image build 단계를 구성한다.

### 현재 판단

현재 GitLab 서버는 `gitlab/gitlab-ce:latest` 이미지를 사용하므로 직접 build하지 않는다.

| 대상 | 판단 |
| --- | --- |
| GitLab Omnibus Container | 직접 build하지 않음 |
| Nginx Custom Image | Reverse Proxy 단계에서 검토 |
| Monitoring Exporter | Monitoring 단계에서 검토 |
| 테스트용 App | Pipeline 검증용으로 추가 가능 |

### Phase 5 기준

Docker Build Job은 구조만 확보한다.

실제 운영 이미지 build는 대상 서비스가 생긴 뒤 추가한다.

---

## 4. Deploy Job 구성

### 목적

GitLab Pipeline에서 Mini PC의 Docker Compose 기반 배포 명령을 실행한다.

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

Phase 5에서는 GitLab 서버 자체 재배포보다 테스트 서비스 또는 설정 검증 중심으로 진행한다.

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

### 판단

Docker Compose 검증이나 Docker Build Job을 CI Job에서 실행하려면 Docker socket 전달이 필요하다.

Docker socket mount는 권한이 강하므로 신뢰 가능한 내부 Repository에서만 사용한다.

---

## 초기 `.gitlab-ci.yml` 구성안

Phase 5 초기 검증용 Pipeline은 실제 배포보다 GitLab Runner 기반 Pipeline 흐름 검증을 우선한다.

### 구성 기준

| 항목 | 내용 |
| --- | --- |
| Stage | `validate`, `test` |
| 목적 | Repository 구조 확인 및 Runner Job 실행 검증 |
| Runner tag | `docker`, `mini-pc`, `wsl2` |
| 기본 이미지 | `alpine:latest` |
| Docker 명령 사용 | 제외 |

### 검증 항목

- 필수 파일 존재 여부 확인
- GitLab Runner가 Job을 정상 수신하는지 확인
- Job 로그가 정상 출력되는지 확인
- Pipeline `Passed` 확인

### 판단

초기 Pipeline에서는 Docker 명령을 사용하지 않는다.

Docker 명령 검증은 Docker socket 전달 설정을 확정한 뒤 별도 단계에서 추가한다.

---

## Docker Compose 검증용 Pipeline 구성안

Docker Compose 검증 Job은 Docker socket 전달 설정을 적용한 뒤 사용한다.

### 목적

GitLab Runner Job 안에서 Docker 명령을 실행할 수 있는지 확인하고, Compose 설정 파일의 문법 오류를 사전에 검증한다.

### 구성 기준

| 항목 | 내용 |
| --- | --- |
| Stage | `validate` |
| 목적 | Docker Compose 설정 검증 |
| 필요 조건 | CI Job Container에서 Docker daemon 접근 가능 |
| 주요 검증 | `docker version`, `docker compose config` |
| 적용 시점 | Docker socket 전달 설정 이후 |

### 판단

현재 Phase 5 초반에는 Docker Compose 검증을 바로 적용하지 않는다.

먼저 기본 Pipeline이 `Passed` 되는 것을 확인한 뒤, Docker socket 전달 설정을 확정하고 Docker Compose 검증 Job을 추가한다.

---

## Deploy Job 구성안

Deploy Job은 실제 배포 명령을 바로 실행하지 않고, 초기에는 dry-run 또는 상태 확인 중심으로 구성한다.

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

GitLab 서버 자체를 GitLab Pipeline으로 재배포하는 구조는 순환 의존성이 생길 수 있다.

따라서 Phase 5에서는 GitLab 서버 자체 재배포가 아니라, 테스트 서비스 또는 설정 검증 중심으로 Deploy Job 범위를 정한다.

실제 `docker compose up -d` 기반 배포는 배포 대상 서비스와 롤백 기준이 정리된 뒤 적용한다.

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