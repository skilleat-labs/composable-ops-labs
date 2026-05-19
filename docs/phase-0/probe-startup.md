# Probe 3 · Startup Probe

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"Spring Boot 뜨는 데 40초 걸리는 앱에 Liveness를 초반에 돌리면 어떻게 되겠어? 멀쩡한 앱을 계속 죽이는 거야. Startup Probe는 그걸 막는 안전장치야."*

---

## 이론

### Startup Probe가 필요한 이유

Liveness Probe는 주기적으로 체크합니다. 그런데 초기화가 긴 앱은 **뜨는 동안 Liveness가 실패해서** 계속 재시작될 수 있습니다.

```text title="Startup Probe 없을 때 — 초기화 40초 앱"
t=0s    컨테이너 시작, 앱 초기화 중...
t=5s    Liveness Probe 실행 → 실패 (아직 초기화 중)
t=10s   Liveness Probe 실행 → 실패
t=15s   Liveness Probe 실행 → 실패  (failureThreshold: 3 초과)
        → 컨테이너 재시작! (앱이 뜨기도 전에)
t=0s    다시 시작... 무한 반복 ❌
```

### Startup Probe의 역할

**Startup Probe가 성공하기 전까지 Liveness와 Readiness를 잠가둡니다.**

```text title="Startup Probe 있을 때"
t=0s    컨테이너 시작
        Liveness/Readiness ← 비활성화 (Startup이 성공할 때까지)
        Startup Probe만 실행 중...

t=40s   앱 초기화 완료 → Startup Probe 성공
        Liveness/Readiness ← 활성화

t=45s   Liveness Probe 시작
        Readiness Probe 시작 → READY 1/1
```

### 최대 대기 시간 설정

Startup Probe는 `failureThreshold × periodSeconds`로 최대 대기 시간을 설정합니다:

```yaml
startupProbe:
  failureThreshold: 30   # 30회
  periodSeconds: 10      # × 10초 = 최대 300초(5분) 대기
```

이 시간 안에 Startup Probe가 성공하지 못하면 컨테이너를 재시작합니다.

---

## 실습

### Step 1. Startup Probe 동작 확인

`sleep 20`으로 초기화가 20초 걸리는 앱을 시뮬레이션합니다.

```yaml title="pod-startup.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: startup-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/started
        failureThreshold: 30   # 30 × 10s = 최대 300초 대기
        periodSeconds: 10
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-startup.yaml
kubectl get pod startup-app -w
```

!!! success "✅ 확인 포인트"
    처음 20초간 `READY 0/1`이지만 `RESTARTS`는 0으로 유지됩니다.
    20초 후 `READY 1/1`로 전환되면 OK.

---

### Step 2. Startup Probe 없었다면? (비교 실습)

`initialDelaySeconds` 없이 Liveness만 설정하면 어떻게 되는지 확인합니다.

```yaml title="pod-no-startup.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: no-startup-app
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        initialDelaySeconds: 0   # 즉시 시작
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-no-startup.yaml
kubectl get pod no-startup-app -w
```

!!! success "✅ 확인 포인트"
    약 15초 후(5×3=15초) `RESTARTS`가 올라가기 시작합니다.
    Startup Probe의 필요성이 보이면 OK.

---

### Step 3. Startup 시간 초과 → 재시작

`failureThreshold × periodSeconds`를 실제 초기화 시간보다 **짧게** 설정하면 어떻게 될까요?
이 설정 실수가 운영에서 의외로 자주 발생합니다.

앱은 20초가 필요한데, Startup Probe의 최대 허용 시간을 6초(3 × 2)로 설정합니다:

```yaml title="pod-startup-timeout.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: startup-timeout
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 20 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/started
        failureThreshold: 2    # 2회
        periodSeconds: 3       # × 3초 = 최대 6초만 허용 (앱 초기화 20초 필요)
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-startup-timeout.yaml
kubectl get pod startup-timeout -w
```

```text title="출력 예시"
NAME               READY   STATUS    RESTARTS   AGE
startup-timeout    0/1     Running   0          5s
startup-timeout    0/1     Running   1          15s   ← 6초 안에 초기화 못 해서 재시작
startup-timeout    0/1     Running   2          30s   ← 또 재시작 → CrashLoopBackOff로 진행
```

```bash title="터미널"
kubectl describe pod startup-timeout
```

```text title="출력 예시 — Events"
Warning  Unhealthy  Startup probe failed: ...
Normal   Killing    Stopping container app
Normal   Started    Started container app
```

!!! success "✅ 확인 포인트"
    `RESTARTS`가 계속 올라가면 OK. Startup Probe 시간 설정 실수가 어떤 결과를 낳는지 확인.

!!! danger "실무 실수 패턴"
    **초기화 시간을 과소 추정**하는 것이 가장 흔한 실수입니다.
    운영 중 배포할 때 갑자기 CrashLoopBackOff가 나타난다면 Startup Probe의
    `failureThreshold × periodSeconds` 값을 먼저 점검하세요.

    ```
    잘못된 설정:  failureThreshold: 5 × periodSeconds: 5  = 25초  (앱 초기화 30초 필요)
    올바른 설정:  failureThreshold: 10 × periodSeconds: 5 = 50초  (여유분 포함)
    ```

---

### Step 4. 3 Probe 조합 — 활성화 순서 직접 확인

Startup → Readiness → Liveness가 **순서대로** 작동하는 것을 한 Pod에서 확인합니다.
이게 Startup Probe의 핵심입니다.

```yaml title="pod-all-probes.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: all-probes
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "sleep 15 && touch /tmp/started && sleep 3600"
      startupProbe:
        exec:
          command: [cat, /tmp/started]
        failureThreshold: 10
        periodSeconds: 3       # 최대 30초 대기
      readinessProbe:
        exec:
          command: [cat, /tmp/started]
        periodSeconds: 3
        failureThreshold: 2
      livenessProbe:
        exec:
          command: [cat, /tmp/started]
        periodSeconds: 10
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-all-probes.yaml
kubectl get pod all-probes -w
```

```text title="출력 예시 — 시간 흐름"
NAME         READY   STATUS    RESTARTS   AGE
all-probes   0/1     Running   0          3s    ← Startup 실행 중, Liveness/Readiness 잠김
all-probes   0/1     Running   0          6s    ← Startup 계속 실패 중 (RESTARTS는 0 유지!)
all-probes   0/1     Running   0          12s   ← 아직 초기화 중
all-probes   1/1     Running   0          18s   ← Startup 성공 → Readiness 즉시 통과 → READY!
```

!!! success "✅ 확인 포인트"
    15초간 `RESTARTS`가 0으로 유지되면 Startup Probe가 Liveness를 잘 막고 있는 것.
    15초 후 `READY 1/1`로 한번에 전환되면 OK.

이벤트로 Probe 흐름을 상세히 확인합니다:

```bash title="터미널"
kubectl describe pod all-probes
```

```text title="출력 예시 — 정상 흐름"
Events:
  Normal   Scheduled   Pod가 노드에 배치됨
  Normal   Pulled      이미지 pull 완료
  Normal   Created     컨테이너 생성
  Normal   Started     컨테이너 시작
  # (15초간 Startup Probe 실패 이벤트는 없음 — failureThreshold 미초과)
  # 15초 후 Startup 성공, Readiness 활성화, READY 1/1
```

### 3 Probe 활성화 타임라인 정리

```text title="3 Probe 타임라인"
t=0s   컨테이너 시작
       ┌─────────────────────────────────────────┐
       │  Startup Probe 실행 중                   │  Liveness ← 잠김
       │  (3초마다 /tmp/started 체크)             │  Readiness ← 잠김
       └─────────────────────────────────────────┘
t=15s  /tmp/started 생성 → Startup 성공!
       Startup Probe 종료
       ┌──────────────────┐  ┌──────────────────┐
       │  Readiness 활성화 │  │  Liveness 활성화  │
       │  READY 1/1       │  │  10초마다 체크    │
       └──────────────────┘  └──────────────────┘
```

---

### 정리

```bash title="터미널"
kubectl delete pod startup-app no-startup-app startup-timeout all-probes
```

### 3가지 Probe 한눈에 비교

| Probe | 실패 시 동작 | 언제 사용 |
|-------|------------|---------|
| **Liveness** | 컨테이너 재시작 | 교착 상태·무한 루프 감지 |
| **Readiness** | Service Endpoint에서 제거 | DB 연결 전, 초기화 중 트래픽 차단 |
| **Startup** | Liveness/Readiness 보호 | 초기화가 긴 앱 (JVM, Spring Boot 등) |

### 실무 설정 가이드

| Probe | 권장 설정 |
|-------|---------|
| Startup | `failureThreshold × periodSeconds` ≥ 최대 초기화 시간 |
| Readiness | `periodSeconds` 5~10s |
| Liveness | `initialDelaySeconds` 충분히, `failureThreshold` 3~5 |

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| 초기화 중 계속 재시작 | Startup Probe 추가 또는 `initialDelaySeconds` 늘리기 |
| Startup Probe도 실패 | `failureThreshold × periodSeconds` 값이 초기화 시간보다 짧은지 확인 |

---

## 다음 단계

[:material-arrow-left: Probe 2 · Readiness](probe-readiness.md){ .md-button }
[K8s 2 · Resource & QoS :material-arrow-right:](resource-qos.md){ .md-button .md-button--primary }
