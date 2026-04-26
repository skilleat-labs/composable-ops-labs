# Phase 6-2 · 미니 프로젝트: 장애 분석 Agent 팀

!!! info "예상 소요 2시간"

---

!!! quote "이대리"
    *"이론은 충분해. 이제 실제로 만들어봐. 팀당 30초 브리핑 목표야."*

---

## 미션

!!! quote "이대리의 요구사항"
    *"주문 조회 API가 죽었다. 장애 분석 Agent 팀을 붙여서, 원인을 30초 안에 브리핑해주면 좋겠어. 사람이 하던 일을 대체하는 게 아니라, **사람이 판단하기 전에 정보를 정리해주는** 역할."*

**팀당 3~4명으로 구성합니다.**

---

## 스켈레톤 코드 확인

```bash title="터미널"
cd ~/hanbat-order-app-s2/phase6/skeleton
ls -R
```

```text title="출력 예시"
agents/
  supervisor.py      ← TODO: 팀에서 완성
  log_agent.py       ← 일부 구현 완료
  metrics_agent.py   ← TODO
  event_agent.py     ← TODO
tools/
  kubectl_logs.py    ← 구현 완료
  kubectl_top.py     ← 구현 완료
  kubectl_events.py  ← 구현 완료
main.py              ← 진입점
requirements.txt
```

```bash title="터미널"
pip install -r requirements.txt
```

---

## Step 1. 완성된 도구 코드 읽기 (10분)

세 개의 도구는 이미 완성되어 있습니다. 각자 한 개씩 읽고 팀에 설명해봅니다.

```bash title="터미널"
cat tools/kubectl_logs.py
cat tools/kubectl_top.py
cat tools/kubectl_events.py
```

```python title="tools/kubectl_logs.py"
import subprocess, os

NAMESPACE = os.environ.get("K8S_NAMESPACE", "hanbat-parkdaeri")

def get_pod_logs(service_name: str, tail_lines: int = 50) -> str:
    """지정한 서비스의 최근 Pod 로그를 반환한다."""
    result = subprocess.run(
        ["kubectl", "logs", f"deployment/{service_name}",
         "-n", NAMESPACE, f"--tail={tail_lines}"],
        capture_output=True, text=True, timeout=15,
    )
    return result.stdout or result.stderr or "(로그 없음)"
```

```python title="tools/kubectl_top.py"
import subprocess, os

NAMESPACE = os.environ.get("K8S_NAMESPACE", "hanbat-parkdaeri")

def get_pod_metrics(namespace: str = None) -> str:
    """Pod CPU/메모리 사용량을 반환한다 (kubectl top pods)."""
    ns = namespace or NAMESPACE
    result = subprocess.run(
        ["kubectl", "top", "pods", "-n", ns],
        capture_output=True, text=True, timeout=15,
    )
    return result.stdout or result.stderr or "(메트릭 없음)"
```

```python title="tools/kubectl_events.py"
import subprocess, os

NAMESPACE = os.environ.get("K8S_NAMESPACE", "hanbat-parkdaeri")

def get_k8s_events(namespace: str = None) -> str:
    """최근 Kubernetes 이벤트를 반환한다."""
    ns = namespace or NAMESPACE
    result = subprocess.run(
        ["kubectl", "get", "events", "-n", ns,
         "--sort-by=.lastTimestamp"],
        capture_output=True, text=True, timeout=15,
    )
    return result.stdout or result.stderr or "(이벤트 없음)"
```

---

## Step 2. 팀 설계 (30분)

코드보다 **설계가 먼저**입니다. 종이나 화이트보드에 그립니다.

### 결정해야 할 것

1. **어떤 패턴을 쓰나?** — Supervisor, Sequential, Parallel 중 선택
2. **각 Agent의 입력/출력은?** — 무엇을 받아서 무엇을 반환하나
3. **Agent 간 정보 전달 방식은?** — 이전 결과를 어떻게 다음 Agent에게 넘기나
4. **실패 처리는?** — 하나의 Agent가 에러를 내면 전체가 멈추나, 계속 진행하나

### 설계 템플릿

```
패턴 선택: _______________

[사용자 입력] → [Agent 1: _______] → [Agent 2: _______] → [최종 출력]

각 Agent 역할:
- Agent 1: 입력(_____), 출력(_____)
- Agent 2: 입력(_____), 출력(_____)
- ...

실패 처리: _______________
```

!!! tip "설계 힌트 — Supervisor 패턴 선택 시"
    Supervisor가 Worker를 호출하는 방법:
    ```python
    # Supervisor 도구 목록에 이걸 추가
    {
        "name": "call_log_agent",
        "description": "Log Agent를 호출해서 지정한 서비스의 로그를 분석한다",
        "parameters": {
            "service_name": {"type": "string"},
            "question": {"type": "string", "description": "Log Agent에게 물어볼 질문"}
        }
    }
    ```
    Supervisor가 이 "도구"를 호출하면, 실제로 `log_agent.run(question)` 을 실행합니다.

---

## Step 3. 구현 (60분)

팀원이 파트를 나눠서 코딩합니다.

### log_agent.py (일부 구현 완료)

```python title="agents/log_agent.py"
import json, os
from openai import AzureOpenAI
from tools.kubectl_logs import get_pod_logs
from tools.kubectl_top import get_pod_metrics  # 필요 시 활용

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version="2024-02-01",
)
deployment = os.environ.get("AZURE_OPENAI_DEPLOYMENT", "gpt-4o-mini")

SYSTEM_PROMPT = """당신은 한밭푸드 운영팀의 로그 분석 전문 Agent입니다.
kubectl logs 결과를 받아 에러 패턴, 예외, 비정상 응답을 찾아 요약합니다.
없으면 "정상"이라고 답합니다. 한국어로 간결하게 답합니다."""

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_pod_logs",
            "description": "지정한 서비스의 최근 Pod 로그를 조회한다",
            "parameters": {
                "type": "object",
                "properties": {
                    "service_name": {"type": "string"},
                    "tail_lines": {"type": "integer", "default": 50},
                },
                "required": ["service_name"],
            },
        },
    }
]

def run(question: str) -> str:
    """Log Agent 실행. 질문을 받아 로그를 조회하고 분석 결과를 반환한다."""
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": question},
    ]
    for _ in range(5):
        resp = client.chat.completions.create(
            model=deployment,
            messages=messages,
            tools=tools,
            tool_choice="auto",
            temperature=0,
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:
            return msg.content
        messages.append(msg)
        for tc in msg.tool_calls:
            args   = json.loads(tc.function.arguments)
            result = get_pod_logs(**args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": result,
            })
    return "분석 실패: 최대 루프 초과"
```

### metrics_agent.py (TODO)

Phase 5의 `log_agent.py`를 참고해서 작성합니다. 도구는 `get_pod_metrics`를 씁니다.

```python title="agents/metrics_agent.py (팀에서 완성)"
# TODO: log_agent.py 구조를 그대로 따라서
# - SYSTEM_PROMPT: "CPU/메모리 메트릭 분석 전문 Agent"
# - tools: get_pod_metrics 함수 정의
# - run(question): log_agent.py와 동일한 ReAct 루프
```

### event_agent.py (TODO)

```python title="agents/event_agent.py (팀에서 완성)"
# TODO: get_k8s_events 도구를 사용하는 Agent
# "최근 Kubernetes 이벤트(OOMKilled, Evicted, FailedScheduling 등)를 찾아 요약"
```

### supervisor.py (TODO)

```python title="agents/supervisor.py (팀에서 완성)"
import json, os
from openai import AzureOpenAI
from agents import log_agent, metrics_agent, event_agent

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version="2024-02-01",
)
deployment = os.environ.get("AZURE_OPENAI_DEPLOYMENT", "gpt-4o-mini")

SYSTEM_PROMPT = """당신은 한밭푸드 운영팀 장애 대응 Supervisor입니다.
장애 상황을 파악하기 위해 Log Agent, Metrics Agent, Event Agent 중 필요한 것을 호출합니다.
모든 정보를 수집한 후 30초 브리핑으로 요약합니다. 한국어로 답합니다."""

tools = [
    {
        "type": "function",
        "function": {
            "name": "call_log_agent",
            "description": "Log Agent를 호출해서 지정 서비스의 로그를 분석한다",
            "parameters": {
                "type": "object",
                "properties": {
                    "service_name": {"type": "string"},
                    "question": {"type": "string"},
                },
                "required": ["service_name", "question"],
            },
        },
    },
    # TODO: call_metrics_agent, call_event_agent 추가
]

def run_tool(name: str, args: dict) -> str:
    if name == "call_log_agent":
        return log_agent.run(f"{args['service_name']} 서비스: {args['question']}")
    # TODO: metrics_agent, event_agent 연결
    return f"[unknown tool: {name}]"

def run(situation: str) -> str:
    """Supervisor 실행. 장애 상황을 받아 Agent 팀을 조율하고 브리핑을 반환한다."""
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": situation},
    ]
    for step in range(10):
        resp = client.chat.completions.create(
            model=deployment,
            messages=messages,
            tools=tools,
            tool_choice="auto",
            temperature=0,
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:
            return msg.content
        print(f"[Step {step+1}] Supervisor → {[tc.function.name for tc in msg.tool_calls]}")
        messages.append(msg)
        for tc in msg.tool_calls:
            args   = json.loads(tc.function.arguments)
            result = run_tool(tc.function.name, args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": result,
            })
    return "브리핑 실패: 최대 루프 초과"
```

### main.py (진입점)

```python title="main.py"
import os
from agents import supervisor

if __name__ == "__main__":
    situation = """
    order-api 파드가 간헐적으로 502를 반환하고 있습니다.
    5분 전부터 시작됐고, 배포는 1시간 전에 완료됐습니다.
    원인을 분석해서 브리핑해주세요.
    """
    print("=== 장애 분석 Agent 팀 ===")
    print(f"상황: {situation.strip()}\n")
    result = supervisor.run(situation)
    print("\n=== 30초 브리핑 ===")
    print(result)
```

---

## Step 4. 환경변수 설정 및 실행

```bash title="터미널"
export K8S_NAMESPACE="hanbat-<본인_GitHub_사용자명>"
```

```bash title="터미널"
python3 main.py
```

```text title="출력 예시"
=== 장애 분석 Agent 팀 ===
상황: order-api 파드가 간헐적으로 502를 반환하고 있습니다...

[Step 1] Supervisor → ['call_log_agent']
[Step 2] Supervisor → ['call_metrics_agent']

=== 30초 브리핑 ===
**장애 분석 브리핑 (2026-04-26 14:32)**

🔴 order-api — 502 간헐 발생
- 로그: payment-api 호출 타임아웃 반복 (최근 10분, 23건)
- 메트릭: order-api CPU 정상 (12%), payment-api 메모리 94% (한도: 128Mi)
- 이벤트: payment-api OOMKilled 2회 (14:28, 14:31)

**원인 추정:** payment-api 메모리 부족으로 간헐 재시작 → order-api 타임아웃

**권장 조치:**
1. `k8s/payment-api-deployment.yaml` memory limit을 256Mi로 증가 후 git push
2. 즉시 완화: `kubectl rollout restart deployment/payment-api -n hanbat-<사용자명>`
```

!!! success "✅ 확인 포인트"
    - `[Step N] Supervisor → [...]` 로그가 출력됐다
    - 30초 브리핑이 실제 클러스터 상태를 반영했다
    - Supervisor가 어떤 Agent를 호출할지 스스로 결정했다

---

## Step 5. 장애 상황 주입 후 재실행

실제 장애를 만들어서 Agent 팀이 잡아내는지 확인합니다.

```bash title="터미널 — payment-api 메모리 한도 낮춰서 OOMKill 유도"
kubectl set resources deployment/payment-api \
  -n hanbat-${GITHUB_USER} \
  --limits=memory=32Mi
```

```bash title="터미널 — 부하 주입"
hey -c 10 -n 50 http://<INGRESS_IP>/api/orders/1
```

```bash title="터미널 — Agent 팀 실행"
python3 main.py
```

브리핑에 OOMKilled 언급이 나오면 성공입니다.

```bash title="터미널 — 실습 후 복구"
kubectl set resources deployment/payment-api \
  -n hanbat-${GITHUB_USER} \
  --limits=memory=128Mi
```

---

## Step 6. 팀 데모 발표 (20분)

각 팀이 순서대로 발표합니다. (팀당 5분)

발표 항목:
1. **어떤 패턴을 골랐나** — 이유 1줄
2. **실제 Agent 팀 실행** — 장애 상황 주입 후 브리핑 시연
3. **아쉬운 점 or 다음에 추가하고 싶은 것**

---

## Step 7. 상호 리뷰 (10분)

다른 팀의 설계를 보고 다음 중 하나를 골라 질문합니다.

- Supervisor가 잘못된 판단을 내리면 어떻게 감지하나?
- Parallel로 바꾸면 지금보다 빨라지는가? 무엇이 달라지는가?
- 실제 프로덕션에서 이 Agent 팀을 24시간 켜두면 어떤 문제가 생길까?

---

## 정리

| 확인 내용 | 결과 |
| --- | --- |
| Agent 팀 설계 (패턴 선택 + 역할 분리) | ✅ |
| Supervisor → Worker 호출 구조 구현 | ✅ |
| 실제 클러스터 장애에서 브리핑 생성 | ✅ |
| 팀 데모 발표 | ✅ |

!!! success "✅ Phase 6 완료 체크리스트"
    - [ ] 설계 다이어그램 작성 완료 (손그림/화이트보드 캡처)
    - [ ] `python3 main.py` 실행 → Supervisor 루프 로그 확인
    - [ ] 장애 주입 후 브리핑에 원인이 포함됨
    - [ ] 팀 데모 발표 완료

---

## 다음 단계

[:material-arrow-left: Phase 6-1 · Multi-Agent 설계 패턴](multi-agent-patterns.md){ .md-button }
[Phase 7 · 회고와 제안서 :material-arrow-right:](../phase-7/index.md){ .md-button .md-button--primary }
