# Probe 1 · Liveness Probe

!!! info "예상 소요 15분"

---

!!! quote "김팀장"
    *"프로세스가 살아있다고 앱이 정상인 건 아니야. 무한 루프에 빠진 앱도 프로세스는 멀쩡하거든. Liveness Probe는 그걸 잡아내는 거야."*

---

## 이론

### Liveness Probe란?

K8s는 컨테이너가 실행 중인지는 알 수 있습니다. 하지만 **실행 중인 앱이 정상적으로 동작하는지**는 모릅니다.

예를 들어:

- 교착 상태(Deadlock)에 빠진 Java 앱 — 프로세스는 살아있지만 요청을 처리 못 함
- 무한 루프에 빠진 Worker — CPU만 먹고 아무 일도 안 함

이런 상황을 감지해서 컨테이너를 **자동 재시작**하는 것이 Liveness Probe입니다.

```
Liveness Probe 실패 → 컨테이너 재시작
```

### 핵심 파라미터

| 파라미터 | 의미 |
|---------|------|
| `initialDelaySeconds` | 컨테이너 시작 후 **첫 Probe까지 대기** (앱 초기화 시간 확보) |
| `periodSeconds` | 이후 **반복 주기** |
| `failureThreshold` | **연속 몇 회 실패**해야 재시작할지 |

### 타이밍 흐름

```text title="Probe 타이밍 예시 (initialDelaySeconds:5, periodSeconds:5)"
t=0s    컨테이너 시작
        ↕ initialDelaySeconds: 5 — 앱 초기화 대기
t=5s    첫 번째 Probe 실행
        ↕ periodSeconds: 5
t=10s   두 번째 Probe 실행
        ↕ periodSeconds: 5
t=15s   세 번째 Probe 실행
        ...5초마다 반복...
```

!!! note "failureThreshold: 3 의 의미"
    한 번 실패해도 바로 재시작하지 않습니다.
    **연속 3회 실패** = 최대 `periodSeconds × failureThreshold = 5 × 3 = 15초` 감지 후 재시작.

!!! tip "initialDelaySeconds가 왜 필요한가?"
    컨테이너가 뜨는 순간 앱은 아직 초기화 중입니다.
    이 시점에 Probe를 바로 실행하면 정상 앱도 실패로 판정되어 불필요하게 재시작됩니다.
    Spring Boot, JVM처럼 기동이 느린 앱은 `initialDelaySeconds: 30` 이상으로 설정하세요.

---

## 실습

### Step 1. 정상 동작 확인

Liveness Probe가 계속 성공하는 Pod를 만듭니다.

```yaml title="pod-liveness-ok.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: liveness-ok
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/healthy && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-liveness-ok.yaml
kubectl get pod liveness-ok -w
```

!!! success "✅ 확인 포인트"
    `RESTARTS`가 0으로 유지되면 OK. `Ctrl+C`로 종료.

---

### Step 2. Liveness 실패 → 재시작 확인

30초 후 `/tmp/healthy` 파일을 삭제해서 Probe를 의도적으로 실패시킵니다.

```yaml title="pod-liveness-fail.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 3600"
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

```bash title="터미널"
kubectl apply -f pod-liveness-fail.yaml
kubectl get pod liveness-fail -w
```

!!! success "✅ 확인 포인트"
    30~60초 후 `RESTARTS` 숫자가 1로 증가하면 OK.

이벤트 로그로 재시작 원인을 확인합니다:

```bash title="터미널"
kubectl describe pod liveness-fail
```

```text title="출력 예시 — Events 섹션"
Events:
  Warning  Unhealthy  Liveness probe failed: ...
  Normal   Killing    Stopping container app
  Normal   Started    Started container app
```

---

### 정리

```bash title="터미널"
kubectl delete pod liveness-ok liveness-fail
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod가 계속 재시작됨 | `kubectl describe pod` → Events에서 Liveness 실패 원인 확인 |
| 정상 앱인데 재시작됨 | `initialDelaySeconds` 늘리거나 Startup Probe 추가 |

---

## 다음 단계

[:material-arrow-left: 환경 세팅](environment-setup.md){ .md-button }
[Probe 2 · Readiness :material-arrow-right:](probe-readiness.md){ .md-button .md-button--primary }
