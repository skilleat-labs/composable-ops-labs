# Phase 4-3 · GitOps 자동 배포 체험

!!! info "예상 소요 30분"

---

!!! quote "김팀장"
    *"이제 git push 한 번만 해봐. 내가 뭘 시키는지 봐."*

---

## 이번 단계의 목표

Phase 3~4에서 만든 전체 파이프라인을 처음부터 끝까지 한 번에 체험합니다.

```
코드 수정 → git push
    │
    ▼ (~1분)
[GitHub Actions]
    빌드 → ACR 푸시 → k8s/*.yaml 태그 업데이트 → [skip ci] 커밋
    │
    ▼ (~2~3분, ArgoCD polling)
[ArgoCD]
    Git 변경 감지 → AKS에 새 Pod 배포 → 기존 Pod 종료
    │
    ▼
새 버전 서비스 정상 응답
```

**이 과정에서 `kubectl apply`를 단 한 번도 직접 치지 않습니다.**

---

## Step 1. 코드 변경

`order-api/app.py`의 health 엔드포인트 응답을 수정합니다.

```bash title="터미널"
cd ~/hanbat-order-app-s2
```

`order-api/app.py`에서 이 부분을 찾아 버전을 수정합니다.

```python title="order-api/app.py (수정 전)"
@app.get("/health")
async def health():
    return {"status": "ok", "service": "order-api"}
```

```python title="order-api/app.py (수정 후)"
@app.get("/health")
async def health():
    return {"status": "ok", "service": "order-api", "version": "v2"}
```

---

## Step 2. Push

```bash title="터미널"
git add order-api/app.py
git commit -m "feat: add version field to health endpoint"
git push origin main
```

---

## Step 3. GitHub Actions 실행 관찰

브라우저에서 **GitHub → Actions 탭**을 엽니다.

```
CI 실행 중...
  ✅ 코드 체크아웃
  ✅ Azure 로그인 (OIDC)
  ✅ 이미지 태그 설정    → TAG=abc1234
  ✅ order-api 빌드 및 푸시
  ✅ payment-api 빌드 및 푸시
  ✅ order-web 빌드 및 푸시
  ✅ k8s 매니페스트 이미지 태그 업데이트
  ✅ 매니페스트 변경사항 자동 커밋  → [skip ci]
```

완료까지 약 3~5분 걸립니다.

Actions가 완료되면 GitHub 레포의 커밋 히스토리를 확인합니다.

```text title="커밋 히스토리 예시"
abc1234  feat: add version field to health endpoint    ← 본인 커밋
def5678  ci: update image tags to abc1234 [skip ci]    ← Actions 자동 커밋
```

---

## Step 4. ArgoCD 자동 Sync 관찰

ArgoCD UI(`https://<ArgoCD_IP>`)에서 본인 Application을 엽니다.

Actions 완료 후 약 1~3분 이내에 ArgoCD가 변경을 감지합니다.

```
Synced → OutOfSync (새 태그 감지)
       → Syncing   (클러스터에 반영 중)
       → Synced    (완료)
```

UI에서 **App 상세 화면**을 보면 Pod이 교체되는 과정이 애니메이션으로 보입니다.

```
order-api-7d8f9b4c6-xk9pq  (기존 — 종료됨)
order-api-8e9g0c5d7-yl0qr  (신규 — 기동 중)
```

!!! success "GitOps 성공"
    새 Pod이 뜨고 기존 Pod이 사라지는 걸 **kubectl 한 번 안 치고** 봤습니다.
    이게 GitOps입니다.

---

## Step 5. 새 버전 확인

AKS에 배포된 API를 직접 호출해봅니다.

먼저 ingress IP를 확인합니다.

```bash title="터미널"
kubectl get ingress -n hanbat-${GITHUB_USER}
```

```text title="출력 예시"
NAME      CLASS   HOSTS   ADDRESS          PORTS
hanbat    nginx   *       20.xxx.xxx.xxx   80
```

!!! note "왜 port-forward로 확인하나요?"
    `k8s/ingress.yaml`의 라우팅 규칙을 보면:

    - `/` → order-web (포트 80)
    - `/api` → order-api (포트 8080)

    `curl http://<IP>/health` 는 `/` 규칙에 매칭되어 **order-web**으로 전달됩니다.
    order-api의 `/health`에 접근하려면 `/api/health` 경로가 있어야 하는데, 현재 ingress에는 그 규칙이 없습니다.
    따라서 order-api를 직접 테스트할 때는 port-forward로 클러스터 내부 서비스에 바로 연결합니다.

order-api의 health 엔드포인트를 port-forward로 직접 확인합니다.

```bash title="터미널"
kubectl port-forward svc/order-api 8889:8080 -n hanbat-${GITHUB_USER} &
curl http://localhost:8889/health
```

```json title="출력 예시 (새 버전)"
{"status": "ok", "service": "order-api", "version": "v2"}
```

`"version": "v2"` 가 보이면 새 버전이 배포된 겁니다.

!!! success "✅ 확인 포인트"
    - GitHub Actions 가 성공으로 완료됐다
    - ArgoCD가 자동으로 Sync됐다 (UI에서 확인)
    - `/health` 응답에 `"version": "v2"` 가 포함됐다
    - **그 사이 `kubectl apply` 를 한 번도 치지 않았다**

---

## GitOps의 또 다른 능력 — 자동 복구(Self-Heal)

실수로 Pod을 직접 삭제해봅니다.

**ArgoCD UI를 먼저 열어두고** Pod을 삭제하세요. UI에서 Pod이 사라졌다가 즉시 새로 생기는 과정을 실시간으로 볼 수 있습니다.

```bash title="터미널"
kubectl delete pod -l app=order-api -n hanbat-${GITHUB_USER}
```

```bash title="터미널"
kubectl get pods -n hanbat-${GITHUB_USER} -w
```

```text title="출력 예시"
order-api-8e9g0c5d7-yl0qr   Terminating
order-api-8e9g0c5d7-zn1rs   ContainerCreating   ← 즉시 새 Pod 생성
order-api-8e9g0c5d7-zn1rs   Running
```

ArgoCD의 `selfHeal: true` 설정이 Deployment를 Git 상태(replicas: 1)로 자동 복구합니다.

!!! quote "박대리"
    *"Pod 지웠는데 혼자 살아났어요?"*

!!! quote "김팀장"
    *"Git이 '정답'이야. ArgoCD는 클러스터 상태를 계속 Git과 비교하면서 다르면 맞춰줘. 누군가 실수로 건드려도 자동으로 원상복구 돼."*

---

## 정리

Phase 3~4에서 완성한 것:

| 단계 | 자동화 내용 |
| --- | --- |
| **Phase 3** | git push → 이미지 빌드 → ACR 푸시 → 매니페스트 태그 업데이트 |
| **Phase 4** | 매니페스트 변경 → ArgoCD Sync → AKS 배포 |
| **합계** | 코드 한 줄 수정 → push → **자동으로 서비스에 반영** |

이대리가 매 배포마다 40분씩 쓰던 시간이 **`git push` 한 줄**로 줄었습니다.

!!! success "✅ Phase 4 완료 체크리스트"
    - [ ] AKS kubeconfig 설정 완료
    - [ ] 본인 네임스페이스 생성 완료
    - [ ] ArgoCD Application 등록 완료 (Synced + Healthy)
    - [ ] git push → 자동 배포 end-to-end 체험 완료
    - [ ] Self-Heal 동작 확인 완료

---

## 다음 단계

[:material-arrow-left: Phase 4-2 · ArgoCD Application 등록](application-register.md){ .md-button }
[Phase 5 · Agent란 무엇인가 :material-arrow-right:](../phase-5/index.md){ .md-button .md-button--primary }
