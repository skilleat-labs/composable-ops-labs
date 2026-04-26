# Phase 1-1 · 코드로 보는 호출 구조

!!! info "예상 소요 1시간"

---

!!! quote "이대리"
    *"박대리, 이거 코드 한번 같이 봐. 왜 결제 API가 느려지면 주문 조회까지 죽는지 코드에 다 나와있어."*

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

## Step 1. 코드 구조 파악

레포로 이동합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
```

디렉터리 구조를 봅니다.

```bash title="터미널"
ls -1
```

```text title="출력 예시"
order-api/      ← 주문 조회 API (박대리가 담당)
payment-api/    ← 결제 API (외부 PG사와 통신)
order-web/      ← 주문 조회 화면
k8s/            ← Kubernetes 매니페스트
docker-compose.yml
```

---

## Step 2. order-api 코드 읽기

`order-api/app.py`를 열어봅니다.

```bash title="터미널"
cat order-api/app.py
```

전체 흐름에서 핵심은 이 부분입니다.

```python title="order-api/app.py (핵심 부분)"
@app.get("/api/orders/{order_id}")
async def get_order(order_id: str):

    order = ORDERS.get(order_id)  # ① 주문 데이터 조회

    # ② payment-api 호출 ← 여기가 핵심
    async with httpx.AsyncClient(timeout=None) as client:  # ⚠️ timeout=None !
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        payment = resp.json()

    return {**order, "payment": payment}  # ③ 합쳐서 응답
```

!!! warning "timeout=None 이 뭔가요?"
    `timeout=None` 은 **응답을 무한정 기다린다**는 뜻입니다.
    payment-api가 10초, 1분이 걸려도 order-api는 그냥 기다립니다.
    이게 지난 목요일 장애의 직접적인 원인입니다.

---

## Step 3. payment-api 코드 읽기

```bash title="터미널"
cat payment-api/app.py
```

이 부분을 찾아보세요.

```python title="payment-api/app.py (핵심 부분)"
FAULT_DELAY_MS: int = int(os.getenv("FAULT_DELAY_MS", "0"))

@app.get("/api/payments/{order_id}")
async def get_payment(order_id: str):

    # 환경변수로 지연 주입 가능
    if FAULT_DELAY_MS > 0:
        await asyncio.sleep(FAULT_DELAY_MS / 1000)

    return PAYMENTS.get(order_id)
```

!!! note "FAULT_DELAY_MS 란?"
    **장애를 인위적으로 만드는 스위치**입니다.
    `FAULT_DELAY_MS=8000` 으로 설정하면 payment-api가 모든 요청에 8초씩 늦게 응답합니다.
    실제 PG사 응답 지연 상황을 그대로 재현할 수 있습니다.

---

## Step 4. 호출 흐름 그리기

코드에서 파악한 흐름을 정리하면 이렇습니다.

```
사용자 브라우저
    │  GET /api/orders/ORD-001
    ▼
order-api (8082 포트)
    │  GET /api/payments/ORD-001
    │  timeout=None → 응답 올 때까지 무한 대기
    ▼
payment-api (8081 포트)
    │  FAULT_DELAY_MS 만큼 대기
    ▼
  응답 반환
```

---

## Step 5. 앱 실행 — 정상 상태 확인

docker compose로 두 API를 한 번에 실행합니다.

```bash title="터미널"
sudo docker compose up --build
```

처음 실행 시 이미지 빌드로 2~3분 걸립니다. 아래처럼 나오면 준비 완료입니다.

```text title="출력 예시"
payment-api  | INFO:     Application startup complete.
order-api    | INFO:     Application startup complete.
```

새 터미널을 열고 정상 동작을 확인합니다.

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
    "amount": 32000,
    "method": "신용카드",
    "status": "승인완료",
    "pg_txn_id": "PG-20250401-001",
    "response_time_ms": 0        // payment-api 내부 처리 시간 — 정상 시 0~2ms
  },
  "total_response_time_ms": 12   // 환경마다 다름 (5~30ms 정상)
}
```

!!! note "response_time_ms: 0 이 맞나요?"
    네, 정상입니다. `response_time_ms` 는 payment-api **내부** 처리 시간입니다.
    `FAULT_DELAY_MS=0` 일 때 처리가 너무 빠르면 반올림하여 0ms로 표시됩니다.
    `total_response_time_ms` 는 order-api 기준 전체 시간으로 네트워크 구간이 포함되어 수ms가 나옵니다.

!!! success "✅ 확인 포인트"
    `total_response_time_ms` 가 50ms 이하이고 `payment` 데이터가 포함되면 정상입니다.

---

## Step 6. 정리

| 항목 | 내용 |
| --- | --- |
| order-api 가 하는 일 | 주문 데이터 + 결제 데이터를 합쳐서 응답 |
| payment-api 를 부르는 방식 | **동기 HTTP 호출** (응답 올 때까지 대기) |
| 문제의 코드 | `httpx.AsyncClient(timeout=None)` |
| 장애 재현 스위치 | `FAULT_DELAY_MS` 환경변수 |

!!! quote "이대리"
    *"봤지? timeout 하나 없어서 payment-api가 버티는 동안 order-api도 같이 버티는 거야. 다음 실습에서 직접 장애 만들어볼게."*

---

## 다음 단계

[:material-arrow-left: Phase 1 개요](index.md){ .md-button }
[Phase 1-2 · 장애 전파 체험 :material-arrow-right:](cascade-failure.md){ .md-button .md-button--primary }
