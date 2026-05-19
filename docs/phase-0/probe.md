# K8s 1 · Probe 설정

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"AKS 클러스터 올리기 전에 기억 좀 되살리자. Pod가 살아있는지, 트래픽 받을 준비가 됐는지 — 이걸 K8s가 어떻게 아는지 알아야 운영이 돼."*

---

## 개념 정리

| Probe | 실패 시 동작 | 주요 용도 |
|-------|------------|---------|
| **Liveness** | 컨테이너 **재시작** | 교착 상태·무한 루프 감지 |
| **Readiness** | Service 엔드포인트에서 **제거** (재시작 없음) | DB 연결 전 트래픽 차단 |
| **Startup** | 성공 전까지 Liveness/Readiness **비활성화** | 느리게 뜨는 앱 보호 |

---

## 1) Liveness Probe — 살아있냐?

컨테이너가 **교착 상태**에 빠졌지만 프로세스는 살아있을 때 Liveness Probe가 잡아냅니다.

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
        initialDelaySeconds: 5   # 컨테이너 시작 후 첫 Probe까지 대기
        periodSeconds: 5         # 이후 반복 주기
        failureThreshold: 3      # 연속 3회 실패 시 재시작
```

타이밍 흐름:

```text title="Probe 타이밍"
t=0s   컨테이너 시작
       ↕ initialDelaySeconds: 5
t=5s   첫 번째 Probe → 성공
       ↕ periodSeconds: 5
t=10s  두 번째 Probe → 성공
       ...5초마다 반복...
```

!!! note "failureThreshold: 3 의 의미"
    한 번 실패해도 바로 재시작하지 않습니다.
    **연속 3회 실패** = 최대 `5 × 3 = 15초` 감지 후 재시작.

```bash title="터미널"
kubectl apply -f pod-liveness-ok.yaml
kubectl get pod liveness-ok -w
```

!!! success "✅ 확인 포인트"
    `RESTARTS`가 0으로 유지되면 OK.

### Liveness 실패 → 재시작 확인

```yaml title="pod-liveness-fail.yaml"
# 30초 후 /tmp/healthy 파일 삭제 → Probe 실패 → 재시작
command:
  - sh
  - -c
  - "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 3600"
```

```bash title="터미널"
kubectl apply -f pod-liveness-fail.yaml
kubectl get pod liveness-fail -w
```

!!! success "✅ 확인 포인트"
    30~60초 후 `RESTARTS` 숫자가 증가하면 OK.

---

## 2) Readiness Probe — 트래픽 받을 준비됐냐?

Pod가 `Running`이어도 앱 초기화가 덜 됐으면 트래픽을 보내면 안 됩니다.
Readiness Probe는 실패하면 **재시작 없이 Service Endpoint에서만 제외**합니다.

```yaml title="deploy-readiness.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-app
  template:
    metadata:
      labels:
        app: readiness-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
spec:
  selector:
    app: readiness-app
  ports:
    - port: 80
      targetPort: 80
```

```bash title="터미널"
kubectl apply -f deploy-readiness.yaml
kubectl get pods -l app=readiness-app
kubectl get endpoints readiness-svc
```

!!! success "✅ 확인 포인트"
    3개 Pod가 `READY 1/1`이 되면 Endpoints에 IP 3개가 나타납니다.

### Pod 하나를 강제로 Readiness 실패시키기

```bash title="터미널"
kubectl exec <pod이름> -- nginx -s stop
kubectl get pods -l app=readiness-app
kubectl get endpoints readiness-svc
```

!!! success "✅ 확인 포인트"
    Pod는 `Running`이지만 `READY 0/1` — Endpoints에서 해당 IP가 사라집니다.

---

## 3) Startup Probe — 뜨는 데 오래 걸리는 앱 보호

JVM, Spring Boot처럼 초기화가 긴 앱은 Liveness가 너무 일찍 체크해서 멀쩡한 앱을 죽입니다.
Startup Probe가 **성공하기 전까지 Liveness/Readiness를 잠가둡니다**.

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

!!! success "✅ 확인 포인트"
    처음 20초간 Startup Probe가 실패하지만 Pod는 재시작되지 않습니다.
    20초 후 `READY 1/1` 전환 확인.

---

## 정리 및 리소스 삭제

```bash title="터미널"
kubectl delete pod liveness-ok liveness-fail startup-app
kubectl delete -f deploy-readiness.yaml
```

### 실무 설정 가이드

| Probe | 권장 |
|-------|------|
| Startup | `failureThreshold × periodSeconds` ≥ 최대 초기화 시간 |
| Readiness | `periodSeconds` 5~10s, 빠르게 감지 |
| Liveness | `initialDelaySeconds` 충분히, `failureThreshold` 3~5 |

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod가 계속 재시작 | `kubectl describe pod` → Liveness 실패 원인 |
| Running인데 READY=0/1 | Readiness 실패 — 앱 아직 준비 안 됨 |
| 정상 앱이 초기화 전에 재시작 | Startup Probe 추가 또는 `initialDelaySeconds` 늘리기 |

---

## 다음 단계

[:material-arrow-left: 환경 세팅](environment-setup.md){ .md-button }
[K8s 2 · Resource & QoS :material-arrow-right:](resource-qos.md){ .md-button .md-button--primary }
