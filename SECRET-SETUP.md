# spring-env-secret 사전 설정 가이드

## 개요
Spring Boot 앱은 `spring-env-secret` Secret을
envFrom으로 참조합니다.
ArgoCD 배포 전에 클러스터에 사전 생성 필요합니다.

## 필요한 환경변수 목록

`apps/base/spring/templates/deployment.yaml`은 아래처럼
`envFrom.secretRef`로 Secret 전체를 참조하며,
개별 키 이름을 명시하지 않습니다:

```yaml
envFrom:
  - secretRef:
      name: spring-env-secret
```

즉, `spring-env-secret`에 들어있는 모든 key-value 쌍이
컨테이너 환경변수로 그대로 주입됩니다. 템플릿 자체에는
구체적인 키 목록이 정의되어 있지 않으므로, 아래 목록은
Spring Boot 앱 운영에 필요할 것으로 예상되는 항목이며
실제 애플리케이션 코드(application.yml 등)를 기준으로
정확한 키 이름과 값을 확정해야 합니다.

- `SPRING_PROFILES_ACTIVE` (예: `prod`)
- 데이터베이스 접속 정보 (예: `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`)
- 카카오 OAuth 클라이언트 정보 (예: `KAKAO_CLIENT_ID`, `KAKAO_CLIENT_SECRET`, `KAKAO_REDIRECT_URI`)
- 기타 앱에서 참조하는 시크릿 값 (JWT 시크릿 등)

> [확인 필요] 위 키 이름은 예시이며, 실제 애플리케이션의
> `application.yml`/`application-prod.yml` 설정을 기준으로
> 정확한 키 이름 목록으로 교체해야 합니다.

## 생성 방법

### 방법 1 — kubectl 직접 생성
```bash
kubectl create secret generic spring-env-secret \
  --from-literal=KEY=VALUE \
  -n <네임스페이스>
```

### 방법 2 — AWS SSM Parameter Store 연동
GitHub Actions deploy.sh 방식으로
SSM에서 값을 가져와 Secret 자동 생성

### 방법 3 — External Secrets Operator (권장)
ESO를 통해 AWS Secrets Manager 또는
SSM Parameter Store와 자동 동기화

## ArgoCD App of Apps 등록 전 체크리스트
- [ ] spring-env-secret 생성 완료
- [ ] EBS StorageClass 확인
- [ ] ACM 인증서 ARN 확인
