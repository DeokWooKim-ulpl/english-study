# English Study Platform — 구축 계획서

> 작성일: 2026-11-04 · 상태: 계획 v0.1
> 근거 수준 표기: **[공식]** 공식 문서 · **[커뮤니티]** 블로그/기사 · **[추정]** Claude 추정·일반 지식(검증 필요)

---

## 0. 결론 (TL;DR)

| 항목 | 결정 |
|---|---|
| 앱 형태 | **PWA(Progressive Web App)** 하나로 휴대폰·태블릿·PC 모두 대응. "홈 화면에 추가"하면 앱처럼 실행됨 |
| 호스팅 | GitHub Pages (repo가 private이면 GitHub Pro 필요 → 대안: Cloudflare Pages 무료) |
| 백그라운드 자동화 | **GitHub Actions 예약 실행(cron)** + **Gemini API** (자료 수집·정리·문제 생성) |
| Gemini 앱 "예약된 작업" | 보조 용도로만 (휴대폰 알림용 "오늘의 표현" 등). GitHub에 직접 쓰기 불가 → 메인 엔진으로 부적합 |
| 복습 알고리즘 | **FSRS** (Anki 최신 기본 알고리즘, `ts-fsrs` 라이브러리) |
| 데이터 저장 | 학습 자료 = repo `data/` 폴더(JSON/MD, Actions가 매일 커밋) · 학습 기록 = 기기 IndexedDB → GitHub API로 `progress/`에 동기화 |
| 리마인드 | 1단계: 매일 이메일/텔레그램 · 2단계: Web Push(PWA 알림) |
| 첫 번째 할 일 | ① Gemini API 키 발급 → ② GitHub Secret 등록 → ③ MVP(독해 + 단어 복습 + 일일 자료 수집) 구축 |

---

## 1. 상용 플랫폼 벤치마크 (무엇을 가져올지)

> 아래 기능 설명은 각 서비스의 공개된 대표 기능 기준 **[추정]**. 세부 UI는 미확인.

| 플랫폼 | 핵심 기능 | 우리 플랫폼에 가져올 것 |
|---|---|---|
| Duolingo | 짧은 레슨, streak(연속 학습일), 일일 목표, XP | 홈 화면 streak·일일 목표 링, 5~10분 단위 세션 |
| Anki | 간격 반복(SRS), FSRS 알고리즘, 4단계 평가(Again/Hard/Good/Easy) | 복습 엔진 전체 (모든 모듈의 오답·신규 표현을 카드화) |
| LingQ / Readlang | 원문 읽으며 단어 탭 → 뜻·저장, 아는/모르는 단어 추적 | 독해 리더: 탭-저장, 문장 단위 번역 토글 |
| Grammarly | 인라인 교정, 수정 이유 설명, 톤 조절 | 작문 피드백: diff 표시 + 이유 + 대안 표현 |
| Writefull | 학술 문장 교정, 논문 코퍼스 기반 표현 제안 | 논문 섹션별 표현 뱅크 (Abstract/Intro/Method/Result) |
| Speak / Praktika | AI 튜터와 역할극(role-play) 회화 | 시나리오 회화: 일상 / 학회 Q&A / 랩미팅 |
| ELSA Speak | 발음 점수, 음소 단위 피드백 | 녹음 → Gemini 오디오 분석 → 발음·유창성 피드백 |
| Ringle | 1:1 수업 후 상세 리포트(교정 문장, 단어 사용 통계) | 회화·발표 후 "세션 리포트" 자동 생성 → 교정 문장을 카드로 |
| Notion/Obsidian | 노트 축적 | 모든 기록을 Markdown으로도 export (Obsidian 호환) |

**핵심 설계 원칙**: 모든 모듈(독해·작문·말하기)의 결과물이 **하나의 복습 큐(SRS)** 로 흘러들어가게 함 → "학습함/안함/복습할 것"이 한 화면에서 관리됨.

---

## 2. 기능 명세

### 2.1 공통 (홈 대시보드)
- **오늘 할 일(Today)**: 복습 카드 N개 · 신규 독해 1편 · 작문 1개 · 말하기 1세션 → 체크리스트
- streak, 주간 학습 시간, 모듈별 진척도 그래프
- **미학습 알림**: 3일 이상 안 한 모듈 강조 표시 + 리마인드 발송
- 전역 검색: 저장한 단어·표현·문장

### 2.2 독해 (Reading) — 영어 원문 읽기
| 기능 | 설명 |
|---|---|
| 일일 자료 | Actions가 매일 수집: arXiv `physics.optics` 초록, 과학 뉴스 RSS, 일반 기사 |
| 난이도 태그 | Gemini가 CEFR(유럽공통참조기준) 레벨·분야 태그 부여 |
| 리더 화면 | 문장 단위 분할, 단어/구 탭 → 뜻·예문·저장, 문장 번역 토글 |
| 이해도 확인 | Gemini가 생성한 요약 질문 3개 + 핵심 표현 5개 |
| 저작권 | 전문 저장은 오픈 라이선스(arXiv CC-BY 등)만. 그 외는 링크 + 요약 + 발췌만 저장 |

### 2.3 작문 (Writing) — 논문·이메일
| 기능 | 설명 |
|---|---|
| 과제 유형 | 이메일(문의/공동연구 제안/리뷰어 회신/일정 조율), 논문(Abstract/Intro/Method/Result/Discussion/Cover letter/Rebuttal) |
| 피드백 | 원문 vs 교정본 diff, 수정 이유, 톤(격식도) 평가, 더 자연스러운 대안 2개 |
| 표현 뱅크 | 섹션·기능별 표현 모음 (예: "We demonstrate…", "In contrast to previous reports…") |
| 오답 노트 | 반복 오류 패턴(관사, 시제, 전치사 등) 통계 → 자동 카드화 |
| 한→영 연습 | 한국어 문장 제시 → 영작 → 피드백 |

### 2.4 말하기 (Speaking) — 일상 회화·학회 발표
| 기능 | 설명 |
|---|---|
| 역할극 회화 | 시나리오: 카페/여행/스몰토크 · 학회 커피브레이크 · 포스터 세션 · 발표 후 Q&A · 랩미팅 |
| 음성 입출력 | 브라우저 녹음(MediaRecorder) → Gemini API 오디오 입력으로 전사·피드백 **[공식: Gemini API는 오디오 입력 지원]** |
| 발표 연습 | 슬라이드별 스크립트 업로드 → 녹음 → 속도(WPM)·필러(um, uh)·발음·논리 흐름 피드백 |
| 섀도잉 | 문장 TTS 재생 → 따라 말하기 → 비교 |
| 세션 리포트 | 교정 문장, 새 표현, 다음 연습 추천 → 복습 카드화 |
| 실시간 대화(2단계) | Gemini Live API 검토 **[추정: 비용·지연 미확인]** |

### 2.5 복습 엔진 (SRS)
- 알고리즘: FSRS — `ts-fsrs` (TypeScript, FSRS v6 구현) **[공식: open-spaced-repetition/ts-fsrs]**
- 카드 종류: 단어/표현, 문장 교정(틀린 문장 → 맞는 문장), 영작(한→영), 듣고 말하기
- 평가 버튼 4개(Again/Hard/Good/Easy) → 다음 복습일 자동 계산

### 2.6 리마인드
| 단계 | 방법 | 비고 |
|---|---|---|
| 1단계 | Actions가 매일 아침 "오늘 복습 N개, 미학습 모듈" 메일/텔레그램 발송 | 가장 간단, 확실 |
| 2단계 | Web Push (PWA 알림). Actions에서 VAPID 키로 발송 | iOS는 홈 화면에 추가한 PWA에서만 가능 (iOS 16.4+) **[공식: WebKit]** |
| 보조 | Gemini 앱 "예약된 작업"으로 매일 "연구 영어 표현 3개" 질문 | 유료 플랜, 최대 10개 활성 **[커뮤니티]** |

---

## 3. 시스템 구조 (Architecture)

```
┌───────────────── GitHub repo: english-study ─────────────────┐
│                                                               │
│  .github/workflows/                                           │
│    daily-collect.yml  (매일 06:00 KST)                         │
│      1) RSS/arXiv에서 자료 수집 (scripts/collect.py)          │
│      2) Gemini API로 요약·난이도·단어·문제 생성               │
│      3) data/ 에 JSON 저장 → git commit & push                │
│    daily-remind.yml   (매일 08:00 KST)                         │
│      progress/ 읽고 → 복습 대상·미학습 계산 → 메일/푸시       │
│    deploy.yml         (push 시) → 앱 빌드 → Pages 배포         │
│                                                               │
│  data/        학습 자료 (자동 생성, 날짜별)                    │
│  progress/    학습 기록 (앱이 GitHub API로 커밋)               │
│  app/         PWA 소스 (React + Vite + TypeScript)             │
│  scripts/     Python 수집·정리 스크립트                        │
└───────────────────────────────────────────────────────────────┘
          │ 배포                         ▲ 기록 동기화
          ▼                              │ (GitHub API, 개인 토큰)
   ┌──────────────── PWA (휴대폰 / 태블릿 / PC) ───────────────┐
   │  IndexedDB(오프라인 저장) · Gemini API 직접 호출(피드백)   │
   └────────────────────────────────────────────────────────────┘
```

### 3.1 기술 스택 선택 이유
| 선택지 | 장점 | 단점 | 판단 |
|---|---|---|---|
| **PWA (React+Vite)** | 한 코드로 모든 기기, 설치 불필요, 무료 호스팅 | iOS 일부 기능 제한(백그라운드 실행 불가) | **채택** |
| Flutter | 네이티브 앱 품질 | 앱스토어 배포 번거로움, 학습 곡선 | 보류 |
| React Native | 네이티브 | PC 대응 별도 | 보류 |
| Streamlit (Python) | Python만으로 빠르게 | 모바일 UX 약함, 서버 필요 | 비채택 |

### 3.2 디렉터리 구조 (예정)
```
english-study/
├── README.md
├── progress_notes.md          # 세션별 진행 기록
├── docs/
│   ├── 01_PLAN.md             # 이 문서
│   └── 02_STUDY_PLAN.md       # 학습 계획
├── app/                       # PWA
├── scripts/                   # collect.py, generate.py, remind.py
├── data/
│   ├── reading/2026-11-05.json
│   ├── writing/prompts/*.json
│   ├── speaking/scenarios/*.json
│   └── phrases/*.json         # 표현 뱅크
├── progress/
│   ├── cards.json             # SRS 카드 상태
│   ├── reviews.jsonl          # 복습 로그 (1줄 = 1회 복습)
│   └── sessions/*.json        # 작문·말하기 세션 리포트
└── .github/workflows/
```

### 3.3 데이터 스키마 (초안)
```json
// progress/cards.json 의 카드 1개
{
  "id": "phr-20261105-003",
  "type": "phrase",              // phrase | correction | translate | speak
  "front": "~을 입증하다 (논문 Abstract)",
  "back": "We demonstrate that ...",
  "source": "reading/2026-11-05.json#2",
  "tags": ["academic", "abstract"],
  "fsrs": { "due": "2026-11-06T00:00:00Z", "stability": 1.2, "difficulty": 5.1, "reps": 1, "state": "Learning" }
}
```

### 3.4 Gemini 연동 방식 정리
| 방식 | 용도 | 비고 |
|---|---|---|
| **Gemini API (AI Studio 키)** | Actions 자동 수집·생성, 앱 내 피드백 | 메인. 무료 등급 한도는 모델·시기별로 변동 → AI Studio에서 본인 프로젝트 한도 직접 확인 (미확인) |
| Gemini CLI GitHub Action | Actions 안에서 Gemini 에이전트 실행 | 대안 **[추정: 사용법 확인 필요]** |
| Gemini 앱 예약된 작업 | 휴대폰 알림형 미니 학습 | GitHub 연동 불가, 보조용 |

> 보안: API 키는 repo에 절대 커밋 안 함. Actions는 `Secrets`에, 앱은 각 기기 브라우저 저장소에 직접 입력.

### 3.5 주의사항
- **GitHub Pages + private repo**: 무료 계정은 public repo만 Pages 가능, private은 GitHub Pro 필요 **[공식: GitHub Docs]**. 또한 Pages 사이트 자체는 공개 URL이 됨 → 학습 기록(progress/)은 페이지에 포함하지 않고 API로만 접근.
- **Actions 예약 실행**: public repo에서 60일간 활동 없으면 schedule이 자동 비활성화 **[공식: GitHub Docs]**. 매일 자동 커밋이 있으면 대체로 문제없음 **[추정]**.
- **Actions 사용량**: public repo 무료, private repo는 월 무료 분량 한도 있음 (현재 한도 미확인).
- **음성 인식**: 브라우저 Web Speech API는 Chrome 계열 지원 양호, Safari는 제한적 **[커뮤니티]** → 녹음 후 Gemini로 전사하는 방식이 기기 간 일관성 높음.

---

## 4. 개발 로드맵

| 단계 | 기간(목표) | 내용 | 완료 기준 |
|---|---|---|---|
| **0. 준비** | 1일 | 계획 문서, repo 구조, Gemini API 키, Secrets 등록 | 이 문서 확정 |
| **1. MVP** | 1~2주 | PWA 뼈대(홈/독해/복습), `daily-collect` Action, FSRS 카드 | 폰에서 매일 새 글 읽고 단어 저장·복습 가능 |
| **2. 작문** | 1주 | 이메일·논문 과제, Gemini 피드백, 오답→카드 | 이메일 1통 쓰고 교정 리포트 받음 |
| **3. 말하기** | 1~2주 | 녹음, 역할극, 발표 연습, 세션 리포트 | 5분 학회 Q&A 연습 후 리포트 |
| **4. 동기화·알림** | 1주 | GitHub API 기록 동기화, 이메일/푸시 리마인드 | PC↔폰 기록 일치, 매일 알림 수신 |
| **5. 고도화** | 상시 | 통계 대시보드, 약점 분석, Obsidian export, 실시간 회화 | — |

---

## 5. 지금 사용자가 직접 할 일 (Phase 0 체크리스트)

1. [ ] Google AI Studio에서 Gemini API 키 발급 (https://aistudio.google.com)
2. [ ] GitHub repo → Settings → Secrets and variables → Actions → `GEMINI_API_KEY` 등록
3. [ ] repo 공개 여부 결정 (public: Pages 무료 / private: Pro 또는 Cloudflare Pages)
4. [ ] 리마인드 수단 선택 (이메일 / 텔레그램 / 둘 다)
5. [ ] 관심 분야 키워드 확정 (예: ultrafast laser, nonlinear optics, photonics …) → 독해 자료 수집 필터로 사용
