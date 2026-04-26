# Phase 5-3 · Tool Calling으로 Agent 만들기

!!! info "예상 소요 1시간"

---

!!! quote "박대리"
    *"그러면 진짜로 kubectl 실행시키는 Agent 만들어보는 거잖아요?"*

!!! quote "이대리"
    *"맞아. 근데 지금은 읽기 전용만이야. get, logs, describe. 지워달라고 해도 못하게 해놨어."*

---

## 이번 단계에서 만들 것

```
사용자: "order-api 로그 분석해줘"
    │
    ▼
[Agent — log_agent.py]
  Thought: "로그를 보려면 get_pod_logs 도구가 필요해"
  Action:  get_pod_logs("order-api", tail_lines=50)
  Obs:     [실제 kubectl logs 결과]
  Thought: "에러 패턴이 보이는지 분석할 수 있겠다"
  Answer:  "order-api 최근 로그에 OOMKilled 흔적이 있습니다..."
```

스켈레톤 파일 `phase5/skeleton/log_agent.py`가 이미 준비되어 있습니다.

---

## Step 1. 스켈레톤 코드 확인

```bash title="터미널"
cd ~/hanbat-order-app-s2/phase5/skeleton
cat log_agent.py
```

핵심 구조는 세 부분으로 나뉩니다.

### 1. 도구 정의

```python title="log_agent.py — 도구 정의"
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_pod_logs",
            "description": "지정한 서비스의 최근 Pod 로그를 조회한다",
            "parameters": {
                "type": "object",
                "properties": {
                    "service_name": {
                        "type": "string",
                        "description": "서비스 이름 (order-api, payment-api, order-web 중 하나)",
                    },
                    "tail_lines": {
                        "type": "integer",
                        "description": "조회할 최근 로그 줄 수 (기본값 50)",
                        "default": 50,
                    },
                },
                "required": ["service_name"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_pod_status",
            "description": "지정한 네임스페이스의 Pod 상태를 조회한다",
            "parameters": {
                "type": "object",
                "properties": {
                    "namespace": {
                        "type": "string",
                        "description": "쿠버네티스 네임스페이스",
                    }
                },
                "required": ["namespace"],
            },
        },
    },
]
```

### 2. 도구 실행 함수

```python title="log_agent.py — 도구 실행"
import subprocess, json

NAMESPACE = os.environ.get("K8S_NAMESPACE", "hanbat-parkdaeri")

def run_tool(name: str, args: dict) -> str:
    if name == "get_pod_logs":
        service = args["service_name"]
        tail    = args.get("tail_lines", 50)
        result  = subprocess.run(
            ["kubectl", "logs", f"deployment/{service}",
             "-n", NAMESPACE, f"--tail={tail}"],
            capture_output=True, text=True, timeout=15,
        )
        return result.stdout or result.stderr

    if name == "get_pod_status":
        ns = args.get("namespace", NAMESPACE)
        result = subprocess.run(
            ["kubectl", "get", "pods", "-n", ns],
            capture_output=True, text=True, timeout=15,
        )
        return result.stdout or result.stderr

    return f"[unknown tool: {name}]"
```

### 3. ReAct 루프

```python title="log_agent.py — ReAct 루프"
def run_agent(user_question: str) -> str:
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": user_question},
    ]

    for step in range(10):          # 최대 10번 루프
        response = client.chat.completions.create(
            model=deployment,
            messages=messages,
            tools=tools,
            tool_choice="auto",
            temperature=0,
        )
        msg = response.choices[0].message

        # LLM이 도구를 부르지 않으면 → 최종 답변
        if not msg.tool_calls:
            return msg.content

        # 도구 호출이 있으면 → 실행 후 결과를 메시지에 추가
        messages.append(msg)
        for tc in msg.tool_calls:
            args   = json.loads(tc.function.arguments)
            result = run_tool(tc.function.name, args)
            messages.append({
                "role":         "tool",
                "tool_call_id": tc.id,
                "content":      result,
            })

    return "최대 루프 횟수를 초과했습니다."
```

---

## Step 2. 환경변수 추가

Phase 5-2에서 설정한 변수에 네임스페이스를 추가합니다.

```bash title="터미널"
export K8S_NAMESPACE="hanbat-<본인_GitHub_사용자명>"
```

---

## Step 3. Agent 실행

```bash title="터미널"
python3 log_agent.py
```

```text title="출력 예시"
=== 로그 분석 Agent ===
질문: order-api 최근 로그에 에러 있어?

[Step 1] Tool call: get_pod_logs(service_name='order-api', tail_lines=50)
[Step 1] Tool result: (50줄의 실제 kubectl logs 출력)

[최종 답변]
order-api의 최근 로그를 분석한 결과, 특별한 에러는 발견되지 않았습니다.
주요 상태:
- 요청 처리 정상 (200 OK 다수)
- 메모리 경고나 OOMKilled 흔적 없음
- 평균 응답 시간 정상 범위

현재 서비스는 안정적으로 동작 중입니다.
```

!!! success "✅ 확인 포인트"
    - `[Step N] Tool call:` 로그가 출력됐다 — Agent가 도구를 선택했다는 증거
    - `[최종 답변]` 이 실제 로그 내용을 반영해서 나왔다

---

## Step 4. 멀티 스텝 질문 체험

Agent가 여러 번 도구를 호출하는 상황을 만들어봅니다.

`log_agent.py` 마지막 줄의 질문을 바꿉니다.

```python title="log_agent.py (마지막 줄 수정)"
answer = run_agent("전체 파드 상태 확인하고, 문제 있는 파드가 있으면 그 서비스 로그도 보여줘")
print(answer)
```

```bash title="터미널"
python3 log_agent.py
```

```text title="출력 예시 (payment-api CrashLoop 상태일 때)"
[Step 1] Tool call: get_pod_status(namespace='hanbat-parkdaeri')
[Step 1] Tool result:
NAME                            READY   STATUS             RESTARTS
order-api-7d8f9b4c6-xk9pq      1/1     Running            0
payment-api-5c9d7b8f4-mn2lp    0/1     CrashLoopBackOff   5
order-web-6b8c7d9f3-jk4np      1/1     Running            0

[Step 2] Tool call: get_pod_logs(service_name='payment-api', tail_lines=30)
[Step 2] Tool result:
OOMKilled: container exceeded memory limit (128Mi)
...

[최종 답변]
파드 상태 분석 결과:

**🔴 이상 감지: payment-api**
- 상태: CrashLoopBackOff (재시작 5회)
- 원인: OOMKilled — 메모리 한도(128Mi) 초과

**권장 조치:**
`k8s/payment-api-deployment.yaml`의 `resources.limits.memory`를 256Mi 이상으로 늘린 후
git push → ArgoCD 자동 배포로 반영하세요.

order-api, order-web은 정상 동작 중입니다.
```

Agent가 **스스로 판단해서 2번 도구를 호출**했습니다.
이것이 ReAct 루프입니다.

!!! success "✅ 확인 포인트"
    - Step 1에서 파드 상태를 조회했다
    - 문제 있는 파드를 발견하고 Step 2에서 **자동으로** 로그를 추가 조회했다
    - 최종 답변이 kubectl 결과를 근거로 구체적인 조치를 제안했다

---

## Step 5. 쓰기 작업 방어 확인

Agent가 위험한 명령을 요청받았을 때 어떻게 동작하는지 확인합니다.

```python title="log_agent.py (마지막 줄 수정)"
answer = run_agent("payment-api 파드 지워줘. 재시작하면 살아날 것 같아서.")
print(answer)
```

```bash title="터미널"
python3 log_agent.py
```

```text title="출력 예시"
[최종 답변]
죄송합니다. 저는 현재 읽기 전용 도구(로그 조회, 상태 확인)만 사용할 수 있습니다.
파드 삭제나 재시작 같은 쓰기 작업은 제 도구 목록에 없습니다.

파드를 재시작하려면 직접 실행하세요:
kubectl delete pod -l app=payment-api -n hanbat-<사용자명>
```

도구 목록에 `kubectl delete` 가 없으므로 Agent는 실행할 수 없습니다.
**"무엇을 할 수 있는가"는 도구 목록이 결정합니다.**

!!! warning "Agent 보안 설계 원칙"
    Agent에게 제공하는 도구 목록이 **권한 경계**입니다.
    쓰기 도구를 주지 않으면 쓰기를 할 수 없습니다.
    Phase 6에서도 이 원칙을 유지합니다.

---

## Tool Calling 흐름 전체 그림

```
개발자 코드                    Azure OpenAI API
─────────────────────────────────────────────────────
messages + tools 정의  ──→  LLM
                       ←──  tool_calls (함수명 + 인수)
run_tool() 실행
결과를 messages에 추가  ──→  LLM
                       ←──  최종 텍스트 답변
```

| 역할 | 담당 |
| --- | --- |
| 어떤 도구를 어떤 인수로 부를지 | LLM (Azure OpenAI) |
| 실제로 도구를 실행하는 코드 | Python (run_tool 함수) |
| 도구 목록 정의 · 권한 설계 | 개발자 |

---

## 정리

| 확인 내용 | 결과 |
| --- | --- |
| Tool Calling으로 kubectl 실행 | ✅ |
| ReAct 멀티 스텝 루프 동작 | ✅ 확인 |
| 쓰기 도구 없으면 쓰기 불가 | ✅ 확인 |

!!! success "✅ Phase 5 완료 체크리스트"
    - [ ] `basic_chat.py` 실행 → Azure OpenAI 연결 확인
    - [ ] `log_agent.py` 실행 → Tool call 로그 확인
    - [ ] 멀티 스텝 질문 → Agent가 2회 이상 도구 호출
    - [ ] 쓰기 요청 → Agent가 거부(도구 없음)

---

## 다음 단계

[:material-arrow-left: Phase 5-2 · Azure OpenAI 기본 호출](aoai-basics.md){ .md-button }
[Phase 6 · 운영팀의 새 멤버 :material-arrow-right:](../phase-6/index.md){ .md-button .md-button--primary }
