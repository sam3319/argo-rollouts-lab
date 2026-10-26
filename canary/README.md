# Argo Rollouts Canary 실습

신규 이미지의 Pod 비중을 단계적으로 늘려 배포하는 기본 Canary 실습입니다. Argo CD가 Git의 리소스 설정을 적용하고 Argo Rollouts가 단계별 배포와 대기를 처리합니다.

## 구성

```text
canary/
├── rollouts.yaml
├── service.yaml
└── README.md
```

| 항목 | 현재 설정 |
| --- | --- |
| Rollout | `rollout-canary` |
| Service | `rollout-canary` |
| Namespace | `argo-rollouts` |
| replicas | `5` |
| Pod label / selector | `app: nginx-canary` |
| 초기 이미지 태그 | `1.0.0` |
| Service / 컨테이너 포트 | `80` / `80` |
| Service targetPort | 이름 있는 컨테이너 포트 `http` |

이미지는 이전 웹 서버 실습의 저장소를 재사용하며, 실제 주소는 `rollouts.yaml`에서 확인합니다. Blue-Green의 label과 이름을 공유하지 않으므로 같은 네임스페이스에서 구분해 실행할 수 있습니다. 별도 Deployment를 만들지 않고 Rollouts 컨트롤러가 ReplicaSet과 Pod를 관리합니다.

## 배포 단계

```text
초기 1.0.0 배포 완료
    ↓ Git의 이미지 태그를 1.0.1로 변경
신규 버전 20% 목표
    ↓ 수동 승인까지 대기
신규 버전 60% 목표
    ↓ 2분 대기 후 자동 진행
신규 버전 100%
```

- `pause: {}`: 사용자가 Promote할 때까지 무기한 대기합니다.
- `pause.duration: 2m`: 해당 대기 단계에 진입한 뒤 2분이 지나면 자동으로 다음 단계로 진행합니다.
- 최초 배포에서는 이전 버전이 없으므로 Canary 단계별 대기를 건너뛰고 초기 버전을 배포합니다.

이번 구성에는 Ingress나 서비스 메시를 통한 trafficRouting이 없습니다. setWeight를 Pod 개수로 근사하므로 실제 HTTP 요청 비율을 정확히 보장하지 않습니다. replicas 5개 기준으로 20%에서는 신규 1개, 60%에서는 신규 3개가 목표 비중의 예시입니다. 전환 중에는 surge 및 readiness 상태에 따라 Pod 개수가 일시적으로 달라질 수 있습니다.

## 1. 사전 확인과 YAML 검증

클러스터에 Argo Rollouts 컨트롤러와 CRD, kubectl 플러그인이 설치되어 있어야 합니다. 이미지 저장소에는 `1.0.0`, `1.0.1` 태그가 있어야 하며 두 이미지의 화면 표시 내용도 구분되어야 합니다.

클러스터에 접근 가능한 터미널에서 이 canary 디렉터리로 이동한 후 실행합니다.

```bash
kubectl get crd rollouts.argoproj.io
kubectl argo rollouts version
kubectl apply --dry-run=server -f rollouts.yaml -f service.yaml
```

마지막 명령은 실제 클러스터의 CRD와 API로 검증만 수행합니다. 리소스를 생성하지 않습니다. 문법 확인 뒤 Git에 Commit/Push하고 실제 배포는 Argo CD에서 진행하세요.

처음에는 Image Updater를 이 Application에 연결하지 않고 이미지 태그를 Git에서 직접 변경합니다.

## 2. Argo CD Application 설정

Git 저장소가 bluegreen과 canary 디렉터리로 나뉜 구성을 기준으로 합니다.

| 항목 | 설정 |
| --- | --- |
| Application Name | `nginx-canary-lab` |
| Project | `default` 또는 해당 소스와 대상을 허용하는 프로젝트 |
| Repository URL | 직접 만든 실습 Git 저장소 주소 |
| Revision | `main` |
| Path | `canary` |
| Source 유형 | `Directory` |
| Destination Cluster | 같은 클러스터라면 `https://kubernetes.default.svc` |
| Destination Namespace | `argo-rollouts` |
| Sync Policy | Git 변경을 자동 반영하려면 Automatic |

canary 파일만 별도 저장소의 루트에 올렸다면 Path는 `.`입니다. 기존 Blue-Green Application의 경로는 변경하지 않고 Canary Application을 추가합니다. Helm 파라미터는 필요하지 않습니다.

## 3. 초기 버전 확인

```bash
kubectl argo rollouts get rollout rollout-canary -n argo-rollouts --watch
kubectl get pods -n argo-rollouts -l app=nginx-canary -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image'
kubectl get svc rollout-canary -n argo-rollouts
```

watch는 실행 상태를 유지합니다. 다른 터미널에서 후속 명령을 실행하거나 Ctrl+C로 종료하세요. 초기 Pod가 모두 `1.0.0`이고 준비됐는지 확인합니다.

대시보드를 사용하려면 서버에서 실행합니다.

```bash
kubectl argo rollouts dashboard -n argo-rollouts
```

개인 PC에서 VM의 대시보드에 접근할 때는 PC에서 SSH 터널을 실행하고 `http://localhost:3100/rollouts`를 엽니다.

```bash
ssh -N -L 3100:127.0.0.1:3100 <SSH사용자>@<VM_IP>
```

대시보드에서 `rollout-canary`를 선택합니다.

## 4. 신규 버전 배포와 20% 단계 확인

Git의 `canary/rollouts.yaml`에서 컨테이너 이미지 태그를 `1.0.0`에서 `1.0.1`로 변경하고 Commit/Push합니다. Argo CD에서 최신 커밋이 Sync됐는지 확인하세요.

```bash
kubectl argo rollouts get rollout rollout-canary -n argo-rollouts --watch
```

신규 Pod가 준비된 후 20% 단계의 수동 승인 대기 상태가 됩니다. 기존/신규 ReplicaSet과 Pod 수를 관찰합니다. 이때 Paused 또는 Suspended 표시는 의도한 대기 상태일 수 있습니다.

```bash
kubectl get pods -n argo-rollouts -l app=nginx-canary -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image'
```

## 5. Service를 통한 두 버전 응답 확인

Service는 `app: nginx-canary` label을 가진 준비된 기존/신규 Pod 모두로 연결됩니다. `kubectl port-forward svc/rollout-canary ...`는 Pod 하나를 선택하므로 요청 분산 관찰에는 사용하지 않습니다.

Linux 클러스터 터미널에서 테스트용 Pod를 실행합니다. 다음 명령은 테스트 Pod를 실제로 하나 생성하며, 종료하면 제거합니다.

```bash
kubectl run canary-client -n argo-rollouts --image=busybox:1.36 --restart=Never --rm -it -- sh
```

테스트 Pod의 셸 안에서 반복 요청합니다.

```sh
for i in $(seq 1 30); do
  wget -qO- http://rollout-canary | grep 'class="version"'
  sleep 1
done
```

이 명령은 이전 실습 HTML의 `class="version"` 줄을 출력합니다. HTML 구조가 다르면 grep 조건을 바꾸거나 제거하세요. 요청별로 1.0.0과 1.0.1 응답이 나오는지 확인하되, 30회 중 정확히 20%가 신규 버전일 것으로 기대하지 않습니다. 테스트를 마치면 `exit`로 종료합니다.

## 6. 60%로 승격하고 자동 완료 관찰

20% 검증을 마치면 실행합니다.

```bash
kubectl argo rollouts promote rollout-canary -n argo-rollouts
```

일반 Promote는 현재 수동 대기를 해제하고 다음 단계를 진행합니다. 60% 단계에서는 2분 대기를 거쳐 자동으로 100%로 진행하므로, 그 사이에 Pod 개수와 응답을 확인하세요. 모든 단계를 건너뛰는 Promote Full은 이번 단계별 관찰에서는 사용하지 않습니다.

```bash
kubectl argo rollouts get rollout rollout-canary -n argo-rollouts --watch
```

최종적으로 Rollout이 Healthy이고 실행 중인 웹 서버 Pod가 모두 `1.0.1`이면 완료입니다.

## 7. 중단과 롤백

진행 중인 신규 버전에 문제가 있으면 중단할 수 있습니다.

```bash
kubectl argo rollouts abort rollout-canary -n argo-rollouts
```

abort는 Git을 수정하지 않습니다. Git의 이미지 태그도 안정 버전으로 되돌려 원하는 상태를 일치시키세요. 완료된 배포를 되돌릴 때도 Git의 이미지 태그를 이전 버전으로 변경하고 Commit/Push하는 GitOps 흐름을 사용합니다. 이후 상태와 대기 단계를 확인합니다.

롤백을 위해 Docker Hub의 신규 이미지를 삭제할 필요는 없습니다. 이전 태그가 남아 있다면 그 이미지를 다시 선택할 수 있습니다.

## 검토 시 수정한 구조

- Rollout의 잘못된 `spec.spec` 대신 `spec.template.metadata`와 `spec.template.spec` 구조를 사용했습니다.
- Pod label을 추가해 Rollout selector 및 Service selector와 일치시켰습니다.
- strategy를 Pod 설정 내부가 아닌 `spec.strategy`로 이동했습니다.
- Service의 들여쓰기를 정리했으며 선택 조건과 포트 설정은 유지했습니다.
- 사용자가 작성한 20% 수동 대기, 60%에서 2분 자동 대기 정책을 유지했습니다.

## 참고 문서

- [Canary 전략](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
- [Rollouts 기본 실습](https://argo-rollouts.readthedocs.io/en/stable/getting-started/)
- [대시보드](https://argo-rollouts.readthedocs.io/en/stable/dashboard/)
