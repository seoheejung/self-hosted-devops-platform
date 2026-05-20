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
│  │  ├─ gitlab.local.crt
│  │  └─ gitlab.local.key
│  └─ nginx.conf
│
└─ docs/
   └─ reverse-proxy-ssl.md
```

---

## 작업 진행

### 1. hosts 설정

Main PC에서 `gitlab.local`을 Mini PC IP로 연결한다.

- Windows hosts 파일 수정
- `172.30.1.69 gitlab.local` 추가
- `ping gitlab.local`로 확인


### 2. SSL 인증서 생성

Mini PC WSL2 Ubuntu에서 `gitlab.local`용 자체 서명 인증서를 생성한다.

- 인증서 저장 경로: `nginx/ssl/`
- 인증서 파일: `gitlab.local.crt`
- Private key 파일: `gitlab.local.key`
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
- `nginx/nginx.conf` mount
- `nginx/conf.d` mount
- `nginx/ssl` mount
- GitLab Container와 같은 Docker network 사용
- 기존 `8080` 직접 접근 경로 유지


### 5. GitLab external_url 검토

GitLab은 `external_url`을 기준으로 Web UI 링크, redirect URL, clone URL을 생성한다.

먼저 Nginx Reverse Proxy 접근을 검증한다.

> `external_url` 변경은 GitLab Web UI, clone URL, GitLab Runner 통신에 영향을 줄 수 있으므로 바로 적용하지 않고 별도 검토한다.

---

## 검증 기준

### Nginx 기준

- Nginx Container 실행
- Nginx config 문법 정상
- `http://gitlab.local` 접근 시 HTTPS redirect 확인
- `https://gitlab.local` 접근 가능

### GitLab 기준

- GitLab 로그인 페이지 표시
- root 로그인 가능
- Project 목록 접근 가능
- 기존 `http://172.30.1.67:8080` 접근 유지

### Runner 기준

- GitLab Runner 상태 `Online` 또는 `Idle` 유지
- 기존 Pipeline 실행 영향 없음

---

## 주의사항

- `nginx/ssl/*.key`, `nginx/ssl/*.crt`, `nginx/ssl/*.pem` 파일은 Git에 commit하지 않는다.
- 자체 서명 인증서는 브라우저 신뢰 경고가 발생할 수 있다.
- 기존 `http://172.30.1.67:8080` 접근 경로는 복구 경로로 유지한다.
- 외부 인터넷 공개가 아니라 내부망 HTTPS 접근만 검증한다.

---