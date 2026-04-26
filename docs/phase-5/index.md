# Phase 5 · Agent란 무엇인가

!!! info "예상 소요 3시간 · 난이도 ⭐⭐⭐"

---

## 스토리

회의실. 커피가 식어가는 오후. CTO가 화면에 본사 이메일 한 통을 띄웁니다.

!!! quote "본사 CTO"
    *"요즘 **Agentic AI**라는 게 유행입니다. 본사에서는 이걸 쿠버네티스 운영 자동화에 써볼 수 있을지 각 지사에 검토를 요청합니다. 답변 기한 다음 주. 한밭푸드는 여러분이 주력이 되어 답변을 준비해주세요."*

박대리가 손을 듭니다.

!!! quote "박대리"
    *"저... Agent랑 LLM이랑 뭐가 다른 거죠?"*

!!! quote "김팀장"
    *"그게 정확히 오늘 배울 주제예요."*

---

## 핵심 질문

1. **LLM**(단순 챗봇)과 **Agent**는 무엇이 다른가?
2. **ReAct** 패턴이란 무엇이며, 왜 중요한가?
3. Tool calling으로 Agent가 외부 세계와 상호작용하는 방식은?
4. 우리가 만들 Agent는 **어떤 일**을 할 수 있는가?

---

## 학습 목표

- [x] LLM 호출과 Agent의 구조적 차이를 **생각 → 행동 → 관찰 → 생각** 루프로 설명한다
- [x] Azure OpenAI Python SDK로 기본 Chat Completion을 호출한다
- [x] Tool calling(Function calling)으로 LLM이 외부 함수를 호출하게 만든다
- [x] Pod 로그를 조회하는 **간단한 운영 Agent**를 완성한다

---

## 세부 단원

### Phase 5-1 · LLM vs Agent, ReAct 패턴 (1시간)

| 소단원 | 활동 |
| --- | --- |
| 정의 비교 | LLM = 텍스트 입력 → 텍스트 출력 / Agent = **루프 있는 LLM** |
| ReAct 패턴 | **Re**asoning + **Act**ing. 생각(Thought) → 행동(Action) → 관찰(Observation) → 다시 생각 |
| 사례 분석 | "서울 오늘 날씨 알려줘" 질문에 LLM vs Agent가 각각 어떻게 답하는지 비교 |
| 토론 | **어떤 업무에 Agent가 적합하고, 어떤 업무는 부적합한가?** |

!!! tip "ReAct 이해하기 좋은 비유"
    - **LLM**: 눈 감고 떠오르는 대로 답하는 사람
    - **Agent**: 질문 → 관련 자료 검색 → 자료 읽고 → 메모하고 → 다시 검색 → 답변하는 사람

### Phase 5-2 · Azure OpenAI 기본 호출 (1시간)

| 소단원 | 활동 |
| --- | --- |
| Python SDK 설정 | `pip install openai` 확인, 환경변수 로드 |
| 기본 Chat Completion | system + user 메시지로 기본 호출 |
| 대화 맥락 유지 | messages 배열에 대화 이력 누적 |
| 파라미터 실험 | `temperature`, `max_tokens`, `top_p` 변화 관찰 |
| 한계 체감 | 질문: "이 클러스터의 Pod 상태 알려줘" → LLM은 **모른다** |

!!! info "이 단계에서 깨달아야 할 것"
    LLM 단독으로는 한밭푸드 클러스터 상태를 **절대** 알 수 없습니다. 자체 지식(훈련 데이터)만으로 답하기 때문입니다. **이게 Agent가 필요한 이유**입니다.

### Phase 5-3 · Tool calling으로 Agent 만들기 (1시간)

이 단계에서 첫 번째 진짜 Agent를 만듭니다.

| 소단원 | 활동 |
| --- | --- |
| Tool 정의 | JSON Schema로 `get_pod_logs(namespace, pod_name)` 함수 명세 작성 |
| Tool 구현 | Python 함수로 `kubectl logs` 실행 (subprocess) 후 결과 반환 |
| Agent 루프 구현 | LLM 응답에서 tool_call 감지 → Tool 실행 → 결과를 다시 LLM에 전달 → 최종 답변 |
| 실전 실험 | "order-api 파드의 최근 에러 로그 3개 요약해줘" → Agent가 스스로 명령어 결정 후 실행 |

```python title="agent.py 구조 (개념)"
# 의사 코드
while True:
    response = llm.complete(messages, tools=[get_pod_logs])
    if response.has_tool_call():
        result = execute_tool(response.tool_call)
        messages.append(result)
    else:
        print(response.content)
        break
```

---

## 예상 결과물

- :material-file-code: `phase5/basic_chat.py` — 기본 Azure OpenAI 호출 스크립트
- :material-tools: `phase5/log_agent.py` — `get_pod_logs` Tool을 가진 운영 Agent
- :material-file-chart: Agent가 실제 Pod 로그를 분석해서 한국어 요약을 출력한 터미널 캡처

---

## 토론 주제 (Phase 내)

| 주제 | 예시 질문 |
| --- | --- |
| Agent의 위험성 | 잘못된 명령어를 실행하면 어떻게 막나? (`kubectl delete`...) |
| 비용 | 매 호출마다 토큰 사용료가 발생. 어떤 작업에 쓰는 게 타당한가? |
| 신뢰도 | LLM이 환각(hallucination)으로 틀린 로그 분석을 하면? |

---

## 다음 단계

[:material-arrow-left: Phase 4 · 사람 손 떼기 (2부)](../phase-4/index.md){ .md-button }
[Phase 6 · 운영팀의 새 멤버 :material-arrow-right:](../phase-6/index.md){ .md-button .md-button--primary }
