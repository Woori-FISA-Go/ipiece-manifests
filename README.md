# IPiece Kubernetes Manifests

**IPiece 프로젝트의 Kubernetes 인프라를 정의하는 GitOps 저장소입니다.**

- 모든 리소스는 **ArgoCD**를 통해 Kubernetes 클러스터(EKS)에 지속적으로 동기화됩니다.
- 애플리케이션별 Kubernetes 리소스는 **Kustomize**로 Dev / Prod 환경을 분리하여 관리합니다.

---

## 📁 Repository Structure

```bash
.
├── apps/
│   ├── backend/             # Spring Boot Backend (Prod/Dev 오버레이)
│   │   ├── base/            # 공통 Deployment / Service / Ingress
│   │   └── overlays/
│   │       ├── dev/
│   │       └── prod/
│   │
│   └── frontend/            # Next.js Frontend (Prod/Dev 오버레이)
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── prod/
│
├── system/
│   └── networking/          # 공용 ALB(Ingress) 설정 (ArgoCD, Monitoring)
│       ├── argocd-ingress.yaml
│       └── monitoring-ingress.yaml
│
└── argo-apps/               # ArgoCD Applications (App 정의)
    ├── ipiece-backend-prod.yaml
    ├── ipiece-frontend-prod.yaml
    ├── ipiece-monitoring.yaml
    └── ipiece-networking.yaml

```

- **apps/**
    - 비즈니스 애플리케이션 워크로드(frontend / backend)를 정의합니다.
    - `base` + `overlays/dev|prod` 구조로 환경별 차이를 최소 diff로 관리합니다.
- **system/**
    - 클러스터 공용 네트워킹 구성(내부용 ALB, ArgoCD, 모니터링 진입점 등)을 정의합니다.
- **argo-apps/**
    - ArgoCD에서 읽어가는 `Application` 리소스를 정의합니다.
    - 현재는 `backend-prod`, `frontend-prod`, `monitoring`, `networking`을 각각 개별 앱으로 관리합니다.

---

## ⚙️ GitOps Workflow

<img width="1154" height="500" alt="Image" src="https://github.com/user-attachments/assets/c00ec3b7-ab9e-4cd7-8360-06273235f686" />

1. **애플리케이션 저장소(Frontend/Backend)** 에서 코드 변경 → GitHub에 Push
2. GitHub Actions(CI)가 동작
    - Docker 이미지를 빌드
    - ECR에 Push
    - 이 저장소(`ipiece-manifests`)의 `kustomization.yaml` 이미지 태그를 업데이트
3. 변경된 Manifest가 `develop` 브랜치에 머지
4. ArgoCD가 `develop` 브랜치 변경을 감지
5. EKS 클러스터에 자동으로 Sync & 배포

> 인프라 변경은 항상 Git Commit → PR → Merge → ArgoCD Sync 순서로 진행되며,
> 
> 
> 수동 `kubectl apply`는 예외 상황에서만 사용합니다.
> 

---

## 🏗 apps/backend: Spring Boot Backend

### Base 리소스

경로: `apps/backend/base/`

- **Deployment**: `ipiece-server`
    - 이미지: `235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-server`
    - 컨테이너 포트: `8080`
    - 환경 변수:
        - `SPRING_PROFILES_ACTIVE=prod`
        - `SERVER_SERVLET_CONTEXT_PATH=/api`
    - 헬스 체크:
        - Liveness: `GET /api/healthz` (port 8080)
        - Readiness: `GET /api/healthz` (port 8080)
- **Service**: `ipiece-server`
    - Type: `ClusterIP`
    - Port: `80 → 8080`
- **Ingress**: `ipiece-backend-ingress`
    - Class: `alb` (AWS Load Balancer Controller)
    - Annotations:
        - `alb.ingress.kubernetes.io/scheme: internet-facing`
        - `alb.ingress.kubernetes.io/target-type: ip`
        - `alb.ingress.kubernetes.io/group.name: ipiece` (프론트와 같은 ALB 공유)
        - `alb.ingress.kubernetes.io/healthcheck-path: /api/healthz`
    - 라우팅:
        - `/api/` prefix → Service `ipiece-server:80`

> 결과적으로 외부 사용자는 https://<ALB>/api/** 로 백엔드 API를 호출합니다.
> 

### Overlays

### Dev

경로: `apps/backend/overlays/dev/`

- 이미지 태그:
    - `newTag: "3893089"` (예: 특정 커밋/빌드 번호)
- replicas:
    - `ipiece-server: 1`
- 환경:
    - Kustomize 패치로 `SPRING_PROFILES_ACTIVE=dev` 로 교체

### Prod

경로: `apps/backend/overlays/prod/`

- `resources: ../../base`
- replicas:
    - `ipiece-server: 2` (Prod 트래픽용)
- 이미지 태그:
    - `newTag: 21de53d` (배포에 사용된 Git SHA/빌드 태그)

---

## 🏗 apps/frontend: Next.js Frontend

### Base 리소스

경로: `apps/frontend/base/`

- **Deployment**: `ipiece-frontend`
    - 컨테이너 포트: `3000`
    - 이미지 플레이스홀더: `ipiece-frontend` (Kustomize로 ECR 경로로 치환)
    - `NODE_ENV=production`
- **Service**: `ipiece-frontend`
    - Type: `ClusterIP`
    - Port: `80 → 3000`
    - Annotation:
        - `service.kubernetes.io/topology-mode: "Auto"`
            
            → 같은 AZ의 파드를 우선 사용 (비용 절감 + 지연시간 감소)
            
- **Ingress**: `ipiece-frontend-ingress`
    - Class: `alb`
    - Annotations:
        - `alb.ingress.kubernetes.io/scheme: internet-facing`
        - `alb.ingress.kubernetes.io/target-type: ip`
        - `alb.ingress.kubernetes.io/healthcheck-path: /`
        - `alb.ingress.kubernetes.io/group.name: ipiece` (백엔드와 ALB 공유)
    - 라우팅:
        - `/` prefix → Service `ipiece-frontend:80`

> 동일한 ALB에서 /는 Frontend, /api/는 Backend를 처리하도록 구성되어 있습니다.
> 

### Overlays

### Dev

경로: `apps/frontend/overlays/dev/`

- 이미지:
    - `ipiece-frontend` → `…dkr.ecr…/ipiece-frontend:27a8d8f3...`
- replicas:
    - `ipiece-frontend: 1` (개발/테스트 환경)

### Prod

경로: `apps/frontend/overlays/prod/`

- 이미지:
    - `ipiece-frontend` → `…dkr.ecr…/ipiece-frontend:78970ab4...`
- replicas:
    - `ipiece-frontend: 2` (운영 트래픽 대응)

---

## 🌐 system/networking: 공용 네트워킹 구성

경로: `system/networking/`

### ArgoCD Ingress

- 파일: `argocd-ingress.yaml`
- 목적: 내부용 ALB를 통해 ArgoCD UI 접속
- 주요 설정:
    - `alb.ingress.kubernetes.io/scheme: internal`
    - `alb.ingress.kubernetes.io/group.name: common-alb`
    - Health check: `/argocd/healthz`
    - `/argocd` → Service `argocd-server:80`

### Monitoring Ingress

- 파일: `monitoring-ingress.yaml`
- 목적: 내부용 ALB를 통해 모니터링 엔트리 엔드포인트 제공
- 주요 라우팅:
    - `/api/v1/write` → Prometheus Remote Write
    - `/loki/api/v1/push` → Loki
    - `/grafana` → Grafana UI

> 이 Ingress들은 argo-apps/ipiece-networking.yaml 를 통해 ArgoCD에 의해 관리됩니다.
> 

---

## 🚀 ArgoCD Applications

경로: `argo-apps/`

현재는 App-of-Apps 대신, 각 역할별로 Application을 분리해 관리합니다.

### ipiece-backend-prod

- `path: apps/backend/overlays/prod`
- `targetRevision: develop`
- `destination.namespace: default`
- `syncPolicy.automated` 활성화
    - `prune: true`
    - `selfHeal: true`

### ipiece-frontend-prod

- `path: apps/frontend/overlays/prod`
- `namespace: ipiece-frontend-prod`
- Prod Frontend 전용 네임스페이스에 배포

### ipiece-monitoring

- `path: system/monitoring`
- Helm 차트 + `values.yaml` 조합으로 Prometheus/Loki/Grafana 스택 구성 (Planned/구축 중)

### ipiece-networking

- `path: system/networking`
- 공용 internal ALB, ArgoCD Ingress, Monitoring Ingress 등을 관리

---

## ⏪ Rollback Strategy

이 저장소는 GitOps 철학에 따라 **Git 기록을 기준으로 롤백**합니다.

1. 원하는 시점의 커밋을 기준으로 `git revert` 수행
2. `develop` 또는 `release/*` 브랜치에 Push
3. ArgoCD가 변경을 감지 → 해당 시점 Manifest로 자동 Sync

> 클러스터에서 수동으로 리소스를 수정하지 않고,
> 
> 
> **모든 변경을 Git 이력으로 추적**하는 것을 목표로 합니다.
> 

---

## 📌 Current Status

- ✅ **Frontend**
    - Dev / Prod Kustomize 오버레이 및 ArgoCD Prod Application 구성 완료
    - ECR 기반 이미지 태그로 운영 중
- ✅ **Backend**
    - Kustomize 기반 Dev / Prod 구조 및 Prod Application 구성 완료
    - 헬스체크(`/api/healthz`) 및 ALB 설정 정리 완료
- 🕓 **Monitoring**
    - Ingress 및 ArgoCD Application 정의 완료
    - Helm 기반 스택(system/monitoring)은 구축/튜닝 진행 중
