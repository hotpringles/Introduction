# 소개 SOGAE · Next.js 재구축 설계

작성일: 2026-09-26
상태: 검토 대기
관련 자료
- 디자인: claude.ai 캔버스 "SOGAE 서비스 화면 디자인"의 Editorial 페이지 (로그인, 대시보드, 구조 선택, 에디터)
- 도식: Obsidian `소개/SOGAE 데이터 계약과 분석 흐름.excalidraw.md`
- 이전 구현: github.com/hotpringles/SOGAE `develop` 브랜치 (Vite + React)

## 1. 목표

소개는 사용자가 직접 쓴 글의 논리 구조와 글쓰기 습관을 분석하고, 사용자의 문체를 유지하면서 개선 방향을 제안하는 글쓰기 코칭 서비스다. AI는 글을 대신 쓰지 않고 `진단 → 설명 → 질문 → 개선 제안` 흐름으로 돕는다.

이번 범위는 Editorial 디자인의 네 화면을 Next.js로 새로 만들고, 로그인·문서·분석을 Mock으로 실제 동작시키는 것이다. 이후 Supabase와 실제 LLM으로 바꿀 때 교체할 경계를 지금 만들어 둔다.

### 완료 조건

- 로그인 → 대시보드 → 구조 선택 → 에디터 작성 → 분석 → 피드백 확인이 끊김 없이 동작한다.
- 새로고침 후에도 문서, 문단 역할, 분석 결과가 남아 있다.
- 1280px 이상 데스크톱에서 캔버스 디자인과 같은 배치·글꼴·색을 쓴다.
- `npm test`, `npm run lint`, `npm run build`가 모두 통과한다.

### 이번 범위에서 제외

회원가입, 비밀번호 찾기, 구글 로그인(버튼은 비활성으로 표시), 히스토리·마이페이지, 버전 비교(diff)와 복원 화면, 모바일 바텀 시트, Supabase 연동, 실제 LLM 연동, Playwright E2E.

## 2. 기술 스택

| 영역 | 선택 |
| --- | --- |
| 프레임워크 | Next.js App Router (초기 세팅 시점의 최신 안정 버전), TypeScript, Node 22 |
| 스타일 | Tailwind CSS 4, Editorial 디자인 토큰을 `@theme`에 정의 |
| 글꼴 | `next/font/google`로 Work Sans, Noto Sans KR, Noto Serif KR |
| 서버 상태 | TanStack Query |
| 클라이언트 UI 상태 | Zustand (피드백 창 열림, 선택 탭, 펼친 지적 사항) |
| 검증 | Zod (입력, 저장소 경계, API 요청·응답) |
| 폼 | React Hook Form + Zod resolver |
| 에디터 | TipTap, 문단 식별자 확장 `ParagraphId` |
| 테스트 | Vitest, React Testing Library, jsdom |
| 배포 | Vercel |

## 3. 디자인 토큰

| 토큰 | 값 | 쓰임 |
| --- | --- | --- |
| `--color-paper` | `#efeeea` | 페이지 배경 |
| `--color-paper-raised` | `#f7f6f2` | 추천 구조 카드 |
| `--color-tile` | `#d9d8d3` | 최근 문서 타일 |
| `--color-ink` | `#111111` | 본문, 괘선 |
| `--color-ink-2` | `#333333` | 설명 문단 |
| `--color-muted` | `#555555` | 보조 글씨 |
| `--color-rule-soft` | `#bdbcb7` | 옅은 괘선 |
| `--color-accent` | `#c2410c` | 주요 버튼, 선택 표시, 하이라이트 |
| `--color-accent-strong` | `#9a3412` | 옅은 배경 위 강조 글씨 |
| `--radius-control` | `4px` | 버튼, 입력창 |
| `--shadow-lift` | `0 14px 36px rgb(194 65 12 / .18), 0 3px 8px rgb(194 65 12 / .10)` | 선택된 구조 카드 |

글꼴: 제목·숫자는 Work Sans 900 + Noto Sans KR 900, 본문은 Noto Sans KR 400, 워드마크·인용·드롭캡은 Noto Serif KR 600. 본문 문단은 양쪽 정렬, `word-break: keep-all`.

## 4. 구조

### 라우트

```text
app/
  (auth)/login/page.tsx
  (app)/layout.tsx               EditorialHeader, QueryClient, RepositoryProvider
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
  widgets/     editorial-header, feedback-panel, document-outline, recent-documents
  features/    sign-in, select-structure, edit-document, analyze-document, apply-suggestion
  entities/    session, document, analysis, structure
  shared/      ui, api(storage-adapter, app-error), lib(demo-scenario), config
  server/      analyzer (서버 전용, import "server-only")
```

의존 방향은 `app → views → widgets → features → entities → shared`로 고정하고 ESLint `no-restricted-imports`로 역방향을 막는다. `src/server`는 `app/api`에서만 불러온다. 슬라이스 밖에서는 각 슬라이스의 `index.ts`만 불러온다.

## 5. 화면

### 공통 머리글 `EditorialHeader`
높이 80px, 좌·중·우 세 칸. 왼쪽은 명조 워드마크 "소개". 가운데는 화면별 슬롯(대시보드·구조 선택: 오늘 날짜, 에디터: 서식 툴바 + AI 피드백 토글). 오른쪽은 메뉴(대시보드, 새 글 쓰기, 로그아웃) 또는 에디터 상태(저장됨, 버전 저장, 대시보드).

### 로그인 `/login`
왼쪽: 72px 제목 "다시 만나서 반갑습니다", 서비스 소개 2단 칼럼, 명조 인용구. 세로 괘선. 오른쪽: 이메일·비밀번호 폼(Zod 즉시 검증), 주황 로그인 버튼, 비활성 구글 버튼.

### 대시보드 `/dashboard`
64px 인사말 아래 네 칸 격자(`300px | 1fr | 1px | 340px`).
- 최근 문서: 회색 타일 2장, 원형 화살표 버튼
- 이번 주: 날짜, 160px 최근 점수, 요약 문단, 새 글 쓰기 버튼, 지적 사항 링크
- 괘선
- 분석과 글 유형: 분석 요약, 항목별 점수표(가장 낮은 항목 표시), 글 유형 목록
글쓰기 패턴은 문서가 3편 미만이면 "준비 중"을 표시한다.

### 구조 선택 `/write/structure`
60px 제목. 왼쪽 240px 글 유형 목록, 오른쪽에 추천 구조 카드 3장과 다른 구조 7개. 선택한 카드는 배경을 바꾸지 않고 `--shadow-lift`와 4px 떠오름으로 표시한다. 카드의 설명 문단은 카드 아래에 붙여 세 카드의 높이를 맞춘다. 하단 막대의 "이 구조로 작성 시작"은 문서를 만들고 에디터로 이동한다.

### 에디터 `/documents/[id]/edit`
격자 `220px | 1fr | 1px | 400px`, 행 높이 `minmax(0, 1fr)`.
- 문서 정보: 제목, 글 유형·구조, 문단 목록(지적 사항 있는 문단 표시)
- 본문: TipTap, 첫 문단 드롭캡, 문단마다 역할 선택 버튼, 800ms 자동 저장. 피드백 창이 열려 있으면 최대 600px, 닫히면 남은 폭 전체
- AI 피드백 패널: 머리(제목, 다시 분석) 고정, 그 아래만 스크롤. 스크롤 영역에 점수, 요약, 항목별 점수, 위에 붙어 따라오는 탭(지적 사항 / 구조 비교), 각주식 지적 사항 목록. 아래쪽 48px 페이드
- 토글: 머리글 서식 툴바 옆 버튼 하나. 닫으면 패널과 괘선이 사라지고 집중 모드·하이라이트가 꺼진다
- 집중 모드: 펼친 지적 사항의 문단만 불투명도 1, 나머지 0.35

### 반응형
1280px 이상은 디자인 그대로. 1024px 미만은 칸을 세로로 쌓고, 에디터 피드백 패널은 본문 아래 전체 폭 영역이 된다.

## 6. 데이터

### 엔티티

- `User { id, email, displayName, createdAt }`
- `Session { userId, issuedAt, expiresAt }` — localStorage와 쿠키 `sogae_session`에 함께 저장
- `Document { id, userId, title(0–100), writingType, structure, content: TipTapDoc, createdAt, updatedAt }`
- `TipTapDoc` — `doc > paragraph[attrs: { paragraphId, role }] > text`. 문단 역할은 따로 저장하지 않고 문단 노드 속성으로 본문 안에 둔다
  - `paragraphId`: string(1–128), `role`: ParagraphRole | null
  - 새 문단은 새 uuid. 분할하면 뒤 문단이 새 id와 역할 없음으로 시작하고(`keepOnSplit: false`), 병합하면 앞 문단의 id와 역할을 유지한다
  - id가 없거나 겹치면 플러그인이 새 id를 붙인다. 문서 안에서 복사한 문단이 원본보다 앞에 붙어도 원본이 id를 유지한다(이전 상태의 위치를 트랜잭션 매핑으로 추적)
  - 역할 변경은 `setParagraphRole(paragraphId, role)` 명령으로 하며 실행 취소에 포함된다
  - 문단 순서는 저장하지 않고 `readParagraphs(doc)`가 계산한다. 문서 정보 칸, 분석 요청, 구조 비교 탭이 이 함수를 같이 쓴다
- `DocumentSummary { id, title, writingType, structure, updatedAt, latestScore: number | null, excerpt: string }` — 대시보드 최근 문서 타일용. latestScore는 최근 분석 6개 점수의 평균(소수 첫째 자리), excerpt는 첫 문단 앞 60자
- `DocumentVersion { id, documentId, content, reason: VersionReason, createdAt }` — 버전 저장, 예시 적용 직전, 복원 직전에만 만든다
- `VersionReason` = `"manual" | "before_apply_suggestion" | "before_restore"`
- `Analysis { id, documentId, versionId | null, snapshot: ParagraphSnapshot[], schemaVersion: "1.0.0", model, promptVersion | null, result: AnalysisResponse, createdAt }`
- `ParagraphSnapshot { paragraphId(1–128), index(int ≥ 0), role | null, text(1–2000) }`
- `AnalysisResponse` — develop의 `analysisResponseSchema` v1.0.0을 그대로 옮긴다 (summary 1–1000, scores 6종 int 0–10, detectedStructure, issues ≤ 6, structureComparison)
- `Issue` — paragraphId와 paragraphIndexAtAnalysis는 둘 다 null이거나 둘 다 값. excerpt ≤ 400, reason ≤ 300, suggestion ≤ 300, question ≤ 200, revisedExample ≤ 400
- Enum: WritingType 5, LogicalStructure 10, ParagraphRole 12 (develop과 같은 값)

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
  createVersion(id: string, reason: VersionReason): Promise<DocumentVersion>
  listVersions(id: string): Promise<DocumentVersion[]>
}
interface AnalysisRepository {
  latest(documentId: string): Promise<Analysis | null>
  list(documentId: string): Promise<Analysis[]>
  save(input: Omit<Analysis, 'id' | 'createdAt'>): Promise<Analysis>
}
interface AnalyzerPort {            // src/server/analyzer, 서버 전용
  analyze(request: AnalyzeRequest): Promise<AnalysisResponse>
}
```

구현: `LocalStorageSessionRepository`, `LocalStorageDocumentRepository`, `LocalStorageAnalysisRepository`(모두 `StorageAdapter` 사용, 키 접두사 `sogae:v1:`), `MockAnalyzer`. 구현은 `RepositoryProvider`(React Context)로 주입하고 테스트에서는 메모리 구현으로 바꾼다. `getAnalyzer()`는 환경변수 `ANALYZER`(기본 `mock`)로 구현을 고른다.

Mock 로그인은 이메일 형식과 비밀번호 8자 이상이면 통과한다. 처음 로그인하면 `seedDemoScenario()`가 데모 문서 2편(자기소개서 "데이터로 설득했던 순간", 논술 "원격 수업은 대면 수업을 대체할 수 있는가")과 분석 1건을 넣는다.

### 분석 요청 흐름

1. 피드백 패널 "다시 분석" 클릭
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
13. 지적 사항의 paragraphId가 현재 문서에 있으면 하이라이트와 집중 모드, 없으면 excerpt 스냅샷과 "현재 문단과 연결되지 않음"

예시 적용: `createVersion(id, 'before_apply_suggestion')` → excerpt 위치를 revisedExample로 교체 → `update` → 기존 분석은 "이전 버전 기준"으로 표시.

## 7. 오류 처리

| 상황 | 처리 |
| --- | --- |
| 로그인 입력 오류 | 폼 아래 한 줄 안내, 입력값 유지 |
| 보호 화면 직접 접근 | `/login?next=원래주소`, 로그인 후 복귀 |
| 없는 문서 | `not-found.tsx` — "문서를 찾을 수 없어요", 대시보드 링크 |
| `STORAGE_CORRUPTED` | 깨진 키만 비우고 "저장된 데이터를 읽지 못해 초기화했어요" |
| `STORAGE_UNAVAILABLE` | 상단 띠 "이 브라우저에서는 글이 저장되지 않아요" |
| 자동 저장 실패 | 머리글 상태 "저장 안 됨 · 다시 시도" |
| 분석 400·500·네트워크 | 패널에 오류 문장과 "다시 시도", 이전 분석 결과 유지 |
| 예상 못 한 오류 | 라우트별 `error.tsx`, "다시 시도" |

`AppError.code`: `VALIDATION`, `ANALYSIS_CONTRACT`, `NETWORK`, `STORAGE_CORRUPTED`, `STORAGE_UNAVAILABLE`, `NOT_FOUND`. 오류 문장은 무슨 일이 있었고 무엇을 하면 되는지 말한다.

## 8. 테스트

- 스키마: 글자 수 상한, issues 6개 제한, Issue null 짝 규칙 (develop 테스트 이식)
- MockAnalyzer: R1–R5 규칙별 단위 테스트, 같은 입력의 결과 동일성, 점수 범위
- Route Handler: 200, 400, 500 응답
- 저장소: 읽기·쓰기, 깨진 값 복구, 데모 시나리오 1회 주입
- `buildAnalyzeRequest`: 빈 문단 제외, 순서, 50개 제한
- 문단 확장: 분할·병합·외부 붙여넣기·문서 안 복사·실행 취소에서 id와 역할 규칙, `readParagraphs` 순서와 빈 문단 제외
- 화면: 로그인 → 대시보드 → 구조 선택 → 에디터 이동, 분석 중복 클릭 방지, 피드백 창 열기·닫기와 본문 폭, 지적 사항 펼침과 문단 하이라이트, 연결이 끊긴 지적 표시

TDD로 작성한다. 완료 조건은 `npm test`, `npm run lint`, `npm run build` 통과.

## 9. 작업 방식

- 초기 세팅(`create-next-app`, Tailwind·ESLint·Vitest 설정)은 실행할 명령과 생길 파일 목록을 보여주고 한 번에 승인받는다.
- 그 뒤 직접 작성하는 파일은 파일마다 전체 코드 또는 diff를 보여주고 승인받은 뒤 만든다.
- 저장소 규칙은 `AGENTS.md`에 적는다 (초기 세팅 단계에서 함께 제안).

## 10. 다음 단계 (이번 범위 밖)

Supabase 인증·저장소 구현 교체, LlmAnalyzer와 외부 전송 동의, 버전 비교·복원 화면, 모바일 바텀 시트, 회원가입, Playwright E2E.
