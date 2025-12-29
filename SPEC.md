# Tetris HTML Spec

## 목적
- 단일 HTML 파일로 동작하는 브라우저 테트리스 구현.
- 현재 `tetris.html`과 동일한 UI/UX, 게임 규칙, 사운드, 애니메이션을 재현.

## 산출물/제약
- 산출물: `tetris.html` 1개 파일 (HTML/CSS/JS 모두 포함).
- 외부 라이브러리/빌드 도구/네트워크 의존 없음.
- 한국어 UI 텍스트 유지.

## DOM 구조 (필수 id/class)
```
<body>
  <div class="page">
    <header class="hero">
      <div>
        <h1>Tetris</h1>
        <p>클래식 테트리스를 브라우저에서 즐겨보세요.</p>
      </div>
      <div class="status" id="status">대기 중</div>
    </header>

    <main class="grid">
      <section class="game">
        <canvas id="board"></canvas>
        <div class="overlay" id="overlay">
          <div>
            <h2 id="overlay-title">일시정지</h2>
            <p id="overlay-sub">시작 버튼을 눌러 계속하세요.</p>
          </div>
        </div>
      </section>

      <aside class="panel">
        <div class="card">
          <h2>Score</h2>
          <div class="big" id="score">0</div>
        </div>
        <div class="card">
          <h2>Lines</h2>
          <div class="big" id="lines">0</div>
        </div>
        <div class="card">
          <h2>Level</h2>
          <div class="big" id="level">1</div>
        </div>
        <div class="card">
          <h2>Next</h2>
          <canvas id="next"></canvas>
        </div>
        <div class="card controls">
          <button class="primary" id="start">Start</button>
          <button id="pause">Pause</button>
          <button id="restart">Restart</button>
          <button id="sound" aria-pressed="true">Sound: On</button>
        </div>
        <div class="card key">
          <h2>Controls</h2>
          <p>← → 이동 · ↑ 회전 · ↓ 소프트 드롭 · Space 하드 드롭</p>
          <p>P 일시정지 · R 다시 시작</p>
        </div>
      </aside>
    </main>

    <section class="touch">
      <button data-action="left">◀</button>
      <button data-action="right">▶</button>
      <button data-action="down">▼</button>
      <button data-action="rotate">⟳</button>
      <button data-action="drop">DROP</button>
    </section>
  </div>
</body>
```

## 스타일/비주얼 가이드
### 공통
- 폰트: `"Lucida Console", "Courier New", "Apple SD Gothic Neo", "Malgun Gothic", monospace`.
- 전체 분위기: CRT/레트로 테마, 다층 그라디언트, 스캔라인/글로우.
- `canvas`는 `image-rendering: pixelated`.

### CSS 변수 (필수 값)
```
:root {
  --bg-1: #090d10;
  --bg-2: #111b22;
  --bg-3: #1b2c30;
  --panel: rgba(7, 11, 13, 0.92);
  --panel-2: rgba(12, 16, 18, 0.98);
  --panel-border: rgba(94, 217, 255, 0.28);
  --panel-border-strong: rgba(125, 255, 122, 0.45);
  --accent: #7dff7a;
  --accent-2: #ffd166;
  --accent-3: #5ed9ff;
  --text: #f4f1e9;
  --muted: rgba(244, 241, 233, 0.72);
  --grid: rgba(94, 217, 255, 0.12);
  --shadow: rgba(0, 0, 0, 0.55);
  --shadow-soft: rgba(0, 0, 0, 0.3);
  --glow: rgba(94, 217, 255, 0.35);
  --edge: rgba(125, 255, 122, 0.28);
}
```

### 핵심 레이아웃
- `.page`: max-width 1200px, 중앙 정렬, 상/하 여백.
- `.hero`: 제목/설명 + 상태 배지.
- `.grid`: 2열(게임/패널) → 900px 이하에서 1열.
- `.game`: 어두운 패널 + 네온 보더/그림자, 내부에 `#board` 캔버스.
- `.panel .card`: 카드형 정보 패널(Score/Lines/Level/Next/Controls).
- `.touch`: 모바일용 5버튼 그리드, 640px 이하에서 3열 + DROP 버튼은 전체 폭.

### 배경/효과 (요지)
- `body` 배경: 다중 radial gradient + linear gradient.
- `body::before`: 스캔라인/노이즈 패턴 + `scanlineMove`(10s), `crtFlicker`(0.18s steps).
- `body::after`: 비네팅(중앙 투명/가장자리 어둡게).
- `#board` 배경: 여러 radial/linear gradient + `boardBgShift`(16s), `boardBgPulse`(6s).
- `.card`: `floatIn` 애니메이션(순차 딜레이), hover 시 보더/그림자 강화.
- `.overlay`: 게임 일시정지/오버 표시, `show` 클래스에서 표시.

### 반응형
- `max-width: 900px`: `.grid` 1열, `.hero` 세로 배치.
- `max-width: 640px`: `.page` 패딩 축소, `.touch` 3열, DROP 버튼 전폭.
- `prefers-reduced-motion`: `#board` 배경 애니메이션 비활성화.

## 게임 규칙/로직
### 기본 상수
- `COLS = 10`, `ROWS = 20`, `BLOCK = 30` (보드 크기 300x600).
- `PREVIEW = 140` (Next 캔버스 정사각형).
- `CLEAR_DURATION = 260ms` (라인 클리어 애니메이션).

### 테트로미노 색상
```
I: #22d3ee
J: #3b82f6
L: #f59e0b
O: #fde047
S: #22c55e
T: #ef4444
Z: #f97316
```

### 모양 정의 (행렬)
- I: 4x4, 가운데 가로줄.
- J/L/S/T/Z: 3x3.
- O: 2x2.
- 각 행렬의 1은 블록, 0은 빈칸.

### 7-백 랜덤
- `bag`에 7개 타입을 섞어(피셔-예이츠) 순서대로 소진.
- `drawFromBag()`로 다음 조각 생성.

### 스폰/충돌
- 새 조각 y = -1, x = 중앙 정렬.
- 스폰 시 충돌이면 즉시 게임 오버.
- 충돌 체크: 보드 외부 또는 이미 채워진 셀.

### 회전
- 시계 방향/반시계 방향 회전 함수.
- x 킥 배열 `[0, -1, 1, -2, 2]` 순서로 시도.
- 성공 시 사운드 재생.

### 드롭
- 자동 드롭: `dropInterval`마다 1칸 하강.
- 소프트 드롭(↓): 1칸 하강 후 충돌 시 고정.
- 하드 드롭(Space): 충돌 직전까지 즉시 하강 후 고정.

### 고정/라인 클리어
- 고정 시 보드에 합치고 라인 검사.
- 가득 찬 라인 있으면 `clearing` 상태로 전환, 260ms 애니메이션 후 제거.
- 클리어 중에는 입력/드롭 정지.

### 점수/레벨
- 라인 점수: `[0, 100, 300, 500, 800] * level` (1~4줄).
- `lines` 누적 10줄마다 `level` +1.
- `dropInterval = max(120, 800 - (level-1) * 60)`.
- 점수 UI는 420ms ease-out 애니메이션으로 증가.

## 렌더링
- 보드 캔버스: 그리드 라인 → 고정 블록 → 클리어 효과 → 고스트 → 현재 조각.
- 셀 렌더링: 기본 색 + 상단 하이라이트/하단 그림자(입체감).
- 고스트: 현재 조각 색상에 알파 0.2.
- Next 미리보기: 24px 셀, 실루엣 박스 기준 중앙 정렬.

## 입력/조작
- 키보드:
  - 좌/우 이동: ←/→
  - 회전: ↑
  - 소프트 드롭: ↓
  - 하드 드롭: Space
  - 일시정지: P
  - 재시작: R
  - 실행 시작: Enter (게임 미실행/오버 상태에서만)
- 터치 버튼: data-action 기준 좌/우/하강/회전/드롭.
- 일시정지/클리어 중에는 조작 무시.
- 화살표/Space는 기본 브라우저 스크롤 방지.

## 상태/오버레이
- 상태 텍스트(`status`): `대기 중` → `진행 중` → `일시정지` → `게임 오버`.
- 오버레이 표시:
  - 시작 전: `시작 준비` / `Start 버튼을 눌러 게임을 시작하세요.`
  - 일시정지: `일시정지` / `Start 버튼으로 계속할 수 있어요.`
  - 게임 오버: `게임 오버` / `Restart 버튼으로 다시 시작하세요.`
- 창 blur 시 자동 일시정지 + 오버레이.

## 오디오
### 효과음 (Web Audio API)
- `move`, `rotate`, `drop`, `line`, `gameover`, `start`, `pause`, `resume`.
- 각 사운드는 `freq`, `type`, `duration`, `volume` 조합으로 생성.
- 사운드 활성화 전 `unlockAudio()`로 컨텍스트 활성화.

### BGM
- 템포: 118 BPM, 16 step 루프.
- `lead`, `bass` 시퀀스 기반으로 스케줄링.
- lookahead 0.12s, schedule interval 70ms.
- 게임 진행 중(실행 중 + 비일시정지 + 비게임오버)일 때만 재생.

## 초기화/흐름
- 로드 시 `resetGame()` 호출 후 상태 `대기 중`.
- 오버레이는 시작 안내로 표시.
- `requestAnimationFrame` 루프는 최초 `startGame()` 시 시작.

## 캔버스 설정
- `devicePixelRatio` 적용하여 실제 캔버스 크기/스타일 크기 분리.
- `ctx.setTransform(ratio, 0, 0, ratio, 0, 0)`로 스케일 고정.

## 요구 사항 요약
- UI/텍스트/키 바인딩/사운드/룰/애니메이션을 위와 동일하게 구현.
- 동작이 동일해야 하며, 외부 의존 없이 단일 HTML에서 실행 가능해야 함.
