# Phase 3-3 · ACR 빌드 및 이미지 푸시 자동화

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"이제 진짜 핵심이야. 코드 push하면 이미지가 자동으로 ACR에 올라가야 해. 근데 Actions가 Azure에 로그인하려면 인증이 필요한데 — 비밀키 방식은 절대 쓰지 마."*

!!! quote "박대리"
    *"왜요?"*

!!! quote "이대리"
    *"비밀키는 GitHub Secret에 저장되는 순간 유출 경로가 생겨. OIDC는 키 자체가 없어. GitHub이 Azure한테 '나 이 레포 맞아'라고 증명서를 보내는 방식이야."*

---

## 개념 — OIDC 인증이란

```
[GitHub Actions Runner]
        │  "나는 parkdaeri/hanbat-order-app-s2 의 main 브랜치야"
        │  (서명된 토큰 발급)
        ▼
[Azure AD]
        │  GitHub의 공개키로 토큰 검증
        │  "맞네. 이 레포에게 Contributor 권한 줘도 돼"
        ▼
[ACR 접근 허용]
```

비밀키(`client_secret`)가 어디에도 저장되지 않습니다.
누군가 GitHub Secret을 훔쳐도 Azure에 로그인할 수 없습니다.

---

## Step 1. ACR 생성

VM에서 Azure CLI로 ACR을 만듭니다.
ACR 이름은 전 세계에서 유일해야 합니다. `hanbat` + 본인 이니셜 + 숫자 조합을 권장합니다.

```bash title="터미널"
# 예시: hanbatpd01  (parkdaeri 이니셜)
ACR_NAME="hanbat<이니셜><숫자>"   # 예: hanbatpd01

az acr create \
  --name $ACR_NAME \
  --resource-group hanbat-s2-lab-rg \
  --sku Basic
```

```text title="출력 예시"
{
  "loginServer": "hanbatpd01.azurecr.io",
  ...
}
```

`loginServer` 값을 메모해둡니다. (`<이름>.azurecr.io` 형식)

!!! success "✅ 확인 포인트"
    ```bash
    az acr show --name $ACR_NAME --query loginServer -o tsv
    ```
    ACR 주소가 출력되면 OK.

---

## Step 2. OIDC Federated Credential 설정

GitHub Actions가 Azure에 비밀키 없이 인증하기 위한 설정입니다.
아래 명령을 **순서대로** 실행합니다.

```bash title="터미널 — App Registration 생성"
APP_ID=$(az ad app create \
  --display-name "hanbat-github-actions" \
  --query appId -o tsv)

echo "APP_ID: $APP_ID"
```

```bash title="터미널 — Service Principal 생성"
SP_OBJ_ID=$(az ad sp create --id $APP_ID --query id -o tsv)

echo "SP_OBJ_ID: $SP_OBJ_ID"
```

```bash title="터미널 — ACR Contributor 권한 부여"
ACR_ID=$(az acr show \
  --name $ACR_NAME \
  --resource-group hanbat-s2-lab-rg \
  --query id -o tsv)

az role assignment create \
  --assignee $SP_OBJ_ID \
  --role Contributor \
  --scope $ACR_ID
```

```bash title="터미널 — 구독 Reader 권한 부여"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az role assignment create \
  --assignee $SP_OBJ_ID \
  --role Reader \
  --scope /subscriptions/$SUBSCRIPTION_ID
```

!!! warning "AcrPush만으론 부족합니다"
    `az acr build`는 빌드 큐에 소스를 업로드할 때 `listBuildSourceUploadUrl` 액션이 필요합니다.
    `AcrPush` 역할에는 이 권한이 없어서 빌드가 실패합니다.
    ACR 리소스에는 **Contributor**, 구독 레벨에는 **Reader** 권한이 필요합니다.

```bash title="터미널 — Federated Credential 등록"
# <본인_GitHub_사용자명> 을 실제 GitHub 사용자명으로 교체하세요
GITHUB_USER="<본인_GitHub_사용자명>"

az ad app federated-credential create \
  --id $APP_ID \
  --parameters "{
    \"name\": \"github-main\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${GITHUB_USER}/hanbat-order-app-s2:ref:refs/heads/main\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
```

!!! warning "subject 값이 정확해야 합니다"
    `subject`의 레포 이름과 브랜치가 **실제 Fork한 레포**와 일치해야 합니다.
    대소문자도 구분합니다.

---

## Step 3. GitHub Secrets 등록

GitHub Actions Workflow에서 참조할 값을 Secrets에 저장합니다.

```bash title="터미널 — 값 확인"
echo "AZURE_CLIENT_ID:       $APP_ID"
echo "AZURE_TENANT_ID:       $(az account show --query tenantId -o tsv)"
echo "AZURE_SUBSCRIPTION_ID: $(az account show --query id -o tsv)"
echo "ACR_NAME:              $ACR_NAME"
```

출력된 값 4개를 GitHub에 등록합니다.

**GitHub 레포 → Settings → Secrets and variables → Actions → New repository secret**

| Secret 이름 | 값 |
| --- | --- |
| `AZURE_CLIENT_ID` | 위에서 출력된 APP_ID |
| `AZURE_TENANT_ID` | 위에서 출력된 Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | 위에서 출력된 Subscription ID |
| `ACR_NAME` | ACR 이름 (예: `hanbatpd01`) |

!!! success "✅ 확인 포인트"
    Settings → Secrets and variables → Actions 에서 4개 Secret이 모두 보이면 OK.

---

## Step 4. CI Workflow 작성 — 빌드 + 푸시

`.github/workflows/ci.yml` 파일을 만듭니다.

```yaml title=".github/workflows/ci.yml"
name: CI

on:
  push:
    branches: [main]
    paths:
      - 'order-api/**'
      - 'payment-api/**'
      - 'order-web/**'
  workflow_dispatch:    # 수동 트리거 (Actions 탭에서 Run workflow 버튼)

permissions:
  id-token: write    # OIDC 토큰 발급 권한
  contents: write    # 매니페스트 자동 커밋 권한 (Phase 3-4에서 사용)

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: "코드 체크아웃"
        uses: actions/checkout@v4

      - name: "Azure 로그인 (OIDC)"
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: "이미지 태그 설정"
        id: tag
        run: |
          echo "TAG=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: "order-api 빌드 및 푸시"
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image order-api:${{ steps.tag.outputs.TAG }} \
            --image order-api:latest \
            order-api/

      - name: "payment-api 빌드 및 푸시"
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image payment-api:${{ steps.tag.outputs.TAG }} \
            --image payment-api:latest \
            payment-api/

      - name: "order-web 빌드 및 푸시"
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image order-web:${{ steps.tag.outputs.TAG }} \
            --image order-web:latest \
            order-web/
```

!!! note "az acr build 란?"
    `az acr build`는 Docker 데몬 없이 **Azure에서 직접** 이미지를 빌드합니다.
    Runner에 Docker를 설치할 필요가 없고, ACR 로그인도 별도로 안 해도 됩니다.
    빌드 로그는 Azure Portal → ACR → 실행 탭에서 확인할 수 있습니다.

```bash title="터미널"
git add .github/workflows/ci.yml
git commit -m "ci: add ACR build and push workflow"
git push origin main
```

---

## Step 5. 실행 결과 확인

**GitHub → Actions 탭**에서 `CI` Workflow 실행 상태를 확인합니다.

각 Step의 로그를 펼쳐보면:

```text title="이미지 태그 설정 로그"
TAG=abc1234
```

```text title="order-api 빌드 및 푸시 로그"
Queued a build with ID: cb1
...
Run ID: cb1 was successful after 1m23s
```

ACR에 이미지가 올라갔는지도 확인합니다.

```bash title="터미널"
az acr repository list --name $ACR_NAME -o table
```

```text title="출력 예시"
Result
----------
order-api
order-web
payment-api
```

```bash title="터미널"
az acr repository show-tags --name $ACR_NAME --repository order-api -o table
```

```text title="출력 예시"
Result
----------
abc1234
latest
```

!!! success "✅ 확인 포인트"
    - Actions Workflow가 성공(초록 체크)으로 완료됐다
    - ACR에 `order-api`, `payment-api`, `order-web` 이미지가 생겼다
    - 각 이미지에 커밋 SHA 7자리 태그와 `latest` 태그가 모두 있다

---

## 다음 단계

[:material-arrow-left: Phase 3-2 · GitHub Actions 기초](actions-basics.md){ .md-button }
[Phase 3-4 · 매니페스트 태그 자동 업데이트 :material-arrow-right:](manifest-update.md){ .md-button .md-button--primary }
