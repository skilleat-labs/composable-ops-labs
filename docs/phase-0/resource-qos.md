# K8s 2 · Resource requests/limits & QoS

!!! info "예상 소요 30분"

---

!!! quote "이대리"
    *"requests랑 limits 헷갈리지? 간단해. requests는 '이 정도는 보장해줘', limits는 '이 이상은 절대 안 줘'. 그리고 이 둘의 관계가 QoS Class를 결정해."*

---

## 개념 정리

| 설정 | 역할 | 초과 시 |
|------|------|--------|
| `requests.cpu` | 스케줄러 배치 기준 (최소 보장) | CPU throttling — 강제 종료 없음 |
| `requests.memory` | 스케줄러 배치 기준 (최소 보장) | — |
| `limits.cpu` | 최대 CPU 사용량 | CPU throttling |
| `limits.memory` | 최대 메모리 사용량 | **OOMKilled** — 컨테이너 강제 종료 |

!!! warning "메모리는 CPU와 다릅니다"
    CPU는 초과해도 속도가 느려질 뿐이지만, 메모리는 limits를 초과하면 컨테이너가 **즉시 강제 종료(OOMKilled)**됩니다.

---

## 1) QoS Class — 세 가지 등급

K8s는 requests/limits 설정에 따라 Pod에 자동으로 QoS Class를 부여합니다.
노드 자원이 부족할 때 **BestEffort → Burstable → Guaranteed 순으로** 먼저 퇴거(Eviction)됩니다.

### BestEffort — requests/limits 없음

```yaml title="pod-besteffort.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-besteffort
spec:
  containers:
    - name: app
      image: nginx:1.25
      # resources 설정 없음
```

```bash title="터미널"
kubectl apply -f pod-besteffort.yaml
kubectl get pod pod-besteffort -o yaml | grep -A 2 "qosClass"
```

결과: `qosClass: BestEffort`

---

### Burstable — requests < limits

```yaml title="pod-burstable.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-burstable
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
```

결과: `qosClass: Burstable`

---

### Guaranteed — requests == limits

```yaml title="pod-guaranteed.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-guaranteed
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: 200m
          memory: 256Mi
```

결과: `qosClass: Guaranteed`

### 세 Pod 한눈에 비교

```bash title="터미널"
kubectl get pods -o custom-columns=\
"NAME:.metadata.name,QoS:.status.qosClass,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory"
```

---

## 2) OOMKilled 직접 확인

limits.memory를 20Mi로 설정하고 50MB를 쓰려는 Pod를 만들어봅니다.

```yaml title="pod-oom.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: pod-oom
spec:
  containers:
    - name: app
      image: busybox:1.36-musl
      command:
        - sh
        - -c
        - |
          dd if=/dev/zero of=/dev/shm/waste bs=1M count=50 && sleep 3600
      resources:
        requests:
          memory: 10Mi
        limits:
          memory: 20Mi   # 20MB 제한이지만 50MB 시도 → OOMKilled
```

```bash title="터미널"
kubectl apply -f pod-oom.yaml
kubectl get pod pod-oom -w
```

!!! success "✅ 확인 포인트"
    잠시 후 `OOMKilled` 상태 확인.
    `kubectl describe pod pod-oom` → `Reason: OOMKilled`

---

## 3) ResourceQuota — Namespace 단위 총량 제한

팀별 네임스페이스에 자원 총량을 제한할 때 사용합니다.

```yaml title="resourcequota.yaml"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "4"
    limits.memory: 2Gi
    count/pods: "10"
```

```bash title="터미널"
kubectl apply -f resourcequota.yaml
kubectl describe resourcequota lab-quota
```

---

## 4) LimitRange — 기본값 자동 적용

requests/limits 없이 Pod를 만들면 기본값을 자동으로 채워줍니다.
ResourceQuota가 있을 때 requests 없는 Pod가 생성 거부되는 것을 막아줍니다.

```yaml title="limitrange.yaml"
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-limitrange
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "1"
        memory: 1Gi
      min:
        cpu: 50m
        memory: 64Mi
```

```bash title="터미널"
kubectl apply -f limitrange.yaml
```

---

## 정리 및 리소스 삭제

```bash title="터미널"
kubectl delete pod pod-besteffort pod-burstable pod-guaranteed pod-oom
kubectl delete resourcequota lab-quota
kubectl delete limitrange lab-limitrange
```

### requests 설정 기준

```text title="실무 기준"
requests = 평균 사용량
limits    = 피크 사용량 + 20~30% 여유

예시:
  평균 CPU 150m  → requests.cpu: 150m
  피크 CPU 400m  → limits.cpu: 500m
  평균 Memory 200Mi → requests.memory: 200Mi
  피크 Memory 400Mi → limits.memory: 512Mi
```

### 트러블슈팅

| 증상 | 확인 |
|------|------|
| Pod `Pending` | `kubectl describe node` → Allocatable 리소스 확인 |
| 계속 재시작, `OOMKilled` | `limits.memory` 증가 또는 앱 메모리 누수 점검 |
| 생성 거부 `exceeded quota` | `kubectl describe resourcequota` |

---

## 다음 단계

[:material-arrow-left: K8s 1 · Probe](probe.md){ .md-button }
[K8s 3 · HPA :material-arrow-right:](hpa.md){ .md-button .md-button--primary }
