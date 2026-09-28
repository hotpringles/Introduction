# 소개 SOGAE · Next.js 재구축 설계

작성일: 2026-09-26
수정일: 2026-09-29 (화면 프로토타입 반영)
상태: 검토 대기
관련 자료
- 화면 프로토타입: `prototypes/coaching-desk/index.html` — 빌드 없이 브라우저에서 여는 단일 파일. 화면 배치, 색, 상호작용은 이 파일을 기준으로 한다
- 처음 디자인: claude.ai 캔버스 "SOGAE 서비스 화면 디자인"의 Editorial 페이지 (프로토타입으로 대체)
- 도식: Obsidian `소개/SOGAE 데이터 계약과 분석 흐름.excalidraw.md`
- 이전 구현: github.com/hotpringles/SOGAE `develop` 브랜치 (Vite + React)

## 1. 목표

소개는 사용자가 직접 쓴 글의 논리 구조와 글쓰기 습관을 분석하고, 사용자의 문체를 유지하면서 개선 방향을 제안하는 글쓰기 코칭 서비스다. AI는 글을 대신 쓰지 않고 `진단 → 설명 → 질문 → 개선 제안` 흐름으로 돕는다.

이번 범위는 프로토타입의 네 화면(로그인, 대시보드, 구조 선택, 작업대)을 Next.js로 새로 만들고, 로그인·문서·분석을 Mock으로 실제 동작시키는 것이다. 이후 Supabase와 실제 LLM으로 바꿀 때 교체할 경계를 지금 만들어 둔다.

### 완료 조건

- 로그인 → 대시보드 → 구조 선택 → 작업대 작성 → 분석 → 피드백 확인이 끊김 없이 동작한다.
- 새로고침 후에도 문서, 문단 역할, 분석 결과가 남아 있다.
- 데스크톱과 휴대폰 폭에서 프로토타입과 같은 배치·글꼴·색을 쓴다.
- `npm test`, `npm run lint`, `npm run build`가 모두 통과한다.

### 이번 범위에서 제외

회원가입, 비밀번호 찾기, 구글 로그인(버튼은 비활성으로 표시), 히스토리·마이페이지, 버전 비교(diff)와 복원 화면, 모바일 바텀 시트, Supabase 연동, 실제 LLM 연동, Playwright E2E.

## 2. 기술 스택

| 영역 | 선택 |
| --- | --- |
| 프레임워크 | Next.js App Router (초기 세팅 시점의 최신 안정 버전), TypeScript, Node 22 |
| 스타일 | Tailwind CSS 4, 프로토타입 디자인 토큰을 `@theme`에 정의 |
| 글꼴 | Pretendard (`next/font/local`), IBM Plex Mono (`next/font/google`) |
| 서버 상태 | TanStack Query |
| 클라이언트 UI 상태 | Zustand (피드백 창 열림과 폭, 선택 탭, 펼친 지적 사항, 현재 문단) |
| 검증 | Zod (입력, 저장소 경계, API 요청·응답) |
| 폼 | React Hook Form + Zod resolver |
| 에디터 | TipTap, 문단 확장 `SogaeParagraph` (문단 id와 역할) |
| 테스트 | Vitest, React Testing Library, jsdom |
| 배포 | Vercel |

## 3. 디자인 토큰

| 토큰 | 값 | 쓰임 |
| --- | --- | --- |
| `--color-night` | `#171c22` | 사이드바, 로그인 소개 칸 배경 |
| `--color-night-text` | `#e7ecf2` | 어두운 배경 위 본문 |
| `--color-night-muted` | `#c5ced8` | 어두운 배경 위 보조 글씨 |
| `--color-night-bright` | `#f4f7f5` | 어두운 배경 위 제목, 워드마크 |
| `--color-desk` | `#f2f5f8` | 카드, 호버, 선택한 목록 배경 |
| `--color-sheet` | `#ffffff` | 본문 종이, 피드백 창 |
| `--color-ink` | `#1a212b` | 본문 |
| `--color-muted` | `#3e4c5c` | 보조 글씨 |
| `--color-line` | `#d5dde6` | 구분선, 점수 막대 바탕 |
| `--color-line-strong` | `#c5ced6` | 입력창 테두리, 역할 묶음 막대 |
| `--color-hover` | `#e7edf2` | 선택한 탭 |
| `--color-folder` | `#e2b657` | 주요 버튼, 폴더 아이콘, 현재 화면 표시 |
| `--color-folder-tab` | `#c4a15a` | 강조 선, 현재 문단의 묶음 막대, 지적 번호, 가장 낮은 점수 막대 |
| `--color-cream` | `#f8f1de` | 현재 문단과 펼친 지적 사항 배경(70%) |
| `--color-on-folder` | `#1a160c` | 폴더색 위 글씨 |
| `--color-alert` | `#a3432b` | 입력 오류, 저장 실패, 구조 불일치 |
| `--shadow-lift` | `0 14px 36px rgb(196 161 90 / .22), 0 3px 8px rgb(196 161 90 / .14)` | 선택된 추천 구조 카드 |

모서리는 버튼·입력창 6px, 카드 8px. 글꼴은 제목·본문 모두 Pretendard(제목 600, 자간 좁게)이고, 날짜·숫자·글자 수·저장 상태 같은 메타 정보와 작업대 본문 입력은 IBM Plex Mono(본문 15px, 줄 높이 28px)다. 한국어 문단은 `word-break: keep-all`. 움직임은 `prefers-reduced-motion`이면 끈다.

## 4. 구조

### 라우트

```text
app/
  (auth)/login/page.tsx
  (app)/layout.tsx               AppSidebar, QueryClient, RepositoryProvider
  (app)/dashboard/page.tsx
  (app)/write/structure/page.tsx
  (app)/documents/[id]/edit/page.tsx
  api/analyze/route.ts
  not-found.tsx, error.tsx
middleware.ts                    쿠키 sogae_session 없으면 /login?next=… 로 이동
```

`page.tsx`는 `src/views`의 화면 컴포넌트를 불러오기만 한다.

### 폴더 (FSD)

```text
src/
  views/       login, dashboard, structure-select, editor
  widgets/     app-sidebar, feedback-panel, paragraph-outline, document-shelf
  features/    sign-in, select-structure, edit-document, analyze-document, apply-suggestion, remove-document
  entities/    session, document, analysis, structure
  shared/      ui, api(storage-adapter, app-error), lib, config
  server/      analyzer (서버 전용, import "server-only")
```

의존 방향은 `app → views → widgets → features → entities → shared`로 고정하고 ESLint `no-restricted-imports`로 역방향을 막는다. `src/server`는 `app/api`에서만 불러온다. 슬라이스 밖에서는 각 슬라이스의 `index.ts`만 불러온다.

## 5. 화면

### 공통 사이드바 `AppSidebar`
768px 이상에서는 왼쪽 192px 세로 막대(night 배경), 그보다 좁으면 위쪽 가로 막대. 위에서부터 폴더 아이콘과 워드마크 "SOGAE", 이동(대시보드, 새 글, 작업대), 맨 아래 이메일과 로그아웃. 작업대는 마지막으로 연 문서가 있을 때만 보이고 그 문서로 간다. 현재 화면은 폴더색 글씨와 왼쪽 2px 막대로 표시한다.

### 로그인 `/login`
왼쪽: night 배경에 워드마크, 제목 "다시 만나서 반갑습니다", 서비스 소개 2단 칼럼, 왼쪽에 폴더색 선을 두른 인용구. 오른쪽 480px 흰 칸: 이메일·비밀번호 폼(제출 후에는 입력할 때마다 검증 문구 갱신), 폴더색 로그인 버튼, 비활성 "Google로 계속하기 · 준비 중".

### 대시보드 `/dashboard`
- 위: 오늘 날짜와 "오늘 고친 글 n편, 분석 n번"
- 이어서 쓸 글: 가장 최근에 고친 글의 유형·구조, 제목, 첫 문단 발췌, 수정 시각·글자 수·문단 수, "이어 쓰기"와 "새 글 쓰기". 글이 없으면 "아직 글이 없습니다"와 새 글 쓰기
- 1280px 이상은 그 아래를 `minmax(0, 21rem) | 1px | 1fr` 세 칸으로 나눈다. 더 좁으면 세로로 쌓는다
  - 왼쪽 "내 글 점수": 글마다 가장 최근 분석 하나를 모아 낸 항목별 평균, 80px 평균 점수, 가장 낮은·높은 항목을 말하는 문장, 6항목 점수 막대, "가장 먼저 손볼 글"(평균이 가장 낮은 글, 누르면 첫 지적을 펼친 작업대). 그 아래 "글쓰기 패턴": 글이 3편 미만이면 "준비 중"과 지금 편수, 이상이면 여러 글에서 가장 많이 되풀이된 지적 종류
  - 괘선
  - 오른쪽: 글 유형 폴더 5개(글 수 표시, 고른 유형은 폴더가 열린다), 고른 유형의 글 목록(제목, 발췌 60자, 구조, 수정 시각, 점수 또는 "분석 전", 치우기). 목록만 따로 스크롤하고 아래에 48px 페이드. "+ 새 글"은 그 유형을 고른 채로 구조 선택을 연다
- 치우기는 확인을 받은 뒤 문서와 그 분석·버전을 함께 지운다

### 구조 선택 `/write/structure`
위에 오늘 날짜와 "글 유형 → 구조 → 작성", 제목 "어떤 구조로 쓸까요?". 왼쪽 192px 글 유형 목록, 오른쪽에 추천 구조 카드 3장과 다른 구조 7개 목록.
- 추천 카드: 추천 번호, 이름, 단계 칩, 설명. 설명은 카드 아래에 붙여 세 카드 높이를 맞춘다. 선택한 카드는 배경을 바꾸지 않고 folder-tab 테두리, `--shadow-lift`, 4px 떠오름으로 표시한다
- 다른 구조 목록: 이름, 설명, 단계 흐름("배경 → 문제 → …"). 선택하면 왼쪽 2px folder-tab 선과 크림 배경
- 글 유형 아래에 "선택한 구조" 요약(유형, 구조 이름, 단계 칩, "이 구조로 작성 시작")이 붙어 스크롤을 따라온다. 좁은 화면에서는 구조 목록 뒤에 온다
- 대시보드에서 `?type=`으로 오면 그 유형과 첫 추천 구조를 고른 채로 연다
- "이 구조로 작성 시작"은 구조 단계마다 그 역할이 붙은 빈 문단을 하나씩 둔 문서를 만들고(자유 구조는 역할 없는 빈 문단 하나) 작업대로 이동해 첫 빈 문단에 커서를 둔다

### 작업대 `/documents/[id]/edit`
본문 칸 | 끌 수 있는 경계 | AI 피드백 창. 피드백 창은 기본 416px이고, 경계를 끌거나 경계에 초점을 두고 좌우 화살표로 320px부터 (전체 − 360px)까지 바꾼다. 창을 닫으면 본문이 남은 폭을 모두 쓴다.

**머리**: "글 유형 · 구조", 제목 입력(100자, Enter는 첫 문단으로 이동), 저장 상태("저장 중…" / "저장됨 · 시각" / "저장 안 됨 · 다시 시도"), 글자 수·문단 수, 버전 저장, AI 피드백 토글. 800ms 자동 저장.

**문단**: 문단마다 76px 역할 칸과 본문.
- 연속해서 같은 역할을 가진 문단은 한 묶음이다. 저장은 문단 단위 그대로이고, 묶음은 화면에서만 계산한다
- 묶음의 첫 문단에만 역할 상자가 보인다. 문단이 둘 이상인 묶음은 역할 칸과 본문 사이에 첫 줄부터 마지막 줄까지 이어지는 2px 세로 막대를 둔다
- 이어지는 문단의 역할 상자는 평소 숨기고, 그 역할 칸에 마우스를 올리거나 상자에 초점이 갈 때만 보인다. 본문을 입력하는 동안에는 숨긴 채로 둔다
- 첫 문단의 역할을 바꾸면 묶음 전체가 함께 바뀐다. 이어지는 문단의 역할을 바꾸면 그 문단만 묶음에서 떨어진다
- 지적이 걸린 문단은 역할 칸에 "지적 1·3"처럼 번호를 적는다. 이어지는 문단에서는 첫 줄 높이에 둔다
- 현재 문단은 크림 배경(70%)으로만 표시하고, 그 문단이 속한 묶음 막대를 folder-tab 색으로 바꾼다. 문단 왼쪽에 따로 선을 긋지 않는다
- 빈 문단 안내 문구는 역할별 질문(예: 행동 "내가 직접 한 일을 동사로 적어 보세요."), 이어지는 문단은 "이어서 적어 보세요."
- 키보드: Enter는 커서 자리에서 문단을 나누고 뒤 문단은 같은 역할을 잇는다. 문단 맨 앞 Backspace는 앞 문단과, 맨 끝 Delete는 뒤 문단과 합치며 앞 문단의 id와 역할이 남는다. 문단 맨 앞·맨 끝의 ↑·↓는 이전·다음 문단으로 옮긴다. 여러 문단을 붙여넣으면 새로 생긴 문단은 붙여넣은 자리 문단의 역할을 잇는다
- 맨 아래 "+ 문단 · 역할" 버튼은 끝에 빈 문단을 붙인다. 새 문단의 역할은 `nextStageRole()`로 정하고, 버튼에 그 역할을 미리 보여 준다
  - 마지막으로 쓴 구조 단계(뒤에서부터 찾아 처음 나온, 구조에 속한 역할)의 다음 단계
  - 그 단계가 구조의 마지막 단계면 같은 단계
  - 구조에 속한 역할을 가진 문단이 없으면 첫 단계
  - 자유 구조처럼 단계가 없으면 마지막 문단의 역할

**문단 목록**: 본문 칸 폭이 800px 이상이면 본문 왼쪽 192px에 테두리 없이 붙어 스크롤을 따라온다. 더 좁으면 왼쪽 아래 "문단 n" 버튼으로 여는 목록이 된다. 구조 단계 칩(내용 있는 문단이 맡은 단계는 채워서 표시), "역할 흐름 · n개, n문단", 묶음마다 문단 범위(`01–03`), 역할, 문단 수, 첫 문장 40자, 지적 점. 누르면 그 묶음의 첫 문단으로 이동한다.

**AI 피드백 창**: 머리 48px(제목, "분석하기"·"다시 분석"·"분석 중…", 닫기)는 고정하고 그 아래만 스크롤한다.
- 분석 전: "아직 분석하지 않았습니다"와 안내
- 분석 후: 분석 시각·몇 번째 분석·model, 60px 평균 점수, 분석 뒤 글이 바뀌었으면 "이전 버전 기준" 안내, 요약, 6항목 점수 막대(가장 낮은 항목 강조), 위에 붙어 따라오는 탭(지적 사항 n / 구조 비교), 아래 48px 페이드
- 지적 사항: 번호, 위치("n문단" / "글 전체" / 연결이 끊겼으면 "분석 때 n문단"), 종류, 이유. 한 번에 하나만 펼친다. 펼치면 인용, 생각해 볼 질문, 개선 방향, 예시(`[기간]` 같은 빈칸을 강조하고 "칸은 직접 채워 주세요"), 적용 버튼. 분석이 끝나면 첫 지적을 펼친다
- 구조 비교: 일치 / 부분 일치 / 불일치, 선택한 구조와 읽힌 구조, 설명, 선택한 구조의 단계(아무 문단도 맡지 않은 단계는 점선), 분석한 글의 문단 순서
- 집중 모드: 펼친 지적이 지금 문단과 연결되면 그 문단만 불투명도 1, 나머지 0.35로 두고 그 문단이 보이게 스크롤한다. 피드백 창을 닫으면 꺼진다

**버전 저장**: 직전 버전과 내용이 같으면 새로 만들지 않고 "이미 남겨 둔 버전과 같습니다."를 알린다.

### 없는 문서
"문서를 찾을 수 없어요", 대시보드로 가는 버튼.

### 반응형
768px 이상은 사이드바를 세로로 두고 작업대를 좌우로 나눈다. 768px 미만은 사이드바가 위쪽 가로 막대가 되고, 작업대의 피드백 창은 본문 아래 전체 폭 영역이 된다. 대시보드의 세 칸은 1280px 이상에서만 나란히 둔다.

## 6. 데이터

### 엔티티

- `User { id, email, displayName, createdAt }`
- `Session { userId, issuedAt, expiresAt }` — localStorage와 쿠키 `sogae_session`에 함께 저장
- `Document { id, userId, title(0–100), writingType, structure, content: TipTapDoc, createdAt, updatedAt }`
- `TipTapDoc` — `doc > paragraph[attrs: { paragraphId, role }] > text`. 문단 역할은 따로 저장하지 않고 문단 노드 속성으로 본문 안에 둔다
  - `paragraphId`: string(1–128), `role`: ParagraphRole | null
  - 새 문서는 선택한 구조의 단계마다 그 역할이 붙은 빈 문단을 하나씩 둔다. 자유 구조는 역할 없는 빈 문단 하나
  - 새 문단은 새 uuid. 분할하면 뒤 문단은 새 id를 받고 역할은 앞 문단과 같다(`paragraphId`는 `keepOnSplit: false`, `role`은 `keepOnSplit: true`). 병합하면 앞 문단의 id와 역할을 유지한다
  - 붙여넣기로 새로 생긴 문단(id 없는 문단)은 붙여넣은 자리 문단의 역할을 받는다
  - id가 없거나 겹치면 플러그인이 새 id를 붙인다. 문서 안에서 복사한 문단이 원본보다 앞에 붙어도 원본이 id를 유지한다(이전 상태의 위치를 트랜잭션 매핑으로 추적). 사본은 새 id와 역할 없음으로 시작한다
  - 역할 변경은 `setParagraphRole(paragraphId, role)` 명령으로 하며 실행 취소에 포함된다. 묶음 전체의 역할 변경은 한 트랜잭션으로 묶는다
  - 문단 순서는 저장하지 않고 `readParagraphs(doc)`가 계산한다. 문단 목록, 분석 요청, 구조 비교 탭이 이 함수를 같이 쓴다
- `DocumentSummary { id, title, writingType, structure, updatedAt, latestScore: number | null, excerpt: string }` — 대시보드 글 목록용. latestScore는 가장 최근 분석의 6항목 평균(소수 첫째 자리), excerpt는 첫 문단 앞 60자
- `DocumentVersion { id, documentId, content, reason: VersionReason, createdAt }` — 버전 저장, 예시 적용 직전, 복원 직전에만 만든다. 직전 버전과 content가 같으면 새로 만들지 않고 그 버전을 돌려준다. 문서마다 최근 40개만 남긴다
- `VersionReason` = `"manual" | "before_apply_suggestion" | "before_restore"`
- `Analysis { id, documentId, versionId | null, snapshot: ParagraphSnapshot[], schemaVersion: "1.0.0", model, promptVersion | null, result: AnalysisResponse, createdAt }`
- `ParagraphSnapshot { paragraphId(1–128), index(int ≥ 0), role | null, text(1–2000) }`
- `AnalysisResponse` — develop의 `analysisResponseSchema` v1.0.0을 그대로 옮긴다 (summary 1–1000, scores 6종 int 0–10, detectedStructure, issues ≤ 6, structureComparison)
- `Issue` — paragraphId와 paragraphIndexAtAnalysis는 둘 다 null이거나 둘 다 값. excerpt ≤ 400, reason ≤ 300, suggestion ≤ 300, question ≤ 200, revisedExample ≤ 400
- Enum: WritingType 5 (`essay`, `cover_letter`, `report`, `university_assignment`, `free_writing`), LogicalStructure 10, ParagraphRole 12 (develop과 같은 값)

### 구조 단계와 추천

구조마다 기대하는 문단 역할 순서(`STRUCTURE_ROLES`)는 `entities/structure`에 두고 분석기(R3, R5)와 화면(새 문서의 빈 문단, 단계 칩, `+ 문단`)이 함께 쓴다.

| 구조 | 이름 | 단계 |
| --- | --- | --- |
| `intro_body_conclusion` | 서론·본론·결론 | 배경 → 설명 → 결론 |
| `claim_evidence_example_conclusion` | 주장·근거·예시·결론 | 주장 → 근거 → 예시 → 결론 |
| `problem_cause_solution_effect` | 문제·원인·해결·효과 | 문제 → 설명 → 해결 → 결과 |
| `star` | STAR | 배경 → 문제 → 행동 → 결과 |
| `prep` | PREP | 주장 → 설명 → 예시 → 결론 |
| `comparison` | 비교·대조 | 배경 → 주장 → 반론 → 결론 |
| `scqa` | SCQA | 배경 → 문제 → 설명 → 해결 |
| `four_act` | 기승전결 | 배경 → 행동 → 문제 → 결과 |
| `topic_first_and_last` | 양괄식 | 주장 → 근거 → 결론 |
| `free_structure` | 자유 구조 | 정해진 단계 없음 |

글 유형별 추천 구조(카드 순서대로, `RECOMMENDED_STRUCTURES`):

| 글 유형 | 추천 1 | 추천 2 | 추천 3 |
| --- | --- | --- | --- |
| 자기소개서 | STAR | PREP | 기승전결 |
| 논술 | 주장·근거·예시·결론 | 비교·대조 | 서론·본론·결론 |
| 보고서 | 문제·원인·해결·효과 | SCQA | 양괄식 |
| 대학 과제 | 서론·본론·결론 | 주장·근거·예시·결론 | 문제·원인·해결·효과 |
| 자유 글 | 자유 구조 | 기승전결 | 서론·본론·결론 |

한국어 이름: 글 유형은 논술, 자기소개서, 보고서, 대학 과제, 자유 글. 문단 역할은 주장, 근거, 예시, 설명, 반론, 재반박, 결론, 배경, 문제, 해결, 행동, 결과.

### 계약

```ts
interface SessionRepository {
  getSession(): Promise<Session | null>
  signIn(input: { email: string; password: string }): Promise<Session>
  signOut(): Promise<void>
}
interface DocumentRepository {
  list(): Promise<DocumentSummary[]>
  get(id: string): Promise<Document | null>
  create(input: { writingType: WritingType; structure: LogicalStructure }): Promise<Document>
  update(id: string, patch: { title?: string; content?: TipTapDoc }): Promise<Document>
  remove(id: string): Promise<void>          // 문서와 그 버전을 지운다
  createVersion(id: string, reason: VersionReason): Promise<DocumentVersion>
  listVersions(id: string): Promise<DocumentVersion[]>
}
interface AnalysisRepository {
  latest(documentId: string): Promise<Analysis | null>
  list(documentId: string): Promise<Analysis[]>
  save(input: Omit<Analysis, 'id' | 'createdAt'>): Promise<Analysis>
  removeAll(documentId: string): Promise<void>
}
interface AnalyzerPort {            // src/server/analyzer, 서버 전용
  analyze(request: AnalyzeRequest): Promise<AnalysisResponse>
}
```

구현: `LocalStorageSessionRepository`, `LocalStorageDocumentRepository`, `LocalStorageAnalysisRepository`(모두 `StorageAdapter` 사용, 키 접두사 `sogae:v1:`), `MockAnalyzer`. 구현은 `RepositoryProvider`(React Context)로 주입하고 테스트에서는 메모리 구현으로 바꾼다. `getAnalyzer()`는 환경변수 `ANALYZER`(기본 `mock`)로 구현을 고른다. 치우기는 `remove-document` 기능이 `DocumentRepository.remove`와 `AnalysisRepository.removeAll`을 차례로 부른다.

Mock 로그인은 이메일 형식과 비밀번호 8자 이상이면 통과한다. 처음 로그인하면 `seedDemoScenario()`가 데모 문서 2편(자기소개서 "데이터로 설득했던 순간", 논술 "원격 수업은 대면 수업을 대체할 수 있는가")과 분석 1건을 넣는다.

**계약에 없는 화면 정보**: 프로토타입은 지적마다 종류(구체성, 명료성, 구조, 논리, 어조 일관성)와 빠진 단계의 역할을 계약 밖 보조 정보로 들고 있다. 지적 사항의 종류 표시, 글쓰기 패턴 집계, 빠진 단계 지적의 "'역할' 문단 추가"가 이 정보를 쓴다. `AnalysisResponse` v1.0.0에는 이 필드가 없으므로, 분석 기록 옆에 화면용 보조 정보로 따로 둘지 계약 버전을 올릴지는 계획 2에서 정한다.

### 분석 요청 흐름

1. 피드백 창 "분석하기"(또는 "다시 분석") 클릭. 창이 닫혀 있으면 연다
2. `useAnalyzeDocument()` — 요청 중에는 버튼 비활성
3. `buildAnalyzeRequest(document)` — `readParagraphs(content)`로 `paragraphs[]`를 만든다. 빈 문단 제외, 최대 50개
4. `analyzeRequestSchema.parse` 후 `POST /api/analyze`
5. Route Handler — `analyzeRequestSchema.safeParse`, 실패 시 400 `VALIDATION`
6. `getAnalyzer().analyze()` — MockAnalyzer 규칙
   - R1 뭉뚱그린 표현(많은 노력, 열심히, 최선을 …) → specificity 감점, 지적
   - R2 80자를 넘는 문장 → clarity 감점, 지적
   - R3 선택 구조의 단계 역할 누락 → structure 감점, 지적, alignment 계산
   - R4 마지막 문단 역할이 conclusion·result가 아님 → 결론 누락 지적
   - R5 역할 순서와 가장 가까운 구조를 detectedStructure로 선택
   - 점수 = 10 − 감점(0–10으로 제한). 지적은 감점이 큰 순서로 최대 6개. 무작위 값과 현재 시각을 결과에 쓰지 않는다. 수정 예시에서 사용자만 아는 사실은 `[기간]` 같은 빈칸으로 남긴다
7. `analysisResponseSchema.safeParse(result)`, 실패 시 500 `ANALYSIS_CONTRACT` (원문은 로그에 남기지 않는다)
8. 200 `{ result, model: "mock", schemaVersion: "1.0.0" }`
9. 클라이언트에서 `analysisResponseSchema.parse`로 다시 검증
10. `AnalysisRepository.save()` — snapshot은 요청의 paragraphs
11. localStorage `sogae:v1:analyses:{documentId}`에 추가
12. `invalidateQueries(['analysis', documentId])`
13. 첫 지적을 펼친다. 지적 사항의 paragraphId가 현재 문서에 있으면 하이라이트와 집중 모드, 없으면 excerpt 스냅샷과 "현재 문단과 연결되지 않음"

예시 적용: 모두 `createVersion(id, 'before_apply_suggestion')`을 먼저 만든다.
- 문단에 걸린 지적: "예시로 바꾸기". 그 문단에 excerpt가 그대로 있을 때만 그 자리를 revisedExample로 바꾸고, 없으면 "지금 글에서 그 문장을 찾지 못했습니다"를 알린다
- 빠진 단계 지적: 그 역할의 빈 문단이 있으면 "빈 '역할' 문단에 넣기"로 그 문단에 revisedExample을 넣는다. 없으면 "'역할' 문단 추가"로 구조 순서상 맞는 자리(뒤 단계 문단의 바로 앞, 없으면 끝)에 새 문단을 넣는다
- 적용한 뒤 `update`하고, 기존 분석은 "이전 버전 기준"으로 표시한다

## 7. 오류 처리

| 상황 | 처리 |
| --- | --- |
| 로그인 입력 오류 | 입력창 아래 한 줄 안내, 입력값 유지, 첫 오류 칸으로 초점 |
| 보호 화면 직접 접근 | `/login?next=원래주소`, 로그인 후 복귀 |
| 없는 문서 | `not-found.tsx` — "문서를 찾을 수 없어요", 대시보드 링크 |
| `STORAGE_CORRUPTED` | 깨진 키만 비우고 "저장된 데이터를 읽지 못해 초기화했어요" |
| `STORAGE_UNAVAILABLE` | 상단 띠 "이 브라우저에서는 글이 저장되지 않아요" |
| 자동 저장 실패 | 머리 저장 상태 "저장 안 됨 · 다시 시도" |
| 분석할 문단 없음 | 요청을 보내지 않고 피드백 창에 안내 |
| 분석 400·500·네트워크 | 피드백 창에 오류 문장과 "다시 시도", 이전 분석 결과 유지 |
| 예상 못 한 오류 | 라우트별 `error.tsx`, "다시 시도" |

`AppError.code`: `VALIDATION`, `ANALYSIS_CONTRACT`, `NETWORK`, `STORAGE_CORRUPTED`, `STORAGE_UNAVAILABLE`, `NOT_FOUND`. 오류 문장은 무슨 일이 있었고 무엇을 하면 되는지 말한다.

## 8. 테스트

- 스키마: 글자 수 상한, issues 6개 제한, Issue null 짝 규칙 (develop 테스트 이식)
- 구조: `STRUCTURE_ROLES`, `nextStageRole` (다음 단계, 마지막 단계 유지, 첫 단계, 자유 구조)
- MockAnalyzer: R1–R5 규칙별 단위 테스트, 같은 입력의 결과 동일성, 점수 범위
- Route Handler: 200, 400, 500 응답
- 저장소: 읽기·쓰기, 구조 단계대로 만든 새 문서, 삭제, 같은 내용의 버전 중복 방지, 깨진 값 복구, 데모 시나리오 1회 주입
- `buildAnalyzeRequest`: 빈 문단 제외, 순서, 50개 제한
- 문단 확장: 분할(역할 유지)·병합·붙여넣기(역할 잇기)·외부 내용·문서 안 복사·실행 취소에서 id와 역할 규칙, `readParagraphs` 순서와 빈 문단 제외
- 화면: 로그인 → 대시보드 → 구조 선택 → 작업대 이동, 역할 묶음 표시와 묶음 역할 변경, `+ 문단`의 역할, 분석 중복 클릭 방지, 피드백 창 열기·닫기와 본문 폭, 지적 사항 펼침과 문단 하이라이트, 연결이 끊긴 지적 표시

TDD로 작성한다. 완료 조건은 `npm test`, `npm run lint`, `npm run build` 통과.

## 9. 작업 방식

- 초기 세팅(`create-next-app`, Tailwind·ESLint·Vitest 설정)은 실행할 명령과 생길 파일 목록을 보여주고 한 번에 승인받는다.
- 그 뒤 직접 작성하는 파일은 파일마다 전체 코드 또는 diff를 보여주고 승인받은 뒤 만든다.
- 저장소 규칙은 `AGENTS.md`에 적는다 (초기 세팅 단계에서 함께 제안).

## 10. 다음 단계 (이번 범위 밖)

Supabase 인증·저장소 구현 교체, LlmAnalyzer와 외부 전송 동의, 버전 비교·복원 화면, 모바일 바텀 시트, 회원가입, Playwright E2E.
