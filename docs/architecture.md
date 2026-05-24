# Architecture

> Main PC, GitHub Actions, Mini PC WSL2 Ubuntu, GitLab Container, GitLab Runner, Nginx, Prometheus, Grafana로 구성된 Self-Hosted DevOps Platform의 전체 구조를 정리한다.

---

## 전체 구조

```text
[ Main PC ]
 ├─ Development
 ├─ Git Push / PR Merge
 ├─ GitHub Repository 관리
 ├─ GitLab Web UI 접근
 │   └─ https://gitlab.local
 │
 ├─ Prometheus Web UI 접근
 │   └─ http://172.30.1.67:9090
 │
 └─ Grafana Web UI 접근
     └─ http://172.30.1.67:3000

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
 │   │
 │   ├─ GitLab runtime volume
 │   │   ├─ /home/<USER>/gitlab/config
 │   │   ├─ /home/<USER>/gitlab/data
 │   │   └─ /home/<USER>/gitlab/logs
 │   │
 │   ├─ GitLab Runner config
 │   │   └─ /home/<USER>/gitlab-runner/config
 │   │
 │   ├─ Nginx runtime files
 │   │   ├─ /home/gali/nginx/nginx.conf
 │   │   ├─ /home/gali/nginx/conf.d/gitlab.conf
 │   │   └─ /home/gali/nginx/ssl
 │   │
 │   └─ Monitoring runtime files
 │       ├─ /home/gali/monitoring/prometheus/prometheus.yml
 │       └─ /home/gali/monitoring/grafana
 │
 ├─ GitLab Omnibus Container
 ├─ GitLab Runner Container
 ├─ Nginx Reverse Proxy Container
 ├─ Prometheus Container
 ├─ Grafana Container
 └─ Docker Executor
```

---

## 자동화 흐름

```
[Bootstrap / Infrastructure Deploy]
GitHub Repository
 → GitHub Actions
 → Mini PC self-hosted runner
 → runtime 파일 준비
 → Docker Compose
 → GitLab / Nginx / Prometheus / Grafana Container

[GitLab CI/CD]
GitLab Repository
 → GitLab Pipeline
 → GitLab Runner
 → Build / Test / Deploy Job
```

> GitHub Actions는 GitLab 서버와 인프라 구성 요소를 배포하는 부트스트랩 흐름이고,
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
| `/home/gali/nginx/nginx.conf` | Nginx 전역 설정 파일 |
| `/home/gali/nginx/conf.d/gitlab.conf` | GitLab Reverse Proxy 설정 파일 |
| `/home/gali/nginx/ssl` | 자체 서명 인증서 / private key |
| `/home/gali/monitoring/prometheus/prometheus.yml` | Prometheus 설정 파일 |
| `/home/gali/monitoring/grafana/provisioning` | Grafana datasource / dashboard provider 설정 |
| `/home/gali/monitoring/grafana/dashboards` | Grafana Dashboard JSON |

세부 준비 절차는 `docs/installation.md`, `docs/github-actions-deploy.md`, `docs/reverse-proxy-ssl.md`, `docs/monitoring.md`에서 다룬다.

---

## Port 구성

| Host Port | Container Port | 대상 | 용도 |
| --- | --- | --- | --- |
| `80` | `80` | Nginx | HTTP → HTTPS redirect |
| `443` | `443` | Nginx | GitLab HTTPS 접근 |
| `8080` | `80` | GitLab | GitLab Web UI 직접 접근 예비 경로 |
| `2222` | `22` | GitLab | GitLab Repository SSH |
| `9090` | `9090` | Prometheus | Prometheus Web UI |
| `3000` | `3000` | Grafana | Grafana Web UI |

### 접근 주소

| 대상 | 주소 | 비고 |
| --- | --- | --- |
| GitLab Reverse Proxy | `https://gitlab.local` | Main PC hosts 설정 필요 |
| GitLab 직접 접근 | `http://172.30.1.67:8080` | 복구 경로 |
| Prometheus | `http://172.30.1.67:9090` | 직접 접근 |
| Grafana | `http://172.30.1.67:3000` | 직접 접근 |

---

## Runner 구성 요약

| Runner | 위치 | 역할 |
| --- | --- | --- |
| GitHub self-hosted runner | Mini PC WSL2 Ubuntu | GitHub Actions Job 실행, GitLab / Nginx / Monitoring Container 배포 |
| GitLab Runner | Mini PC WSL2 Ubuntu Docker Container | GitLab Pipeline Job 실행 |

### GitLab Runner 기준

| 항목 | 내용 |
| --- | --- |
| Executor | Docker Executor |
| Runner 이름 | `mini-pc-docker-runner` |
| Tags | `docker`, `mini-pc`, `wsl2` |
| Config 경로 | `/home/<USER>/gitlab-runner/config` |

---

## Reverse Proxy 접근 구조

GitLab Container 앞단에 Nginx Reverse Proxy를 추가했다.

```
Main PC Browser
 → https://gitlab.local
 → Nginx Reverse Proxy
 → GitLab Container
```

기존 `http://172.30.1.67:8080` 접근 경로는 복구 경로로 유지한다.

```
Main PC Browser
 → http://172.30.1.67:8080
 → GitLab Container
```

### Reverse Proxy 기준

| 항목 | 내용 |
| --- | --- |
| 도메인 | `gitlab.local` |
| HTTP | `80` |
| HTTPS | `443` |
| 인증서 | 자체 서명 인증서 |
| Nginx runtime 경로 | `/home/gali/nginx` |
| GitLab 직접 접근 경로 | `http://172.30.1.67:8080` |

---

## Monitoring Stack 구조

Prometheus / Grafana 기반 Monitoring Stack을 구성한다.

Prometheus / Grafana 실행 구조, datasource 연결, dashboard provisioning, GitHub Actions 기반 배포 흐름을 검증하는 것이다.

```
GitHub Actions
 ├─ Monitoring 설정 파일 복사
 ├─ Prometheus Container 실행
 └─ Grafana Container 실행

Prometheus
 ├─ Prometheus 자체 Metrics 수집
 └─ Target 상태 확인

Grafana
 ├─ Prometheus Datasource 연결
 ├─ Prometheus 상태 Dashboard 구성
 └─ Dashboard JSON export / Repository 반영
```

---

## 운영 안정성 검증 위치

GitLab Container가 실행 중이라고 해서 기존 GitLab 인스턴스가 유지됐다고 판단하지 않는다.

다음 항목을 기준으로 GitLab 운영 안정성을 확인했다.

| 항목 | 확인 내용 |
| --- | --- |
| GitLab DB | `application_settings`, `users` 생성 시각 확인 |
| Project | 기존 GitLab Project 유지 여부 확인 |
| Runner | GitLab Runner 등록 정보 유지 여부 확인 |
| DB migration log | `db:schema:load` 재발 여부 확인 |
| Backup | GitLab Backup 생성 가능 여부 확인 |

검증 결과, 장기 미가동 후 재기동 시 GitLab DB 재초기화 문제가 확인됐다.

다만 Reverse Proxy / SSL 구성과 Monitoring 구성은 GitLab 장기 운영 확정이 아니라, Mini PC 기반 DevOps 구성 요소의 부트스트랩과 연동 구조 검증 목적으로 진행한다.

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

1. GitLab Runner 기반 Pipeline 검증을 완료했다.
2. GitLab 운영 안정성을 검증했고, 장기 미가동 후 재기동 시 DB 재초기화 문제가 확인됐다.
3. Nginx Reverse Proxy와 내부망 HTTPS 접근 구조를 구성했다.
4. Prometheus / Grafana 기본 Monitoring Stack을 구성하고, Prometheus 자체 상태 Dashboard를 작성한다.

이후 확장 시 Docker Container Metrics, GitLab 상태 Metrics, Runner 상태, Nginx 접근 상태 probe, Backup / Restore 절차를 별도 단계로 검토한다.