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

## 구성 방식

| 항목 | 내용 |
| --- | --- |
| Runner 위치 | Mini PC WSL2 Ubuntu |
| 실행 방식 | Docker Container |
| Executor | Docker Executor |
| Runner 용도 | GitLab CI/CD Job 실행 |
| GitLab URL | `http://172.30.1.69:8080` |
| Runner config 경로 | `/home/<USER>/gitlab-runner/config` |

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

GitLab Runner는 `gitlab/gitlab-runner` 이미지를 사용한다.

```
Container name: gitlab-runner
Restart policy: unless-stopped
Config volume: /home/<USER>/gitlab-runner/config:/etc/gitlab-runner
Docker socket: /var/run/docker.sock:/var/run/docker.sock
```

---

## Runner 등록 기준

| 항목 | 값 |
| --- | --- |
| Runner type | Instance runner |
| Description | `mini-pc-docker-runner` |
| Tags | `docker`, `mini-pc`, `wsl2` |
| Executor | `docker` |
| Default image | `alpine:latest` |

- Runner authentication token은 GitLab Web UI에서 발급받는다.
- Token은 문서, README, Git commit에 남기지 않는다.

---

## 테스트 Pipeline 기준

Phase 4에서는 Docker image build나 deploy job을 만들지 않는다.

검증 목적은 아래에 한정한다.

```
GitLab Repository
 → .gitlab-ci.yml
 → Pipeline 생성
 → GitLab Runner가 Job 수신
 → alpine 기반 test job 실행
 → Job Passed 확인
```

테스트 Job 기준:

- `alpine:latest` 이미지 사용
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