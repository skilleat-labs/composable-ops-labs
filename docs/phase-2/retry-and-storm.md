# Phase 2-1 · Retry의 효과와 역효과

!!! info "예상 소요 45분"

---

## Phase 1에서 Phase 2로 — 장애를 봤다, 이제 어떻게 고치나?

Phase 1에서 우리는 이걸 직접 봤습니다.

```text
payment-api가 8초 늦어지자
  → order-api 응답도 8초 느려졌고
  → 동시 요청이 쌓이면서 사용자 전체가 기다리게 됐다
```

**여러분이라면 어떻게 고치겠어요?**

실제 현장에서도 장애 직후에 가장 먼저 나오는 아이디어가 있습니다.

!!! quote "박대리"
    *"간단하네요. 그냥 **재시도(Retry)** 한 번 넣으면 되는 거 아니에요? 일시적으로 느려진 거니까 두 번 시도하면 되잖아요."*

직관적으로 맞는 말처럼 들립니다. 실패하면 다시 시도한다 — 간단하고 명쾌합니다.

!!! quote "이대리"
    *"한 번 해보죠. 저도 3년 전에 똑같은 얘기 했어요."*

직접 실험해봅니다. Retry가 언제 효과가 있고 언제 역효과가 나는지 눈으로 확인합니다.

---

## 이론 — Retry는 언제 효과가 있나?

Retry(재시도)는 **일시적(transient) 장애**에만 효과가 있습니다.

일시적 장애란 곧 사라지는 장애입니다. 잠깐 네트워크가 흔들렸거나, 서버가 순간적으로 503을 반환했지만 다음 요청은 잘 받는 경우입니다.

```text
일시적 장애 예시:
  요청 1 → 503 (서버 순간 과부하)
  요청 2 → 200 (금방 회복)  ← Retry하면 성공!

지속 장애 예시:
  요청 1 → 503 (PG사 응답 지연)
  요청 2 → 503 (여전히 지연)  ← Retry해도 503
  요청 3 → 503 (여전히 지연)  ← 요청만 3배로 늘어남
```

| 장애 유형 | 예시 | Retry 결과 |
| --- | --- | --- |
| **일시적** | 네트워크 순간 끊김, 간헐적 503 | ✅ 재시도하면 성공 가능 |
| **지속적** | 서버 다운, 응답 지연 8초 | ❌ 재시도해도 같은 실패 → 요청만 3배 증가 |

여기서 핵심은 **Retry 자체가 새로운 HTTP 요청**이라는 점입니다.

```text
사용자 요청 1개
  → order-api가 payment-api 호출 (1번째 시도) → 실패
  → order-api가 payment-api 호출 (2번째 시도) → 실패
  → order-api가 payment-api 호출 (3번째 시도) → 실패
  → 사용자에게 에러 반환
```

사용자는 1번 요청했지만, payment-api 입장에서는 3번 요청이 들어온 겁니다.

지속 장애 상황에서 이게 왜 문제냐면:

```text
payment-api: "나 지금 힘들어서 응답 못 하겠어..."
order-api:   "실패했네, 다시 시도" → 또 실패
             "또 실패했네, 다시 시도" → 또 실패
             "또 실패했네, 다시 시도" → 또 실패

payment-api: "요청이 3배가 됐는데 나는 더 힘들어졌어..."
```

회복할 틈을 주기는커녕 더 괴롭히는 겁니다. 이게 Retry Storm입니다.

지난 목요일 사건은 PG사 **응답 지연(지속 장애)**이었습니다.
Retry를 넣었다면 어떻게 됐을까요? 지금부터 직접 확인합니다.

---

## Step 1. 실습 환경 이해하기 — 장애 주입 스위치

실습에 앞서 `FAULT_DELAY_MS`와 `FAULT_ERROR_RATE`가 무엇인지 짚고 넘어갑니다.

실제 장애는 PG사 서버가 진짜로 느려져야 재현이 됩니다. 실습 환경에서 그걸 만들 수 없으니, payment-api 코드 안에 **인위적으로 장애를 주입하는 스위치**를 미리 만들어둔 겁니다.

| 환경변수 | 실제 장애 상황 | 실습에서 재현하는 방식 |
| --- | --- | --- |
| `FAULT_DELAY_MS=8000` | PG사 서버 응답이 8초씩 걸림 | payment-api가 일부러 8초 늦게 응답 |
| `FAULT_ERROR_RATE=0.7` | PG 게이트웨이가 간헐적으로 503 반환 | payment-api가 70% 확률로 503 반환 |

이런 방식을 **Fault Injection(장애 주입)** 이라고 부릅니다. 실무에서도 장애 대응 훈련 시 사용하는 방식입니다.

!!! note "Chaos Engineering"
    Netflix는 프로덕션 서버를 일부러 랜덤으로 죽이는 "Chaos Monkey"라는 도구를 만들어
    실제 장애 상황에서도 시스템이 살아남는지 훈련했습니다.
    이 실습의 `FAULT_DELAY_MS`, `FAULT_ERROR_RATE`가 그 개념을 환경변수 두 개로 단순화한 겁니다.

이번 실습에서는 `FAULT_ERROR_RATE`를 씁니다.

```text
FAULT_ERROR_RATE: "0.0"  → 0% 에러, 모든 요청 정상 처리
FAULT_ERROR_RATE: "0.7"  → 70% 확률로 503 반환, 30%만 성공
FAULT_ERROR_RATE: "1.0"  → 100% 에러, 모든 요청 실패
```

이걸로 "간헐적으로 PG 게이트웨이가 503을 던지는 상황"을 재현합니다.

---

## Step 2. Retry 없는 상태에서 — 에러가 사용자에게 그대로 노출

먼저 현재 코드(Retry 없음) 상태에서 간헐적 에러가 어떻게 보이는지 확인합니다.

`docker-compose.yml`에서 `FAULT_ERROR_RATE`를 `0.7`로 바꿉니다.

```yaml title="docker-compose.yml (payment-api 환경변수 수정)"
environment:
  FAULT_DELAY_MS: "0"
  FAULT_ERROR_RATE: "0.7"   # "0.0" → "0.7" (70% 확률로 503 반환)
```

payment-api만 재시작합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

이제 요청을 5번 연속으로 보냅니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    1..5 | ForEach-Object {
        Write-Host "--- 요청 $_ ---"
        Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    }
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    for i in {1..5}; do
      echo "--- 요청 $i ---"
      curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    done
    ```

```text title="출력 예시 — Retry 없는 상태"
--- 요청 1 ---
{"detail": {"error": "PG_GATEWAY_UNAVAILABLE", ...}}   ← 실패 (503 그대로 전달)
--- 요청 2 ---
{"order_id": "ORD-001", "payment": {...}}              ← 성공
--- 요청 3 ---
{"detail": {"error": "PG_GATEWAY_UNAVAILABLE", ...}}   ← 실패
--- 요청 4 ---
{"detail": {"error": "PG_GATEWAY_UNAVAILABLE", ...}}   ← 실패
--- 요청 5 ---
{"order_id": "ORD-001", "payment": {...}}              ← 성공
```

5번 중 2번만 성공했습니다. payment-api가 간헐적으로 503을 던지고 있고, order-api는 그걸 그대로 사용자에게 내보내고 있습니다. 재시도가 없으니 첫 번째 시도가 실패하면 그냥 에러입니다.

!!! success "✅ 확인 포인트"
    5번 중 에러가 섞여 나오면 OK. 70% 확률이라 매번 다를 수 있습니다.

---

## Step 3. Retry 직접 구현하기

Retry는 환경변수로 켜고 끄는 게 아닙니다. **order-api 코드 자체를 수정**해야 합니다.
지금부터 Phase 1에서 문제였던 코드를 직접 고쳐봅니다.

### 3-1. 현재 코드 확인

`order-api/app.py`를 열어봅니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
cat order-api/app.py
```

payment-api를 호출하는 부분(88번째 줄 근처)을 찾으면 이런 TODO 주석이 있습니다:

```python title="order-api/app.py (현재)"
# ⚠️  Phase 1: timeout=None (타임아웃 없음) — 이것이 연쇄 장애의 원인!
#
# 💡 Phase 2 TODO:
#    [STEP 1] timeout 추가:
#      async with httpx.AsyncClient(timeout=2.0) as client:
#
#    [STEP 2] tenacity 로 Retry 래핑
#
#    [STEP 3] pybreaker 로 Circuit Breaker 래핑

async with httpx.AsyncClient(timeout=None) as client:  # ← Phase 2에서 수정
```

코드 안에 이미 무엇을 바꿔야 하는지 표시가 되어 있습니다. STEP 1부터 차례로 따라갑니다.

!!! note "tenacity, pybreaker는 이미 설치되어 있습니다"
    `requirements.txt`를 보면 `tenacity==8.3.0`, `pybreaker==1.2.0`이 이미 있습니다.
    라이브러리 추가 없이 바로 코드 수정으로 넘어갑니다.

---

### 3-2. STEP 1 — timeout 추가

`order-api/app.py`를 편집기로 열어서 한 줄만 바꿉니다.

```python title="order-api/app.py — 수정 전"
async with httpx.AsyncClient(timeout=None) as client:
```

```python title="order-api/app.py — 수정 후"
async with httpx.AsyncClient(timeout=2.0) as client:
```

`timeout=None` → `timeout=2.0` 으로만 바꾸면 됩니다.

이것만으로도 payment-api가 2초 안에 응답하지 않으면 더 이상 무한정 기다리지 않습니다.

---

### 3-3. STEP 2 — Retry 추가

이번엔 파일 상단 import 블록에 tenacity를 추가하고, payment-api 호출 부분을 함수로 분리합니다.

**① 파일 상단 import에 추가**

```python title="order-api/app.py — import 추가"
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_exception_type
```

**② payment-api 호출 부분을 아래처럼 교체**

기존:

```python title="order-api/app.py — 수정 전"
async with httpx.AsyncClient(timeout=2.0) as client:
    try:
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        resp.raise_for_status()
        payment = resp.json()
    except httpx.TimeoutException:
        raise HTTPException(status_code=504, ...)
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=502, ...)
```

교체 후:

```python title="order-api/app.py — 수정 후"
@retry(
    stop=stop_after_attempt(3),                                        # 최대 3회
    wait=wait_fixed(0.5),                                              # 재시도 간격 0.5초
    retry=retry_if_exception_type((httpx.TimeoutException, httpx.HTTPStatusError))
)
async def _fetch_payment(order_id: str) -> dict:
    async with httpx.AsyncClient(timeout=2.0) as client:
        resp = await client.get(f"{PAYMENT_API_URL}/api/payments/{order_id}")
        resp.raise_for_status()                                        # 503 → HTTPStatusError → 재시도
        return resp.json()

try:
    payment = await _fetch_payment(order_id)
except httpx.TimeoutException:
    raise HTTPException(status_code=504, detail={"error": "PAYMENT_API_TIMEOUT", "order_id": order_id})
except httpx.HTTPStatusError as e:
    raise HTTPException(status_code=502, detail={"error": "PAYMENT_API_ERROR", "upstream_status": e.response.status_code})
except httpx.RequestError as e:
    raise HTTPException(status_code=502, detail={"error": "PAYMENT_API_UNREACHABLE", "detail": str(e)})
```

!!! note "무엇이 바뀌었나?"
    ```text
    timeout=None  → timeout=2.0   : 2초 안에 응답 없으면 포기
    retry 3회                     : TimeoutException, HTTPStatusError 발생 시 최대 3번 재시도
    resp.raise_for_status()       : 503 응답을 HTTPStatusError로 변환 → 재시도 트리거
    ```

---

### 3-4. order-api 재빌드

코드가 바뀌었으니 이미지를 새로 빌드합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up --build order-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up --build order-api -d
    ```

빌드가 완료되면 단일 요청으로 동작을 확인합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

!!! success "✅ 확인 포인트"
    정상 응답이 오면 코드 수정이 잘 적용된 겁니다.

---

## Step 4. Retry 효과 확인 — 간헐적 503 흡수

`FAULT_ERROR_RATE=0.7` 상태를 유지한 채로 다시 요청합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    1..5 | ForEach-Object {
        Write-Host "--- 요청 $_ ---"
        Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    }
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    for i in {1..5}; do
      echo "--- 요청 $i ---"
      curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    done
    ```

```text title="출력 예시 — Retry 적용 후"
--- 요청 1 ---
{"order_id": "ORD-001", "payment": {...}}   ← 성공
--- 요청 2 ---
{"order_id": "ORD-001", "payment": {...}}   ← 성공
--- 요청 3 ---
{"order_id": "ORD-001", "payment": {...}}   ← 성공
--- 요청 4 ---
{"order_id": "ORD-001", "payment": {...}}   ← 성공
--- 요청 5 ---
{"order_id": "ORD-001", "payment": {...}}   ← 성공
```

성공률이 크게 올라갔습니다. 사용자 입장에서는 에러가 사라진 것처럼 보입니다.

그런데 **payment-api는 실제로 몇 번이나 호출됐을까요?** 로그를 확인합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    docker compose logs payment-api | Select-Object -Last 20
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    sudo docker compose logs payment-api | tail -20
    ```

```text title="출력 예시 — payment-api 로그"
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 503   ← 1번째 시도 실패
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 503   ← 2번째 시도 실패
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 3번째 시도 성공!
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 1번째 시도 성공
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 503
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
...
```

order-api는 요청 1개를 받았지만 payment-api에는 **최대 3번**을 호출했습니다.
Retry가 일시적 503을 흡수하고 있습니다.

!!! success "✅ 확인 포인트"
    - 사용자 응답은 대부분 성공으로 바뀌었다
    - payment-api 로그에서 503 → 503 → 200 순서의 재시도 흔적이 보인다

---

## Step 5. Retry Storm — 지속 장애에서 Retry는 독이 된다

이번엔 **지속 장애** 상황으로 바꿉니다.
`FAULT_DELAY_MS=3000`으로 설정합니다. payment-api가 3초씩 늦게 응답합니다.

```text
왜 3초인가?
  phase2 코드의 타임아웃: 2초
  payment-api 응답 시간: 3초

  → payment-api는 3초 후 응답하지만 order-api는 2초 만에 포기합니다.
  → 모든 요청이 Timeout → Retry 3회 → 총 6초 걸린 후 실패
```

```yaml title="docker-compose.yml (수정)"
environment:
  FAULT_DELAY_MS: "3000"
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

단일 요청으로 먼저 확인합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    $start = Get-Date
    try { Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001 } catch { $_.Exception.Message }
    Write-Host "소요: $(((Get-Date) - $start).TotalSeconds)초"
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    time curl -s http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

```text title="출력 예시"
{"detail": {"error": "PAYMENT_API_TIMEOUT", ...}}
소요: 6.03초   ← 2초 대기 × 3회 재시도 = 6초
```

한 번의 요청이 6초 걸립니다. 이제 동시 요청을 10개 보내봅니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    1..10 | ForEach-Object -ThrottleLimit 10 -Parallel {
        Invoke-WebRequest -Uri http://127.0.0.1:8082/api/orders/ORD-001 -SkipHttpErrorCheck | Out-Null
    }
    Write-Host "완료"
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    ab -c 10 -n 30 -s 60 http://127.0.0.1:8082/api/orders/ORD-001
    ```

부하 테스트가 끝나면 payment-api 로그를 확인합니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    docker compose logs payment-api | Select-Object -Last 40
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    sudo docker compose logs payment-api | tail -40
    ```

```text title="출력 예시 — payment-api 로그"
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200   ← 3초 후 응답, 이미 timeout
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
payment-api  | INFO: "GET /api/payments/ORD-001 HTTP/1.1" 200
... (30개 요청 × 최대 3회 재시도 = 최대 90줄)
```

order-api에는 30개 요청이 왔는데, payment-api 로그에는 **최대 90줄**이 찍혔습니다.

!!! danger "Retry Storm — 재시도 폭주"
    이미 지연으로 힘든 payment-api에 **3배의 부하**가 추가로 쏟아집니다.

    ```text
    사용자 요청: 30개
        ↓  (Retry 3회)
    payment-api 실제 수신: 최대 90개

    payment-api: "이미 응답 못 하고 있는데 요청이 3배가 됐어..."
    ```

    회복될 기회를 주기는커녕 숨통을 더 조이는 겁니다.
    이것이 **Retry Storm(재시도 폭주)**입니다.

!!! quote "이대리"
    *"봤죠? 지속 장애에 Retry를 붙이면 장애 서비스 회복은커녕 숨통을 조입니다. 3년 전 저도 이거 보고 처음엔 이해가 안 됐어요."*

---

## 정리

| 상황 | Retry 결과 |
| --- | --- |
| 일시적 503 (`FAULT_ERROR_RATE=0.7`) | ✅ 재시도로 성공률 향상 |
| 지속 지연 (`FAULT_DELAY_MS=3000`) | ❌ Retry Storm — 장애 서비스에 3배 부하 |

**Retry의 교훈**: 일시적 장애에만 유효. 지속 장애에는 Circuit Breaker가 필요합니다.

!!! success "✅ 확인 포인트"
    - `FAULT_ERROR_RATE=0.7` 상황에서 Retry 후 성공률이 올라간 것을 확인했다
    - payment-api 로그에서 한 사용자 요청에 여러 번 호출된 흔적을 확인했다
    - `FAULT_DELAY_MS=3000` 상황에서 payment-api 로그가 최대 3배로 증가한 것을 확인했다

---

## 다음 실습 준비

`FAULT_DELAY_MS=3000` 설정을 유지한 채로 다음 단계로 넘어갑니다.

---

## 다음 단계

[:material-arrow-left: Phase 2 개요](index.md){ .md-button }
[Phase 2-2 · Circuit Breaker 적용 :material-arrow-right:](circuit-breaker.md){ .md-button .md-button--primary }
