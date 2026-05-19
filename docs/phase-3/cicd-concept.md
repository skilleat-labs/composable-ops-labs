# Phase 3-1 · CI/CD 개념과 파이프라인 흐름

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"박대리, 지난주에 결제 API 코드 수정 5번 있었는데 그때마다 내가 `docker build`, `docker tag`, `az acr login`, `az containerapp update` 다 수동으로 돌렸어. 40분씩 걸려. 이거 자동화 안 하면 진짜 퇴사한다..."*

!!! quote "박대리"
    *"...GitHub Actions로 하면 되는 거죠?"*

!!! quote "이대리"
    *"맞아. 이번 3시간은 그거 만드는 시간이야. 근데 그 전에 Git부터 잠깐 짚고 가자."*

---

## Git — 코드의 저장 이력

CI/CD를 이해하려면 먼저 **Git**이 뭔지 알아야 합니다.

Git은 **코드의 변경 이력을 저장하는 도구**입니다. 게임의 세이브 포인트처럼, 코드를 원하는 시점으로 되돌릴 수 있습니다.

```text
예를 들어:

월요일  → 로그인 기능 완성 → 저장 (commit)
화요일  → 결제 기능 추가   → 저장 (commit)
수요일  → 버그 발생... 화요일 상태로 되돌리기 → 가능!
```

### 자주 나오는 Git 용어

| 용어 | 뜻 | 비유 |
|------|------|------|
| **Repository (레포)** | 코드와 이력이 저장된 프로젝트 폴더 | 파일 서랍 전체 |
| **Commit** | 변경사항을 이력에 저장하는 행위 | 게임 세이브 |
| **Branch** | 독립적으로 작업할 수 있는 코드 흐름 | 원본을 건드리지 않는 작업 복사본 |
| **Push** | 내 PC의 변경사항을 GitHub에 업로드 | USB에 파일 복사 → 클라우드로 올리기 |
| **Pull** | GitHub의 변경사항을 내 PC로 내려받기 | 클라우드에서 파일 내려받기 |

### Branch가 왜 있나요?

```text
main 브랜치  ─────────────────────────────── (배포 중인 안정 버전)
                 │
                 └── feature/login ─────────── (개발 중인 기능)
```

`main` 브랜치는 실제 서비스에 배포된 코드입니다. 새 기능을 개발할 때는 `main`을 직접 건드리지 않고, 별도 브랜치를 만들어서 작업합니다. 완성되면 `main`에 합칩니다.

이번 실습에서는 **`main` 브랜치에 push하는 순간 자동으로 배포가 시작**됩니다.

### GitHub이 뭔가요?

GitHub은 Git 레포지토리를 **인터넷에 저장하고 공유하는 서비스**입니다. 내 PC에서 작업한 코드를 GitHub에 push하면 팀원 모두가 볼 수 있고, 여기서 CI/CD 파이프라인도 돌아갑니다.

```text
내 PC (코드 작성)
    │
    │  git push
    ▼
GitHub (코드 저장소 + 파이프라인 실행)
    │
    │  자동으로 빌드·배포
    ▼
AKS 클러스터 (서비스 운영)
```

---

## 배포를 수동으로 하면 어떻게 되나

코드 한 줄 수정할 때마다 이대리가 직접 해야 했던 작업:

```text
1. 코드 수정
2. docker build -t order-api:v1.2.3 .       ← 이미지 빌드
3. docker tag order-api:v1.2.3 <acr>.azurecr.io/order-api:v1.2.3
4. az acr login --name <acr>                 ← 클라우드 로그인
5. docker push <acr>.azurecr.io/order-api:v1.2.3   ← 이미지 업로드
6. kubectl set image deployment/order-api ...       ← 배포
```

5번 배포 × 40분 = **200분(3시간 20분)**을 배포에만 씁니다.

게다가 실수도 납니다.

```text
자주 발생하는 실수:
  ✗ 태그 번호를 잘못 입력해서 이전 버전이 배포됨
  ✗ ACR 로그인을 깜빡하고 push 실패
  ✗ 3개 서비스 중 1개를 배포 누락
  ✗ 팀원이 배포한 건지 내가 배포한 건지 불분명
```

**CI/CD는 이 반복 작업을 코드로 자동화한 것입니다.**
`git push` 한 번이면 나머지는 기계가 합니다.

---

## CI와 CD — 무엇이 다른가

| 구분 | 이름 | 하는 일 |
| --- | --- | --- |
| **CI** | Continuous Integration (지속적 통합) | 코드를 합치고 → 빌드하고 → 테스트까지 자동 |
| **CD** | Continuous Delivery/Deploy (지속적 배포) | 검증된 결과물을 → 실제 환경에 자동 배포 |

이름이 어렵게 느껴지지만 실제로는 단순합니다.

```text
[CI가 하는 일]
  코드가 push되면:
    → "이 코드가 제대로 빌드가 되나?" 확인
    → "테스트는 통과하나?" 확인
    → "이미지로 만들어서 저장소에 올리기"

[CD가 하는 일]
  CI가 만든 이미지가 준비되면:
    → "실제 서버에 새 버전 배포하기"
    → "문제 생기면 이전 버전으로 롤백"
```

전체 흐름:

```text
개발자 → git push
             │
             ▼
         [CI — GitHub Actions]
             │  코드 빌드
             │  테스트 실행
             │  이미지 생성 → ACR 업로드
             │  k8s 매니페스트 태그 업데이트
             ▼
         [CD — ArgoCD]
             │  Git 변경 감지 → Sync
             ▼
         실서비스 반영
```

!!! note "CI와 CD를 왜 분리하나요?"
    GitHub Actions가 배포까지 직접 하면 편할 것 같지만, 실무에서는 분리합니다.

    - **CI(Actions)**: "무엇을 배포할지" 결정 (이미지 빌드, 태그)
    - **CD(ArgoCD)**: "어떻게 배포할지" 결정 (롤링 업데이트, 롤백, Sync 상태 시각화)

    ArgoCD는 Git을 **Single Source of Truth(유일한 진실의 원천)**로 삼습니다.
    누군가 실수로 `kubectl`로 직접 수정해도 **자동으로 Git 상태로 되돌립니다.**
    이게 **GitOps**의 핵심입니다.

---

## GitOps — Git이 배포의 기준

GitOps는 "Git에 있는 것 = 실제 서버에 있는 것"을 보장하는 방식입니다.

```text
일반적인 배포:
  개발자가 kubectl로 직접 수정 → 서버 상태가 바뀜
  → Git과 서버가 달라짐 → 나중에 뭐가 배포된 건지 모름

GitOps 배포:
  Git을 수정 → ArgoCD가 감지 → 서버를 Git에 맞춤
  → Git이 항상 정답 → 이력 추적 가능
```

"누가 언제 무엇을 배포했나?"를 Git 커밋 이력으로 항상 확인할 수 있습니다.

---

## 우리가 만들 파이프라인 — 패턴 A

이번 Phase 3~4에서 완성할 파이프라인 전체 그림입니다.

```text
[개발자 PC]
    │  git push (코드 수정)
    ▼
[GitHub — hanbat-order-app-s2 레포]
    │
    ├─ [GitHub Actions CI 자동 실행]
    │      1. 이미지 빌드 (order-api, payment-api, order-web)
    │      2. ACR에 푸시 (태그: abc1234 ← 커밋 ID 앞 7자리)
    │      3. k8s/*.yaml 이미지 태그 수정 (PLACEHOLDER → abc1234)
    │      4. main 브랜치에 자동 커밋 [skip ci]
    │                    │
    │                    ▼ (Git 변경 감지)
    └─ [ArgoCD CD]
           │  k8s/*.yaml 변경 확인 → AKS에 Sync
           ▼
       [AKS 클러스터]
           └─ order-api, payment-api, order-web 새 버전 배포
```

!!! note "패턴 A의 특징"
    - **단일 레포**: 앱 코드와 K8s 매니페스트가 같은 저장소에 있습니다
    - **자동 커밋**: Actions가 매니페스트를 수정하고 Git에 직접 push합니다
    - **[skip ci] 플래그**: 자동 커밋이 또 다른 빌드를 trigger하지 않도록 막습니다

    실무에서는 앱 레포와 매니페스트 레포를 분리하지만, 15시간 실습에서는 단일 레포가 훨씬 빠르게 end-to-end를 경험할 수 있습니다.

---

## GitHub Actions의 구성 요소

GitHub Actions는 GitHub에서 제공하는 CI 도구입니다. `.github/workflows/` 폴더에 YAML 파일을 넣으면 됩니다.

```yaml title=".github/workflows/ci.yml — 구조 설명"
name: CI Pipeline          # Workflow 이름 (GitHub Actions 탭에 표시됨)

on:                        # 언제 실행할지 (트리거)
  push:
    branches: [main]       # main 브랜치에 push되면 실행

jobs:                      # 실행할 작업 묶음
  build:                   # Job 이름 (자유롭게 지정)
    runs-on: ubuntu-latest # 어떤 환경에서 실행할지 (GitHub이 제공하는 가상 머신)

    steps:                 # Job 안의 순차적인 단계들
      - name: 코드 체크아웃          # Step 이름
        uses: actions/checkout@v4   # 미리 만들어진 Action 사용

      - name: 이미지 빌드
        run: docker build -t myapp .  # 직접 쉘 명령 실행
```

| 개념 | 설명 |
| --- | --- |
| **Workflow** | `.github/workflows/*.yml` 파일 하나 = Workflow 하나 |
| **Trigger** | `on:` 아래 정의. push, PR, 수동(`workflow_dispatch`) 등 |
| **Job** | 독립적으로 실행되는 작업 단위. 기본적으로 병렬 실행 |
| **Step** | Job 내 순차 실행 단계. `uses`(외부 Action) 또는 `run`(쉘 명령) |
| **Runner** | Step을 실행하는 가상 머신 (`ubuntu-latest`, `windows-latest` 등) |

!!! note "Runner는 뭔가요?"
    GitHub이 제공하는 **임시 가상 머신**입니다. Workflow가 실행될 때마다 새로 만들어지고, 끝나면 삭제됩니다.

    ```text
    git push 발생
        → GitHub이 ubuntu 가상 머신 1대 생성
        → 그 안에서 Step들 순서대로 실행
            (코드 다운로드 → 빌드 → 이미지 푸시 → ...)
        → 완료되면 가상 머신 폐기
    ```

    내 PC나 서버를 쓰는 게 아니라 GitHub이 컴퓨터를 빌려주는 겁니다.

---

## 이번 Phase에서 만들 것

| 단계 | 파일 | 결과 |
| --- | --- | --- |
| Phase 3-2 | `.github/workflows/hello.yml` | Actions 기초 익히기 — "Hello World" 출력 |
| Phase 3-3 | `.github/workflows/ci.yml` (1부) | ACR 빌드·푸시 자동화 |
| Phase 3-4 | `.github/workflows/ci.yml` (완성) | 매니페스트 태그까지 자동 업데이트 |

!!! quote "이대리"
    *"개념은 여기까지야. 직접 만들면서 배우는 게 훨씬 빠르니까 바로 해보자."*

---

## 다음 단계

[:material-arrow-left: Phase 3 개요](index.md){ .md-button }
[Phase 3-2 · GitHub Actions 기초 :material-arrow-right:](actions-basics.md){ .md-button .md-button--primary }
