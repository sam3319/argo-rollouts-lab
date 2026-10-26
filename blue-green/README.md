# Argo Rollouts Blue-Green 실습

기존 버전과 신규 버전을 동시에 실행하고, 신규 버전을 확인한 뒤 Active Service의 연결 대상을 전환하는 실습입니다. Argo CD는 Git의 YAML을 적용하고, Argo Rollouts는 버전별 Pod와 Service 전환을 관리합니다.

## 리소스 구성

```text
bluegreen-lab/
├── rollouts.yaml
├── service-active.yaml
├── service-preview.yaml
└── README.md
```

| 파일 | 리소스 이름 | 역할 |
| --- | --- | --- |
| `rollouts.yaml` | `rollout-bluegreen` | Pod 2개와 Blue-Green 배포 전략 정의 |
| `service-active.yaml` | `rollout-bluegreen-active` | 현재 서비스 중인 버전에 연결 |
| `service-preview.yaml` | `rollout-bluegreen-preview` | 전환 전 신규 버전을 확인 |

모든 리소스의 네임스페이스는 `argo-rollouts`입니다. 기본 Pod label과 Service selector는 `app: nginx-bluegreen`입니다. 컨트롤러가 Service selector에 ReplicaSet 해시를 추가해 각 버전으로 요청을 나눕니다. 별도의 Deployment나 ReplicaSet 파일을 작성하지 않습니다.

현재 초기 이미지 태그는 `1.0.0`이며 컨테이너 포트는 `80`입니다. Service의 `targetPort: http`는 컨테이너의 이름 있는 포트 `http`를 참조합니다. `autoPromotionEnabled: false`이므로 신규 버전의 준비가 끝나면 수동 승격을 기다립니다. 최초 배포는 기존 Active 버전이 없으므로 업데이트 때와 다르게 바로 서비스가 시작됩니다.

## 사전 준비

- Kubernetes 클러스터에 Argo Rollouts 컨트롤러와 CRD 설치
- kubectl의 Argo Rollouts 플러그인 설치
- Argo CD 설치 및 해당 클러스터 접근 가능
- 이미지 레지스트리에 초기 버전 `1.0.0`과 신규 버전 `1.0.1` 보관
- 신규 버전 준비 중 기존 버전과 함께 실행할 수 있는 클러스터 여유 자원 확보

현재 replicas 2개, 별도 previewReplicaCount가 없는 구성에서는 업데이트 중 기존 2개와 신규 2개의 Pod가 함께 실행될 수 있습니다.

클러스터에서 확인합니다.

```bash
kubectl get crd rollouts.argoproj.io
kubectl get pods -n argo-rollouts
kubectl argo rollouts version
```

처음에는 Image Updater를 이번 Application에 연결하지 않습니다. Git에서 직접 이미지 태그를 변경하며 전환 과정을 확인합니다.

## 1. YAML 검증 후 Git에 Push

이 디렉터리에서 실행합니다.

```bash
kubectl apply --dry-run=server -f rollouts.yaml -f service-active.yaml -f service-preview.yaml
```

이 명령은 API 서버에서 검증만 수행합니다. 실제 배포는 이후 Argo CD의 Sync로 진행합니다. Git 저장소를 만들고 세 YAML과 이 README를 커밋한 뒤 Push하세요.

## 2. Argo CD Application 생성

웹 UI의 NEW APP에서 설정합니다.

| 항목 | 설정 |
| --- | --- |
| Application Name | 예: `nginx-bluegreen-lab` |
| Project | `default` 또는 해당 배포를 허용하는 프로젝트 |
| Repository URL | 직접 생성한 Git 저장소 주소 |
| Revision | `main` |
| Path | 세 YAML이 저장소 루트에 있다면 `.` |
| Source 유형 | `Directory` |
| Destination Cluster | 같은 클러스터라면 `https://kubernetes.default.svc` |
| Destination Namespace | `argo-rollouts` |
| Sync Policy | Git 변경을 자동 반영하려면 Automatic |

별도의 Helm 파라미터는 필요하지 않습니다. Application 이름과 Rollout 리소스 이름은 서로 다릅니다. 이후 kubectl argo rollouts 명령에는 Application 이름이 아니라 `rollout-bluegreen`을 사용합니다.

## 3. 초기 배포 확인

```bash
kubectl argo rollouts get rollout rollout-bluegreen -n argo-rollouts --watch
kubectl get pods -n argo-rollouts -l app=nginx-bluegreen
kubectl get svc rollout-bluegreen-active rollout-bluegreen-preview -n argo-rollouts
```

watch 명령은 계속 실행되므로 다른 터미널에서 다음 명령을 실행하거나 Ctrl+C로 종료하세요.

초기 상태에서는 두 Service가 모두 `1.0.0` 버전으로 연결되는 것이 정상입니다.

```text
Active Service  → 1.0.0 Pod
Preview Service → 1.0.0 Pod
```

## 4. Active와 Preview 화면 확인

서로 다른 터미널에서 실행하고 연결을 유지합니다.

```bash
kubectl port-forward -n argo-rollouts svc/rollout-bluegreen-active 8080:80
```

```bash
kubectl port-forward -n argo-rollouts svc/rollout-bluegreen-preview 8081:80
```

- Active: `http://localhost:8080`
- Preview: `http://localhost:8081`

localhost는 명령을 실행한 컴퓨터입니다. VM에서 실행하고 개인 PC에서 보려면 PC에서 다음 SSH 터널을 추가로 실행하세요.

```bash
ssh -L 8080:127.0.0.1:8080 -L 8081:127.0.0.1:8081 <SSH사용자>@<VM_IP>
```

Service에 대한 port-forward도 시작 시 선택한 하나의 Pod로 연결됩니다. Service selector 전환을 자동으로 따라가는 접속 방법은 아니므로, 업데이트나 승격 뒤에는 해당 port-forward를 종료하고 다시 실행하세요. 실제 Service 연결 대상은 아래의 selector와 EndpointSlice로 확인할 수 있습니다.

## 5. 신규 버전 배포

`rollouts.yaml`의 컨테이너 이미지 태그를 `1.0.0`에서 `1.0.1`로 변경하고 Commit/Push합니다. Docker Hub에 `1.0.1`이 이미 있다면 다시 빌드할 필요가 없습니다.

Argo CD에서 최신 커밋이 Sync됐는지 확인한 뒤 Rollout 상태를 관찰하세요.

```bash
kubectl argo rollouts get rollout rollout-bluegreen -n argo-rollouts --watch
```

신규 Pod가 Ready가 되면 수동 승격을 기다리는 상태가 됩니다. Argo CD에서 Suspended로 표시될 수 있으며, 이때는 의도한 대기 상태인지 Rollout 메시지로 확인합니다.

```text
Active Service  → 기존 1.0.0
Preview Service → 신규 1.0.1
```

Preview의 port-forward를 다시 실행하고 신규 화면을 확인하세요. Pod 이미지도 확인합니다.

```bash
kubectl get pods -n argo-rollouts -l app=nginx-bluegreen -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image'
```

## 6. 수동 승격

신규 버전 검증을 마치면 실행합니다.

```bash
kubectl argo rollouts promote rollout-bluegreen -n argo-rollouts
```

Active Service가 신규 ReplicaSet을 가리키도록 전환됩니다. 기존 Pod는 즉시 없어지는 것이 아니라 scale-down 지연 후 축소됩니다. Active port-forward를 다시 실행해 신규 화면을 확인하세요.

```bash
kubectl get svc rollout-bluegreen-active rollout-bluegreen-preview -n argo-rollouts -o wide
kubectl get endpointslices -n argo-rollouts -l kubernetes.io/service-name=rollout-bluegreen-active
kubectl argo rollouts get rollout rollout-bluegreen -n argo-rollouts
```

## 7. 기존 이미지로 시작하기와 롤백

Docker Hub는 이미지를 보관하는 레지스트리입니다. 최신 태그가 `1.0.1`이어도 `1.0.0` 태그를 삭제하거나 덮어쓰지 않았다면 둘은 함께 존재합니다. 어떤 태그를 실행할지는 Rollout의 이미지 설정이 결정합니다.

따라서 이번 새 실습을 `1.0.0`으로 시작하는 것은 이전 Application을 롤백하는 작업이 아닙니다. 이 디렉터리의 Rollout이 `1.0.0`을 지정한 상태로 신규 배포되는 것입니다. 이전 실습의 클러스터 리소스나 Docker Hub 이미지를 삭제할 필요가 없습니다. 이전 태그가 없다면 동일한 원본 소스/빌드 정보로 복원해야 하며, 신규 이미지를 단순히 이전 태그로 바꾸는 것은 이전 버전 복원이 아닙니다.

### GitOps 방식의 롤백

이미 승격한 이번 Rollout을 `1.0.0`으로 되돌리려면:

1. Git의 `rollouts.yaml` 이미지 태그를 `1.0.0`으로 변경합니다.
2. 변경을 Commit/Push합니다. 이미지 변경 커밋만 되돌리는 git revert도 사용할 수 있습니다.
3. Argo CD가 이 커밋을 Sync하도록 합니다.
4. Rollout 진행 상태를 확인하고, 승격 대기 상태라면 검증 후 promote합니다.

이전 ReplicaSet 보관 상태와 롤백 설정에 따라 전환 과정은 달라질 수 있습니다. Git에 원하는 이전 버전을 기록해 두는 것이 Argo CD 관리 상태와 일치하는 방법입니다.

### 승격 전 신규 버전 중단

신규 Preview에 문제가 있고 아직 승격하지 않았다면 Active는 기존 버전을 계속 서비스합니다. 진행 중인 업데이트는 다음 명령으로 중단할 수 있습니다.

```bash
kubectl argo rollouts abort rollout-bluegreen -n argo-rollouts
```

abort는 Git의 이미지 설정을 수정하지 않습니다. 이후 Git의 원하는 이미지 태그도 안정 버전으로 되돌려 상태를 일치시키세요.

### CLI undo와 Git의 관계

Argo Rollouts에는 이전 리비전으로 되돌리는 명령도 있습니다.

```bash
kubectl argo rollouts undo rollout-bluegreen -n argo-rollouts
```

보관된 이전 리비전이 있어야 하며 Git은 수정되지 않습니다. Argo CD self-heal이 켜져 있으면 Git에 남아 있는 신규 설정이 다시 적용될 수 있으므로, 이번 GitOps 실습에서는 Git의 이미지 태그 변경을 기본 롤백 방법으로 사용합니다.

## 참고 문서

- [Blue-Green 전략](https://argo-rollouts.readthedocs.io/en/stable/features/bluegreen/)
- [Rollouts 기본 실습](https://argo-rollouts.readthedocs.io/en/stable/getting-started/)
- [Rollouts undo](https://argo-rollouts.readthedocs.io/en/stable/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_undo/)
- [Rollouts abort](https://argo-rollouts.readthedocs.io/en/stable/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_abort/)
