# IPiece Kubernetes Manifests

**IPiece 프로젝트의 Kubernetes 인프라를 정의하는 GitOps 저장소입니다.**

모든 리소스는 ArgoCD를 통해 지속적으로 동기화되며, Kustomize로 Dev / Prod 환경을 관리합니다.

---

## 📁 Repository Structure

```
.
├── apps/
│   ├── frontend/       # Next.js FE (Active)
│   └── backend/        # Spring Boot BE (Scaffolded)
│
├── system/
│   └── monitoring/     # Prometheus / Loki 스택 (Planned)
│
└── argo-apps/          # ArgoCD App of Apps
    ├── ipiece-root-app.yaml
    ├── ipiece-frontend-dev.yaml
    └── ipiece-frontend-prod.yaml

```

- **apps/** — 비즈니스 애플리케이션
- **system/** — 클러스터 운영 구성 요소
- **argo-apps/** — ArgoCD에서 실제로 관리하는 앱 정의

---

## ⚙️ GitOps Workflow

<img width="1154" height="500" alt="Image" src="https://github.com/user-attachments/assets/c00ec3b7-ab9e-4cd7-8360-06273235f686" />

CI (GitHub Actions)

→ Docker Build & ECR Push

→ Manifest 이미지 태그 자동 업데이트

→ ArgoCD가 변경 감지

→ EKS 자동 배포

---

## 🏗 Kustomize Structure

### Base (공통)

- Deployment
- Service
- Ingress (AWS ALB)
- 이미지 alias: `ipiece-frontend` (Kustomize가 실제 ECR 주소로 치환)

### Overlays (환경별)

| 항목 | Dev | Prod |
| --- | --- | --- |
| 목적 | 테스트 | 실서비스 |
| 트리거 | develop push | release/** |
| replicas | 1 | 2 |
| 이미지 태그 | dev-latest | Git SHA |
| 네임스페이스 | ipiece-frontend-dev | ipiece-frontend-prod |

---

## 🚀 Deployment Strategy

### Development 배포

- FE 저장소 → **develop** 브랜치 Merge
- GitHub Actions 자동 실행
- `overlays/dev/` 이미지 태그 업데이트
- ArgoCD Sync → EKS 반영

### Production 배포

- FE 저장소 → **release/** 또는 main Push
- `overlays/prod/` 자동 업데이트
- ArgoCD가 운영 환경에 반영

---

## 🦑 ArgoCD: Root App Pattern

이 저장소는 **App of Apps(Root App)** 패턴을 사용합니다.

- ArgoCD에는 `ipiece-root-app.yaml` **딱 하나만** 등록
- 루트 앱이 `argo-apps/` 아래 YAML을 모두 읽어서
    
    → frontend-dev, frontend-prod 등 앱들이 자동 생성됨
    

**장점**

- 새 애플리케이션 추가 시 ArgoCD UI에서 아무것도 할 필요 없음
- YAML 추가 = 자동 관리

---

## 📊 Monitoring (Planned)

- **Prometheus**: EKS 내부 EBS를 사용 (Self-hosted, 비용절감)
- **Loki**: AWS S3 백엔드로 로그 저장
- **Grafana**: 온프레미스 + 클라우드 통합 대시보드

---

## ⏪ Rollback

ArgoCD UI에서 수동 롤백보다

**Git revert**가 확실하고 GitOps 철학에 맞음.

1. 원하는 커밋 `git revert`
2. develop 또는 release 브랜치로 Push
3. ArgoCD가 자동으로 이전 상태로 Sync

---

## 📌 Status

- ✔ FE: Active, Dev/Prod 파이프라인 완성
- ✔ Backend: Scaffold만 완료 (이미지 대기 중)
- 🕓 Monitoring: 설치 준비 완료
