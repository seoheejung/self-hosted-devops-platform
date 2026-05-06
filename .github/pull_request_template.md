## 작업 내용
<!-- 이번 PR에서 실제로 수행한 작업 작성 -->
- 

---

## 변경 요약
- 

---

## 구현 단계 (현재 위치 체크)

- [ ] Phase 1. Mini PC 기본 환경 구성
- [ ] Phase 2. GitLab 서버 구축
- [ ] Phase 3. GitLab Runner 분리 구성
- [ ] Phase 4. CI/CD Pipeline 구성
- [ ] Phase 5. Reverse Proxy 및 SSL 구성
- [ ] Phase 6. Monitoring 구성
- [ ] Phase 7. 운영 문서화

---

## 체크리스트

### 기본 검증
- [ ] 로컬 실행 확인
- [ ] 기존 구성 영향 없음
- [ ] 불필요 파일 제외 (.env, logs, data 등)

### GitLab / Runner 검증
- [ ] GitLab Container 정상 실행 확인
- [ ] GitLab Web UI 접근 확인
- [ ] GitLab Runner 정상 등록 확인
- [ ] Pipeline 정상 실행 확인

### Docker / 인프라 검증
- [ ] Docker Compose 정상 동작 확인
- [ ] Container 간 네트워크 통신 확인
- [ ] Volume 정상 마운트 확인
- [ ] WSL2 환경 정상 동작 확인

### Reverse Proxy / SSL 검증
- [ ] Nginx Reverse Proxy 정상 동작 확인
- [ ] HTTPS 적용 여부 확인
- [ ] 인증서 갱신 방식 확인

### Monitoring 검증
- [ ] Prometheus 메트릭 수집 확인
- [ ] Grafana Dashboard 정상 표시 확인
- [ ] Host / Container 리소스 모니터링 확인

### 성능 / 리소스 검토
- [ ] CPU 사용량 검토
- [ ] Memory 사용량 검토
- [ ] Docker Desktop 리소스 제한 확인
- [ ] GitLab 성능 영향 검토

### 배포 / 안정성
- [ ] 롤백 가능 여부 확인
- [ ] 장애 발생 시 영향 범위 파악
- [ ] 데이터 영속성 유지 여부 확인

### 보안
- [ ] 민감 정보 포함 없음
- [ ] 로그에 민감 정보 출력 없음
- [ ] 외부 노출 포트 검토 완료

---

## 성능 / 결과 검증
- [ ] Git Push → Pipeline 흐름 확인
- [ ] Build / Test / Deploy 정상 수행 확인
- [ ] Container 재시작 후 정상 복구 확인
- [ ] 이전 대비 성능 변화 확인