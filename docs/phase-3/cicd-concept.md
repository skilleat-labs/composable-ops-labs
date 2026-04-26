# Phase 3-1 · CI/CD 개념과 파이프라인 흐름

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"박대리, 지난주에 결제 API 코드 수정 5번 있었는데 그때마다 내가 `docker build`, `docker tag`, `az acr login`, `az containerapp update` 다 수동으로 돌렸어. 40분씩 걸려. 이거 자동화 안 하면 진짜 퇴사한다..."*

!!! quote "박대리"
    *"...GitHub Actions로 하면 되는 거죠?"*

!!! quote "이대리"
    *"맞아. 이번 3시간은 그거 만드는 시간이야."*

---

## CI/CD가 왜 필요한가

코드 한 줄 수정할 때마다 이대리가 직접 해야 했던 작업:

```
1. 코드 수정
2. docker build -t order-api:v1.2.3 .
3. docker tag order-api:v1.2.3 <acr>.azurecr.io/order-api:v1.2.3
4. az acr login --name <acr>
5. docker push <acr>.azurecr.io/order-api:v1.2.3
6. kubectl set image deployment/order-api ...   (또는 ArgoCD에서 수동 Sync)
```

5번 배포 × 40분 = **200분(3시간 20분)**을 배포에만 씁니다.
실수도 납니다. 태그 번호 틀리기, 이전 이미지 덮어쓰기, 배포 누락...

**CI/CD는 이 반복 작업을 코드로 자동화한 것입니다.**

---

## CI와 CD — 무엇이 다른가

| 구분 | 이름 | 하는 일 |
| --- | --- | --- |
| **CI** | Continuous Integration | 코드를 합치고 → 빌드하고 → 테스트까지 자동 |
| **CD** | Continuous Delivery/Deploy | 검증된 결과물을 → 실제 환경에 자동 배포 |

```
개발자 → git push
             │
             ▼
         [CI — GitHub Actions]
             │  코드 빌드
             │  테스트 실행
             │  이미지 생성 → ACR 업로드
             │  매니페스트 태그 업데이트
             ▼
         [CD — ArgoCD]
             │  Git 변경 감지
             │  K8s에 자동 배포
             ▼
         실서비스 반영
```

!!! note "CI와 CD를 왜 분리하나요?"
    GitHub Actions가 배포까지 직접 하면 편할 것 같지만, 실무에서는 분리합니다.

    - **CI(Actions)**: "무엇을 배포할지" 결정 (이미지 빌드, 태그)
    - **CD(ArgoCD)**: "어떻게 배포할지" 결정 (롤링 업데이트, 롤백, Sync 상태 시각화)

    ArgoCD는 Git을 Single Source of Truth로 삼기 때문에, 누군가 실수로 `kubectl` 직접 수정해도 **자동으로 원래 상태로 되돌립니다.** 이게 GitOps의 핵심입니다.

---

## 우리가 만들 파이프라인 — 패턴 A

이번 Phase 3~4에서 완성할 파이프라인 전체 그림입니다.

```
[개발자 PC]
    │  git push (코드 수정)
    ▼
[GitHub — hanbat-order-app-s2]
    │
    ├─ [GitHub Actions CI] ──────────────────────────────┐
    │      1. 이미지 빌드 (order-api, payment-api, order-web) │
    │      2. ACR에 푸시 (태그: abc1234)                    │
    │      3. k8s/*.yaml 이미지 태그 수정                    │
    │      4. main 브랜치에 자동 커밋 [skip ci]              │
    │                                                     │
    └─ [ArgoCD] ◄────────────────────────────────────────┘
           │  Git 변경 감지 → Sync
           ▼
       [AKS 클러스터]
           └─ order-api, payment-api, order-web 배포
```

!!! note "패턴 A의 특징"
    - **단일 레포**: 앱 코드와 K8s 매니페스트가 같은 저장소에 있습니다
    - **자동 커밋**: Actions가 매니페스트를 수정하고 Git에 직접 push합니다
    - **[skip ci] 플래그**: 자동 커밋이 또 다른 빌드를 trigger하지 않도록 막습니다

    실무에서는 앱 레포와 매니페스트 레포를 분리하지만, 15시간 실습에서는 단일 레포가 훨씬 빠르게 end-to-end를 경험할 수 있습니다.

---

## GitHub Actions의 구성 요소

```yaml
# .github/workflows/ci.yml

name: CI Pipeline          # Workflow 이름

on:                        # 트리거 조건
  push:
    branches: [main]

jobs:                      # 병렬 실행 단위
  build:                   # Job 이름
    runs-on: ubuntu-latest # 실행 환경 (GitHub이 제공하는 가상 머신)

    steps:                 # 순차 실행 단위
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: 이미지 빌드
        run: docker build -t myapp .
```

| 개념 | 설명 |
| --- | --- |
| **Workflow** | `.github/workflows/*.yml` 파일 하나 = Workflow 하나 |
| **Trigger** | `on:` 아래 정의. push, PR, 수동(workflow_dispatch) 등 |
| **Job** | 독립적으로 실행되는 작업 단위. 기본적으로 병렬 실행 |
| **Step** | Job 내 순차 실행 단계. `uses`(외부 Action) 또는 `run`(쉘 명령) |
| **Runner** | Step을 실행하는 가상 머신 (`ubuntu-latest`, `windows-latest` 등) |

---

## 이번 Phase에서 만들 것

| 단계 | 파일 | 결과 |
| --- | --- | --- |
| Phase 3-2 | `.github/workflows/hello.yml` | Actions 기초 익히기 |
| Phase 3-3 | `.github/workflows/ci.yml` (1부) | ACR 빌드·푸시 자동화 |
| Phase 3-4 | `.github/workflows/ci.yml` (완성) | 매니페스트 태그까지 자동 업데이트 |

---

## 다음 단계

[:material-arrow-left: Phase 3 개요](index.md){ .md-button }
[Phase 3-2 · GitHub Actions 기초 :material-arrow-right:](actions-basics.md){ .md-button .md-button--primary }
