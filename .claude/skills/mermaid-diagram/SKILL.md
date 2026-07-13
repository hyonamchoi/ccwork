---
name: mermaid-diagram
description: Generate and view a Mermaid-based architecture diagram for this project. Use when asked to visualize architecture, component dependencies, state flow, or structure of the src/ directory. Produces docs/architecture/index.html and opens it in a browser or takes a screenshot.
---

# mermaid-diagram

`src/`를 분석해 컴포넌트 의존성과 상태 흐름을 Mermaid 기반 HTML로 생성하고,
`npx serve`로 서빙한 뒤 Chrome 헤드리스 스크린샷으로 검증한다.

출력: `docs/architecture/index.html` + `screenshot.png` (이 스킬 디렉토리)

---

## Prerequisites

- Node.js / npm (프로젝트 이미 설치됨)
- Chrome: `C:\Program Files\Google\Chrome\Application\chrome.exe`

---

## 분석 단계

`src/` 하위에서 다음 항목을 파악한다:

| 항목 | 확인 방법 |
|------|----------|
| 컴포넌트 렌더 트리 | `src/components/*.tsx` JSX 반환 읽기 |
| Context / 전역 상태 | `src/context/*.tsx` — Provider가 노출하는 값과 CRUD 함수 |
| API 경계 | `src/api/*.ts` — fetch 래퍼 함수, 타임스탬프 주입 여부 |
| 타입 정의 | `src/types/*.ts` — 공유 인터페이스 |
| 로컬 상태 | 각 컴포넌트 `useState` 목록 |

---

## Diagram 설계 규칙

**Diagram 1 — 컴포넌트 의존성** (`graph TD`)
- 실선 `-->` : 렌더 관계 (부모 → 자식)
- 점선 `-.->` : `useXxx()` 훅 구독
- 굵은 선 `==>` : REST API 호출
- `---` : 타입 참조 (import only)
- 레이어별 subgraph + 색상 구분:
  - App Layer `fill:#2d1f3d`
  - State Layer `fill:#1e2d4a`
  - UI Layer `fill:#1a2d1a`

**Diagram 2 — 상태 흐름** (`flowchart LR`)
- 상태를 세 그룹으로 분류: App UI State / Context Server State / 컴포넌트 Local State
- 각 상태가 영향을 주는 컴포넌트를 화살표로 연결
- 레이블로 영향 종류 명시 ("목록 렌더", "폼 초기값 결정" 등)

**Reference 섹션**
- 파일별 역할 카드 그리드
- 레이어 태그: `State`, `UI`, `API`, `Type`

---

## HTML 생성 규칙

- Mermaid.js CDN: `mermaid@10`
- `mermaid.initialize({ theme: 'dark', ... })` — 다크 테마 필수
- 저장: `docs/architecture/index.html`
- `docs/architecture/` 디렉토리 없으면 먼저 생성

---

## Run (Agent path) — 서버 + 스크린샷

```bash
# 1. 서버 기동 (백그라운드)
npx serve docs/architecture --listen 4000 &

# 2. 서버 응답 확인
curl -s -o /dev/null -w "%{http_code}" http://localhost:4000
# → 200 이면 정상

# 3. 헤드리스 스크린샷
"C:/Program Files/Google/Chrome/Application/chrome.exe" \
  --headless=new \
  --screenshot=".claude/skills/mermaid-diagram/screenshot.png" \
  --window-size=1400,900 \
  --no-sandbox \
  --disable-gpu \
  http://localhost:4000

# 4. 스크린샷 확인 (Read 툴로 이미지 읽어서 렌더링 확인)
```

---

## Run (Human path) — 브라우저에서 직접 확인

```powershell
# PowerShell
npx serve docs/architecture --listen 4000
# → http://localhost:4000 브라우저에서 열기
Start-Process "http://localhost:4000"
```

---

## Gotchas

- **Mermaid가 빈 박스로 렌더**: subgraph 내 노드 ID에 `/` 또는 특수문자 포함 금지. 노드 레이블은 `["..."]`로 감싸서 분리할 것.
- **서버 이미 실행 중**: `npx serve` 재실행 시 포트 충돌. `curl localhost:4000` 으로 먼저 확인.
- **스크린샷 타이밍**: Mermaid는 JS로 렌더링되므로 헤드리스 캡처 시 다이어그램이 빈 상태로 찍힐 수 있음. `--virtual-time-budget=3000` 옵션 추가로 해결.
- **`docs/architecture/` 경로 오타**: 사용자가 `architecrue`로 요청해도 실제 저장은 `architecture`(올바른 철자)로 생성.
