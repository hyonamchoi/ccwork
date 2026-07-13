# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 목적

React 19 + TypeScript + Vite 기반 **노트 앱 실습 프로젝트**. Claude Code 학습 목적으로 사용하는 코드베이스다. `Note` 타입에 `tags` 필드가 의도적으로 빠져 있으며, 강의 실습 중 추가 예정이다.

## 개발 명령어

```bash
npm run dev          # Vite(5173) + JSON Server(3001) 동시 실행
npm run build        # tsc + vite build
npm run lint         # ESLint --fix
npm run format       # Prettier
npm test             # Vitest (단발 실행)
npm run test:watch   # Vitest (감시 모드)
npm run server       # JSON Server만 단독 실행
```

- 앱: http://localhost:5173
- API: http://localhost:3001/notes

## 아키텍처

```
src/
├── api/notes.ts          # fetch 래퍼 (CRUD) — 외부 경계
├── types/note.ts         # Note 인터페이스 단일 정의
├── context/NotesContext.tsx  # 전역 상태 + 비즈니스 로직
├── components/
│   ├── Layout.tsx        # 2단 레이아웃 (sidebar | main) + 헤더
│   ├── NoteList.tsx      # 노트 목록
│   ├── NoteItem.tsx      # 개별 노트 행 (삭제 포함)
│   └── NoteEditor.tsx    # 생성/편집 폼
└── App.tsx               # 선택 상태(selectedNoteId, isCreating) 관리
```

**데이터 흐름**: `api/notes.ts` → `NotesContext` (낙관적 업데이트) → 컴포넌트

- `NotesContext`가 `notes[]`, `loading`, `error`, `addNote/editNote/removeNote`를 노출
- `App.tsx`는 선택 UI 상태만 관리하고, 데이터 조작은 모두 Context에 위임
- `db.json`이 JSON Server의 영구 저장소 역할 (git 추적됨)

## 기술 스택

| 도구 | 버전 | 비고 |
|------|------|------|
| React | 19 | |
| TypeScript | 5.7 | strict 모드 |
| Vite | 6 | |
| Tailwind CSS | 4 | `@tailwindcss/vite` 플러그인 방식 |
| JSON Server | 1.0.0-beta | mock REST API |
| Vitest | 3 | jsdom 환경, globals: true |
| Testing Library | 16 | `src/test-setup.ts`에 jest-dom 설정 |

## 테스트 작성 패턴

- `vitest.config`는 `vite.config.ts` 내 `test` 블록에 통합
- 테스트 파일 위치: `src/` 하위 어디든 (`*.test.tsx` / `*.spec.ts`)
- `useNotes` 훅 테스트 시 반드시 `NotesProvider`로 래핑
- API 호출은 `vi.mock('../api/notes')` 로 모킹

## 구현 패턴

### 컴포넌트
- `export function XxxComponent` 함수 선언식 + named export
- props 타입은 `interface XxxProps`로 컴포넌트 바로 위에 선언
- 상태 없는 표시 컴포넌트(`NoteItem`, `Layout`)와 데이터 소비 컴포넌트(`NoteList`, `NoteEditor`) 명확히 구분
- 조건부 early return 순서: `loading` → `error` → 빈 상태 → 정상 렌더

### 상태 관리
- 전역 **데이터 상태** (`notes[]`, `loading`, `error`): `NotesContext`에서 관리
- 로컬 **UI 선택 상태** (`selectedNoteId`, `isCreating`): `App.tsx`에서 관리
- 낙관적 업데이트 패턴: API 응답 값으로 `setNotes` 직접 반영
  - create → `prev => [...prev, newNote]`
  - update → `prev.map(n => n.id === id ? updated : n)`
  - delete → `prev.filter(n => n.id !== id)`

### API 호출
- 모든 fetch 로직은 `src/api/notes.ts`에만 위치
- 패턴: `if (!res.ok) throw new Error('...')` 후 `return res.json()`
- `createdAt` / `updatedAt` 타임스탬프는 **클라이언트**에서 `new Date().toISOString()`으로 생성 후 전송 (서버 미처리)
- `createNote` 인자 타입: `Omit<Note, 'id' | 'createdAt' | 'updatedAt'>` — id는 JSON Server 자동 생성

### 네이밍
| 분류 | 패턴 | 예시 |
|------|------|------|
| 컴포넌트/타입 | PascalCase | `NoteEditor`, `NoteItemProps` |
| 이벤트 핸들러 (로컬) | `handleXxx` | `handleSave`, `handleSelectNote` |
| 이벤트 prop | `onXxx` | `onSelect`, `onDelete`, `onDone` |
| boolean prop/state | `isXxx` | `isCreating`, `isSelected` |
| API 함수 | `verbNoun` camelCase | `fetchNotes`, `createNote` |
| Context hook | `useXxx` | `useNotes` |

## 코딩 규칙

- Named export만 사용 (`App.tsx`의 default export 제외)
- `useNotes` 훅 외부에서 직접 `NotesContext`에 접근하지 않음
- Tailwind 클래스 우선, 동적 계산 불가능한 경우에만 인라인 `style` 허용
- 에러 처리는 `console.error()`만 사용, `alert()` 금지

## ⚠️ 코드베이스 내 패턴 불일치

새 코드 작성 시 기존 코드와 맞춰야 하는 항목들이 아직 통일되지 않았다.

**1. boolean 상태 네이밍**: `isXxx` 규칙이 내부 state에서 깨짐
- props: `isCreating`, `isSelected` → `is-` prefix 있음
- `NoteEditor` 내부 state: `saving` → `isSaving`이어야 일관성이 맞음

**2. 인라인 `style` 사용**: `Layout.tsx`에 두 군데 있음
- `style={{ height: 'calc(100vh - 65px)' }}` — Tailwind `h-[calc(100vh-65px)]`로 대체 가능
- `style={{ fontFamily: 'Boogaloo, sans-serif' }}` — Tailwind `font-` 커스텀 설정 필요

**3. 에러 처리 방식 혼재**
- `fetchNotes` 실패 → Context의 `error` state로 처리 (UI 렌더)
- `addNote` / `editNote` 실패 → 컴포넌트(`NoteEditor`)의 try/catch + `console.error()`
- `removeNote` 실패 → `NoteItem`에서 catch 없이 전파됨 (에러 처리 누락)