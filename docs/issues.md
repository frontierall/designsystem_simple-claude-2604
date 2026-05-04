# Issues & Resolutions

이 문서는 Design System Extractor 개발 과정에서 발생한 이슈와 해결 방법을 기록합니다.

---

## ISSUE-001 · 색상이 전혀 추출되지 않는 문제

**발생일**: 2026-05-04  
**커밋**: `5992a9e`  
**심각도**: Critical

### 증상
HTML을 붙여넣고 분석해도 색상 탭에 아무것도 표시되지 않음.

### 원인 분석

**원인 1 — canvas context 한계 초과**  
색상 정규화 시 색상마다 `document.createElement('canvas').getContext('2d')`를 새로 호출.  
브라우저는 동시에 유지할 수 있는 canvas context 수에 상한이 있어, 초과 시 `getContext('2d')`가 `null`을 반환.  
try-catch 블록 안에서 null 참조가 발생하면 해당 색상이 조용히 무시되는 구조였음.

**원인 2 — Named color 미지원**  
정규식이 `#hex`, `rgb()`, `rgba()`, `hsl()`, `hsla()`만 매칭하고 있었음.  
`color: white`, `fill: red`, `background: black` 등 CSS 속성값에 쓰인 named color는 전혀 감지하지 못함.

**원인 3 — 최신 CSS 포맷 미지원**  
`oklch()`, `lch()`, `lab()`, `hwb()`, `color()` 함수 표기는 정규식에 없었음.

**원인 4 — alpha 중복 제거 오류**  
`rgba(255,0,0,0.5)`와 `rgb(127,0,0)`이 canvas 정규화 후 같은 RGB로 뭉개져 중복으로 처리됨.  
alpha 값이 dedup 키에 포함되지 않았기 때문.

### 해결 방법

| 원인 | 해결 |
|------|------|
| canvas context 재생성 | 전역 단일 canvas(`_cc`) / context(`_cx`)를 재사용하도록 변경 |
| Named color 미지원 | color 관련 CSS 속성(`color:`, `fill:`, `background:` 등)에 대해 named color 전용 정규식 추가 |
| 최신 포맷 미지원 | 메인 정규식에 `oklch`, `lch`, `lab`, `hwb`, `color()` 추가 |
| alpha dedup 오류 | dedup 키를 `r,g,b` → `r,g,b,a` 로 변경 |

---

## ISSUE-002 · Next.js / Tailwind 사이트에서 색상 미추출

**발생일**: 2026-05-04  
**커밋**: `9f94191`  
**심각도**: High  
**재현 사이트**: `gabrielvaldivia.com`

### 증상
Next.js + Tailwind로 만들어진 사이트의 HTML을 붙여넣어도 색상이 거의 추출되지 않음.

### 원인 분석

**원인 1 — 모든 CSS가 외부 파일에 존재**  
Next.js 빌드 결과물은 CSS를 `/_next/static/css/*.css`에 번들링함.  
붙여넣은 HTML에는 `<link rel="stylesheet" href="/_next/static/css/...">` 태그만 있고 실제 색상 정의는 없음.

**원인 2 — RSC/SSR JSON 내부의 인라인 색상**  
Next.js Server Components는 `<script>` 태그 안에 JSON 직렬화된 데이터를 내장함.  
이 JSON 문자열 안에도 hex / rgba 색상값이 포함될 수 있으나, HTML 파싱 단계에서 무시됨.

### 해결 방법

| 원인 | 해결 |
|------|------|
| 외부 CSS 파일 | pageUrl 입력 시 `<link rel="stylesheet">` href를 fetch하여 CSS 전체 파싱 |
| `<script>` 내 JSON 색상 | `<script>` 태그의 textContent에서 hex·rgba·CSS 변수를 추가 스캔 |
| fetch 성공 여부 표시 | statsBar와 완료 토스트에 외부 CSS 로드 개수 표시 |

---

## ISSUE-003 · CORS 차단으로 외부 CSS fetch 실패

**발생일**: 2026-05-04  
**커밋**: `ba820fb`  
**심각도**: High  
**재현 사이트**: `jessicahische.is`

### 증상
pageUrl을 입력해도 일반 웹사이트(jessicahische.is 등)의 CSS를 가져오지 못해 색상이 추출되지 않음.  
또한 `<link href="https://...">` 절대 URL이 있어도 pageUrl을 입력하지 않으면 아예 시도조차 안 됨.

### 원인 분석

**원인 1 — CORS 헤더 없는 CSS 서버**  
브라우저는 다른 출처의 리소스 fetch 시 서버가 `Access-Control-Allow-Origin` 헤더를 반환해야 허용함.  
대부분의 일반 사이트는 이 헤더를 CSS 파일에 설정하지 않아 `mode: 'cors'` fetch가 차단됨.  
에러가 catch 블록에서 조용히 삼켜져 사용자에게 피드백도 없었음.

**원인 2 — 절대 URL임에도 pageUrl 필수**  
`<link href="https://jessicahische.is/assets/styles/style.css">` 처럼 이미 절대 URL인 경우에도  
`resolvedBase`(pageUrl)가 없으면 href 해석을 건너뛰는 로직이었음.

### 해결 방법

| 원인 | 해결 |
|------|------|
| CORS 차단 | 직접 fetch 실패 시 `api.allorigins.win` CORS 프록시로 재시도 (8초 타임아웃) |
| 절대 URL 미처리 | `href`가 `https://`로 시작하면 pageUrl 없이도 fetch 시도하도록 조건 수정 |

### 참고
allorigins.win 프록시는 제3자 무료 서비스이므로 응답이 수 초 지연될 수 있음.  
이를 감안해 분석 중 메시지를 "외부 CSS 로드 중"으로 변경하여 대기 상태를 명확히 표시.

---

## 이슈 상태 요약

| ID | 제목 | 상태 | 커밋 |
|----|------|------|------|
| ISSUE-001 | 색상 전혀 미추출 (canvas / named color / alpha) | ✅ 해결 | `5992a9e` |
| ISSUE-002 | Next.js/Tailwind 사이트 색상 미추출 | ✅ 해결 | `9f94191` |
| ISSUE-003 | CORS 차단으로 외부 CSS fetch 실패 | ✅ 해결 | `ba820fb` |
