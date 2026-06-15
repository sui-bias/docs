# Bias 와이어프레임 작성 가이드

> 출처: `mybias/design/wireframe-guide.md` 기반, Bias 모바일 웹뷰에 맞게 재정의.

---

## 1. 폴더·파일 구조

```
docs/docs/wireframes/
├── wireframes.md              결정 사항 기록 (대화 중 갱신)
├── design/                    디자인 시스템 SSOT
│   ├── tokens.css
│   ├── components.css         (@import 매니페스트)
│   ├── components/            (01~09 분할)
│   └── wireframe-guide.md     (본 파일)
└── screens/                   화면별 HTML (추후 추가)
    ├── shared/                공통 CSS·JS
    ├── onboarding/
    ├── list/
    ├── chat/
    ├── open-chat/
    ├── character/
    └── mypage/
```

---

## 2. 공통 HTML 골격

### 2.1 link/script

```html
<link rel="stylesheet" href="../design/tokens.css">
<link rel="stylesheet" href="../design/components.css">
<link rel="stylesheet" href="../shared/frame.css">
```

### 2.2 아이폰 프레임

```html
<div class="iphone-frame light">
  <div class="iphone-screen">
    <div class="status-bar"><!-- 9:41 --></div>
    <div class="topbar"><!-- 헤더 --></div>
    <div class="content"><!-- 콘텐츠 --></div>
    <nav class="bias-bottom-nav" role="navigation">
      <!-- 5탭 -->
    </nav>
    <div class="home-indicator"><div class="home-indicator-bar"></div></div>
  </div>
</div>
```

### 2.3 하단 네비게이션 표준 마크업

```html
<nav class="bias-bottom-nav" role="navigation" aria-label="하단 탭바">
  <a class="bias-nav-item" aria-label="리스트" aria-current="page" href="#">
    <svg viewBox="0 0 24 24"><!-- users icon --></svg>
    <span class="bias-nav-label">리스트</span>
  </a>
  <a class="bias-nav-item" aria-label="채팅" href="#">
    <svg viewBox="0 0 24 24"><!-- message-circle icon --></svg>
    <span class="bias-nav-label">채팅</span>
  </a>
  <a class="bias-nav-item" aria-label="오픈채팅" href="#">
    <svg viewBox="0 0 24 24"><!-- message-square icon --></svg>
    <span class="bias-nav-label">오픈채팅</span>
  </a>
  <a class="bias-nav-item" aria-label="캐릭터 생성" href="#">
    <svg viewBox="0 0 24 24"><!-- plus-circle icon --></svg>
    <span class="bias-nav-label">캐릭터</span>
  </a>
  <a class="bias-nav-item" aria-label="마이페이지" href="#">
    <svg viewBox="0 0 24 24"><!-- user icon --></svg>
    <span class="bias-nav-label">MY</span>
  </a>
</nav>
```

---

## 3. 작성 규칙

| 항목 | 규칙 |
|------|------|
| 플랫폼 | 모바일 웹뷰 전용 (데스크탑 없음) |
| 아이콘 | Lucide SVG (`stroke="currentColor"` · `fill="none"` · `stroke-width="1.75"`) — 이모지 금지 |
| 색·크기 | 토큰 참조만 (`--colors-*` · `--spacing-*` · `--radius-*`) — 인라인 hex/px 금지 |
| 모드 | 라이트 기본 + 우상단 토글로 다크 |
| 스크롤 | 화면 내 콘텐츠 스크롤 허용 (1페이지 스케일 강제 없음 — 채팅 화면 특성상 자연 스크롤) |
| 오버레이 | 바텀시트·다이얼로그는 기본 프레임에 숨김 상태로 포함 |
| 아이폰 프레임 | `iphone-frame light` / `iphone-frame dark` 클래스로 모드 전환 |

---

## 4. 화면별 진입점 정의

| 화면 그룹 | 진입 탭 | 주요 화면 |
|-----------|---------|-----------|
| 온보딩 | (탭 없음 — 게이트) | 제품 소개 → 지갑 연결 → 프로필 초기화 → 캐릭터 게이트 |
| 리스트 | 탭 1 | 팔로잉 유저 + 캐릭터 단일 피드 |
| 채팅 | 탭 2 | 채팅 목록 (1:1 + 그룹 통합) → 1:1 채팅방 / 그룹 채팅방 |
| 오픈채팅 | 탭 3 | 캐릭터 그리드 → 캐릭터 오픈채팅방 |
| 캐릭터 생성 | 탭 4 | 캐릭터 생성 폼 (plus) / plus 업그레이드 안내 (free) |
| 마이페이지 | 탭 5 | 내 프로필 / 내 캐릭터 목록 / 요금제·결제 |

---

## 5. 컴포넌트 빠른 참조

| 컴포넌트 | 파일 | 클래스 |
|----------|------|--------|
| 하단 5탭 | `05-navigation.css` | `.bias-bottom-nav` · `.bias-nav-item` |
| 채팅 목록 행 | `07-chat.css` | `.chat-row` |
| 말풍선 (수신) | `07-chat.css` | `.msg-bubble.msg-bubble-received` |
| 말풍선 (발신) | `07-chat.css` | `.msg-bubble.msg-bubble-sent` |
| 채팅 입력창 | `07-chat.css` | `.chat-input-bar` |
| 타이핑 인디케이터 | `07-chat.css` | `.typing-indicator` |
| 캐릭터 카드 | `03-display.css` | `.char-card` |
| 친밀도 배지 | `03-display.css` | `.affinity-badge-lv{1~4}` |
| 플랜 배지 | `03-display.css` | `.plan-badge-free` · `.plan-badge-plus` |
| 바텀시트 | `08-overlay.css` | `.bs` · `.bs-overlay` |
| 다이얼로그 | `08-overlay.css` | `.dialog` |
| 토스트 | `08-overlay.css` | `.toast` |
| 연결 상태 | `09-feedback.css` | `.connection-banner` |
| Topbar | `04-header.css` | `.topbar` |
| 버튼 | `01-action.css` | `.btn .btn-fill-primary .btn-lg` |

---

## 6. 친밀도 단계 (Affinity)

| 단계 | 컬러 | 의미 |
|------|------|------|
| lv1 | Indigo `#818CF8` | 첫 만남 |
| lv2 | Pink `#F75794` | 친해진 사이 |
| lv3 | Red `#EF4444` | 특별한 관계 |
| lv4 | Amber `#F59E0B` | 운명적 관계 |

토큰: `--colors-affinity-lv{1~4}` · `--colors-affinity-lv{1~4}-bg`

---

## 7. 금지 사항

- 인라인 hex/px (토큰 경유 필수)
- 이모지 아이콘 사용
- 데스크탑 레이아웃 추가
- MyCream 전용 컴포넌트 재사용 (NSFW 게이트, 스토리 그리드, 크리에이터 탭바 등)
