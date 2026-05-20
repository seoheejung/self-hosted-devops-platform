# Self-Hosted DevOps Platform

> Windows 11 기반 Mini PC 환경에서 WSL2와 Docker Desktop을 활용하여 GitLab 기반 Self-Hosted DevOps 인프라를 단계적으로 구축하고, CI/CD 실행 흐름과 운영 안정성을 검증하는 프로젝트

- GitHub Actions self-hosted runner 기반 GitLab 초기 배포 자동화
- GitLab + Runner 기반 CI/CD 실행 환경 구성
- Docker Executor 기반 Pipeline 실행 구조 구성
- Docker Compose 기반 배포 검증 흐름 구성
- Nginx Reverse Proxy 및 내부망 HTTPS 접근 구조 구성
- Prometheus/Grafana 기반 모니터링 구성
- 제한된 리소스 환경에서 GitLab 운영 가능성 평가
- Self-hosted GitOps 운영 경험 확보

---

## 핵심 목표

### 1. Self-Hosted DevOps 인프라 구축

Windows 11 기반 Mini PC에서 WSL2, Docker Desktop, GitLab, GitLab Runner를 활용해 자체 DevOps 실행 환경을 구성한다.

### 2. CI/CD 실행 흐름 구성

GitHub Actions self-hosted runner로 GitLab 서버를 초기 배포하고, GitLab 구축 이후에는 GitLab Runner 기반 Pipeline 실행 구조로 전환한다.

### 3. 제한된 리소스 환경 검증

Intel N100 기반 Mini PC에서 GitLab Omnibus, GitLab Runner, Docker Executor를 실행하고 리소스 제약 안에서 운영 가능성을 확인한다.

### 4. 운영 확장 기반 확보

Reverse Proxy, HTTPS, Monitoring, Backup, 장애 대응으로 확장 가능한 Self-Hosted DevOps 운영 기반을 만든다.

---

## 프로젝트 환경

### 하드웨어

| 항목 | 내용 |
|---|---|
| Device | EcoBe-A1 Mini PC |
| CPU | Intel N100 |
| Core / Thread | 4C / 4T |
| Memory | 16GB |
| 특징 | 저전력 (TDP 6W) |

---

### 기술 스택

| 영역 | 기술 |
|---|---|
| Host OS | Windows 11 |
| Linux Runtime | WSL2 Ubuntu |
| Container Runtime | Docker Desktop |
| SCM | GitLab |
| CI/CD | GitLab Runner |
| Reverse Proxy | Nginx |
| Monitoring | Prometheus + Grafana |

---

## 아키텍처

```text
[ Main PC ]
 ├─ Development
 ├─ Git Push
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
 │   ├─ Docker Compose 실행
 │   ├─ GitLab runtime volume
 │   │   ├─ /home/gali/gitlab/config
 │   │   ├─ /home/gali/gitlab/data
 │   │   └─ /home/gali/gitlab/logs
 │   └─ Nginx runtime files
 │       ├─ /home/gali/nginx/nginx.conf
 │       ├─ /home/gali/nginx/conf.d/gitlab.conf
 │       └─ /home/gali/nginx/ssl
 │
 ├─ GitLab Omnibus Container
 ├─ GitLab Runner Container
 ├─ Nginx Reverse Proxy Container
 ├─ Docker Executor
 └─ GitLab 운영 안정성 검증

[ Extended DevOps Platform ]
 ├─ Prometheus
 ├─ Grafana
 ├─ Backup / Restore
 └─ Deployment Target
```

### 배포 흐름
```
Main PC
 → GitHub Repository
 → GitHub Actions
 → Mini PC self-hosted runner
 → Nginx runtime 파일 준비
 → Docker Compose
 → GitLab / Nginx Container
```
- GitHub-hosted runner는 내부망 사설 IP의 Mini PC에 직접 접근할 수 없으므로 사용하지 않는다.
- Mini PC WSL2 Ubuntu에 GitHub self-hosted runner를 등록하고, 해당 runner가 GitHub Actions job을 받아 Docker Compose를 실행한다.
- GitHub Actions는 Repository의 Nginx 설정 파일을 Mini PC runtime 경로 `/home/gali/nginx`로 복사한 뒤 Docker Compose를 실행한다.
- GitLab runtime data는 `/home/gali/gitlab`, Nginx runtime files는 `/home/gali/nginx` 아래에 유지한다.

---

## 디렉토리 구조
```
self-hosted-devops-platform
│
├─ docs/
│  ├─ screenshots/
│  ├─ architecture.md
│  ├─ installation.md
│  ├─ github-actions-deploy.md
│  ├─ gitlab-ci-pipeline.md
│  ├─ gitlab-runner.md
│  ├─ monitoring.md
│  ├─ backup-strategy.md
│  ├─ troubleshooting.md
│  ├─ reverse-proxy-ssl.md
│  ├─ operations-stability.md
│  └─ optimization.md
│
├─ infra/
│  ├─ docker/
│  │  ├─ gitlab/
│  │  ├─ nginx/
│  │  ├─ prometheus/
│  │  └─ grafana/
│  │
│  ├─ compose/
│  │  ├─ docker-compose.gitlab.yml
│  │  ├─ docker-compose.monitoring.yml
│  │  └─ docker-compose.full.yml
│  │
│  └─ scripts/
│     ├─ backup.sh
│     ├─ restore.sh
│     └─ health-check.sh
│
├─ monitoring/
│  ├─ prometheus/
│  │  └─ prometheus.yml
│  │
│  ├─ grafana/
│  │  ├─ dashboards/
│  │  └─ provisioning/
│  │
│  └─ exporters/
│
├─ nginx/
│  ├─ conf.d/
│  │  └─ gitlab.conf
│  ├─ ssl/
│  │  └─ .gitkeep
│  └─ nginx.conf
│
├─ gitlab/
│  ├─ config/
│  ├─ data/
│  └─ logs/
│
├─ .github/
│  ├─ workflows/
│  │  └─ deploy-gitlab.yml
│  └─ pull_request_template.md
│
├─ .gitlab-ci.yml
├─ .gitignore
└─ README.md
```

### GitLab runtime 데이터 경로

- Repository 안의 `gitlab/` 디렉토리는 구조 표시용 placeholder로 유지한다.
- 실제 GitLab 운영 데이터는 GitHub Actions workspace가 아니라 Mini PC WSL2 Ubuntu 내부 고정 경로에 저장한다.
| 경로                         | 역할                              |
| -------------------------- | ------------------------------- |
| `/home/<USER>/gitlab/config` | GitLab 설정 파일                    |
| `/home/<USER>/gitlab/data`   | Repository, DB, 업로드 파일 등 실제 데이터 |
| `/home/<USER>/gitlab/logs`   | GitLab 로그                       |

### Nginx runtime 파일 경로

- Repository 안의 `nginx/` 디렉토리는 설정 원본이다.
- 실제 Nginx Container에는 Mini PC WSL2 내부 고정 runtime 경로를 mount한다.

| 경로 | 역할 |
| --- | --- |
| `nginx/nginx.conf` | GitHub Repository의 Nginx 전역 설정 원본 |
| `nginx/conf.d/gitlab.conf` | GitHub Repository의 GitLab Reverse Proxy 설정 원본 |
| `/home/gali/nginx/nginx.conf` | Nginx Container mount 대상 |
| `/home/gali/nginx/conf.d/gitlab.conf` | Nginx Container mount 대상 |
| `/home/gali/nginx/ssl/gitlab.local.crt` | HTTPS 인증서 |
| `/home/gali/nginx/ssl/gitlab.local.key` | HTTPS private key |

`nginx/ssl/*.crt`, `nginx/ssl/*.key`, `nginx/ssl/*.pem` 파일은 Git에 commit하지 않는다.

---

## 변경 관리 프로세스 (Git Workflow)

### PR 사용 목적
- 인프라 변경으로 인한 장애 리스크 통제 목적
- 협업 도구가 아닌 변경 검증 게이트로 사용
- 코드 리뷰가 아닌 self-review 강제 수단
- GitLab, Runner, Nginx, Monitoring 설정 변경은 전체 DevOps 플랫폼에 영향을 주므로 변경 단위 분리 및 사전 검증 필요

### 작업 흐름

```bash
git checkout -b feature/xxx
git add .
git commit -m "feat: xxx"
git push origin feature/xxx
```
1. 기능 단위 브랜치 생성 후 작업 수행
2. 원격 저장소로 push
3. PR 생성 (`Compare & pull request`)
4. 체크리스트 기반 검증
5. 검증 완료 후 main 브랜치로 merge
6. CI/CD를 통한 배포 진행

### 검증 방식
- 체크리스트 기반 self-review
- 모든 PR 동일 기준 적용
- `.github/pull_request_template.md`를 통해 자동 적용

### 브랜치 전략
```
feature/xxx
```
- 기능 단위로 브랜치 분리
- main 브랜치는 항상 배포 가능한 상태 유지
- main 브랜치 직접 커밋 금지

---

## 문서

| Phase | 문서 | 내용 |
|---|---|---|
| - | [architecture.md](docs/architecture.md) | Main PC, GitHub Actions, Mini PC self-hosted runner, GitLab Container로 구성된 전체 아키텍처 설명 |
| Phase 1~3 | [installation.md](docs/installation.md) | Windows 11, WSL2, Docker Desktop 기반 GitLab 실행 환경 준비 절차 |
| Phase 2 | [github-actions-deploy.md](docs/github-actions-deploy.md) | GitHub self-hosted runner를 통한 Mini PC 내부 Docker Compose 배포 자동화 구성 |
| Phase 4 | [gitlab-runner.md](docs/gitlab-runner.md) | Docker Executor 기반 GitLab Runner 등록, 테스트 Pipeline 실행, Runner 운영 기준 |
| Phase 5 | [gitlab-ci-pipeline.md](docs/gitlab-ci-pipeline.md) | GitLab Repository 기준 validate-compose / deploy-readiness-check Pipeline 검증 |
| Phase 6 | [operations-stability.md](docs/operations-stability.md) | 장기 종료 후 재기동, GitLab DB 유지, Project / Runner 상태 검증 |
| Phase 7 | [reverse-proxy-ssl.md](docs/reverse-proxy-ssl.md) | Nginx Reverse Proxy 및 HTTPS 접근 구성 |
| Phase 8 | [monitoring.md](docs/monitoring.md) | Prometheus/Grafana 기반 모니터링 구성 |
| Phase 9 | [backup-strategy.md](docs/backup-strategy.md) | GitLab 데이터 백업 및 복구 전략 |
| 공통 | [troubleshooting.md](docs/troubleshooting.md) | 구축 및 운영 중 발생한 문제와 해결 기록 |
| 공통 | [optimization.md](docs/optimization.md) | GitLab 및 Docker Desktop 성능 튜닝 기록 |

---

## 구현 단계

### Phase 1. Mini PC 기본 환경 구성

#### 작업 내용
- Windows 11 환경 정리
- WSL2 Ubuntu 설치
- Docker Desktop 설치 및 WSL2 연동
- Docker 리소스 제한 설정
- Mini PC 고정 IP 또는 내부 접근 주소 정리
- Main PC에서 Mini PC로 SSH 접속 확인

#### 구현 목표
- Windows 11 + WSL2 기반 Container 환경 구성
- Docker Desktop 리소스 최적화
- Main PC에서 Mini PC 원격 제어 가능 상태 확보

---

### Phase 2. GitHub Actions 기반 초기 배포 구성

#### 작업 내용
- GitHub-hosted runner의 내부망 Mini PC SSH 접근 한계 확인
- Mini PC WSL2 Ubuntu에 GitHub self-hosted runner 등록
- GitHub Actions Workflow 작성
- `runs-on: self-hosted` 기반 배포 실행 구조 구성
- GitHub Secrets를 통한 Mini PC 내부 접근 주소 관리
- Docker credential helper 문제 대응
- GitHub Actions에서 Mini PC WSL2 내부 Docker Compose 실행 검증

#### 구현 목표
- GitHub-hosted runner의 내부망 접근 제약 회피
- Mini PC가 GitHub Actions job을 직접 받아 실행하는 구조 확보
- SSH/SCP 배포 없이 Mini PC 내부에서 Docker Compose 실행
- Git Push 이후 GitLab Container 자동 배포 경로 확보

#### 구조 판단

```text
GitHub-hosted runner
 → Mini PC 내부망 SSH 접속
 → 실패: 사설 IP 접근 불가

GitHub Actions
 → Mini PC self-hosted runner
 → Docker Compose 실행
 → GitLab Container 배포
```

---

### Phase 3. GitLab 서버 구축

#### 작업 내용
- GitLab Omnibus Container 구성
- Docker Compose 작성
- GitLab Volume 절대 경로 구성
- GitHub Actions self-hosted runner를 통한 Docker Compose 실행
- 초기 관리자 계정 설정
- GitLab Web UI 접근 확인

#### 구현 목표
- GitLab 기반 Self-hosted SCM 구축
- Persistent Volume 기반 데이터 유지
- Main PC 브라우저에서 Mini PC GitLab Web UI 접근 가능
- 사용자 및 권한 관리 기반 확보

---

### Phase 4. GitLab Runner 구성

#### 작업 내용
- GitLab Runner 실행 위치 결정
- Mini PC WSL2 Ubuntu에 GitLab Runner config 경로 구성
- GitLab Runner Docker Container 실행
- GitLab Web UI에서 Runner authentication token 발급
- Docker Executor 기반 Runner 등록
- Runner tag 설정
- 테스트용 GitLab Repository 생성
- `.gitlab-ci.yml` 기반 테스트 Pipeline 작성
- 테스트 Pipeline 실행 및 `Passed` 확인
- Mini PC 재시작 후 Runner Container 재기동 확인

#### 구현 목표
- GitLab Repository의 CI/CD Job 실행 기반 확보
- GitLab 서버와 CI Job 실행 책임 분리
- Docker Executor 기반 격리된 Pipeline Job 실행
- GitLab Runner가 GitLab Pipeline Job을 정상 수신하고 실행하는지 검증
- 테스트 Pipeline `Passed` 확인

#### 자동화 범위
- GitHub Actions self-hosted runner는 GitLab 서버 Container 배포/갱신에 사용한다.
- GitLab Runner는 GitLab Repository의 `.gitlab-ci.yml` Pipeline 실행에 사용한다.
- Phase 4에서는 GitHub Actions를 대체하지 않고, GitLab 내부 CI/CD 실행 환경을 추가한다.
- Docker image build 및 실제 deploy job은 Phase 5에서 별도 구성한다.

---

### Phase 5. CI/CD Pipeline 구성

#### Pipeline Flow

```text
GitLab Repository
 → GitLab Pipeline
 → Validate
 → Deploy Check
 → Pipeline Passed
```

### 작업 내용
- `.gitlab-ci.yml` Pipeline 구조 작성
- `validate` / `deploy-check` Stage 구성
- Validate Job 구성
- Deploy Check Job 구성
- Docker socket 사용 여부 확인
- Docker Compose config 검증 Job 구성
- Docker Image Build 필요 여부 판단
- Deploy Job 실행 가능 범위 확인
- Pipeline 실행 결과 검증

### 구현 목표
- GitLab Runner 기반 Pipeline 구조 확보
- Repository 변경 시 Pipeline 자동 실행 확인
- GitLab Runner tag 기반 Job 실행 검증
- CI Job Container 안에서 Docker 명령 실행 가능 여부 확인
- Docker Compose config 검증
- Docker Build 및 실제 Deploy Job 확장 기준 정리
- 실제 배포 명령 실행 전 안전한 검증 Pipeline 확보

### 완료 기준

- `validate-compose` Job `Passed`
- `deploy-readiness-check` Job `Passed`
- 전체 Pipeline `Passed`
- 실제 배포 명령은 수행하지 않음

---

### Phase 6. GitLab 운영 안정성 검증

#### 작업 내용

- 장기 종료 후 GitLab 재기동 검증
- GitLab 프로젝트 유지 여부 확인
- GitLab Runner 등록 정보 유지 여부 확인
- root 계정 상태 유지 여부 확인
- `application_settings`, `users`, `projects`, `ci_runners` 상태 확인
- `gitlab-rails-db-migrate` 로그 확인
- `db:schema:load` 재발 여부 확인
- GitLab Backup 생성 가능 여부 확인
- GitLab Omnibus 장기 운영 가능 여부 판단

#### 검증 결과
- 기존 GitLab 프로젝트가 유지되지 않음
- GitLab Runner 등록 정보가 유지되지 않음
- root 계정이 재생성됨
- `application_settings`가 재생성됨
- `gitlab-rails-db-migrate` 로그에서 `db:schema:load` 실행 확인

---

### Phase 7. Reverse Proxy 및 SSL 구성

#### 작업 내용
- GitLab 앞단에 Nginx Reverse Proxy Container 추가
- `gitlab.local` 기반 내부망 도메인 접근 구성
- HTTP → HTTPS redirect 구성
- 자체 서명 인증서 기반 HTTPS 접근 검증
- Docker Compose에 `gitlab-nginx` service 추가
- GitHub Actions Workflow에 `nginx/**` 변경 감지 추가
- GitHub Actions에서 Nginx runtime 파일 준비 단계 추가
- Mini PC runtime 경로 `/home/gali/nginx` 기준 Nginx 설정/인증서 mount 구조 구성
- 기존 GitLab 직접 접근 경로 `http://172.30.1.67:8080` 복구 경로로 유지
- GitLab Runner 영향 확인 및 Pipeline 실행 검증

#### 구현 목표
- GitLab Web UI를 `https://gitlab.local`로 접근 가능하게 구성
- GitHub Actions 기반으로 GitLab / Nginx Container를 함께 배포
- GitHub Actions workspace가 아니라 Mini PC 고정 runtime 경로를 사용해 Nginx 설정과 인증서를 유지
- 자체 서명 인증서를 사용한 내부망 HTTPS 접근 구조 확보
- 기존 Runner URL과 GitLab `external_url`은 즉시 변경하지 않고 복구 경로 기준으로 유지

#### 검증 내용
- GitHub Actions 실행 완료
- `gitlab-nginx` Container 실행 확인
- Nginx 80/443 포트 publish 확인
- `docker exec gitlab-nginx nginx -t` 성공
- Main PC hosts 설정 확인
- `gitlab.local → 172.30.1.67` 해석 확인
- `curl -I http://gitlab.local` 요청 시 `https://gitlab.local` redirect 확인
- `curl -k -I https://gitlab.local` 접근 확인
- 브라우저에서 `https://gitlab.local` GitLab Web UI 접근 확인
- GitLab Runner 검증용 Pipeline `Passed` 확인

---

### Phase 8. Monitoring 구성

#### 작업 내용
- Prometheus 구성
- Grafana 구성
- Docker Container 리소스 수집
- GitLab 상태 수집
- Dashboard 작성

#### 수집 대상
- Windows Host 리소스
- WSL2 Ubuntu 리소스
- Docker Container 리소스
- GitLab 상태
- GitLab Runner 상태
- Nginx 상태

#### 구현 목표
- Grafana Dashboard 구성
- Resource Monitoring
- Alert 기준 설계

---

### Phase 9. Backup / 운영 문서화

#### 작업 내용
- 설치 절차 문서화
- 백업/복구 절차 문서화
- 장애 대응 절차 문서화
- 성능 튜닝 결과 정리

#### 구현 목표
- 운영 문서 표준화
- 장애 대응 절차 확보
- 성능 최적화 기록 관리

---
