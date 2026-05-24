# Monitoring 구성

> Mini PC 기반 GitLab 운영 환경에서 Prometheus와 Grafana를 활용하여 리소스 상태와 주요 서비스 상태를 관측한다.

---

## 목적

Mini PC 기반 Self-Hosted DevOps 환경의 Monitoring 구성을 진행한다.

Monitoring의 목적은 GitLab 운영 중 문제가 발생했을 때 확인해야 할 리소스 상태, 서비스 상태, 접근 상태, 로그 확인 기준을 정리하는 것이다.

현재 구성은 Windows 11 Host 위에서 WSL2 Ubuntu와 Docker Desktop을 사용하고, 그 위에 GitLab, GitLab Runner, Nginx Reverse Proxy가 Container로 실행되는 구조다.

```text
Mini PC - Windows 11
 ├─ WSL2 Ubuntu
 │   ├─ Docker Compose 실행
 │   ├─ GitLab runtime volume
 │   └─ Nginx runtime files
 │
 ├─ GitLab Container
 ├─ GitLab Runner Container
 ├─ Nginx Reverse Proxy Container
 └─ Monitoring
     ├─ Prometheus
     └─ Grafana
```

---

## 구성 기준

| 항목            | 내용                                            |
| ------------- | --------------------------------------------- |
| Metrics 수집    | Prometheus                                    |
| 시각화           | Grafana                                       |
| 실행 방식         | Docker Compose                                |
| Compose 파일    | `infra/compose/docker-compose.monitoring.yml` |
| Prometheus 설정 | `infra/monitoring/prometheus/prometheus.yml`  |
| Grafana 설정    | `infra/monitoring/grafana/`                   |
| Prometheus 접근 | `http://172.30.1.67:9090`                     |
| Grafana 접근    | `http://172.30.1.67:3000`                     |
| Alert         | 구성 필요 여부 검토                                   |


---

## Monitoring 대상

| 대상                   | 확인 목적                                   |
| -------------------- | --------------------------------------- |
| Windows Host 리소스     | Mini PC 전체 운영 상태 확인                     |
| WSL2 Ubuntu 리소스      | GitLab 실행 환경의 CPU, Memory, Disk 상태 확인   |
| Docker Container 리소스 | GitLab / Runner / Nginx Container 상태 확인 |
| GitLab 상태            | Web UI, Project, Pipeline 상태 확인         |
| GitLab Runner 상태     | Runner Online 여부와 Job 실행 가능 여부 확인       |
| Nginx 상태             | Reverse Proxy 접근 경로 유지 여부 확인            |


---

## 전체 흐름

```
Prometheus
 ├─ Metrics 수집
 ├─ Resource 상태 수집
 ├─ Docker Container 상태 수집
 └─ GitLab 상태 수집

Grafana
 ├─ Prometheus Datasource 연결
 ├─ Resource Monitoring Dashboard 구성
 ├─ GitLab 상태 Dashboard 구성
 └─ Alert 구성 필요 여부 검토
```

---

## 디렉토리 구조

```
self-hosted-devops-platform
├─ infra/
│  ├─ compose/
│  │  ├─ docker-compose.gitlab.yml
│  │  ├─ docker-compose.monitoring.yml
│  │  └─ docker-compose.full.yml
│  │
│  └─ monitoring/
│     ├─ prometheus/
│     │  └─ prometheus.yml
│     │
│     └─ grafana/
│        ├─ provisioning/
│        │  ├─ datasources/
│        │  │  └─ prometheus.yml
│        │  └─ dashboards/
│        │     └─ dashboards.yml
│        │
│        └─ dashboards/
│           └─ mini-pc-devops-overview.json
```

### 경로 기준

| 경로                  | 역할           |
| ------------------------- | --------------------------------- |
| `infra/compose/docker-compose.monitoring.yml` | Monitoring Stack Compose 파일|
| `infra/monitoring/prometheus/prometheus.yml` | Prometheus 설정|
| `infra/monitoring/grafana/provisioning/datasources/prometheus.yml` | Grafana Prometheus datasource 설정  |
| `infra/monitoring/grafana/provisioning/dashboards/dashboards.yml`  | Grafana dashboard provisioning 설정 |
| `infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json` | Grafana Dashboard JSON |

---

## Prometheus 구성 기준

> Prometheus는 Monitoring Stack의 Metrics 수집 서버로 사용

### 수집 대상

| 대상                              | 목적                                      |
| ------------------------------- | --------------------------------------- |
| Prometheus 자체 상태                | Metrics 수집 서버 상태 확인                     |
| Windows Host 또는 WSL2 Ubuntu 리소스 | Mini PC 실행 환경의 리소스 상태 확인                |
| Docker Container 리소스            | GitLab / Runner / Nginx Container 상태 확인 |
| GitLab 상태                       | GitLab Web UI, Project, Pipeline 상태 확인  |
| GitLab Runner 상태                | Runner Online 여부와 Job 실행 가능 여부 확인       |
| Nginx 상태                        | Reverse Proxy 접근 경로 유지 여부 확인            |


### 판단 기준

- Prometheus는 Phase 8의 Metrics 수집 기준 도구로 사용한다.
- 수집 방식은 대상별로 구현 시점에 결정한다.
- Docker Container 리소스는 GitLab / Runner / Nginx 운영 상태 확인에 사용한다.
- GitLab 상태는 Metrics만으로 판단하지 않는다.
- GitLab 상태는 Web UI, Project 유지 여부, Runner 등록 정보, Pipeline 실행 가능 여부를 함께 확인한다.

---

## Grafana 구성 기준

> Grafana는 Prometheus에 수집된 Metrics를 시각화하는 용도로 사용

### Datasource 기준

| 항목      | 값                        |
| ------- | ------------------------ |
| Type    | Prometheus               |
| URL     | `http://prometheus:9090` |
| Access  | proxy                    |
| Default | true                     |


Grafana datasource는 provisioning 파일로 관리한다.

```
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### Dashboard provisioning 기준

Grafana dashboard는 파일 기반 provisioning으로 관리한다.

```
apiVersion: 1

providers:
  - name: Mini PC DevOps
    orgId: 1
    folder: Mini PC DevOps
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

Dashboard는 Grafana UI에서 작성한 뒤 JSON으로 export하여 Repository에 저장한다.

```
infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json
```

---

## Resource Monitoring 기준

Resource Monitoring은 Mini PC 운영 상태를 확인하기 위한 Dashboard 영역이다.

### 확인 대상

- Windows Host 리소스
- WSL2 Ubuntu 리소스
- Docker Container 리소스

### 확인 항목

- CPU 사용량
- Memory 사용량
- Disk 사용량
- Network 상태
- Container 실행 상태
- Container별 리소스 사용 추이
- GitLab runtime volume 사용량

### 판단 기준

- Resource Monitoring은 Prometheus에서 수집 가능한 Metrics를 기준으로 구성한다.
- Windows Host 리소스와 WSL2 Ubuntu 리소스는 동일하게 해석하지 않는다.
- Docker Container 리소스는 GitLab / Runner / Nginx 상태 확인에 사용한다.
- 구체적인 수집 방식은 실제 Compose 구성 시 결정한다.

---

## Docker Container 리소스 확인 기준

Docker Container 리소스는 GitLab, GitLab Runner, Nginx가 Container 기반으로 동작하기 때문에 Monitoring 대상에 포함한다.

### 확인 대상

- GitLab Container
- GitLab Runner Container
- Nginx Container
- Monitoring 관련 Container

### 확인 항목

- Container 실행 상태
- CPU 사용량
- Memory 사용량
- Restart 여부
- Log 확인 경로
- Container별 리소스 사용 추이

---

## GitLab 상태 확인 기준

> GitLab Container가 실행 중이어도 내부 DB 상태나 프로젝트 유지 상태가 정상이라고 단정할 수 없기 때문에, GitLab 상태는 Container 실행 여부와 별도로 확인

### 확인 항목

- GitLab Web UI 접근 가능 여부
- root 계정 로그인 가능 여부
- 기존 Project 유지 여부
- Runner 등록 정보 유지 여부
- Pipeline 실행 가능 여부
- GitLab 주요 로그 이상 여부
- GitLab 상태 수집 가능 여부

### 수동 확인 기준

```
docker ps --filter "name=gitlab"
docker exec -it gitlab gitlab-ctl status
docker logs --tail=100 gitlab
```

### GitLab Rails DB 상태 확인이 필요한 경우
```
docker exec -it gitlab gitlab-rails runner "puts \"projects=#{Project.count}\""
docker exec -it gitlab gitlab-rails runner "puts \"runners=#{Ci::Runner.count}\""
```

### 주의
- `gitlab-rails runner`는 수동 확인용이다.
- GitHub Actions Workflow 검증에는 메모리 부담이 큰 명령을 넣지 않는다.
- GitLab 상태는 Metrics 수집 결과만으로 판단하지 않는다.
- Project / Runner / Pipeline 상태를 함께 확인한다.

---

## GitLab Runner 상태 확인 기준

> Runner 상태는 GitLab UI와 Container 상태를 함께 확인

### 확인 항목
- GitLab Runner Container 실행 여부
- GitLab UI에서 Runner Online 여부
- Runner Tag 유지 여부
- Docker Executor 동작 여부
- 테스트 Pipeline 실행 가능 여부

### 확인 기준

```
docker ps --filter "name=gitlab-runner"
docker logs --tail=100 gitlab-runner
docker exec -it gitlab-runner gitlab-runner verify
```

#### GitLab Web UI 확인 경로
```
Admin Area
 → CI/CD
 → Runners
```

#### 정상 기준
- Runner 상태 `Online`
- Runner tag 유지
- Pipeline Job 수신 가능
- 테스트 Pipeline `Passed`

---

## Nginx 상태 확인 기준

> Reverse Proxy 접근 경로가 유지되는지 확인

### 확인 항목
- Nginx Container 실행 여부
- Nginx 설정 검증 가능 여부
- HTTP → HTTPS redirect 유지 여부
- `https://gitlab.local` 접근 가능 여부
- GitLab 직접 접근 경로 유지 여부

### 확인 기준
```
docker ps --filter "name=gitlab-nginx"
docker exec gitlab-nginx nginx -t
curl -I http://gitlab.local
curl -k -I https://gitlab.local
curl -I http://172.30.1.67:8080
```

#### 정상 기준
- `gitlab-nginx` Container 실행 중
- `nginx -t` 성공
- `http://gitlab.local` 요청 시 HTTPS redirect
- `https://gitlab.local` 접근 가능
- `http://172.30.1.67:8080` 직접 접근 유지

---

## Dashboard 구성 기준

> Dashboard는 운영 상태 확인에 필요한 패널 중심으로 구성

| Dashboard 영역        | 목적                                      |
| ------------------- | --------------------------------------- |
| Resource Monitoring | Mini PC / WSL2 / Container 리소스 상태 확인    |
| Docker Container 상태 | GitLab / Runner / Nginx Container 상태 확인 |
| GitLab 상태           | Web UI / Project / Pipeline 상태 확인 기준 연결 |
| Runner 상태           | Runner Online / Pipeline 실행 여부 확인 기준 연결 |
| Nginx 상태            | Reverse Proxy 접근 경로 확인 기준 연결            |
| Alert 기준            | 장애 판단 기준 정리                             |


Dashboard는 Grafana UI에서 작성한 뒤 JSON으로 export하여 Repository에 저장한다.
```
infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json
```

---

## Alert 판단 기준

Alert는 실제 연동보다 어떤 상태를 장애로 볼지 기준을 먼저 정리한다.

| 장애 후보               | 판단 기준                              | 확인 대상                    |
| ------------------- | ---------------------------------- | ------------------------ |
| GitLab Web UI 접근 실패 | `https://gitlab.local` 응답 실패       | Nginx / GitLab           |
| GitLab 직접 접근 실패     | `http://172.30.1.67:8080` 응답 실패    | GitLab                   |
| GitLab Container 중지 | Container 상태 `exited`              | Docker / GitLab logs     |
| Runner Offline      | GitLab UI에서 Runner Offline         | Runner Container / token |
| Pipeline 실패         | 테스트 Pipeline Failed                | Runner logs / Job logs   |
| Nginx 접근 실패         | HTTPS redirect 또는 proxy 실패         | Nginx config / logs      |
| Disk 사용량 증가         | GitLab runtime volume 사용량 증가       | `/home/gali/gitlab/data` |
| Memory 사용량 지속 증가    | GitLab / Docker Desktop 메모리 사용량 증가 | GitLab / Docker Desktop  |

---

## 검증 기준

| 구분           | 정상 기준                               |
| ------------ | ----------------------------------- |
| Prometheus   | Web UI 접근 가능                        |
| Grafana      | Web UI 접근 가능                        |
| Datasource   | Grafana에서 Prometheus 연결 가능          |
| Resource 상태  | CPU / Memory / Disk 사용량 확인 가능       |
| Container 상태 | GitLab / Runner / Nginx 실행 상태 확인 가능 |
| GitLab 상태    | Web UI 접근 및 Project 유지 여부 확인 가능     |
| Runner 상태    | Runner Online 및 Pipeline 실행 가능      |
| Nginx 상태     | `https://gitlab.local` 접근 가능        |

---

## 주의사항

- GitLab 상태는 Container 실행 여부만으로 판단하지 않는다.
- GitLab 상태는 Project / Runner / Pipeline 상태를 함께 확인한다.
- Grafana는 Phase 8에서 Reverse Proxy 뒤에 두지 않는다.
- Grafana는 `http://172.30.1.67:3000`으로 직접 접근한다.
- Monitoring 관련 volume은 운영 데이터이므로 불필요하게 삭제하지 않는다.

---

## 완료 기준

- `docker-compose.monitoring.yml` 작성 완료
- Prometheus / Grafana 실행 완료
- Prometheus Web UI 접근 확인
- Grafana Web UI 접근 확인
- Grafana에서 Prometheus datasource 연결 확인
- Docker Container 리소스 수집 확인
- GitLab 상태 수집 확인
- GitLab / GitLab Runner / Nginx 상태 확인 기준 정리
- Grafana Dashboard 작성
- Alert 판단 기준 문서화

---