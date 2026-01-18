# 미니큐브 -> kind 변경 방법 및 비교

이 문서는 현재 프로젝트를 미니큐브에서 kind로 변경하는 방법, 주의 사항, 그리고 흐름 비교를 정리합니다.

## 변경 방법 (요약)

1. kind 클러스터 생성
   - 예: `kind create cluster --name metacoding`
2. 이미지 준비 방식 변경
   - 미니큐브: `minikube image build ...`
   - kind: 로컬 Docker에서 빌드 후 `kind load docker-image ...`
3. 네임스페이스/리소스 적용
   - 동일하게 `kubectl apply -f k8s/ --recursive`
4. 접속 방식 변경
   - 미니큐브: `minikube service frontend-service -n metacoding --url`
   - kind: `kubectl port-forward` 또는 Ingress/NodePort 설정

## 주의 사항

- 이미지 로딩 방식 차이
  - kind는 노드가 별도 컨테이너이므로 로컬 이미지를 자동으로 보지 않습니다.
  - 반드시 `kind load docker-image metacoding/backend:1` 처럼 노드에 로드해야 합니다.
- PV/HostPath 경로
  - 현재 PV는 `hostPath: /data/mysql`로 설정되어 있습니다.
  - kind 노드는 컨테이너이므로 실제 노드 파일시스템과 다릅니다.
  - 데이터 보존/공유가 필요하면 hostPath 대신 local-path/스토리지클래스 고려.
- Service 접근
  - kind는 `minikube service` 명령이 없습니다.
  - `kubectl port-forward service/frontend-service 8080:80 -n metacoding` 같은 방식 필요.
- NodePort 사용 시 포트 충돌 주의
  - 로컬 개발 환경에서 기존 포트와 충돌할 수 있습니다.
- 이미지 태그 고정
  - `:latest` 대신 버전 태그를 사용해 로딩 혼동을 줄이는 것이 안전합니다.

## 명령어 예시

```bash
# 1) kind 클러스터 생성
kind create cluster --name metacoding

# 2) 로컬 Docker에서 이미지 빌드
docker build -t metacoding/db:1 ./db
docker build -t metacoding/backend:1 ./backend
docker build -t metacoding/frontend:1 ./frontend
docker build -t metacoding/redis:1 ./redis

# 3) kind 노드로 이미지 로드
kind load docker-image metacoding/db:1 --name metacoding
kind load docker-image metacoding/backend:1 --name metacoding
kind load docker-image metacoding/frontend:1 --name metacoding
kind load docker-image metacoding/redis:1 --name metacoding

# 4) 리소스 적용
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/ --recursive

# 5) 접속 (포트포워딩 예시)
kubectl port-forward service/frontend-service 8080:80 -n metacoding
```

## 흐름 비교 (미니큐브 vs kind)

```mermaid

flowchart LR
  %% Minikube 흐름
  subgraph M[미니큐브 흐름]
    m1[1 minikube image build] --> m2[2 kubectl apply]
    m2 --> m3[3 서비스 생성/연결]
    m3 --> m4[4 minikube service로 접속]
  end

  %% Kind 흐름
  subgraph K[kind 흐름]
    k1[1 docker build] --> k2[2 kind load docker-image]
    k2 --> k3[3 kubectl apply]
    k3 --> k4[4 port-forward/Ingress로 접속]
  end

  %% 비교 메모
  m1 -.차이점.-> k1
  m4 -.차이점.-> k4
```

## 구성 리소스 관점 비교

```mermaid

flowchart TD
  subgraph Minikube[Minikube]
    mBuild[로컬 + 미니큐브 내부 이미지 빌드]
    mPV[PV hostPath: /data/mysql]
    mAccess[minikube service]
  end

  subgraph Kind[kind]
    kBuild[로컬 Docker 빌드 + kind load]
    kPV[hostPath는 노드 컨테이너 내부]
    kAccess[port-forward/Ingress/NodePort]
  end

  mBuild --> kBuild
  mPV --> kPV
  mAccess --> kAccess
```

## kind 아키텍처 (LR)

아래는 `minikube-architecture.md` 구성을 kind 기준으로 바꾼 LR 플로우차트입니다.

```mermaid

flowchart LR
  %% 로컬 이미지 빌드 + kind 로드
  subgraph Build[로컬 Docker 빌드 및 kind 로드]
    b_db[db 이미지\<br>metacoding/db:1]
    b_be[backend 이미지\<br>metacoding/backend:1]
    b_fe[frontend 이미지\<br>metacoding/frontend:1]
    b_redis[redis 이미지\<br>metacoding/redis:1]
    load[kind load docker-image]
    b_db --> load
    b_be --> load
    b_fe --> load
    b_redis --> load
  end

  %% 클러스터/네임스페이스
  subgraph Cluster[kind 클러스터]
    subgraph NS[네임스페이스: metacoding]
      subgraph FE[프론트엔드 구역]
        fe_svc[Service: frontend-service\<br>port 80]
        fe_pod[Pod: frontend\<br>replicas 1]
        fe_svc --> fe_pod
      end

      subgraph BE[백엔드 구역]
        be_svc[Service: backend-service\<br>port 8080]
        be_pod[Pod: backend\<br>replicas 2]
        be_cfg[ConfigMap: backend-configmap]
        be_sec[Secret: backend-secret]
        be_svc --> be_pod
        be_cfg --> be_pod
        be_sec --> be_pod
      end

      subgraph DB[DB 구역]
        db_svc[Service: db-service\<br>port 3306]
        db_pod[Pod: db\<br>replicas 1]
        db_sec[Secret: db-secret]
        pvc[PVC: db-pvc]
        pv[PV: db-pv\<br>/data/mysql]
        db_svc --> db_pod
        db_sec --> db_pod
        db_pod --> pvc --> pv
      end

      subgraph RD[Redis 구역]
        redis_svc[Service: redis-service\<br>port 6379]
        redis_pod[Pod: redis\<br>replicas 1]
        redis_svc --> redis_pod
      end
    end
  end

  %% 이미지 로딩 결과가 배포에 사용됨
  load --> fe_pod
  load --> be_pod
  load --> db_pod
  load --> redis_pod

  %% 접속 흐름 번호 표기
  user[사용자 브라우저]
  subgraph Access[kind 접속 방식]
    pf[port-forward]
    ing[Ingress]
    np[NodePort]
  end
  user -->|1 접속| pf
  user -->|1 접속| ing
  user -->|1 접속| np
  pf --> fe_svc
  ing --> fe_svc
  np --> fe_svc
  fe_pod -->|2 API 요청| be_svc
  be_pod -->|3 DB 조회| db_svc
  be_pod -->|4 캐시 사용| redis_svc
```

## PV 대안 흐름 (kind 기준)

kind에서 `hostPath` 대신 사용할 수 있는 대안을 간단히 비교합니다.

```mermaid

flowchart LR
  subgraph A[현재 구성]
    a1[PV hostPath: /data/mysql]
    a2[노드 컨테이너 내부 경로]
    a1 --> a2
  end

  subgraph B[대안 1: local-path]
    b1[StorageClass: local-path]
    b2[PVC 자동 바인딩]
    b3[노드 내부 경로 관리]
    b1 --> b2 --> b3
  end

  subgraph C[대안 2: hostPath + extraMounts]
    c1[kind 설정: extraMounts]
    c2[호스트 경로를 노드 컨테이너로 마운트]
    c3[PV hostPath를 마운트 경로로 지정]
    c1 --> c2 --> c3
  end

  a2 -.대안.-> b1
  a2 -.대안.-> c1
```
