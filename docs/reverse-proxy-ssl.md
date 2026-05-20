# Reverse Proxy 및 SSL 구성

> GitLab Container 앞단에 Nginx Reverse Proxy를 구성하고, 내부망에서 HTTPS 기반 GitLab 접근을 검증한다.

---

## 목적

현재 GitLab Web UI는 Mini PC 내부 IP와 GitLab Container 포트를 직접 사용하여 접근한다.

```text
Main PC
 → http://172.30.1.67:8080
 → GitLab Container
```

GitLab 앞단에 Nginx Reverse Proxy를 두고, `gitlab.local` 도메인으로 접근하는 구조를 구성한다.

```
Main PC
 → https://gitlab.local
 → Nginx Reverse Proxy
 → GitLab Container
```

기존 `http://172.30.1.67:8080` 접근 경로는 복구 경로로 유지한다.

---

## 전제 조건

- GitLab Container 실행 중
- GitLab Web UI 기존 주소 접근 가능
- Docker Desktop 실행 중
- WSL2 Ubuntu 실행 중
- Main PC에서 hosts 파일 수정 가능
- GitLab Runner 상태 유지 필요

---

## 구성 기준

| 항목 | 내용 |
| --- | --- |
| Reverse Proxy | Nginx |
| 접근 도메인 | `gitlab.local` |
| HTTPS | 자체 서명 인증서 |
| 외부 접근 | `https://gitlab.local` |
| 내부 전달 | `http://gitlab:80` |
| 기존 복구 경로 | `http://172.30.1.67:8080` |

---

## 디렉토리 구조

```
self-hosted-devops-platform
├─ infra/
│  └─ compose/
│     └─ docker-compose.gitlab.yml
│
├─ nginx/
│  ├─ conf.d/
│  │  └─ gitlab.conf
│  ├─ ssl/
│  │  └─ .gitkeep
│  └─ nginx.conf
│
└─ docs/
   └─ reverse-proxy-ssl.md
```

`nginx/ssl/` 디렉토리는 Repository 구조 표시용으로만 유지한다.

`gitlab.local.crt`, `gitlab.local.key`는 Git에 commit하지 않는다.

실제 인증서와 Nginx runtime 파일은 Mini PC WSL2 내부 고정 경로 `/home/gali/nginx/` 아래에 생성한다.

---

## 실제 적용 기준

Phase 7에서 Nginx Reverse Proxy는 GitHub Actions가 실행하는 `docker compose` 기준으로 적용된다.

```text
GitHub Repository
 → main merge
 → GitHub Actions 실행
 → Mini PC self-hosted runner
 → GitHub Repository checkout
 → /home/gali/nginx runtime 파일 준비
 → docker compose up -d
 → GitLab / Nginx Container 반영
```

GitHub Repository의 `nginx/` 디렉토리는 Nginx 설정 원본이다.

실제 Nginx Container는 GitHub Actions workspace를 직접 mount하지 않고, Mini PC WSL2 내부 고정 runtime 경로인 `/home/gali/nginx`를 mount한다.

| 파일 | 위치 | Git 포함 여부 | 역할 |
| --- | --- | --- | --- |
| `nginx/nginx.conf` | GitHub Repository | 포함 | Nginx 전역 설정 원본 |
| `nginx/conf.d/gitlab.conf` | GitHub Repository | 포함 | GitLab Reverse Proxy 설정 원본 |
| `infra/compose/docker-compose.gitlab.yml` | GitHub Repository | 포함 | GitLab / Nginx Container 구성 |
| `.github/workflows/deploy-gitlab.yml` | GitHub Repository | 포함 | GitHub Actions 배포 Workflow |
| `/home/gali/nginx/nginx.conf` | Mini PC Runtime | 제외 | Nginx Container mount 대상 |
| `/home/gali/nginx/conf.d/gitlab.conf` | Mini PC Runtime | 제외 | Nginx Container mount 대상 |
| `/home/gali/nginx/ssl/gitlab.local.crt` | Mini PC Runtime | 제외 | HTTPS 인증서 |
| `/home/gali/nginx/ssl/gitlab.local.key` | Mini PC Runtime | 제외 | HTTPS private key |

`nginx/ssl/*.key`, `nginx/ssl/*.crt`, `nginx/ssl/*.pem` 파일은 Git에 commit하지 않는다.

Repository 안의 `nginx/ssl/`은 실제 인증서 저장 위치로 사용하지 않는다.

---

## 작업 진행

### 1. hosts 설정

Main PC에서 `gitlab.local`을 Mini PC IP로 연결한다.

- Windows hosts 파일 수정
- `172.30.1.67 gitlab.local` 추가
- `ping gitlab.local`로 확인


### 2. Nginx runtime 파일 및 SSL 인증서 준비

GitHub Repository의 `nginx/` 설정 파일은 원본이다.

GitHub Actions는 해당 설정 파일을 Mini PC runtime 경로 `/home/gali/nginx/`로 복사한다.

인증서가 없으면 GitHub Actions 실행 중 `/home/gali/nginx/ssl/` 아래에 자체 서명 인증서를 생성한다.

- Nginx runtime 경로: `/home/gali/nginx/`
- 인증서 저장 경로: `/home/gali/nginx/ssl/`
- 인증서 파일: `/home/gali/nginx/ssl/gitlab.local.crt`
- Private key 파일: `/home/gali/nginx/ssl/gitlab.local.key`
- Private key는 Git에 commit하지 않는다.


### 3. Nginx 설정 작성

`nginx/conf.d/gitlab.conf` 파일을 작성한다.

설정 기준:

- HTTP 요청은 HTTPS로 redirect
- HTTPS 요청은 GitLab Container로 proxy
- `server_name`은 `gitlab.local` 사용
- GitLab Web UI 사용을 위해 proxy header 설정
- 업로드 제한 문제를 피하기 위해 body size 제한 완화


### 4. Docker Compose 반영

`infra/compose/docker-compose.gitlab.yml`에 Nginx Container를 추가한다.

구성 기준:

- Nginx image 사용
- Host `80`, `443` 포트 사용
- `/home/gali/nginx/nginx.conf` → `/etc/nginx/nginx.conf` mount
- `/home/gali/nginx/conf.d` → `/etc/nginx/conf.d` mount
- `/home/gali/nginx/ssl` → `/etc/nginx/ssl` mount
- GitLab Container와 같은 Docker network 사용
- 기존 `8080` 직접 접근 경로 유지

### 5. GitHub Actions Workflow 반영

`.github/workflows/deploy-gitlab.yml`에 Nginx 설정 변경 감지를 추가한다.

```yaml
paths:
  - "infra/compose/docker-compose.gitlab.yml"
  - "nginx/**"
  - ".github/workflows/deploy-gitlab.yml"
```

GitHub Actions는 Nginx 설정 원본을 Mini PC runtime 경로로 복사한다.

```
mkdir -p /home/gali/nginx/conf.d
mkdir -p /home/gali/nginx/ssl

cp nginx/nginx.conf /home/gali/nginx/nginx.conf
cp nginx/conf.d/gitlab.conf /home/gali/nginx/conf.d/gitlab.conf
```

인증서가 없으면 자체 서명 인증서를 생성한다.

```
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /home/gali/nginx/ssl/gitlab.local.key \
  -out /home/gali/nginx/ssl/gitlab.local.crt \
  -subj "/CN=gitlab.local"
```

### 6. GitLab external_url 검토

Phase 7에서는 GitLab `external_url`을 `https://gitlab.local`로 변경하지 않는다.

현재는 기존 복구 경로와 Runner 통신 안정성을 우선하여 다음 값을 유지한다.

```text
external_url 'http://${MINI_PC_HOST}:8080'
```

> https://gitlab.local 전환은 clone URL, Runner URL, 자체 서명 인증서 신뢰 설정까지 함께 검토해야 하므로 별도 단계에서 진행한다.

---

## 검증 기준

### Nginx 기준

- `gitlab-nginx` Container 실행 확인
- Host 80/443 포트 publish 확인

```text
80/tcp -> 0.0.0.0:80
443/tcp -> 0.0.0.0:443
```

- Nginx config 문법 확인 완료

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Main PC 기준

- `gitlab.local` hosts 해석 확인
```
Ping gitlab.local [172.30.1.67]
손실 = 0%
```

- HTTP → HTTPS redirect 확인
```
HTTP/1.1 301 Moved Permanently
Location: https://gitlab.local/
```

- HTTPS GitLab Web UI 접근 확인
```
https://gitlab.local
```

### Runner 기준

- GitLab Runner 검증용 Pipeline 실행 완료
- Pipeline `Passed` 확인
- Runner URL은 기존 복구 경로 유지

```
url = "http://172.30.1.67:8080"
```

### GitLab 기준

- GitLab 로그인 페이지 표시
- root 로그인 가능
- Project 목록 접근 가능
- 기존 `http://172.30.1.67:8080` 접근 유지

---

## 주의사항

- `nginx/ssl/*.key`, `nginx/ssl/*.crt`, `nginx/ssl/*.pem` 파일은 Git에 commit하지 않는다.
- 자체 서명 인증서는 브라우저 신뢰 경고가 발생할 수 있다.
- 기존 `http://172.30.1.67:8080` 접근 경로는 복구 경로로 유지한다.
- 외부 인터넷 공개가 아니라 내부망 HTTPS 접근만 검증한다.

---