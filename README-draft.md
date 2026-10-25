# Gitory

**깃허브 커밋 기록을 자소서에 쓸 수 있는 경험 카드로 바꿔 주는 서비스.**

이력서를 쓰려고 앉으면 "뭘 했는지"가 기억나지 않는 사람이 타깃이다. 커밋·PR은 수백 개 쌓여 있는데
막상 자소서 칸 앞에서는 빈 화면이 된다. 저장소를 고르면 Gitory가 커밋을 경험 단위로 묶고,
STAR 네 칸의 초안을 **근거 커밋을 달아서** 써 준다. 비어 있는 칸은 AI가 되묻고, 사용자가 답하면 채워진다.

**지어낸 경험이 아니라 증명 가능한 경험**이 목표다. 모든 문장에는 그 문장의 근거가 된 커밋 SHA가 붙고,
클릭하면 실제 GitHub 커밋으로 간다. 근거가 없는 문장은 화면에 내보내지 않는다.

카카오테크 캠퍼스 4기 2단계 팀 프로젝트 — 경북대 1팀

| | |
|---|---|
| **써 보기** | https://3-37-14-211.sslip.io · GitHub 계정으로 로그인 (읽기 권한만 요청) |
| **처음 작업한다면** | [`docs/COLLABORATION.md`](docs/COLLABORATION.md) — 브랜치·리뷰·컨벤션 규칙 |
| **문서를 찾는다면** | 아래 [문서 지도](#문서-지도) |

---

## 무엇을 하고, 무엇을 하지 않는가

**한다**
- 저장소의 커밋·PR·이슈·리뷰를 읽어 **내가 한 일**만 추려낸다 (로그인한 계정 기준)
- 흩어진 커밋을 **경험 단위로 묶어** 후보로 제안한다 (AI)
- 고른 후보를 **STAR 네 칸 초안**으로 쓴다 — 문장마다 근거 커밋 SHA를 단다 (AI)
- 빈 칸을 보고 **무엇을 물어야 할지 정해** 되묻는다 (AI)
- 회사명·팀원 이름을 **가린 버전**으로 공유할 수 있게 한다 (원문은 보존)
- 확정한 카드를 **채용공고 조건과 연결**해 보여준다

**하지 않는다**
- **저장소에 쓰지 않는다.** OAuth 권한도 읽기만 요청한다 (비공개 저장소 접근 안 함)
- **근거 없는 문장을 쓰지 않는다.** 모델이 지어낸 커밋 SHA는 버리고, 출처 없는 수치는 "확인 필요"로 내린다
- **사용자 대신 확정하지 않는다.** 확정 버튼은 사람이 누른다
- **경력을 대신 포장하지 않는다.** 없는 경험을 만들어 주는 도구가 아니다

---

## 핵심 흐름

```
① 저장소 고르기        읽을 범위를 먼저 보여주고 동의를 받는다 (READ ONLY)
       ↓
② 분석 (비동기 Job)     커밋·PR 수집 → AI가 경험 후보로 묶기
       ↓                진행 상황 4단계 표시 · 새로고침해도 이어짐 · 취소 가능
③ 후보 고르기          제안된 후보 중 카드로 만들 것을 최대 8개
       ↓
④ 카드 초안            AI가 STAR 4칸 작성 + 칸마다 근거 커밋
       ↓
⑤ 되묻기              빈 칸을 AI가 묻고, 답하면 그 칸이 채워진다 (새 버전으로 저장)
       ↓
⑥ 가리기 · 확정        회사명·이름 치환 후 확정 → 버전 고정
       ↓
⑦ 공고 매칭            확정 카드가 채용공고의 어떤 조건을 만족하는지 근거로 연결
```

직접 작성 경로도 있다. 저장소 없이 제목·상황·행동만 적고 되묻기로 보완해 확정할 수 있다.

---

## 저장소 구조

```
backend/     Spring Boot · Java 21       인증·Job 워커·카드·버전·매칭 API (모듈러 모놀리스)
ai/          FastAPI · Python 3.12       커밋 수집, 경험 그룹화, STAR 초안, 되묻기 질문 (상태를 갖지 않음)
frontend/    React · Vite · TypeScript   사용자 화면
contracts/   OpenAPI 스냅샷              BE·AI 계약 — 깨지면 CI가 막는다
deploy/      compose · Caddy             운영 배포 구성 (EC2 1대 · HTTPS)
e2e/         Playwright + 실제 스택      DB·AI·BE·web을 띄워 도는 E2E
docs/        팀 공통 문서                아키텍처·배포·가드레일·트러블슈팅
scripts/     개발 보조 스크립트
```

**AI 서비스는 상태를 갖지 않는다.** DB도 GitHub 토큰도 직접 만지지 않고, Spring이 보낸 것만 처리해 돌려준다.
토큰은 `consent` 모듈이 `X-GitHub-Token` 헤더로만 싣는다 — 이 경계는 테스트로 강제된다.

---

## 실행

### 전체를 한 번에 (권장)

```bash
cd deploy
cp .env.example .env     # 값 채우기 — 생성 방법은 .env.example 주석
docker compose up -d --build
curl -f http://localhost/api/health
```

**브라우저는 `http://localhost` 로 연다** (5173 아님). Caddy를 거쳐야 `/api` 가 Spring으로 간다.

`.env`에 GitHub OAuth 앱 정보와 LLM 설정이 필요하다. **LLM 값이 없으면** 수집까지만 되고
경험 그룹화는 건너뛴다 — 에러가 아니라 설계된 동작이다(`ANALYSIS_GROUPING_ENABLED`).

### 파트별로

| 파트 | 준비 | 실행 | 테스트 |
|---|---|---|---|
| `backend/` | Java 21 | `./gradlew bootRun` | `./gradlew test integrationTest` |
| `ai/` | Python 3.12 | `uvicorn main:app --reload` | `pytest` |
| `frontend/` | Node 22 | `npm run dev` | `npm test` · `npm run e2e` |

프론트는 백엔드 없이도 돈다 — `VITE_API_MOCK=true`면 목 서버가 응답한다.

---

## 기능 스위치

머지와 켜기를 분리한다. **꺼진 채로 머지하고, 설정이 들어간 뒤 켠다.**

| 스위치 | 기본 | 켜면 |
|---|---|---|
| `JOB_WORKER_ENABLED` | true | 분석·초안 Job을 실제로 처리한다 |
| `ANALYSIS_GROUPING_ENABLED` | false | 수집 뒤 AI가 경험 후보를 묶는다 (LLM 설정 필요) |
| `CARD_AI_DRAFT_ENABLED` | false | 후보로 만든 카드에 AI 초안을 채운다 (LLM 설정 필요) |
| `VITE_CARD_METADATA_ENABLED` | — | 카드 제목·기간 수정 UI. **빌드 시점에 이미지에 박힌다** — 바꾸면 web 재빌드 |
| `API_DOCS_ENABLED` | false | Swagger UI 노출 |

**LLM 설정이 없는데 그룹화를 켜면 모든 분석이 실패한다.** 그래서 기본값이 false다.

---

## 배포

`develop → main` 머지 시 self-hosted runner가 EC2에서 배포한다. 자세한 절차·롤백은
[`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

> 이 저장소는 public이다. 인스턴스 ID·공인 IP·비밀값은 적지 않는다. 서버의 `deploy/.env`에서 확인한다.

---

## 테스트

| 종류 | 어디 | CI |
|---|---|---|
| 단위·통합 | `backend/src/test` · `ai/tests` · `frontend/src` | 필수 체크 |
| 계약 | `contracts/` OpenAPI 대조 | 필수 체크 |
| E2E (목) | `frontend/e2e` — 프로덕션 빌드 + preview에 실행 | 필수 체크 |
| E2E (스택) | `e2e/` — DB·AI·BE·web을 실제로 띄움. GitHub·LLM은 가짜 | 별도 워크플로 |

마이그레이션은 번호 순서를 CI가 검사한다. `alembic`/Flyway 번호 충돌은 머지 전에 막힌다.

---

## 문서 지도

| 찾는 것 | 어디 |
|---|---|
| 왜 이걸 만드는가 · 누구를 위해 · 성공 기준 | [`PRODUCT.md`](PRODUCT.md) |
| 브랜치·리뷰·컨벤션 규칙 | [`docs/COLLABORATION.md`](docs/COLLABORATION.md) |
| 모듈 경계와 의존 규칙 | [`backend/ARCHITECTURE.md`](backend/ARCHITECTURE.md) |
| BE와 AI가 주고받는 것 | [`docs/BE_AI_INTEGRATION.md`](docs/BE_AI_INTEGRATION.md) |
| AI가 잘못된 출력을 막는 방법 | [`docs/GUARDRAILS.md`](docs/GUARDRAILS.md) |
| 모델 선정·품질 측정 기록 | [`ai/evals/`](ai/evals/) |
| 배포·롤백 | [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) |
| 안 되는 게 있을 때 | [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) |
| API 스펙 | [`docs/openapi/`](docs/openapi/) |

작업 기록(날짜별 UI 점검·검증 로그)은 [`docs/CHANGELOG.md`](docs/CHANGELOG.md)로 옮겼다.

---

## 지금 상태 (2026-10-__ 갱신)

- 분석 → 후보 → 카드 초안 → 되묻기 → 확정 → 공고 매칭까지 **운영에서 동작한다**
- 공고는 로켓펀치에서 수집한다 (현재 611건)
- 모바일 전용 앱은 없다 — 반응형 웹으로 지원한다
- 매칭 근거 등급(A/B/C) 산정 기준은 정리 중이다

---

## 브랜치와 PR

- 작업 브랜치는 `feature/*` → `develop`, 주간 멘토 리뷰는 `develop` → `main`
- 마이그레이션을 추가하면 번호 순서를 확인한다 (CI가 막지만 미리 보는 게 빠르다)
- 큰 작업은 PR을 만들기 전에 **이슈 단계에서 쪼갠다** — 쌓아 올린 PR의 base를 옮기는 비용이 크다
