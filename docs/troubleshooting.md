# troubleshooting

> Self-Hosted DevOps Platform 구축 과정에서 발생한 문제와 원인, 해결 절차를 기록한다.

## GitHub-hosted runner에서 Mini PC SSH 접속 실패

### 증상

GitHub Actions에서 Mini PC 내부망 IP로 SSH 접속을 시도했지만 timeout이 발생했다.

```
dial tcp <MINI_PC_IP>:22: i/o timeout
```

### 원인
- GitHub-hosted runner는 GitHub 클라우드에서 실행되므로 내부망 사설 IP의 Mini PC에 직접 접근할 수 없다.
- Mini PC는 내부망 사설 IP를 사용하므로 GitHub-hosted runner가 직접 접근할 수 없다.

```
GitHub-hosted runner
 → Mini PC 내부망 IP로 SSH 접속
 → Docker Compose 실행
```

### 해결
- Mini PC WSL2 Ubuntu에 GitHub self-hosted runner를 등록하고 Workflow 실행 환경을 변경한다.
```
runs-on: self-hosted
```

#### 변경 후 구조
```
GitHub Actions
 → Mini PC self-hosted runner
 → Mini PC WSL2 Ubuntu 내부에서 Docker Compose 실행
```

> 이 방식은 GitHub가 Mini PC로 inbound SSH 접속하는 구조가 아니라, Mini PC의 runner가 GitHub Actions job을 outbound로 받아 실행하는 구조다.

---

## Docker credential helper 오류

### 증상

self-hosted runner에서 Docker image pull 단계가 실패했다.

```
error getting credentials - err: exit status 1, out: `A specified logon session does not exist. It may already have been terminated.`
```

### 원인

WSL2 Ubuntu의 Docker 설정이 Docker Desktop credential helper를 사용하도록 되어 있었다.

#### 문제 설정
```
{
  "credsStore": "desktop.exe"
}
```
- self-hosted runner 세션에서는 Windows Docker Desktop credential helper 접근이 실패할 수 있다.

### 해결

GitLab CE public image pull에는 Docker Hub 인증이 필요하지 않으므로 credential store를 사용하지 않도록 초기화한다.

```
mkdir -p ~/.docker

cat > ~/.docker/config.json <<'EOF'
{
  "auths": {}
}
EOF
```

#### 확인
```
cat ~/.docker/config.json
```

#### 정상 기준
```
{
  "auths": {}
}
```

#### Workflow에 Docker Compose 실행 전에 다음 step을 추가

```
- name: Reset Docker credential config
  run: |
    mkdir -p ~/.docker
    cat > ~/.docker/config.json <<'EOF'
    {
      "auths": {}
    }
    EOF
```

---

## GitLab Internal API unreachable

### 증상

GitLab 주요 서비스는 실행 중이지만 `gitlab-rake gitlab:check`에서 Internal API 실패가 발생했다.

```
docker exec -it gitlab gitlab-rake gitlab:check SANITIZE=true
```

#### 오류
```
Internal API available: FAILED - Internal API unreachable
gitlab-shell self-check failed
```

### 확인 결과

- 컨테이너 내부 80번 포트 readiness 확인이 실패했다.
```
docker exec -it gitlab curl -I http://127.0.0.1/-/readiness
```

#### 오류
```
curl: (7) Failed to connect to 127.0.0.1 port 80
```

- 반면 8080 포트에서는 Nginx 응답이 확인됐다.
```
docker exec -it gitlab curl -I http://127.0.0.1:8080/-/readiness
```

#### 응답
```
HTTP/1.1 502 Bad Gateway
Server: nginx
```

### 원인

- `external_url`에 8080 포트를 지정했다.
```
external_url 'http://${MINI_PC_HOST}:8080'
```
- 이 설정으로 인해 GitLab 내부 Nginx가 컨테이너 내부에서도 8080 포트를 기준으로 구성될 수 있다.

#### Docker Compose 포트 매핑 구조
```
ports:
  - "8080:80"
```
- 즉, 외부에서는 `Mini PC:8080`으로 접근하지만 컨테이너 내부에서는 80번 포트가 열려 있어야 한다.

#### 불일치 구조
```
Host 8080 → Container 80
GitLab 내부 Nginx → Container 8080
```

### 해결

GitLab 내부 Nginx listen port를 80으로 고정한다.

```
nginx['listen_port'] = 80
nginx['listen_https'] = false
```

#### 최종 설정
```
environment:
  GITLAB_OMNIBUS_CONFIG: |
    external_url 'http://${MINI_PC_HOST}:8080'

    nginx['listen_port'] = 80
    nginx['listen_https'] = false

    gitlab_rails['gitlab_shell_ssh_port'] = 2222

    puma['worker_processes'] = 1
    sidekiq['concurrency'] = 5
    postgresql['shared_buffers'] = "256MB"
```

### 적용

```
MINI_PC_HOST=172.30.1.69 docker compose -f infra/compose/docker-compose.gitlab.yml up -d --force-recreate
```

### 확인

#### 컨테이너 내부 80번 포트 확인
```
docker exec -it gitlab curl -I http://127.0.0.1/-/readiness
```

#### GitLab 서비스 상태 확인
```
docker exec -it gitlab gitlab-ctl status
```

### 판단 기준

- `docker ps`에서 GitLab Container가 `Up` 상태
- `gitlab-ctl status`에서 주요 서비스가 `run` 상태
- 컨테이너 내부 80번 포트에서 Nginx 또는 GitLab readiness 응답 확인
- Main PC에서 GitLab Web UI 접근 가능

---

## Docker Compose 실행 실패

### 확인 항목

- Docker Desktop 실행 여부 확인
- Docker Desktop WSL Integration 활성화 여부 확인
- Mini PC WSL2 Ubuntu에서 `docker version` 실행 가능 여부 확인
- `MINI_PC_HOST` Secret 등록 여부 확인
- `docker-compose.gitlab.yml`의 volume 경로 확인
- `external_url` 환경변수 치환 결과 확인

#### 확인 명령
```
MINI_PC_HOST=172.30.1.69 docker compose -f infra/compose/docker-compose.gitlab.yml config
```

---

## GitLab Web UI 접근 실패

### 확인 항목

- GitLab Container 상태 확인
- `docker logs -f gitlab` 확인
- `gitlab-ctl status` 확인
- `external_url` 값 확인
- 내부 Nginx listen port 확인
- Mini PC 방화벽 8080 포트 허용 여부 확인
- Main PC와 Mini PC 동일 네트워크 여부 확인

확인 명령:

```
docker ps --filter name=gitlab
docker exec -it gitlab gitlab-ctl status
docker exec -it gitlab curl -I http://127.0.0.1/-/readiness
```

---

## GitLab 재시작 후 프로젝트와 Runner 정보가 사라짐

### 증상

GitLab 재시작 또는 GitHub Actions 배포 이후 Phase 4에서 생성한 항목이 보이지 않았다.

- `runner-test` 프로젝트 없음
- `mini-pc-docker-runner` Runner 없음
- Admin Area → CI/CD → Runners 수 `0`

### 확인

#### 1. GitLab volume mount 확인

```bash
docker inspect gitlab --format '{{json .Mounts}}' | python3 -m json.tool
```

#### 정상 기준
```
/home/gali/gitlab/config → /etc/gitlab
/home/gali/gitlab/logs   → /var/log/gitlab
/home/gali/gitlab/data   → /var/opt/gitlab
```

### 2. GitLab DB 기준 프로젝트 확인
```
docker exec -it gitlab gitlab-rails runner "puts \"projects=#{Project.count}\"; Project.order(:id).each { |p| puts \"#{p.id} #{p.full_path}\" }"
```

#### 문제 발생 시 결과
```
projects=0
```

### 3. GitLab DB 기준 Runner 확인
```
docker exec -it gitlab gitlab-rails runner "puts \"runners=#{Ci::Runner.count}\"; Ci::Runner.order(:id).each { |r| puts \"#{r.id} #{r.description} active=#{r.active}\" }"
```

#### 문제 발생 시 결과
```
runners=0
```

### 판단

Volume mount는 정상이나 GitLab DB 기준으로 프로젝트와 Runner 정보가 없었다.

```
projects=0
runners=0
```
> 즉, UI 표시 문제가 아니라 현재 `/home/gali/gitlab/data` 기준 GitLab DB가 빈 상태로 초기화된 것으로 판단한다.

#### 가능한 원인
- Phase 4 수행 당시와 현재 GitLab 데이터 경로가 달랐을 가능성
- `/home/gali/gitlab/data`가 삭제 또는 재생성되었을 가능성
- GitHub Actions workspace 내부 상대 경로를 volume으로 사용했던 시점이 있었을 가능성

### 해결

#### 현재 정상 mount 경로 기준으로 Phase 4 재수행
```
docker rm -f gitlab-runner

rm -rf /home/gali/gitlab-runner/config
mkdir -p /home/gali/gitlab-runner/config
```

#### GitLab Web UI에서 Runner 재생성
```
Admin Area
 → CI/CD
 → Runners
 → Create instance runner
```

#### Runner Container 재실행
```
docker run -d \
  --name gitlab-runner \
  --restart unless-stopped \
  -v /home/gali/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

#### Runner 등록
```
docker exec -it gitlab-runner gitlab-runner register
```

`runner-test` 프로젝트와 `.gitlab-ci.yml`을 다시 생성한 뒤 Pipeline `Passed`를 확인한다.

### 재발 방지

- GitLab runtime volume은 `/home/gali/gitlab` 하위 절대 경로만 사용한다.
- GitHub Actions workspace 내부 경로를 GitLab runtime volume으로 사용하지 않는다.
- `/mnt/c`, `/mnt/d` 경로를 GitLab runtime volume으로 사용하지 않는다.
- `docker compose down -v`를 사용하지 않는다.
- `docker system prune --volumes`를 사용하지 않는다.
- `/home/gali/gitlab/data`를 삭제하지 않는다.
- 재배포 전후 mount 경로를 확인한다.