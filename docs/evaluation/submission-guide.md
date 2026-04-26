# 제출 가이드

모든 산출물은 **본인 GitHub 레포**를 통해 제출합니다. 별도 파일 업로드 없이, 레포 URL만 강사에게 제출하면 됩니다.

---

## 제출 방법

### 1. 레포 공개 설정 확인

본인 fork한 `hanbat-order-app-s2` 레포가 **Public**인지 확인합니다.

GitHub 레포 → **Settings → General → Danger Zone → Change visibility → Public**

### 2. 제출 파일 위치

```
<본인_레포>/
├── .github/workflows/ci.yml                     # Phase 3 산출물
├── k8s/
│   ├── deployment.yaml
│   └── argocd-application.yaml                  # Phase 4 산출물
├── order-api/
│   └── app.py                                   # Phase 2 산출물 (Retry, CB 적용)
├── phase5/
│   └── log_agent.py                             # Phase 5 산출물
├── phase6/
│   └── <팀명>/                                  # Phase 6 팀 프로젝트
│       ├── agents/
│       ├── tools/
│       └── main.py
└── reports/
    ├── team_proposal.md                         # Phase 7 팀 제안서
    └── personal_reflection.md                   # 개인 회고
```

### 3. 팀 제안서 작성

`reports/team_proposal.md` 파일을 아래 템플릿에 따라 작성합니다.

=== "템플릿"

    ```markdown
    # 한밭푸드 운영 성숙도 제안서

    **팀명**: <팀명>
    **팀원**: <팀원 이름>
    **작성일**: YYYY-MM-DD

    ## 1. 이번 분기에 배운 것 (1줄)

    <여기에 작성>

    ## 2. 세 가지 의사결정

    ### 2-1. 플랫폼 (ACA vs AKS)
    - **우리 팀의 선택**: <ACA 유지 / AKS 전환 / Hybrid 중 택1>
    - **근거**:
      1. <근거 1>
      2. <근거 2>
      3. <근거 3>

    ### 2-2. 장애 대응 범위
    - **적용 대상**: <어떤 호출에 Retry/CB 적용할 것인가>
    - **근거**: <왜 그 범위인가>

    ### 2-3. Agent 도입 여부
    - **우리 팀의 선택**: <도입 안 함 / 읽기 전용 / 제한된 실행 / 자율 운영 중 택1>
    - **로드맵**: <지금 / 6개월 / 1년 후 무엇을>

    ## 3. 다음 분기 Top 3

    1. <할 일 1 (구체적으로)>
    2. <할 일 2>
    3. <할 일 3>

    ## 4. 위험 요소 및 가정

    - <가정 1>
    - <위험 1>
    ```

=== "예시 (부분)"

    ```markdown
    ### 2-1. 플랫폼 (ACA vs AKS)
    - **우리 팀의 선택**: ACA 유지 + 신규 서비스만 AKS 실험
    - **근거**:
      1. 현재 ACA 운영 경험이 1년 누적되어 있고 안정적
      2. AKS 전환 시 팀 학습 비용이 예상치 대비 큼 (실습에서 체감)
      3. GitOps 이점은 신규 서비스로 먼저 검증 후 판단
    ```

### 4. 개인 회고 작성

`reports/personal_reflection.md`

```markdown
# 개인 회고 — <본인 이름>

## 가장 인상 깊었던 순간
<200자 내외>

## 가장 어려웠던 개념
<200자 내외>

## 실무에 바로 적용할 것 하나
<200자 내외>
```

---

## 제출 확인

제출 완료 전 아래 체크리스트로 자가 점검하세요.

- [ ] 레포가 **Public**으로 설정되어 있다
- [ ] `.github/workflows/ci.yml`이 main 브랜치에 있고, Actions 탭에서 **녹색 체크** 기록이 있다
- [ ] `k8s/argocd-application.yaml`이 있다
- [ ] `reports/team_proposal.md`, `reports/personal_reflection.md`가 모두 채워져 있다
- [ ] 레포의 README에 **팀명, 본인 이름, 역할**이 간단히 명시되어 있다

---

## 최종 제출

아래 내용을 강사에게 제출:

```
이름: <본인>
팀: <팀명>
레포 URL: https://github.com/<username>/hanbat-order-app-s2
Phase 7 발표 녹화 (선택): <링크>
```

---

## 자주 묻는 질문

??? question "팀 프로젝트는 팀원 중 누구 레포에 올려야 하나요?"
    각 팀원이 **본인 fork에 동일한 파일**을 올리는 것을 권장합니다. (git pull로 팀원 코드를 받아오면 됨) 팀 공동 레포를 새로 만드는 것도 OK.

??? question "API 키나 비밀번호를 실수로 커밋했어요."
    1. **즉시 강사에게 알리세요** (Azure OpenAI 키 재발급 필요)
    2. 해당 커밋을 되돌리는 것만으로는 깃 히스토리에 남아있습니다 → `git filter-branch` 또는 `BFG Repo Cleaner` 사용
    3. 이후 `.env`, `*.key` 등은 `.gitignore`에 추가

??? question "Actions 실행이 실패해도 제출 가능한가요?"
    제출은 가능합니다. 단, 평가 시 "왜 실패했는가"를 **팀 제안서 부록**에 설명해주세요. 실패도 학습의 일부입니다.

---

## 다음 단계

[:material-arrow-left: 평가 개요](index.md){ .md-button }
[참고 자료 보기 :material-arrow-right:](../reference/common-errors.md){ .md-button .md-button--primary }
