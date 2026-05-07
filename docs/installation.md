# Installation

> Windows 11 기반 Mini PC에서 WSL2와 Docker Desktop을 구성하고, GitLab Omnibus Container를 실행할 수 있는 기본 환경을 준비한다.


---

## 전체 흐름

```text
Mini PC - Windows 11
 ├─ WSL2 Ubuntu
 ├─ Docker Desktop
 ├─ Docker Compose
 └─ GitLab Container 실행 준비

Main PC
 └─ GitLab Web UI 접근 확인
```

> GitHub Actions 자동 배포 구성은 docs/github-actions-deploy.md에서 다룬다.

---

## 사전 준비

### Mini PC 환경

| 항목                | 내용              |
| ----------------- | --------------- |
| OS                | Windows 11      |
| Linux Runtime     | WSL2 Ubuntu     |
| Container Runtime | Docker Desktop  |
| Network           | Main PC와 동일 내부망 |



### Main PC 환경

| 항목      | 내용                  |
| ------- | ------------------- |
| Browser | GitLab Web UI 접근 확인 |
| Git     | Repository 작업용      |


---

## 작업 진행

### 1. WSL2 설치 및 확인

#### PowerShell 관리자 권한 실행
```
wsl --install
```
- 설치 완료 후 재부팅

#### Ubuntu 설치 확인
```
wsl -l -v
```

#### 정상 기준
```
NAME              STATE           VERSION
* Ubuntu-22.04     Running         2
  docker-desktop   Running         2
```

#### WSL1이면 변환
```
wsl --set-version Ubuntu-22.04 2
```

---

### 2. Docker Desktop 설치

#### Docker Desktop 설치

```text
https://www.docker.com/products/docker-desktop/
```

#### WSL2 연동 설정

```text
# Docker Desktop

Settings
 → General
   → Use the WSL 2 based engine 체크

Settings
 → Resources
   → WSL Integration
   → Ubuntu-22.04 활성화
```

#### Docker 동작 확인
```bash
# WSL2 Ubuntu 터미널

docker version
docker compose version
docker run hello-world
```

---

### 3. Docker 리소스 제한 설정

#### Windows 사용자 홈 경로

```text
C:\Users\<WINDOWS_USER>\.wslconfig
```

#### 내용

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
localhostForwarding=true
```

#### 적용
```powershell
wsl --shutdown
```
- Docker Desktop 재시작

---

### 4. Mini PC 내부망 접근 확인

#### Mini PC의 내부 IP 확인
```
ipconfig
```

#### Main PC에서 Mini PC 도달성 확인
```
ping <MINI_PC_IP>
```
- 정상 기준: Mini PC에서 응답 있음

> 가능하면 공유기 DHCP 예약으로 Mini PC 내부 IP를 고정한다.
> 현재 구성에서는 `MINI_PC_HOST` GitHub Secret에 이 IP를 등록한다.

---

### 5. GitLab runtime volume 경로 준비

> GitLab runtime 데이터는 WSL2 Ubuntu 내부 고정 경로에 저장한다.

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

> Repository checkout 경로나 `/mnt/c`, `/mnt/d` 경로는 GitLab runtime volume 용도로 사용하지 않는다.
> 

---

### 6. GitLab Docker Compose 작성

- 파일: `infra/compose/docker-compose.gitlab.yml`

#### 구성 기준
| 항목 | 내용 |
| --- | --- |
| Image | `gitlab/gitlab-ce` |
| HTTP Port | `8080:80` |
| HTTPS Port | `8443:443` |
| Git SSH Port | `2222:22` |
| external_url | `http://${MINI_PC_HOST}:8080` |
| Volume | `/home/<USER>/gitlab` 하위 절대 경로 |

#### 주요 튜닝 값
```
puma['worker_processes'] = 1
sidekiq['concurrency'] = 5
postgresql['shared_buffers'] = "256MB"
```

---

### 7. GitLab Container 수동 실행 확인

> 자동 배포 전, Mini PC WSL2 Ubuntu에서 Docker Compose 실행 가능 여부를 확인한다.

```
docker compose -f infra/compose/docker-compose.gitlab.yml up -d
```

#### 상태 확인
```
docker ps
```

#### 로그 확인
```
docker logs -f gitlab
```

> GitLab 초기화는 시간이 오래 걸릴 수 있다.

---

---

## 다음 단계

GitHub Actions self-hosted runner 기반 자동 배포 구성은 아래 문서에서 진행한다.

```text
docs/github-actions-deploy.md
```