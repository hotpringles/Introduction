# 소개 · SOGAE

생각을 대신 쓰지 않고, 사용자의 논리와 문체를 발견하도록 돕는 글쓰기 코칭 서비스입니다.

AI는 글 전체를 대신 쓰지 않습니다. `진단 → 설명 → 질문 → 개선 제안` 흐름으로 사용자가 스스로 더 잘 쓰도록 돕습니다.

## 현재 상태

설계 단계입니다. 이전 Vite + React 구현([hotpringles/SOGAE](https://github.com/hotpringles/SOGAE) `develop` 브랜치)을 바탕으로, Editorial 디자인을 Next.js로 처음부터 다시 만듭니다.

첫 목표는 로그인, 대시보드, 구조 선택, 에디터와 AI 피드백 네 화면을 Mock 데이터로 실제 동작시키는 것입니다. 이후 Supabase 인증·저장소와 실제 LLM 분석으로 교체합니다.

## 문서

- 설계: [`docs/specs/2026-09-26-nextjs-rebuild-design.md`](docs/specs/2026-09-26-nextjs-rebuild-design.md)
- 구현 계획: `docs/plans/` (작성 예정)

## 기술 스택 (예정)

TypeScript · Next.js App Router · Tailwind CSS 4 · TanStack Query · Zustand · React Hook Form · Zod · TipTap

테스트는 Vitest · React Testing Library · jsdom을 사용합니다.

## 실행

초기 세팅 후 이 절에 설치와 실행 방법, 검증 명령(`npm test`, `npm run lint`, `npm run build`)을 적습니다.
