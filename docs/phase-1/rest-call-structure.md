# Phase 1-1 · 요청이 어떻게 흘러가는가

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"박대리, 지난 목요일 장애 기억하지? 결제 API가 느려졌는데 왜 주문 조회까지 죽었는지 — 그게 코드에 다 나와 있어. 같이 따라가 보자."*

---

## 준비 — Docker 설치

앱을 로컬에서 실행하려면 Docker가 필요합니다. 아직 설치 안 됐으면 아래 명령을 실행하세요.

```bash title="터미널"
sudo snap install docker
sudo addgroup --system docker 2>/dev/null; sudo adduser $USER docker
```

!!! warning "`newgrp docker` 는 실행하지 마세요"
    Ubuntu 24.04에서 `/bin` 심볼릭 링크가 삭제되어 SSH 접속이 불가능해지는 버그가 있습니다.
    그룹 적용은 **SSH 재접속**으로 해주세요.

터미널을 닫고 다시 SSH 접속합니다.

```bash title="터미널 (로컬 PC)"
ssh labuser@<공용_IP_주소>
```

재접속 후 설치 확인:

```bash title="터미널"
docker --version
docker compose version
```

!!! success "✅ 확인 포인트"
    두 명령 모두 버전이 출력되면 OK.

---

## Step 1. 앱을 먼저 띄우고, 정상 응답을 눈으로 확인한다

이론보다 먼저 직접 실행해봅니다. 뭔가 나오는 걸 보고 나서 코드를 읽으면 훨씬 빨리 이해됩니다.

레포로 이동합니다:

```bash title="터미널"
cd ~/hanbat-order-app-s2
```

앱을 실행합니다:

```bash title="터미널"
sudo docker compose up --build
```

처음 실행 시 이미지 빌드로 2~3분 걸립니다. 아래처럼 나오면 준비 완료입니다.

```text title="출력 예시"
payment-api  | INFO:     Application startup complete.
order-api    | INFO:     Application startup complete.
```

**새 터미널을 열고** 주문을 조회해봅니다:

```bash title="터미널 (새 창)"
curl http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
```

```json title="출력 예시"
{
  "order_id": "ORD-001",
  "product_name": "유기농 쌀 10kg",
  "amount": 32000,
  "payment": {
    "order_id": "ORD-001",
    "status": "승인완료",
    "method": "신용카드",
    "response_time_ms": 0
  },
  "total_response_time_ms": 12
}
```

!!! success "✅ 확인 포인트"
    `payment` 데이터가 포함되고 `total_response_time_ms` 가 50ms 이하이면 정상입니다.

이 응답 하나를 만들어내기 위해 **내부에서 무슨 일이 일어났는지** 지금부터 따라가 봅니다.

---

## Step 2. 출발점 — 사용자가 주문 내역을 조회한다

고객이 "내 주문 어떻게 됐지?" 하고 앱에서 주문 내역 화면을 열었습니다.
브라우저는 이 요청을 order-api에 보냅니다:

```text
GET /api/orders/ORD-001
```

order-api가 이 요청을 받는 코드를 직접 확인해봅니다:

```bash title="터미널"
cat order-api/app.py
```

코드 중에서 이 함수를 찾아보세요:

```python title="order-api/app.py"
@app.get("/api/orders/{order_id}")      # ← "GET /api/orders/ORD-001" 요청이 여기로 옴
async def get_order(order_id: str):

    order = ORDERS.get(order_id)        # ① 주문 정보 꺼내기 (내부 데이터)
    ...
```

① 단계까지는 간단합니다. order-api 안에 주문 데이터가 있고, 바로 꺼낼 수 있습니다.
문제는 다음입니다.

---

## Step 3. order-api가 혼자 처리할 수 없는 것 — 결제 정보

Step 1에서 받은 응답을 다시 보면, 주문 정보 안에 결제 정보(`payment`)가 같이 들어있습니다.

```json
{
  "order_id": "ORD-001",
  "product_name": "유기농 쌀 10kg",   ← 주문 정보
  "payment": {
    "status": "승인완료",              ← 결제 정보
    "method": "신용카드"
  }
}
```

그런데 order-api는 결제 정보를 갖고 있지 않습니다. 결제 정보는 **payment-api가 따로 관리**합니다. 그래서 order-api는 응답을 완성하려면 payment-api에게 직접 물어봐야 합니다.

코드에서 그 부분을 찾아보면:

```python title="order-api/app.py"
    # 이렇게 payment-api에게 HTTP 요청을 보냅니다
    async with httpx.AsyncClient(timeout=None) as client:
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        payment = resp.json()
```

한 줄씩 읽으면:

```text
httpx.AsyncClient(timeout=None)
  → HTTP 요청을 보내는 도구를 준비한다
  → timeout=None : 응답이 올 때까지 무한정 기다린다 ← ⚠️ 문제의 설정

client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
  → payment-api에게 "ORD-001의 결제 정보 줘" 라고 요청한다
  → 이 줄에서 order-api는 payment-api의 응답을 기다리며 멈춥니다

resp.json()
  → 돌아온 응답을 JSON으로 변환한다
```

payment-api의 응답이 오면, 주문 정보와 결제 정보를 합쳐서 최종 응답을 만듭니다:

```python title="order-api/app.py"
    return {**order, "payment": payment}
    # {**order} → 주문 정보를 펼쳐놓고
    # "payment": payment → 결제 정보를 옆에 붙인다
    # 결과: 주문 + 결제가 합쳐진 JSON 하나
```

정리하면 order-api는 이런 순서로 동작합니다:

```text
① 내부에서 주문 정보 꺼내기          (빠름, 0ms)
② payment-api에게 결제 정보 요청     (느릴 수 있음, 응답 올 때까지 대기)
③ 주문 + 결제 합쳐서 응답 반환
```

② 단계에서 order-api는 payment-api의 응답을 기다리며 **완전히 멈춥니다.** `timeout=None` 이니까 1초든 1분이든 상관없이 기다립니다.

그런데 `timeout=None` 이 붙어 있습니다.

!!! danger "timeout=None — 이게 지난 목요일 장애의 직접 원인"
    `timeout=None` 은 **"응답이 올 때까지 무한정 기다린다"** 는 뜻입니다.

    웹서버는 요청 하나를 처리하기 위해 스레드(일꾼) 하나를 씁니다.
    payment-api가 8초 걸리면, 그 스레드는 8초 동안 payment-api만 기다립니다.

    ```text
    요청 1 → 스레드 1이 payment-api 기다리는 중... (8초)
    요청 2 → 스레드 2가 payment-api 기다리는 중... (8초)
    요청 3 → 스레드 3이 payment-api 기다리는 중... (8초)
    ...
    요청 N → 스레드 없음 → 503 에러
    ```

    스레드가 전부 묶이면 order-api는 새 요청을 처리할 수 없습니다.
    **결제 API가 느린 것인데 주문 API가 죽는 이유가 바로 이겁니다.**

!!! note "이건 Python만의 문제가 아닙니다 — HTTP 호출이 있는 곳이라면 어디든"
    `timeout=None` 은 Python 문법이지만, **이 개념은 언어에 상관없이 동일합니다.**
    대부분의 HTTP 클라이언트는 timeout을 명시하지 않으면 기본값이 "무한 대기"입니다.

    ```text
    Python  requests.get(url)                  → 기본값: 무한 대기
    Python  httpx.AsyncClient(timeout=None)    → 명시적 무한 대기
    Java    restTemplate.getForObject(url)     → 기본값: 무한 대기
    Node.js axios.get(url)                     → 기본값: 무한 대기
    Go      http.Get(url)                      → 기본값: 무한 대기
    ```

    언어가 바뀌어도, 외부 API를 호출하면서 timeout을 설정하지 않으면
    **똑같은 장애가 똑같은 이유로 발생합니다.**

!!! tip "개발자만 알면 되는 건 아닙니다"
    | 역할 | 알아야 하는 이유 |
    |------|----------------|
    | **개발자** | timeout 없이 코드를 짜면 장애를 만드는 사람이 됨 |
    | **운영 엔지니어** | 모니터링에서 이상 징후를 보고 "timeout 문제다" 진단할 수 있어야 함 |

    운영 엔지니어라면 코드를 보지 않아도 이런 패턴이 보일 때 timeout 문제를 의심해야 합니다:

    ```text
    ✔ 특정 API 응답시간이 갑자기 10배 이상 증가
    ✔ 에러율은 낮은데 대기 중인 요청(pending)이 급증
    ✔ 연결된 다른 서비스까지 연쇄적으로 느려짐
    ```

    **"왜 결제 API가 문제인데 주문 API가 죽었냐"** — 이 질문에 바로 답할 수 있어야
    장애 상황에서 팀이 원인을 빠르게 찾을 수 있습니다.

---

## Step 4. payment-api 안에서 — 지연은 어디서 오는가

이제 payment-api 쪽을 봅니다:

```bash title="터미널"
cat payment-api/app.py
```

```python title="payment-api/app.py"
FAULT_DELAY_MS: int = int(os.getenv("FAULT_DELAY_MS", "0"))   # 환경변수로 지연 설정

@app.get("/api/payments/{order_id}")
async def get_payment(order_id: str):

    if FAULT_DELAY_MS > 0:
        await asyncio.sleep(FAULT_DELAY_MS / 1000)   # ← 이 시간만큼 일부러 늦게 응답

    return PAYMENTS.get(order_id)
```

평소엔 `FAULT_DELAY_MS=0` 이라 즉시 응답합니다.
`FAULT_DELAY_MS=8000` 으로 바꾸면 매 요청마다 8초씩 늦게 응답합니다.

이게 **지난 목요일 외부 PG사 응답 지연 상황을 그대로 재현하는 스위치**입니다.

!!! note "Fault Injection — 장애를 인위적으로 주입한다"
    실제 장애는 PG사 서버가 진짜로 문제가 생겨야 재현됩니다.
    실습 환경에서 그걸 기다릴 수 없으니, payment-api 코드 안에 **장애 주입 스위치**를 미리 만들어둔 겁니다.

    | 환경변수 | 실제 장애 상황 | 실습에서 재현하는 방식 |
    | --- | --- | --- |
    | `FAULT_DELAY_MS=8000` | PG사 서버 응답이 8초씩 걸림 | payment-api가 일부러 8초 늦게 응답 |
    | `FAULT_ERROR_RATE=0.7` | PG 게이트웨이가 간헐적으로 503 반환 | payment-api가 70% 확률로 503 반환 |

    이 방식을 **Fault Injection(장애 주입)** 이라고 부릅니다.
    Netflix는 프로덕션 서버를 일부러 랜덤으로 죽이는 "Chaos Monkey"로 유명한데,
    이 실습의 환경변수가 그 개념을 단순화한 겁니다.
    Phase 2에서 `FAULT_ERROR_RATE` 도 같이 사용합니다.

---

## Step 5. 전체 흐름 한 눈에

지금까지 따라온 흐름을 정리하면 이렇습니다:

```text title="정상 상태 흐름"
① 고객이 주문 내역 화면 오픈
      │  GET /api/orders/ORD-001
      ▼
② order-api: 주문 정보는 내가 갖고 있어
      내부 데이터에서 주문 정보 꺼냄 (빠름)
      │
      │  "결제 정보도 필요한데, payment-api한테 물어봐야 해"
      │  GET /api/payments/ORD-001
      │  timeout=None → 응답 올 때까지 이 스레드는 대기
      ▼
③ payment-api: 결제 정보 찾아서 응답
      (정상 시: 0~2ms, FAULT_DELAY_MS=8000 시: 8초 후)
      │
      ▼
④ order-api: 주문 + 결제 합쳐서 고객에게 응답
```

!!! warning "지금은 정상. 다음 실습에서 고장낸다"
    지금 `FAULT_DELAY_MS=0` 이라서 전체 응답이 10~30ms입니다.
    다음 실습에서 `FAULT_DELAY_MS=8000` 을 넣으면 스레드가 묶이기 시작하고
    부하가 쌓이면서 order-api 전체가 먹통이 됩니다.
    지난 목요일이 어떻게 44분 장애가 됐는지 직접 체험합니다.

---

## 핵심 정리

| | 내용 |
|---|---|
| order-api 혼자 못 하는 것 | 결제 정보 — payment-api에 물어봐야 함 |
| 요청 방식 | HTTP 호출 — 응답 올 때까지 스레드가 대기 |
| 문제의 코드 한 줄 | `httpx.AsyncClient(timeout=None)` |
| 장애 재현 스위치 | `FAULT_DELAY_MS` 환경변수 |

!!! quote "이대리"
    *"코드 딱 한 줄이야. timeout 하나 없어서 payment-api가 버티는 동안 order-api 스레드 전부가 같이 버티는 거야. 다음엔 직접 만들어보자."*

---

## 다음 단계

[:material-arrow-left: Phase 1 개요](index.md){ .md-button }
[Phase 1-2 · 장애 전파 체험 :material-arrow-right:](cascade-failure.md){ .md-button .md-button--primary }
