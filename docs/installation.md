# Installation

> Windows 11 기반 Mini PC 환경에서 WSL2와 Docker Desktop을 활용하여 
> GitLab Omnibus Container를 실행하고, GitHub Actions 기반 초기 자동 배포 환경을 구성한다.

---

## 전체 흐름

```text
Main PC
 └─ Git Push
        ↓

GitHub Actions
 └─ SSH Deploy
        ↓

Mini PC
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 └─ GitLab Container
```

---

## 사전 준비

### Mini PC 환경

| 항목 | 내용 |
|---|---|
| OS | Windows 11 |
| WSL | WSL2 Ubuntu |
| Container Runtime | Docker Desktop |
| SSH | OpenSSH Server |

### Main PC 환경

| 항목 | 내용 |
|---|---|
| Git | 설치 필요 |
| SSH Client | 설치 필요 |
| GitHub Account | 필요 |

---

## 작업 진행

### 1. WSL2 설치 및 확인

#### PowerShell 관리자 권한 실행

```powershell
wsl --install
```

설치 완료 후 재부팅.

#### Ubuntu 설치 확인

```powershell
wsl -l -v
```

#### WSL1이면 변환

```powershell
wsl --set-version Ubuntu 2
```

---

### 2. Docker Desktop 설치

#### Docker Desktop 설치

```text
https://www.docker.com/products/docker-desktop/
```

#### WSL2 연동 설정

Docker Desktop:

```text
Settings
 → General
   → Use the WSL 2 based engine 체크

Settings
 → Resources
   → WSL Integration
   → Ubuntu 활성화
```

#### Docker 동작 확인

WSL2 Ubuntu 터미널:

```bash
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

### 4. Mini PC OpenSSH Server 구성

#### OpenSSH Server 설치

- PowerShell 관리자 권한
```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'

Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

#### SSH 서비스 활성화

```powershell
Start-Service sshd

Set-Service -Name sshd -StartupType Automatic
```

#### 방화벽 허용

```powershell
New-NetFirewallRule `
  -Name sshd `
  -DisplayName 'OpenSSH Server' `
  -Enabled True `
  -Direction Inbound `
  -Protocol TCP `
  -Action Allow `
  -LocalPort 22
```

---

### 5. Main PC → Mini PC SSH 접속 확인

#### Mini PC IP 확인

```powershell
ipconfig
```
> 가능하면 공유기 DHCP 예약으로 내부 IP 고정 권장.

#### Main PC에서 SSH 접속

```bash
ssh <WINDOWS_USER>@<MINI_PC_IP>
```
> 정상 기준: Mini PC PowerShell 또는 WSL 터미널 접근 가능

---

### 6. 프로젝트 디렉토리 구성

#### WSL2 Ubuntu 내부에서 작업
```bash
mkdir -p ~/projects/self-hosted-devops-platform

cd ~/projects/self-hosted-devops-platform
```

> 주의: /mnt/c 기반 경로는 GitLab Volume 용도로 비추천.

#### 권장
```text
/home/<USER>/projects/self-hosted-devops-platform
```

---

### 7. GitLab 디렉토리 생성

```bash
mkdir -p infra/compose

mkdir -p gitlab/config
mkdir -p gitlab/data
mkdir -p gitlab/logs
```

---

### 8. GitLab Docker Compose 작성

#### 파일
```text
infra/compose/docker-compose.gitlab.yml
```

#### 구성 내용
- GitLab Omnibus Container 실행
- GitLab HTTP / HTTPS / SSH 포트 매핑
- GitLab Volume 영속성 구성
- GitLab 성능 최적화 옵션 적용
- Main PC에서 접근 가능한 `external_url` 설정

#### 주요 설정
- `puma['worker_processes'] = 1`
- `sidekiq['concurrency'] = 5`
- `postgresql['shared_buffers'] = "256MB"`

#### Volume 구성
```text
gitlab/config
gitlab/logs
gitlab/data
```

> 주의: external_url은 Main PC에서 접근 가능한 Mini PC IP로 설정.

---

#### 9. GitLab Container 실행
```bash
docker compose \
  -f infra/compose/docker-compose.gitlab.yml \
  up -d
```

#### 상태 확인
```bash
docker ps
```

#### 로그 확인
```bash
docker logs -f gitlab
```

> GitLab 초기화는 시간이 오래 걸릴 수 있음.

---

### 10. GitLab Web UI 접근

#### Main PC 브라우저

```text
http://<MINI_PC_IP>:8080
```

> 정상 기준 : GitLab 로그인 페이지 표시

---

#### 11. 초기 root 비밀번호 확인

#### Mini PC에서 실행
```bash
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

#### 로그인 정보
```text
Username: root
Password: initial_root_password 내용
```

> 주의 : initial_root_password 파일은 일정 시간이 지나면 삭제될 수 있음.

---

### 12. 기본 검증

#### Container 상태 확인
```bash
docker ps
```

> 정상 기준 : gitlab container Up 상태

#### GitLab 서비스 상태 확인

```bash
docker exec -it gitlab gitlab-ctl status
```

#### 포트 확인

```bash
docker port gitlab
```

#### Web UI 접근 확인

```text
http://<MINI_PC_IP>:8080
```

---

## 완료 기준

- [ ] WSL2 Ubuntu 정상 실행
- [ ] Docker Desktop WSL2 연동 완료
- [ ] Main PC → Mini PC SSH 접속 성공
- [ ] GitLab Container 정상 실행
- [ ] Main PC 브라우저에서 GitLab Web UI 접근 가능
- [ ] root 초기 비밀번호 확인 가능
- [ ] GitLab 로그인 가능

---
