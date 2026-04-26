# Phase 3-2 · GitHub Actions 기초

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"Actions 써본 적 없으면 Hello World부터 해. 직접 workflow 파일 하나 만들어서 실행해보면 구조가 머릿속에 잡혀."*

---

## 사전 준비 — 레포 Fork

Phase 1~2에서는 읽기 전용으로 clone만 했습니다.
GitHub Actions Workflow를 추가하려면 **내 계정의 레포**가 필요합니다. Fork 합니다.

1. 브라우저에서 `https://github.com/skilleat-labs/hanbat-order-app-s2` 접속
2. 우측 상단 **Fork** → **Create fork**
3. Owner를 **본인 GitHub 계정**으로 확인 후 **Create fork**

Fork가 완료되면 `https://github.com/<본인_사용자명>/hanbat-order-app-s2` 가 생깁니다.

이제 VM에서 remote를 내 Fork로 변경합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
git remote set-url origin https://github.com/<본인_사용자명>/hanbat-order-app-s2.git
git remote -v   # 확인
```

```text title="출력 예시"
origin  https://github.com/parkdaeri/hanbat-order-app-s2.git (fetch)
origin  https://github.com/parkdaeri/hanbat-order-app-s2.git (push)
```

!!! note "GitHub push 인증"
    HTTPS로 push할 때 비밀번호 대신 **Personal Access Token(PAT)**을 써야 합니다.

    **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
    → Generate new token (classic) → Note에 이름 입력

    권한 체크 (반드시 선택):

    - ☑ **`repo`** — 레포 읽기/쓰기 전체 (push에 필수)
    - ☑ **`workflow`** — `.github/workflows/` 파일 push에 필수

    나머지는 체크하지 않아도 됩니다. → Generate token → 토큰 복사 (창 닫으면 다시 못 봄)

    push 시 username은 GitHub 사용자명, password에 토큰을 붙여넣습니다.

    매번 입력이 번거로우면:
    ```bash title="터미널"
    git config --global credential.helper store
    ```
    첫 push 때 한 번만 입력하면 저장됩니다.

!!! success "✅ 확인 포인트"
    `git remote -v` 결과가 본인 GitHub 사용자명으로 나오면 OK.

---

## 사전 준비 — main 브랜치로 전환

Phase 2에서 `phase2` 브랜치로 작업했습니다. Phase 3부터는 `main`에서 진행합니다.

```bash title="터미널"
git checkout main
git branch   # * main 이 보여야 함
```

```text title="출력 예시"
* main
  phase2
```

!!! warning "브랜치 확인 필수"
    `phase2` 상태에서 커밋하면 Actions가 트리거되지 않거나 엉뚱한 브랜치에 쌓입니다.
    반드시 `* main` 인지 확인 후 진행하세요.

---

## Step 1. 첫 번째 Workflow — Hello World

`.github/workflows/` 디렉터리에 Workflow 파일을 만듭니다.

```bash title="터미널"
mkdir -p ~/hanbat-order-app-s2/.github/workflows
```

```yaml title=".github/workflows/hello.yml"
name: Hello World

on:
  push:
    branches: [main]
  workflow_dispatch:      # 수동 실행 버튼 추가

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: "코드 체크아웃"
        uses: actions/checkout@v4

      - name: "인사하기"
        run: |
          echo "안녕하세요, ${{ github.actor }}님! 커밋: ${{ github.sha }}"

      - name: "파일 목록 출력"
        run: ls -la
```

push합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
git add .github/workflows/hello.yml
git commit -m "ci: add hello world workflow"
git push origin main
```

GitHub 레포 → **Actions** 탭에서 실행 결과를 확인합니다.

!!! success "✅ 확인 포인트"
    - Actions 탭에 Workflow 실행이 나타났다 (초록 체크 또는 실행 중)
    - `인사하기` Step 로그에서 본인 GitHub 사용자명과 커밋 SHA가 출력됐다

!!! note "Node.js 20 deprecation warning"
    `Node.js 20 actions are deprecated` 경고가 나와도 무시하세요.
    `actions/checkout@v4`의 런타임 버전 안내일 뿐, 동작에 문제없습니다.

---

## Step 2. 트리거 타입 이해

`on:` 아래 여러 트리거를 지정할 수 있습니다.

```yaml title="트리거 예시"
on:
  push:
    branches: [main]          # main 브랜치에 push될 때
    paths:
      - 'order-api/**'        # 이 경로 파일이 변경될 때만

  pull_request:
    branches: [main]          # main으로 PR이 열릴 때

  workflow_dispatch:          # GitHub UI에서 수동 실행
    inputs:
      environment:
        description: '배포 환경'
        required: true
        default: 'staging'
```

!!! note "`paths` 필터를 쓰는 이유"
    `k8s/` 폴더만 변경될 때(ArgoCD 자동 커밋)는 빌드가 다시 돌면 안 됩니다.
    `paths: - 'order-api/**'` 를 지정하면 앱 코드가 바뀔 때만 빌드합니다.

---

## Step 3. Secrets 다루기

API 키, 토큰 같은 민감 정보는 코드에 직접 넣지 않고 **Secrets**에 저장합니다.

!!! note "이 Secrets는 GitHub Actions 전용입니다"
    여기서 말하는 Secrets는 **Workflow 파일 안에서 `${{ secrets.이름 }}`으로 꺼내 쓰는 값**입니다.
    코드(.py, .env)에 직접 쓰는 것이 아니라, CI 파이프라인이 실행될 때만 환경변수로 주입됩니다.
    Phase 3-3에서 Azure 인증 정보를 이 방식으로 저장합니다.

### Secret 생성

1. GitHub 레포 상단 **Settings** 탭 클릭
2. 왼쪽 메뉴 **Secrets and variables → Actions**
3. **New repository secret** 클릭
4. 입력:
    - **Name**: `MY_SECRET`
    - **Secret**: `hello-secret`
5. **Add secret** 클릭

목록에 `MY_SECRET`이 나타나면 생성 완료입니다.

!!! warning "Secret은 저장 후 값을 다시 볼 수 없습니다"
    생성 후에는 값을 확인할 수 없고 덮어쓰기만 가능합니다.
    잃어버리면 새로 만들어야 합니다.

### Workflow에서 참조

`hello.yml`에 Step을 추가합니다.

```yaml title=".github/workflows/hello.yml (Step 추가)"
      - name: "Secret 참조"
        run: |
          echo "Secret 앞 3자리: ${{ secrets.MY_SECRET }}"
```

```bash title="터미널"
git add .github/workflows/hello.yml
git commit -m "ci: test secrets"
git push origin main
```

```text title="Actions 로그 출력 예시"
Secret 앞 3자리: ***
```

!!! warning "Secret은 로그에서 자동으로 마스킹됩니다"
    `***`으로 표시되는 건 정상입니다. 값이 로그에 노출되지 않습니다.
    이게 Secrets를 쓰는 이유입니다.

!!! success "✅ 확인 포인트"
    - GitHub Settings에서 `MY_SECRET`이 목록에 보인다
    - Actions 로그에서 Secret 값이 `***`으로 마스킹된 것을 확인했다

---

## Step 4. 환경변수와 출력값(outputs)

Step 간에 값을 전달하려면 `$GITHUB_OUTPUT`을 사용합니다.
Phase 3-3에서 이미지 태그를 전달할 때 이 패턴을 씁니다.

```yaml title="outputs 예시"
      - name: "태그 생성"
        id: tag
        run: |
          echo "TAG=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: "태그 사용"
        run: |
          echo "이미지 태그는 ${{ steps.tag.outputs.TAG }} 입니다"
```

`${GITHUB_SHA::7}` 은 커밋 SHA의 앞 7자리입니다.
예: `abc1234ef...` → `abc1234`

이 태그가 곧 **이미지 버전**이 됩니다.

---

## 정리

| 개념 | 핵심 |
| --- | --- |
| Workflow 트리거 | `on: push:` / `pull_request:` / `workflow_dispatch:` |
| Step 실행 | `uses:` (외부 Action) 또는 `run:` (쉘 명령) |
| 값 참조 | `${{ github.sha }}`, `${{ secrets.XXX }}`, `${{ steps.id.outputs.KEY }}` |
| 무한 루프 방지 | 자동 커밋 메시지에 `[skip ci]` 추가 |

!!! success "✅ 확인 포인트"
    - Fork 후 `git remote -v`가 본인 레포를 가리키는 것을 확인했다
    - Hello World Workflow가 Actions 탭에서 성공으로 완료됐다
    - Secret이 `***`으로 마스킹된 것을 확인했다

---

## 다음 단계

[:material-arrow-left: Phase 3-1 · CI/CD 개념](cicd-concept.md){ .md-button }
[Phase 3-3 · ACR 빌드 및 이미지 푸시 :material-arrow-right:](acr-push.md){ .md-button .md-button--primary }
