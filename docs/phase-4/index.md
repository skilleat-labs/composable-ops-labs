# Phase 4 · 사람 손 떼기 (2부)

!!! info "예상 소요 2시간 · 난이도 ⭐⭐⭐"

---

## 스토리

Phase 3을 마치고 나니 박대리는 뿌듯합니다. 이제 `git push`만 하면 이미지가 자동으로 빌드되고 ACR에 올라갑니다.

!!! quote "박대리"
    *"근데 팀장님, 이미지까지는 올라가는데 실제 AKS 배포는 아직 수동으로 `kubectl apply` 해야 하는 거 아니에요?"*

!!! quote "김팀장"
    *"맞아. 그래서 오늘 오후에 ArgoCD 세팅을 할 거야. 이게 소위 말하는 **GitOps**야. Git에 있는 YAML이 곧 클러스터의 상태가 되는 구조."*

---

## 핵심 질문

1. GitOps란 무엇이며, 왜 "pull" 방식인가?
2. ArgoCD의 **Application** 리소스는 무엇을 의미하나?
3. Sync Policy의 **manual vs automatic**은 어떤 차이를 만드나?
4. ArgoCD UI에서 보이는 "Out of Sync", "Synced", "Healthy"는 각각 무엇을 의미하나?

---

## 학습 목표

- [x] GitOps의 **4가지 원칙**(선언적, 버전 관리, 자동 적용, 지속 조정)을 설명할 수 있다
- [x] AKS에 ArgoCD를 Helm으로 설치하고 웹 UI에 접근한다
- [x] ArgoCD Application 리소스를 작성해 본인 GitHub 레포를 소스로 등록한다
- [x] 코드 변경 → `git push` → **손을 전혀 대지 않고** AKS에 새 버전이 뜨는 것을 UI로 확인한다

---

## 세부 단원

### Phase 4-1 · AKS 클러스터 확인 및 ArgoCD 설치 (30분)

| 소단원 | 활동 |
| --- | --- |
| ArgoCD 네임스페이스 생성 | `kubectl create namespace argocd` |
| Helm으로 ArgoCD 설치 | `helm repo add argo` → `helm install argocd argo/argo-cd` |
| LoadBalancer로 노출 | `argocd-server` Service를 `LoadBalancer` 타입으로 패치 |
| 초기 admin 비밀번호 확인 | `argocd-initial-admin-secret`에서 복호화 |
| 웹 UI 접속 | 공인 IP로 접속해 로그인 |

!!! tip "학습 환경에서는 port-forward도 OK"
    LoadBalancer가 IP 할당에 시간이 걸리거나 비용이 걱정되면 `kubectl port-forward svc/argocd-server -n argocd 8443:443`으로도 충분합니다.

### Phase 4-2 · ArgoCD Application 등록 (1시간)

| 소단원 | 활동 |
| --- | --- |
| Application 매니페스트 작성 | `k8s/argocd-application.yaml` 작성 |
| Source 지정 | 본인 GitHub 레포 URL + `k8s/` 경로 |
| Destination 지정 | in-cluster + `hanbat-<username>` 네임스페이스 |
| Sync Policy 설정 | automated + prune + selfHeal |
| Application 생성 | `kubectl apply -f k8s/argocd-application.yaml` |
| UI 확인 | ArgoCD Web UI에서 Application이 **Synced + Healthy** 상태인지 확인 |

### Phase 4-3 · git push → 자동 배포 체험 (30분)

**이 부분이 Phase 4의 하이라이트입니다.**

| 활동 | 기대하는 경험 |
| --- | --- |
| `order-api` 코드의 응답 메시지를 바꾼다 | 예: `"service": "order-api v2"` → `"service": "order-api v3"` |
| commit + push | 1초 안에 GitHub Actions가 시작 |
| ArgoCD UI를 지켜본다 | 약 2~3분 후 새 Pod이 뜨고 기존 Pod이 사라지는 애니메이션 관찰 |
| 실제 API 호출 | `curl http://<ingress>/api/orders/1` 로 새 버전 응답 확인 |
| 손을 쓰지 않았음을 확인 | 그 사이 **한 번도 `kubectl apply`를 치지 않았다** |

!!! success "GitOps 성공의 감동 포인트"
    이 순간을 위해 앞의 2시간 반을 투자한 겁니다. "코드 한 줄 바꾸고 push했더니 클러스터가 알아서 바뀐다." 이게 현대 배포의 본질입니다.

---

## 예상 결과물

- :material-kubernetes: AKS에 설치된 ArgoCD (UI 접속 가능)
- :material-file-code: 본인 레포에 커밋된 `k8s/argocd-application.yaml`
- :material-video: (선택) ArgoCD UI에서 자동 배포가 일어나는 모습 녹화 또는 스크린샷

---

## 이 단계에서 알아두면 좋은 트러블슈팅

??? failure "Application이 OutOfSync 상태로 멈춤"
    - Repository URL이 정확한지 확인 (`https://github.com/...git`)
    - ArgoCD가 비공개 레포에 접근하려면 Repository Credential 등록 필요
    - `kubectl -n argocd logs deployment/argocd-repo-server`로 Sync 에러 확인

??? failure "새 이미지가 pull 안 됨 (ImagePullBackOff)"
    - ACR에 Pod이 접근하려면 AKS에 ACR 연결 필요: `az aks update --attach-acr <ACR_NAME>`
    - 또는 imagePullSecret 수동 생성

---

## 다음 단계

[:material-arrow-left: Phase 3 · 사람 손 떼기 (1부)](../phase-3/index.md){ .md-button }
[Phase 5 · Agent란 무엇인가 :material-arrow-right:](../phase-5/index.md){ .md-button .md-button--primary }
