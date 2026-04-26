# Phase 2-2 · Circuit Breaker 적용

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"Retry가 역효과라면 뭘 써야 하냐고요? 누전 차단기 아세요? 과전류가 감지되면 아예 전원을 끊어버리는 거요. Circuit Breaker가 딱 그겁니다."*

---

## 개념 — 차단기의 3가지 상태

Circuit Breaker는 **실패가 임계값을 넘으면 요청 자체를 차단**합니다.
느린 서비스를 계속 호출하는 대신, 즉시 에러를 반환합니다.

```
[Closed] 정상 상태 — 요청을 그대로 통과시킴
    │
    │  연속 5회 실패 (임계값 초과)
    ▼
[Open] 차단 상태 — 요청을 즉시 차단, 빠른 실패(Fast Fail) 반환
    │
    │  10초 경과 (reset_timeout)
    ▼
[Half-Open] 테스트 상태 — 요청 1개만 통과시켜봄
    │                     성공 → Closed 복귀
    │                     실패 → 다시 Open
    └─────────────────────────────────────────
```

!!! note "Fast Fail이 왜 좋은가요?"
    payment-api가 응답 지연 상태일 때:

    - **Retry만 있을 때**: 요청마다 2초 대기 × 3회 = 6초 후 504 에러
    - **Circuit Breaker Open 상태**: 즉시 (0ms) Fallback 응답 반환

    사용자가 6초 기다리는 것보다 즉시 "잠시 후 다시 시도"를 받는 게 낫습니다.
    그리고 payment-api에도 부하가 가지 않으므로 **스스로 복구될 시간**을 줄 수 있습니다.

---

## Step 1. 현재 상태 확인

Phase 2-1에서 `FAULT_DELAY_MS=3000`으로 설정한 상태입니다.
(설정이 바뀌었다면 다시 맞춰주세요.)

```bash title="터미널"
curl -s http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
```

```json title="출력 예시 — Circuit Breaker 동작 전"
{"detail": {"error": "PAYMENT_API_TIMEOUT", "order_id": "ORD-001"}}
```

매 요청마다 2초씩 기다린 후 504 에러가 옵니다. (× 최대 3회 재시도 = 6초)

---

## Step 2. Circuit Breaker Open 상태 만들기

연속으로 요청을 보냅니다. Circuit Breaker는 **5회 연속 실패** 시 Open됩니다.

```bash title="터미널"
for i in {1..10}; do
  echo -n "요청 $i: "
  curl -s -o /dev/null -w "%{http_code} (%{time_total}s)\n" \
    http://localhost:8082/api/orders/ORD-001
done
```

```text title="출력 예시"
요청 1: 504 (6.03s)   ← Timeout 후 504 (3회 재시도 포함)
요청 2: 504 (6.02s)
요청 3: 504 (6.01s)
요청 4: 504 (6.03s)
요청 5: 504 (6.02s)   ← 5회 실패 → Circuit Breaker Open!
요청 6: 200 (0.00s)   ← 즉시 응답! (Fallback)
요청 7: 200 (0.00s)
요청 8: 200 (0.00s)
요청 9: 200 (0.00s)
요청 10: 200 (0.00s)
```

6번째 요청부터 응답 시간이 **0초**로 바뀌었습니다. payment-api를 아예 호출하지 않고 즉시 Fallback을 반환하는 것입니다.

Fallback 응답 내용을 확인합니다.

```bash title="터미널"
curl http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
```

```json title="출력 예시 — Circuit Breaker Open 상태 Fallback 응답"
{
  "order_id": "ORD-001",
  "product_name": "유기농 쌀 10kg",
  "amount": 32000,
  "status": "배송완료",
  "payment": {
    "status": "조회불가",
    "message": "결제 서비스 일시 중단 — 잠시 후 다시 시도해주세요"
  },
  "total_response_time_ms": 1
}
```

!!! success "✅ 확인 포인트"
    - 5회 실패 후 6번째 요청부터 즉시(0~1ms) 응답이 나온 것을 확인했다
    - 결제 정보가 없어도 주문 정보는 보인다 — **부분 기능으로 서비스가 살아남음**

payment-api 로그도 확인합니다.

```bash title="터미널"
sudo docker compose logs payment-api | tail -20
```

```text title="출력 예시"
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 3초 후 응답
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
(여기서 멈춤 — Circuit Breaker가 이후 요청을 차단했기 때문)
```

!!! quote "박대리"
    *"6번째부터는 payment-api를 아예 안 부르네요. 숨통을 조이는 대신 '지금 안 된다'고 바로 말해주는 거네요."*

---

## Step 3. 자동 복구 확인 (Half-Open)

10초가 지나면 Circuit Breaker가 Half-Open 상태로 전환됩니다.
payment-api를 정상으로 복구한 후 자동 복구되는지 확인합니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "0"      # 정상 복구
  FAULT_ERROR_RATE: "0.0"
```

```bash title="터미널 (새 창)"
sudo docker compose up --build payment-api -d
```

10초를 기다린 후 요청을 보냅니다.

```bash title="터미널"
sleep 11
curl http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
```

```json title="출력 예시 — 복구 후"
{
  "order_id": "ORD-001",
  "product_name": "유기농 쌀 10kg",
  "payment": {
    "order_id": "ORD-001",
    "status": "승인완료",
    "response_time_ms": 0
  },
  "total_response_time_ms": 5
}
```

!!! success "✅ 확인 포인트"
    - 10초 후 Half-Open 상태에서 테스트 요청 1개 통과
    - payment-api 응답 정상 → Circuit Breaker가 Closed로 복귀
    - 이후 요청도 정상 응답 (`payment.status: "승인완료"`)

---

## Step 4. Phase 1과 비교

지난 목요일 상황을 재현해봅니다. `FAULT_DELAY_MS=8000`으로 설정합니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "8000"
  FAULT_ERROR_RATE: "0.0"
```

```bash title="터미널 (새 창)"
sudo docker compose up --build payment-api -d
```

```bash title="터미널"
hey -c 10 -n 30 http://localhost:8082/api/orders/ORD-001
```

```text title="출력 예시 — Phase 2 Circuit Breaker 적용 후"
Status code distribution:
  [504] 5 responses    ← Circuit Breaker가 Open되기 전 5개만 실패
  [200] 25 responses   ← 나머지는 Fallback으로 즉시 성공
```

Phase 1에서는 **60개 전부 8초씩** 걸렸습니다.
Phase 2에서는 **5개만 느리고, 25개는 즉시** Fallback 응답을 받습니다.

!!! quote "김팀장"
    *"지난 목요일에 이게 있었다면 44분이 아니라 5초 안에 Fast Fail로 전환됐을 거야."*

---

## Step 5. 원상복구

실습이 끝났으면 장애 설정을 되돌리고, 브랜치도 main으로 돌아갑니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "0"
  FAULT_ERROR_RATE: "0.0"
```

```bash title="터미널 (새 창)"
sudo docker compose up --build payment-api -d
```

```bash title="터미널"
git fetch origin
git checkout main
```

---

## 정리

Phase 2에서 경험한 것:

| 패턴 | 효과 | 한계 |
| --- | --- | --- |
| **Timeout** | 느린 요청이 무한 대기하지 않음 | 너무 짧으면 정상도 끊김 |
| **Retry** | 일시적 503 흡수 | 지속 장애엔 Retry Storm 유발 |
| **Circuit Breaker** | 지속 장애 시 즉시 Fast Fail, 복구 시간 확보 | 임계값 설정이 까다로움 |

!!! warning "Circuit Breaker도 만능은 아닙니다"
    지금 실습에서는 order-api가 1개입니다.
    실제 서비스에서는 order-api 인스턴스가 여러 개이고, 각각 독립적으로 Circuit Breaker 상태를 가집니다.
    인스턴스 A는 Open인데 B는 Closed면 일부 요청은 여전히 느린 서비스로 흘러갑니다.
    분산 환경에서는 Redis 같은 외부 상태 저장소로 CB 상태를 공유해야 합니다.

!!! success "✅ 확인 포인트"
    - 5회 연속 실패 후 Circuit Breaker가 Open되어 즉시 Fallback 응답이 나온 것을 확인했다
    - 10초 후 Half-Open → 복구 후 Closed로 돌아온 것을 확인했다
    - Phase 1과 비교해 장애 영향 범위가 크게 줄어든 것을 확인했다

---

## 다음 단계

[:material-arrow-left: Phase 2-1 · Retry의 효과와 역효과](retry-and-storm.md){ .md-button }
[Phase 3 · 사람 손 떼기 (1부) :material-arrow-right:](../phase-3/index.md){ .md-button .md-button--primary }
