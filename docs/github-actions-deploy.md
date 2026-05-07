# GitHub Actions Deploy

> GitLab 서버가 아직 구축되지 않은 초기 단계에서 GitHub Actions와 Mini PC WSL2 Ubuntu의 self-hosted runner를 사용하여 GitLab Container를 자동 배포한다.

---

## 전체 흐름

```text
Main PC
 └─ Git Push / PR Merge
        ↓

GitHub Repository
 └─ GitHub Actions
        ↓

Mini PC self-hosted runner
 └─ Job 수신
        ↓

Mini PC WSL2 Ubuntu
 ├─ Docker Compose 실행
 ├─ GitLab runtime volume 생성
 └─ GitLab Container 실행
```

---

## 구조 판단

### 실패한 방식

```
GitHub-hosted runner
 └─ Mini PC 내부망 IP로 SSH 접속
        ↓
실패: 내부망 사설 IP 접근 불가
```
- GitHub-hosted runner는 GitHub 클라우드에서 실행되므로 내부망 사설 IP의 Mini PC에 직접 SSH 접속할 수 없다.
```
dial tcp <MINI_PC_IP>:22: i/o timeout
```

### 적용한 방식

```
GitHub Actions
 └─ Mini PC self-hosted runner
        ↓
Mini PC WSL2 Ubuntu 내부에서 Docker Compose 실행
```

> GitHub가 Mini PC로 inbound SSH 접속하는 구조가 아니라, Mini PC의 runner가 GitHub Actions job을 outbound로 받아 실행한다.

---

## 전제 조건

- Main PC에서 GitHub Repository로 push 가능
- Mini PC에 WSL2 Ubuntu 구성 완료
- Mini PC에서 Docker Desktop 실행 가능
- Docker Desktop WSL Integration 활성화
- Mini PC WSL2 Ubuntu에서 Docker 명령어 사용 가능
- Mini PC WSL2 Ubuntu에 GitHub self-hosted runner 등록 완료
- GitHub Repository Secrets에 `MINI_PC_HOST` 등록 완료

---


## 작업 진행

### 1. Mini PC WSL2 Ubuntu 상태 확인

#### Mini PC에서 실행
```
wsl -l -v
```

#### 정상 기준
```
NAME              STATE           VERSION
* Ubuntu-22.04     Running         2
  docker-desktop   Running         2
```

#### WSL Ubuntu로 접속

```
wsl -d Ubuntu-22.04
```

---


### 2. Docker 사용 가능 여부 확인

#### Mini PC WSL2 Ubuntu에서 실행
```
docker version
docker compose version
```

#### 정상 기준
- Docker Client / Server 정보 출력
- Docker Compose version 출력

---

### 3. GitHub self-hosted runner 설치

#### GitHub Repository 화면에서 runner 등록 명령 확인
```
Settings
 → Actions
 → Runners
 → New self-hosted runner
 → Linux
 → x64
```

#### Mini PC WSL2 Ubuntu에서 실행
```
mkdir -p ~/actions-runner
cd ~/actions-runner
```

#### GitHub 화면에서 제공하는 runner 다운로드 명령 실행
```
curl -o actions-runner-linux-x64-<VERSION>.tar.gz -L <GITHUB_RUNNER_DOWNLOAD_URL>
tar xzf ./actions-runner-linux-x64-<VERSION>.tar.gz
```

#### Runner 등록
```
./config.sh --url <GITHUB_REPO_URL> --token <GITHUB_RUNNER_TOKEN>
```

#### 주의
- `<GITHUB_RUNNER_TOKEN>`은 GitHub 화면에서 발급되는 값을 사용한다.
- Runner token은 문서나 Git에 저장하지 않는다.
- Runner token은 노출되면 새로 발급받는다.


#### Runner 실행
```
./run.sh
```

#### 정상 기준
- √ Connected to GitHub
- Listening for Jobs

#### 마무리
- GitHub Repository의 runner 화면에서 상태 확인
```
Settings
 → Actions
 → Runners
```

#### 정상 기준
- Status: Idle 또는 Online
- Labels: self-hosted, Linux, X64

---

### 4. GitHub Secrets 등록

```
Settings
 → Secrets and variables
 → Actions
 → New repository secret
```

#### 등록 값
| Secret | 값 |
| --- | --- |
| `MINI_PC_HOST` | Mini PC 내부 IP |

> `MINI_PC_HOST`는 GitLab `external_url` 구성에 사용한다.

---

### 5. GitLab runtime volume 경로 준비

> GitLab runtime 데이터는 GitHub Actions workspace가 아니라 Mini PC WSL2 Ubuntu 내부 고정 경로에 저장한다.

```
mkdir -p /home/<USER>/gitlab/config
mkdir -p /home/<USER>/gitlab/data
mkdir -p /home/<USER>/gitlab/logs
```

#### 경로 역할

| 경로 | 역할 |
| --- | --- |
| `/home/<USER>/gitlab/config` | GitLab 설정 파일 |
| `/home/<USER>/gitlab/data` | Repository, DB, 업로드 파일 등 실제 데이터 |
| `/home/<USER>/gitlab/logs` | GitLab 로그 |

---

### 6. GitHub Actions Workflow 작성

- 파일: `.github/workflows/deploy-gitlab.yml`

Workflow는 `main` 브랜치에 GitLab Compose 파일 또는 배포 Workflow 파일이 반영될 때 실행된다.

#### 실행 기준

- 실행 브랜치: `main`
- 실행 조건:
    - `infra/compose/docker-compose.gitlab.yml` 변경
    - `.github/workflows/deploy-gitlab.yml` 변경
- 실행 환경:
    - GitHub-hosted runner가 아닌 Mini PC WSL2 Ubuntu의 self-hosted runner
- 실행 방식:
    - `actions/checkout@v4`로 Repository checkout
    - GitLab runtime volume 디렉토리 생성
    - Docker credential config 초기화
    - Docker Compose로 GitLab Container 실행

#### 핵심 설정

```
runs-on: self-hosted
```
- `runs-on: self-hosted`를 사용하므로 GitHub Actions job은 Mini PC WSL2 Ubuntu에 등록된 runner에서 실행된다.
- GitHub-hosted runner는 내부망 사설 IP의 Mini PC에 직접 접근할 수 없기 때문에 사용하지 않는다.

#### 배포 시 수행 작업
1. Repository checkout
2. `/home/<USER>/gitlab/config` 생성
3. `/home/<USER>/gitlab/data` 생성
4. `/home/<USER>/gitlab/logs` 생성
5. Docker credential config 초기화
6. docker compose up -d 실행
7. docker ps로 GitLab Container 상태 확인

#### GitHub Secrets 사용

| Secret | 용도 |
| --- | --- |
| `MINI_PC_HOST` | GitLab `external_url`에 사용할 Mini PC 내부 IP |

`MINI_PC_HOST`는 Docker Compose 실행 시 환경변수로 주입된다.

Compose 파일에서는 아래 형태로 사용한다.

```
external_url 'http://${MINI_PC_HOST}:8080'
```

---

### 7. GitHub Actions 실행 확인

```
Actions
 → Deploy GitLab to Mini PC
```

#### 정상 기준
- Workflow 실행됨
- self-hosted runner에서 job 수신
- Checkout 성공
- GitLab volume 디렉토리 생성 성공
- Docker credential config 초기화 성공
- Docker Compose 실행 성공
- GitLab Container Up 상태 확인

---

### 8. Mini PC 상태 확인

#### gitlab container Up 확인
```
docker ps
```

#### GitLab 로그 확인
```
docker logs -f gitlab
```

#### GitLab 서비스 상태 확인

```
docker exec -it gitlab gitlab-ctl status
```

#### Main PC에서 GitLab Web UI 접근

```
http://<MINI_PC_IP>:8080
```

---

### 9. 초기 root 비밀번호 확인

#### Mini PC WSL2 Ubuntu에서 실행
```
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

#### 로그인 정보

```
Username: root
Password: initial_root_password 파일 내용
```

> 주의: initial_root_password 파일은 일정 시간이 지나면 삭제될 수 있으므로 초기 실행 후 바로 확인한다.

---

## 트러블슈팅

### GitHub-hosted runner에서 Mini PC SSH 접속 실패

#### 증상

```
dial tcp <MINI_PC_IP>:22: i/o timeout
```

#### 원인
- GitHub-hosted runner는 GitHub 클라우드에서 실행되므로 내부망 사설 IP의 Mini PC에 직접 접근할 수 없다.

#### 해결
- Mini PC WSL2 Ubuntu에 GitHub self-hosted runner를 등록하고 Workflow를 다음과 같이 설정한다.

```
runs-on: self-hosted
```

### Docker credential helper 오류

#### 증상

```
error getting credentials - err: exit status 1, out: `A specified logon session does not exist. It may already have been terminated.`
```

#### 원인
- WSL2의 Docker config가 Docker Desktop credential helper를 사용하도록 설정되어 있고, self-hosted runner 세션에서 해당 credential helper 접근이 실패한다.

- 문제 설정 예시
```
{
  "credsStore": "desktop.exe"
}
```

#### 해결

```
mkdir -p ~/.docker

cat > ~/.docker/config.json <<'EOF'
{
  "auths": {}
}
EOF
```

### Docker Compose 실행 실패

#### 확인 항목

- Docker Desktop 실행 여부 확인
- Docker Desktop WSL Integration 활성화 여부 확인
- Mini PC WSL2 Ubuntu에서 `docker version` 실행 가능 여부 확인
- `MINI_PC_HOST` Secret 등록 여부 확인
- `docker-compose.gitlab.yml`의 volume 경로 확인

### GitLab Web UI 접근 실패

#### 확인 항목

- GitLab Container 상태 확인
- `docker logs -f gitlab` 확인
- `external_url` 값 확인
- Mini PC 방화벽 8080 포트 허용 여부 확인
- Main PC와 Mini PC 동일 네트워크 여부 확인

---