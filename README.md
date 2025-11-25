# IPiece Kubernetes Manifests

IPiece 프로젝트의 **Kubernetes 인프라**를 정의하는 GitOps 저장소입니다.

- 모든 워크로드는 **Argo CD**를 통해 EKS 클러스터와 동기화됩니다.
- 애플리케이션별 Kubernetes 리소스는 **Kustomize**로 Dev / Prod 환경을 분리해 관리합니다.
- 백엔드는 GitHub Actions 기반 CI/CD 파이프라인으로 ECR 이미지 + manifests를 자동 갱신합니다.

---

## 📁 Repository Structure

```bash
├── ipiece-root-app.yaml        # Argo CD App-of-Apps 루트 Application
├── README.md
│
├── apps/                       # 실제 서비스 워크로드 (Backend/Frontend)
│   ├── backend/                # Spring Boot Backend (Kustomize)
│   │   ├── base/               # 공통 Deployment / Service / Ingress 정의
│   │   └── overlays/           # 환경별 설정 (dev / prod)
│   │
│   └── frontend/               # Next.js Frontend (Kustomize)
│       ├── base/               # 공통 Deployment / Service / Ingress 정의
│       └── overlays/           # 환경별 설정 (dev / prod)
│
├── system/                     # 클러스터 공용 인프라 리소스
│   ├── monitoring/             # Monitoring Helm Chart (Prometheus + Loki + Grafana + OTel)
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   └── networking/             # (옵션) ArgoCD / Monitoring용 ALB Ingress
│       ├── argocd-ingress.yaml
│       └── monitoring-ingress.yaml
│
└── argo-apps/                  # Argo CD Applications (App-of-Apps 패턴)
    ├── ipiece-backend-prod.yaml
    ├── ipiece-frontend-prod.yaml
    ├── ipiece-monitoring.yaml
    └── ipiece-networking.yaml
```

- **apps/**
    - 실제 서비스 트래픽을 받는 Product Backend / Frontend 워크로드 정의
    - `base` + `overlays/dev|prod` 구조로 공통 설정과 환경별 차이만 최소 diff로 유지
- **system/**
    - 모니터링 스택 및 (옵션) 공용 ALB Ingress 등 **클러스터 공용 인프라 리소스**
- **argo-apps/**
    - Argo CD에서 관리하는 `Application` 리소스 (App-of-Apps 패턴)

---

## 🔁 GitOps Workflow 개요

IPiece 인프라는 다음 흐름으로 관리됩니다.

1. 애플리케이션 코드 변경 (Backend / Frontend)
2. GitHub에 PR / Merge → `main` / `develop` / `release/*` 브랜치에 push
3. GitHub Actions(CI) 실행
    - Backend:
        - Gradle 빌드
        - Docker 이미지 빌드 & ECR push (`ipiece-server`)
        - `apps/backend/overlays/prod/kustomization.yaml` 의 이미지 태그 자동 업데이트
        - `ipiece-backend-env` Secret을 EKS에 재생성 (GitHub Secrets → K8s Secret 동기화)
4. `ipiece-manifests` 저장소의 `develop` 브랜치가 업데이트됨
5. Argo CD
    - `ipiece-root` Application이 `argo-apps` 폴더를 감시
    - 각 하위 Application (`ipiece-backend-prod`, `ipiece-frontend-prod`, `ipiece-monitoring`, `ipiece-networking`)을 통해
    - EKS 클러스터와 **자동 Sync (자동 배포)**

> 운영 중에는 kubectl apply 대신 Git 커밋 & Argo CD Sync를 기준으로 변경 내역을 관리합니다.
> 
> 
> Git 기록 = 인프라 변경 이력
> 

---

## 🧩 Backend (apps/backend)

### Base 리소스 (`apps/backend/base/`)

- **Deployment: `ipiece-server`**
    - 이미지:
        
        `235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-server:<tag>`
        
    - 컨테이너 포트: `8080`
    - 환경 변수:
        - `SPRING_PROFILES_ACTIVE=prod`
        - `SERVER_SERVLET_CONTEXT_PATH=/api`
        - DB/Redis/JWT/BESU 등 민감 값은
            
            `envFrom.secretRef.name = ipiece-backend-env`
            
    - 헬스체크:
        - Liveness: `GET /api/healthz`
        - Readiness: `GET /api/healthz`
- **Service: `ipiece-server`**
    - Type: `ClusterIP`
    - Port: `80 → 8080`
- **Ingress: `ipiece-backend-ingress`**
    - IngressClass: `alb` (AWS Load Balancer Controller)
    - Annotations:
        - `alb.ingress.kubernetes.io/scheme: internet-facing`
        - `alb.ingress.kubernetes.io/target-type: ip`
        - `alb.ingress.kubernetes.io/group.name: ipiece`
        - `alb.ingress.kubernetes.io/healthcheck-path: /api/healthz`
    - 라우팅:
        - `/api/` → Service `ipiece-server:80`

➡️ 결과: **외부에서** `https://<ALB>/api/**` 로 Backend API에 접근.

### Prod 오버레이 (`apps/backend/overlays/prod/`)

- `resources: ../../base`
- 이미지 태그:
    - GitHub Actions가 `kustomize edit set image`로
        
        `newTag: <최근 SHA>` 자동 갱신
        
- Replica 수:
    - 기본 2개 (고가용성)

Dev 오버레이는 동일한 구조로, 태그/replica/환경설정만 다르게 가져갈 수 있습니다.

---

## 🎨 Frontend (apps/frontend)

### Base 리소스 (`apps/frontend/base/`)

- **Deployment: `ipiece-frontend`**
    - 컨테이너 포트: `3000`
    - 이미지: `ipiece-frontend` (오버레이에서 ECR 경로로 치환)
    - 환경 변수:
        - `NODE_ENV=production`
- **Service: `ipiece-frontend`**
    - Type: `ClusterIP`
    - Port: `80 → 3000`
    - Annotation:
        - `service.kubernetes.io/topology-mode: "Auto"`
            
            → 같은 AZ 내 파드로 우선 라우팅 (비용/레이턴시 이점)
            
- **Ingress: `ipiece-frontend-ingress`**
    - IngressClass: `alb`
    - Annotations:
        - `alb.ingress.kubernetes.io/scheme: internet-facing`
        - `alb.ingress.kubernetes.io/target-type: ip`
        - `alb.ingress.kubernetes.io/healthcheck-path: /`
        - `alb.ingress.kubernetes.io/group.name: ipiece`
    - 라우팅:
        - `/` → Service `ipiece-frontend:80`

➡️ 결과:

- **외부에서** `https://<ALB>/` 로 Frontend 접속
- `/api/**` 트래픽은 같은 ALB 내 Backend로 라우팅 (그룹 이름 `ipiece` 공유)

### Prod 오버레이 (`apps/frontend/overlays/prod/`)

- 이미지:
    - `ipiece-frontend` →
        
        `235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-frontend:<tag>`
        
- Replica 수: 2개 (Prod 기본 값)

---

## 📊 Monitoring Stack (system/monitoring)

`system/monitoring`은 **Helm Chart**로 구성된 통합 모니터링 스택입니다.

- `Chart.yaml`
    - `kube-prometheus-stack`
    - `loki-stack`
    - `opentelemetry-collector` (alias: `otel`)
- `values.yaml`
    - **Prometheus**
        - Remote write receiver 활성화
        - EBS (gp3) 기반 PVC 사용 (`50Gi`)
    - **Grafana**
        - admin 비밀번호: `"admin"` (실운영 시 변경 권장)
        - PVC (gp3, `10Gi`)
        - 추가 데이터소스: Loki (`http://ipiece-loki:3100/`)
        - `/grafana` 서브패스로 서비스될 수 있도록 env 설정
    - **Loki**
        - fullnameOverride: `ipiece-loki`
        - S3 `ipiece-loki-store`를 backend로 사용
        - `loki-s3-secret`에서 AWS 자격 증명 주입
    - **OTel Collector (`otel`)**
        - DaemonSet 모드
        - 노드 메트릭, kubeletstats, 파일 로그 수집
        - Metrics → Prometheus(remote write)
        - Logs → Loki(OTLP HTTP)

⚠️ Loki S3 연동에는 `monitoring` 네임스페이스에

`loki-s3-secret`(AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY)이 **필수**입니다.

---

## 🌐 Networking (system/networking)

현재 `system/networking`에는 다음 Ingress가 정의되어 있습니다.

- `argocd-ingress.yaml`
    - Argo CD UI를 internal ALB로 노출하는 Ingress
- `monitoring-ingress.yaml`
    - Grafana 및 Loki/Prometheus 엔드포인트를 internal ALB로 노출하는 Ingress

> 📌 현재 운영 전략
> 
> 
> Argo CD / Grafana는 ALB로 노출하지 않고,
> 
> **kubectl port-forward (터널링)** 으로 `localhost`에서 접속하는 방식을 기본으로 사용합니다.
> 
> ALB로 노출할 필요가 생기면:
> 
> - `ipiece-networking` Application을 Sync시키고
> - `system/networking`의 Ingress들을 활성화해서 사용

---

## 🚀 Argo CD Applications (App-of-Apps)

`argo-apps/` 아래의 Application들이 `ipiece-root`에 의해 한 번에 관리됩니다.

- **`ipiece-root-app.yaml`**
    - Argo CD에 최초로 적용하는 **루트 Application**
    - `path: argo-apps` 를 바라보고, 나머지 App들을 생성
- **`ipiece-backend-prod.yaml`**
    - `path: apps/backend/overlays/prod`
    - `namespace: default`
    - Prod Backend 배포 & 자동 Sync
- **`ipiece-frontend-prod.yaml`**
    - `path: apps/frontend/overlays/prod`
    - `namespace: ipiece-frontend-prod` (별도 네임스페이스)
- **`ipiece-monitoring.yaml`**
    - `path: system/monitoring`
    - Helm Chart + `values.yaml`로 모니터링 스택 배포
    - `namespace: monitoring`
- **`ipiece-networking.yaml`**
    - `path: system/networking`
    - internal ALB용 Ingress 리소스 배포 (ArgoCD / Monitoring용)
    - 현재는 **포트포워딩으로 접근하는 전략**을 기본으로 하기 때문에
        
        필요할 때만 사용 가능
        

---

## 🔐 접근 방법 (포트포워딩)

Argo CD / Grafana는 ALB 대신 **kubectl port-forward**로 접근합니다.

### Argo CD UI

kubectl port-forward svc/argocd-server -n argocd 8080:80

브라우저:

http://localhost:8080

### Grafana UI

kubectl port-forward svc/ipiece-monitoring-grafana -n monitoring 3000:80

브라우저:

http://localhost:3000/grafana

---

## 📌 Secrets 정리

- `default` 네임스페이스
    - `ipiece-backend-env`
        - **GitHub Actions에서 자동 생성/업데이트**
        - GitHub Secrets (`DB_HOST`, `JWT_SECRET`, `BESU_RPC_URL`, `KRWT_CONTRACT_ADDRESS`, `SOLAPI_*` 등)이 소스
    - `ecr-secret`
        - ECR 도커 레지스트리 로그인 정보
        - 새 클러스터에서는 아래 명령으로 재생성:
            
            aws ecr get-login-password --region ap-northeast-2
            
            | kubectl create secret docker-registry ecr-secret
            
            --docker-server=235625001959.dkr.ecr.ap-northeast-2.amazonaws.com
            
            --docker-username=AWS
            
            --docker-password-stdin
            
            -n default
            
- `monitoring` 네임스페이스
    - `loki-s3-secret`
        - Loki가 S3(`ipiece-loki-store`)에 로그를 쓰기 위한 AWS 자격증명
        - 새 클러스터를 만들면 **다시 생성 필요**

---

## ✅ Rollback Strategy

GitOps 구조이기 때문에, 에러가 날 경우:

1. 문제가 된 시점 이전의 Git Commit을 기준으로 `git revert`
2. `develop` 브랜치에 push
3. Argo CD가 변경 내역을 감지 → EKS에 자동 반영

> 가능하면 클러스터 안에서 직접 리소스를 수정하지 않고,
> 
> 
> **항상 Git을 통해서만 변경**하는 것을 목표로 합니다.
> 

---

# EKS 재생성 체크리스트

(나중에 `docs/eks-recreate.md` 같은 파일로 분리해도 됨)

## 1. EKS 클러스터 기본 정보

- Region: `ap-northeast-2`
- Cluster Name: `ipiece-cluster1`
    - GitHub Actions에서 사용 중:
        
        aws eks update-kubeconfig --region ap-northeast-2 --name ipiece-cluster1
        
    - 클러스터 이름을 바꾸면 위 이름도 같이 수정해야 함.
- NodeGroup:
    - 최소 2노드 (Prometheus + Loki + 앱까지 고려)
    - t3.medium ~ t3.large 수준 (상황에 따라 조정)

---

## 2. EKS 애드온 / 컨트롤러 설치

1. kubectl 컨텍스트 설정
    
    aws eks update-kubeconfig --region ap-northeast-2 --name ipiece-cluster1
    
    kubectl get nodes
    
2. IAM OIDC Provider 연결
    - aws-load-balancer-controller, EBS CSI 등용 (공식 문서대로)
3. AWS Load Balancer Controller 설치
    - Helm 또는 공식 매뉴얼대로 설치
    - IngressClass `alb`를 사용하므로 **필수**
4. EBS CSI Driver 설치
    - EKS Add-on 또는 Helm
    - Prometheus / Grafana PVC가 `gp3` StorageClass를 사용하므로 필요

---

## 3. 네임스페이스 및 기본 Secret

1. 네임스페이스 생성(필요 시):
    
    kubectl create namespace argocd
    
    kubectl create namespace monitoring
    
    kubectl create namespace ipiece-frontend-prod
    
2. ECR Secret 생성 (`default` 네임스페이스):
    
    aws ecr get-login-password --region ap-northeast-2
    
    | kubectl create secret docker-registry ecr-secret
    
    --docker-server=235625001959.dkr.ecr.ap-northeast-2.amazonaws.com
    
    --docker-username=AWS
    
    --docker-password-stdin
    
    -n default
    
3. Loki용 S3 Secret (`monitoring` 네임스페이스):
    
    kubectl create secret generic loki-s3-secret -n monitoring
    
    --from-literal=S3_AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
    
    --from-literal=S3_AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
    
    > 위 키 값은 IAM User에서 발급받은 것을 따로 안전하게 보관해두었다가 사용.
    > 

---

## 4. Argo CD 설치 및 ipiece-root 적용

1. Argo CD 설치
    - `argocd` 네임스페이스에 Helm 또는 공식 manifest로 설치
2. 루트 Application 적용
    
    (로컬에서, `ipiece-manifests` 루트에서)
    
    kubectl apply -f ipiece-root-app.yaml -n argocd
    
3. Argo CD UI 접속 (포트포워딩):
    
    kubectl port-forward svc/argocd-server -n argocd 8080:80
    
    브라우저: http://localhost:8080
    
4. Argo CD에서:
    - `ipiece-root` Application 생성 확인
    - `ipiece-backend-prod`, `ipiece-frontend-prod`, `ipiece-monitoring` 순서대로 Sync
    
    > ipiece-networking은 ArgoCD/Grafana를 ALB로 노출하고 싶을 때만 Sync.
    > 

---

## 5. GitHub Actions / Secrets 점검

1. Backend 레포 GitHub Secrets:
    - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
    - `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`
    - `REDIS_HOST`, `REDIS_PORT`
    - `JWT_SECRET`
    - `SOLAPI_API_KEY`, `SOLAPI_API_SECRET`, `SOLAPI_SENDER`
    - `ADMIN_PRIVATE_KEY`
    - `BESU_RPC_URL`
    - `KRWT_CONTRACT_ADDRESS`
    - `MANIFESTS_REPO_TOKEN` (ipiece-manifests repo push 권한)
2. GitHub Actions 워크플로우가 정상적으로:
    - ECR에 이미지 push
    - `apps/backend/overlays/prod/kustomization.yaml` 이미지 태그 갱신
    - `default` 네임스페이스에 `ipiece-backend-env` Secret 생성하는지 확인

---

## 6. 모니터링 / 접근 확인

1. 모든 Pod 상태 확인:
    
    kubectl get pods -A
    
2. Grafana 포트포워딩:
    
    kubectl port-forward svc/ipiece-monitoring-grafana -n monitoring 3000:80
    
    브라우저: http://localhost:3000/grafana
    
    - ID: `admin`
    - PW: `admin` (values.yaml 기본값, 운영 시 변경 권장)
3. Frontend / Backend ALB 확인:
    
    kubectl get ingress -A
    
    - `ipiece-frontend-ingress`, `ipiece-backend-ingress`에 할당된 ALB 도메인 확인
