# Phase 2-1 · Retry의 효과와 역효과

!!! info "예상 소요 30분"

---

!!! quote "박대리"
    *"간단하네요. 그냥 **재시도(Retry)** 한 번 넣으면 되는 거 아니에요? 일시적으로 느려진 거니까 두 번 시도하면 되잖아요."*

!!! quote "이대리"
    *"한 번 해보죠. 저도 3년 전에 똑같은 얘기 했어요."*

---

## 개념 — 언제 Retry가 유효한가

Retry는 **일시적(transient) 장애**에만 효과가 있습니다.

| 장애 유형 | 예시 | Retry 결과 |
| --- | --- | --- |
| **일시적** | 네트워크 순간 끊김, 간헐적 503 | ✅ 재시도하면 성공 가능 |
| **지속적** | 서버 다운, 응답 지연 8초 | ❌ 재시도해도 같은 실패 → 요청만 3배로 증가 |

지난 목요일 사건은 PG사 **응답 지연(지속 장애)**이었습니다.
Retry를 넣었다면 어떻게 됐을까요?

---

## Step 1. 간헐적 에러 주입

먼저 **일시적 에러** 상황을 만들어봅니다.
`docker-compose.yml`에서 `FAULT_ERROR_RATE`를 `0.7`로 바꿉니다.

```yaml title="docker-compose.yml (payment-api 환경변수 수정)"
environment:
  FAULT_DELAY_MS: "0"
  FAULT_ERROR_RATE: "0.7"   # "0.0" → "0.7" (70% 확률로 503 반환)
```

payment-api를 재시작합니다.

```bash title="터미널 (새 창)"
cd ~/hanbat-order-app-s2
sudo docker compose up --build payment-api -d
```

현재 상태(Retry 없음)에서 요청해봅니다.

```bash title="터미널"
for i in {1..5}; do
  curl -s http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
  echo "---"
done
```

```text title="출력 예시 — Retry 없는 상태"
{"detail":{"error":"PG_GATEWAY_UNAVAILABLE", ...}}   ← 실패 (503 그대로 전달)
---
{"order_id":"ORD-001", "payment":{...}}               ← 성공
---
{"detail":{"error":"PG_GATEWAY_UNAVAILABLE", ...}}   ← 실패
---
```

70%는 실패. 일시적 에러인데도 사용자에게 에러가 그대로 노출됩니다.

---

## Step 2. Retry 적용 — phase2 브랜치로 전환

Retry + Timeout + Circuit Breaker가 모두 적용된 코드로 전환합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
git fetch origin
git checkout phase2
```

order-api를 재빌드합니다.

```bash title="터미널 (새 창)"
sudo docker compose up --build order-api -d
```

---

## Step 3. Retry 효과 확인 — 간헐적 503

`FAULT_ERROR_RATE=0.7` 상태에서 다시 요청해봅니다.

```bash title="터미널"
for i in {1..5}; do
  curl -s http://localhost:8082/api/orders/ORD-001 | python3 -m json.tool
  echo "---"
done
```

```text title="출력 예시 — Retry 적용 후"
{"order_id":"ORD-001", "payment":{...}}   ← 성공
---
{"order_id":"ORD-001", "payment":{...}}   ← 성공
---
{"order_id":"ORD-001", "payment":{...}}   ← 성공
---
```

성공률이 크게 올라갔습니다. payment-api 로그를 보면 재시도 흔적이 보입니다.

```bash title="터미널"
sudo docker compose logs payment-api | tail -20
```

```text title="출력 예시"
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 503
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 503
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 3번째 시도에서 성공
```

order-api는 요청 1번을 받았지만 payment-api에는 최대 3번을 호출했습니다.
**Retry가 일시적 503을 흡수하고 있습니다.**

!!! success "✅ 확인 포인트"
    - 5번 요청 중 대부분이 성공으로 바뀌었다
    - payment-api 로그에서 503 → 503 → 200 순서의 재시도 흔적이 보인다

---

## Step 4. Retry Storm — 지속 장애에서는?

이제 **지속 장애**로 바꿔봅니다.
`FAULT_DELAY_MS=3000` (3초 지연) → timeout 2초보다 길어서 **모든 요청이 Timeout**됩니다.

```yaml title="docker-compose.yml (수정)"
environment:
  FAULT_DELAY_MS: "3000"
  FAULT_ERROR_RATE: "0.0"
```

```bash title="터미널 (새 창)"
sudo docker compose up --build payment-api -d
```

부하 테스트를 실행합니다.

```bash title="터미널"
hey -c 10 -n 30 http://localhost:8082/api/orders/ORD-001
```

```text title="출력 예시"
Summary:
  Total:        66.12 secs
  Slowest:       6.05 secs   ← 2초 대기 × 3회 시도 = 6초
  Average:       6.03 secs

Status code distribution:
  [504] 30 responses   ← 전부 504 Timeout
```

payment-api 로그를 확인합니다.

```bash title="터미널"
sudo docker compose logs payment-api | tail -40
```

```text title="출력 예시"
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 3초 후 응답, 이미 timeout
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: ... "GET /api/payments/ORD-001 HTTP/1.1" 200
... (30개 요청 × 최대 3회 재시도 = 최대 90줄)
```

!!! danger "Retry Storm"
    order-api는 30개 요청을 보냈는데, payment-api에는 **최대 90개** 요청이 쏟아집니다.
    이미 힘든 payment-api에 **3배의 부하**를 추가로 얹는 셈입니다.
    이것이 **Retry Storm(재시도 폭주)**입니다.

!!! quote "이대리"
    *"봤죠? 지속 장애에 Retry를 붙이면 장애 서비스 회복은커녕 숨통을 조입니다. 3년 전 저도 이거 보고 처음엔 이해가 안 됐어요."*

---

## 정리

| 상황 | Retry 결과 |
| --- | --- |
| 일시적 503 (FAULT_ERROR_RATE=0.7) | ✅ 재시도로 성공률 향상 |
| 지속 지연 (FAULT_DELAY_MS=3000) | ❌ Retry Storm — 장애 서비스에 3배 부하 |

**Retry의 교훈**: 일시적 장애에만 유효. 지속 장애에는 Circuit Breaker가 필요합니다.

!!! success "✅ 확인 포인트"
    - `FAULT_ERROR_RATE=0.7` 상황에서 Retry 후 성공률이 올라간 것을 확인했다
    - `FAULT_DELAY_MS=3000` 상황에서 payment-api 로그가 최대 3배로 증가한 것을 확인했다

---

## 다음 실습 준비

`FAULT_DELAY_MS=3000` 설정을 유지한 채로 다음 단계로 넘어갑니다.

---

## 다음 단계

[:material-arrow-left: Phase 2 개요](index.md){ .md-button }
[Phase 2-2 · Circuit Breaker 적용 :material-arrow-right:](circuit-breaker.md){ .md-button .md-button--primary }
