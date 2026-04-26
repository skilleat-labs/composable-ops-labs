# 유용한 명령어 모음

실습 중 반복해서 쓰는 명령어를 한 페이지에 모았습니다. 북마크 추천.

---

## Azure CLI

=== "로그인 & 구독"

    ```bash
    # 디바이스 코드 로그인 (브라우저 없이)
    az login --use-device-code

    # 현재 계정 확인
    az account show

    # 구독 변경
    az account set --subscription "<subscription_id>"
    ```

=== "AKS"

    ```bash
    # kubeconfig 가져오기
    az aks get-credentials -g <RG> -n <CLUSTER>

    # 클러스터에 ACR 연결
    az aks update -g <RG> -n <CLUSTER> --attach-acr <ACR_NAME>

    # 노드 크기 정보
    az aks show -g <RG> -n <CLUSTER> --query agentPoolProfiles
    ```

=== "ACR"

    ```bash
    # 로그인
    az acr login -n <ACR_NAME>

    # 이미지 목록
    az acr repository list -n <ACR_NAME>

    # 특정 이미지의 태그 목록
    az acr repository show-tags -n <ACR_NAME> --repository order-api

    # 원격 빌드 (로컬 Docker 불필요)
    az acr build -t order-api:v1 -r <ACR_NAME> .
    ```

---

## kubectl

=== "조회"

    ```bash
    # 모든 네임스페이스의 Pod
    kubectl get pods -A

    # 특정 네임스페이스
    kubectl get pods -n hanbat-<username>

    # 상세 정보
    kubectl describe pod <POD_NAME> -n <NS>

    # 로그
    kubectl logs <POD_NAME> -n <NS>
    kubectl logs <POD_NAME> -n <NS> -f                # 실시간
    kubectl logs <POD_NAME> -n <NS> --previous        # 이전 컨테이너 로그
    kubectl logs <POD_NAME> -n <NS> --tail=50         # 최근 50줄

    # 이벤트 (장애 분석에 매우 유용)
    kubectl get events -n <NS> --sort-by=.lastTimestamp

    # 전체 네임스페이스 이벤트 최신순
    kubectl get events -A --sort-by=.lastTimestamp | tail -30
    ```

=== "디버깅"

    ```bash
    # Pod에 shell 접속
    kubectl exec -it <POD_NAME> -n <NS> -- /bin/sh
    kubectl exec -it <POD_NAME> -n <NS> -- /bin/bash

    # 포트 포워딩
    kubectl port-forward svc/<SVC_NAME> -n <NS> 8080:80

    # Pod의 리소스 사용량
    kubectl top pod -n <NS>
    kubectl top node
    ```

=== "적용 & 삭제"

    ```bash
    # YAML 적용
    kubectl apply -f deployment.yaml
    kubectl apply -f k8s/

    # 삭제
    kubectl delete pod <POD_NAME> -n <NS>
    kubectl delete -f deployment.yaml

    # 즉시 재시작 (Rolling)
    kubectl rollout restart deployment/<DEPLOY_NAME> -n <NS>

    # 롤백
    kubectl rollout undo deployment/<DEPLOY_NAME> -n <NS>
    ```

---

## ArgoCD CLI

```bash
# CLI 로그인 (Web UI 주소와 동일)
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD>

# Application 목록
argocd app list

# 상태 확인
argocd app get <APP_NAME>

# 수동 Sync
argocd app sync <APP_NAME>

# 롤백
argocd app rollback <APP_NAME> <REVISION>

# 로그
argocd app logs <APP_NAME>
```

---

## Docker / Docker Compose

```bash
# 이미지 빌드
docker build -t order-api:v1 ./order-api

# 이미지 태그 추가
docker tag order-api:v1 <ACR>.azurecr.io/order-api:v1

# 푸시
docker push <ACR>.azurecr.io/order-api:v1

# 실행 중 컨테이너 로그
docker logs -f <CONTAINER>

# 모든 컨테이너 정리
docker container prune -f

# 이미지 정리
docker image prune -a -f

# Compose 실행
docker compose up -d
docker compose down
docker compose logs -f <SERVICE>
```

---

## Git

```bash
# 현재 브랜치 확인
git branch --show-current

# 변경사항 확인
git status
git diff

# 단계별 커밋
git add <FILE>
git commit -m "<MESSAGE>"
git push

# 원격 변경사항 가져오기 (팀 작업 시)
git pull --rebase

# 최근 커밋 히스토리 (그래프)
git log --oneline --graph --all -20

# 마지막 커밋 수정
git commit --amend

# 특정 파일만 이전 커밋으로 되돌리기
git checkout HEAD~1 -- <FILE>
```

---

## 부하 테스트 도구

실습 Phase 1에서 장애 재현할 때 사용.

=== "hey (권장)"

    ```bash
    # 설치
    sudo apt install -y hey

    # 30개 동시 요청으로 1000회 전송
    hey -c 30 -n 1000 http://<IP>:8080/api/orders/1

    # 10초 동안 지속 부하
    hey -c 30 -z 10s http://<IP>:8080/api/orders/1
    ```

=== "ab (Apache Bench)"

    ```bash
    # 설치
    sudo apt install -y apache2-utils

    # 동시 30개로 1000회
    ab -c 30 -n 1000 http://<IP>:8080/api/orders/1
    ```

=== "curl 반복"

    ```bash
    # 간단 루프
    for i in {1..100}; do
      curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" http://<IP>:8080/api/orders/1
    done
    ```

---

## 환경변수 관리

```bash
# 현재 세션에서 확인
env | grep AOAI

# .bashrc에 영구 저장
echo 'export MY_VAR="value"' >> ~/.bashrc
source ~/.bashrc

# 특정 스크립트 실행 시에만 적용
MY_VAR=value python3 script.py

# .env 파일 로드
set -a; source .env; set +a
```

---

## 한밭푸드 실습 전용 단축 명령

시즌 2 실습 중 자주 쓰는 명령을 `~/.bashrc`에 alias로 등록해두면 편합니다.

```bash
cat >> ~/.bashrc << 'EOF'

# 한밭푸드 시즌 2 단축 명령
alias hp='kubectl -n hanbat-$USER'
alias hplog='kubectl -n hanbat-$USER logs -f'
alias hpev='kubectl -n hanbat-$USER get events --sort-by=.lastTimestamp'
alias hpstat='kubectl -n hanbat-$USER get pods,svc,ingress'

EOF
source ~/.bashrc
```

사용:

```bash
hp get pods              # = kubectl -n hanbat-labuser get pods
hplog order-api-xxx      # 실시간 로그
hpev                     # 최근 이벤트
hpstat                   # 전체 상태
```

---

## 다음 단계

[:material-arrow-left: 자주 만나는 오류](common-errors.md){ .md-button }
[시작 페이지로 돌아가기 :material-home:](../index.md){ .md-button .md-button--primary }
