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
curl http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
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

주문 조회 응답에는 결제 정보(`payment`)도 포함됩니다. 그런데 order-api는 결제 정보를 갖고 있지 않습니다. 결제는 **payment-api가 관리**합니다.

그래서 order-api는 응답을 만들기 위해 payment-api에게 물어봐야 합니다:

```python title="order-api/app.py"
    # ② payment-api에게 결제 정보 요청
    async with httpx.AsyncClient(timeout=None) as client:   # ⚠️ 여기!
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        payment = resp.json()

    return {**order, "payment": payment}   # ③ 주문 + 결제 합쳐서 응답
```

② 단계에서 order-api는 payment-api로 HTTP 요청을 날리고, **응답이 올 때까지 기다립니다.**

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

!!! note "실제 장애에서 PG사는 왜 느려졌나?"
    외부 PG사(결제 대행사) 서버에 문제가 생겼거나 네트워크가 막혔을 때
    payment-api는 PG사의 응답을 기다리느라 자신도 느려집니다.
    `FAULT_DELAY_MS` 는 그 상황을 실습에서 인위적으로 만드는 장치입니다.

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
