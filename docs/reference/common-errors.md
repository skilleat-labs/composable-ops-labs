# 자주 만나는 오류

실습 중 학생들이 자주 막히는 지점과 해결법을 모았습니다. 문제가 생기면 이 페이지를 먼저 찾아보세요.

---

## Phase 0 · 환경 세팅

??? failure "`ssh: Connection refused`"
    - VM이 **실행 중**인지 Azure Portal에서 확인
    - **공용 IP**가 VM 재시작 후 바뀌었을 수 있음
    - NSG에서 포트 22가 허용되어 있는지 확인
    - 디버그: `ssh -v labuser@<IP>`

??? failure "`az: command not found`"
    ```bash
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    ```

??? failure "`kubectl: command not found`"
    ```bash
    sudo az aks install-cli
    ```

??? failure "`kubectl get nodes` → Unauthorized"
    강사에게 본인 Object ID 전달 후 Cluster User Role 부여 요청.
    ```bash
    az ad signed-in-user show --query id -o tsv
    ```

??? failure "Azure OpenAI 호출 → `AuthenticationError`"
    - `echo $AOAI_API_KEY`로 환경변수 로드 확인
    - `source ~/.bashrc` 다시 실행
    - API 키 복사 시 앞뒤 공백 없는지 확인

---

## Phase 2 · Retry / Circuit Breaker

??? failure "`ModuleNotFoundError: No module named 'pybreaker'`"
    ```bash
    pip3 install --user pybreaker tenacity
    ```

??? failure "Circuit Breaker가 열리지 않음"
    - `failure_threshold` 값이 너무 크게 설정되어 있을 수 있음 (기본 5 → 3으로 낮춰 테스트)
    - 예외 타입이 CB가 인식하는 타입인지 확인 (HTTP 5xx vs 타임아웃)

---

## Phase 3 · GitHub Actions

??? failure "Workflow가 실행되지 않음"
    - `.github/workflows/` 디렉터리 경로 정확한지 (슬래시 2개)
    - YAML 문법 오류 확인 ([yamllint.com](https://www.yamllint.com))
    - 브랜치 트리거가 `main`으로 되어 있는지

??? failure "`Error: Login failed with Error: Using auth-type: SERVICE_PRINCIPAL`"
    OIDC 설정 누락. Federated Credential이 저장소/브랜치/환경 모두 정확히 매칭되는지 확인.
    ```bash
    az ad app federated-credential list --id <APP_ID>
    ```

??? failure "ACR push → `denied: requested access to the resource is denied`"
    - `acrpush` 역할 부여 확인:
      ```bash
      az role assignment create \
        --assignee <APP_ID> \
        --role AcrPush \
        --scope $(az acr show -n <ACR_NAME> --query id -o tsv)
      ```

??? failure "`git-auto-commit-action` 무한 루프"
    Workflow 트리거에 `paths-ignore: ['k8s/**']` 추가하거나 커밋 메시지에 `[skip ci]` 포함.

---

## Phase 4 · ArgoCD

??? failure "`helm install argocd` 실패"
    ```bash
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
    kubectl create namespace argocd
    helm install argocd argo/argo-cd --namespace argocd
    ```

??? failure "ArgoCD Web UI 접속 불가"
    LoadBalancer IP 할당 대기 중일 수 있음. `kubectl get svc -n argocd argocd-server`로 EXTERNAL-IP 확인. 당장 테스트가 필요하면:
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8443:443
    # 브라우저에서 https://localhost:8443
    ```

??? failure "Application이 `OutOfSync`에서 멈춤"
    - GitHub 레포가 Public인지 또는 Credential이 등록되어 있는지 확인
    - `kubectl -n argocd logs deployment/argocd-repo-server` 로그 확인
    - UI에서 **REFRESH** → **SYNC** 수동 실행

??? failure "Pod 배포 시 `ImagePullBackOff`"
    AKS에 ACR이 연결되어 있지 않음:
    ```bash
    az aks update -g <RG> -n <CLUSTER> --attach-acr <ACR_NAME>
    ```

---

## Phase 5–6 · Agent / Azure OpenAI

??? failure "`openai.RateLimitError: 429`"
    TPM(분당 토큰) 초과. 강사에게 공용 리소스 TPM 확인 요청 또는 다른 학생과 호출 시점 분산.

??? failure "`openai.NotFoundError: Resource not found`"
    - `AOAI_DEPLOYMENT` 값이 배포 이름과 정확히 일치하는지 (대소문자 구분)
    - `AOAI_API_VERSION`이 배포 지원 버전인지 확인 (`2024-08-01-preview` 또는 최신)

??? failure "Tool calling 결과가 반영되지 않음"
    - `messages` 배열에 `role: tool`로 추가했는지 확인
    - `tool_call_id`를 Assistant 응답에서 그대로 복사했는지 확인

??? failure "`subprocess.CalledProcessError` (kubectl 실행 시)"
    Agent 프로세스가 실행 중인 쉘의 kubeconfig 권한 확인. `kubectl get pods -A`가 먼저 직접 동작해야 함.

---

## 일반

??? failure "실습 중 VM이 멈추거나 느려짐"
    `top`으로 프로세스 확인, Docker 컨테이너 정리:
    ```bash
    docker ps
    docker container prune
    docker image prune
    ```

??? failure "Git push 실패 (`remote: Support for password authentication was removed`)"
    GitHub는 비밀번호 인증을 지원하지 않습니다. **Personal Access Token**을 비밀번호 자리에 입력하세요.
    - [Settings → Developer settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens)
    - 권한: `repo` 체크

---

## 그래도 해결이 안 되면

1. `kubectl get events -A --sort-by=.lastTimestamp | tail -20` 실행 결과와 에러 메시지를 모아
2. 강사에게 **[어떤 Phase] + [실행한 명령] + [에러 출력]** 3요소를 한 번에 전달

---

## 다음 단계

[:material-arrow-left: 제출 가이드](../evaluation/submission-guide.md){ .md-button }
[유용한 명령어 모음 :material-arrow-right:](useful-commands.md){ .md-button .md-button--primary }
