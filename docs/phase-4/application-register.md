# Phase 4-2 · ArgoCD Application 등록

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"ArgoCD에 Application을 등록하면 ArgoCD가 내 레포의 k8s/ 폴더를 계속 감시해. 거기서 변화가 생기면 클러스터에 자동으로 반영하는 거야."*

---

## 개념 — ArgoCD Application이란

ArgoCD의 **Application** 리소스는 다음 두 가지를 연결합니다.

```
[Git 소스]                        [배포 대상]
GitHub 레포 URL         →         AKS 클러스터
k8s/ 경로                         hanbat-<username> 네임스페이스
main 브랜치
```

ArgoCD는 3분마다 Git을 polling하여 매니페스트 변경을 감지하고 자동으로 Sync합니다.

| 상태 | 의미 |
| --- | --- |
| **Synced** | Git과 클러스터 상태가 일치 |
| **OutOfSync** | Git과 클러스터 상태가 다름 (새 배포 대기 중) |
| **Healthy** | 모든 Pod이 정상 실행 중 |
| **Degraded** | Pod이 CrashLoop 또는 Pending |

---

## Step 1. argocd-application.yaml 수정

레포의 `k8s/argocd-application.yaml` 파일에서 placeholder를 본인 정보로 교체합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2

# ⚠️ 반드시 실제 GitHub 사용자명으로 교체하세요
GITHUB_USER="keepcraft-go"   # 예시 — 본인 사용자명으로 변경

sed -i "s|<본인_GitHub_사용자명>|${GITHUB_USER}|g" k8s/argocd-application.yaml
```

치환이 잘 됐는지 반드시 확인합니다. `<본인_GitHub_사용자명>` 문자열이 남아있으면 안 됩니다.

```bash title="터미널"
grep -E "name:|repoURL:|namespace:" k8s/argocd-application.yaml
```

```yaml title="k8s/argocd-application.yaml (수정 후 예시)"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hanbat-s2-parkdaeri
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/parkdaeri/hanbat-order-app-s2
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: hanbat-parkdaeri
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

!!! note "syncPolicy.automated 의미"
    | 항목 | 의미 |
    | --- | --- |
    | `automated` | Git 변경 감지 시 자동 Sync (수동 클릭 불필요) |
    | `prune: true` | Git에서 삭제된 리소스를 클러스터에서도 삭제 |
    | `selfHeal: true` | 누군가 `kubectl` 로 직접 수정해도 Git 상태로 자동 복구 |

---

## Step 2. Application 생성

```bash title="터미널"
kubectl apply -f k8s/argocd-application.yaml
```

```text title="출력 예시"
application.argoproj.io/hanbat-s2-parkdaeri created
```

ArgoCD가 레포에 접근하려면 **공개 레포**여야 합니다.
GitHub 레포 메인 페이지(`github.com/<사용자명>/hanbat-order-app-s2`)에서 레포 이름 옆의 뱃지를 확인합니다.

!!! warning "Fork 레포는 visibility 변경 불가"
    Fork한 레포는 Settings → Danger Zone에서 visibility 변경이 불가합니다.
    뱃지가 **Private**으로 표시된다면 원본 레포 오너에게 Public 전환을 요청하거나,
    ArgoCD에 GitHub 인증 정보를 등록해야 합니다.

---

## Step 3. 초기 Sync 확인

브라우저에서 ArgoCD UI(`https://<ArgoCD_IP>`)에 접속합니다.

`hanbat-s2-<사용자명>` Application 카드가 생성된 것을 확인합니다.
카드를 클릭하면 Deployment, Service, Pod이 트리 형태로 보이며 실시간으로 상태가 업데이트됩니다.

처음엔 **OutOfSync** 또는 **Progressing** 상태입니다. k8s/ 폴더의 이미지 태그가 아직 실제 ACR 이미지를 가리키지 않기 때문입니다.

Phase 3에서 완성한 CI 파이프라인을 통해 이미지가 빌드되고 매니페스트 태그가 업데이트되면 자동으로 Sync됩니다.

아직 이미지가 ACR에 없다면 트리거 push를 합니다.

```bash title="터미널"
echo "# argocd test" >> order-api/app.py
git add order-api/app.py
git commit -m "ci: trigger first deployment"
git push origin main
```

GitHub Actions가 완료되길 기다립니다. (약 3~5분)

---

## Step 4. Synced + Healthy 확인

Actions 완료 후 ArgoCD UI에서 카드 상태가 변하는 걸 지켜봅니다.

```
OutOfSync → Syncing → Synced
Missing   →          Healthy
```

카드가 초록색(**Synced + Healthy**)으로 바뀌면 배포 완료입니다.
카드 클릭 → Pod 트리에서 각 Pod이 초록색 하트로 표시되는지 확인합니다.

터미널에서도 확인할 수 있습니다:

```bash title="터미널"
kubectl get pods -n hanbat-${GITHUB_USER}
```

```text title="출력 예시"
NAME                           READY   STATUS    RESTARTS
order-api-7d8f9b4c6-xk9pq     1/1     Running   0
payment-api-5c9d7b8f4-mn2lp   1/1     Running   0
order-web-6b8c7d9f3-jk4np     1/1     Running   0
```

!!! success "✅ 확인 포인트"
    - ArgoCD UI에서 Application 카드가 **Synced + Healthy** (초록색)
    - 카드 상세 화면에서 Deployment, Service, Pod 트리가 모두 초록색
    - `kubectl get pods` 에서 세 Pod 모두 `Running`

---

## 트러블슈팅

??? failure "OutOfSync 상태로 멈춤"
    ```bash
    kubectl -n argocd logs deployment/argocd-repo-server | tail -20
    ```
    - Repository URL이 `https://github.com/<username>/hanbat-order-app-s2` 형식인지 확인
    - 레포가 Public인지 확인

??? failure "ImagePullBackOff — 이미지를 못 가져옴"
    AKS와 ACR이 연결되지 않았습니다. 강사에게 ACR 연결을 요청합니다.
    또는 본인이 권한이 있다면:
    ```bash
    az aks update \
      --resource-group $RG \
      --name $AKS_NAME \
      --attach-acr $ACR_NAME
    ```

??? failure "git push 시 rejected (fetch first) 오류"
    ArgoCD의 자동 커밋과 충돌한 경우입니다.
    ```bash
    git stash && git pull --rebase origin main && git stash pop && git push origin main
    ```

??? failure "Pod이 CrashLoopBackOff"
    ```bash
    kubectl logs deployment/order-api -n hanbat-<username>
    ```
    로 에러 메시지를 확인합니다.

---

## 다음 단계

[:material-arrow-left: Phase 4-1 · ArgoCD 설치](argocd-install.md){ .md-button }
[Phase 4-3 · GitOps 자동 배포 체험 :material-arrow-right:](gitops-experience.md){ .md-button .md-button--primary }
