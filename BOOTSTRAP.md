# ArgoCD App of Apps 최초 등록 가이드

## 사전 조건
- EKS 클러스터가 생성되어 있어야 함
  (kakaotech-eks-infra Terraform 적용 완료)
- AWS Load Balancer Controller 설치 완료
  (Terraform helm 모듈에서 자동 설치)
- ArgoCD 설치 완료
  (Terraform helm 모듈에서 자동 설치)
- kubectl이 EKS 클러스터에 연결되어 있어야 함

## 1단계 — kubectl EKS 연결
```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name <클러스터명>
```

## 2단계 — spring-env-secret 사전 생성
SECRET-SETUP.md 참고

## 3단계 — ArgoCD 접속
```bash
# ArgoCD 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# ArgoCD UI 접속
# https://argocd.<도메인>
```

## 4단계 — App of Apps 등록
```bash
# dev 환경
kubectl apply -f argo-apps/app-of-apps-dev.yaml

# prod 환경
kubectl apply -f argo-apps/app-of-apps-prod.yaml
```

## 5단계 — 동기화 확인
ArgoCD UI에서 apps-root-dev/prod Application이
정상 동기화되는지 확인

## syncWave 배포 순서
Wave -1: EBS CSI Driver
Wave  0: EBS StorageClass, Metrics Server
Wave  1: Elasticsearch, Redis
Wave  2: Kibana, APM Server
Wave  3: Filebeat, Metricbeat
Wave  4: Spring Boot App

## 문제 해결
- Pod가 뜨지 않는 경우:
  kubectl describe pod <pod명> -n <namespace>
- ArgoCD sync 실패 시:
  ArgoCD UI → Application → Sync 상태 확인

## ACM 인증서 설정

### 인증서 요구사항
아래 3개 도메인을 커버하는 ACM 인증서가 필요합니다:
- api.ktbkoco.com
- kibana.ktbkoco.com
- apm.ktbkoco.com

### 권장 방식 — 와일드카드 인증서
*.ktbkoco.com 와일드카드 인증서 1개로
모든 서브도메인을 커버할 수 있습니다.

### 인증서 발급 방법
```bash
aws acm request-certificate \
  --domain-name "*.ktbkoco.com" \
  --validation-method DNS \
  --region ap-northeast-2
```

### 인증서 ARN 교체 위치
발급된 ARN을 아래 파일에 교체:
1. env/prod/values-prod.yaml → ingress.certificateArn
2. env/dev/values-dev.yaml → ingress.certificateArn

### 인증서 커버리지 확인 방법
```bash
aws acm describe-certificate \
  --certificate-arn <ARN> \
  --region ap-northeast-2 \
  --query 'Certificate.SubjectAlternativeNames'
```

## ⚠️ 보안 주의사항 (데모/개발 환경 기준)

### 현재 상태
이 프로젝트는 데모/개발 환경 기준으로 구성되어 있으며
아래 보안 설정이 비활성화 상태입니다.

| 서비스 | 현재 상태 | 프로덕션 권장 |
|--------|---------|------------|
| Elasticsearch | 인증 비활성화 | xpack.security.enabled: true |
| Kibana | 익명 접근 허용 + 인터넷 노출 | OIDC/IP 제한 추가 |
| Redis | 인증 비활성화 | auth.enabled: true |
| 자격증명 | 평문 하드코딩 | Sealed Secrets / ESO 사용 |

### 프로덕션 전환 시 개선 계획
1. Elasticsearch xpack.security 활성화
   - elastic 사용자 비밀번호 Sealed Secrets으로 관리
2. Kibana 인증 추가
   - ALB OIDC 인증 또는 IP 제한 설정
3. Redis 인증 활성화
   - 비밀번호 Sealed Secrets으로 관리
4. External Secrets Operator 도입
   - AWS Secrets Manager 연동으로 자격증명 중앙 관리

## 연관 프로젝트

이 GitOps 리포지토리는 단독으로 동작하지 않습니다.
아래 EKS 인프라 Terraform 프로젝트와 함께 사용합니다.

### kakaotech-eks-infra (EKS 인프라)
- EKS 클러스터 생성
- AWS Load Balancer Controller 설치 (Helm)
- ArgoCD 설치 (Helm)
- VPC, IAM, Security Group, ECR 등 AWS 인프라 구성
- Route53 DNS 레코드 설정

### 전체 구성 순서

1. kakaotech-eks-infra Terraform 적용
   ```bash
   terraform -chdir=koco/environments/prod init
   terraform -chdir=koco/environments/prod apply
   ```

2. EKS 클러스터 연결
   ```bash
   aws eks update-kubeconfig \
     --region ap-northeast-2 \
     --name koco-prod-cluster
   ```

3. spring-env-secret 사전 생성
   (SECRET-SETUP.md 참고)

4. App of Apps 등록 (이 리포지토리)
   ```bash
   kubectl apply -f argo-apps/app-of-apps-prod.yaml
   ```

### 역할 분리
| 항목 | kakaotech-eks-infra | 이 리포(GitOps) |
|------|-------------------|----------------|
| EKS 클러스터 | ✅ | ❌ |
| LB Controller | ✅ (Helm) | ❌ |
| ArgoCD | ✅ (Helm) | ❌ |
| Spring Boot | ❌ | ✅ |
| ELK 스택 | ❌ | ✅ |
| Redis | ❌ | ✅ |
| EBS CSI Driver | ❌ | ✅ (ArgoCD) |

> **참고**: EBS CSI Driver의 IRSA 역할(`AmazonEKS_EBS_CSI_Driver_IRSA`)은
> kakaotech-eks-infra의 Terraform(`koco/modules/iam/main.tf`)에서 생성되지만,
> 실제 드라이버 Helm 배포는 이 리포지토리의 ArgoCD Application
> (`argo-apps/apps-prod/ebs-csi-driver-application.yaml`)이 담당합니다 —
> IAM 리소스 정의와 워크로드 배포가 두 리포지토리에 나뉘어 있는 구조입니다.
