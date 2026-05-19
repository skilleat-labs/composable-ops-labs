# Probe 2 · Readiness Probe

!!! info "예상 소요 15분"

---

!!! quote "이대리"
    *"Pod가 Running이라고 바로 트래픽 보내면 안 돼. DB 연결이 아직 안 됐을 수도 있잖아. Readiness Probe는 '진짜 준비됐냐'를 확인하는 거야."*

---

## 이론

### Readiness Probe란?

Liveness Probe가 "살아있냐?"를 물어본다면, Readiness Probe는 **"트래픽 받을 준비됐냐?"** 를 물어봅니다.

Pod가 `Running` 상태여도 다음 상황에서는 트래픽을 보내면 안 됩니다:

- 앱이 DB 연결을 아직 맺는 중
- 캐시 워밍업이 진행 중
- 배포 직후 초기화 작업 중

### Liveness와의 결정적 차이

```
Liveness 실패  →  컨테이너 재시작
Readiness 실패 →  Service Endpoint에서 제거 (재시작 없음)
```

Pod는 살아있지만 Service가 해당 Pod로 트래픽을 보내지 않습니다.
준비가 되면 자동으로 Endpoint에 다시 추가됩니다.

### 동작 구조

```text title="Readiness Probe 동작"
[Pod A]  READY 1/1  ──┐
[Pod B]  READY 1/1  ──┼──▶  Service  ──▶  사용자 요청
[Pod C]  READY 0/1  ✗ ┘     (Pod C는 Endpoint 제외)
```

Readiness가 실패한 Pod C는 Service의 Endpoint 목록에서 빠집니다.
롤링 업데이트 중 새 Pod가 완전히 준비되기 전까지 구 Pod로만 트래픽이 가는 것도 이 원리입니다.

---

## 실습

### Step 1. Readiness Probe가 있는 Deployment 배포

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
    3개 Pod가 `READY 1/1`이 되면 Endpoints에 IP 3개가 표시됩니다.

---

### Step 2. Readiness 실패 → Endpoint 제외 확인

Pod 하나의 nginx를 강제로 중단해서 Readiness를 실패시킵니다.

```bash title="터미널"
# Pod 이름 확인
kubectl get pods -l app=readiness-app

# 첫 번째 Pod의 nginx 중단
kubectl exec <pod이름> -- nginx -s stop
```

```bash title="터미널"
kubectl get pods -l app=readiness-app
kubectl get endpoints readiness-svc
```

!!! success "✅ 확인 포인트"
    해당 Pod는 `Running`이지만 `READY 0/1` 상태가 됩니다.
    `kubectl get endpoints`에서 해당 Pod의 IP가 사라진 것을 확인하세요.

!!! note "Liveness와 비교"
    Liveness였다면 Pod가 재시작됩니다.
    Readiness는 Pod를 재시작하지 않고, Service에서만 잠깐 빼놓습니다.

---

### 정리

```bash title="터미널"
kubectl delete -f deploy-readiness.yaml
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod는 Running인데 READY=0/1 | Readiness 실패 — 앱이 아직 준비 안 됨, `kubectl describe pod` 확인 |
| Endpoint에 Pod IP가 없음 | `kubectl get endpoints <svc이름>`으로 Readiness 상태 확인 |

---

## 다음 단계

[:material-arrow-left: Probe 1 · Liveness](probe-liveness.md){ .md-button }
[Probe 3 · Startup :material-arrow-right:](probe-startup.md){ .md-button .md-button--primary }
