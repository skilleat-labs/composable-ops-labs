# 한밭푸드 시즌 2 · 운영의 계절

**Composable Architecture 운영 자동화 및 고도화 · 15시간 강의 · 학생용 실습 가이드**

지난 분기, 여러분은 한밭푸드 주문 조회 시스템을 Azure Container Apps로 무사히 이관했습니다. 그런데 운영 3개월차, 진짜 싸움은 이제부터입니다.

## 지난 분기 요약 (시즌 1)

!!! info "시즌 1을 아직 못 보신 분이라면"
    [한밭푸드 ACA 이관 실습 사이트](https://skilleat-labs.github.io/container-service-labs/)를 먼저 훑어보시면 맥락이 빠르게 잡힙니다. 꼭 실습하지 않으셔도 **시나리오**와 **아키텍처**만 읽어도 충분합니다.

- 주문 조회 시스템(Web + API)을 ACA로 이관 완료
- 무중단 배포, 자동 확장, 점진적 배포까지 갖춤
- CS팀 컴플레인 3종 세트 모두 해결

## 지난주 목요일 오후 2시, 무슨 일이 있었나

CTO에게 한 통의 메일이 다시 왔습니다.

> *"지난 목요일 결제 API가 느려지는 동안, 주문 조회 API까지 같이 죽었습니다. 고객은 결제는커녕 자기 주문 내역도 못 봤어요. 한 군데 문제가 왜 전체로 번지는지 설명해주세요. 그리고 이번엔 '사람 손이 덜 가는 배포'와 '장애가 번지지 않는 구조'를 만들어주세요. 덤으로 본사에서 숙제 하나 내려왔습니다 — **AI Agent로 운영 자동화가 가능한지 검토**해보라네요."*

여러분은 이제 박주임이 아니라 **박대리**입니다. 1년 사이에 진급도 했고, 동료 이대리, 김팀장과 함께 이 세 가지 숙제를 풀어야 합니다.

---

## 6개의 대단원, 15시간의 여정

<div class="grid cards" markdown>

- :material-alert-octagon:{ .lg .middle } **장애 대응**

    ---

    한 서비스의 느려짐이 왜 전체 장애가 되는지 직접 체험하고, Retry와 Circuit Breaker로 **복원력**을 설계합니다.

    `Phase 1–2 · 2.5h`

- :material-cog-sync:{ .lg .middle } **CI/CD 자동화**

    ---

    GitHub Actions로 Build-Test-Deploy를 자동화하고, ArgoCD로 **GitOps** 기반 무중단 배포까지 완성합니다.

    `Phase 3–4 · 5h`

- :material-robot-outline:{ .lg .middle } **Agentic AI**

    ---

    LLM과 Agent의 차이부터 Multi-Agent 설계 패턴까지, Azure OpenAI로 직접 만들어봅니다.

    `Phase 5–6 · 6h`

- :material-check-circle-outline:{ .lg .middle } **평가 및 회고**

    ---

    팀별로 "한밭푸드 운영 성숙도 제안서"를 작성하고, 의사결정 매트릭스로 다음 분기를 설계합니다.

    `Phase 7 · 1h`

</div>

---

## 시작 전 준비 체크리스트

강의 시작 전에 아래 항목을 모두 확인해주세요.

- [ ] 강사가 공유한 **Azure 계정** 정보 확보
- [ ] 본인 **GitHub 계정** 준비 (없다면 [github.com](https://github.com)에서 가입)
- [ ] SSH 클라이언트 준비 (Windows: PowerShell / Mac·Linux: Terminal)
- [ ] **Azure OpenAI 리소스 접근 권한** 확인 (강사가 안내)
- [ ] 브라우저 탭 열어두기: [Azure Portal](https://portal.azure.com), [GitHub](https://github.com)

!!! warning "Azure OpenAI는 사전 신청이 필요할 수 있음"
    Azure OpenAI 서비스는 구독에 따라 접근 신청이 필요합니다. 강사가 **공용 Azure OpenAI 리소스**를 제공하거나, 개인 리소스 신청 방법을 안내할 것입니다.

---

## 실습 환경 개요

| 항목 | 내용 |
| --- | --- |
| 실습 VM | Ubuntu 22.04 LTS (시즌 1에서 쓰던 VM 재활용 가능) |
| 사전 설치 | Docker, Docker Compose, Azure CLI, Git, kubectl |
| Kubernetes | **AKS** (강사가 사전 프로비저닝 또는 실습 중 생성) |
| Azure 리전 | Korea Central |
| LLM | **Azure OpenAI** (GPT-4o / GPT-4o-mini) |
| Git 저장소 | GitHub (개인 계정) |
| 실습 앱 | 한밭푸드 주문 조회 시스템 v2 (Web + API + 결제 API mock) |

---

## 지금 시작하기

[시즌 2 시나리오 보기 :material-arrow-right:](scenario/index.md){ .md-button .md-button--primary }
[Phase 0 환경 세팅 바로가기 :material-arrow-right:](phase-0/index.md){ .md-button }
