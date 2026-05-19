# K8s 개념 복기 3 · HPA (Horizontal Pod Autoscaler)

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"트래픽이 몰리면 Pod 수를 자동으로 늘리고, 빠지면 다시 줄인다. 이게 HPA야. 단, 전제 조건이 있어 — Metrics Server랑 requests 설정."*

---

## 개념 정리

HPA는 메트릭(CPU, Memory 등)을 보고 Deployment의 Replica 수를 자동 조절합니다.

```text title="HPA 동작 공식"
목표 Replica = ceil(현재 Replica × (현재 메트릭 / 목표 메트릭))

예시:
  현재 Replica: 2
  현재 CPU:     90%
  목표 CPU:     50%
  → ceil(2 × 90/50) = ceil(3.6) = 4
```

### HPA 전제 조건 체크리스트

- [ ] Deployment에 `resources.requests.cpu` 설정 (없으면 HPA가 메트릭 계산 불가)
- [ ] Metrics Server 설치 및 정상 동작
- [ ] HPA의 `scaleTargetRef`가 실제 Deployment 이름과 일치

---

## 1) Metrics Server 설치

```bash title="터미널"
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Docker Desktop / Rancher Desktop 환경 추가 설정

```bash title="터미널"
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

```bash title="터미널"
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl top nodes
kubectl top pods -A
```

!!! success "✅ 확인 포인트"
    `kubectl top nodes`에서 CPU/Memory 수치가 나오면 OK.

---

## 2) HPA 대상 Deployment 배포

```yaml title="deploy-hpa-target.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-app
  template:
    metadata:
      labels:
        app: hpa-app
    spec:
      containers:
        - name: app
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m       # HPA 동작에 필수
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-svc
spec:
  selector:
    app: hpa-app
  ports:
    - port: 80
      targetPort: 80
```

!!! note "Apple Silicon (M1/M2/M3) 대체 이미지"
    `no matching manifest for linux/arm64` 오류 시:
    ```yaml
    image: registry.k8s.io/e2e-test-images/resource-consumer:1.10
    command: ["/consume"]
    args: ["--port=80"]
    ```

```bash title="터미널"
kubectl apply -f deploy-hpa-target.yaml
```

---

## 3) HPA 생성

=== "kubectl 명령어"

    ```bash title="터미널"
    kubectl autoscale deployment hpa-app \
      --cpu-percent=50 \
      --min=1 \
      --max=10
    ```

=== "YAML 파일"

    ```yaml title="hpa.yaml"
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: hpa-app
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: hpa-app
      minReplicas: 1
      maxReplicas: 10
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 50
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60   # 1분 안정화 후 축소
    ```

    ```bash title="터미널"
    kubectl apply -f hpa.yaml
    ```

```bash title="터미널"
kubectl get hpa -w
```

!!! success "✅ 확인 포인트"
    `TARGETS`의 왼쪽 값이 `<unknown>`에서 실제 수치(예: `12%/50%`)로 바뀌면 OK. 1~2분 소요.

---

## 4) 부하 발생 → 자동 확장 확인

터미널을 3개 열어서 동시에 진행합니다.

```bash title="터미널 1 — HPA 상태 감시"
kubectl get hpa -w
```

```bash title="터미널 2 — Pod 수 감시"
kubectl get pods -l app=hpa-app -w
```

```bash title="터미널 3 — 부하 발생"
kubectl run load-generator \
  --image=busybox:1.36-musl \
  --restart=Never \
  -- sh -c "while true; do wget -q -O- http://hpa-svc & done"
```

```text title="출력 예시 — 자동 확장"
NAME      REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS
hpa-app   Deployment/hpa-app   12%/50%    1         10        1
hpa-app   Deployment/hpa-app   85%/50%    1         10        1
hpa-app   Deployment/hpa-app   85%/50%    1         10        3
hpa-app   Deployment/hpa-app   72%/50%    1         10        5
```

!!! note "HPA 반응 속도"
    HPA는 약 15초마다 메트릭을 확인하고 replica 수를 조정합니다.

---

## 5) 부하 제거 → 자동 축소 확인

```bash title="터미널"
kubectl delete pod load-generator
```

!!! success "✅ 확인 포인트"
    `stabilizationWindowSeconds`(60초) 후 Replica가 1로 축소됩니다.

### 관찰 기록표

| 시점 | CPU 사용률 | Replica 수 |
|------|-----------|-----------|
| 부하 발생 전 | | 1 |
| 부하 발생 후 1분 | | |
| 부하 발생 후 3분 | | |
| 부하 제거 후 2분 | | |
| 최종 안정 | | 1 |

---

## 정리 및 리소스 삭제

```bash title="터미널"
kubectl delete hpa hpa-app
kubectl delete -f deploy-hpa-target.yaml
kubectl delete pod load-generator --ignore-not-found
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| TARGETS 계속 `<unknown>` | Metrics Server Pod 상태 및 `--kubelet-insecure-tls` 패치 확인 |
| CPU 100%인데 Scale-up 안 됨 | `resources.requests.cpu` 설정 여부 확인 |
| Scale-down이 느림 | `stabilizationWindowSeconds` 기본값 300초 — 실습에선 60초로 설정 |
| `kubectl top` 오류 | Metrics Server Running 여부 확인 |

---

## 다음 단계

[:material-arrow-left: K8s 복기 2 · Resource & QoS](resource-qos.md){ .md-button }
[Phase 1 · 연쇄 장애의 밤 :material-arrow-right:](../phase-1/index.md){ .md-button .md-button--primary }
