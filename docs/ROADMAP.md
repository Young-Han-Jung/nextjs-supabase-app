# 개발 로드맵 (ROADMAP)

> **버전:** 1.0.0
> **최종 수정:** 2026-05-19
> **기준 브랜치:** `master`

---

## 현재 상태 요약

```
[완료] 이메일/비밀번호 인증 전체 플로우
[완료] 보호된 라우트 (세션 기반 접근 제어)
[완료] 이메일 OTP 확인 및 비밀번호 재설정
[완료] 다크/라이트 테마 전환
[완료] shadcn/ui 컴포넌트 기반 UI
[진행] instruments 테이블 연동 (읽기만 구현)
[미착수] 사용자 프로필 관리
[미착수] 데이터 CRUD
[미착수] 폼 유효성 검사 (Zod)
[미착수] 소셜 로그인
```

---

## 개발 원칙

- **작게 자주 커밋** — PR 단위는 기능 하나
- **타입 먼저** — 구현 전 Zod 스키마 & TypeScript 타입 정의
- **서버 우선** — `'use client'`는 인터랙션이 필수인 경우만
- **RLS 동시 작성** — 테이블 생성과 RLS 정책은 같은 마이그레이션에
- **테스트 병행** — 기능 완료 후 `/test-writer`로 테스트 작성

---

## Phase 1 — 기반 다지기

> **목표:** 프로덕션 배포 가능한 수준으로 현재 코드 완성
> **기간:** 약 3~4주

### 1-1. 폼 유효성 검사 적용 `FR-SEC-02`

현재 4개 폼(`LoginForm`, `SignUpForm`, `ForgotPasswordForm`, `UpdatePasswordForm`) 모두 기본 HTML 검증만 사용 중.

**작업 목록:**
- [ ] `zod` + `react-hook-form` 설치
  ```bash
  npm install zod react-hook-form @hookform/resolvers
  ```
- [ ] `lib/validations/auth.ts` — 공통 Zod 스키마 정의
  - `loginSchema`: email(필수, 이메일 형식), password(필수, 8자 이상)
  - `signUpSchema`: loginSchema 확장 + repeatPassword 일치 검증
  - `forgotPasswordSchema`: email만
  - `updatePasswordSchema`: password + 강도 검사(대소문자, 숫자, 특수문자)
- [ ] 4개 폼 컴포넌트 `useForm` + `zodResolver`로 교체
- [ ] 에러 메시지 한국어화

**완료 기준:** 빈 값 제출, 잘못된 이메일 형식, 비밀번호 불일치 시 인라인 에러 표시

---

### 1-2. 토스트 알림 시스템 도입 `FR-UX-01`

**작업 목록:**
- [ ] shadcn/ui `Sonner` 컴포넌트 추가
  ```bash
  npx shadcn@latest add sonner
  ```
- [ ] `app/layout.tsx`에 `<Toaster />` 추가
- [ ] 인증 폼의 성공/에러 처리를 `toast()`로 교체
  - 로그인 성공 → "환영합니다!"
  - 비밀번호 재설정 이메일 발송 → "이메일을 확인하세요"
  - 에러 → `toast.error(message)`

**완료 기준:** 모든 인증 액션에 Toast 피드백 적용

---

### 1-3. `profiles` 테이블 및 트리거 생성 `FR-PROFILE-01`

**Supabase 마이그레이션:**
```sql
-- profiles 테이블
create table public.profiles (
  id uuid references auth.users(id) on delete cascade primary key,
  nickname text,
  avatar_url text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- RLS 활성화
alter table public.profiles enable row level security;

create policy "본인 프로필 읽기" on public.profiles
  for select using (auth.uid() = id);

create policy "본인 프로필 수정" on public.profiles
  for update using (auth.uid() = id);

-- 회원가입 시 자동 생성 트리거
create function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id) values (new.id);
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
```

**작업 목록:**
- [ ] 위 SQL 마이그레이션 Supabase 대시보드에서 실행
- [ ] `lib/types/database.ts` — DB 타입 정의 (Supabase CLI로 자동 생성 권장)
- [ ] `instruments` 테이블에 `user_id` 컬럼 추가 및 RLS 정책 적용

**완료 기준:** 신규 회원가입 시 `profiles` 레코드 자동 생성 확인

---

### 1-4. 세션 만료 처리 `FR-AUTH-03`

**작업 목록:**
- [ ] `components/auth-provider.tsx` — `onAuthStateChange` 리스너로 세션 감지
- [ ] 세션 만료 이벤트 시 Toast 알림 후 `/auth/login` 리다이렉트
- [ ] `app/layout.tsx`에 AuthProvider 적용

**완료 기준:** 토큰 만료 후 보호 라우트 접근 시 알림 + 자동 리다이렉트

---

## Phase 2 — 핵심 기능 구현

> **목표:** 실제 서비스 가능한 사용자 기능 완성
> **기간:** Phase 1 완료 후 약 4~6주

### 2-1. 사용자 프로필 페이지 `FR-PROFILE-01` `FR-PROFILE-02`

**새 라우트:** `/protected/profile`

**작업 목록:**
- [ ] `app/protected/profile/page.tsx` — 프로필 조회/수정 페이지
- [ ] `components/profile-form.tsx` — 닉네임 수정 폼 (Zod 스키마 포함)
- [ ] `components/avatar-upload.tsx` — 이미지 드래그&드롭 업로드
  - Supabase Storage `avatars` 버킷 생성 (public)
  - 업로드 후 `profiles.avatar_url` 업데이트
- [ ] 내비게이션에 프로필 링크 추가 (`DropdownMenu` 활용)
- [ ] 계정 삭제 기능 (확인 다이얼로그 포함) `FR-AUTH-04`

**완료 기준:** 프로필 사진 변경 및 닉네임 수정 후 즉시 반영

---

### 2-2. 데이터 CRUD 완성 `FR-DASH-02`

**기존 `/instruments` → 완전한 CRUD 페이지로 개편**

**DB 스키마 업데이트:**
```sql
alter table public.instruments
  add column if not exists user_id uuid references auth.users(id) on delete cascade,
  add column if not exists name text not null,
  add column if not exists description text,
  add column if not exists created_at timestamptz default now();

-- RLS: 본인 데이터만
alter table public.instruments enable row level security;

create policy "본인 항목 전체 접근" on public.instruments
  for all using (auth.uid() = user_id);
```

**작업 목록:**
- [ ] `app/protected/instruments/page.tsx` — 목록 페이지 (Server Component)
  - 무한 스크롤 또는 페이지네이션
  - 검색 (`?q=` 쿼리 파라미터)
  - 정렬 옵션 (최신순, 이름순)
- [ ] `app/protected/instruments/new/page.tsx` — 추가 폼
- [ ] `app/protected/instruments/[id]/page.tsx` — 상세/수정 페이지
- [ ] Server Actions (`app/protected/instruments/actions.ts`)
  - `createInstrument(formData)`
  - `updateInstrument(id, formData)`
  - `deleteInstrument(id)`
- [ ] `components/instrument-card.tsx` — 목록 아이템 카드 컴포넌트
- [ ] 낙관적 업데이트 (삭제 시 즉시 목록에서 제거)

**완료 기준:** 항목 추가/수정/삭제 전 과정 동작, 다른 사용자 데이터 접근 시 403

---

### 2-3. 대시보드 개편 `FR-DASH-01`

**`/protected` 페이지를 실제 대시보드로 교체**

**작업 목록:**
- [ ] 현재 JWT Claims 표시 제거
- [ ] `components/dashboard/stats-card.tsx` — 수치 요약 카드
- [ ] `components/dashboard/recent-activity.tsx` — 최근 활동 목록
- [ ] 빠른 액션 버튼 (새 항목 추가 등)
- [ ] 환영 메시지 (`profiles.nickname` 활용)

**완료 기준:** 로그인 직후 의미 있는 정보가 있는 대시보드 표시

---

### 2-4. 소셜 로그인 `FR-AUTH-02`

**작업 목록:**
- [ ] Supabase 대시보드 → Authentication → Providers에서 활성화
  - Google OAuth 앱 생성 (Google Cloud Console)
  - GitHub OAuth 앱 생성 (GitHub Developer Settings)
- [ ] `components/social-login-buttons.tsx` — Google/GitHub 버튼
- [ ] `LoginForm`과 `SignUpForm`에 소셜 버튼 추가
- [ ] 콜백 처리: `/auth/confirm` 라우트 재활용
- [ ] 소셜 로그인 시 `profiles` 레코드 자동 생성 확인

**완료 기준:** Google/GitHub 계정으로 가입/로그인 후 대시보드 진입

---

## Phase 3 — 고도화

> **목표:** 성능, 접근성, 운영 안정성 개선
> **기간:** Phase 2 완료 후 지속

### 3-1. 성능 최적화

- [ ] `next/image` 적용 (아바타, 콘텐츠 이미지 전체)
- [ ] React `Suspense` + `loading.tsx` 경계 세분화
- [ ] 목록 페이지 `generateStaticParams` 또는 ISR 검토
- [ ] Lighthouse CI 파이프라인 구성 (점수 90+ 유지)
- [ ] Bundle Analyzer 도입 (`@next/bundle-analyzer`)

### 3-2. 접근성 개선 `FR-UX-02`

- [ ] 전체 폼 요소 `aria-label` / `aria-describedby` 추가
- [ ] 키보드 포커스 순서 검토 (`Tab` 순서)
- [ ] 색상 대비 감사 (Tailwind 팔레트 기준)
- [ ] 스크린리더 테스트 (NVDA / macOS VoiceOver)

### 3-3. 테스트 인프라

- [ ] Vitest 설치 및 설정
  ```bash
  npm install -D vitest @testing-library/react @testing-library/user-event jsdom
  ```
- [ ] 단위 테스트: Zod 스키마, 유틸 함수
- [ ] 컴포넌트 테스트: `LoginForm`, `SignUpForm`, 등
- [ ] E2E 테스트: Playwright로 인증 전체 플로우 자동화
  - 회원가입 → 이메일 인증 → 로그인 → 로그아웃
  - 비밀번호 재설정 플로우
- [ ] GitHub Actions CI 구성 (PR 시 테스트 자동 실행)

### 3-4. 모니터링 및 에러 추적

- [ ] Sentry 도입 (클라이언트/서버 에러 추적)
- [ ] Vercel Analytics 활성화 (Core Web Vitals)
- [ ] Supabase 대시보드 DB 쿼리 성능 모니터링
- [ ] 에러 로그 알림 설정 (Slack 또는 이메일)

### 3-5. 관리자 패널 (선택)

- [ ] `/admin` 라우트 (별도 RLS: `is_admin` 컬럼 기반)
- [ ] 사용자 목록 및 상태 관리
- [ ] 데이터 통계 차트 (`recharts` 또는 `tremor`)

---

## 기술 부채 목록

아래 항목은 현재 코드에서 식별된 개선 필요 사항입니다.

| 항목 | 위치 | 우선순위 | 설명 |
|------|------|----------|------|
| JWT Claims 노출 | `app/protected/page.tsx:16` | 높음 | 개발용 코드, 프로덕션 전 제거 필요 |
| 에러 타입 불명확 | 모든 폼 컴포넌트 | 중간 | `catch (error: unknown)` → 명확한 타입 가드로 개선 |
| 하드코딩된 리다이렉트 | `components/sign-up-form.tsx:47` | 중간 | `window.location.origin` → 환경변수로 추출 |
| 튜토리얼 컴포넌트 제거 | `components/tutorial/` | 낮음 | 실제 서비스 배포 전 삭제 (`FetchDataSteps` 등) |
| `instruments` 테이블 RLS 미적용 | DB | 매우 높음 | 현재 인증 없이 전체 데이터 조회 가능 |
| `DeployButton` 제거 | `app/page.tsx`, `app/protected/layout.tsx` | 낮음 | Vercel 템플릿 잔재, 실서비스 불필요 |

---

## 버전 히스토리

| 버전 | 날짜 | 내용 |
|------|------|------|
| 1.0.0 | 2026-05-19 | 초기 로드맵 작성 — 현재 상태 기반 Phase 1~3 정의 |
