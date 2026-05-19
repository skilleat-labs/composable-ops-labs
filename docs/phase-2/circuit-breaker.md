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

## Step 1. 현재 상태 확인

Phase 2-1에서 `FAULT_DELAY_MS=3000`으로 설정한 상태입니다.
(설정이 바뀌었다면 다시 맞춰주세요.)

단일 요청으로 현재 응답을 확인합니다.

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

## Step 2. Circuit Breaker Open 상태 만들기

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

## Step 3. 자동 복구 확인 — Half-Open

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

## Step 4. Phase 1과 비교 — 장애 영향 범위가 얼마나 줄었나?

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

## Step 5. 원상복구

실습이 끝났으면 장애 설정을 되돌리고, 브랜치도 main으로 돌아갑니다.

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

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    git fetch origin
    git checkout main
    docker compose up --build order-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    git fetch origin
    git checkout main
    sudo docker compose up --build order-api -d
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
