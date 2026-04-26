# Phase 5-2 · Azure OpenAI 기본 호출

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"일단 LLM 호출부터 해봐. 환경변수 세팅하고 한 줄 보내는 것부터."*

---

## Step 1. Azure AI Foundry 프로젝트 생성

[Azure AI Foundry](https://ai.azure.com)에 접속합니다.

1. **새 프로젝트 만들기** 클릭
2. 프로젝트 이름 입력 (예: `hanbat-aoai-<이니셜>`)
3. **고급 옵션** 선택:

    | 항목 | 값 |
    | --- | --- |
    | 구독 | 본인 구독 |
    | 리소스 그룹 | `hanbat-s2-lab-rg` |
    | 지역 | **East US** 또는 **Sweden Central** |

    !!! warning "지역 선택 주의"
        Korea Central은 gpt-4o-mini를 지원하지 않습니다. **East US** 또는 **Sweden Central**을 선택하세요.

4. **프로젝트 만들기** 클릭 → 완료까지 대기

---

## Step 2. gpt-4o-mini 모델 배포

프로젝트 생성 완료 후:

1. 왼쪽 메뉴 → **모델 + 엔드포인트** → **모델 배포** → **+ 모델 배포**
2. 모델 검색: **gpt-4o-mini** 선택
3. 배포 이름: `gpt-4o-mini` (그대로 유지)
4. **배포** 클릭 → 상태가 **성공**으로 바뀔 때까지 대기

---

## Step 3. 엔드포인트와 API 키 확인

왼쪽 메뉴 → **프로젝트 설정** → **연결된 리소스** → Azure OpenAI 리소스 클릭

또는 Azure Portal → 생성된 Azure OpenAI 리소스 → **키 및 엔드포인트**

| 항목 | 위치 |
| --- | --- |
| 엔드포인트 | `https://<이름>.openai.azure.com/` |
| API 키 | 키 1 또는 키 2 |

!!! success "✅ 확인 포인트"
    엔드포인트와 API 키 두 값을 메모해둡니다.

!!! note "참고 문서"
    [Microsoft Learn — Azure AI Foundry 빠른 시작](https://learn.microsoft.com/azure/ai-foundry/how-to/deploy-models-openai)

---

## Step 4. 환경변수 설정

Step 3에서 확인한 값으로 환경변수를 설정합니다.

```bash title="터미널"
export AZURE_OPENAI_ENDPOINT="https://<본인_엔드포인트>.openai.azure.com/"
export AZURE_OPENAI_API_KEY="<본인_API_키>"
export AZURE_OPENAI_DEPLOYMENT="gpt-4o-mini"
```

재접속해도 유지하려면 `.bashrc`에 추가합니다.

```bash title="터미널"
cat >> ~/.bashrc << 'EOF'
export AZURE_OPENAI_ENDPOINT="https://<본인_엔드포인트>.openai.azure.com/"
export AZURE_OPENAI_API_KEY="<본인_API_키>"
export AZURE_OPENAI_DEPLOYMENT="gpt-4o-mini"
EOF
source ~/.bashrc
```

---

## Step 5. 라이브러리 설치

```bash title="터미널"
cd ~/hanbat-order-app-s2/phase5/skeleton
pip install -r requirements.txt
```

---

## Step 6. basic_chat.py 실행

스켈레톤 파일이 이미 준비되어 있습니다.

```bash title="터미널"
python3 basic_chat.py
```

```text title="출력 예시"
=== Azure OpenAI 기본 연결 테스트 ===
안녕하세요! 저는 한밭푸드 운영팀을 돕는 AI 어시스턴트입니다.
현재 시스템 모니터링, 장애 분석, 운영 자동화 등의 업무를 지원합니다.
무엇을 도와드릴까요?
```

!!! success "✅ 확인 포인트"
    응답이 출력되면 Azure OpenAI 연결 성공입니다.

---

## Step 7. 코드 읽기

`basic_chat.py` 코드를 열어 구조를 살펴봅니다.

```bash title="터미널"
cat basic_chat.py
```

핵심 구조:

```python title="basic_chat.py (핵심 부분)"
response = client.chat.completions.create(
    model=deployment,
    messages=[
        {"role": "system", "content": "당신은 한밭푸드 운영팀을 돕는 AI 어시스턴트입니다."},
        {"role": "user",   "content": user_message},
    ],
)
return response.choices[0].message.content
```

| 요소 | 역할 |
| --- | --- |
| `role: system` | LLM의 성격·역할을 설정 (프롬프트 엔지니어링) |
| `role: user` | 사용자 질문 |
| `role: assistant` | LLM의 이전 응답 (대화 이력 유지 시 추가) |

---

## Step 8. LLM의 한계 체험

LLM이 클러스터 상태를 모른다는 걸 직접 확인합니다.
`basic_chat.py`를 열어 질문을 바꿔봅니다.

```python title="basic_chat.py (마지막 줄 수정)"
answer = chat("지금 order-api 파드 상태가 어때? 에러 있어?")
print(answer)
```

```bash title="터미널"
python3 basic_chat.py
```

```text title="출력 예시"
죄송합니다. 저는 실시간 쿠버네티스 클러스터 정보에 접근할 수 없어서
현재 order-api 파드의 상태를 직접 확인할 수 없습니다.

실제 상태를 확인하려면 다음 명령어를 사용하세요:
kubectl get pods -n <네임스페이스>
kubectl logs deployment/order-api -n <네임스페이스>
```

LLM은 **명령어를 안내**는 하지만 **실제로 실행하지는 못합니다**.

!!! quote "박대리"
    *"그럼 실제로 kubectl 실행하게 하면 되는 거잖아요?"*

!!! quote "이대리"
    *"그게 Tool Calling이야. 다음 단계에서 해."*

---

## Step 9. 파라미터 실험 (선택)

`temperature`를 바꾸면 응답의 창의성이 달라집니다.

```python title="파라미터 비교"
# 결정론적 응답 (temperature=0)
response = client.chat.completions.create(
    model=deployment,
    messages=[...],
    temperature=0,   # 항상 같은 답
)

# 창의적 응답 (temperature=1)
response = client.chat.completions.create(
    model=deployment,
    messages=[...],
    temperature=1,   # 매번 조금씩 다른 답
)
```

| 파라미터 | 값 범위 | 효과 |
| --- | --- | --- |
| `temperature` | 0.0 ~ 2.0 | 낮을수록 결정론적, 높을수록 창의적 |
| `max_tokens` | 1 ~ 모델 한계 | 응답 최대 길이 제한 |

운영 자동화 용도에서는 **temperature=0** 을 권장합니다. 동일 입력에 동일 출력이 나와야 신뢰할 수 있습니다.

---

## 정리

| 확인 내용 | 결과 |
| --- | --- |
| Azure OpenAI 연결 | ✅ |
| LLM은 클러스터 상태를 모름 | ✅ 확인 |
| Tool이 필요한 이유 | ✅ 이해 |

!!! success "✅ 확인 포인트"
    - `basic_chat.py` 실행 시 응답이 한국어로 출력됐다
    - "파드 상태 알려줘" 질문에 LLM이 "모른다"고 답한 것을 확인했다

---

## 다음 단계

[:material-arrow-left: Phase 5-1 · LLM vs Agent](llm-vs-agent.md){ .md-button }
[Phase 5-3 · Tool Calling으로 Agent 만들기 :material-arrow-right:](tool-calling.md){ .md-button .md-button--primary }
