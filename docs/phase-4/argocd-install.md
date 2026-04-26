# Phase 4-1 · AKS 생성 및 ArgoCD 설치

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"ArgoCD는 Git을 계속 감시하다가 변화가 생기면 클러스터에 알아서 반영해. kubectl apply를 사람이 치는 게 아니라 ArgoCD가 치는 거야."*

---

## 환경변수 설정

이 페이지에서 사용할 변수를 먼저 설정합니다.

```bash title="터미널"
RG="hanbat-s2-lab-rg"
AKS_NAME="<본인_AKS_이름>"   # 예: hanbat-pd-aks
```

---

## Step 1. AKS 클러스터 생성

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

!!! warning "5~10분 소요"
    AKS 생성은 시간이 걸립니다. 완료되는 동안 아래 사전 준비를 먼저 진행하세요.

---

## AKS 생성 대기 중 — CLI 사전 설치

클러스터 없이도 VM에 CLI 도구를 미리 설치할 수 있습니다.

**helm 설치 확인**

```bash title="터미널"
helm version
```

없으면 설치:

```bash title="터미널"
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**kubectl 설치 확인**

```bash title="터미널"
kubectl version --client
```

**argocd CLI 설치**

```bash title="터미널"
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
argocd version --client
```

**kubectl 자동완성 설정**

```bash title="터미널"
apt-get install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

탭 키로 명령어와 리소스 이름이 자동완성됩니다.

AKS 생성이 완료되면 아래 Step 2로 넘어갑니다.

---

## Step 4-1. nginx Ingress Controller 설치

!!! note "Ingress와 Ingress Controller의 차이"
    - `ingress.yaml` — "이 URL은 이 서비스로 보내줘" 라는 **규칙**
    - Ingress Controller — 그 규칙을 실제로 **실행하는 주체** (nginx)

    규칙만 있고 실행자가 없으면 `kubectl get ingress` 의 ADDRESS가 영원히 비어있습니다.
    AKS는 기본적으로 Ingress Controller가 없으므로 직접 설치해야 합니다.

```bash title="터미널"
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

외부 IP가 할당될 때까지 대기합니다:

```bash title="터미널"
kubectl get svc -n ingress-nginx -w
```

```text title="출력 예시"
NAME                       TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)
ingress-nginx-controller   LoadBalancer   10.0.xx.xx   20.xxx.xxx.xxx   80:...
```

`EXTERNAL-IP`에 IP가 표시되면 `Ctrl+C` 로 종료합니다.

!!! success "✅ 확인 포인트"
    ```bash
    kubectl get ingressclasses
    ```
    `nginx` 클래스가 보이면 OK.

!!! success "✅ 확인 포인트"
    ```bash
    az aks show --resource-group $RG --name $AKS_NAME --query provisioningState -o tsv
    ```
    `Succeeded` 가 출력되면 OK.

---

## Step 2. kubeconfig 설정

VM에서 AKS 클러스터에 접근하기 위한 kubeconfig를 받습니다.

```bash title="터미널"
az aks get-credentials \
  --resource-group $RG \
  --name $AKS_NAME
```

연결 확인:

```bash title="터미널"
kubectl get nodes
```

```text title="출력 예시"
NAME                                STATUS   ROLES   AGE
aks-nodepool1-12345678-vmss000000   Ready    agent   10m
aks-nodepool1-12345678-vmss000001   Ready    agent   10m
```

!!! success "✅ 확인 포인트"
    노드가 `Ready` 상태이면 연결 성공입니다.

---

## Step 3. 본인 네임스페이스 생성

```bash title="터미널"
kubectl create namespace hanbat-${GITHUB_USER}
kubectl get namespace hanbat-${GITHUB_USER}
```

```text title="출력 예시"
NAME              STATUS   AGE
hanbat-parkdaeri  Active   5s
```

---

## Step 4. ArgoCD 설치

```bash title="터미널"
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer
```

Pod가 모두 뜰 때까지 기다립니다:

```bash title="터미널"
kubectl get pods -n argocd -w
```

모든 Pod가 `Running` 상태가 되면 `Ctrl+C` 로 종료합니다.

!!! success "✅ 확인 포인트"
    ```bash
    kubectl get svc argocd-server -n argocd
    ```
    `EXTERNAL-IP` 에 IP 주소가 표시되면 OK.

---

## Step 5. ACR 연결 (이미지 pull 권한)

AKS가 본인 ACR에서 이미지를 pull할 수 있도록 연결합니다.

```bash title="터미널"
az aks update \
  --resource-group $RG \
  --name $AKS_NAME \
  --attach-acr $ACR_NAME
```

!!! success "✅ 확인 포인트"
    ```bash
    az aks check-acr --resource-group $RG --name $AKS_NAME --acr ${ACR_NAME}.azurecr.io
    ```
    `Your cluster can pull images from ...` 가 출력되면 OK.

---

## Step 6. ArgoCD UI 접속

```bash title="터미널"
kubectl -n argocd get svc argocd-server
```

```text title="출력 예시"
NAME            TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)
argocd-server   LoadBalancer   10.0.xx.xx   20.xxx.xxx.xxx   443:...
```

브라우저에서 `https://<EXTERNAL-IP>` 접속합니다.

!!! warning "인증서 경고"
    ArgoCD 기본 설치는 자체 서명 인증서를 사용합니다.
    브라우저에서 "고급 → 안전하지 않음으로 진행" 을 선택하면 접속할 수 있습니다.

초기 비밀번호 확인:

```bash title="터미널"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

| 항목 | 값 |
| --- | --- |
| Username | `admin` |
| Password | 위에서 확인한 값 |

로그인 후 **빈 Applications 화면**이 나오면 정상입니다.

!!! success "✅ 확인 포인트"
    - `kubectl get nodes` 에서 노드가 Ready 상태
    - ArgoCD UI에 로그인 성공
    - 본인 네임스페이스(`hanbat-<사용자명>`)가 생성됨

---

## 다음 단계

[:material-arrow-left: Phase 4 개요](index.md){ .md-button }
[Phase 4-2 · ArgoCD Application 등록 :material-arrow-right:](application-register.md){ .md-button .md-button--primary }
