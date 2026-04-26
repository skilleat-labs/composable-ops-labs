# Phase 0 · 복귀

!!! info "예상 소요 30분"

---

## 이번 Phase에서 할 일

지난 분기 마지막 출근일, 박주임은 이관 완료 보고서를 제출하고 모처럼 3일짜리 휴가를 떠났습니다. 복귀한 월요일 아침, 자리에는 포스트잇 한 장.

!!! quote ""
    *"박대리 승진 축하! 근데 우리 이번 분기 숙제가 좀 많아. 커피 한 잔 하고 11시에 회의실로. — 김팀장"*

Phase 0에서는 시즌 1의 **실습 환경을 되살리고**, 시즌 2에서 새로 쓸 **도구(AKS, ArgoCD CLI, Azure OpenAI)**를 확인합니다.

---

## 학습 목표

- [x] Azure에서 **실습 VM을 직접 생성**하고 cloud-init으로 CLI를 자동 설치한다
- [x] AKS 클러스터는 **Phase 4에서 직접 생성**한다 (비용 절감)
- [x] Azure OpenAI 환경변수 설정은 **Phase 5에서 진행**한다
- [x] 실습 레포를 VM에 **clone** 한다 (fork는 Phase 3에서)

---

## 준비물 체크리스트

- [ ] Azure 계정 (VM 생성 권한 있는 구독)
- [ ] GitHub 계정 + **Personal Access Token** (classic, `repo` 권한)
- [ ] 강사가 전달한 **AKS 클러스터 정보** (리소스 그룹, 클러스터 이름)
- [ ] 강사가 전달한 **Azure OpenAI 정보** (엔드포인트, API 키, 배포 이름)

---

## 소요 시간

| 단계 | 시간 |
| --- | --- |
| 실습 VM 생성 (Azure Portal + User Data) | 10분 |
| CLI 설치 확인 | 5분 |
| GitHub 레포 fork | 5분 |
| 스모크 테스트 (kubectl, OpenAI 호출) | 5분 |

---

## 다음 단계

[환경 세팅 시작하기 :material-arrow-right:](environment-setup.md){ .md-button .md-button--primary }
