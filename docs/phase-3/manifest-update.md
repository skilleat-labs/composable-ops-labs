# Phase 3-4 · 매니페스트 태그 자동 업데이트

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"ACR에 이미지 올라가는 건 됐어. 근데 ArgoCD는 Git을 보잖아. Git의 매니페스트에 이미지 태그가 안 바뀌면 ArgoCD가 새 이미지를 배포 안 해. 파이프라인 마지막 퍼즐이야."*

---

## 왜 매니페스트를 자동으로 수정해야 하나

현재 `k8s/order-api-deployment.yaml` 파일을 보면:

```yaml title="k8s/order-api-deployment.yaml (현재)"
containers:
  - name: order-api
    image: <ACR_NAME>.azurecr.io/order-api:PLACEHOLDER
```

`PLACEHOLDER`가 실제 커밋 SHA로 바뀌어야 ArgoCD가 "아, 새 버전 배포해야겠다"고 인식합니다.
Actions가 이미지를 빌드한 직후, 이 태그를 자동으로 교체하고 Git에 커밋합니다.

---

## Step 1. 네임스페이스 placeholder 교체

`k8s/` 디렉터리의 모든 파일에서 `<본인_GitHub_사용자명>` placeholder를 실제 값으로 바꿉니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
GITHUB_USER="<본인_GitHub_사용자명>"   # 실제 GitHub 사용자명으로 교체

# 모든 k8s 파일의 namespace placeholder 교체
sed -i "s/hanbat-<본인_GitHub_사용자명>/hanbat-${GITHUB_USER}/g" k8s/*.yaml

# 확인
grep "namespace:" k8s/order-api-deployment.yaml
```

```text title="출력 예시"
  namespace: hanbat-parkdaeri
```

```bash title="터미널"
git add k8s/
git commit -m "chore: set namespace to hanbat-${GITHUB_USER}"
git push origin main
```

---

## Step 2. ci.yml에 매니페스트 업데이트 Step 추가

Phase 3-3에서 만든 `ci.yml`에 두 개의 Step을 추가합니다.
`order-web 빌드 및 푸시` Step **아래에** 붙여넣습니다.

```yaml title=".github/workflows/ci.yml (추가할 Step)"
      - name: k8s 매니페스트 이미지 태그 업데이트
        run: |
          TAG=${{ steps.tag.outputs.TAG }}
          ACR=${{ secrets.ACR_NAME }}

          sed -i "s|image: .*order-api:.*|image: ${ACR}.azurecr.io/order-api:${TAG}|" \
            k8s/order-api-deployment.yaml

          sed -i "s|image: .*payment-api:.*|image: ${ACR}.azurecr.io/payment-api:${TAG}|" \
            k8s/payment-api-deployment.yaml

          sed -i "s|image: .*order-web:.*|image: ${ACR}.azurecr.io/order-web:${TAG}|" \
            k8s/order-web-deployment.yaml

          echo "=== 업데이트된 태그 확인 ==="
          grep "image:" k8s/order-api-deployment.yaml
          grep "image:" k8s/payment-api-deployment.yaml
          grep "image:" k8s/order-web-deployment.yaml

      - name: 매니페스트 변경사항 자동 커밋
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "ci: update image tags to ${{ steps.tag.outputs.TAG }} [skip ci]"
          file_pattern: k8s/*.yaml
```

!!! note "[skip ci] 가 왜 필요한가요?"
    `git-auto-commit-action`이 `k8s/*.yaml`을 수정해서 main에 push합니다.
    이 push가 또 다른 CI Workflow를 trigger하면 무한 루프가 됩니다.

    `[skip ci]` 플래그가 커밋 메시지에 있으면 GitHub Actions가 해당 push는 무시합니다.
    또는 Workflow의 `on.push.paths`에서 `k8s/**`를 제외해도 됩니다.

완성된 `ci.yml` 전체 모습입니다. 아래 내용으로 **전체 교체**한 뒤 push합니다.

```bash title="터미널"
git add .github/workflows/ci.yml
git commit -m "ci: add manifest update step"
git push origin main
```

```yaml title=".github/workflows/ci.yml (완성본)"
name: CI

on:
  push:
    branches: [main]
    paths:
      - 'order-api/**'
      - 'payment-api/**'
      - 'order-web/**'
  workflow_dispatch:

permissions:
  id-token: write
  contents: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: Azure 로그인 (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: 이미지 태그 설정
        id: tag
        run: echo "TAG=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: order-api 빌드 및 푸시
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image order-api:${{ steps.tag.outputs.TAG }} \
            --image order-api:latest \
            order-api/

      - name: payment-api 빌드 및 푸시
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image payment-api:${{ steps.tag.outputs.TAG }} \
            --image payment-api:latest \
            payment-api/

      - name: order-web 빌드 및 푸시
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image order-web:${{ steps.tag.outputs.TAG }} \
            --image order-web:latest \
            order-web/

      - name: k8s 매니페스트 이미지 태그 업데이트
        run: |
          TAG=${{ steps.tag.outputs.TAG }}
          ACR=${{ secrets.ACR_NAME }}

          sed -i "s|image: .*order-api:.*|image: ${ACR}.azurecr.io/order-api:${TAG}|" \
            k8s/order-api-deployment.yaml
          sed -i "s|image: .*payment-api:.*|image: ${ACR}.azurecr.io/payment-api:${TAG}|" \
            k8s/payment-api-deployment.yaml
          sed -i "s|image: .*order-web:.*|image: ${ACR}.azurecr.io/order-web:${TAG}|" \
            k8s/order-web-deployment.yaml

          grep "image:" k8s/order-api-deployment.yaml

      - name: 매니페스트 변경사항 자동 커밋
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "ci: update image tags to ${{ steps.tag.outputs.TAG }} [skip ci]"
          file_pattern: k8s/*.yaml
```

---

## Step 3. end-to-end 테스트

파이프라인 전체를 테스트합니다. `order-api/app.py` 에 주석 한 줄을 추가해 push합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
echo "# phase3 test" >> order-api/app.py
git add order-api/app.py
git commit -m "test: trigger CI pipeline"
git push origin main
```

GitHub → **Actions 탭**에서 실행을 지켜봅니다.

```text title="예상 실행 순서"
✅ 코드 체크아웃
✅ Azure 로그인 (OIDC)
✅ 이미지 태그 설정
✅ order-api 빌드 및 푸시        ← ACR에 abc1234 태그로 업로드
✅ payment-api 빌드 및 푸시
✅ order-web 빌드 및 푸시
✅ k8s 매니페스트 이미지 태그 업데이트
✅ 매니페스트 변경사항 자동 커밋   ← [skip ci] 커밋으로 k8s/*.yaml 업데이트
```

완료 후 GitHub 레포에서 `k8s/order-api-deployment.yaml` 파일을 확인합니다.

```yaml title="k8s/order-api-deployment.yaml (자동 업데이트 후)"
containers:
  - name: order-api
    image: hanbatpd01.azurecr.io/order-api:abc1234   ← PLACEHOLDER → 실제 SHA로 교체됨
```

!!! success "✅ 확인 포인트"
    - Actions Workflow가 모든 Step 초록으로 완료됐다
    - Git 커밋 히스토리에 `ci: update image tags to xxxxxxx [skip ci]` 커밋이 생겼다
    - `k8s/order-api-deployment.yaml` 의 `image:` 값이 실제 ACR 주소와 SHA 태그로 바뀌었다
    - `[skip ci]` 커밋이 새 Workflow를 trigger하지 않은 것을 확인했다 (Actions 탭에 추가 실행 없음)

---

## 정리

Phase 3에서 완성한 파이프라인:

```
git push (앱 코드 변경)
    │
    ▼
[GitHub Actions]
    ├─ Azure OIDC 로그인 (비밀키 없이)
    ├─ 이미지 빌드 → ACR 푸시 (커밋 SHA 태그)
    ├─ k8s/*.yaml 이미지 태그 sed로 교체
    └─ 자동 커밋 [skip ci] → main push
                │
                ▼ (Phase 4에서 연결)
           [ArgoCD가 감지 → K8s 자동 배포]
```

!!! quote "이대리"
    *"됐어. 이제 코드만 push하면 나머지는 기계가 해. 다음은 ArgoCD가 이 매니페스트를 보고 AKS에 자동 배포하는 거야."*

!!! success "✅ Phase 3 완료 체크리스트"
    - [ ] 레포 Fork 완료
    - [ ] OIDC Federated Credential 설정 완료
    - [ ] GitHub Secrets 4개 등록 완료
    - [ ] push → 이미지 빌드 → ACR 푸시 자동화 완료
    - [ ] k8s 매니페스트 이미지 태그 자동 업데이트 완료

---

## 다음 단계

[:material-arrow-left: Phase 3-3 · ACR 빌드 및 이미지 푸시](acr-push.md){ .md-button }
[Phase 4 · 사람 손 떼기 (2부) :material-arrow-right:](../phase-4/index.md){ .md-button .md-button--primary }
