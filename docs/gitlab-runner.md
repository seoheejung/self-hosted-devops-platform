# GitLab Runner

> GitLab 서버 구축 이후, CI/CD Job 실행 책임을 GitLab 서버 Container에서 분리하고 별도 GitLab Runner를 등록하여 Pipeline 실행 환경을 구성한다.

---

## 목적

GitLab Container는 소스 코드 저장소, Web UI, 사용자 관리, Pipeline 상태 관리 역할에 집중한다.

Build, Test, Deploy 같은 실제 작업 실행은 별도 GitLab Runner가 담당한다.

즉, GitLab은 “작업을 만들고 관리하는 서버”이고, GitLab Runner는 “작업을 실제로 실행하는 실행기”다.

```text
GitLab Container
 ├─ Repository 관리
 ├─ Pipeline 생성
 └─ Job 상태 관리

GitLab Runner
 ├─ Job 수신
 ├─ Build 실행
 ├─ Test 실행
 └─ Docker Executor 기반 Job 실행
```

 ### 용어 정리

| 용어 | 의미 |
| --- | --- |
| Repository | 소스 코드와 변경 이력을 저장하는 공간 |
| Web UI | 브라우저에서 GitLab 프로젝트, 사용자, Pipeline 상태를 관리하는 화면 |
| Pipeline | 코드 변경 후 자동으로 실행되는 작업 흐름 |
| CI Job | Pipeline 안에서 실행되는 개별 작업 단위. 예: Build, Test, Deploy |
| GitLab Runner | GitLab에서 생성한 CI Job을 실제로 실행하는 프로그램 |
| Docker Executor | CI Job을 Docker Container 안에서 실행하는 Runner 실행 방식 |

---

## Runner 역할 구분

| 구분 | 역할 | 사용 시점 |
| --- | --- | --- |
| GitHub self-hosted runner | GitHub Actions Job 실행 | GitLab 서버 초기 부트스트랩 배포 |
| GitLab Runner | GitLab Pipeline Job 실행 | GitLab 서버 구축 이후 CI/CD 실행 |

> GitHub self-hosted runner는 GitLab 서버를 배포하기 위한 초기 자동화 수단이다.
> GitLab Runner는 GitLab 내부 Repository의 `.gitlab-ci.yml`을 실행하기 위한 CI/CD 실행기다.

---

## 자동화 범위 구분

현재 프로젝트에는 두 종류의 자동화 흐름이 존재한다.

| 구분 | 역할 | 실행 주체 | 대상 |
| --- | --- | --- | --- |
| GitHub Actions 자동 배포 | GitLab 서버 Container 배포 및 갱신 | GitHub self-hosted runner | Mini PC Docker Compose |
| GitLab Runner Pipeline | GitLab Repository의 CI Job 실행 | GitLab Runner | `.gitlab-ci.yml` |

### GitHub Actions 자동 배포

GitHub Actions 자동 배포는 GitLab 서버를 Mini PC에 배포하기 위한 초기 부트스트랩 자동화다.

```text
GitHub Repository
 └─ main merge
        ↓

GitHub Actions
 └─ Mini PC self-hosted runner
        ↓

Mini PC WSL2 Ubuntu
 └─ Docker Compose 실행
        ↓

GitLab Container 배포 / 갱신
```
이 자동화는 GitLab 서버 자체를 배포하는 것이 목적이다.

### GitLab Runner Pipeline

GitLab Runner Pipeline은 구축된 GitLab 서버 안에서 Repository의 CI Job을 실행하기 위한 자동화다.

```
GitLab Repository
 └─ .gitlab-ci.yml commit / push
        ↓

GitLab Pipeline 생성
        ↓

GitLab Runner
 └─ CI Job 실행
        ↓

Build / Test / Deploy
```

GitLab Runner는 GitHub Repository의 변경 사항을 자동으로 가져오지 않는다.

`.gitlab-ci.yml`은 GitLab 서버 안에 생성된 Repository에 존재해야 하며, 해당 GitLab Repository에 push될 때 Pipeline이 실행된다.

### 주의

GitHub Repository와 Mini PC GitLab Repository는 서로 다른 저장소다.

```
GitHub Repository
 └─ GitLab 서버 배포 소스

Mini PC GitLab Repository
 └─ GitLab Runner Pipeline 실행 대상
```

따라서 GitHub Repository에 `.gitlab-ci.yml`을 추가해도 Mini PC GitLab Repository에 자동 반영되지 않는다.

Phase 4에서는 GitLab Runner 동작 검증이 목적이므로, 별도 테스트 프로젝트를 생성해 `.gitlab-ci.yml`을 추가하고 Pipeline `Passed`를 확인한다.


---

## 구성 기준

| 항목 | 내용 |
|---|---|
| Runner 위치 | Mini PC WSL2 Ubuntu |
| 실행 방식 | Docker Container |
| Executor | Docker Executor |
| Runner config 경로 | `/home/<USER>/gitlab-runner/config` |
| Runner 이름 | `mini-pc-docker-runner` |
| Tags | `docker`, `mini-pc`, `wsl2` |
| 기본 이미지 | `alpine:3.20` |

---

## 선택 이유

- GitLab 서버와 CI Job 실행 책임을 분리한다.
- Runner를 Container로 실행하여 재시작과 제거가 쉽도록 구성한다.
- Docker Executor를 사용해 Job 실행 환경을 Container 단위로 격리한다.
- 이후 필요하면 Main PC, 별도 Linux 서버, VM 기반 Runner로 확장할 수 있다.

---

## 운영 경로

```
/home/<USER>/gitlab-runner/
└─ config/
   └─ config.toml
```

| 경로 | 역할 |
| --- | --- |
| `/home/<USER>/gitlab-runner/config` | Runner 설정 파일 저장 |
| `/etc/gitlab-runner` | Runner Container 내부 설정 경로 |
| `/var/run/docker.sock` | Docker Executor가 Host Docker daemon에 접근하기 위한 socket |

---

## Container 실행 기준

GitLab Runner는 GitLab 서버 버전과 맞춘 고정 버전 이미지를 사용한다.

```
Container name: gitlab-runner
Restart policy: unless-stopped
Config volume: /home/<USER>/gitlab-runner/config:/etc/gitlab-runner
Docker socket: /var/run/docker.sock:/var/run/docker.sock
Runner image: gitlab/gitlab-runner:alpine-v18.11.2
```

---

## Runner 등록 기준

| 항목 | 값 |
| --- | --- |
| Runner type | Instance runner |
| Description | `mini-pc-docker-runner` |
| Tags | `docker`, `mini-pc`, `wsl2` |
| Executor | `docker` |
| Default image | `alpine:3.20` |

> Runner 생성은 GitLab Web UI에서 수행하고, Runner 등록은 Mini PC WSL2 Ubuntu의 `gitlab-runner` Container에서 수행한다.

---

## Runner 등록 토큰 발급

GitLab Web UI에서 Runner를 먼저 생성하고, 생성 후 표시되는 Runner authentication token을 사용해 Runner Container를 등록한다.

```text
GitLab Web UI
 → Admin Area
 → CI/CD
 → Runners
 → Create instance runner
```

### Runner 생성 기준
| 항목                | 값                           |
| ----------------- | --------------------------- |
| Runner type       | Instance runner             |
| Description       | `mini-pc-docker-runner`     |
| Tags              | `docker`, `mini-pc`, `wsl2` |
| Run untagged jobs | 필요 시 활성화                    |

> 생성 후 표시되는 Runner authentication token을 복사한다.

#### Token 관리 기준
- Runner authentication token은 GitLab Runner 등록에 사용한다.
- Token은 문서, README, Git commit에 남기지 않는다.
- Token이 노출되면 GitLab UI에서 해당 Runner를 삭제하고 다시 생성한다.
- 기존 Runner registration token 방식은 deprecated 상태이며 GitLab 20.0에서 제거 예정이므로 사용하지 않는다.
- Runner authentication token은 보통 `glrt-` prefix를 가진다.

---

## 테스트 Pipeline 기준

Phase 4에서는 Docker image build나 deploy job을 만들지 않는다.

검증 목적은 GitLab Runner가 GitLab Repository의 Job을 정상 수신하고 실행하는지 확인하는 것이다.

```text
Mini PC GitLab 테스트 Repository
 → .gitlab-ci.yml commit / push
 → Pipeline 생성
 → GitLab Runner가 Job 수신
 → alpine 기반 test job 실행
 → Job Passed 확인
```

테스트 Job 기준:

- `alpine:3.20` 이미지 사용
- Runner tag 지정
- 간단한 echo / uname 명령 실행
- Pipeline `Passed` 확인

---

## 주의사항

- GitHub self-hosted runner와 GitLab Runner는 역할이 다르다.
- GitHub self-hosted runner는 GitLab 서버 초기 배포용이다.
- GitLab Runner는 GitLab Pipeline Job 실행용이다.
- Docker socket mount는 강한 권한을 가지므로 신뢰 가능한 프로젝트에서만 사용한다.
- Docker image build 및 deploy job은 Phase 5에서 별도 구성한다.

---