# Claude Code 작업 가이드 — 한밭푸드 시즌 2

> 이 문서는 Claude Code가 이 프로젝트에서 작업할 때 **반드시 먼저 읽어야 할** 컨텍스트입니다. 새 세션을 시작할 때마다 이 파일을 확인하세요.

---

## 1. 프로젝트 정체

- **무엇**: 15시간 강의 "Composable Architecture 운영 자동화 및 고도화"의 학생용 실습 가이드 사이트
- **기술 스택**: MkDocs + Material for MkDocs, GitHub Actions로 GitHub Pages 자동 배포
- **공개 URL**: https://skilleat-labs.github.io/composable-ops-labs/
- **강사**: 김누리 (Skilleat, @feeltechedu)
- **선행 작품**: 시즌 1 사이트 — https://skilleat-labs.github.io/container-service-labs/ (스타일·톤의 원본. 판단이 애매할 때 여기를 참조)

## 2. 시즌 2 스토리 (일관성 필수)

시즌 1에서 한밭푸드 주문 조회 시스템을 Azure Container Apps로 이관 완료. 시즌 2는 그 **1년 후** 이야기.

- **학생 역할**: 박대리 (시즌 1의 박주임이 승진)
- **팀 구성**: 김팀장(팀장), 이대리(시즌 1 박주임과 동료), **박대리(학생)**, 최인턴(신규)
- **발단 사건**: 지난 목요일 오후 2시, 결제 API가 외부 PG사 응답 지연으로 느려지자 주문 조회 API까지 스레드 풀 포화로 44분간 전체 장애
- **CTO의 3대 숙제**:
  1. 장애 전파 차단 (Timeout + Retry + Circuit Breaker)
  2. 사람 손 덜 가는 배포 (GitHub Actions + ArgoCD)
  3. AI Agent 운영 자동화 검토 (Azure OpenAI)

등장인물의 말투·대사는 `docs/index.md`, `docs/scenario/index.md`를 기준으로 일관성 유지.

## 3. 확정된 기술 결정 (절대 변경 금지)

| 항목 | 결정 | 이유 |
|------|------|------|
| CI 도구 | **GitHub Actions** | 공식 교수계획서 명시 |
| CD 도구 | **ArgoCD** | GitOps + UI 시각화, 채용 시장 우세 |
| 파이프라인 구조 | **패턴 A (단일 레포, 매니페스트 자동 커밋)** | 15시간 안에 end-to-end 성공 경험 |
| 매니페스트 | **순수 YAML** (Helm/Kustomize 미사용) | 학습 부담 최소화 |
| 런타임 | **AKS** (Korea Central) | ArgoCD가 K8s 네이티브 |
| LLM | **Azure OpenAI gpt-4o-mini** | 강사 제공 리소스 |
| OpenAI 리전 | East US 또는 Sweden Central | Korea Central 일부 모델 미지원 |
| Azure 인증 | **OIDC (Federated Credential)** | Service Principal 비밀키 방식 금지 |

실습 레포: `skilleat-labs/hanbat-order-app-s2` (별도 레포, 아직 미생성 — 아래 §6 참고)

## 4. 스타일 가이드 (시즌 1과 완벽히 일치시킬 것)

### Admonition 패턴

```markdown
!!! info "부제"       # 정보 안내
!!! tip "부제"        # 팁
!!! warning "부제"    # 경고
!!! success "✅ 확인 포인트"  # 실습 성공 검증
!!! failure "..."     # 실패 상황
!!! danger "부제"     # 보안/데이터 손실 경고
!!! quote "화자"      # 인물 대사
!!! note "부제"       # 참고
!!! question "부제"   # FAQ
??? failure "접힌 제목"  # 접이식 (물음표 3개)
??? info "강사 사전 준비 (학생은 건너뛰세요)"  # 강사 전용 섹션 패턴
```

### 코드 블록

반드시 title 속성을 붙인다:

````markdown
```bash title="터미널"
kubectl get nodes
```

```bash title="터미널 (Windows PowerShell)"
ssh labuser@1.2.3.4
```

```text title="출력 예시"
NAME     STATUS   ROLES
node1    Ready    <none>
```

```yaml title="k8s/deployment.yaml"
apiVersion: apps/v1
```
````

### 탭 그룹 (OS별 분기가 있을 때)

````markdown
=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    ssh labuser@<공용_IP_주소>
    ```

=== "Mac / Linux (Terminal)"

    ```bash title="터미널 (Mac / Linux)"
    ssh labuser@<공용_IP_주소>
    ```
````

### 페이지 하단 네비게이션

모든 서브 페이지 맨 아래는 반드시 이 패턴:

```markdown
## 다음 단계

[:material-arrow-left: 이전 페이지 제목](../previous/path.md){ .md-button }
[다음 페이지 제목 :material-arrow-right:](next-path.md){ .md-button .md-button--primary }
```

### 아이콘

Material Design Icons(`:material-xxx:`) 및 FontAwesome(`:fontawesome-xxx:`) 사용. 이모지(🔥 등)도 감정 강조용으로 시즌 1에서 드물게 사용. 남용 금지.

### 페이지 상단 구조

```markdown
# 페이지 제목

!!! info "예상 소요 30분"   # 또는 "PHASE X · 예상 X시간"

---

## (본문 시작)
```

### 기타 반복 패턴

- ✅ 확인 포인트: `!!! success "✅ 확인 포인트"` 로 각 Step 끝마다 성공 여부 체크
- 경고 메시지는 한국어, 명령줄·파라미터는 영어 그대로
- 따옴표 안 placeholder: `<강사_제공_API_키>`, `<본인_GitHub_사용자명>`
- 줄바꿈·가로줄(`---`) 사용해 섹션 구분

## 5. 작업 우선순위 (위에서 아래로)

### 🔥 우선순위 1: 실습 소스 레포 생성

**왜 먼저 하나**: Phase 1~6 세부 가이드가 실제 코드 경로·파일명을 참조하므로, 레포가 없으면 가이드가 뜬구름 잡는 소리가 됨.

생성 위치: `skilleat-labs/hanbat-order-app-s2` (새 GitHub 레포 / 별도 작업 디렉터리에서 생성)

필요한 구조:

```
hanbat-order-app-s2/
├── order-api/
│   ├── app.py                    # FastAPI, /api/orders/{id} 엔드포인트
│   │                             # 내부에서 payment-api를 동기 호출
│   │                             # Phase 1: 타임아웃 없음 (의도적)
│   │                             # Phase 2: Retry + Circuit Breaker 추가 지점
│   ├── requirements.txt          # fastapi, uvicorn, httpx, tenacity, pybreaker
│   └── Dockerfile
├── order-web/
│   ├── nginx.conf
│   ├── index.html                # 주문 조회 화면
│   └── Dockerfile
├── payment-api/
│   ├── app.py                    # FastAPI, /api/payments/{order_id}
│   │                             # FAULT_DELAY_MS 환경변수로 지연 주입 가능
│   │                             # FAULT_ERROR_RATE로 에러율 주입 가능
│   ├── requirements.txt
│   └── Dockerfile
├── k8s/
│   ├── namespace.yaml            # hanbat-<username> 네임스페이스
│   ├── order-api-deployment.yaml
│   ├── order-api-service.yaml
│   ├── order-web-deployment.yaml
│   ├── order-web-service.yaml
│   ├── payment-api-deployment.yaml
│   ├── payment-api-service.yaml
│   ├── ingress.yaml              # AKS default ingress
│   └── argocd-application.yaml   # Phase 4에서 사용
├── .github/
│   └── workflows/
│       └── .gitkeep              # 비어있음 — 학생이 Phase 3에서 채움
├── phase5/
│   └── skeleton/                 # Phase 5-6 Agent 실습 스켈레톤
│       ├── basic_chat.py
│       ├── log_agent.py
│       └── requirements.txt
├── phase6/
│   └── skeleton/                 # Multi-Agent 팀 프로젝트용
│       ├── agents/
│       │   ├── supervisor.py     # TODO 마커
│       │   ├── log_agent.py      # 일부 구현
│       │   ├── metrics_agent.py  # TODO
│       │   └── event_agent.py    # TODO
│       ├── tools/
│       │   ├── kubectl_logs.py
│       │   ├── kubectl_top.py
│       │   └── kubectl_events.py
│       ├── main.py
│       └── requirements.txt
├── docker-compose.yml            # 로컬 테스트용 (선택)
├── .gitignore
└── README.md
```

핵심 요구사항:
- `order-api`의 `payment-api` 호출 부분에 **Phase 1에선 타임아웃 없음**, Phase 2에서 학생이 Retry/CB 추가할 지점 명시 주석
- `payment-api`는 환경변수로 지연/에러 제어 가능해야 Phase 1 장애 재현이 가능
- 모든 Dockerfile은 멀티 아키텍처 빌드 가능(linux/amd64, linux/arm64) — 누리님 Mac M칩 로컬 테스트 고려
- K8s 매니페스트의 `image:` 태그는 placeholder(`<ACR>.azurecr.io/order-api:PLACEHOLDER`)로 두고 Phase 3에서 자동 치환됨
- ingress는 학습용으로 간단히, 실제 FQDN 대신 IP 기반

### 우선순위 2: Phase별 세부 실습 가이드

각 Phase는 개요(index.md)만 작성되어 있음. 세부 하위 페이지를 만들면서 `mkdocs.yml`의 `nav` 섹션도 업데이트해야 함.

**Phase 1 · 연쇄 장애의 밤** (2개 서브페이지)
- `phase-1/rest-call-structure.md` — REST 기반 서비스 통신 구조 이해 (1h)
  - 주문 API → 결제 API 호출 코드 리딩
  - 타임아웃·연결풀 개념
  - 호출 흐름 다이어그램 그리기 실습
- `phase-1/cascade-failure.md` — 장애 전파 체험 (0.5h)
  - `FAULT_DELAY_MS=8000` 주입
  - `hey -c 30`으로 부하
  - 스레드 포화 관찰

**Phase 2 · 복원력 실험실** (2개 서브페이지)
- `phase-2/retry-and-storm.md` — Retry의 효과와 역효과 (0.5h)
- `phase-2/circuit-breaker.md` — Circuit Breaker 적용 (0.5h)

**Phase 3 · 사람 손 떼기 (1부)** (4개 서브페이지)
- `phase-3/cicd-concept.md` — CI/CD 개념 (0.5h)
- `phase-3/actions-basics.md` — GitHub Actions 기초 (1h)
- `phase-3/acr-push.md` — ACR 빌드·푸시 자동화 + **OIDC 설정** (0.5h)
- `phase-3/manifest-update.md` — 매니페스트 태그 자동 업데이트 (패턴 A 핵심) (1h)

**Phase 4 · 사람 손 떼기 (2부)** (3개 서브페이지)
- `phase-4/argocd-install.md` — AKS 접근 + ArgoCD Helm 설치 (0.5h)
- `phase-4/application-register.md` — Application 리소스 등록 (1h)
- `phase-4/gitops-experience.md` — git push → 자동 배포 체험 (0.5h)

**Phase 5 · Agent란 무엇인가** (3개 서브페이지)
- `phase-5/llm-vs-agent.md` — LLM vs Agent, ReAct 패턴 (1h)
- `phase-5/aoai-basics.md` — Azure OpenAI 기본 호출 (1h)
- `phase-5/tool-calling.md` — Tool calling으로 로그 분석 Agent (1h)

**Phase 6 · 운영팀의 새 멤버** (2개 서브페이지)
- `phase-6/multi-agent-patterns.md` — 3가지 설계 패턴 (1h)
- `phase-6/mini-project.md` — 장애 분석 Agent 팀 미니 프로젝트 (2h)

## 6. 작업 시 반드시 지킬 것

1. **한 턴에 모든 걸 하지 말 것**: Phase 1 끝나면 검증 → 누리님 확인 → Phase 2. 세션 당 Phase 1~2개 권장.
2. **빌드 검증**: 파일 생성 후 반드시 `mkdocs build --strict`로 검증. 링크 깨짐·문법 오류 없어야 함.
3. **`mkdocs.yml` nav 동기화**: 새 서브페이지 만들 때마다 nav 섹션 업데이트.
4. **시즌 1 스타일 일관성**: 판단 애매할 때 https://skilleat-labs.github.io/container-service-labs/ 해당 Phase 페이지 참조.
5. **스토리 어조 유지**: 박대리·이대리·김팀장·최인턴의 성격과 말투는 `docs/scenario/index.md`와 `docs/phase-*/index.md`에 이미 정립됨. 새 대사 쓸 때 그 톤 유지.
6. **강사 사전 준비 섹션**: 학생이 혼자 할 수 없는 사전 작업(OIDC 앱 등록, AKS 생성 등)은 반드시 `??? info "강사 사전 준비 (학생은 건너뛰세요)"` 접이식으로 페이지 맨 위에 배치.
7. **코드는 실제 동작해야 함**: 실습 레포와 코드 경로가 일치해야 함. 뜬구름 잡는 의사코드 금지.
8. **보안**: API 키·비밀번호·Azure 구독 ID 등 민감 정보를 가이드에 하드코딩하지 말 것. placeholder 사용.

## 7. 로컬 검증

```bash
# 의존성 설치 (최초 1회)
pip install --break-system-packages mkdocs-material

# 빌드 검증 (엄격 모드)
mkdocs build --strict

# 로컬 미리보기
mkdocs serve
# → http://127.0.0.1:8000
```

## 8. 배포

main 브랜치에 push하면 `.github/workflows/deploy.yml`이 자동으로 빌드·배포.
GitHub 저장소 Settings → Pages → Source는 **GitHub Actions**로 설정되어 있어야 함 (최초 1회).

## 9. 저작권

모든 페이지 하단의 Copyright 문구는 `mkdocs.yml`의 `copyright` 필드에서 자동 삽입됨. 개별 페이지에는 추가하지 말 것.

```
Copyright © 2026 Skilleat · 본 자료는 교육 목적으로만 사용 가능
```

## 10. 참고 링크

- 시즌 1 사이트: https://skilleat-labs.github.io/container-service-labs/
- 시즌 1 레포: https://github.com/skilleat-labs/container-service-labs
- 시즌 1 실습 소스: https://github.com/skilleat-labs/hanbat-order-app
- Material for MkDocs 문서: https://squidfunk.github.io/mkdocs-material/reference/

---

**변경 이력**
- 2026-04-24: 초기 작성 (Claude Opus 4.7, 뼈대 완성 시점)
