# Devths — Cloud Infrastructure (fork)

> 카카오 테크 부트캠프 3기 팀 프로젝트 **Devths**(개발자 취업 AI 플랫폼, 2026.01 – 2026.03)의 인프라 레포 포크입니다.
> 원본: [100-hours-a-week/9-team-Devths-CLOUD](https://github.com/100-hours-a-week/9-team-Devths-CLOUD) · 서비스: [devths.com](https://devths.com) · 팀 위키: [Cloud & Infra WIKI](https://github.com/100-hours-a-week/9-team-Devths-WIKI/wiki/Cloud-&-Infra-WIKI)
>
> Cloud 2인 중 1인으로 참여했습니다. 아래 **내가 한 일**에 제 담당 영역과 경로를 정리했습니다.

## 아키텍처 요약

| 단계 | 구성 |
|---|---|
| V1 · 2026.01 | 단일 EC2 · CodeDeploy 배포 |
| V2 · 2026.02 | 3-Tier(ALB + ASG) · 컨테이너 · Private Subnet · SSM · RDS |
| V3 · 2026.03 | K8s(kubeadm, 운영 6 · 비운영 3노드) · Helm · Kustomize · ArgoCD(GitOps) |

| 디렉터리 | 내용 |
|---|---|
| `terraform/` | AWS 리소스(VPC · ALB/ASG · RDS · ElastiCache · ECR · CodeDeploy · Route53 · S3 · SSM), 스택별 S3 원격 state |
| `k8s-helm/` | 플랫폼 차트(Gateway API · cert-manager · NetworkPolicy) · 릴리스(Calico · ArgoCD · External Secrets · 모니터링) |
| `k8s-kustomize/` | BE · FE · AI 서비스 base / dev · stg · prod overlay |
| `argocd/` | 환경별 root app(app-of-apps) |
| `monitoring/` | EC2 기반 운영 · 비운영 관측 스택(Prometheus · Loki · Tempo · Grafana · Alertmanager) |
| `migration_scripts/` | 자체 PostgreSQL → RDS 무중단 이전 스크립트 |
| `loadTest/` · `mock/` | k6 부하 시나리오 · WireMock 외부 의존 모사 |

## 내가 한 일

### 1. 운영 DB 무중단 이전 — 단독
- 운영 중인 EC2 자체 설치 PostgreSQL 14 → RDS PostgreSQL 14. 논리 복제 · 앱 먼저/DB 나중 2단계 컷오버 · 시퀀스 버퍼 동기화
- **컷오버 중 에러율 0% · 데이터 유실 0건**(k6 20 VU 기준), 개발 → 스테이징 → 운영 순서로 3회 수행
- 코드: [`migration_scripts/`](./migration_scripts) — lag 감시(`lag_monitor.sh`), 시퀀스 동기화(`optimize_sequences.sh`), 고아 복제 슬롯 정리(`drop_orphan_slot.sql`), 컷오버 후 감시 · 롤백 데이터 정리 (커밋 `83e6fa7`, PR #34)
- 문서: [설계 · 결과(WIKI)](https://github.com/100-hours-a-week/9-team-Devths-WIKI/wiki/DB-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98(Henry)) · 고아 슬롯로 WAL이 쌓인 트러블슈팅 [Issue #304](https://github.com/100-hours-a-week/9-team-Devths-WIKI/issues/304)

### 2. K8s 관측 스택(PLG + OTel) · 대시보드 — 구축 담당(설계 · SLO는 공동)
- 비운영 클러스터에 kube-prometheus-stack · Loki(S3) · Tempo · Alloy 구성, BE · AI 서비스 ServiceMonitor 연결
- Grafana 대시보드 **7개**(인프라 개요 · K8s 워크로드 · Spring Boot · K8s 로그 · AI 서비스 · 분산 트레이스 · BE 상세)
- 알림 webhook URL을 git 추적에서 빼고 런타임 생성으로 전환(`8fcaf6d`)
- 코드: [`k8s-helm/releases/monitoring/`](./k8s-helm/releases/monitoring) · [`monitoring/nonprod/`](./monitoring/nonprod) · Loki S3 · 모니터링 EC2 Terraform(`38396b8`) · PR #94 ~ #104
- `monitoring/` 커밋 13개 중 11개
- 문서: [SLI/SLO 정의(WIKI)](https://github.com/100-hours-a-week/9-team-Devths-WIKI/wiki/SLI-SLO) · Pod 편중 장애 [#307](https://github.com/100-hours-a-week/9-team-Devths-WIKI/issues/307) · [#308](https://github.com/100-hours-a-week/9-team-Devths-WIKI/issues/308)

### 3. K8s 네트워크 설정 — 최초 작성
- Gateway(NGINX Gateway Fabric) · HTTPRoute · cert-manager ClusterIssuer/Certificate · NetworkPolicy 초기 구성(이슈 #38 · PR #39 · #40 · #41)
- 백엔드 → 메시지 브로커 Egress 개방(이슈 #42 · PR #43)
- 이후 팀원이 Helm 차트 [`k8s-helm/charts/platform-chart/`](./k8s-helm/charts/platform-chart)로 옮겨 다듬었습니다.

### 4. AI 서비스 K8s 매니페스트
- AI API · Celery worker/beat 매니페스트와 dev · stg overlay, AI 네트워크 정책(이슈 #71 · PR #79)

### 5. V1 인프라(현재 `main` 이력에는 없음, [`archive/260203`](https://github.com/100-hours-a-week/9-team-Devths-CLOUD/tree/archive/260203))
- EC2 user_data · KMS 변수화(PR #6), logrotate(PR #9), CodeDeploy 자동 롤백(PR #11), AI 모의 면접 E2E(Playwright, PR #15)
<!-- 확인 필요: V1 Terraform(PR #4 의 VPC · 서브넷 · IAM(SSM) · EC2)을 팀 구성으로 그대로 썼는지, 개인 실습 후 팀 코드로 대체했는지(PR #6 에서 devths-infra-henry/ 삭제) -->

## 공동 작업

- **Terraform**: 팀원과 공동으로 설계 · 작성했고, 커밋은 주로 팀원 계정으로 남아 있습니다.
  <!-- 확인 필요: 함께 작성한 스택 · 모듈 이름(예: terraform/infrastructure/... , terraform/modules/...) -->
- **배포 파이프라인(GitOps 전환)**: 앱 · 운영 레포 분리 → 이미지 빌드 · ECR → 이미지 태그 갱신 → ArgoCD 동기화, 비운영 자동 / 운영 수동 승인. [CD 설계(WIKI)](https://github.com/100-hours-a-week/9-team-Devths-WIKI/wiki/CD)
  <!-- 확인 필요: 이 레포 argocd/ 커밋은 모두 팀원 계정. 본인 담당(AI 파이프라인 · 비운영 ArgoCD)이 어느 레포 · PR 에 있는지 -->
- **부하 시나리오**: 시나리오 설계는 본인, k6 스크립트 · 실행은 팀원(`loadTest/`)

## 성과

| 항목 | 수치 | 단서 |
|---|---|---|
| DB 무중단 이전 | 컷오버 중 에러율 0% · 데이터 유실 0건 | 단독 |
| 배포 반영 시간 | PR 머지 → 서버 반영 약 20분 → 5~8분 | 공동 · 서비스별 자체 측정 |
| 관측 | 대시보드 7개 · 2단계(critical · warning) Discord 알림 | SLI/SLO · 알림 설계 공동 |
| 장애 대응 | Pod 편중 장애 후 노드 메모리 40% 이하 안정화 | 비운영 클러스터 · 운영 배포 전 포착 |

## 관련 문서

- 팀 위키: [Cloud & Infra WIKI](https://github.com/100-hours-a-week/9-team-Devths-WIKI/wiki/Cloud-&-Infra-WIKI)
- 포트폴리오 주제: D1 DB 무중단 이전 · D2 K8s 장애와 관측 · D3 배포 <!-- 확인 필요: 포트폴리오 PDF 공개 링크 -->
