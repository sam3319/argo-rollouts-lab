# Argo Rollouts 배포 전략 실습

Argo CD와 Argo Rollouts로 **Blue-Green**과 **Canary** 배포를 비교하는 학습용 저장소입니다. Git에서 배포할 이미지 태그를 변경하고 전략별 대기·검증·승격 과정을 관찰합니다.

## 실습 바로가기

| 전략 | 안내서 | 주요 관찰 내용 |
| --- | --- | --- |
| Blue-Green | [Blue-Green 실습](blue-green/README.md) | Preview 검증 후 Active Service 전환 |
| Canary | [Canary 실습](canary/README.md) | 20% 수동 대기 → 60%에서 2분 대기 → 100% 전환 |

## 저장소 구성

```text
rollouts/
├── README.md
├── blue-green/
│   ├── README.md
│   ├── rollouts.yaml
│   ├── service-active.yaml
│   └── service-preview.yaml
└── canary/
    ├── README.md
    ├── rollouts.yaml
    └── service.yaml
```

일반 YAML을 사용하는 구성으로 Helm 차트가 아닙니다. Argo CD Application 두 개가 동일 저장소의 서로 다른 경로를 관리합니다.

## 도구의 역할

| 도구 | 역할 |
| --- | --- |
| Git | 배포할 이미지와 리소스 설정 보관 |
| 이미지 레지스트리 | 실행할 컨테이너 이미지 보관 |
| Argo CD | Git 설정을 클러스터에 Sync |
| Argo Rollouts | ReplicaSet과 Pod를 관리하며 단계별 배포 처리 |
| kubectl Rollouts 플러그인 | 상태 확인, 대시보드, 승격·중단 |

이 실습에는 Image Updater 설정을 포함하지 않습니다. 먼저 Git에서 이미지 태그를 직접 변경해 배포 전략을 학습합니다.

## 전략 비교

| 항목 | Blue-Green | 기본 Canary |
| --- | --- | --- |
| Rollout 이름 | `rollout-bluegreen` | `rollout-canary` |
| Pod label | `app: nginx-bluegreen` | `app: nginx-canary` |
| replicas | 2 | 5 |
| Service | Active / Preview 두 개 | 공통 Service 한 개 |
| 업데이트 방식 | 신규 버전 검증 후 Active 연결 대상 전환 | 신규 버전 Pod 비중을 단계적으로 증가 |
| 수동 대기 | 신규 버전 승격 전 | 20% 단계 |
| 자동 진행 | 자동 승격 비활성화 | 60% 단계에서 2분 후 자동 진행 |

두 전략 모두 네임스페이스는 `argo-rollouts`, 컨테이너 포트는 `80`입니다. 이름과 label을 구분해 Service에 서로의 Pod가 섞이지 않게 합니다. 각 Application은 해당 경로의 실습 리소스만 관리합니다.

Canary에는 별도 트래픽 라우터가 없습니다. 비중은 Pod 개수로 근사하며 실제 HTTP 요청 비율을 정확히 보장하지 않습니다.

## 사전 준비

- Kubernetes 클러스터와 kubectl 접근 권한
- Argo CD 설치 및 대상 클러스터 연결
- Argo Rollouts 컨트롤러와 CRD 설치
- kubectl의 Argo Rollouts 플러그인 설치
- 웹 화면으로 구분 가능한 이미지 태그 `1.0.0`, `1.0.1`
- 업데이트 중 신규 Pod를 추가로 준비할 수 있는 여유 자원

```bash
kubectl get crd rollouts.argoproj.io
kubectl get pods -n argo-rollouts
kubectl argo rollouts version
```

이미지 주소는 각 rollouts.yaml에서 확인합니다. 현재 파일에 기록된 태그는 Blue-Green이 `1.0.1`, Canary가 `1.0.0`입니다. 실제 배포 상태는 클러스터에서 별도로 확인해야 합니다. 반복 실습 시 먼저 원하는 초기 태그로 배포를 안정화한 뒤 신규 태그로 변경하세요.

## Argo CD Application 설정

저장소를 Git에 Push한 뒤 웹 UI에서 각각 생성합니다.

| 항목 | Blue-Green | Canary |
| --- | --- | --- |
| Application Name | `nginx-bluegreen-lab` | `nginx-canary-lab` |
| Project | `default` 또는 허용된 프로젝트 | 동일 |
| Repository URL | 이 저장소의 Git 주소 | 동일 |
| Revision | `main` | `main` |
| Path | **`blue-green`** | **`canary`** |
| Source 유형 | `Directory` | `Directory` |
| Destination Cluster | 같은 클러스터라면 `https://kubernetes.default.svc` | 동일 |
| Destination Namespace | `argo-rollouts` | `argo-rollouts` |
| Sync Policy | 자동 반영을 원하면 Automatic | 동일 |

Application이 저장소 루트 전체를 재귀적으로 배포하게 하지 말고 각 실습 경로를 지정하세요. Helm 파라미터는 필요하지 않습니다.

Automatic Sync와 승격은 서로 다릅니다. Argo CD는 Git 변경을 Rollout에 적용하고, Rollouts는 설정된 대기·승격 규칙을 따릅니다. 따라서 Synced 상태에서도 Rollout은 수동 승인 대기로 Suspended/Paused 상태일 수 있습니다.

## 권장 진행 순서

1. 각 디렉터리의 YAML을 API 서버 dry-run으로 검증합니다.
2. 변경 사항을 Git에 Commit/Push합니다.
3. 해당 Application을 Sync해 초기 배포를 확인합니다.
4. Git에서 이미지 태그를 변경하고 다시 Commit/Push합니다.
5. 대시보드와 CLI에서 신규 Pod, 대기 단계, Service 연결을 확인합니다.
6. 필요한 시점에 수동 Promote하고 최종 상태를 확인합니다.

저장소 루트에서 실행하는 검증 명령입니다. 실제 리소스를 생성하지 않습니다.

```bash
kubectl apply --dry-run=server -f blue-green/
kubectl apply --dry-run=server -f canary/
```

단계별 상세 명령은 각 실습 README를 참고하세요.

## 공통 대시보드

클러스터에 접근 가능한 서버에서 실행합니다.

```bash
kubectl argo rollouts dashboard -n argo-rollouts
```

같은 컴퓨터의 브라우저에서 `http://localhost:3100/rollouts`에 접속합니다. VM에서 실행했다면 PC에 SSH 터널을 추가합니다.

```bash
ssh -N -L 3100:127.0.0.1:3100 <SSH사용자>@<VM_IP>
```

목록에서 `rollout-bluegreen` 또는 `rollout-canary`를 선택합니다. CLI에서도 확인할 수 있습니다.

```bash
kubectl argo rollouts get rollout rollout-bluegreen -n argo-rollouts
kubectl argo rollouts get rollout rollout-canary -n argo-rollouts
```

## 관찰과 롤백 시 참고

- Service port-forward는 시작 시 선택한 Pod 하나에 연결됩니다. Blue-Green 전환 후에는 다시 연결하고, Canary 요청 분산은 클러스터 내부에서 Service로 반복 요청해 관찰합니다.
- 최초 배포는 기존 버전이 없어 업데이트 단계를 건너뜁니다. 이미지 태그 등 Pod 템플릿을 변경해야 신규 배포가 시작됩니다.
- GitOps 롤백은 Git의 이미지 설정을 이전 버전으로 되돌리는 방법을 기본으로 합니다. 신규 이미지를 레지스트리에서 삭제할 필요가 없습니다.
- Promote, Abort, Undo는 Git 파일을 수정하지 않습니다. 특히 Abort/Undo 후에는 Git의 원하는 상태도 확인하세요.
- 인증 토큰, 비밀번호, kubeconfig, SSH 키는 저장소에 추가하지 않습니다.

## 참고 문서

- [Rollouts 기본 실습](https://argo-rollouts.readthedocs.io/en/stable/getting-started/)
- [Blue-Green 전략](https://argo-rollouts.readthedocs.io/en/stable/features/bluegreen/)
- [Canary 전략](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
- [Argo CD Directory 소스](https://argo-cd.readthedocs.io/en/stable/user-guide/directory/)
