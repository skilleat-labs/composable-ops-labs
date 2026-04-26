# Phase 5-1 · LLM vs Agent, ReAct 패턴

!!! info "예상 소요 1시간"

---

!!! quote "박대리"
    *"저... Agent랑 LLM이랑 뭐가 다른 거죠?"*

!!! quote "김팀장"
    *"LLM은 질문에 답하는 기계야. Agent는 목표를 위해 스스로 행동하는 기계고. 한 줄 차이처럼 들리지만 실제로는 완전히 다른 구조야."*

---

## LLM — 텍스트 입력 → 텍스트 출력

LLM(Large Language Model)은 단순합니다.

```
입력: "order-api 파드 상태가 어떻게 돼?"

출력: "죄송합니다. 저는 실시간 클러스터 정보에 접근할 수 없어서
       현재 파드 상태를 알 수 없습니다."
```

LLM은 **훈련 데이터**만 알고 있습니다.
한밭푸드 AKS 클러스터 상태는 훈련 데이터에 없습니다.
**질문하면 모른다고 합니다.**

---

## Agent — 루프 있는 LLM

Agent는 LLM에 **행동 능력**을 더한 것입니다.

```
입력: "order-api 파드 상태가 어떻게 돼?"

Agent 내부 루프:
  생각: "파드 상태를 알려면 kubectl 을 실행해야 해"
  행동: kubectl get pods -n hanbat-parkdaeri 실행
  관찰: order-api-xxx Running 1/1  payment-api-xxx CrashLoopBackOff
  생각: "payment-api 가 죽어있네. 로그도 봐야겠다"
  행동: kubectl logs deployment/payment-api 실행
  관찰: OOMKilled: container exceeded memory limit
  생각: "메모리 초과로 죽었구나. 이제 답할 수 있다"

출력: "payment-api 파드가 메모리 초과(OOMKilled)로 CrashLoop 상태입니다.
       k8s/payment-api-deployment.yaml 의 memory limit 증가가 필요합니다."
```

LLM이 **스스로 어떤 도구를 쓸지 결정하고, 결과를 보고 다음 행동을 결정**합니다.

---

## ReAct 패턴

Agent의 루프 구조는 **ReAct** 패턴으로 설명할 수 있습니다.

**Re**asoning + **Act**ing — 추론과 행동의 반복입니다.

```
사용자 질문
    │
    ▼
[Thought]  "이 질문에 답하려면 무엇이 필요한가?"
    │
    ▼
[Action]   도구 선택 → 실행 (kubectl, curl, DB 조회 등)
    │
    ▼
[Observation]  도구 실행 결과 수신
    │
    ├── 더 필요한 정보가 있나? → [Thought] 로 돌아감
    │
    └── 충분히 파악됐다 → [Answer] 최종 답변
```

!!! tip "ReAct를 이해하기 좋은 비유"
    **LLM**: 눈 감고 기억에 의존해 답하는 사람

    **Agent**: 질문 → "이걸 알려면 자료가 필요해" → 검색 → 자료 읽고 → "아직 부족해" → 추가 검색 → 메모 → 답변하는 사람

---

## 어떤 업무에 Agent가 맞나

| 업무 | Agent 적합성 | 이유 |
| --- | --- | --- |
| 장애 원인 분석 | ✅ 적합 | 로그·메트릭을 보고 단계적으로 추론 |
| 일상적 FAQ 답변 | ⚠️ 과함 | 단순 LLM 호출로 충분 |
| `kubectl delete` 자동 실행 | ❌ 위험 | 실수 시 복구 불가 — 반드시 사람 승인 필요 |
| 정기 리포트 생성 | ✅ 적합 | 여러 소스 조합 후 요약 |

!!! warning "Agent가 위험한 이유"
    Agent는 **스스로 명령을 결정**합니다.
    "파드를 재시작해줘" 라는 질문에 Agent가 `kubectl delete pod`를 실행할 수 있습니다.
    프로덕션에서 이 명령이 실수로 실행되면 서비스가 중단됩니다.

    Phase 5~6 실습에서는 **읽기 전용 도구(get, logs, describe)**만 제공합니다.
    쓰기 작업은 항상 사람이 확인 후 실행합니다.

---

## Tool Calling — Agent가 도구를 쓰는 방법

OpenAI API에는 **Tool Calling(Function Calling)** 기능이 있습니다.
LLM에게 "이런 함수가 있어"라고 알려주면, LLM이 필요할 때 "이 함수를 이 인수로 호출해줘"라고 요청합니다.

```python title="개념 코드"
# 개발자가 정의한 도구 명세
tools = [{
    "type": "function",
    "function": {
        "name": "get_pod_logs",
        "description": "지정한 서비스의 최근 Pod 로그를 조회한다",
        "parameters": {
            "type": "object",
            "properties": {
                "service_name": {"type": "string"},
                "tail_lines":   {"type": "integer"}
            }
        }
    }
}]

# LLM 응답에서 tool_call 이 오면
# → 실제 함수를 실행하고
# → 결과를 다시 LLM에 전달
```

LLM은 **어떤 함수를 어떤 인수로 부를지**를 결정합니다.
실제 함수를 실행하는 건 개발자 코드(Python)입니다.

---

## 이번 Phase에서 만들 것

| 단계 | 파일 | 결과 |
| --- | --- | --- |
| Phase 5-2 | `phase5/basic_chat.py` | Azure OpenAI 기본 호출 체험 |
| Phase 5-3 | `phase5/log_agent.py` | Tool Calling으로 로그 분석 Agent 실행 |

---

## 다음 단계

[:material-arrow-left: Phase 5 개요](index.md){ .md-button }
[Phase 5-2 · Azure OpenAI 기본 호출 :material-arrow-right:](aoai-basics.md){ .md-button .md-button--primary }
