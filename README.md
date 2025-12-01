# ☁️ IPiece Infrastructure Repository
> **Hybrid Cloud Observability & GitOps on AWS EKS**

## 1. 📖 프로젝트 개요
 **IPiece 서비스의 인프라 구성(YAML/Helm/Kustomize)을 GitOps 방식으로 관리**하기 위한 저장소입니다.

**EKS 위에 올라가는 모든 Kubernetes 리소스**

- Frontend / Backend Deployment, Service, Ingress
- 모니터링 스택 (Prometheus, Loki, Grafana, OTel Collector, metrics-server)
- AWS Load Balancer Controller
- ArgoCD Application(App-of-Apps) 구성
- **하이브리드 모니터링 구조**
    - 온프레미스(ESXi/vSphere) 환경에서 수집한 로그·메트릭을
    - VPN을 통해 EKS 모니터링 스택으로 전송하는 파이프라인

### ✅ 관리 대상 및 범위
| 구분 | 상세 내용 |
|:---:|:---|
| **Workloads** | Frontend/Backend Deployment, Service, Ingress |
| **Observability** | Prometheus, Loki, Grafana, OpenTelemetry Collector, Metrics-server |
| **Network** | AWS Load Balancer Controller, ALB(Public), NLB(Internal/OTLP) |
| **GitOps** | ArgoCD (App-of-Apps 패턴), ArgoCD Notifications + Slack 알림 |
| **Hybrid Mon** | 온프레미스 서버 → VPN → EKS OTel Collector 연결 파이프라인 |

> 💡 **Core Concept:** 이 레포지토리 하나만 있으면 IPiece의 **"배포 구조 + 관측 구조"** 전체를 완벽하게 재현할 수 있습니다.

### 📂 디렉토리 구조
```plaintext
ipiece-manifests/
├─ .deploy-trigger                  # CI 파이프라인이 업데이트하는 트리거 파일 (GitOps Trigger)
├─ ipiece-root-app.yaml             # ArgoCD App of Apps 엔트리 포인트
├─ otel-collector.example.yaml      # 온프레미스용 OTel 설정 예시
├─ argo-apps/                       # ArgoCD Application 정의 (Child Apps)
│  ├─ ipiece-frontend-prod.yaml
│  ├─ ipiece-backend-prod.yaml
│  ├─ ipiece-monitoring.yaml
│  └─ ...
├─ apps/                            # Business Workloads (Kustomize)
│  ├─ backend/ (base + overlays/prod)
│  └─ frontend/ (base + overlays/prod)
└─ system/                          # Platform / Infra Components
   ├─ monitoring/ (Helm Umbrella Chart)
   ├─ aws-load-balancer-controller/
   └─ argocd-notifications/
```
### 1.2 GitOps 구조 (ArgoCD App-of-Apps)
<img width="1920" height="1080" alt="Image" src="https://github.com/user-attachments/assets/48fa7976-1946-4c77-b9bd-94f482e7616d" />

- **`ipiece-root-app.yaml`**
    - ArgoCD **App-of-Apps 엔트리 포인트**입니다.
    - `argo-apps/` 디렉토리를 하나의 루트 애플리케이션(`ipiece-root`)으로 등록하여,
        
        하위 Application들을 **자식(App of Apps Children)** 으로 관리합니다.
        
    - Root App이 Healthy 상태면, 구조적으로 필요한 Child App 정의는 모두 Git에 존재한다는 것을 의미합니다.
- **`argo-apps/`**
    - 개별 ArgoCD Application 정의 모음입니다.
    - 예시:
        - `ipiece-frontend-prod.yaml` → `apps/frontend/overlays/prod`를 동기화
        - `ipiece-backend-prod.yaml` → `apps/backend/overlays/prod`를 동기화
        - `ipiece-monitoring.yaml` → `system/monitoring/` (관측 스택)
        - `ipiece-aws-lb-controller.yaml` → `system/aws-load-balancer-controller/`
        - `ipiece-argocd-notifications.yaml` → `system/argocd-notifications/`
    - 스크린샷의 **각 카드 하나가 이 디렉토리의 YAML 하나**에 대응합니다.
- **`apps/` (Business Workloads)**
    - 실제 비즈니스 애플리케이션(Frontend/Backend)의 Kubernetes 리소스를 정의합니다.
    - **`base`**
        - Deployment / Service / Ingress 정의의 공통 부분을 모아둔 레이어입니다.
        - 로컬 테스트, 스테이지 환경 등 다른 overlay에서 재사용 가능합니다.
    - **`overlays/prod`**
        - ECR 이미지 태그, replica 수, 리소스 요청/제한, IRSA Role, 도메인/Path 등
            
            **운영(prod) 환경에 특화된 설정**만 오버레이합니다.
            
    - ArgoCD는 항상 `overlays/prod`를 대상으로 동기화하여,
        
        Git에 정의된 prod 스펙이 그대로 클러스터 상태에 반영되도록 합니다.
        
- **`system/` (Platform/Infra Components)**
    - 어플리케이션과 별개로 동작하는 **플랫폼 레이어**를 정의합니다.
    - 예: 모니터링 스택, LB 컨트롤러, ArgoCD Notifications 등
    - 이들 역시 각각 별도의 ArgoCD Application으로 관리하여,
        
        **플랫폼 변경(예: 모니터링 버전 업그레이드)** 도 GitOps 방식으로 수행할 수 있습니다.

---

## 2. 🏗️ 전체 아키텍처 (Architecture Overview)

![Architecture Diagram](https://github.com/user-attachments/assets/9d13dd3e-dbfe-4bb7-85d8-289512290409)

### 2.1 트래픽 제어 (Traffic Control)
* **Public Domain:** `ipiece.store`
* **Request Flow:** `Route53` → `ACM(SSL)` → `ALB(Internet-facing)` → `Ingress` → `Service` → `Pod`

**Path 기반 라우팅 전략 (Single ALB)**
비용 절감 및 관리 단순화를 위해 **단일 ALB**에 Ingress Group을 적용했습니다.

* `/` 👉 **Frontend** (Next.js UI)
* `/api` 👉 **Backend** (Spring Boot API)
* `/grafana/` 👉 **Monitoring** (Grafana Dashboard)

### 2.2 워크로드 격리 (Namespace Bulkhead)
보안과 운영 안정성을 위해 네임스페이스 단위로 리소스를 철저히 격리했습니다.

| 구분 | Frontend (UI) | Backend (API) |
|:---:|---|---|
| **Tech Stack** | Next.js (Node.js) | Spring Boot (Java) |
| **Namespace** | `ipiece-frontend-prod` | `default` |
| **Ingress Path** | `/` | `/api`, `/api/actuator` |
| **Internal Port** | `80` → `3000` | `80` → `8080` |
| **Security (IRSA)** | - | `ipiece-backend-s3-role` (S3 접근) |
| **Health Check** | `/` | `/api/healthz` |

* **운영 네임스페이스**
    * `monitoring`: 관측 스택 (Prometheus, Loki, Grafana, OTel)
    * `argocd`: CD 엔진 및 알림 시스템
    * `kube-system`: AWS Load Balancer Controller 등

### 2.3 하이브리드 관측성 (Hybrid Observability)
* **☁️ 클라우드 (EKS)**
    * **Prometheus:** 메트릭 수집 (EBS 50Gi, 15일 보존)
    * **Loki:** 로그 수집 (S3 `ipiece-loki-store`, 30일 보존)
    * **OTel Collector:** DaemonSet으로 노드/파드 데이터 수집 및 허브 역할
* **🏢 온프레미스 (On-Prem)**
    * **구성:** 각 서버(Vault, Validator 등)에 OTel Collector 컨테이너 배포
    * **전송:** VPN을 통해 EKS Internal NLB (Port 4317/4318)로 데이터 전송
* **📊 통합 뷰 (Single Pane of Glass)**
    * 위치(온프레/클라우드)와 관계없이 **Grafana 한 곳에서 전체 시스템을 관제**

---

## 3. 🛠️ 핵심 컴포넌트 상세 설정
![Component Details](https://github.com/user-attachments/assets/dcc95524-8159-4f4e-9c0c-55591cbc8981)

### 3.1 애플리케이션 (App Workloads)
* **Backend (Spring Boot):**
    * `ClusterIP` 서비스 구성.
    * `ipiece-backend-s3-role` IRSA를 통해 Access Key 없이 S3 접근.
    * OTel Java Agent를 주입하여 트레이스 데이터 자동 수집.
* **Frontend (Next.js):**
    * 사용자는 ALB(`https://ipiece.store/`)로 접속.
    * Frontend 서버 내에서 `/api` 호출 시 백엔드로 라우팅.

### 3.2 모니터링 스택 (Observability)
| 컴포넌트 | 저장소 | 보존 기간 | 역할 |
|---|---|---|---|
| **Prometheus** | EBS (gp2) | 15일 | 수치 데이터(Metrics) 수집 |
| **Loki** | S3 (Bucket) | 30일 | 텍스트 로그(Logs) 수집 및 검색 |
| **Grafana** | - | - | 데이터 시각화 (`/grafana/` 서브패스) |

### 3.3 OpenTelemetry & Network
* **EKS OTel (DaemonSet):**
    * `hostmetrics`, `kubeletstats`, `filelog` 리시버 활성화.
    * `prometheusremotewrite` exporter로 Prometheus 연동.
* **AWS Load Balancer Controller:**
    * **ALB:** 인터넷 트래픽 처리 (`group.name: ipiece`로 통합)
    * **NLB:** 온프레미스 OTLP 데이터 수신용 Internal LB (Port 4317/4318)

---
## 4. 🧩 아키텍처 상세 및 설계 선택 이유

### 4.1 설계 목표 및 기준

IPiece 인프라를 설계할 때 가장 먼저 잡은 목표는 다음과 같습니다.

1. **재현 가능성 (Reproducibility)**
    - 레포지토리 하나로 **동일한 인프라 상태를 반복해서 재구성**할 수 있어야 합니다.
    - 수동 콘솔 작업 대신, Git에 기록된 YAML/Helm/Kustomize가 곧 “설계서이자 실행 스크립트”가 되도록 구성했습니다.
2. **관측 가능성 (Observability First)**
    - 장애가 났을 때, “왜 그런지”를 추적할 수 있는 수준의 **로그·메트릭·트레이스**가 탑재되어야 합니다.
    - 온프레/클라우드 어디서 문제가 나도 **Grafana 한 곳에서 전체 상태를 볼 수 있는 구조**를 목표로 했습니다.        
3. **하이브리드 구조**
    - 온프레와 클라우드가 동시에 존재하는 환경을 전제로 하고,
        
        “처음부터 하이브리드 통합”을 고려해 설계했습니다.
        
    - 이후 온프레 자산이 늘어나도 **관측 파이프라인은 그대로 재사용**할 수 있도록 구성했습니다.

---

### 4.2 네트워크: 단일 ALB + Ingress 그룹 전략

- **결정**
    - ALB 여러 개 대신, **ALB 1개 + 다수 Ingress (Frontend/Backend/Grafana)** 구조를 채택했습니다.
- **이유**
    - **비용 절감**
        - ALB 개수 및 LCU 기반 과금을 최소화합니다.
    - **관리 효율**
        - ACM 인증서, WAF, 보안 그룹, Route53 레코드를 **단일 엔드포인트**에서 관리합니다.
    - **고가용성**
        - AWS 관리형 ALB의 **Multi-AZ 특성**을 그대로 활용하여 가용성을 확보합니다.            

---

### 4.3 관측성: 하이브리드 통합 & 스토리지 분리

- **결정**
    - 온프레 데이터는 **VPN + Internal NLB(OTLP)** 경로로만 수집합니다.
    - 저장소는 메트릭/로그를 **Prometheus(EBS) / Loki(S3)** 로 분리했습니다.
- **이유**
    - **Single Pane of Glass**
        - 온프레/클라우드 구분 없이 **Grafana 한 곳에서** 전체 시스템을 관제할 수 있습니다.
    - **내구성 & 복원력**
        - EKS 클러스터에 장애가 발생하거나, 재생성이 필요하더라도
            S3에 저장된 로그는 그대로 유지되어 **사후 분석이 가능**합니다.
            
    - **보안**
        - OTLP용 NLB는 `internal` 스킴으로 생성하여 **VPC 내부에서만 접근 가능**하도록 설계했습니다.
        - 외부 공개 엔드포인트(ALB)와 관측 엔드포인트(NLB)를 **네트워크 레벨에서 명확히 분리**했습니다.

---

### 4.4 운영: 네임스페이스 격리

- **결정**
    - 앱, 모니터링, GitOps 도구를 서로 다른 네임스페이스로 분리했습니다.
        - 예: `ipiece-frontend-prod` / `default` / `monitoring` / `argocd` 등
- **이유**
    - **격리 (Bulkhead Pattern)**
        - 한 영역(Pod 폭주, 메모리 누수, 모니터링 장애 등)의 문제가
            다른 영역으로 전파되지 않도록 네임스페이스를 구분했습니다.           
    - **역할 기반 권한 설계 용이**
        - 팀/역할 단위로 RBAC를 설계하기 용이합니다.
        - 예: 프론트/백엔드 개발자, 인프라 담당자의 권한을 네임스페이스 기준으로 명확히 분리 가능.
    - **운영 가독성**
        - `kubectl get pods -n <ns>` 만으로도
            “어떤 리소스가 어떤 역할을 하는지” 직관적으로 파악할 수 있습니다.


## 5. 🚀 CI/CD 및 운영 가이드
![CI/CD Flow](https://github.com/user-attachments/assets/c822343e-7dbd-4907-b860-047bebdccbe9)

### 5.1 배포 파이프라인 (Workflow)
1.  **Code Push:** 개발자가 Feature/Main 브랜치에 코드 푸시
2.  **CI Build:** GitHub Actions에서 빌드 & Docker 이미지 생성 → ECR Push
3.  **Git Update:** CI가 `ipiece-manifests` 레포의 `.deploy-trigger` 또는 이미지 태그 업데이트 (Commit)
4.  **ArgoCD Sync:** ArgoCD가 변경 사항 감지 후 EKS 클러스터에 롤링 업데이트 수행
5.  **Notification:** Slack으로 배포 성공/실패 알림 전송

> **📌 이미지 태그 정책:** `<commit SHA>-<run_id>-<attempt>`  
> (예: `3be2f6c...-19808065788-1`) 형태로 유일성을 보장하여 추적 가능하게 관리합니다.

---

## 6. 📝 초기 셋업 (Setup Guide)

### 6.1 사전 준비 사항 (Prerequisites)
1.  **EKS Cluster:** `ipiece-cluster1` 및 OIDC Provider 설정 완료.
2.  **IAM Role (IRSA):**
    * `aws-load-balancer-controller`
    * `ipiece-backend-s3-role`
    * `ipiece-loki-s3-role`
3.  **S3 Bucket:** Loki 저장용 `ipiece-loki-store`.
4.  **Network:** Route53 Hosted Zone & ACM 인증서(`ap-northeast-2`).

### 6.2 배포 순서
1.  **ArgoCD 설치:** `argocd` 네임스페이스 생성 및 설치.
2.  **Root App 적용:** `kubectl apply -f ipiece-root-app.yaml`
3.  **플랫폼 배포 확인:** `system/` 하위의 LB Controller, Monitoring 스택 자동 배포.
4.  **앱 배포 확인:** `apps/` 하위의 Frontend/Backend 배포 및 Ingress/ALB 생성 확인.
5.  **검증:**
    * `https://ipiece.store` 접속 확인.
    * Grafana 대시보드 데이터 수신 확인.
    * 온프레미스 OTel Collector 연결 확인.

---

## 7. 💰 비용 최적화 및 향후 계획

IPiece 인프라는 **“운영 가능한 안정성”과 “프로젝트 수준의 비용”** 사이에서 균형을 맞추도록 설계되었습니다.  
이 섹션에서는 현재 아키텍처에서의 주요 비용 포인트와, 이를 최적화하기 위한 전략 및 향후 개선 계획을 정리합니다.

---

### 7.1 현재 비용 구조 개요

현재 인프라의 비용은 크게 다음 네 영역에서 발생합니다.

1. **로드 밸런서 비용**
   - ALB (Application Load Balancer)
   - NLB (Network Load Balancer, OTLP 전용)

2. **컴퓨팅 비용**
   - EKS 클러스터 요금
   - 워커 노드(EC2) 비용

3. **스토리지 비용**
   - EBS (Prometheus, Grafana 등)
   - S3 (Loki 로그 저장소)

4. **네트워크/전송 비용**
   - 온프레미스 ↔ AWS 간 VPN 트래픽
   - AWS 내부 트래픽 (VPC 내 통신, S3 업로드 등)

이 중, IPiece에서는 **로드 밸런서 통합, 스토리지 보존 기간 조정, 온프레 로그 전송량 관리**를 중심으로 비용 최적화를 수행하고 있습니다.

---

### 7.2 비용 최적화 포인트

#### 7.2.1 ALB 통합 전략

- **전략**
  - **Frontend / Backend / Grafana** 를 각각 별도의 ALB로 두지 않고,
    **단일 ALB + Ingress Group (`group.name: ipiece`)** 구조로 통합.
  - Path 기반 라우팅으로 역할 분리:
    - `/` → Frontend
    - `/api` → Backend
    - `/grafana/` → Grafana

- **효과**
  - **ALB 개수 감소** → 고정 비용 및 LCU(Load Balancer Capacity Unit) 비용 절감
  - 인증서(ACM), 보안 그룹, Route53 레코드 관리 대상이 1개로 단순화
  - GitOps 관점에서도 Ingress 리소스를 서비스별로 나누되,
    실제 AWS 리소스는 하나로 묶여 관리 부담 감소

---

#### 7.2.2 Internal NLB로 온프레미스 트래픽 분리

- **전략**
  - 온프레미스 → EKS OTLP 트래픽은
    퍼블릭 엔드포인트를 사용하지 않고,
    **VPC 내부 전용 Internal NLB (ipiece-monitoring-otel)** 를 통해 수신.
  - 해당 NLB는 다음과 같은 포트만 개방:
    - `4317` (OTLP gRPC)
    - `4318` (OTLP HTTP)

- **효과**
  - 사용자 트래픽(ALB)와 관측용 트래픽(NLB)의 **용도 분리**
  - 외부 인터넷을 통하지 않는 내부 통신으로 **보안 강화**
  - 필요 최소 NLB 1개만 사용하여 **로드 밸런서 비용 제한**

---

#### 7.2.3 메트릭 스토리지 정책 (Prometheus + EBS)

- **구성**
  - Prometheus:
    - EBS 50Gi (gp2)
    - 메트릭 보존 기간: **15일**
    - `enableRemoteWriteReceiver: true` 로 OTel remote write 수신

- **비용 관점**
  - 메트릭은 **주기적 수집 + 장기 보관 시 비용 증가**가 빠른 영역.
  - 15일이라는 기간은,
    - 최근 장애/성능 문제 분석에는 충분
    - 디스크 사용량과 I/O 비용이 과도하게 늘어나지 않는 타협점

- **장점**
  - Prometheus 디스크 용량 및 성능 부담을 제한하면서도,
    운영 분석에 필요한 데이터는 유지
  - 향후 Thanos/Mimir 등을 통한 장기 보관 아키텍처로 확장 가능한 구조 유지

---

#### 7.2.4 로그 스토리지 정책 (Loki + S3 + Lifecycle)

- **구성**
  - Loki:
    - `boltdb-shipper + S3` 구조
    - 전용 버킷: `ipiece-loki-store`
    - 기본 보존 기간: **30일**
  - IRSA:
    - `ipiece-loki-s3-role` 을 통해 버킷 접근을 최소 권한으로 제한

- **비용 관점**
  - 로그는 메트릭보다 **데이터 양이 훨씬 많고**, S3 저장/요청 비용에 직접 영향.
  - 기본 30일 보존으로
    - 장애 전후 맥락을 텍스트 로그로 분석할 수 있는 최소 기간 확보
    - 그 이상은 **S3 Lifecycle 정책**으로 Glacier/삭제 처리 예정

- **Lifecycle 정책 방향**
  - 30일 이후:
    - 중요하지 않은 로그 → 자동 삭제
    - 필요하다면 일부 Prefix만 Glacier로 이동
  - 나중에 비용/요구사항에 따라 아래처럼 조정 가능:
    - 운영 로그: 14~30일
    - 감사/보안 로그: 90일~180일 + Glacier

---

#### 7.2.5 온프레 로그/메트릭 전송량 관리

- **기본 원칙**
  - “온프레에서 나오는 모든 로그를 다 올린다”가 아니라,
    **운영에 꼭 필요한 정보만, 필요한 기간 만큼만 올린다.**

- **현재 전략**
  - filelog 수집 시:
    - 회전 로그/압축 로그(`*.gz`, `*.1`, `*.zip`, `*.tar` 등) 기본 제외
    - 불필요한 시스템 로그 경로는 수집 대상에서 제외 가능
  - 메트릭:
    - 수집 주기(예: 10~15초)를 현실적인 범위로 유지
    - 과도한 Custom Metrics 생성 지양

- **운영상 활용 패턴**
  - 문제 발생 시:
    - 해당 기간 동안만 특정 서비스의 로그 레벨를 DEBUG로 상향
    - 원인 분석 후 다시 INFO로 회귀
  - 이렇게 하면,
    - 평소에는 **INFO 수준 + 주요 로그만 전송**
    - 이슈 발생 시에만 **일시적으로 자세한 로그** 확보

---

### 7.3 컴퓨팅 리소스(EKS/EC2) 측면 최적화

- **노드 그룹 크기 조정**
  - Auto Scaling 설정에서 `min/desired/max` 값을
    실제 트래픽/리소스 사용량 기준으로 조절
  - 프로젝트 성격상 “24/7 고부하 서비스”가 아니므로,
    **최소 노드 수를 낮게 유지**하는 것이 비용에 직접적 영향을 줌

- **리소스 요청/리밋 튜닝**
  - Deployment의 `requests/limits` 를 실제 사용량에 맞춰 조정
  - 과도한 request 설정은 불필요한 노드 스케일업으로 이어져 비용 증가

- **모니터링 워크로드의 우선 순위**
  - 모니터링 스택(Prometheus/Loki/Grafana/OTel)은
    “서비스 가시성을 위한 필수 요소”이지만,
    리소스 폭주 시 전체 클러스터를 먹지 않도록 리밋 설정으로 보호

---
