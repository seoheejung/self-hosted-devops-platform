# Architecture

> Main PC, GitHub Actions, Mini PC WSL2 Ubuntu, GitLab Container, GitLab Runner, Nginx Reverse Proxy로 구성된 Self-Hosted DevOps Platform의 전체 구조를 정리한다.

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

### 현재 구성

| Host Port | Container Port | 용도 |
| --- | --- | --- |
| `8080` | `80` | GitLab Web UI 직접 접근 |
| `2222` | `22` | GitLab Repository SSH |

### Phase 6 목표 구성

| Host Port | Container Port | 용도 |
| --- | --- | --- |
| `80` | `80` | Nginx HTTP redirect |
| `443` | `443` | Nginx HTTPS |
| `8080` | `80` | GitLab Web UI 직접 접근 예비 경로 |
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

Phase 5에서는 GitLab Runner 기반 Pipeline 검증을 완료했고, 이후 단계에서 Docker Build와 실제 배포 자동화를 확장한다.

---

## Reverse Proxy 접근 구조

Phase 6에서는 GitLab Container 앞단에 Nginx Reverse Proxy를 추가한다.

```text
Main PC Browser
 → https://gitlab.local
 → Nginx Reverse Proxy
 → GitLab Container
```

기존 `http://172.30.1.69:8080` 접근 경로는 Phase 6 작업 중 복구 경로로 유지한다.

---