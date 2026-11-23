# k8s-deploy-was

PetClinic의 **WAS(Spring Boot REST API)** 와 **DB(MySQL 8.4)** 를 Kubernetes에 올리는 Helm 차트입니다.
WEB(React + Nginx)과 입구(Envoy Gateway)는 [k8s-deploy-web](https://github.com/kdt-6-aws-3/k8s-deploy-web)에 있습니다.

배포 환경은 두 가지입니다.

| | kubeadm (3노드) | EKS (Auto Mode) |
|---|---|---|
| DB | 클러스터 안 `container-project-db` (StatefulSet + local-path PVC) | RDS MySQL Multi-AZ + RDS Proxy |
| DB 인증 | 비밀번호 (Secret) | IAM 토큰 (IRSA, 15분짜리) + TLS |
| 이미지 | GHCR `container-project-was:0.0.3` | ECR `was:0.0.4` (aws-advanced-jdbc-wrapper 포함) |
| Pod 분산 | 노드(hostname) 기준 | AZ(2a · 2c) 기준 |
| values | `values.yaml` | `values.yaml` + `values-eks.yaml` |

## 구성

```
.
├── petclinic-namespace.yaml        # namespace petclinic (PSA restricted)
├── container-project-db/           # MySQL StatefulSet (kubeadm 전용)
├── container-project-was/          # Spring Boot Deployment · Service · PDB · ConfigMap · Secret · SA
│   ├── values.yaml                 # 기본값 (kubeadm)
│   └── values-eks.yaml             # EKS 덮어쓰기
└── eks/
    └── storage-efs.yaml            # EFS StorageClass(efs-sc) + RWX PVC (EKS 전용)
```

## 공통 보안 설정

- namespace에 Pod Security Admission `restricted`를 강제합니다 (non-root, 권한 상승 금지, capability 전부 제거).
- Pod마다 전용 ServiceAccount를 쓰고, Kubernetes API 토큰은 마운트하지 않습니다.
- PDB `minAvailable: 1`, `preStop` 10초 대기로 rollout · drain 중에도 요청이 끊기지 않게 합니다.

## kubeadm 배포

```bash
kubectl apply -f petclinic-namespace.yaml

# DB: 비밀번호는 파일에 적지 않고 설치할 때 넣는다
helm upgrade --install db ./container-project-db -n petclinic \
  --set auth.rootPassword='<root 비밀번호>' --set auth.password='<앱 비밀번호>'

# WAS: 빈 DB에 처음 설치할 때만 샘플 데이터를 넣는다 (1개로 띄운다)
helm upgrade --install was ./container-project-was -n petclinic \
  --set datasource.password='<앱 비밀번호>' \
  --set config.sqlInitMode=always --set replicaCount=1

# 데이터가 들어가면 평상시 설정(never, 2개)으로 바꾼다
helm upgrade was ./container-project-was -n petclinic --set datasource.password='<앱 비밀번호>'
```

## EKS 배포

### 먼저 준비할 AWS 리소스

- VPC: public 2개 (`kubernetes.io/role/elb=1`), private 4개 (EKS · EFS용 2개, RDS용 2개), Regional NAT
- EKS Auto Mode 클러스터, IAM OIDC provider
- RDS MySQL Multi-AZ (DB `petclinic`) + RDS Proxy (IAM 인증 REQUIRED, TLS 필수)
- DB 사용자 `petclinic_app` (Proxy 백엔드용 Secrets Manager secret)
- IAM 역할
  - WAS용: `rds-db:connect` 허용, 신뢰 정책 sub = `system:serviceaccount:petclinic:was`
  - EFS CSI용: `AmazonEFSCSIDriverPolicy`
- EFS + 마운트 타깃 (private 2개), EKS 애드온 `aws-efs-csi-driver`
- 보안 그룹 체인: 클러스터 SG → Proxy SG(3306) → RDS SG(3306), 클러스터 SG → EFS SG(2049)
- 모든 리소스에 태그 `Project=k8s-personal`

### 설치

```bash
kubectl apply -f petclinic-namespace.yaml
kubectl apply -f eks/storage-efs.yaml

# 빈 RDS에 처음 한 번만: 샘플 데이터 적재
# application-mysql.properties의 socketTimeout(10초)으로는 큰 INSERT가 끊기므로 이번에만 늘린다
helm upgrade --install was ./container-project-was -n petclinic -f container-project-was/values-eks.yaml \
  --set config.sqlInitMode=always --set replicaCount=1 \
  --set-string config.javaOpts="-XX:MaxRAMPercentage=75 -Dspring.datasource.hikari.data-source-properties.socketTimeout=1800000"

# 이후 평상시 (never, 2개, 기본 JAVA_OPTS)
helm upgrade was ./container-project-was -n petclinic -f container-project-was/values-eks.yaml
```

`values-eks.yaml`의 `datasource.password: "iam-auth"`는 차트의 필수값 검사를 통과시키기 위한 자리표시 값입니다.
실제 접속할 때는 JDBC wrapper의 `iam` 플러그인이 IRSA 자격 증명으로 만든 토큰을 비밀번호 자리에 넣습니다.

### 확인

```bash
kubectl -n petclinic get pod -o wide
kubectl -n petclinic exec deploy/was -- wget -qO- http://127.0.0.1:8080/actuator/health
kubectl -n petclinic exec deploy/was -- wget -qO- "http://127.0.0.1:8080/api/owners?size=1"
```

## EFS 참고

`eks/storage-efs.yaml`의 PVC `petclinic-shared`는 Bound 상태로 준비만 해 두었습니다.
지금 앱에는 파일을 저장하는 경로가 없어서 어떤 Pod에도 마운트하지 않습니다.

## 정리 (비용)

발표 · 실습이 끝나면 다음을 지웁니다: EKS 노드, NAT Gateway, RDS(Multi-AZ), RDS Proxy, EFS, NLB.
