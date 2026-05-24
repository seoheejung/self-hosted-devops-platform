# Monitoring 구성

> Mini PC 기반 GitLab 운영 환경에서 Prometheus와 Grafana를 활용하여 리소스 상태와 주요 서비스 상태를 관측한다.

---

## 목적

Phase 8에서는 Mini PC 기반 Self-Hosted DevOps 환경의 Monitoring 구성을 진행한다.

Monitoring의 목적은 Grafana Dashboard 작성 자체가 아니라, GitLab 운영 중 문제가 발생했을 때 확인해야 할 리소스 상태, 서비스 상태, 접근 상태, 로그 확인 기준을 명확히 정리하는 것이다.

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
 └─ Monitoring Stack
     ├─ Prometheus
     ├─ Grafana
     ├─ cAdvisor
     └─ node_exporter
```

---

## 구성 기준

| 항목 | 내용 |
| --- | --- |
| Metrics 수집 | Prometheus |
| 시각화 | Grafana |
| Docker Container Metrics | cAdvisor |
| WSL2 Ubuntu Metrics | node_exporter |
| GitLab 상태 확인 | GitLab metrics endpoint + Web UI + Pipeline 상태 |
| GitLab Runner 상태 확인 | GitLab UI + Runner Container + Pipeline 결과 |
| Nginx 상태 확인 | Container 상태 + `nginx -t` + 접근 테스트 |
| 실행 방식 | Docker Compose |
| Compose 파일 | `infra/compose/docker-compose.monitoring.yml` |
| Prometheus 설정 | `infra/monitoring/prometheus/prometheus.yml` |
| Grafana 설정 | `infra/monitoring/grafana/` |
| Prometheus 접근 | `http://172.30.1.67:9090` |
| Grafana 접근 | `http://172.30.1.67:3000` |
| cAdvisor 접근 | `http://172.30.1.67:8081` |
| Alert | 장애 판단 기준 문서화 |

---

## Monitoring 대상

| 대상 | 확인 목적 | 수집 / 확인 기준 |
| --- | --- | --- |
| Prometheus | Metrics 수집 서버 상태 확인 | Prometheus 자체 scrape |
| Grafana | Dashboard 접근 및 datasource 상태 확인 | Web UI / datasource test |
| Docker Container | GitLab / Runner / Nginx / Monitoring Container 리소스 확인 | cAdvisor |
| WSL2 Ubuntu | CPU, Memory, Disk, Network 상태 확인 | node_exporter |
| GitLab | Web UI, Project, Pipeline 상태 확인 | metrics endpoint + Web UI + 수동 확인 |
| GitLab Runner | Runner Online 여부와 Job 실행 가능 여부 확인 | GitLab UI + Runner Container + Pipeline 결과 |
| Nginx | Reverse Proxy 접근 경로 유지 여부 확인 | Container 상태 + `nginx -t` + 접근 테스트 |
| Windows Host | Mini PC 전체 운영 상태 참고 | 직접 Metrics 수집 제외 |

---

## 전체 흐름

```
Prometheus
 ├─ Prometheus 자체 상태 수집
 ├─ Docker Container 리소스 수집
 │   └─ cAdvisor
 ├─ WSL2 Ubuntu 리소스 수집
 │   └─ node_exporter
 └─ GitLab Metrics 수집
     └─ GitLab metrics endpoint 사용 가능 여부 확인

Grafana
 ├─ Prometheus Datasource 연결
 ├─ Resource Monitoring Dashboard 구성
 ├─ Container 상태 시각화
 ├─ GitLab 상태 확인 기준 반영
 ├─ GitLab Runner 상태 확인 기준 반영
 └─ Nginx 상태 확인 기준 반영
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
│
└─ docs/
   └─ monitoring.md
```

### 경로 기준

| 경로 | 역할 |
| --- | --- |
| `infra/compose/docker-compose.monitoring.yml` | Monitoring Stack Compose 파일 |
| `infra/monitoring/prometheus/prometheus.yml` | Prometheus scrape 설정 |
| `infra/monitoring/grafana/provisioning/datasources/prometheus.yml` | Grafana Prometheus datasource 설정 |
| `infra/monitoring/grafana/provisioning/dashboards/dashboards.yml` | Grafana dashboard provisioning 설정 |
| `infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json` | Grafana Dashboard JSON |
| `docs/monitoring.md` | Monitoring 구성 문서 |

---

## Prometheus 구성 기준

> Prometheus는 Monitoring Stack의 Metrics 수집 서버로 사용

### Scrape 대상

| Job | Target | 목적 |
| --- | --- | --- |
| `prometheus` | `prometheus:9090` | Prometheus 자체 상태 확인 |
| `cadvisor` | `cadvisor:8080` | Docker Container 리소스 수집 |
| `node-exporter` | `node-exporter:9100` | WSL2 Ubuntu 리소스 수집 |
| `gitlab` | `gitlab:80/-/metrics` | GitLab Metrics 수집 |

### Prometheus 설정 기준

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: cadvisor
    static_configs:
      - targets:
          - cadvisor:8080

  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100

  - job_name: gitlab
    metrics_path: /-/metrics
    static_configs:
      - targets:
          - gitlab:80
```

### 판단 기준

- `prometheus`, `cadvisor`, `node-exporter` target은 기본 수집 대상이다.
- `gitlab` target은 GitLab metrics endpoint와 Docker network 연결 상태를 함께 확인한다.
- GitLab target이 `DOWN`이어도 GitLab 상태를 즉시 장애로 판단하지 않는다.
- GitLab 상태는 Web UI, Project, Runner, Pipeline 상태와 함께 판단한다.

---

## Grafana 구성 기준

> Grafana는 Prometheus에 수집된 Metrics를 시각화하는 용도로 사용

### Datasource 기준

| 항목 | 값 |
| --- | --- |
| Type | Prometheus |
| URL | `http://prometheus:9090` |
| Access | proxy |
| Default | true |

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

Dashboard JSON은 다음 경로에 저장한다.

```
infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json
```

---

## Docker Container Metrics 기준

> Docker Container 리소스는 cAdvisor를 통해 확인

### 확인 대상 Container
- `gitlab`
- `gitlab-runner`
- `gitlab-nginx`
- `gitlab-prometheus`
- `gitlab-grafana`
- `gitlab-cadvisor`
- `gitlab-node-exporter`

### 확인 항목
- Container 실행 상태
- CPU 사용량
- Memory 사용량
- Restart 여부
- Last seen 상태
- Container별 리소스 사용 추이

### Prometheus Query 후보

```
container_last_seen
```

```
container_memory_usage_bytes
```

```
rate(container_cpu_usage_seconds_total[5m])
```

---

## WSL2 Ubuntu Metrics 기준

> WSL2 Ubuntu 리소스는 node_exporter를 통해 확인

### 확인 항목
- CPU 사용량
- Memory 사용량
- Disk 사용량
- Network 상태
- Filesystem 상태

### Prometheus Query 후보

```
node_cpu_seconds_total
```

```
node_memory_MemAvailable_bytes
```

```
node_filesystem_avail_bytes
```

```
node_network_receive_bytes_total
```

### 주의

node_exporter는 WSL2 Ubuntu 기준의 Linux Metrics를 수집한다.

Windows Host 전체 리소스와 완전히 동일한 의미로 해석하지 않는다.

Windows Host 직접 Metrics 수집은 별도 Exporter 도입이 필요한 별도 작업이다.

---

## GitLab 상태 확인 기준

> GitLab 상태는 Container 실행 여부 외 추가로 확인

GitLab Container가 실행 중이어도 내부 DB 상태나 프로젝트 유지 상태가 정상이라고 단정할 수 없다.

### 확인 항목

- GitLab Web UI 접근 가능 여부
- root 계정 로그인 가능 여부
- 기존 Project 유지 여부
- Runner 등록 정보 유지 여부
- Pipeline 실행 가능 여부
- GitLab 주요 로그 이상 여부
- GitLab metrics endpoint 사용 가능 여부

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
- GitLab metrics endpoint가 `UP`이어도 Project / Runner / Pipeline 상태를 함께 확인한다.

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

| Panel | 목적 |
| --- | --- |
| Prometheus Target 상태 | scrape target 상태 확인 |
| Container CPU 사용량 | GitLab / Runner / Nginx 리소스 확인 |
| Container Memory 사용량 | Container별 메모리 사용량 확인 |
| Container Last Seen | Container 상태 확인 |
| WSL2 CPU 사용량 | WSL2 Ubuntu CPU 상태 확인 |
| WSL2 Memory 사용량 | WSL2 Ubuntu 메모리 상태 확인 |
| WSL2 Disk 사용량 | GitLab runtime volume 사용량 확인 |
| GitLab 상태 | Web UI / Metrics / Project 유지 여부 확인 기준 연결 |
| Runner 상태 | Runner Online / Pipeline 실행 여부 확인 기준 연결 |
| Nginx 상태 | Reverse Proxy 접근 경로 확인 기준 연결 |

Dashboard는 Grafana UI에서 작성한 뒤 JSON으로 export하여 Repository에 저장한다.

```
infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json
```

---

## Alert 판단 기준

Alertmanager, Slack, Email 연동은 제외하고, 어떤 상태를 장애로 볼지 판단 기준을 문서화한다.

| 장애 후보 | 판단 기준 | 확인 대상 |
| --- | --- | --- |
| GitLab Web UI 접근 실패 | `https://gitlab.local` 응답 실패 | Nginx / GitLab |
| GitLab 직접 접근 실패 | `http://172.30.1.67:8080` 응답 실패 | GitLab |
| GitLab Container 중지 | Container 상태 `exited` | Docker / GitLab logs |
| Runner Offline | GitLab UI에서 Runner Offline | Runner Container / token |
| Pipeline 실패 | 테스트 Pipeline Failed | Runner logs / Job logs |
| Nginx 접근 실패 | HTTPS redirect 또는 proxy 실패 | Nginx config / logs |
| Disk 사용량 증가 | GitLab runtime volume 사용량 증가 | `/home/gali/gitlab/data` |
| Memory 사용량 지속 증가 | GitLab / Docker Desktop 메모리 사용량 증가 | GitLab / Docker Desktop |
| Prometheus Target DOWN | scrape target DOWN | exporter / network / service |

---

## 검증 기준

| 구분 | 정상 기준 |
| --- | --- |
| Prometheus | Web UI 접근 가능 |
| Grafana | Web UI 접근 가능 |
| cAdvisor | Web UI 접근 가능 |
| Datasource | Grafana에서 Prometheus 연결 가능 |
| Prometheus Target | `prometheus`, `cadvisor`, `node-exporter` UP |
| GitLab Target | GitLab metrics 설정 후 UP |
| Container 상태 | GitLab / Runner / Nginx 실행 상태 확인 가능 |
| Resource 상태 | CPU / Memory / Disk 사용량 확인 가능 |
| GitLab 상태 | Web UI 접근 및 Project 유지 여부 확인 가능 |
| Runner 상태 | Runner Online 및 Pipeline 실행 가능 |
| Nginx 상태 | `https://gitlab.local` 접근 가능 |

---

## 주의사항

- Monitoring 문서는 작업 순서가 아니라 운영 관측 기준을 정리하는 문서다.
- cAdvisor는 Docker Container 리소스 확인용으로 사용한다.
- node_exporter는 WSL2 Ubuntu 리소스 확인용으로 사용한다.
- Windows Host 직접 Metrics 수집은 이번 Phase에서 제외한다.
- Windows Host 전체 상태와 WSL2 Ubuntu Metrics는 동일하지 않다.
- GitLab 상태는 Container 실행 여부만으로 판단하지 않는다.
- GitLab metrics endpoint가 `UP`이어도 Project / Runner / Pipeline 상태를 함께 확인한다.
- Grafana는 Phase 8에서 Reverse Proxy 뒤에 두지 않는다.
- Grafana는 `http://172.30.1.67:3000`으로 직접 접근한다.
- `prometheus-data`, `grafana-data` volume은 운영 데이터이므로 불필요하게 삭제하지 않는다.

---

## 완료 기준

- `docker-compose.monitoring.yml` 작성 완료
- Prometheus / Grafana / cAdvisor / node_exporter 실행 완료
- Prometheus Web UI 접근 확인
- Grafana Web UI 접근 확인
- cAdvisor Web UI 접근 확인
- Grafana에서 Prometheus datasource 연결 확인
- Prometheus Targets 상태 확인
- Docker Container Metrics 수집 확인
- WSL2 Ubuntu Metrics 수집 확인
- GitLab metrics endpoint 사용 가능 여부 확인
- GitLab / GitLab Runner / Nginx 상태 확인 기준 정리
- Grafana Dashboard 작성
- Alert 판단 기준 문서화

---