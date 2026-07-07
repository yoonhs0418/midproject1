# 금융 웹 서비스를 위한 DevOps 구축 (Mid Project 1)

> Kubernetes 기반 금융 웹 서비스의 고가용성(HA) 인프라 구축 및 CI/CD·GitOps 배포 자동화

- **기간** : 2024. 08. 12 ~ 2024. 08. 19
- **인원** : 3명
- **스택** : `Kubernetes` `Docker` `GitHub Actions` `ArgoCD` `NFS` `Grafana` `Node.js(Express)` `MySQL`

---

## 1. 프로젝트 개요

금융 웹 서비스(마이뱅크)를 **무중단으로 운영할 수 있는 인프라**를 목표로,
단일 서버 배포에서 출발해 다중 노드 Kubernetes 클러스터 위의 **이중화 + 배포 자동화** 구성까지 고도화한 프로젝트입니다.

애플리케이션은 회원 인증(네이버 소셜 로그인), 자산 관리, 부동산 조회, 챗봇 기능을 제공하는
Frontend / Backend / Database 3-Tier 구조입니다.

## 2. 아키텍처

```
                      ┌─────────────────────────────────────────┐
 GitHub (main push)   │           Kubernetes Cluster            │
   │                  │                                         │
   ▼                  │  Ingress ──► Frontend ──► Backend ──► MySQL
 GitHub Actions       │                │ HPA        │ HPA       │
   │  Docker Build    │                └── Grafana Monitoring   │
   ▼                  │                                         │
 Docker Hub ◄─────────│──── ArgoCD (GitOps Sync)                │
                      │         NFS Server (데이터 영속화)        │
                      └─────────────────────────────────────────┘
```

## 3. 아키텍처 결정 이유 (Why)

### 왜 다중 노드 / 다중 파드 이중화인가?
금융 서비스는 짧은 다운타임도 사용자 신뢰 문제로 직결됩니다.
단일 노드·단일 파드 구성은 노드 장애가 곧 전체 서비스 중단(SPOF)이 되기 때문에,
**노드 레벨과 파드 레벨 양쪽에서 이중화**하여 가용성을 확보했습니다.

### 왜 GitHub Actions인가?
3명 규모 팀에서 Jenkins 같은 별도 CI 서버를 직접 운영하는 것은 관리 비용이 과하다고 판단했습니다.
코드가 있는 GitHub에서 빌드까지 한 플랫폼으로 처리하여 파이프라인 유지 부담을 최소화했습니다.
`docker/build-push-action` 기반으로 **Frontend / Backend / Database 3개 이미지를 push 시 자동 빌드·배포**합니다.

### 왜 ArgoCD(GitOps)인가?
CI에서 `kubectl apply`로 밀어 넣는 push 방식은 시간이 지나면 Git의 선언 상태와
클러스터 실제 상태가 어긋나는 **drift** 문제가 생깁니다.
ArgoCD는 Git을 단일 진실 원천(Single Source of Truth)으로 삼아 상태를 지속 동기화하고,
배포 현황을 UI로 시각화할 수 있어 롤백과 운영 관점에서 유리했습니다.

### 왜 NFS인가?
온프레미스 성격의 클러스터라 클라우드 관리형 스토리지(EBS 등)를 사용할 수 없었고,
Ceph·Longhorn 같은 분산 스토리지는 소규모 클러스터에는 운영 복잡도가 과합니다.
NFS는 구조가 단순하면서 **ReadWriteMany** 접근이 가능해,
파드가 어느 노드로 재스케줄되어도 MySQL 데이터가 유지되는 영속성 환경을 확보했습니다.
(`PersistentVolume` + `PersistentVolumeClaim`, `storageClassName: nfs-storage`)

## 4. 핵심 구현

| 구분 | 내용 |
|------|------|
| CI | GitHub Actions — main push 시 Frontend/Backend/Database 이미지 자동 빌드 → Docker Hub 푸시 |
| CD | ArgoCD — Git 매니페스트 기준 클러스터 상태 자동 동기화(GitOps) |
| 오토스케일링 | HPA — Backend CPU 40% 기준, 1~3 레플리카 자동 확장 |
| 데이터 영속성 | NFS 서버 구축, PV/PVC(ReadWriteMany)로 MySQL 데이터 유지 |
| 모니터링 | Grafana 대시보드 구성 (Ingress로 외부 노출) |
| 라우팅 | Ingress 기반 서비스 라우팅 |

## 5. 트러블슈팅 — CoreDNS 네트워크 이슈

- **현상** : ArgoCD `argocd-repo-server`와 Git 저장소 간 통신 실패로 배포 자동화 중단
- **원인** : CoreDNS가 내부 도메인(`argocd-repo-server`)을 IP로 해석하지 못하는 DNS 해석 오류
- **해결** : `kubectl rollout restart`로 CoreDNS 파드를 재기동하여 네트워크 모듈 정상화 후,
  ArgoCD 통신 및 배포 프로세스 복구 확인
- **배운 점** : 쿠버네티스 제어 평면의 핵심인 CoreDNS의 동작 원리를 이해하고,
  네트워크 이슈 발생 시 상태를 진단·해결하는 경험을 쌓음

## 6. 디렉토리 구조

```
├── .github/workflows/   # GitHub Actions CI 파이프라인
├── Frontend/            # Node.js(Express) + EJS 프론트엔드
├── Backend/             # Node.js(Express) REST API 서버
├── Database/            # MySQL 초기화(Dockerfile, create.sql)
├── kubernetes/          # Deployment/Service/HPA/Ingress 매니페스트
└── grafana/             # Grafana 배포 및 NFS PV/PVC 매니페스트
```
