# 구축 계획 v2 — Gemini 앱 + Google Sheets + Apps Script

> 2026-11-05 · v2.0 · 이 문서가 최신 계획. [01_PLAN.md](01_PLAN.md)·[03_GEMINI_TASK_PIPELINE.md](03_GEMINI_TASK_PIPELINE.md)의 구조 부분을 대체함 (벤치마크·작업 프롬프트 초안은 참고용으로 유지)
> 근거 수준: **[공식]** Google 공식 도움말 · **[커뮤니티]** 기사/블로그 · **[추정]** Claude 추정 · **[테스트 필요]**

---

## 0. 결론

| 질문 | 답 |
|---|---|
| 버튼 누르면 Gemini 앱이 자동으로 피드백? | **가능성 높음 (테스트 필요).** Gemini 작업은 "받은편지함에 이메일이 도착하면" 실행하는 트리거를 지원함 **[공식]**. 버튼 → Apps Script가 나에게 `[ES-FB]` 메일 발송 → Gemini 작업이 메일을 받아 피드백 작성 → Apps Script가 결과 수거 → 앱에 표시 |
| 얼마나 걸리나? | 실행 시각은 "대략적"이고 지연될 수 있다고 명시됨 **[공식]**. 수 분~수십 분 예상 **[추정]** → 실측 필요 |
| 중간 저장소 | **Google Sheets 하나(`ES_DB`)를 데이터베이스로 사용.** 내 입력·Gemini 출력·복습 기록 전부 행 단위로 영구 저장 |
| 앱 | **Apps Script 웹 앱**으로 1차 구축 (Sheets에 바로 연결, 토큰 불필요, 휴대폰·태블릿·PC 브라우저에서 열림, 홈 화면에 바로가기 추가) |
| GitHub 역할 | 코드 저장소 + 매일 밤 데이터 백업(JSON) |
| 말하기 | 하루 **1개** 스크립트(약 1분, 120~150단어) 제시 → 소리 내 읽기 → "읽었음" 체크. 미완료는 다음날 1개까지만 이월 |
| 복습 | 모든 입력·출력이 자동으로 복습 카드가 됨 + 정기 복습(일일 카드 / 주간 재작성 / 월간 리포트) |

---

## 1. 전체 구조

```
┌──────────────────────── 내 Google 계정 ─────────────────────────┐
│                                                                 │
│  [앱] Apps Script 웹 앱  ◀────────▶  [DB] Google Sheets "ES_DB"  │
│   (폰/태블릿/PC 브라우저)               탭: inbox, submissions,   │
│        │ ① "Gemini로 피드백" 버튼          feedback, cards, ...  │
│        ▼                                    ▲                    │
│  Apps Script: 나에게 메일 발송               │ ④ 결과 파싱·저장   │
│   제목 "[ES-FB] W-20261105-01"              │                    │
│        │                                    │                    │
│        ▼ ② 이메일 도착 트리거                 │                    │
│  Gemini 작업 "작문 피드백"  ──③ 결과 회신──▶  Gmail "[ES-RESULT]"  │
│   (메일 본문의 내 글을 교정)                  (Apps Script가 1~5분마다 확인)
│                                                                 │
│  Gemini 작업 (예약): 독해 자료·주간 과제팩·리마인드·주간 리뷰       │
│        └──▶ Google 문서 "ES_inbox_…" ──▶ Apps Script 수거 ──▶ ES_DB │
└─────────────────────────────────────────────────────────────────┘
                     │ 매일 밤 백업 (JSON)
                     ▼
               GitHub english-study  (코드 + backup/)
```

### 1.1 왜 이메일을 매개로 쓰나
| 방법 | 자동화 | 확실성 | 판단 |
|---|---|---|---|
| **이메일 트리거 → Gemini 작업** | 버튼 1번 | 트리거 지원은 공식, Gemini가 결과를 메일로 회신하는지는 테스트 필요 | **주 경로** |
| Sheets `=AI()` 함수 | 셀에 수식 | AI Pro/Ultra 필요 **[커뮤니티]**. Apps Script로 자동 생성되는지 불명확 (브라우저에서 "생성" 클릭이 필요할 수 있음) **[추정]** | 테스트 후 보조 경로 후보 |
| 복사 + Gemini 앱 열기 + 결과 붙여넣기 | 수동 2단계 | 항상 동작 | **예비 경로** (위 방법 실패·지연 시) |
| Gemini API / Apps Script의 Vertex AI | 즉시 | 확실 | 사용자 요청으로 제외 |

> 모든 경로의 결과는 같은 형식(JSON)으로 `feedback` 탭에 저장되므로, 어느 경로로 받아도 복습 시스템은 동일하게 동작함.

### 1.2 왜 앱을 Apps Script 웹 앱으로 바꾸나 (v1: GitHub Pages PWA)
| 항목 | Apps Script 웹 앱 (v2 채택) | GitHub Pages PWA (v1) |
|---|---|---|
| Sheets/Gmail 접근 | 바로 가능 | 별도 API 서버·토큰 필요 |
| 보안 | "나만 접근"으로 배포 → 내 Google 로그인으로 보호 | 공개 URL, 토큰을 기기마다 저장 |
| 기기 | 모든 브라우저 | 모든 브라우저 |
| 단점 | 오프라인 불가, 첫 로딩 1~3초 **[추정]**, 상단에 Google 안내 문구 | 구축 복잡 |

→ 1차는 Apps Script 웹 앱. 나중에 오프라인이 꼭 필요해지면 PWA 화면을 별도로 얹을 수 있음.

---

## 2. 데이터 저장 — `ES_DB` 스프레드시트

**원칙: 지우지 않음.** 모든 행에 고유 ID와 시각을 남기고, 수정은 새 버전 행으로 추가.

| 탭 | 한 행 = | 주요 열 |
|---|---|---|
| `materials` | 독해 자료 1편 | id, date, category(research/news/daily), title, url, cefr, excerpt, summary_ko, read_at, read_seconds |
| `phrases` | 표현 1개 | id, en, ko, example, source_id, section, created_at |
| `submissions` | 내 작문 1건 | id, created_at, mode(email/paper/translate…), prompt_id, my_text, status(pending/sent/done/error), route(email/manual) |
| `feedback` | Gemini 피드백 1건 | id, submission_id, received_at, corrected, reasons_ko, error_types, alternatives, formality, raw_json |
| `speaking` | 말하기 스크립트 1개 | id, date, type(daily/academic/talk), title, script, est_seconds, done(TRUE/FALSE), done_at, reads, self_rating |
| `cards` | 복습 카드 1장 | id, kind, front, back, source_id, due, stability, difficulty, reps, lapses, state |
| `reviews` | 복습 1회 | card_id, reviewed_at, rating(1~4), elapsed_days |
| `log` | 시스템 이벤트 | time, event(mail_sent/result_parsed/error…), detail |

### 백업 (영구 보존)
- 매일 03:00 Apps Script가 전 탭을 JSON으로 → GitHub `backup/YYYY-MM-DD/*.json` 커밋 (fine-grained 토큰, 이 repo만, Contents 쓰기)
- Sheets 자체 "버전 기록"도 있어 2중 보존
- 월 1회 Obsidian용 Markdown export (`notes/{연도}-{월}_review.md`) — 선택

---

## 3. 기능 명세 v2

### 3.1 홈 (오늘)
- 오늘 할 일 4칸: ① 복습 N장 ② 독해 1편 ③ 작문 1개(또는 휴식) ④ 말하기 스크립트 1개
- 각 칸에 완료 체크, 연속 학습일(streak), 이번 주 달성률
- "피드백 대기 중" 배지 (메일 보냄 → 결과 도착 전)

### 3.2 독해
- 매일 아침 Gemini 작업이 수집한 3편 중 1편 추천 (나머지는 선택)
- 문장 단위 표시, 표현 탭 → `phrases` 저장 → 카드 생성
- 이해도 질문 3개 → 정답 확인

### 3.3 작문 + "Gemini로 피드백" 버튼
흐름:
1. 과제 선택(주간 과제팩) 또는 자유 작문 → 입력
2. **[Gemini로 피드백]** 클릭 → `submissions`에 저장(status=pending) → 나에게 메일 발송, 제목 `[ES-FB] {submission_id}`, 본문 = 지시문 + 내 글
3. Gemini 작업(이메일 트리거)이 회신 메일 `[ES-RESULT] {submission_id}` 작성 (본문 JSON)
4. Apps Script가 1~5분마다 Gmail 확인 → JSON 파싱 → `feedback` 저장 → 카드 생성 → status=done
5. 30분(설정값) 지나도 결과 없으면 앱에 **[수동으로 받기]** 버튼 표시 → 프롬프트 복사 + Gemini 열기 → 결과를 붙여넣으면 같은 형식으로 저장

피드백 화면: 원문/교정본 diff, 이유, 오류 유형, 대안 2개, 격식도

### 3.4 말하기 (스크립트 낭독)
- 하루 **1개**만 제시. 평일 = 120~150단어(약 1분), 토요일 = 발표형 250단어(약 2분), 일요일 = 없음
- 요일 순환: 월 일상 · 화 학회 커피브레이크 · 수 연구 소개 · 목 Q&A 답변 · 금 랩미팅 · 토 발표
- 화면: 스크립트 + 강세·끊어 읽기 표시(`/`), 핵심 표현 3개
- 체크: **[읽었음]** 버튼 (+ 선택: 읽은 횟수, 스스로 평가 1~5)
- 미완료는 다음날 1개까지만 이월 → 쌓이지 않음 (그 이상은 자동 보관, 나중에 "밀린 스크립트"에서 선택 가능)
- 스크립트의 핵심 표현 3개는 카드로 자동 생성

### 3.5 보관함 (언제든 꺼내보기)
- 탭별 목록: 독해 자료 / 표현 / 내 작문+피드백 / 말하기 스크립트
- 필터: 날짜·유형·태그·오류 유형, 검색창
- 작문 상세: 내 원문 ↔ 피드백 ↔ 재작성 이력을 한 화면에

---

## 4. 복습 시스템

| 주기 | 무엇을 | 방식 |
|---|---|---|
| **매일** | 기한이 된 카드 | FSRS 간격 반복 (Again/Hard/Good/Easy). 신규 카드 하루 최대 10장, 복습 최대 50장 |
| **매일** | 어제 말하기 스크립트 | 핵심 표현 3개 빠른 확인 (카드에 포함) |
| **주간 (일)** | 1·2·4주 전 내 작문 1개 | **다시 쓰기**: 원문 안 보고 같은 과제 재작성 → 버튼으로 피드백 → 이전 버전과 비교 (오류 줄었는지) |
| **주간 (일)** | 이번 주 반복 오류 Top 3 | 해당 오류 카드 집중 세션 |
| **월간 (말일)** | 한 달 기록 | Gemini 작업이 월간 리포트 작성 (학습일, 오류 변화, 많이 쓴 표현) → `notes/`에 저장 |

카드 종류 (전부 자동 생성):
| kind | 앞면 | 뒷면 | 출처 |
|---|---|---|---|
| phrase | 한국어 뜻 + 상황 | 영어 표현 + 예문 | 독해·과제·말하기 |
| correction | 내가 틀린 문장 | 교정 문장 + 이유 | 작문 피드백 |
| translate | 한국어 문장 | 모범 영작 | 한→영 과제 |
| comprehension | 독해 질문 | 정답 | 독해 |

FSRS 계산: Apps Script(JavaScript)로 `ts-fsrs`를 옮기거나 FSRS 공식을 직접 구현 **[테스트 필요: Apps Script 런타임 호환성]**.

---

## 5. Gemini 작업 목록 v2

| # | 이름 | 트리거 | 출력 |
|---|---|---|---|
| T1 | 독해 자료 수집 | 매일 05:00 | 문서 `ES_inbox_reading_날짜` |
| T2 | 주간 과제팩 | 일요일 18:00 | 문서 `ES_inbox_weekpack_날짜`: 작문 과제 6개 + 말하기 스크립트 6개 (한 번에 생성 → 실행 횟수 절약) |
| T3 | **작문 피드백** | **이메일 도착: 제목에 `[ES-FB]`** | 회신 메일 `[ES-RESULT] {id}` (실패 시 문서 `ES_inbox_feedback_{id}`) |
| T4 | 아침 리마인드 | 매일 07:30 | 알림 (오늘 할 일·미완료 체크) |
| T5 | 주간·월간 리뷰 | 일요일 20:00 | 문서 `ES_inbox_review_날짜` |

> 동시 실행 한도 15개 **[커뮤니티]** 이내.
> T4가 읽을 현황은 Apps Script가 매일 04:30 문서 `ES_status`로 써둠 (Gemini가 Sheets를 직접 읽는지 확실하지 않아서 문서로 둠).

### T3 프롬프트 (초안)
```
제목에 "[ES-FB]"가 포함된 메일이 오면 실행.
메일 본문의 "=== MY TEXT ===" 아래 영어 글을 교정해.
결과를 나에게 메일로 보내. 제목: "[ES-RESULT] " + 원래 제목의 ID(예: W-20261105-01)
본문은 아래 JSON만 (설명 문장 금지):
===ES-JSON-BEGIN===
{"submission_id":"","corrected":"","reasons_ko":[""],"error_types":["article|tense|preposition|agreement|word_order|vocabulary|tone"],
 "alternatives":["",""],"formality":3,"sentence_pairs":[{"original":"","corrected":"","why_ko":""}]}
===ES-JSON-END===
메일 보내기가 안 되면 Google 문서 "ES_inbox_feedback_{ID}"로 저장해.
```
(나머지 T1·T2·T4·T5 프롬프트는 03 문서 초안을 v2 형식으로 Phase 1에서 수정)

---

## 6. 개발 로드맵 v2

| 단계 | 내용 | 완료 기준 |
|---|---|---|
| **0. 검증 (지금)** | 아래 7절 테스트 4개 | 피드백 경로·Drive 저장 동작 여부 확정 |
| **1. 기반** | `ES_DB` 시트 생성 스크립트, Apps Script 웹 앱 뼈대(홈·보관함), GitHub 백업 | 폰에서 웹 앱 열림, 빈 DB 생성됨 |
| **2. 작문 + 피드백** | 작문 입력, 메일 발송, `[ES-RESULT]` 수거, 수동 예비 경로 | 버튼 → 피드백이 앱에 도착 |
| **3. 복습** | 카드 자동 생성, FSRS, 복습 화면 | 피드백 다음날 교정 카드 복습 |
| **4. 독해 + 말하기** | T1·T2 수거, 리더, 스크립트 체크 | 매일 자료·스크립트 자동 표시 |
| **5. 정기 복습·통계** | 주간 다시 쓰기, 월간 리포트, 대시보드 | 첫 주간 리뷰 완료 |

---

## 7. Phase 0 검증 테스트 (사용자, 약 30분)

| # | 테스트 | 방법 | 알려줄 것 |
|---|---|---|---|
| A | **이메일 트리거 피드백** | Gemini 앱 → 작업 → 5절 T3 프롬프트로 새 작업, 트리거 "이메일 도착" 선택 → 나에게 제목 `[ES-FB] TEST-01`, 본문 `=== MY TEXT ===` + 영어 2~3문장 메일 발송 | 실행됐는지 / 걸린 시간 / 회신 메일이 왔는지(아니면 문서 생성) / JSON 형식 유지됐는지 |
| B | 트리거 설정 화면 | 이메일 트리거에 "제목에 특정 문자 포함" 같은 조건을 넣을 수 있는지 | 조건 설정 가능 여부 (안 되면 모든 메일에 실행돼 한도 소모 → 대안 필요) |
| C | 문서 저장 | 03 문서 T1 프롬프트를 일정 없이 1회 실행 | 문서 생성 여부, 저장 위치 |
| D | Sheets `=AI()` | 빈 시트에 `=AI("Correct this sentence: He go to school yesterday.")` 입력 | 바로 결과가 나오는지, "생성" 버튼을 눌러야 하는지 |

테스트 결과에 따라 주 경로(A or D)를 확정하고 Phase 1 코드 작성 시작.
