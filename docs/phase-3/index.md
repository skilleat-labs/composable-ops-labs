# Phase 3 · 사람 손 떼기 (1부)

!!! info "예상 소요 3시간 · 난이도 ⭐⭐⭐"

---

## 스토리

점심을 먹고 들어온 이대리가 노트북 앞에서 한숨을 쉽니다.

!!! quote "이대리"
    *"박대리, 지난주에 결제 API 코드 수정 5번 있었는데 그때마다 내가 `docker build`, `docker tag`, `az acr login`, `az containerapp update` 다 수동으로 돌렸어. 40분씩 걸려. 이거 자동화 안 하면 진짜 퇴사한다..."*

!!! quote "박대리"
    *"...GitHub Actions로 하면 되는 거죠?"*

!!! quote "이대리"
    *"맞아. 이번 3시간은 그거 만드는 시간이야."*

---

## 핵심 질문

1. CI, CD, 그리고 이 둘을 합친 파이프라인은 각각 무엇이 다른가?
2. GitHub Actions의 **Workflow / Job / Step**은 어떤 관계인가?
3. Azure Container Registry에 **안전하게** 이미지를 푸시하려면 어떤 인증 방식을 써야 하나?
4. "매니페스트의 이미지 태그를 자동으로 수정"한다는 건 어떻게 구현하나?

---

## 학습 목표

- [x] CI/CD 파이프라인의 **Build → Test → Deploy** 3단계를 구분해서 설명한다
- [x] 간단한 GitHub Actions Workflow를 **직접 작성**하고 실행한다
- [x] **OIDC(Federated Credential)**로 GitHub Actions가 Azure에 인증하도록 설정한다
- [x] 코드 변경 시 자동으로 이미지가 빌드되어 ACR에 푸시되고, **K8s 매니페스트의 태그까지 업데이트**되는 파이프라인을 완성한다

---

## 세부 단원

### Phase 3-1 · CI/CD 개념과 Build-Test-Deploy (30분)

| 소단원 | 활동 |
| --- | --- |
| CI/CD가 필요한 이유 | 이대리가 3년간 겪은 사건 사례 강의 |
| 3단계 흐름 | Build(이미지 만들기) / Test(검증) / Deploy(배포) 각 단계의 성공 기준 |
| CI와 CD의 분리 | 현대 GitOps는 **CI는 Actions, CD는 ArgoCD**로 분리한다는 점 소개 |

### Phase 3-2 · GitHub Actions 기초 (1시간)

| 소단원 | 활동 |
| --- | --- |
| Workflow 파일 구조 | `.github/workflows/hello.yml` 만들어서 "Hello World" 출력 Workflow 완성 |
| 트리거 타입 | `push`, `pull_request`, `workflow_dispatch` 차이 |
| Job과 Step, Matrix | 여러 Job을 병렬로, Matrix로 Python 버전 3개 테스트 |
| Secrets 다루기 | Repository Secrets에 값 저장 → Workflow에서 `${{ secrets.XXX }}`로 참조 |

### Phase 3-3 · ACR 빌드 및 이미지 푸시 자동화 (30분)

| 소단원 | 활동 |
| --- | --- |
| Azure OIDC 설정 | `az ad app create` + `az ad sp create` + Federated Credential 설정 |
| Actions에서 Azure 로그인 | `azure/login@v2` with OIDC (API 키 저장 불필요!) |
| ACR 빌드 + 푸시 | `docker/build-push-action@v5` 또는 `az acr build` 사용 |
| 태그 전략 | `${{ github.sha }}` 앞 7자리 + `latest` 태그 병행 |

!!! tip "왜 Service Principal + 비밀키 방식을 쓰지 않나"
    옛날 가이드들은 Service Principal을 만들고 JSON 키를 GitHub Secret에 저장하는 방식을 썼지만, **OIDC 방식은 비밀키가 필요 없어 보안상 월등히 좋습니다.** 현업에서도 이 방식이 표준입니다.

### Phase 3-4 · 매니페스트 태그 자동 업데이트 (1시간)

이게 **패턴 A (단일 레포)**의 핵심입니다.

| 소단원 | 활동 |
| --- | --- |
| 왜 매니페스트를 자동 수정하나 | ArgoCD가 보는 건 Git의 매니페스트이므로, 이미지 태그가 바뀌어야 ArgoCD가 Sync 한다 |
| `yq`로 YAML 수정 | Workflow 안에서 `k8s/deployment.yaml`의 `image:` 태그를 새 SHA로 교체 |
| 자동 커밋 | `stefanzweifel/git-auto-commit-action@v5`로 변경사항을 main에 push |
| 무한 루프 방지 | 자동 커밋이 또 Workflow를 trigger 하지 않도록 `[skip ci]` 플래그 또는 paths-ignore 사용 |

!!! warning "실무에서는 레포를 분리합니다"
    패턴 A(단일 레포 + 자동 커밋)는 실습에서 편리하지만, 실무에서는 **앱 레포 ↔ 매니페스트 레포 분리**가 표준입니다. Phase 7 회고에서 "왜 실무에선 분리하는가"를 토론합니다.

---

## 예상 결과물

- :material-file-code: `.github/workflows/ci.yml` — `push` 시 빌드·푸시·매니페스트 업데이트까지 자동화된 Workflow
- :material-docker: ACR에 SHA 태그로 올라간 이미지 3개 (order-api, order-web, payment-api)
- :material-git: `k8s/*.yaml` 파일의 이미지 태그가 최신 커밋 SHA로 업데이트된 상태의 main 브랜치

---

## 다음 단계

[:material-arrow-left: Phase 2 · 복원력 실험실](../phase-2/index.md){ .md-button }
[Phase 4 · 사람 손 떼기 (2부) :material-arrow-right:](../phase-4/index.md){ .md-button .md-button--primary }
