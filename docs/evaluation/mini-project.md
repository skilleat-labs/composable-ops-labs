# 미니 프로젝트 평가

!!! info "예상 소요 1시간 ~ 1시간 30분"

---

!!! quote "김팀장"
    *"실습 때 한 걸 다시 해봐. 근데 이번엔 다른 서비스야. 똑같이 할 수 있으면 네가 진짜 이해한 거야."*

---

## 개요

실습에서 구성한 CI/CD 파이프라인을 **새로운 서비스**에 처음부터 적용합니다.
코드를 작성하지 않습니다. **파이프라인 구성 능력**을 평가합니다.

```
[주어진 것]
hanbat-notification-api 레포
  ✅ Dockerfile 완성
  ✅ k8s/ yaml 완성
  ❌ .github/workflows/ 비어있음  ← 직접 작성

[해야 할 것]
AKS 재생성 → OIDC 설정 → ci.yml 작성 → ArgoCD 연결 → git push → 자동 배포 확인
```

---

## 채점 기준

| Task | 배점 | 만점 조건 |
|------|------|-----------|
| Task 1. AKS + ArgoCD 재설치 | 20점 | 노드 Ready + ArgoCD UI 접속 |
| Task 2. CI 파이프라인 구성 | 30점 | Actions 성공 + 태그 자동 업데이트 커밋 |
| Task 3. ArgoCD 배포 | 30점 | Synced + Healthy |
| Task 4. 자동 배포 확인 | 20점 | `"version": "v1.1"` 응답 확인 |
| **합계** | **100점** | |

!!! success "70점 달성 기준"
    Task 1 ~ Task 3 완료 시 80점입니다.

---

## 사전 준비

### 레포 Fork

[skilleat-labs/hanbat-notification-api](https://github.com/skilleat-labs/hanbat-notification-api) 를 본인 GitHub 계정으로 Fork합니다.

GitHub → 레포 → **Fork** → 본인 계정 선택

!!! warning "레포가 Public인지 확인하세요"
    ArgoCD가 Git에 접근하려면 레포가 Public이어야 합니다.
    Fork한 레포 메인 페이지에서 **Public** 뱃지를 확인하세요.

### 환경변수 설정

```bash title="터미널"
RG="hanbat-s2-lab-rg"
AKS_NAME="<본인_AKS_이름>"
ACR_NAME="<본인_ACR_이름>"
GITHUB_USER="<본인_GitHub_사용자명>"
```

---

## Task 1. AKS + ArgoCD 재설치 (20점)

기존 AKS를 삭제하고 새 클러스터에 ArgoCD를 설치합니다.

!!! warning "ACR은 삭제하지 마세요"
    기존 ACR을 재사용합니다. GitHub Secrets도 그대로 유지하세요.

**Step 1. 기존 AKS 삭제**

```bash title="터미널"
az aks delete --resource-group $RG --name $AKS_NAME --yes --no-wait
```

**Step 2. 새 AKS 생성**

```bash title="터미널"
az aks create \
  --resource-group $RG \
  --name $AKS_NAME \
  --node-count 2 \
  --node-vm-size Standard_B2s_v2 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --network-plugin azure
```

**Step 3. kubeconfig + nginx Ingress + ArgoCD 설치**

Phase 4-1 가이드를 참고하여 순서대로 설치합니다.

```bash title="터미널"
az aks get-credentials --resource-group $RG --name $AKS_NAME

# nginx Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# ArgoCD
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer

# ACR 연결
az aks update --resource-group $RG --name $AKS_NAME --attach-acr $ACR_NAME
```

!!! success "✅ Task 1 제출"
    아래 명령어 결과를 스크린샷으로 제출합니다.
    ```bash title="터미널"
    kubectl get nodes
    kubectl get svc argocd-server -n argocd
    ```

---

## Task 2. CI 파이프라인 구성 (30점)

Fork한 레포에 `.github/workflows/ci.yml` 을 직접 작성합니다.

**조건**

- OIDC 인증 (비밀키 방식 사용 금지)
- `notification-api` 이미지 빌드 → ACR 푸시
- `k8s/deployment.yaml` 이미지 태그 자동 업데이트
- 매니페스트 자동 커밋 (`[skip ci]`)

!!! tip "힌트"
    실습에서 작성한 `hanbat-order-app-s2`의 `ci.yml` 구조를 참고하세요.
    `order-api` → `notification-api` 로 바꾸는 것이 핵심입니다.

```bash title="터미널 — 작성 후 push"
git add .github/workflows/ci.yml
git commit -m "ci: add CI pipeline"
git push origin main
```

!!! success "✅ Task 2 제출"
    GitHub → Actions 탭에서 워크플로우 성공 스크린샷을 제출합니다.
    Git 커밋 히스토리에 `[skip ci]` 커밋이 생긴 것도 확인하세요.

---

## Task 3. ArgoCD Application 등록 (30점)

`k8s/argocd-application.yaml` 의 placeholder를 교체하고 ArgoCD에 등록합니다.

**Step 1. placeholder 교체**

```bash title="터미널"
cd ~/hanbat-notification-api   # 또는 Fork한 레포 경로

sed -i "s|<본인_GitHub_사용자명>|${GITHUB_USER}|g" k8s/*.yaml
grep -E "name:|repoURL:|namespace:" k8s/argocd-application.yaml
```

**Step 2. ArgoCD Application 등록**

```bash title="터미널"
kubectl apply -f k8s/argocd-application.yaml
```

**Step 3. ArgoCD UI 확인**

브라우저에서 ArgoCD UI 접속 → `hanbat-notification-<사용자명>` Application 카드 확인.

```bash title="터미널 — 초기 비밀번호 확인"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

!!! success "✅ Task 3 제출"
    ArgoCD UI에서 Application이 **Synced + Healthy** 상태인 스크린샷을 제출합니다.

---

## Task 4. git push → 자동 배포 확인 (20점)

코드를 수정하고 push하여 전체 파이프라인이 자동으로 동작하는지 확인합니다.

**Step 1. 코드 수정**

`app/main.py`의 `/health` 응답에 `version` 필드를 추가합니다.

```python title="app/main.py (수정)"
@app.get("/health")
async def health():
    return {"status": "ok", "service": "notification-api", "version": "v1.1"}
```

**Step 2. Push**

```bash title="터미널"
git add app/main.py
git commit -m "feat: add version field to health endpoint"
git push origin main
```

**Step 3. 파이프라인 관찰**

1. GitHub → Actions 탭에서 빌드 확인
2. ArgoCD UI에서 OutOfSync → Synced 전환 확인

**Step 4. 새 버전 확인**

```bash title="터미널"
kubectl port-forward svc/notification-api 8889:8080 -n hanbat-${GITHUB_USER} &
curl http://localhost:8889/health
```

```json title="출력 예시 (새 버전)"
{"status": "ok", "service": "notification-api", "version": "v1.1"}
```

!!! success "✅ Task 4 제출"
    - ArgoCD UI에서 자동 Sync 완료 스크린샷
    - `curl` 결과에서 `"version": "v1.1"` 확인 스크린샷

---

## 제출 방법

1. Fork한 레포 URL (예: `https://github.com/keepcraft-go/hanbat-notification-api`)
2. Task 1~4 스크린샷을 하나의 문서(PPT / PDF / Notion)로 정리
3. 강사에게 제출

---

## 다음 단계

[:material-arrow-left: 평가 개요](index.md){ .md-button }
[제출 가이드 :material-arrow-right:](submission-guide.md){ .md-button .md-button--primary }
