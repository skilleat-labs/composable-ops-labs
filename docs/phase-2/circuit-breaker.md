# Phase 2-2 · Circuit Breaker 적용

!!! info "예상 소요 45분"

---

## Phase 2-1에서 Phase 2-2로 — Retry도 답이 아니었다

Phase 2-1에서 우리는 이걸 확인했습니다.

```text
일시적 503 (간헐적 에러)  → Retry가 효과 있음
지속 지연 (8초 응답 지연) → Retry가 오히려 3배 부하를 만들어냄
```

**그럼 지속 장애에는 뭘 써야 할까요?**

!!! quote "박대리"
    *"Retry도 답이 아니면... 그냥 포기하는 건가요? 에러를 그냥 사용자한테 내보내는 거예요?"*

!!! quote "이대리"
    *"포기가 아니라 차단이에요. 누전 차단기 아세요? 과전류가 감지되면 아예 전원을 끊어버리는 거요. Circuit Breaker가 딱 그겁니다."*

---

## 이론 — Circuit Breaker란?

### 현실 세계의 누전 차단기

집에 있는 누전 차단기를 떠올려보세요. 과전류가 감지되면 전체 전원을 차단합니다. 전원을 계속 연결한 채로 감전 사고나 화재가 나도록 두지 않습니다.

Circuit Breaker는 소프트웨어 세계에서 같은 역할을 합니다. payment-api가 계속 실패하면, 호출 자체를 차단해서 사용자에게 즉시 "지금 안 됩니다"를 알려줍니다.

### 3가지 상태

```text
[Closed] 정상 상태
    요청을 payment-api에 그대로 전달합니다.
    실패를 카운트하고 있습니다.
    │
    │  연속 5회 실패 (임계값 초과)
    ▼
[Open] 차단 상태
    payment-api를 호출하지 않습니다.
    즉시 Fallback 응답을 반환합니다. (0ms)
    │
    │  10초 경과 (reset_timeout)
    ▼
[Half-Open] 테스트 상태
    요청 1개만 payment-api에 보내봅니다.
    성공하면 → Closed로 복귀 (정상 운영 재개)
    실패하면 → Open으로 돌아감 (10초 더 차단)
```

### Fast Fail이 왜 좋은가?

payment-api가 지연 상태일 때 두 경우를 비교해보면:

```text
Retry만 있을 때:
  사용자 요청 → 2초 대기 → 실패 → 2초 대기 → 실패 → 2초 대기 → 실패
  → 6초 후 504 에러

Circuit Breaker Open 상태:
  사용자 요청 → 즉시 Fallback 반환 (0ms)
  → "결제 서비스 일시 중단" 메시지
```

사용자가 6초를 기다린 후 에러를 받는 것보다, 즉시 "잠시 후 다시 시도해주세요"를 받는 게 낫습니다. 그리고 payment-api 입장에서도 호출이 차단되니 **스스로 회복할 시간**이 생깁니다.

---

## Step 1. Circuit Breaker 직접 구현하기

이론을 봤으니 코드로 붙입니다. Retry와 마찬가지로 **order-api 코드를 직접 수정**합니다.

!!! note "pybreaker — Circuit Breaker 라이브러리"
    Retry에 tenacity를 쓴 것처럼, Circuit Breaker는 **pybreaker** 라이브러리를 씁니다.
    `requirements.txt`에 이미 `pybreaker==1.2.0`이 포함되어 있으니 별도 설치는 필요 없습니다.

    pybreaker의 핵심은 두 가지입니다.

    **① CircuitBreaker 인스턴스 생성**:
    ```python
    payment_breaker = pybreaker.CircuitBreaker(
        fail_max=5,        # 몇 번 실패하면 Open으로 전환할지
        reset_timeout=10,  # Open 후 몇 초 뒤 Half-Open으로 전환할지
    )
    ```

    **② `call_async`로 함수 감싸기**:
    ```python
    result = await payment_breaker.call_async(some_async_func, arg1, arg2)
    ```
    `payment_breaker.call_async(함수, 인자...)`는 "CB가 허락할 때만 함수를 실행해줘"라는 뜻입니다.
    CB가 Open이면 함수를 실행하지 않고 즉시 `CircuitBreakerError`를 던집니다.

현재 `order-api/app.py`의 상태를 먼저 확인합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
cat order-api/app.py
```

Phase 2-1에서 작성한 `_fetch_payment` 함수(Retry 데코레이터 붙은 것)가 보일 겁니다. 이 함수를 3단계로 수정합니다.

---

### Step 1-1 — `import pybreaker` 추가

파일 맨 위 import 블록을 찾습니다. 이런 모양입니다:

```python title="order-api/app.py — 현재 import 부분"
import httpx
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_exception_type
```

`import httpx` 바로 위에 `import pybreaker`를 한 줄 추가합니다:

```python title="order-api/app.py — 수정 후"
import httpx
import pybreaker
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_exception_type
```

---

### Step 1-2 — `_fetch_payment` 함수 재구성

현재 파일에서 이 부분을 찾습니다:

```python title="order-api/app.py — 현재 _fetch_payment 함수"
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(0.5),
    retry=retry_if_exception_type((httpx.TimeoutException, httpx.HTTPStatusError)),
    reraise=True
)
async def _fetch_payment(order_id: str) -> dict:
    async with httpx.AsyncClient(timeout=2.0) as client:
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        resp.raise_for_status()
        return resp.json()
```

이 함수 **전체를 지우고** 아래 3개 블록으로 교체합니다:

```python title="order-api/app.py — 교체할 내용"
# ---------------------------------------------------------------------------
# Circuit Breaker 설정
# ---------------------------------------------------------------------------
payment_breaker = pybreaker.CircuitBreaker(
    fail_max=5,        # 연속 5회 실패 시 Open (차단)
    reset_timeout=10,  # 10초 후 Half-Open 전환 (회복 테스트)
)

@retry(
    stop=stop_after_attempt(3),       # 최대 3회 시도
    wait=wait_fixed(0.5),             # 재시도 간격 0.5초
    retry=retry_if_exception_type((httpx.TimeoutException, httpx.HTTPStatusError)),
    reraise=True
)
async def _fetch_payment_inner(order_id: str) -> dict:
    async with httpx.AsyncClient(timeout=2.0) as client:
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        resp.raise_for_status()
        return resp.json()

async def _fetch_payment(order_id: str) -> dict:
    return await payment_breaker.call_async(_fetch_payment_inner, order_id)
```

!!! note "왜 함수를 두 개로 나눴나요?"
    CB는 **Retry 바깥**을 감싸야 합니다. 순서가 중요합니다.

    ```text
    요청이 들어오면:

    ① _fetch_payment 호출
        └→ payment_breaker.call_async 실행
             ├─ CB가 Open이면? → CircuitBreakerError 즉시 던짐 (payment-api 호출 안 함)
             └─ CB가 Closed이면? → _fetch_payment_inner 실행
                  └→ @retry 작동 (최대 3회 시도)
                       └→ 실제 HTTP 요청
    ```

    만약 CB가 Retry 안쪽에 있으면, 3회 재시도마다 CB가 실패를 카운트하게 됩니다.
    그러면 1번 요청에 3번 카운트가 쌓여 금방 Open이 되어버립니다.

    CB는 Retry 전체 결과(성공 or 최종 실패)만 보면 됩니다:

    ```text
    Retry 3회 모두 실패 → _fetch_payment_inner가 예외를 던짐
                        → CB가 실패 1회 카운트
    ```

    - `_fetch_payment_inner`: 실제 HTTP 호출 + Retry (내부)
    - `_fetch_payment`: CB 게이트 역할 (외부 — "지금 호출해도 돼?" 판단)

---

### Step 1-3 — Fallback 처리 추가

`get_order` 함수 안을 찾아봅니다. 이런 `try/except` 블록이 있을 겁니다:

```python title="order-api/app.py — 현재 get_order 안 try 블록"
    try:
        payment = await _fetch_payment(order_id)
    except httpx.TimeoutException:
        raise HTTPException(
            status_code=504,
            detail={"error": "PAYMENT_API_TIMEOUT", "order_id": order_id},
        )
    except httpx.HTTPStatusError as e:
        raise HTTPException(
            status_code=502,
            detail={"error": "PAYMENT_API_ERROR", "upstream_status": e.response.status_code},
        )
    except httpx.RequestError as e:
        raise HTTPException(
            status_code=502,
            detail={"error": "PAYMENT_API_UNREACHABLE", "detail": str(e)},
        )
```

`try:` 바로 다음 줄 — `except httpx.TimeoutException:` 앞에 아래 블록을 **끼워 넣습니다**:

```python title="추가할 except 블록 (TimeoutException 위에 삽입)"
    except pybreaker.CircuitBreakerError:
        # CB Open 상태 — payment-api를 아예 호출하지 않고 즉시 Fallback 응답
        payment = {
            "status": "조회불가",
            "message": "결제 서비스 일시 중단 — 잠시 후 다시 시도해주세요",
        }
```

수정 후 전체 블록은 이렇게 됩니다:

```python title="order-api/app.py — 수정 완료 후 try 블록 전체"
    try:
        payment = await _fetch_payment(order_id)
    except pybreaker.CircuitBreakerError:
        # CB Open 상태 — payment-api를 아예 호출하지 않고 즉시 Fallback 응답
        payment = {
            "status": "조회불가",
            "message": "결제 서비스 일시 중단 — 잠시 후 다시 시도해주세요",
        }
    except httpx.TimeoutException:
        raise HTTPException(
            status_code=504,
            detail={"error": "PAYMENT_API_TIMEOUT", "order_id": order_id},
        )
    except httpx.HTTPStatusError as e:
        raise HTTPException(
            status_code=502,
            detail={"error": "PAYMENT_API_ERROR", "upstream_status": e.response.status_code},
        )
    except httpx.RequestError as e:
        raise HTTPException(
            status_code=502,
            detail={"error": "PAYMENT_API_UNREACHABLE", "detail": str(e)},
        )
```

!!! note "CircuitBreakerError일 때 왜 에러를 던지지 않나요?"
    Timeout이나 HTTP 에러는 payment-api를 찌르다가 실패한 겁니다. 뭔가 이상이 생긴 거니 504, 502로 에러를 올려보냅니다.

    CB Open은 다릅니다. "이미 여러 번 실패한 걸 알고 있어서 아예 시도조차 안 한" 상태입니다.
    이 경우에는 주문 정보(이름, 금액, 배송 상태)는 멀쩡히 있습니다. 결제 상태만 지금 당장 못 가져올 뿐입니다.

    그래서 에러(4xx/5xx)를 올리는 대신 **`payment` 변수에 임시 데이터를 넣어두고** 나머지 응답을 정상으로 만드는 겁니다. 고객은 주문 내역은 볼 수 있고, 결제 부분만 "잠시 후 다시 시도" 메시지를 보게 됩니다.

    ```text
    Timeout  → raise HTTPException(504) → 화면 전체가 에러
    CB Open  → payment = {fallback}     → 화면은 뜨고, 결제 부분만 "조회불가"
    ```

    서비스가 완전히 죽지 않고 **부분 기능으로 살아남는** 것 — 이게 Circuit Breaker의 진짜 목적입니다.

---

### Step 1-4 — order-api 재빌드

코드 수정이 끝났으면 이미지를 새로 빌드합니다. **코드를 바꿀 때마다 반드시 빌드**해야 변경이 반영됩니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up --build order-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up --build order-api -d
    ```

빌드 로그에서 아래처럼 나오면 완료입니다:

```text title="출력 예시"
 ✔ Container hanbat-order-app-s2-order-api-1  Started
```

빌드 완료 후 단일 요청으로 정상 동작을 확인합니다. (지금은 `FAULT_DELAY_MS=0` 또는 낮은 값이면 정상 응답이 옵니다.)

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

```json title="출력 예시 — 정상 응답"
{
  "order_id": "ORD-001",
  "payment": {
    "status": "승인완료",
    "method": "신용카드"
  },
  "total_response_time_ms": 12
}
```

!!! success "✅ 확인 포인트"
    정상 응답이 오면 Circuit Breaker 코드가 잘 적용된 겁니다.
    지금 CB는 **Closed(정상)** 상태입니다. 아직 실패가 없으니 차단하지 않고 그냥 통과시킵니다.

이제 `FAULT_DELAY_MS=3000`을 주입해서 CB가 Open으로 전환되는 과정을 직접 봅니다.

```yaml title="docker-compose.yml (payment-api 환경변수 수정)"
environment:
  FAULT_DELAY_MS: "3000"   # payment-api가 3초씩 늦게 응답
  FAULT_ERROR_RATE: "0.0"
```

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

---

## Step 2. 현재 상태 확인 — CB Closed (정상)

`FAULT_DELAY_MS=3000` 상태를 유지한 채로 단일 요청을 보냅니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    $start = Get-Date
    try {
        Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    } catch {
        Write-Host $_.Exception.Message
    }
    Write-Host "소요: $(((Get-Date) - $start).TotalSeconds)초"
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    time curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

```json title="출력 예시"
{"detail": {"error": "PAYMENT_API_TIMEOUT", "order_id": "ORD-001"}}
소요: 6.03초   ← 2초 대기 × 3회 재시도 = 6초
```

매 요청마다 6초씩 걸리고 있습니다. Circuit Breaker가 이 상황을 어떻게 바꾸는지 봅니다.

!!! success "✅ 확인 포인트"
    응답 시간이 6초 이상이면 OK. 지금은 Circuit Breaker가 아직 Closed(정상) 상태입니다.

---

## Step 3. Circuit Breaker Open 상태 만들기

연속으로 요청을 10번 보냅니다. 각 요청의 상태 코드와 응답 시간을 같이 출력합니다.

Circuit Breaker는 **5회 연속 실패** 시 Open으로 전환됩니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    1..10 | ForEach-Object {
        $start = Get-Date
        $code = try {
            (Invoke-WebRequest -Uri http://127.0.0.1:8082/api/orders/ORD-001 -SkipHttpErrorCheck).StatusCode
        } catch { "ERR" }
        $elapsed = [math]::Round(((Get-Date) - $start).TotalSeconds, 2)
        Write-Host "요청 ${_}: $code (${elapsed}s)"
    }
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    for i in {1..10}; do
      echo -n "요청 $i: "
      curl -s -o /dev/null -w "%{http_code} (%{time_total}s)\n" \
        http://127.0.0.1:8082/api/orders/ORD-001
    done
    ```

```text title="출력 예시"
요청 1:  504 (6.03s)   ← Timeout 후 504 (3회 재시도 포함)
요청 2:  504 (6.02s)
요청 3:  504 (6.01s)
요청 4:  504 (6.03s)
요청 5:  504 (6.02s)   ← 5회 실패 → Circuit Breaker Open!
요청 6:  200 (0.00s)   ← 즉시 응답! (Fallback)
요청 7:  200 (0.00s)
요청 8:  200 (0.00s)
요청 9:  200 (0.00s)
요청 10: 200 (0.00s)
```

6번째 요청부터 응답 시간이 **0초**로 바뀌었습니다.

payment-api를 아예 호출하지 않고 즉시 Fallback 응답을 반환하는 겁니다. Fallback 응답 내용을 확인합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
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

결제 정보는 없지만 주문 정보는 보입니다. 서비스가 완전히 죽지 않고 **부분 기능으로 살아남은** 겁니다.

payment-api 로그도 확인합니다. Circuit Breaker가 Open된 이후로 로그가 더 이상 찍히지 않아야 합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    docker compose logs payment-api | Select-Object -Last 20
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    sudo docker compose logs payment-api | tail -20
    ```

```text title="출력 예시 — payment-api 로그"
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 요청 1~5 처리
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
(여기서 멈춤 — 요청 6~10은 Circuit Breaker가 차단)
```

!!! success "✅ 확인 포인트"
    - 5회 실패 후 6번째 요청부터 즉시(0~1ms) 응답이 나온 것을 확인했다
    - Fallback 응답에서 주문 정보는 보이고 결제 정보만 "조회불가"인 것을 확인했다
    - payment-api 로그가 5줄에서 멈춘 것을 확인했다 (이후 호출 차단)

!!! quote "박대리"
    *"6번째부터는 payment-api를 아예 안 부르네요. 숨통을 조이는 대신 '지금 안 된다'고 바로 말해주는 거네요."*

---

## Step 4. 자동 복구 확인 — Half-Open

10초가 지나면 Circuit Breaker가 Half-Open 상태로 전환됩니다.
Half-Open은 "테스트 요청 1개를 payment-api에 보내봐서 회복됐는지 확인하는 상태"입니다.

payment-api를 먼저 정상으로 복구합니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "0"      # 정상 복구
  FAULT_ERROR_RATE: "0.0"
```

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

Circuit Breaker의 reset_timeout(10초)을 기다렸다가 요청을 보냅니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Write-Host "10초 대기 중..."
    Start-Sleep -Seconds 11
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    sleep 11
    curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
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

`payment.status: "승인완료"` — 결제 정보가 다시 정상으로 돌아왔습니다.

```text
내부에서 일어난 일:
  10초 경과 → Half-Open 전환
  → 테스트 요청 1개 통과 → payment-api 응답 정상
  → Circuit Breaker Closed 복귀
  → 이후 요청 모두 정상 처리
```

!!! success "✅ 확인 포인트"
    - 10초 후 `payment.status: "승인완료"` 가 다시 나오면 OK
    - 수동으로 아무것도 하지 않아도 자동으로 복구된 것을 확인했다

---

## Step 5. Phase 1과 비교 — 장애 영향 범위가 얼마나 줄었나?

지난 목요일 상황(`FAULT_DELAY_MS=8000`)을 다시 재현합니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "8000"
  FAULT_ERROR_RATE: "0.0"
```

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

부하 테스트를 실행합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    1..30 | ForEach-Object -ThrottleLimit 10 -Parallel {
        Invoke-WebRequest -Uri http://127.0.0.1:8082/api/orders/ORD-001 -SkipHttpErrorCheck | Out-Null
    }
    Write-Host "완료"
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    ab -c 10 -n 30 -s 60 http://127.0.0.1:8082/api/orders/ORD-001
    ```

```text title="출력 예시 — Phase 2 Circuit Breaker 적용 후"
Status code distribution:
  [504] 5 responses    ← Circuit Breaker가 Open되기 전 5개만 실패
  [200] 25 responses   ← 나머지 25개는 Fallback으로 즉시 응답
```

Phase 1과 비교해보면:

| | Phase 1 (Timeout만) | Phase 2 (CB 추가) |
| --- | --- | --- |
| 30개 요청 결과 | 전부 8초씩 대기 후 실패 | 5개만 느리고 25개는 즉시 Fallback |
| 사용자 경험 | 8초 기다렸다 에러 | 즉시 "잠시 후 다시 시도" |
| payment-api 부하 | 무한 요청 | 5회 이후 차단 |

!!! quote "김팀장"
    *"지난 목요일에 이게 있었다면 44분이 아니라 5초 안에 Fast Fail로 전환됐을 거야."*

---

## Step 6. 원상복구

실습이 끝났으면 장애 설정을 되돌립니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "0"
  FAULT_ERROR_RATE: "0.0"
```

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

---

## 정리

Phase 2에서 경험한 것:

| 패턴 | 효과 | 한계 |
| --- | --- | --- |
| **Timeout** | 느린 요청이 무한 대기하지 않음 | 너무 짧으면 정상도 끊김 |
| **Retry** | 일시적 503 흡수 | 지속 장애엔 Retry Storm 유발 |
| **Circuit Breaker** | 지속 장애 시 즉시 Fast Fail, 복구 시간 확보 | 임계값 설정이 까다로움 |

!!! note "Circuit Breaker도 만능은 아닙니다"
    지금 실습에서는 order-api가 1개입니다.
    실제 서비스에서는 order-api 인스턴스가 여러 개이고, 각각 독립적으로 Circuit Breaker 상태를 가집니다.
    인스턴스 A는 Open인데 B는 Closed면 일부 요청은 여전히 느린 서비스로 흘러갑니다.
    분산 환경에서는 Redis 같은 외부 상태 저장소로 CB 상태를 공유해야 합니다.

!!! success "✅ 확인 포인트"
    - 5회 연속 실패 후 Circuit Breaker가 Open되어 즉시 Fallback 응답이 나온 것을 확인했다
    - payment-api 로그가 Open 이후로 멈춘 것을 확인했다 (호출 차단)
    - 10초 후 Half-Open → 복구 후 Closed로 돌아온 것을 확인했다
    - Phase 1과 비교해 장애 영향 범위가 크게 줄어든 것을 확인했다

---

## 다음 단계

[:material-arrow-left: Phase 2-1 · Retry의 효과와 역효과](retry-and-storm.md){ .md-button }
[Phase 3 · 사람 손 떼기 (1부) :material-arrow-right:](../phase-3/index.md){ .md-button .md-button--primary }
