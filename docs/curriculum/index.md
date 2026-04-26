# 15시간 커리큘럼

모듈 **[11] Composable Architecture 운영 자동화 및 고도화**의 공식 교수계획서(15시간)를 8개 Phase로 재구성했습니다.

---

## 한눈에 보는 커리큘럼

<div class="annotate" markdown>

| Phase | 제목 | 시간 | 학습 목표 |
| --- | --- | --- | --- |
| 0 | 복귀 — 다시 만난 한밭푸드 | 0.5h | 환경 세팅, 시즌 1 복기 |
| 1 | 연쇄 장애의 밤 | 1.5h | REST 통신 구조, 장애 전파 체험 |
| 2 | 복원력 실험실 | 1.0h | Retry, Circuit Breaker의 효과와 한계 |
| 3 | 사람 손 떼기 (1부) | 3.0h | GitHub Actions 기반 CI 파이프라인 |
| 4 | 사람 손 떼기 (2부) | 2.0h | ArgoCD GitOps 기반 CD |
| 5 | Agent란 무엇인가 | 3.0h | Agentic AI 개념, ReAct, Tool calling |
| 6 | 운영팀의 새 멤버 | 3.0h | Multi-Agent 설계 + 미니 프로젝트 |
| 7 | 회고와 제안서 | 1.0h | 의사결정 매트릭스, 팀별 발표 |
| | **합계** | **15.0h** | |

</div>

---

## 공식 교수계획서 매핑

교수계획서 3개 파트의 시간 배분과 Phase 매핑입니다.

### 파트 1 · 서비스 통신 및 장애 대응 전략 (3시간)

| 공식 중단원 | Phase | 비고 |
| --- | --- | --- |
| REST 기반 서비스 통신 구조 이해 (1h) | Phase 1-1 | 주문 API → 결제 API 호출 구조 분석 |
| 장애 전파 구조와 CA 운영 리스크 (1h) | Phase 1-2 | 결제 지연 → 주문 장애 체험 |
| Retry / Circuit Breaker 개념 및 한계 (1h) | Phase 2 | 직접 코드 적용 |

### 파트 2 · CI/CD 실습 (5시간)

| 공식 중단원 | Phase | 비고 |
| --- | --- | --- |
| CI/CD 파이프라인 개요 Build-Test-Deploy (3h) | Phase 3 | GitHub Actions 기초부터 ACR 푸시까지 |
| GitHub Actions 기반 CI/CD 실습 (2h) | Phase 4 | ArgoCD 연동 GitOps (패턴 A) |

!!! note "왜 GitHub Actions + ArgoCD 조합인가"
    공식 교수계획서는 "GitHub Actions 기반 CI/CD"라고만 명시합니다. 현업에서는 **CI는 GitHub Actions, CD는 GitOps 도구(ArgoCD/Flux)로 분리**하는 것이 표준에 가깝고, 이번 실습은 그 구조를 따릅니다.

### 파트 3 · Agentic AI와 CA 아키텍처 (7시간)

| 공식 중단원 | Phase | 비고 |
| --- | --- | --- |
| React 기반 Planning 개념, Agent+API+Tool 구조 (3h) | Phase 5 | 개념 + Azure OpenAI 기본 호출 + Tool calling |
| CA 환경에서 Multi-Agent 설계 (3h) | Phase 6 | 설계 패턴 + 미니 프로젝트 |
| 역량평가/프로그램 평가 (1h) | Phase 7 | 제안서 발표 + Q&A |

---

## Phase별 상세 시간 배분

### Phase 0 · 복귀 (0.5h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| 시즌 1 요약 + 미션 브리핑 | 10분 | CTO 메일 읽기, 3대 숙제 이해 |
| 환경 세팅 확인 | 20분 | VM/CLI 점검, AKS 접근 확인, OpenAI 엔드포인트 확인 |

### Phase 1 · 연쇄 장애의 밤 (1.5h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| REST 기반 서비스 통신 구조 이해 | 1h | 주문 API → 결제 API 호출 흐름 분석, 타임아웃 유무 점검 |
| 장애 전파 체험 | 0.5h | 결제 API에 지연 주입, 주문 API 스레드 포화 관찰 |

### Phase 2 · 복원력 실험실 (1.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| Retry의 효과와 역효과 | 0.5h | Retry 도입 → 재시도 폭주 체감 |
| Circuit Breaker 적용 | 0.5h | `pybreaker` 적용, 장애 격리 확인 |

### Phase 3 · 사람 손 떼기 1부 (3.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| CI/CD 개념과 Build-Test-Deploy | 0.5h | 파이프라인이 필요한 이유, 3단계 개요 |
| GitHub Actions 기초 | 1.0h | Workflow, Job, Step 구조 익히기 |
| ACR 빌드 및 이미지 푸시 자동화 | 0.5h | `az acr login` + `docker build/push` |
| 매니페스트 태그 자동 업데이트 | 1.0h | 패턴 A: 같은 레포의 YAML 태그 교체 → commit |

### Phase 4 · 사람 손 떼기 2부 (2.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| AKS 클러스터 확인 및 ArgoCD 설치 | 0.5h | Helm으로 ArgoCD 설치, LoadBalancer 노출 |
| ArgoCD Application 등록 | 1.0h | Git 레포를 소스로 지정, 자동 Sync 정책 |
| git push → 자동 배포 체험 | 0.5h | 실제 코드 변경 → 모니터에서 배포 확인 |

### Phase 5 · Agent란 무엇인가 (3.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| LLM vs Agent, ReAct 패턴 | 1.0h | 개념 강의 + 토론 |
| Azure OpenAI 기본 호출 | 1.0h | Python SDK로 ChatCompletion 호출 |
| Tool calling으로 Agent 만들기 | 1.0h | 실습: 로그 조회 Tool을 가진 Agent |

### Phase 6 · 운영팀의 새 멤버 (3.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| Multi-Agent 설계 패턴 | 1.0h | 3가지 패턴(Supervisor, Sequential, Parallel) 비교 |
| **미니 프로젝트**: 장애 분석 Agent 팀 | 2.0h | 팀별 설계 → 일부 구현 → 동료 리뷰 |

### Phase 7 · 회고와 제안서 (1.0h)

| 소단원 | 시간 | 활동 |
| --- | --- | --- |
| 의사결정 매트릭스 작성 | 30분 | ACA vs AKS, Agent 도입 여부 |
| 팀별 제안서 발표 | 30분 | "한밭푸드 운영 성숙도 제안서" 3분 피치 |

---

## 수업 모드별 시간

교수계획서 기준, 3가지 교수학습방법의 비중:

| 방법 | 시간 | Phase 내 위치 |
| --- | --- | --- |
| 강의 및 구조 설명 | 약 5.0h | 각 Phase 도입부, Phase 5 개념 |
| 사례 및 토론 중심 학습 | 약 3.5h | Phase 1 장애 분석, Phase 6 팀 설계, Phase 7 제안서 |
| 정리 및 질의응답 | 약 1.0h | Phase 7 |
| **실습 (핸즈온)** | 약 5.5h | Phase 2, 3, 4, 5 후반, 6 후반 |

!!! tip "실습 시간이 넉넉하지는 않습니다"
    15시간 중 실습 비중이 약 37%입니다. 강사님은 **Phase 3–4 (CI/CD)**와 **Phase 6 (미니 프로젝트)**에서 학생이 손을 많이 움직일 수 있도록 시간 관리를 권장합니다.

---

## 다음 단계

[:material-arrow-left: AS-IS vs TO-BE 아키텍처](../scenario/architecture.md){ .md-button }
[Phase 0 · 복귀 시작하기 :material-arrow-right:](../phase-0/index.md){ .md-button .md-button--primary }
