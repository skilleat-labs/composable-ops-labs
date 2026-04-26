# 미니 프로젝트 평가

!!! info "소요 시간 1시간 ~ 1시간 30분"

---

!!! quote "김팀장"
    *"실습 때 한 걸 다시 해봐. 근데 이번엔 다른 서비스야. 똑같이 할 수 있으면 네가 진짜 이해한 거야."*

---

## 개요

실습에서 구성한 CI/CD 파이프라인을 **새로운 서비스**에 처음부터 적용합니다.

```
[주어진 것]
hanbat-notification-api 레포 (Dockerfile + k8s yaml 완성)
.github/workflows/ 비어있음

[해야 할 것]
OIDC 설정 → ci.yml 작성 → ArgoCD 연결 → git push → 자동 배포 확인
```

**코드를 작성하지 않습니다.** CI/CD 파이프라인 구성 능력을 평가합니다.

---

## 준비

**1. 레포 Fork**

[skilleat-labs/hanbat-notification-api](https://github.com/skilleat-labs/hanbat-notification-api) 를 본인 GitHub 계정으로 Fork합니다.

**2. 클러스터 초기화**

기존 AKS 클러스터를 삭제하고 새로 생성합니다.

```bash title="터미널"
az aks delete --resource-group $RG --name $AKS_NAME --yes --no-wait
```

삭제되는 동안 ACR은 그대로 사용해도 됩니다.

!!! warning "ACR은 삭제하지 마세요"
    기존 ACR을 재사용합니다. ACR 이름과 Secrets은 그대로 유지하세요.

---

## 과제

### Task 1. AKS + ArgoCD 재설치 (20점)

실습 가이드 Phase 4-1을 참고하여 새 AKS 클러스터에 ArgoCD를 설치합니다.

**제출**: `kubectl get nodes` 출력 스크린샷

---

### Task 2. GitHub Actions CI 파이프라인 구성 (30점)

Fork한 레포의 `.github/workflows/ci.yml` 을 직접 작성합니다.

조건:
- OIDC 인증 (비밀키 방식 사용 금지)
- `notification-api` 이미지 빌드 → ACR 푸시
- `k8s/deployment.yaml` 이미지 태그 자동 업데이트
- 매니페스트 변경사항 자동 커밋 (`[skip ci]`)

**제출**: GitHub Actions 성공 스크린샷 + `ci.yml` 파일

---

### Task 3. ArgoCD Application 등록 및 배포 (30점)

`k8s/argocd-application.yaml` placeholder를 교체하고 ArgoCD에 등록합니다.

**제출**: ArgoCD UI에서 **Synced + Healthy** 상태 스크린샷

---

### Task 4. git push → 자동 배포 확인 (20점)

`app/main.py`의 `/health` 응답에 `"version": "v1.1"` 필드를 추가하고 push합니다.

```python title="app/main.py (수정)"
@app.get("/health")
async def health():
    return {"status": "ok", "service": "notification-api", "version": "v1.1"}
```

GitHub Actions → ArgoCD 자동 Sync → 새 버전 배포까지 확인합니다.

**제출**: 배포 전후 ArgoCD UI 스크린샷 + 아래 명령어 결과

```bash title="터미널"
kubectl port-forward svc/notification-api 8889:8080 -n hanbat-${GITHUB_USER} &
curl http://localhost:8889/health
```

---

## 채점 기준

| Task | 배점 | 만점 조건 |
|------|------|-----------|
| Task 1. AKS + ArgoCD 재설치 | 20점 | 노드 Ready + ArgoCD UI 접속 가능 |
| Task 2. CI 파이프라인 구성 | 30점 | Actions 성공 + 태그 자동 업데이트 커밋 확인 |
| Task 3. ArgoCD 배포 | 30점 | Synced + Healthy |
| Task 4. 자동 배포 확인 | 20점 | `"version": "v1.1"` 응답 확인 |
| **합계** | **100점** | |

!!! success "70점 달성 기준"
    Task 1~3 완료 시 80점입니다.

---

## 제출 방법

1. Fork한 레포 URL
2. 각 Task 스크린샷을 하나의 문서(PPT/PDF/Notion)로 정리
3. 강사에게 제출

---

## 다음 단계

[:material-arrow-left: 평가 개요](index.md){ .md-button }
