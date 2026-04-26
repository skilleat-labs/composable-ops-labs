# 한밭푸드 시즌 2 · 운영의 계절

**Composable Architecture 운영 자동화 및 고도화 · 15시간 강의 실습 가이드**

시즌 1([한밭푸드 ACA 이관 실습](https://skilleat-labs.github.io/container-service-labs/))에서 클라우드 이관을 완료한 한밭푸드 인프라팀이, 이제 **운영 성숙도**를 끌어올리는 여정입니다.

- 서비스 통신과 장애 전파 대응 (Retry, Circuit Breaker)
- GitHub Actions 기반 CI 파이프라인
- ArgoCD 기반 GitOps CD
- Azure OpenAI를 활용한 Agentic AI와 Multi-Agent 설계

## 사이트 보기

공개 사이트: <https://skilleat-labs.github.io/composable-ops-labs/>

## 로컬에서 미리보기

```bash
# 1. 가상환경 생성 (선택)
python3 -m venv .venv
source .venv/bin/activate

# 2. MkDocs Material 설치
pip install mkdocs-material

# 3. 로컬 서버 실행
mkdocs serve
```

브라우저에서 <http://127.0.0.1:8000> 접속.

## 배포

`main` 브랜치에 push 하면 GitHub Actions가 자동으로 빌드 후 GitHub Pages에 배포합니다.

초기 1회만 GitHub 저장소 **Settings → Pages → Source**를 **GitHub Actions**로 설정해주세요.

## 구조

```
composable-ops-labs/
├── mkdocs.yml
├── docs/
│   ├── index.md
│   ├── scenario/            # 시즌 2 시나리오, 아키텍처
│   ├── curriculum/          # 15시간 커리큘럼
│   ├── phase-0/ ~ phase-7/  # 8개 Phase 실습 가이드
│   ├── evaluation/          # 평가 및 제출 가이드
│   └── reference/           # 자주 만나는 오류, 명령어 모음
└── .github/workflows/deploy.yml
```

## 라이선스

본 자료는 **교육 목적**으로만 사용 가능합니다. Copyright © 2026 Skilleat.
