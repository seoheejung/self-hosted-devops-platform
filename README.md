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

### Hardware

| 항목 | 내용 |
|---|---|
| Device | EcoBe-A1 Mini PC |
| CPU | Intel N100 |
| Core / Thread | 4C / 4T |
| Memory | 16GB |
| 특징 | 저전력 (TDP 6W) |

---

### Software Stack

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

## Architecture

```text
[ Main PC ]
 ├─ Development
 ├─ GitLab Runner
 ├─ Docker Build
 ├─ Test
 └─ CI Pipeline Execution

                Git Push
                     ↓

[ Mini PC - Windows 11 ]
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 │
 ├─ GitLab Omnibus
 ├─ Git Repository
 ├─ Container Registry
 ├─ Nginx
 ├─ Prometheus
 ├─ Grafana
 └─ Deployment Target
```

---

## Repository Structure
```
self-hosted-devops-platform
│
├─ docs/
│  ├─ screenshots/
│  ├─ architecture.md
│  ├─ installation.md
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
├─ .gitignore
└─ README.md
```

---

## Roadmap

### Phase 1. Mini PC 기본 환경 구성

- Windows 11 환경 정리
- WSL2 Ubuntu 설치
- Docker Desktop 설치 및 WSL2 연동
- Docker 리소스 제한 설정
- 고정 IP 또는 내부 접근 주소 정리

### Phase 2. GitLab 서버 구축

- GitLab Omnibus Container 구성
- Docker Compose 작성
- GitLab Volume 구성
- 초기 관리자 계정 설정
- Git Repository 생성

### Phase 3. GitLab Runner 분리 구성

- Main PC에 GitLab Runner 설치
- GitLab Runner 등록
- Docker Executor 설정
- Runner Tag 설정
- 테스트 Pipeline 실행

### Phase 4. CI/CD Pipeline 구성

- Build Job 구성
- Test Job 구성
- Docker Image Build 구성
- Mini PC 배포 Job 구성
- 배포 결과 검증

### Phase 5. Reverse Proxy 및 SSL 구성

- Nginx Reverse Proxy 구성
- GitLab 접근 도메인 또는 로컬 DNS 구성
- HTTPS 적용
- 인증서 갱신 방식 정리

### Phase 6. Monitoring 구성

- Prometheus 구성
- Grafana 구성
- Docker Container 리소스 수집
- GitLab 상태 수집
- Dashboard 작성

### Phase 7. 운영 문서화

- 설치 절차 문서화
- 백업/복구 절차 문서화
- 장애 대응 절차 문서화
- 성능 튜닝 결과 정리

---

## 구현 기능

### GitLab 구축
- GitLab Omnibus Container 구성
- Docker Compose 기반 운영
- Git Repository 관리
- 사용자 및 권한 관리
- Persistent Volume 구성

### GitLab Runner 구축

- Main PC 기반 Runner 분리 구성
- Docker Executor 기반 Pipeline 실행
- Runner Tag 분리
- CI Job 분산 처리

### CI/CD Pipeline 구성

```
Git Push
 → Build
 → Test
 → Docker Build
 → Deploy
```

#### 구현 항목

- 자동 Build
- 테스트 자동화
- Docker Image Build
- 자동 Deploy
- Deploy 결과 검증

### Monitoring

#### 수집 대상

- Windows Host 리소스
- WSL2 Ubuntu 리소스
- Docker Container 리소스
- GitLab 상태
- GitLab Runner 상태
- Nginx 상태

#### 시각화

- Grafana Dashboard 구성
- Resource Monitoring
- Alert 기준 설계 예정

---

