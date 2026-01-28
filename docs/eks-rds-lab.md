# EKS + RDS 실습 가이드 (UI/CLI 포함)

아래 순서대로 진행하면 처음부터 다시 해도 동일하게 성공합니다.  
각 단계는 **(UI)** 또는 **(CLI)**로 표시했고, **모든 명령어에 1줄 설명**을 붙였습니다.

---

## 0) 준비 사항

### (CLI) 로컬/권한 확인
```powershell
aws sts get-caller-identity
# AWS 자격증명이 정상인지 확인합니다.

docker info
# Docker 데몬이 실행 중인지 확인합니다.

kubectl version --client
# kubectl 설치 상태를 확인합니다.
```

---

## 1) EKS 클러스터 생성 (Auto Mode 기본값)

### (CLI) 기본 VPC/서브넷 확인
```powershell
aws ec2 describe-vpcs --region ap-northeast-2 --filters Name=isDefault,Values=true
# 기본 VPC ID를 확인합니다.

aws ec2 describe-subnets --region ap-northeast-2 --filters Name=vpc-id,Values=vpc-033db993991ff1afb --query "Subnets[].SubnetId"
# 기본 VPC의 서브넷 ID를 확인합니다.
```

### (CLI) EKS Auto Mode 클러스터 생성
```powershell
aws eks create-cluster --name metacoding --region ap-northeast-2 --role-arn arn:aws:iam::842903729788:role/AmazonEKSAutoClusterRole --resources-vpc-config "subnetIds=subnet-046fe614a98df0213,subnet-0619800e20e87a1b0,subnet-03d68ec8256612d22,subnet-0c1a266f153fc2f5e,endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=0.0.0.0/0" --access-config "authenticationMode=API,bootstrapClusterCreatorAdminPermissions=true" --compute-config "enabled=true,nodePools=general-purpose,system,nodeRoleArn=arn:aws:iam::842903729788:role/AmazonEKSAutoNodeRole" --storage-config "blockStorage={enabled=true}" --kubernetes-network-config "elasticLoadBalancing={enabled=true}"
# EKS Auto Mode 클러스터를 생성합니다.

aws eks wait cluster-active --name metacoding --region ap-northeast-2
# 클러스터가 ACTIVE 상태가 될 때까지 대기합니다.

aws eks update-kubeconfig --name metacoding --region ap-northeast-2
# 로컬 kubeconfig에 클러스터를 등록합니다.

kubectl get nodes
# 노드가 생성되고 Ready인지 확인합니다.
```

---

## 2) ECR 리포지토리 생성 + 이미지 푸시

### (CLI) 리포지토리 생성
```powershell
aws ecr create-repository --repository-name metacoding-db --region ap-northeast-2
# DB 이미지용 ECR 리포지토리 생성.

aws ecr create-repository --repository-name metacoding-backend --region ap-northeast-2
# 백엔드 이미지용 ECR 리포지토리 생성.

aws ecr create-repository --repository-name metacoding-frontend --region ap-northeast-2
# 프론트 이미지용 ECR 리포지토리 생성.

aws ecr create-repository --repository-name metacoding-redis --region ap-northeast-2
# Redis 이미지용 ECR 리포지토리 생성.
```

### (CLI) 로그인, 빌드, 태그, 푸시
```powershell
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com
# ECR 로그인(도커 푸시 권한 획득).

docker build -t metacoding/db:1 ./db
# DB 이미지를 로컬에서 빌드합니다.

docker build -t metacoding/backend:1 ./backend
# 백엔드 이미지를 로컬에서 빌드합니다.

docker build -t metacoding/frontend:1 ./frontend
# 프론트 이미지를 로컬에서 빌드합니다.

docker build -t metacoding/redis:1 ./redis
# Redis 이미지를 로컬에서 빌드합니다.

docker tag metacoding/db:1 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-db:1
# DB 이미지를 ECR 태그로 변경합니다.

docker tag metacoding/backend:1 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-backend:1
# 백엔드 이미지를 ECR 태그로 변경합니다.

docker tag metacoding/frontend:1 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-frontend:1
# 프론트 이미지를 ECR 태그로 변경합니다.

docker tag metacoding/redis:1 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-redis:1
# Redis 이미지를 ECR 태그로 변경합니다.

docker push 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-db:1
# DB 이미지를 ECR로 푸시합니다.

docker push 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-backend:1
# 백엔드 이미지를 ECR로 푸시합니다.

docker push 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-frontend:1
# 프론트 이미지를 ECR로 푸시합니다.

docker push 842903729788.dkr.ecr.ap-northeast-2.amazonaws.com/metacoding-redis:1
# Redis 이미지를 ECR로 푸시합니다.
```

---

## 3) RDS 생성/설정

### (UI) RDS 생성
1. **RDS 콘솔 → 데이터베이스 생성**
2. 엔진: **MySQL**
3. 마스터 사용자 이름: **admin**
4. 비밀번호: **metacoding** (실습 편의, 실제 환경은 강력한 비밀번호 권장)
5. 퍼블릭 액세스: **예**
6. VPC: **EKS와 동일한 VPC**
7. 보안 그룹: 아래 인바운드 룰 설정

### (UI) RDS 보안 그룹 인바운드
- **TCP 3306** 허용
  - **소스: EKS 노드/클러스터 보안 그룹**
  - 예: `eks-cluster-sg-metacoding-...`

### (UI) 엔드포인트 확인
RDS 상세 화면의 **엔드포인트**를 복사합니다.  
예: `metacoding-db.cfcouus4ix90.ap-northeast-2.rds.amazonaws.com`

---

## 4) 백엔드 설정 변경 (RDS 연결)

### (CLI) ConfigMap/Secret 수정
```powershell
# k8s/backend/backend-configmap.yml
# SPRING_DATASOURCE_URL을 RDS 엔드포인트/DB로 변경합니다.
# 예시:
# jdbc:mysql://metacoding-db.cfcouus4ix90.ap-northeast-2.rds.amazonaws.com:3306/metacoding?useSSL=false&serverTimezone=UTC&useLegacyDatetimeCode=false&allowPublicKeyRetrieval=true

# k8s/backend/backend-secret.yml
# SPRING_DATASOURCE_USERNAME: admin
# SPRING_DATASOURCE_PASSWORD: metacoding
```

---

## 5) 쿠버네티스 배포 (Ingress 없이 LoadBalancer만)

### (CLI) 네임스페이스 및 리소스 적용
```powershell
kubectl apply -f k8s/namespace.yml
# 네임스페이스 생성.

kubectl apply -f k8s/backend/
# 백엔드 리소스 적용.

kubectl apply -f k8s/frontend/
# 프론트 리소스 적용.

kubectl apply -f k8s/redis/
# Redis 리소스 적용.

kubectl apply -f k8s/frontend/frontend-service-loadbalancer.yml
# 프론트 서비스 타입을 LoadBalancer로 적용.
```

### (CLI) 상태 확인
```powershell
kubectl get pods -n metacoding -o wide
# 파드 상태 확인.

kubectl get svc -n metacoding
# LoadBalancer 외부 접속 주소 확인.
```

---

## 6) RDS 초기 데이터 생성 (user_tb)

### (CLI) mysql-client 파드 실행
```powershell
kubectl delete pod mysql-client -n metacoding --ignore-not-found
# 이전 mysql-client 파드 정리.

kubectl run mysql-client -n metacoding --image=mysql:8 --command -- sleep 3600
# DB 초기화를 위한 임시 파드 생성.

kubectl get pod mysql-client -n metacoding
# mysql-client 파드가 Running인지 확인.
```

### (CLI) 테이블/데이터 생성
```powershell
kubectl exec -n metacoding -i mysql-client -- mysql -h metacoding-db.cfcouus4ix90.ap-northeast-2.rds.amazonaws.com -u admin -pmetacoding -e "CREATE DATABASE IF NOT EXISTS metacoding; USE metacoding; CREATE TABLE IF NOT EXISTS user_tb (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50)); INSERT INTO user_tb (name) VALUES ('ssar'),('cos');"
# metacoding DB 및 user_tb 테이블 생성 + 샘플 데이터 삽입.

kubectl delete pod mysql-client -n metacoding --ignore-not-found
# 임시 파드 삭제.
```

---

## 7) 서비스 확인

### (CLI) API 확인
```powershell
$lb = (kubectl get svc -n metacoding frontend-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
Invoke-WebRequest -UseBasicParsing -Uri ("http://" + $lb + "/api/users")
# 백엔드 API 정상 응답 확인.
```

### (UI) 브라우저 확인
- `http://<LoadBalancer DNS>` 접속
- 방문 횟수 및 사용자 리스트 표시 확인

---

## 해결 방법 (트러블슈팅)

1) **UnknownHostException (RDS 엔드포인트)**
   - ConfigMap의 엔드포인트 오타 확인
   - RDS 상태가 `Available`인지 확인

2) **Table 'mysql.user_tb' doesn't exist**
   - DB가 `mysql`로 설정돼 있거나, 테이블이 없음
   - **DB명 `metacoding` 생성 + 테이블 생성** 후 ConfigMap을 `.../metacoding`으로 변경

3) **502 Bad Gateway**
   - 백엔드 컨테이너가 Git clone/Gradle build 중일 수 있음
   - 빌드 완료 후 재시도

4) **mysql-client Pending**
   - 노드에 `karpenter.sh/disrupted` taint가 있을 수 있음
   - 아래 명령으로 taint 제거 후 재시도
     ```powershell
     kubectl taint nodes <노드명> karpenter.sh/disrupted-
     ```

---

## 주의사항

- **Ingress는 사용하지 않음** (LoadBalancer만 사용)
- **RDS 보안 그룹 인바운드**에 EKS 노드/클러스터 SG 허용 필수
- **DB 초기화는 1회만 수행** (중복 INSERT 방지)
- 실습 비밀번호는 단순하게 설정했으므로 운영 환경에서는 강력한 비밀번호 사용

---

## 흐름도 (Mermaid)

```mermaid
flowchart LR
  A[Docker Build] --> B[ECR Push]
  B --> C[EKS Deploy]
  C --> D[LoadBalancer Access]
  C --> E[RDS MySQL]
  E --> F[Init DB (user_tb)]
  D --> G[Frontend UI]
  G --> H[/api/users]
  H --> E
```
