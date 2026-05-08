# Architecture

> Main PC, GitHub Actions, Mini PC WSL2 Ubuntu, GitLab Container, GitLab Runner로 구성된 Self-Hosted DevOps Platform의 전체 구조를 정리한다.

---

## 전체 구조

```text
[ Main PC ]
 ├─ Development
 ├─ Git Push / PR Merge
 └─ GitHub Repository 관리

                Git Push / PR Merge
                     ↓

[ GitHub ]
 ├─ Repository
 ├─ GitHub Actions
 └─ Secrets
     └─ MINI_PC_HOST

                Job Dispatch
                     ↓

[ Mini PC - Windows 11 ]
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 │   ├─ GitHub self-hosted runner
 │   │   └─ GitHub Actions Job 실행
 │   │
 │   ├─ GitLab Runner Container
 │   │   └─ GitLab Pipeline Job 실행
 │   │
 │   ├─ Docker Compose 실행
 │   ├─ GitLab runtime volume
 │   │   ├─ /home/<USER>/gitlab/config
 │   │   ├─ /home/<USER>/gitlab/data
 │   │   └─ /home/<USER>/gitlab/logs
 │   │
 │   └─ GitLab Runner config
 │       └─ /home/<USER>/gitlab-runner/config
 │
 ├─ GitLab Omnibus Container
 ├─ GitLab Runner Container
 ├─ Container Registry
 ├─ Nginx
 ├─ Prometheus
 ├─ Grafana
 └─ Deployment Target
```

---

## 자동화 흐름

```
[Bootstrap Deploy]
GitHub Repository
 → GitHub Actions
 → Mini PC self-hosted runner
 → Docker Compose
 → GitLab Container

[GitLab CI/CD]
GitLab Repository
 → GitLab Pipeline
 → GitLab Runner
 → Build / Test / Deploy Job
```

> GitHub Actions는 GitLab 서버를 배포하는 초기 부트스트랩 흐름이고, 
> GitLab Runner는 GitLab Repository의 Pipeline Job을 실행하는 흐름이다.

---

## Repository 역할

| Repository | 역할 |
| --- | --- |
| GitHub Repository | GitLab 서버 배포 소스 관리 |
| Mini PC GitLab Repository | GitLab Runner Pipeline 실행 대상 |

---

## Runtime 경로 요약

| 경로 | 역할 |
| --- | --- |
| `/home/<USER>/gitlab/config` | GitLab 설정 파일 |
| `/home/<USER>/gitlab/data` | Repository, DB, 업로드 파일 등 GitLab 데이터 |
| `/home/<USER>/gitlab/logs` | GitLab 로그 |
| `/home/<USER>/gitlab-runner/config` | GitLab Runner 설정 파일 |

세부 준비 절차는 `docs/installation.md`와 `docs/github-actions-deploy.md`에서 다룬다.

---

## Port 구성

| Host Port | Container Port | 용도 |
| --- | --- | --- |
| `8080` | `80` | GitLab Web UI HTTP |
| `8443` | `443` | GitLab HTTPS 예비 포트 |
| `2222` | `22` | GitLab Repository SSH |

### 현재 GitLab Web UI 접근 주소
```
http://172.30.1.69:8080
```

---

## Runner 구성 요약

| Runner | 위치 | 역할 |
| --- | --- | --- |
| GitHub self-hosted runner | Mini PC WSL2 Ubuntu | GitHub Actions Job 실행, GitLab Container 배포 |
| GitLab Runner | Mini PC WSL2 Ubuntu Docker Container | GitLab Pipeline Job 실행 |

### GitLab Runner 기준
| 항목 | 내용 |
| --- | --- |
| Executor | Docker Executor |
| Runner 이름 | `mini-pc-docker-runner` |
| Tags | `docker`, `mini-pc`, `wsl2` |
| Config 경로 | `/home/<USER>/gitlab-runner/config` |

---

## 향후 확장 구조

```
GitLab Repository
 └─ Git Push / Merge
        ↓

GitLab Pipeline
 ├─ Validate
 ├─ Test
 ├─ Docker Build
 └─ Deploy
        ↓

Mini PC Services
 ├─ GitLab
 ├─ Nginx
 ├─ Monitoring
 └─ Application Services
```

Phase 5 이후에는 GitLab Runner 기반으로 Build, Test, Docker Build, Deploy 흐름을 단계적으로 구성한다.

---
