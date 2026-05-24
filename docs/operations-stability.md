# Operations Stability (GitLab 운영 안정성 검증)

> 장기 미가동 후 GitLab 재기동 시 데이터 유지 여부를 확인하고, Mini PC WSL2 + Docker Desktop 환경에서 GitLab Omnibus 장기 운영 가능성을 판단한다.

---

## 목적

Phase 6에서는 GitLab Container 실행 여부가 아니라 GitLab 내부 데이터 유지 여부를 기준으로 운영 안정성을 검증한다.

GitLab Omnibus는 PostgreSQL 기반 stateful 서비스이므로, Web UI 접근 가능 여부만으로 기존 GitLab 인스턴스가 정상 유지됐다고 판단하지 않는다.

### 검증 대상
- GitLab 프로젝트 유지 여부
- GitLab Runner 등록 정보 유지 여부
- root 계정 상태 유지 여부
- `application_settings` 유지 여부
- `gitlab-rails-db-migrate` 로그 확인
- `db:schema:load` 재발 여부
- GitLab Omnibus 장기 운영 가능 여부

---

## 발생 상황

Mini PC를 며칠 이상 구동하지 않은 뒤 다시 실행했을 때, GitLab Web UI는 정상 접근되지만 기존 프로젝트와 Runner 등록 정보가 사라지는 문제가 발생했다.

### 확인된 증상
- `runner-test` 프로젝트 없음
- `mini-pc-docker-runner` Runner 없음
- Admin Area → CI/CD → Runners 수 `0`
- root 비밀번호 재설정 필요
- GitLab Web UI는 접근 가능

> 이 상태는 단순 Runner 연결 문제나 Web UI 표시 문제가 아니다.
> GitLab application DB가 새로 bootstrap된 상태로 판단한다.

---

## 확인 1. Project / Runner 수 확인

### Runner 수 확인

```
docker exec -it gitlab gitlab-rails runner "puts \"runners=#{Ci::Runner.count}\""
```

### Project 수 확인

```
docker exec -it gitlab gitlab-rails runner "puts \"projects=#{Project.count}\""
```

### 확인 결과

```
runners=0
projects=0
```

기존 GitLab 프로젝트와 Runner 등록 정보가 유지되지 않았다.

---

## 확인 2. GitLab DB 상태 확인

DB 내부에서도 root 계정과 `application_settings`가 재기동 시점에 새로 생성된 것이 확인되었다.

### users 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT id, username, email, created_at
FROM users
ORDER BY id;
"
```

#### 확인 결과

```
1|root|gitlab_admin_xxx@example.com|2026-05-18 08:40:27
```

### application_settings 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT id, created_at, updated_at
FROM application_settings;
"
```

#### 확인 결과

```
1|2026-05-18 08:40:26|2026-05-18 08:40:27
```

기존 GitLab 인스턴스의 데이터가 유지된 것이 아니라, 재기동 시점에 GitLab 기본 데이터가 새로 생성된 상태다.

---

## 확인 3. GitLab DB bootstrap 로그 확인

### GitLab 재기동 후 `gitlab-rails-db-migrate` 로그
```
Running db:schema:load rake task

Seed from /opt/gitlab/embedded/service/gitlab-rails/db/fixtures/production/001_application_settings.rb
Creating the default ApplicationSetting record.

Seed from /opt/gitlab/embedded/service/gitlab-rails/db/fixtures/production/003_admin.rb
Administrator account created
```

이 로그는 기존 GitLab DB에 일반 migration만 적용된 것이 아니라, GitLab Rails DB schema가 다시 load되고 기본 seed가 실행된 상태를 의미한다.

---

## 영향 범위

이 문제가 발생하면 다음 정보가 유지되지 않는다.

- GitLab 프로젝트
- GitLab Runner 등록 정보
- root 계정 상태
- Pipeline 실행 이력
- GitLab Repository 데이터
- 기존 관리자 비밀번호

GitLab Container가 실행 중이고 Web UI가 표시되더라도 기존 GitLab 운영 상태가 유지되었다고 판단하면 안 된다.

---

## 운영 안정성 판단 기준

GitLab 상태는 컨테이너 실행 여부가 아니라 DB 내부 데이터를 기준으로 판단한다.

### 확인 명령

#### application_settings 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT id, created_at, updated_at
FROM application_settings;
"
```

#### root user 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT id, username, created_at
FROM users
ORDER BY id;
"
```

#### Project 수 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT COUNT(*) FROM projects;
"
```

#### Runner 수 확인

```
docker exec -it gitlab gitlab-psql -d gitlabhq_production -t -A -c "
SELECT COUNT(*) FROM ci_runners;
"
```

#### DB bootstrap 로그 확인

```
docker exec -it gitlab bash -lc '
grep -RInE "db:schema:load|Creating the default ApplicationSetting record|Administrator account created" \
/var/log/gitlab/gitlab-rails/gitlab-rails-db-migrate-*.log 2>/dev/null
'
```

---

## 정상 기준

- `application_settings.created_at`이 기존 구성 시점과 동일
- root user `created_at`이 기존 구성 시점과 동일
- 기존 GitLab 프로젝트가 유지됨
- 기존 GitLab Runner 등록 정보가 유지됨
- `gitlab-rails-db-migrate` 로그에 `db:schema:load`가 없음

---

## 비정상 기준

- `application_settings.created_at`이 재기동 시점으로 변경됨
- root user가 재기동 시점에 새로 생성됨
- `projects = 0`
- `ci_runners = 0`
- `gitlab-rails-db-migrate` 로그에 `db:schema:load`가 표시됨
- `Creating the default ApplicationSetting record` 로그가 표시됨
- `Administrator account created` 로그가 표시됨

---

## 검증 결과

Phase 6 검증 결과, 장기 미가동 후 재기동 시 GitLab Rails DB가 새로 bootstrap되는 문제가 확인되었다.

### 확인된 결과
- 기존 GitLab 프로젝트가 유지되지 않음
- GitLab Runner 등록 정보가 유지되지 않음
- root 계정이 재생성됨
- `application_settings`가 재생성됨
- `gitlab-rails-db-migrate` 로그에서 `db:schema:load` 실행 확인

---

## 원인 판단

> GitLab Rails DB가 재기동 시점에 db:schema:load 경로로 다시 bootstrap됨

### 현재 환경 기준으로 의심되는 범위

- Windows 11 + WSL2 + Docker Desktop 조합에서 장기 종료 후 GitLab stateful data 유지 불안정
- GitLab Omnibus Container 재기동 시 reconfigure / migration 흐름 재실행
- PostgreSQL cluster는 존재하지만 GitLab application data가 초기 상태로 재구성되는 상황
- 기존에 `latest` 이미지를 사용했던 경우 migration / reconfigure 흐름 추적이 어려워진 점

---

## 최종 판단

현재 Mini PC WSL2 + Docker Desktop 환경에서는 GitLab Container 기동, GitLab Web UI 접근, GitLab Runner 등록, Docker Executor 기반 Pipeline 실행까지는 가능하다.

하지만 장기 미가동 후 재기동 시 GitLab DB가 재초기화되는 문제가 확인되었다.

따라서 현재 환경에서는 GitLab Omnibus를 장기 운영 대상으로 확정하지 않는다.

---

## 후속 처리 방향

현재 프로젝트의 Phase 1 ~ Phase 5는 다음 범위까지 검증 완료로 본다.

- GitLab Container 실행
- GitLab Web UI 접근
- GitHub Actions self-hosted runner 기반 초기 배포
- GitLab Runner 등록
- Docker Executor 기반 Pipeline 실행
- GitLab Repository 기준 `validate-compose` / `deploy-readiness-check` Pipeline 검증

다만 GitLab Omnibus 장기 운영은 안정성 기준을 충족하지 못했으므로, Reverse Proxy / Monitoring / 운영 문서화는 GitLab 장기 운영 확장이 아니라 현재 검증 결과를 반영한 후속 정리 범위로 진행한다.

---

## 재발 방지 기준

- GitLab runtime volume은 `/home/<USER>/gitlab` 하위 절대 경로만 사용한다.
- GitHub Actions workspace 내부 경로를 GitLab runtime volume으로 사용하지 않는다.
- `/mnt/c`, `/mnt/d` 경로를 GitLab runtime volume으로 사용하지 않는다.
- `docker compose down -v`를 사용하지 않는다.
- `docker system prune --volumes`를 사용하지 않는다.
- `/home/<USER>/gitlab/data`를 삭제하지 않는다.
- 재배포 전후 mount 경로를 확인한다.
- GitLab image는 명시 버전으로 고정한다.

---

## 결론

Phase 6 GitLab 운영 안정성 검증은 수행 완료했다.

결과는 운영 안정성 통과가 아니라, 장기 미가동 후 GitLab DB 재초기화 문제가 확인된 상태다.

현재 Mini PC WSL2 + Docker Desktop 환경에서는 GitLab Omnibus 장기 운영을 확정하지 않는다.

---