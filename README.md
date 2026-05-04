# Design System Extractor

HTML을 붙여넣으면 색상·타이포그래피·간격·컴포넌트 등 디자인 시스템 토큰을 자동으로 추출하는 단일 파일 도구.  
빌드 단계 없이 `index.html` 하나만으로 동작합니다.

---

## 주요 기능

- **색상 팔레트** — 색조(Hue)별 그룹화, 라이트/다크 모드 페어 감지
- **타이포그래피** — font-family · size · weight · line-height, Google Fonts 감지
- **간격 & 반경** — spacing · border-radius 토큰 목록
- **CSS 변수** — `--custom-property` 전체 목록 + 다크모드 변수 비교
- **컴포넌트** — 클래스 빈도 분석 및 prefix 기반 그룹화
- **WCAG 대비** — 텍스트/배경 색상 쌍의 명암비 자동 계산 (AA / AAA / AA-LG)
- **외부 CSS 자동 로드** — `<link href="https://...">` 절대 URL 자동 fetch (CORS 프록시 폴백)
- **내보내기** — Figma Variables JSON · Tailwind config · CSS Variables 복사
- **북마클릿** — 브라우저에서 직접 실행 → 인라인 CSS 포함 HTML 클립보드 복사

---

## 사용 방법

1. 브라우저에서 `index.html` 열기
2. 분석할 사이트에서 `Ctrl+A` → `Ctrl+C`로 전체 HTML 복사
3. 왼쪽 텍스트 영역에 붙여넣기
4. (선택) 페이지 URL 입력란에 원본 URL 입력 — 상대 경로 CSS 해석에 사용
5. **분석** 버튼 클릭

> **외부 CSS 포함 사이트**: 북마클릿을 북마크 바에 드래그한 뒤 대상 사이트에서 클릭하면  
> 동일 출처 CSS가 모두 인라인화된 HTML을 클립보드에 복사해 줍니다.

---

## 버전 히스토리

### v0.4 — 2026-05-04

**CORS 차단 우회 및 절대 URL 자동 처리**

- `<link href="https://...">` 절대 URL은 pageUrl 입력 없이도 자동 fetch 시도
- 직접 fetch가 CORS로 차단될 경우 `api.allorigins.win` 프록시로 자동 재시도 (8초 타임아웃)
- 분석 중 상태 메시지를 "외부 CSS 로드 중"으로 구체화

> **해결 이슈**: [ISSUE-003](docs/issues.md#issue-003--cors-차단으로-외부-css-fetch-실패)

---

### v0.3 — 2026-05-04

**9가지 기능 대규모 추가**

| 기능 | 설명 |
|------|------|
| 북마클릿 | 북마크 바에 드래그 → 사이트에서 클릭 시 동일 출처 CSS 인라인화 후 HTML 복사 |
| WCAG 대비 체커 | "대비" 탭 추가, 텍스트/배경 페어 자동 감지, AA · AAA · AA-LG 배지 표시 |
| 색상 Hue 그루핑 | Red · Orange · Yellow · Green · Cyan · Blue · Purple · Pink · Neutral 9그룹으로 분류, 밝기순 정렬 |
| `@import` 재귀 fetch | CSS 내 `@import` 규칙을 재귀적으로 추적 (depth ≤ 3, visited set으로 순환 방지) |
| Figma Variables 내보내기 | Figma Variables JSON 형식(collections / modes / variables), 라이트+다크 모드 포함 |
| Tailwind config 내보내기 | `theme.extend.colors / spacing / borderRadius / fontFamily` 형식 |
| 라이트/다크 모드 페어 | `@media (prefers-color-scheme: dark)` 블록을 brace-depth 추적으로 추출, 색상 탭·변수 탭에 표시 |
| 컴포넌트 prefix 그룹화 | 클래스명을 `-` / `_` 기준으로 분리, 공통 prefix 아래 자식 클래스 들여쓰기 |
| Google Fonts 감지 | `<link href="fonts.googleapis.com/...">` 파싱, 폰트 탭에 감지 배지 표시 |

---

### v0.2 — 2026-05-04

**외부 CSS 자동 fetch 및 Next.js 지원**

- pageUrl 입력 시 `<link rel="stylesheet">` 파일을 CORS fetch하여 색상·변수 파싱
- `<script>` 태그 내 Next.js RSC/SSR JSON 데이터에서 hex · rgba · CSS 변수 추가 스캔
- statsBar에 외부 CSS 로드 개수 항목 추가
- 분석 완료 토스트에 외부 CSS 로드 수 표시

> **해결 이슈**: [ISSUE-002](docs/issues.md#issue-002--nextjs--tailwind-사이트에서-색상-미추출)

---

### v0.1 — 2026-05-04

**색상 추출 버그 수정**

- canvas context를 색상마다 새로 생성하던 문제 수정 → 전역 단일 canvas 재사용
- `color: white` / `fill: red` 등 named color 추출 정규식 추가
- `oklch()` · `lch()` · `lab()` · `hwb()` · `color()` 최신 CSS 색상 포맷 지원
- `rgba(r,g,b,a)` alpha 값을 dedup 키에 포함하여 반투명 색상 중복 방지
- 외부 `.css` 파일 미포함 안내 UI 추가

> **해결 이슈**: [ISSUE-001](docs/issues.md#issue-001--색상이-전혀-추출되지-않는-문제)

---

### v0.0 — 2026-05-04

**최초 릴리스**

- HTML 붙여넣기 → 색상 · 타이포그래피 · 간격 · CSS 변수 · 컴포넌트 클래스 추출
- 다크 테마 UI, 탭 기반 결과 뷰
- CSS Variables / CSS Snippet 텍스트 내보내기

---

## 기술 스택

- 순수 HTML / CSS / JavaScript (빌드 단계 없음, 단일 파일)
- DOMParser — 붙여넣은 HTML 파싱
- Canvas API — 색상 정규화 및 중복 제거
- Fetch API — 외부 CSS 로드 (CORS 프록시 폴백: allorigins.win)
- WCAG 2.1 상대 휘도(Relative Luminance) 공식 — 명암비 계산

---

## 문서

- [이슈 & 해결 기록](docs/issues.md)
