# 환경 세팅

!!! info "PHASE 0 · 예상 30분"

---

??? info "강사 사전 준비 (학생은 건너뛰세요)"

    강의 시작 전에 아래 작업을 완료해야 합니다.

    **1. AKS 클러스터 사전 생성**

    학생이 매번 AKS를 직접 만들면 시간이 많이 걸리고 실습 몰입이 끊깁니다. 강사가 사전에 소규모 클러스터를 생성해두는 것을 권장합니다.

    ```bash title="터미널 (강사)"
    # 리소스 그룹 생성
    az group create --name hanbat-s2-rg --location koreacentral

    # AKS 클러스터 생성 (2 노드, B2s)
    az aks create \
      --resource-group hanbat-s2-rg \
      --name hanbat-s2-aks \
      --node-count 2 \
      --node-vm-size Standard_B2s \
      --enable-managed-identity \
      --generate-ssh-keys \
      --network-plugin azure
    ```

    **2. 학생 접근 권한 부여**

    각 학생 계정에 클러스터 읽기/쓰기 권한 부여:

    ```bash title="터미널 (강사)"
    az aks show -g hanbat-s2-rg -n hanbat-s2-aks --query id -o tsv
    # 출력된 리소스 ID 복사 후:
    az role assignment create \
      --assignee <학생_object_id> \
      --role "Azure Kubernetes Service Cluster User Role" \
      --scope <위에서_복사한_리소스_ID>
    ```

    **3. Azure OpenAI 리소스 준비**

    - Azure Portal에서 Azure OpenAI 리소스 생성 (지역: **East US** 또는 **Sweden Central** — Korea Central은 일부 모델 미지원)
    - **gpt-4o-mini** 모델 배포 (배포 이름 예: `gpt-4o-mini`)
    - 학생에게 전달할 정보: **엔드포인트 URL**, **API 키**, **배포 이름**, **API 버전**

    **4. 학생에게 전달할 최종 정보**

    | 항목 | 예시 |
    | --- | --- |
    | AKS 리소스 그룹 | `hanbat-s2-rg` |
    | AKS 클러스터 이름 | `hanbat-s2-aks` |
    | OpenAI 엔드포인트 | `https://hanbat-aoai.openai.azure.com/` |
    | OpenAI 배포 이름 | `gpt-4o-mini` |
    | OpenAI API 버전 | `2024-08-01-preview` |
    | OpenAI API 키 | 강사가 개별 전달 |

---

## Step 1. Azure CLI 설치 (로컬 PC)

VM을 만들기 전에 로컬 PC에 Azure CLI를 설치합니다.

=== "Windows (PowerShell)"

    PowerShell을 **관리자 권한**으로 실행한 뒤 아래 명령을 실행합니다.

    ```powershell title="터미널 (Windows PowerShell — 관리자)"
    winget install Microsoft.AzureCLI
    ```

    설치가 끝나면 PowerShell을 **닫았다가 다시 열어야** `az` 명령이 인식됩니다.

    !!! note "winget이 없다면"
        Windows 10 구버전은 winget이 없을 수 있습니다. Microsoft Store에서 **앱 설치 관리자**를 업데이트하거나, [Azure CLI MSI 설치 파일](https://aka.ms/installazurecliwindows)을 직접 다운로드하세요.

=== "Mac (Terminal)"

    Homebrew로 설치합니다.

    ```bash title="터미널 (Mac)"
    brew update && brew install azure-cli
    ```

    !!! note "Homebrew가 없다면"
        ```bash title="터미널 (Mac)"
        /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
        ```
        설치 후 위 `brew install azure-cli` 를 다시 실행하세요.

설치 확인:

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    az --version
    ```

=== "Mac (Terminal)"

    ```bash title="터미널 (Mac)"
    az --version
    ```

```text title="출력 예시"
azure-cli                         2.60.0
...
```

### Azure 로그인

```bash title="터미널"
az login
```

브라우저가 자동으로 열립니다. 로그인 완료 후 터미널로 돌아오면 구독 목록이 출력됩니다.

!!! note "브라우저가 안 열리는 환경이라면"
    ```bash title="터미널"
    az login --use-device-code
    ```
    출력된 코드를 복사해 `https://microsoft.com/devicelogin` 에서 입력하세요.

!!! success "✅ 확인 포인트"
    `az account show` 실행 시 본인 구독 정보가 출력되면 로그인 성공입니다.

---

## Step 2. 실습 VM 생성 (Azure Portal)

Azure Portal에서 VM을 만들고 **User Data**로 CLI 도구를 자동 설치합니다.

### 2-1. 가상 머신 만들기 시작

1. [Azure Portal](https://portal.azure.com) 접속
2. 상단 검색창에서 **"가상 머신"** 검색 → **만들기** → **Azure 가상 머신**

### 2-2. 기본 사항 탭

| 항목 | 값 |
| --- | --- |
| 구독 | 본인 구독 선택 |
| 리소스 그룹 | **hanbat-s2-lab-rg** (없으면 새로 만들기) |
| 가상 머신 이름 | **hanbat-lab-vm** |
| 지역 | **(Asia Pacific) Korea Central** |
| 가용성 옵션 | 인프라 중복 필요 없음 |
| 이미지 | **Ubuntu Server 22.04 LTS** |
| 크기 | **Standard_B2s_v2** (2 vCPU, 4GB) |

**관리자 계정** 항목:

| 항목 | 값 |
| --- | --- |
| 인증 형식 | **암호** |
| 사용자 이름 | **labuser** |
| 암호 | 본인이 정한 비밀번호 |
| 암호 확인 | 동일하게 입력 |

!!! warning "비밀번호 규칙"
    12자 이상, 대문자·소문자·숫자·특수문자 중 3가지 이상 포함. 예: `Hanbat2026!`

!!! warning "인바운드 포트"
    공용 인바운드 포트 → **선택한 포트 허용** → **SSH(22)** 선택

### 2-3. 고급 탭 — User Data 입력

탭 상단의 **고급**을 클릭합니다.

**사용자 데이터** 항목을 찾아 아래 내용을 **그대로 붙여넣습니다.**

```text title="User Data"
#cloud-config
ssh_pwauth: true
package_update: true
package_upgrade: false

packages:
  - curl
  - git
  - unzip
  - python3-pip
  - snapd

runcmd:
  - snap install docker
  - addgroup --system docker
  - adduser labuser docker
  - curl -sL https://aka.ms/InstallAzureCLIDeb | bash
  - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  - install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
  - rm kubectl
  - curl -LO https://github.com/Azure/kubelogin/releases/latest/download/kubelogin-linux-amd64.zip
  - unzip -q kubelogin-linux-amd64.zip
  - install -m 755 bin/linux_amd64/kubelogin /usr/local/bin/kubelogin
  - rm -rf bin kubelogin-linux-amd64.zip
  - curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  - curl -sSL -o /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
  - install -m 555 /tmp/argocd /usr/local/bin/argocd
  - rm /tmp/argocd
```

!!! warning "주의"
    User Data 입력란에 한글이 들어가면 오류가 납니다. 위 내용을 그대로 복붙하세요.

### 2-4. 검토 + 만들기

**검토 + 만들기** 탭에서 유효성 검사 통과 확인 후 **만들기** 클릭.

배포 완료까지 2~3분 소요됩니다.

### 2-5. 공용 IP 확인

배포 완료 후 **리소스로 이동** → 개요 화면에서 **공용 IP 주소** 복사.

### 2-6. SSH 접속

=== "Windows (PowerShell)"

    ```powershell title="터미널 (Windows PowerShell)"
    ssh labuser@<공용_IP_주소>
    ```

=== "Mac / Linux (Terminal)"

    ```bash title="터미널 (Mac / Linux)"
    ssh labuser@<공용_IP_주소>
    ```

비밀번호 입력 시 화면에 아무것도 표시되지 않는 게 정상입니다. 입력 후 Enter를 누르세요.

!!! success "✅ 확인 포인트"
    프롬프트가 `labuser@hanbat-lab-vm:~$` 형태로 나오면 접속 성공입니다.

---

## Step 3. CLI 설치 확인

cloud-init이 백그라운드에서 실행 중일 수 있습니다. 아래 명령으로 완료 여부를 확인하세요.

```bash title="터미널"
# cloud-init 완료 대기 (최대 5분)
cloud-init status --wait
```

```text title="출력 예시"
status: done
```

완료되면 도구 버전을 확인합니다.

```bash title="터미널"
az --version | head -1
kubectl version --client
helm version --short
argocd version --client --short
```

!!! success "✅ 확인 포인트"
    네 도구 모두 버전이 출력되면 OK. 에러가 나면 [자주 만나는 오류](../reference/common-errors.md) 참고.

### Azure CLI 로그인

```bash title="터미널"
az login --use-device-code
```

출력에 나오는 **코드**를 복사해서 안내된 URL에 입력하면 브라우저 로그인이 완료됩니다.

### Azure CLI 자동완성 설정

`az` 명령어 Tab 자동완성을 활성화합니다. 한 번만 설정하면 재접속 후에도 유지됩니다.

```bash title="터미널"
cat >> ~/.bashrc << 'EOF'
source /etc/bash_completion.d/azure-cli
EOF
source ~/.bashrc
```

이후 `az ` 입력 후 Tab을 누르면 명령어가 자동완성됩니다.

---

## Step 4. AKS 클러스터

!!! info "Phase 4에서 진행합니다"
    AKS 클러스터는 Phase 4(ArgoCD GitOps)에서 처음 필요합니다. Phase 0에서 미리 만들면 Phase 1~3 진행하는 동안 비용이 계속 발생하므로, **Phase 4 시작 시 함께 생성합니다.**

    지금은 넘어가세요.

---

## Step 5. Azure OpenAI 환경변수 설정

!!! info "Phase 5에서 진행합니다"
    Azure OpenAI는 Phase 5(Agent 실습)에서 처음 사용합니다. 지금 설정할 필요 없습니다. **Phase 5 시작 시 강사가 API 키와 함께 안내합니다.**

    지금은 넘어가세요.

---

## Step 6. 실습 레포 clone

VM에서 실습 소스 레포를 clone 합니다.

```bash title="터미널"
cd ~
git clone https://github.com/skilleat-labs/hanbat-order-app-s2.git
cd hanbat-order-app-s2
ls -la
```

```text title="출력 예시"
.
├── .github/
├── k8s/
├── order-api/
├── order-web/
├── payment-api/
├── docker-compose.yml
└── README.md
```

!!! note "시즌 1 레포와 다른 점"
    - `payment-api/` 디렉터리 추가 (결제 API 분리)
    - `k8s/` 디렉터리 추가 (Kubernetes 매니페스트)
    - `.github/workflows/` 는 현재 비어있음 (Phase 3에서 여러분이 채웁니다)

!!! info "Phase 3에서 fork 합니다"
    지금은 읽기 전용으로 clone만 합니다. GitHub Actions CI/CD를 설정하는 **Phase 3 시작 시** 본인 계정으로 fork하고 remote를 변경합니다. GitHub 계정이 없어도 Phase 1~2는 진행할 수 있습니다.

---

## Step 7. 스모크 테스트

CLI 도구가 정상 설치됐는지 확인합니다.

```bash title="터미널"
az --version | head -1
kubectl version --client
helm version --short
argocd version --client --short
```

!!! success "✅ 확인 포인트"
    네 도구 모두 버전이 출력되면 Phase 0 완료입니다.

---

## 자주 만나는 문제

??? failure "cloud-init status: error 가 나왔어요"
    마지막 줄의 `cc_keys_to_console` 경고 때문인 경우가 많습니다. `sudo cat /var/log/cloud-init-output.log | tail -30` 으로 실제 설치 로그를 확인하세요. az, kubectl, helm, argocd 가 모두 설치됐다면 무시해도 됩니다.

??? failure "GitHub clone 시 permission denied"
    - HTTPS로 fork 후 HTTPS로 clone하는지 확인 (SSH 키 설정 불필요)
    - fork가 본인 계정에 제대로 생성되었는지 브라우저에서 확인

---

## 체크리스트

여기까지 했으면 Phase 1로 넘어갈 준비 완료입니다.

- [ ] 실습 VM 생성 및 SSH 접속 OK
- [ ] `kubectl`, `helm`, `argocd` CLI 설치 OK (cloud-init 완료)
- [ ] `kubectl version --client` 버전 확인 OK
- [ ] Azure OpenAI 환경변수 설정 — Phase 5에서 진행
- [ ] VM에 레포 clone OK (`~/hanbat-order-app-s2`)

---

## 다음 단계

[:material-arrow-left: Phase 0 개요](index.md){ .md-button }
[Phase 1 · 연쇄 장애의 밤 :material-arrow-right:](../phase-1/index.md){ .md-button .md-button--primary }
