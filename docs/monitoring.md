# Monitoring 구성

> Mini PC 기반 GitLab 운영 환경에 Prometheus와 Grafana 기반 Monitoring Stack을 구성하고, 기본 Metrics 수집과 Dashboard 연동 흐름을 검증한다.

---

## 목적

Mini PC 기반 Self-Hosted DevOps 환경에 Prometheus와 Grafana 기반 Monitoring Stack을 구성한다.

이번 단계의 목적은 Prometheus / Grafana 실행 구조, datasource 연결, dashboard provisioning, GitHub Actions 기반 배포 흐름을 검증하는 것이다.

현재 구성에서는 Prometheus 자체 상태를 수집 대상으로 두고, Grafana Dashboard에서 Prometheus target 상태와 scrape 상태를 확인한다.

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

---

## Monitoring Stack 구성 대상

| 대상 | 목적 |
| --- | --- |
| Prometheus | Metrics 수집 서버 구성 |
| Grafana | Prometheus Metrics 시각화 |
| Prometheus datasource | Grafana에서 Prometheus 연결 |
| Dashboard provisioning | Dashboard JSON 파일 기반 관리 |
| GitHub Actions 배포 흐름 | Monitoring 설정 파일을 Mini PC runtime 경로에 반영 |
| `/home/gali/monitoring` | Prometheus / Grafana runtime 설정 파일 유지 |

---

## 전체 흐름

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
│        │  │  └─ datasource.yml
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
| `infra/monitoring/grafana/provisioning/datasources/datasource.yml` | Grafana Prometheus datasource 설정  |
| `infra/monitoring/grafana/provisioning/dashboards/dashboards.yml`  | Grafana dashboard provisioning 설정 |
| `infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json` | Grafana Dashboard JSON |

---

## Prometheus 구성 기준

> Prometheus는 Monitoring Stack의 Metrics 수집 서버로 사용

### 수집 대상

| 대상 | 목적 |
| --- | --- |
| Prometheus 자체 상태 | Metrics 수집 서버 상태 확인 |

### 판단 기준

- Prometheus는 Monitoring Stack의 기본 Metrics 수집 서버로 사용한다.
- 현재 단계에서는 Prometheus 자체 상태를 수집 대상으로 구성한다.
- Prometheus `/targets` 화면에서 `prometheus` target이 `UP`이면 기본 scrape 동작이 정상이다.
- 이후 수집 대상이 추가될 경우 `prometheus.yml`의 `scrape_configs`를 확장한다.

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


### Dashboard provisioning 기준

Grafana dashboard는 파일 기반 provisioning으로 관리한다.

Dashboard는 Grafana UI에서 작성한 뒤 JSON으로 export하여 Repository에 저장한다.

```
infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json
```

---

## Resource Monitoring 기준

Resource Monitoring은 Mini PC 운영 환경의 Monitoring 기반을 확인하기 위한 Dashboard 영역이다.

### Grafana에서 확인 가능한 대상

| 대상 | 확인 항목 | 확인 방식 |
| --- | --- | --- |
| Prometheus 자체 상태 | Target UP 여부 | `up{job="prometheus"}` |
| Prometheus scrape 상태 | Scrape 소요 시간 | `scrape_duration_seconds{job="prometheus"}` |
| Prometheus process 상태 | Memory 사용량 | `process_resident_memory_bytes{job="prometheus"}` |
| Prometheus process 상태 | CPU 사용량 | `rate(process_cpu_seconds_total{job="prometheus"}[5m])` |

### Dashboard 구성 항목

| Panel | Query | 정상 기준 |
| --- | --- | --- |
| Prometheus Target Status | `up{job="prometheus"}` | `1` |
| Prometheus Scrape Duration | `scrape_duration_seconds{job="prometheus"}` | 값 조회 가능 |
| Prometheus Memory Usage | `process_resident_memory_bytes{job="prometheus"}` | 값 조회 가능 |
| Prometheus CPU Usage | `rate(process_cpu_seconds_total{job="prometheus"}[5m])` | 값 조회 가능 |
| Monitoring Scope | Text panel | 현재 Dashboard 구성 범위와 향후 수집 대상 정리 |

### 판단 기준

- Resource Monitoring은 현재 Prometheus에서 수집 가능한 Metrics를 기준으로 구성한다.
- 현재 Grafana에서 직접 확인 가능한 항목은 Prometheus 자체 상태다.

---


## Dashboard 구성 기준

> Dashboard는 현재 Prometheus에서 수집 가능한 Metrics를 기준으로 구성

| Dashboard 영역 | 목적 |
| --- | --- |
| Prometheus Target Status | Prometheus scrape target 상태 확인 |
| Prometheus Scrape Duration | Prometheus scrape 소요 시간 확인 |
| Prometheus Memory Usage | Prometheus process memory 사용량 확인 |
| Prometheus CPU Usage | Prometheus process CPU 사용량 확인 |
| Monitoring Scope | 현재 구성 범위와 이후 확장 가능한 수집 대상 정리 |

---

## 검증 기준

| 구분 | 정상 기준 |
| --- | --- |
| Prometheus | Web UI 접근 가능 |
| Prometheus Target | `prometheus` target `UP` |
| Grafana | Web UI 접근 가능 |
| Datasource | Grafana에서 Prometheus 연결 가능 |
| Dashboard provisioning | `Mini PC DevOps Overview` Dashboard 표시 |
| Prometheus Target Status | `up{job="prometheus"}` 값 `1` |
| Prometheus Scrape Duration | 값 조회 가능 |
| Prometheus Memory Usage | 값 조회 가능 |
| Prometheus CPU Usage | 값 조회 가능 |
| Dashboard JSON | Repository에 export 결과 반영 |

---

## 완료 기준

- `docker-compose.monitoring.yml` 작성 완료
- Prometheus / Grafana 실행 완료
- GitHub Actions에서 Monitoring Stack 배포 성공
- `/home/gali/monitoring` runtime 파일 복사 확인
- Prometheus Web UI 접근 확인
- Prometheus target `UP` 확인
- Grafana Web UI 접근 확인
- Grafana에서 Prometheus datasource 연결 확인
- Grafana dashboard provisioning 확인
- Prometheus 자체 상태 Dashboard 작성
- Dashboard JSON export
- `infra/monitoring/grafana/dashboards/mini-pc-devops-overview.json` 반영

---