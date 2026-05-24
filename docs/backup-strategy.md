# Backup / Restore

> GitLab runtime data와 관련 설정 파일을 보존하고, 복구 후 DB 재초기화 여부를 확인하기 위한 백업/복구 기준

## 백업 대상

| 대상 | 경로 | 설명 |
| --- | --- | --- |
| GitLab config | `/home/gali/gitlab/config` | GitLab 설정 |
| GitLab data | `/home/gali/gitlab/data` | Repository, PostgreSQL, uploads |
| GitLab logs | `/home/gali/gitlab/logs` | 장애 분석 로그 |
| GitLab Runner config | `/home/gali/gitlab-runner/config` | Runner 등록 설정 |
| Nginx runtime | `/home/gali/nginx` | Reverse Proxy 설정 및 인증서 |
| Monitoring runtime | `/home/gali/monitoring` | Prometheus / Grafana 설정 |

---

## 백업 기준

GitLab은 PostgreSQL을 포함하는 stateful service이므로 runtime 디렉토리 백업은 Container를 정지한 상태에서 수행한다.

이 문서의 백업 방식은 **cold backup** 기준이다.

### Cold backup이란

Cold backup은 서비스와 관련 프로세스를 정지한 뒤, 데이터 파일을 백업하는 방식이다.

GitLab Container가 실행 중인 상태에서 `/home/gali/gitlab/data`를 압축하면 PostgreSQL 데이터 파일이 쓰기 중일 수 있다. 이 경우 백업 파일 안의 DB 상태가 일관되지 않을 수 있다.

따라서 이 프로젝트에서는 백업 전 GitLab, GitLab Runner, Nginx, Monitoring Container를 정지한 뒤 runtime 디렉토리를 압축한다.

### 선택 이유

| 항목 | 판단 |
| --- | --- |
| 구현 난이도 | 낮음 |
| 데이터 일관성 | 실행 중 백업보다 안전 |
| 서비스 중단 | 발생 |
| 현재 프로젝트 적합성 | 적합 |

현재 환경은 개인용 Mini PC 기반 검증 환경이므로 무중단 백업보다 데이터 일관성과 복구 가능성을 우선한다.

### 주의

Cold backup은 백업 중 서비스가 중단된다.  
운영 중단 없이 백업하려면 GitLab 공식 백업 명령 또는 PostgreSQL dump 기반 hot backup 전략을 별도로 검토해야 한다.

---

## 백업 절차

### 1. Container 정지

```
docker stop gitlab gitlab-runner gitlab-nginx monitoring-prometheus monitoring-grafana 2>/dev/null || true
```

### 2. 백업 생성

```
mkdir -p /home/gali/backups

tar -czf /home/gali/backups/mini-pc-devops-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  /home/gali/gitlab \
  /home/gali/gitlab-runner \
  /home/gali/nginx \
  /home/gali/monitoring
```

### 3. Container 재시작

```
docker start gitlab
docker start gitlab-runner
docker start gitlab-nginx
docker start monitoring-prometheus
docker start monitoring-grafana
```

---

## 복구 절차

### 1. Container 정지

```
docker stop gitlab gitlab-runner gitlab-nginx monitoring-prometheus monitoring-grafana
```

### 2. 기존 runtime 경로 보존

```
mkdir -p /home/gali/restore-before

BACKUP_TIME=$(date +%Y%m%d-%H%M%S)

mv /home/gali/gitlab /home/gali/restore-before/gitlab-$BACKUP_TIME 2>/dev/null || true
mv /home/gali/gitlab-runner /home/gali/restore-before/gitlab-runner-$BACKUP_TIME 2>/dev/null || true
mv /home/gali/nginx /home/gali/restore-before/nginx-$BACKUP_TIME 2>/dev/null || true
mv /home/gali/monitoring /home/gali/restore-before/monitoring-$BACKUP_TIME 2>/dev/null || true
```

### 3. 백업 압축 해제

```
tar -xzf /home/gali/backups/<BACKUP_FILE>.tar.gz -C /
```

### 4. Docker Compose 재실행

```
cd /home/gali/actions-runner/_work/self-hosted-devops-platform/self-hosted-devops-platform

MINI_PC_HOST=172.30.1.67 docker compose -f infra/compose/docker-compose.gitlab.yml up -d
docker compose -f infra/compose/docker-compose.monitoring.yml up -d
```

---

## 복구 후 검증

### GitLab Project 수 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "SELECT COUNT(*) FROM projects;"
```

### GitLab Runner 수 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "SELECT COUNT(*) FROM ci_runners;"
```

### Application settings 생성 시각 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "SELECT id, created_at, updated_at FROM application_settings;"
```

### DB bootstrap 로그 확인

```
docker exec -it gitlab bash -lc '
grep -RInE "db:schema:load|Creating the default ApplicationSetting record|Administrator account created" \
/var/log/gitlab/gitlab-rails/gitlab-rails-db-migrate-*.log 2>/dev/null
'
```

### Runner / Pipeline 확인

- GitLab Web UI에서 Runner 상태 확인
- GitLab Pipeline 실행 가능 여부 확인
- `validate-compose` 또는 테스트 Pipeline 실행 확인

---

## 실패 판단

아래 항목 중 하나라도 해당하면 복구 실패 또는 GitLab DB 재초기화 상태로 판단한다.

- `projects = 0`
- `ci_runners = 0`
- `application_settings.created_at`이 복구 시점으로 변경됨
- `db:schema:load` 로그 발생
- `Creating the default ApplicationSetting record` 로그 발생
- `Administrator account created` 로그 발생

---

## 금지 명령

아래 명령은 GitLab runtime data를 삭제할 수 있으므로 사용하지 않는다.

```
docker compose down -v
docker system prune --volumes
rm -rf /home/gali/gitlab/data
rm -rf /home/gali/gitlab/config
```

---