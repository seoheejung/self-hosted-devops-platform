# Self-Hosted DevOps Platform

> Windows 11 기반 Mini PC 환경에서 WSL2와 Docker Desktop을 활용하여 GitLab 기반 Self-Hosted DevOps 플랫폼을 구축하고 운영하는 프로젝트

- GitLab + Runner 기반 CI/CD 환경 구성
- Docker 기반 서비스 배포 자동화
- Reverse Proxy 및 HTTPS 구성
- Prometheus/Grafana 기반 모니터링 구축
- 제한된 리소스 환경에서 GitLab 성능 최적화
- Self-hosted GitOps 운영 경험 확보

---

## 핵심 목표

### 1. Self-Hosted DevOps 플랫폼 구축

외부 SaaS 의존 없이 GitLab 기반 자체 DevOps 플랫폼 구축

### 2. GitOps 기반 운영 흐름 학습

Git Push를 기준으로 자동 빌드·배포가 수행되는 GitOps 흐름 구성

### 3. 제한된 리소스 환경 운영 경험 확보

Intel N100 기반 저전력 Mini PC 환경에서 성능 최적화 및 안정성 확보

### 4. 실제 운영 중심 경험 확보

단순 설치가 아닌:

- 장애 대응
- 성능 튜닝
- 백업
- 모니터링
- 로그 분석
- 배포 자동화

까지 포함한 운영 경험 확보

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

                Git Push
                     ↓

[ GitHub ]
 ├─ Repository
 ├─ GitHub Actions
 └─ Secrets

                SSH Deploy
                     ↓

[ Mini PC - Windows 11 ]
 ├─ OpenSSH Server
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 │
 ├─ GitLab Omnibus Container
 ├─ Git Repository
 ├─ Container Registry
 ├─ Nginx
 ├─ Prometheus
 ├─ Grafana
 └─ Deployment Target

                이후 전환
                     ↓

[ Main PC ]
 └─ GitLab Runner
```

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
│  ├─ gitlab-runner.md
│  ├─ monitoring.md
│  ├─ backup-strategy.md
│  ├─ troubleshooting.md
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
│  ├─ ssl/
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
├─ .gitignore
└─ README.md
```

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

| 문서 | 내용 |
|---|---|
| [architecture.md](docs/architecture.md) | Main PC, GitHub Actions, Mini PC, GitLab Runner로 구성된 전체 아키텍처 설명 |
| [installation.md](docs/installation.md) | Windows 11, WSL2, Docker Desktop, GitLab Container 설치 절차 |
| [github-actions-deploy.md](docs/github-actions-deploy.md) | GitHub Actions를 통한 Mini PC 초기 배포 자동화 구성 |
| [gitlab-runner.md](docs/gitlab-runner.md) | GitLab 구축 이후 Main PC 기반 GitLab Runner 등록 및 운영 방식 |
| [monitoring.md](docs/monitoring.md) | Prometheus/Grafana 기반 모니터링 구성 |
| [backup-strategy.md](docs/backup-strategy.md) | GitLab 데이터 백업 및 복구 전략 |
| [troubleshooting.md](docs/troubleshooting.md) | 구축 및 운영 중 발생한 문제와 해결 기록 |
| [optimization.md](docs/optimization.md) | GitLab 및 Docker Desktop 성능 튜닝 기록 |

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
- Mini PC OpenSSH Server 활성화
- GitHub Actions용 SSH Key 생성
- GitHub Repository Secrets 등록
- GitHub Actions 배포 Workflow 작성
- GitHub Actions에서 Mini PC SSH 접속 검증

#### 구현 목표
- Git Push 이후 Mini PC에 자동 적용되는 초기 배포 경로 확보
- GitLab 구축 전 단계에서 GitHub Actions 기반 자동화 구성
- 수동 복사/수동 실행 없이 원격 배포 가능한 구조 확보

---

### Phase 3. GitLab 서버 구축

#### 작업 내용
- GitLab Omnibus Container 구성
- Docker Compose 작성
- GitLab Volume 구성
- GitHub Actions를 통한 Docker Compose 실행
- 초기 관리자 계정 설정
- GitLab Web UI 접근 확인

#### 구현 목표
- GitLab 기반 Self-hosted SCM 구축
- Persistent Volume 기반 데이터 유지
- Main PC 브라우저에서 Mini PC GitLab Web UI 접근 가능
- 사용자 및 권한 관리 기반 확보

---

### Phase 4. GitLab Runner 분리 구성

#### 작업 내용
- Main PC에 GitLab Runner 설치
- GitLab Runner 등록
- Docker Executor 설정
- Runner Tag 설정
- 테스트 Pipeline 실행

#### 구현 목표
- Main PC 기반 Runner 분리 구성
- Docker Executor 기반 Pipeline 실행
- CI Job 분산 처리

---

### Phase 5. CI/CD Pipeline 구성

#### Pipeline Flow

```text
Git Push
 → GitLab Pipeline
 → Build
 → Test
 → Docker Build
 → Mini PC Deploy
```

#### 작업 내용
- Build Job 구성
- Test Job 구성
- Docker Image Build 구성
- Mini PC 배포 Job 구성
- 배포 결과 검증

#### 구현 목표
- 자동 Build
- 테스트 자동화
- Docker Image Build
- 자동 Deploy
- Deploy 결과 검증

---

### Phase 6. Reverse Proxy 및 SSL 구성

#### 작업 내용
- Nginx Reverse Proxy 구성
- GitLab 접근 도메인 또는 로컬 DNS 구성
- HTTPS 적용
- 인증서 갱신 방식 정리

#### 구현 목표
- HTTPS 기반 접근 구성
- Reverse Proxy 기반 서비스 운영

---

### Phase 7. Monitoring 구성

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

### Phase 8. 운영 문서화

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
