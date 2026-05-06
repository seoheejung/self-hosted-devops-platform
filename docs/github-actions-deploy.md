# GitHub Actions Deploy

> GitLab 서버가 아직 구축되지 않은 초기 단계에서 GitHub Actions를 사용하여 
> Mini PC에 GitLab Container를 자동 배포하고, GitHub Repository 변경 사항을 SSH 기반으로 Mini PC에 반영한다.

---

## 전체 흐름

```text
Main PC
 └─ Git Push
        ↓

GitHub Repository
 └─ GitHub Actions
        ↓

SSH Deploy
        ↓

Mini PC
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 └─ GitLab Container
```

---

## 전제 조건

- Main PC에서 GitHub Repository로 push 가능
- Mini PC에 OpenSSH Server 활성화
- Main PC에서 Mini PC로 SSH 접속 가능
- Mini PC에서 Docker 명령어 사용 가능
- Mini PC에 프로젝트 작업 디렉토리 존재

---


Main PC
 └─ Git Push
        ↓

GitHub Repository
 └─ GitHub Actions
        ↓

SSH Deploy
        ↓

Mini PC
 ├─ Docker Desktop
 ├─ WSL2 Ubuntu
 └─ GitLab Container
```

---

## 전제 조건

- Main PC에서 GitHub Repository로 push 가능
- Mini PC에 OpenSSH Server 활성화
- Main PC에서 Mini PC로 SSH 접속 가능
- Mini PC에서 Docker 명령어 사용 가능
- Mini PC에 프로젝트 작업 디렉토리 존재

---

## 작업 진행

### 1. Mini PC SSH 접속 확인

#### Main PC에서 실행

```bash
ssh <WINDOWS_USER>@<MINI_PC_IP>
```

#### 예시

```bash
ssh sehan@192.168.0.50
```

#### 정상 기준

```text
Mini PC에 SSH 접속 성공
```

---

### 2. SSH Key 생성

#### Main PC에서 실행

```bash
ssh-keygen -t ed25519 -C "github-actions-mini-pc" -f ~/.ssh/github_actions_mini_pc
```

#### 생성 파일

```text
~/.ssh/github_actions_mini_pc
~/.ssh/github_actions_mini_pc.pub
```

| 파일 | 용도 |
|---|---|
| `github_actions_mini_pc` | GitHub Secrets에 등록할 Private Key |
| `github_actions_mini_pc.pub` | Mini PC에 등록할 Public Key |

---

### 3. Public Key를 Mini PC에 등록

#### Main PC에서 Public Key 확인

```bash
cat ~/.ssh/github_actions_mini_pc.pub
```

출력된 내용을 복사.

#### Mini PC에서 실행

```powershell
mkdir $env:USERPROFILE\.ssh -Force

notepad $env:USERPROFILE\.ssh\authorized_keys
```

`authorized_keys` 파일에 Public Key 내용을 붙여넣고 저장.

#### 권한 설정

```powershell
icacls $env:USERPROFILE\.ssh /inheritance:r

icacls $env:USERPROFILE\.ssh /grant "$env:USERNAME:F"

icacls $env:USERPROFILE\.ssh\authorized_keys /inheritance:r

icacls $env:USERPROFILE\.ssh\authorized_keys /grant "$env:USERNAME:F"
```

---

### 4. SSH Key 기반 접속 확인

#### Main PC에서 실행

```bash
ssh -i ~/.ssh/github_actions_mini_pc <WINDOWS_USER>@<MINI_PC_IP>
```

#### 정상 기준

```text
비밀번호 없이 Mini PC에 SSH 접속 성공
```

---

### 5. GitHub Secrets 등록

#### GitHub Repository 이동

```text
Settings
 → Secrets and variables
 → Actions
 → New repository secret
```

#### 등록 값

| Secret | 값 |
|---|---|
| `MINI_PC_HOST` | Mini PC 내부 IP |
| `MINI_PC_USER` | Mini PC SSH 사용자명 |
| `MINI_PC_SSH_KEY` | `github_actions_mini_pc` Private Key 전체 내용 |

#### Private Key 확인

```bash
cat ~/.ssh/github_actions_mini_pc
```

> 주의:
> Private Key는 Git에 절대 커밋하지 않는다.
> GitHub Secrets에만 등록한다.

---

### 6. Mini PC 작업 디렉토리 준비

#### Mini PC에서 실행

```bash
mkdir -p ~/projects/self-hosted-devops-platform
```

#### 권장 경로

```text
/home/<USER>/projects/self-hosted-devops-platform
```

---

### 7. GitHub Actions Workflow 작성

#### 파일

```text
.github/workflows/deploy-gitlab.yml
```

#### 구성 내용

- GitHub Actions 기반 Workflow 구성
- Git Push 시 Workflow 자동 실행
- GitHub Secrets 기반 SSH 인증 사용
- Mini PC로 compose 파일 복사
- Mini PC에서 Docker Compose 실행
- GitLab Container 상태 확인

#### 사용 기술

- `appleboy/scp-action`
- `appleboy/ssh-action`

---

### 8. Workflow 실행 조건

현재 Workflow는 `main` 브랜치에 push될 때 실행된다.

#### 실행 조건

```yaml
on:
  push:
    branches:
      - main
```

#### 파일 변경 조건

```yaml
paths:
  - "infra/compose/docker-compose.gitlab.yml"
  - ".github/workflows/deploy-gitlab.yml"
```

즉, 해당 파일이 변경되어 `main`에 반영될 때만 배포가 실행된다.

---

### 9. 브랜치 작업 및 PR

#### 작업 브랜치 생성

```bash
git checkout -b feature/gitlab-initial-setup
```

#### 변경 사항 커밋

```bash
git add .

git commit -m "feat: GitLab 초기 실행 환경 구성"
```

#### 원격 브랜치 push

```bash
git push origin feature/gitlab-initial-setup
```

GitHub에서 PR 생성 후 `main`으로 merge한다.

---

### 10. GitHub Actions 실행 확인

#### GitHub Repository

```text
Actions
 → Deploy GitLab to Mini PC
```

#### 정상 기준

- Workflow 실행됨
- Checkout 성공
- SSH 접속 성공
- Docker Compose 실행 성공
- GitLab Container Up 상태 확인

---

### 11. Mini PC 상태 확인

#### Container 상태 확인

```bash
docker ps
```

#### 정상 기준

```text
gitlab container Up
```

#### GitLab 로그 확인

```bash
docker logs -f gitlab
```

#### GitLab 서비스 상태 확인

```bash
docker exec -it gitlab gitlab-ctl status
```

---

### 12. Main PC에서 GitLab Web UI 접근

#### Main PC 브라우저

```text
http://<MINI_PC_IP>:8080
```

#### 예시

```text
http://192.168.0.50:8080
```

#### 정상 기준

```text
GitLab 로그인 페이지 표시
```

---

### 13. 초기 root 비밀번호 확인

#### Mini PC에서 실행

```bash
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

#### 로그인 정보

```text
Username: root
Password: initial_root_password 파일 내용
```

> 주의:
> initial_root_password 파일은 일정 시간이 지나면 삭제될 수 있으므로 초기 실행 후 바로 확인한다.

---

## 완료 기준

- [ ] Main PC에서 Mini PC로 SSH Key 기반 접속 가능
- [ ] GitHub Secrets 등록 완료
- [ ] GitHub Actions Workflow 작성 완료
- [ ] GitHub Actions에서 Mini PC SSH 접속 성공
- [ ] GitHub Actions에서 Docker Compose 실행 성공
- [ ] GitLab Container 정상 실행
- [ ] Main PC에서 GitLab Web UI 접근 가능
- [ ] root 초기 비밀번호 확인 가능
- [ ] GitLab Web UI 로그인 가능

---

## 트러블슈팅

### GitHub Actions에서 SSH 접속 실패

#### 확인 항목

- `MINI_PC_HOST` 값 확인
- `MINI_PC_USER` 값 확인
- `MINI_PC_SSH_KEY` 값 확인
- Mini PC OpenSSH Server 실행 여부 확인
- Mini PC 방화벽 22번 포트 허용 여부 확인

---

### SCP 복사 실패

#### 확인 항목

- target 경로 존재 여부 확인
- SSH 사용자 권한 확인
- GitHub Actions 로그에서 실제 복사 경로 확인

---

### Docker 명령어 실패

#### 확인 항목

- Mini PC에서 Docker Desktop 실행 여부 확인
- SSH 세션에서 `docker version` 실행 가능 여부 확인
- Docker Desktop WSL Integration 활성화 여부 확인

---

### GitLab Web UI 접근 실패

#### 확인 항목

- GitLab Container 상태 확인
- `docker logs -f gitlab` 확인
- `external_url` 값 확인
- Mini PC 방화벽 8080 포트 허용 여부 확인
- Main PC와 Mini PC 동일 네트워크 여부 확인

---