# Phase 1-2 · 장애 전파 체험

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"백문이 불여일견. 직접 장애 만들어보자. FAULT_DELAY_MS=8000 주입하면 지난 목요일 상황 그대로야."*

---

## Step 1. 정상 상태 응답 시간 기록

장애 전후를 비교하기 위해 먼저 정상 상태를 기록합니다.
앱이 아직 실행 중이어야 합니다. (안 돼 있으면 `sudo docker compose up --build`)

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    curl http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

```json title="출력 예시"
{
  "order_id": "ORD-001",
  ...
  "payment": {
    "response_time_ms": 0        // 정상 시 0~2ms
  },
  "total_response_time_ms": 8    // 환경마다 다름 (5~30ms 정상)
}
```

`total_response_time_ms` 값을 기억해두세요. 장애 주입 후 얼마나 늘어나는지 비교할 겁니다.

---

## Step 2. 장애 주입 — payment-api 8초 지연

`docker-compose.yml`을 열고 `FAULT_DELAY_MS` 값을 `"8000"` 으로 바꿉니다.

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    cd ~\hanbat-order-app-s2
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    cd ~/hanbat-order-app-s2
    ```

```yaml title="docker-compose.yml (수정 전 → 후)"
environment:
  FAULT_DELAY_MS: "8000"   # "0" → "8000" 으로 변경
  FAULT_ERROR_RATE: "0.0"
```

payment-api만 재시작합니다. **새 터미널을 열고** 아래 명령을 실행하세요.

!!! note "터미널 두 개 사용"
    - **터미널 1**: `sudo docker compose up --build` 실행 중 (건드리지 마세요)
    - **터미널 2**: 아래 명령 실행 → payment-api만 백그라운드(`-d`)로 재시작

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell, 새 창)"
    docker compose up payment-api -d
    ```

=== "Mac / Linux"

    ```bash title="터미널 (새 창)"
    sudo docker compose up payment-api -d
    ```

---

## Step 3. 단일 요청 — 얼마나 느려졌나?

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    curl http://127.0.0.1:8082/api/orders/ORD-001 | python3 -m json.tool --no-ensure-ascii
    ```

```text title="출력 예시"
{"order_id":"ORD-001", ... "total_response_time_ms":8011}
응답시간: 8.025초
```

0.025초 → **8.025초**. 결제 API가 느려지자 주문 조회도 똑같이 느려집니다.

!!! warning "왜 같이 느려지나요?"
    order-api는 `timeout=None` 이라 payment-api 응답을 그냥 기다립니다.
    payment-api가 8초 걸리면 order-api도 8초 묶여있습니다.

!!! note "503은 왜 안 나나요? — 동기 vs 비동기"
    "느리기만 하고 에러는 안 나네?" 라고 느꼈다면 정확히 보신 겁니다.

    이 앱은 FastAPI(비동기) 기반이라 `await` 를 만나면 스레드를 다른 요청에 양보합니다.
    스레드가 고갈되지 않으니 요청은 다 받되, 단지 느린 겁니다.

    동기 방식 서버(Flask, Spring MVC 등)였다면 결과가 달랐을 겁니다:

    ```text
    스레드 풀: 10개 (예시)

    요청 1  → 스레드 1 점유, payment-api 기다리는 중 (8초)
    요청 2  → 스레드 2 점유, payment-api 기다리는 중 (8초)
    ...
    요청 10 → 스레드 10 점유, payment-api 기다리는 중 (8초)

    요청 11 → 빈 스레드 없음 → 503 Service Unavailable
    ```

    **실제 44분 장애는 이 패턴이었습니다.** 비동기냐 동기냐에 따라 증상이 다르지만
    원인은 동일합니다 — `timeout=None` 으로 인한 무한 대기.

---

## Step 4. 동시 다발 요청 — 응답 지연 확인

이제 동시 요청을 여러 개 보내서 전체가 느려지는 것을 확인합니다.

부하 테스트 도구를 설치합니다.

`ab`(Apache Bench)는 웹 서버에 대량 요청을 보내 응답 시간과 처리량을 측정하는 도구입니다.
`-c`는 동시 연결 수, `-n`은 총 요청 수, `-s`는 소켓 타임아웃(초)입니다.

=== "Windows (PowerShell)"

    PowerShell에서는 별도 설치 없이 반복 요청을 보낼 수 있습니다.

    ```powershell title="터미널 (Windows PowerShell)"
    1..20 | ForEach-Object -ThrottleLimit 5 -Parallel {
        Invoke-RestMethod http://127.0.0.1:8082/api/orders/ORD-001 | Out-Null
    }
    Write-Host "완료"
    ```

=== "Mac / Linux"

    ```bash title="터미널"
    sudo apt-get install -y apache2-utils
    ```

동시 5개 요청을 총 20개 보냅니다.

=== "Windows (PowerShell)"

    위 명령이 완료되는 데 걸린 시간을 확인합니다. 정상 시 1초 미만, 장애 시 8초 이상 걸립니다.

=== "Mac / Linux"

    ```bash title="터미널"
    ab -c 5 -n 20 -s 60 http://127.0.0.1:8082/api/orders/ORD-001
    ```

!!! tip "`localhost` 대신 `127.0.0.1`을 쓰는 이유"
    `localhost`는 시스템에 따라 IPv6(`::1`)로 해석될 수 있습니다.
    Docker는 IPv4만 열려있는 경우가 많아 연결이 실패합니다.
    `127.0.0.1`로 명시하면 IPv4로 확실히 연결됩니다.

```text title="출력 예시 (Mac / Linux)"
Concurrency Level:      5
Time taken for tests:   40.249 seconds
Complete requests:      20
Failed requests:        0

Percentage of the requests served within a certain time (ms)
  50%   8052
  100%  8055 (longest request)   ← 전부 8초씩 걸림
```

!!! danger "이게 지난 목요일에 일어난 일"
    에러는 안 났지만 **60개 요청 전부 8초씩** 걸렸습니다.
    정상 시 8ms → 장애 시 8,000ms. 1,000배 느려진 겁니다.
    사용자 입장에서 8초 기다리는 것은 사실상 서비스 불가 상태입니다.
    실제 서비스에서는 이 상태가 44분간 지속됐습니다.

---

## Step 5. 주문 조회 화면도 확인

브라우저에서 아래 주소로 접속합니다.

```
http://127.0.0.1:3000
```

결제 API가 느려진 상태에서 주문 내역 화면이 어떻게 되는지 직접 확인합니다.

!!! quote "박대리"
    *"주문 조회인데... 결제 때문에 이것도 안 되네요. 고객 입장에서는 결제도 못 하고 주문 내역도 못 보는 거잖아요."*

---

## Step 6. 원상 복구

실습이 끝났으면 장애 설정을 되돌립니다.

```yaml title="docker-compose.yml"
environment:
  FAULT_DELAY_MS: "0"      # 다시 "0" 으로
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

오늘 체험한 것:

| 상황 | 결과 |
| --- | --- |
| payment-api 정상 | order-api 응답 ~12ms |
| payment-api 8초 지연 | order-api도 8초 지연 |
| 동시 30개 요청 + 8초 지연 | 일부 요청 502 에러, 전체 응답 불가 |

**근본 원인**: `timeout=None` → 스레드 점유 → 커넥션 풀 고갈

!!! success "✅ 확인 포인트"
    - 부하 테스트 결과에서 응답 시간이 8초 이상인 것을 확인했다
    - 장애 설정을 `FAULT_DELAY_MS: "0"` 으로 되돌렸다

!!! quote "김팀장"
    *"직접 봤지? Phase 2에서 Timeout이랑 Circuit Breaker 붙이면 이게 어떻게 달라지는지 볼 거야."*

---

## 다음 단계

[:material-arrow-left: Phase 1-1 · 코드로 보는 호출 구조](rest-call-structure.md){ .md-button }
[Phase 2 · 복원력 실험실 :material-arrow-right:](../phase-2/index.md){ .md-button .md-button--primary }
