## 작업 내용
<!-- 이번 PR에서 실제로 수행한 작업을 작성 -->

- 

---

## 변경 요약
<!-- 변경된 파일과 핵심 변경 사항을 작성 -->

- 

---

## 구현 단계
<!-- 이번 PR과 직접 관련 있는 Phase에 체크 -->

- [ ] Phase 1. Mini PC 기본 환경 구성
- [ ] Phase 2. GitHub Actions 기반 초기 배포 구성
- [ ] Phase 3. GitLab 서버 구축
- [ ] Phase 4. GitLab Runner 분리 구성
- [ ] Phase 5. CI/CD Pipeline 구성
- [ ] Phase 6. GitLab 운영 안정성 검증
- [ ] Phase 7. Reverse Proxy 및 SSL 구성
- [ ] Phase 8. Monitoring 구성
- [ ] Phase 9. Backup / 운영 문서화

---

## 기본 검증

- [ ] Main PC ↔ Mini PC 네트워크 접근 확인
- [ ] 기존 구성 영향 없음
- [ ] 불필요 파일 제외 확인 (`.env`, logs, runtime data 등)
- [ ] 민감 정보 포함 없음

---

## GitHub Actions 검증
<!-- GitHub Actions Workflow 또는 초기 배포 구조 변경 시 체크 -->

- [ ] GitHub Actions Workflow 정상 실행 확인
- [ ] GitHub Secrets 정상 적용 확인
- [ ] GitHub self-hosted runner 상태 확인
- [ ] Git Push 또는 PR Merge 이후 자동 배포 흐름 확인
- [ ] Workflow `paths` 변경 영향 확인
- [ ] Mini PC runtime 파일 준비 step 정상 실행 확인

---

## GitLab 검증
<!-- GitLab Container 또는 GitLab 설정 변경 시 체크 -->

- [ ] GitLab Container 정상 실행 확인
- [ ] GitLab 주요 서비스 `run` 상태 확인
- [ ] Main PC 브라우저에서 GitLab Web UI 접근 확인
- [ ] GitLab volume mount 경로 확인
- [ ] GitLab 데이터 영속성 유지 확인
- [ ] 기존 복구 경로 접근 확인 (`http://172.30.1.67:8080`)

---

## GitLab Runner 검증
<!-- GitLab Runner 또는 Pipeline 관련 변경 시 체크 -->

- [ ] GitLab Runner 정상 등록 확인
- [ ] Runner 상태 `Online` 또는 `Idle` 확인
- [ ] Runner tag 확인
- [ ] 테스트 Pipeline 실행 확인
- [ ] Pipeline `Passed` 확인
- [ ] Runner URL 영향 확인

---

## Docker / 인프라 검증

- [ ] Docker Compose config 검증
- [ ] Docker Compose 정상 실행 확인
- [ ] Container 실행 상태 확인
- [ ] Volume 정상 마운트 확인
- [ ] WSL2 환경 정상 동작 확인
- [ ] Docker Desktop 실행 및 WSL Integration 확인

---

## Reverse Proxy / SSL 검증
<!-- Nginx, HTTPS, gitlab.local 관련 변경 시 체크 -->

- [ ] Nginx Reverse Proxy Container 정상 실행 확인
- [ ] Nginx 80/443 포트 publish 확인
- [ ] Nginx 설정 문법 확인 (`nginx -t`)
- [ ] `gitlab.local` hosts 해석 확인
- [ ] HTTP → HTTPS redirect 확인
- [ ] `https://gitlab.local` 접근 확인
- [ ] 자체 서명 인증서 경고 확인
- [ ] `/home/gali/nginx` runtime 경로 확인
- [ ] SSL 인증서 / private key Git 제외 확인
- [ ] GitLab `external_url` 변경 여부 검토

---

## Monitoring 검증
<!-- Prometheus, Grafana, exporter 관련 변경 시 체크 -->

- [ ] Prometheus 메트릭 수집 확인
- [ ] Grafana Dashboard 정상 표시 확인
- [ ] Host / Container 리소스 모니터링 확인
- [ ] GitLab 상태 메트릭 확인
- [ ] GitLab Runner 상태 메트릭 확인
- [ ] Nginx 상태 메트릭 확인

---

## 배포 / 안정성 검증

- [ ] Container 재시작 후 정상 복구 확인
- [ ] GitLab runtime data 유지 확인
- [ ] Nginx runtime file 유지 확인
- [ ] 장애 발생 시 영향 범위 확인
- [ ] 롤백 또는 수동 복구 방법 확인

---

## 보안 검증

- [ ] Token, Secret, Password 등 민감 정보 미포함
- [ ] SSL private key 미포함
- [ ] 로그에 민감 정보 출력 없음
- [ ] 외부 노출 포트 검토
- [ ] Docker socket 사용 여부 검토

---

## 해당 시 검증
<!-- 이번 PR에 해당되는 경우만 체크 -->

### SSH / 원격 접근

- [ ] Main PC → Mini PC SSH 접속 확인
- [ ] OpenSSH Server 정상 동작 확인
- [ ] 방화벽 포트 허용 확인

### Git 인증 / Repository 접근

- [ ] GitHub push 정상 확인
- [ ] GitLab push 정상 확인
- [ ] GitLab Personal Access Token 필요 여부 확인
- [ ] Protected branch 정책 영향 확인

### Backup / Restore

- [ ] GitLab backup 생성 확인
- [ ] GitLab restore 절차 확인
- [ ] `/home/gali/gitlab` runtime data 백업 대상 확인
- [ ] `/home/gali/nginx` runtime file 백업 대상 확인

---

## 결과 검증
<!-- 실제 확인한 결과를 작성 -->

- 

---

## 참고 사항
<!-- 리뷰어 또는 미래의 내가 알아야 할 내용 작성 -->

- 