# Phase 2 · 복원력 실험실

!!! info "예상 소요 1시간 · 난이도 ⭐⭐"

---

## 스토리

Phase 1에서 장애를 재현한 후, 박대리는 자연스럽게 생각했습니다.

!!! quote "박대리"
    *"간단하네요. 그냥 **재시도(Retry)** 한 번 넣으면 되는 거 아니에요? 일시적으로 느려진 거니까 두 번 시도하면 되잖아요."*

이대리가 희미하게 웃으며 대답합니다.

!!! quote "이대리"
    *"한 번 해보죠. 저도 3년 전에 똑같은 얘기 했어요."*

---

## 핵심 질문

1. Retry를 넣으면 장애가 해결될까? (스포일러: 더 나빠질 수도 있음)
2. Circuit Breaker는 어떤 문제를 푸는가?
3. Retry와 Circuit Breaker를 둘 다 쓰는 게 맞는가?
4. 이 패턴들의 **한계**는 무엇인가?

---

## 학습 목표

- [x] Retry의 **역효과**(재시도 폭주, Retry Storm)를 직접 관찰한다
- [x] Circuit Breaker의 3가지 상태(Closed, Open, Half-Open)를 설명할 수 있다
- [x] `pybreaker` 라이브러리로 실제 코드에 Circuit Breaker를 적용한다
- [x] 장애 전파가 차단되는 것을 체감하고, **완벽하지 않은 이유**도 이해한다

---

## 세부 단원

### Phase 2-1 · Retry의 효과와 역효과 (30분)

| 소단원 | 활동 |
| --- | --- |
| Exponential Backoff 개념 | 강의: 지수 백오프, Jitter의 필요성 |
| Retry 적용 | `order-api` 코드에 `tenacity` 라이브러리로 3회 재시도 추가 |
| 재시도 폭주 관찰 | 결제 API가 계속 느린 상태에서 주문 API의 요청 수가 오히려 **3배**로 증가하는 걸 로그로 확인 |
| 교훈 정리 | "Retry는 일시적(transient) 장애에만 유효하다" |

### Phase 2-2 · Circuit Breaker 적용 (30분)

| 소단원 | 활동 |
| --- | --- |
| 3상태 모델 | Closed(정상) / Open(차단) / Half-Open(시험) 상태 전이 |
| `pybreaker` 적용 | 결제 호출 함수에 `@circuit(failure_threshold=5, recovery_timeout=10)` 데코레이터 |
| Open 상태 관찰 | 연속 실패 후 결제 호출 자체를 건너뛰는 동작 확인 |
| Fallback 처리 | "결제 정보 표시 불가 — 주문만 조회 가능" 같은 대체 응답 구성 |

!!! warning "Circuit Breaker도 만능은 아닙니다"
    - **임계값 설정이 까다롭다**: 너무 민감하면 멀쩡한 서비스도 차단, 너무 둔하면 장애 전파 못 막음
    - **상태 공유 문제**: 여러 인스턴스 중 한 개만 Open이고 나머지는 Closed면 일부 요청은 여전히 죽는 서비스로 흘러감 → 분산 환경에선 Redis 등 외부 상태 저장소 필요

### 종합 토론 (시간 여유 시)

- 한밭푸드 상황에서 결제 API 호출에 Circuit Breaker를 적용하는 것이 **현명한 판단**인가?
- 만약 결제가 "주문 조회와 직결된 필수 정보"라면 Fallback을 어떻게 설계해야 하는가?

---

## 예상 결과물

- :material-code-tags: Retry + Circuit Breaker가 적용된 `order-api` 코드
- :material-chart-line: 장애 상황에서 주문 API가 **부분 기능으로 응답**하는 것을 보여주는 터미널 출력
- :material-note-edit: "우리는 어떤 임계값을 선택했고, 왜 그렇게 선택했는가" 한 문단 메모

---

## 다음 단계

[:material-arrow-left: Phase 1 · 연쇄 장애의 밤](../phase-1/index.md){ .md-button }
[Phase 3 · 사람 손 떼기 (1부) :material-arrow-right:](../phase-3/index.md){ .md-button .md-button--primary }
