# AS-IS vs TO-BE 아키텍처

시즌 1이 끝났을 때의 **AS-IS**와, 시즌 2 끝났을 때의 **TO-BE**를 비교합니다.

---

## AS-IS: 지금 우리가 있는 곳

시즌 1 이관 이후 현재 한밭푸드의 시스템 구성입니다.

```
                ┌──────────────────────────────────┐
                │       사용자 (웹 브라우저)        │
                └───────────────┬──────────────────┘
                                │ HTTPS
                                ▼
                ┌──────────────────────────────────┐
                │   Azure Container Apps (ACA)     │
                │                                  │
                │   ┌──────────┐   ┌────────────┐  │
                │   │ Web App  │──▶│  API App   │  │
                │   │ (nginx)  │   │ (FastAPI)  │  │
                │   └──────────┘   └─────┬──────┘  │
                │                        │         │
                └────────────────────────┼─────────┘
                                         │ HTTP (REST)
                                         ▼
                ┌──────────────────────────────────┐
                │   결제 API (분리 완료, ACA)      │
                │   ├─ 외부 PG사 호출              │
                │   └─ 응답 지연/실패 가능         │
                └──────────────────────────────────┘

  배포 방식: 이대리가 로컬에서 az containerapp update 수동 실행
  모니터링: Azure Monitor 기본 메트릭만
  장애 대응: 없음 (타임아웃도 제대로 설정 안 됨)
```

### AS-IS의 약점

!!! danger "지난 목요일에 드러난 문제들"
    - **장애 전파**: 결제 API 지연이 주문 API 전체 장애로 확대
    - **수동 배포**: 배포 때마다 엔지니어가 CLI 명령어 직접 실행
    - **롤백 번거로움**: 문제 생기면 이전 리비전으로 수동 복귀
    - **관찰 불가**: 어느 호출이 어디서 막혔는지 추적 도구 없음

---

## TO-BE: 시즌 2 끝나면 이렇게 됩니다

```
    ┌─────────────┐                                ┌────────────────┐
    │  개발자      │──── git push ───────────────▶ │    GitHub      │
    │  박대리     │                                │   (소스 + YAML) │
    └─────────────┘                                └────────┬───────┘
                                                            │
                        ┌───────────────────────────────────┤
                        │                                   │
                        ▼                                   ▼
          ┌──────────────────────┐            ┌──────────────────────┐
          │   GitHub Actions     │            │       ArgoCD         │
          │   (CI)               │            │       (CD)           │
          │                      │            │                      │
          │  1. Build            │            │  1. Git 변경 감지    │
          │  2. Test             │            │  2. AKS에 Sync       │
          │  3. Push to ACR      │            │  3. 상태 모니터링    │
          │  4. Update YAML tag  │            │                      │
          └──────────┬───────────┘            └──────────┬───────────┘
                     │                                   │
                     ▼                                   ▼
          ┌──────────────────────┐            ┌──────────────────────┐
          │    Azure Container   │            │   AKS Cluster        │
          │    Registry (ACR)    │◀───────────│   (Korea Central)    │
          │                      │    pull    │                      │
          │  - order-api:v2.3    │            │  ┌────────────────┐  │
          │  - order-web:v2.3    │            │  │  Order API Pod │  │
          │  - payment-api:v1.5  │            │  │  + Circuit     │  │
          └──────────────────────┘            │  │    Breaker     │  │
                                              │  └───────┬────────┘  │
                                              │          │           │
                                              │  ┌───────▼────────┐  │
                                              │  │ Payment API Pod│  │
                                              │  └────────────────┘  │
                                              └──────────────────────┘

     + 별도: Azure OpenAI 기반 운영 Agent (Phase 5–6)
```

### TO-BE의 강점

!!! success "시즌 2에서 달성할 것"
    - :material-shield-check: **장애 격리**: 결제 API가 죽어도 주문 조회는 부분 기능으로 유지
    - :material-robot: **자동 배포**: `git push` 한 번으로 빌드·테스트·배포·롤백 가능
    - :material-eye: **가시성**: ArgoCD UI에서 배포 상태 실시간 확인
    - :material-brain: **운영 자동화 실험**: Agent로 장애 로그 1차 분석

---

## 아키텍처 변경 요약표

| 영역 | AS-IS (시즌 1 끝) | TO-BE (시즌 2 끝) | 담당 Phase |
| --- | --- | --- | --- |
| 서비스 간 통신 | 단순 REST 호출 | **Timeout + Retry + Circuit Breaker** | Phase 1–2 |
| 빌드/테스트 | 로컬에서 수동 | **GitHub Actions (CI)** | Phase 3 |
| 배포 | `az containerapp update` 수동 | **ArgoCD GitOps (CD)** | Phase 4 |
| 런타임 플랫폼 | Azure Container Apps | **AKS** (ArgoCD 호환) | Phase 4 |
| 컨테이너 레지스트리 | ACR (유지) | ACR (유지) | — |
| 운영 모니터링 | Azure Monitor 기본 | **+ Agent 기반 장애 분석 (실험)** | Phase 5–6 |

!!! question "ACA는 버리는 건가요?"
    아니요. Phase 7에서 **"ACA vs AKS 의사결정"**을 정식으로 다룹니다. 이번 분기 실습은 AKS + ArgoCD 조합을 **체험**하고, 실제 프로덕션 전환 여부는 각 팀이 제안서로 판단합니다.

---

## 시즌 2 신규 컴포넌트 요약

| 컴포넌트 | 용도 | Phase | 주요 기술 |
| --- | --- | --- | --- |
| Payment API (결제) | 장애 재현 대상 (의도적 지연 주입) | 1 | FastAPI, fault injection |
| Circuit Breaker Lib | 장애 격리 | 2 | `pybreaker` 또는 유사 |
| GitHub Actions 워크플로 | CI/CD 자동화 | 3 | `.github/workflows/*.yml` |
| AKS 클러스터 | GitOps 런타임 | 4 | Kubernetes 1.29+ |
| ArgoCD | CD 엔진 | 4 | Helm 설치, Application 리소스 |
| Azure OpenAI Agent | 장애 분석 실험 | 5–6 | GPT-4o-mini, Tool calling |

---

## 다음 단계

[:material-arrow-left: 시즌 2 시나리오](index.md){ .md-button }
[15시간 커리큘럼 :material-arrow-right:](../curriculum/index.md){ .md-button .md-button--primary }
