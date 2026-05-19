# 프로젝트 개요

Next.js 15 + Supabase 기반 인증 플로우 및 보호된 라우트 애플리케이션.

## 기술 스택

| 분류 | 기술 |
|------|------|
| 프레임워크 | Next.js 15 (App Router), React 19 |
| 언어 | TypeScript (strict) |
| 스타일 | Tailwind CSS, shadcn/ui |
| 백엔드 | Supabase (Auth, PostgreSQL) |
| 상태관리 | Zustand |
| 폼 | React Hook Form + Zod |

## 디렉토리 구조

```
app/
├── auth/           # 인증 관련 라우트 (로그인, 회원가입, 비밀번호)
├── protected/      # 로그인 후 접근 가능한 보호 라우트
├── instruments/    # 데모/테스트 페이지
├── layout.tsx      # 루트 레이아웃
└── page.tsx        # 홈 페이지
```

## 코드 규칙

- **컴포넌트**: PascalCase, `'use client'` 없으면 기본 Server Component
- **파일명**: kebab-case (예: `user-profile.tsx`)
- **들여쓰기**: 2칸
- **타입**: `any` 금지, 명시적 타입 선언 필수
- **Tailwind**: 유틸리티 우선, 커스텀 CSS 최소화

## Supabase 패턴

```typescript
// 서버 컴포넌트
import { createClient } from '@/utils/supabase/server'
const supabase = await createClient()

// 클라이언트 컴포넌트
import { createClient } from '@/utils/supabase/client'
const supabase = createClient()
```

- RLS(Row Level Security) 정책 항상 고려
- 민감한 작업은 서버 컴포넌트 또는 Server Action에서 처리
- `NEXT_PUBLIC_` 접두사 없는 환경변수는 서버에서만 사용

## 환경변수

```
NEXT_PUBLIC_SUPABASE_URL        # Supabase 프로젝트 URL
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY  # 클라이언트용 공개 키
```

민감한 키(`service_role` 등)는 `.env.local`에만 저장, 절대 커밋 금지.

---

# MCP 서버 활용 가이드

이 프로젝트는 `.mcp.json`에 4개 MCP 서버가 설정되어 있습니다.
Claude Code 재시작 시 자동 연결됩니다.

## 1. Supabase MCP

데이터베이스를 자연어로 조작할 수 있습니다.

**활용 예시:**
```
"users 테이블 스키마 보여줘"
"최근 가입한 유저 10명 조회해줘"
"RLS 정책 목록 확인해줘"
"새 테이블 생성 SQL 작성하고 실행해줘"
```

**사전 준비:** `.claude/settings.local.json`의 `SUPABASE_ACCESS_TOKEN` 입력
→ 발급: https://supabase.com/dashboard/account/tokens

## 2. GitHub MCP

이슈·PR·코드리뷰를 Claude와 함께 처리합니다.

**활용 예시:**
```
"현재 오픈된 이슈 목록 보여줘"
"이 변경사항으로 PR 만들어줘"
"#42 이슈에 코멘트 달아줘"
"main 브랜치 최근 커밋 10개 확인해줘"
```

**사전 준비:** `.claude/settings.local.json`의 `GITHUB_PERSONAL_ACCESS_TOKEN` 입력
→ 발급: https://github.com/settings/tokens (repo, issues 권한 필요)

## 3. Playwright MCP

브라우저를 Claude가 직접 조작합니다.

**활용 예시:**
```
"로그인 페이지 열고 스크린샷 찍어줘"
"회원가입 → 로그인 플로우 전체 테스트해줘"
"http://localhost:3000 에서 콘솔 에러 있는지 확인해줘"
"모바일 뷰포트로 보호된 페이지 렌더링 확인해줘"
```

**사전 준비:** 없음 (자동 설치). 테스트 전 `npm run dev` 실행 필요.

## 4. Context7 MCP

라이브러리 공식 문서를 실시간으로 조회합니다.

**활용 예시:**
```
"Next.js 15 use() 훅 사용법 알려줘"  
"Supabase realtime 구독 최신 API 보여줘"
"shadcn/ui Dialog 컴포넌트 예시 코드 줘"
"React 19 Server Actions 패턴 설명해줘"
```

**사전 준비:** 없음 (즉시 사용 가능).

---

# AI 협업 워크플로우

## 개발 시 권장 패턴

### 새 기능 개발
```
1. Context7로 관련 라이브러리 API 확인
2. 코드 작성
3. Playwright로 UI 동작 검증
4. /code-reviewer로 리뷰
5. /git-commit으로 커밋 & PR
```

### DB 스키마 변경
```
1. Supabase MCP로 현재 스키마 확인
2. 마이그레이션 SQL 작성
3. Supabase MCP로 적용 전 영향도 확인
4. 적용 후 RLS 정책 재검토
```

### 버그 수정
```
1. Playwright로 버그 재현
2. Supabase MCP로 데이터 상태 확인
3. /code-reviewer로 수정 코드 리뷰
4. Playwright로 수정 검증
```

## 서브에이전트 커맨드

| 커맨드 | 용도 |
|--------|------|
| `/code-reviewer` | 코드 리뷰 및 이슈 분석 |
| `/test-writer` | 테스트 코드 자동 생성 |
| `/git-commit` | 커밋 메시지 작성 & PR 생성 |
| `/doc-writer` | JSDoc & README 문서화 |

## Claude에게 맥락 제공하는 방법

효과적인 협업을 위해 다음 정보를 함께 제공하세요:
- **파일 경로**: `@app/auth/login/page.tsx`로 직접 참조
- **에러 메시지**: 전체 스택 트레이스 포함
- **기대 동작 vs 실제 동작**: 구체적으로 기술
- **Supabase 테이블명**: 관련 테이블을 명시하면 MCP 활용 효율 상승
