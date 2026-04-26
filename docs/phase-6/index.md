# Phase 6 · 운영팀의 새 멤버

!!! info "예상 소요 3시간 · 난이도 ⭐⭐⭐"

---

## 스토리

Phase 5를 끝낸 저녁, 이대리가 슬랙 DM을 보냅니다.

!!! quote "이대리"
    *"박대리, 오늘 만든 로그 분석 Agent 괜찮네. 근데 생각해봐 — 장애가 나면 로그 분석만 하는 게 아니잖아. 메트릭도 봐야 하고, Slack에 알림도 보내야 하고, 심하면 Pod 재시작까지 해야 하고. 이걸 **에이전트 한 명**한테 다 시키는 게 맞을까?"*

!!! quote "박대리"
    *"...여러 명이 있어야 할 것 같은데요?"*

!!! quote "이대리"
    *"그게 Multi-Agent야. 오늘 오후엔 그걸 설계해볼 거예요."*

---

## 핵심 질문

1. 하나의 거대한 Agent vs 여러 작은 Agent — 각각의 장단점은?
2. Agent들 사이에 **누가 지휘하는가**를 어떻게 결정하나?
3. **Supervisor / Sequential / Parallel** 패턴은 언제 어떤 걸 쓰는가?
4. 실제로 동작하는 Multi-Agent 시스템을 2시간 안에 어디까지 만들 수 있나?

---

## 학습 목표

- [x] Multi-Agent 3가지 설계 패턴을 **구분해서 설명**한다
- [x] 한밭푸드 운영 상황에 맞는 Agent 팀을 **설계**한다
- [x] 팀 프로젝트로 동작하는 Multi-Agent 프로토타입을 **구현**한다 (일부 기능)
- [x] 동료 팀의 설계에 대해 **건설적인 피드백**을 준다

---

## 세부 단원

### Phase 6-1 · Multi-Agent 설계 패턴 (1시간)

| 패턴 | 구조 | 적합한 업무 |
| --- | --- | --- |
| **Supervisor** | 1명의 Supervisor Agent가 여러 Worker Agent를 지휘 | 동적으로 경로가 달라지는 복잡한 업무 |
| **Sequential (Pipeline)** | A → B → C 순서대로 실행 | 각 단계의 결과가 다음 단계의 입력인 경우 |
| **Parallel (Fan-out/Fan-in)** | 여러 Agent가 동시에 작업 후 결과를 합침 | 독립적인 정보 수집 여러 건 |

| 소단원 | 활동 |
| --- | --- |
| 3가지 패턴 강의 | 예시 다이어그램으로 구조 이해 |
| 실사례 분석 | 장애 대응 시나리오에 어떤 패턴이 맞는지 팀별 토론 |
| 도구 선택 | 오늘은 **라이브러리 없이** OpenAI SDK만으로 Multi-Agent 구현. (LangGraph, AutoGen 등은 참고만) |

!!! note "왜 라이브러리 없이 하나"
    LangGraph, AutoGen, CrewAI 같은 프레임워크들이 있지만, 교육 목적에서는 **원리를 이해하는 것**이 우선입니다. 라이브러리는 이후 본인 필요에 따라 선택하면 됩니다.

### Phase 6-2 · 미니 프로젝트: 장애 분석 Agent 팀 (2시간)

**팀당 3~4명으로 구성**합니다.

#### 미션

!!! quote "이대리의 요구사항"
    *"주문 조회 API가 죽었다. 장애 분석 Agent 팀을 붙여서, 원인을 30초 안에 브리핑해주면 좋겠어. 사람이 하던 일을 대체하는 게 아니라, **사람이 판단하기 전에 정보를 정리해주는** 역할."*

#### Agent 팀 구성 예시

| Agent | 역할 | Tool |
| --- | --- | --- |
| **Supervisor** | 다른 Agent에게 지시하고 최종 요약 | — |
| **Log Agent** | Pod 로그 수집 및 에러 패턴 추출 | `get_pod_logs()` |
| **Metrics Agent** | CPU/메모리/네트워크 상태 확인 | `get_pod_metrics()` |
| **Event Agent** | 최근 Kubernetes 이벤트 수집 | `get_k8s_events()` |

#### 제공되는 스켈레톤

강사가 제공하는 `phase6/skeleton/` 폴더에는 다음이 포함됩니다.

```
phase6/skeleton/
├── agents/
│   ├── supervisor.py          # TODO: 팀원이 채움
│   ├── log_agent.py           # 일부 구현 완료
│   ├── metrics_agent.py       # TODO
│   └── event_agent.py         # TODO
├── tools/
│   ├── kubectl_logs.py        # 구현 완료
│   ├── kubectl_top.py         # 구현 완료
│   └── kubectl_events.py      # 구현 완료
├── main.py                    # 진입점
└── README.md
```

#### 진행 방식

1. **팀 설계 (30분)** — 어떤 패턴을 쓸지, 각 Agent의 역할을 종이에 설계
2. **구현 (60분)** — 팀원이 파트를 나눠 코딩
3. **통합 + 데모 (20분)** — 각 팀이 실제 장애 상황을 재현하고 Agent 팀의 분석 결과 시연
4. **상호 리뷰 (10분)** — 다른 팀의 설계에 질문과 피드백

---

## 평가 기준

이 Phase의 산출물은 Phase 7 최종 제안서의 핵심이 됩니다. 기술 완성도 3할, **설계 판단** 7할.

- [ ] 어떤 패턴을 골랐고, 왜 그 패턴인가?
- [ ] Agent 간 **책임 분리**가 명확한가?
- [ ] 한밭푸드 **실제 운영 상황**에서 쓸 만한가?
- [ ] 위험 요소와 한계를 스스로 인식하고 있는가?

---

## 예상 결과물

- :material-account-group: 팀별 Multi-Agent 프로토타입 코드
- :material-draw-pen: 설계 다이어그램 (손그림/화이트보드 캡처)
- :material-presentation: Phase 7 발표용 초안 슬라이드 1~2장

---

## 다음 단계

[:material-arrow-left: Phase 5 · Agent란 무엇인가](../phase-5/index.md){ .md-button }
[Phase 7 · 회고와 제안서 :material-arrow-right:](../phase-7/index.md){ .md-button .md-button--primary }
