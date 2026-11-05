# Gemini 앱 "작업(Task)" + Google Drive 파이프라인

> ⚠ **최신 계획은 [04_PLAN_v2.md](04_PLAN_v2.md)** (저장소: Drive 문서/GitHub → Google Sheets `ES_DB`, 작문 피드백: 이메일 트리거). 이 문서의 T1·T2 프롬프트는 초안으로 참고.

> 2026-11-04 · v0.2 (Gemini API 호출 방식 → Gemini 앱 "작업" 방식으로 변경)
> 근거 수준: **[공식]** 공식 문서 · **[커뮤니티]** 블로그/기사 · **[추정]** Claude 추정 · **[테스트 필요]** 직접 확인 필요

---

## 0. 결론

```
 Gemini 앱 "작업" (예약 실행)            Google Apps Script (Drive 안에서 자동 실행)        GitHub              PWA 앱
 ───────────────────────────            ─────────────────────────────────────────        ──────              ──────
 매일 05:00 자료 조사·문제 생성  ──▶  Drive: "ES_inbox_…" 문서  ──▶  1시간마다 수거·검증  ──▶  data/ 커밋  ──▶  폰/태블릿/PC
                                                                                                         │
 매일 07:30 리마인드 알림        ◀──  Drive: "ES_status" 문서   ◀──  매일 04:30 기록 요약  ◀──  progress/  ◀──┘ 학습 기록
 매일 21:00 작문 피드백          ◀──  Drive: "ES_outbox_…" 문서 ◀──  제출물 내보내기       ◀──  progress/writing/
```

- **Gemini 작업** = 두뇌 (조사·정리·문제 생성·피드백·리마인드). API 키 불필요, 구독 플랜으로 동작
- **Apps Script** = Drive ↔ GitHub 다리. Google 계정 안에서 돌아가서 Drive 인증 설정이 필요 없음
- **GitHub** = 영구 저장소 + 앱 호스팅
- **PWA 앱** = 학습 화면

### 왜 다리를 GitHub Actions 대신 Apps Script로 하는가
| 항목 | Apps Script (채택) | GitHub Actions |
|---|---|---|
| Drive 접근 | 내 계정으로 바로 접근 (설정 없음) | Google Cloud 프로젝트·OAuth 토큰 발급 필요 |
| 저장해야 할 비밀키 | GitHub 토큰 1개 (이 repo 하나만, 최소 권한) | Drive **전체** 접근 토큰을 GitHub에 저장 |
| 실행 주기 | 시간 기반 트리거 (매시간 등) | cron |
| 코드 편집 | 브라우저(script.google.com)에서 복붙 | repo 파일 |

---

## 1. 확인된 사실 vs 테스트 필요

| 항목 | 상태 | 근거 |
|---|---|---|
| Spark → "작업(Task)"으로 이름 변경 | **[커뮤니티]** 여러 기사 일치 | [layer3labs](https://www.layer3labs.io/guides/what-happened-to-gemini-spark), [insidea](https://insidea.com/blog/ai/how-to-use-gemini-spark) |
| 일정(매일/요일/이벤트) 반복 실행 | **[커뮤니티]** | 위 기사 |
| 동시 실행 작업 최대 15개 | **[커뮤니티]** | 위 기사 |
| AI Pro/Ultra 구독 필요, 지역·플랫폼 단계적 출시 | **[커뮤니티]** 출처마다 다름 | 위 기사 |
| 작업이 Google 문서를 만들어 Drive에 저장 | **[테스트 필요]** 채팅에서 문서 생성 사례는 있음 | [jeffsu.org](https://www.jeffsu.org/newsletter-250/) |
| **특정 폴더에 저장** | **[테스트 필요]** "폴더 지정이 쉽지 않다"는 사용자 보고 있음 | 위 동일 |
| 작업이 Drive 문서를 읽기 (ES_status 등) | **[테스트 필요]** | — |
| Docs → Markdown 내보내기 (`text/markdown`) | **[공식]** | [Drive API export formats](https://developers.google.com/drive/api/guides/ref-export-formats) |

**설계 대응**: 폴더 저장이 안 되더라도 동작하게, Apps Script가 **문서 제목(`ES_inbox_` 접두어)으로 Drive 전체를 검색**해서 수거함. 수거 후 지정 폴더로 이동시킴.

---

## 2. Drive 폴더 구조

```
내 드라이브/
└── english-study/
    ├── inbox/        # Gemini 작업 결과 (가능하면 여기에 저장하도록 지시)
    ├── processed/    # Apps Script가 수거 완료한 문서 이동
    ├── outbox/       # 앱 → Gemini로 보내는 것 (작문 제출물)
    └── ES_status     # 매일 갱신되는 학습 현황 (Gemini 작업이 읽음)
```

### 문서 이름 규칙
| 접두어 | 작성자 | 예 |
|---|---|---|
| `ES_inbox_reading_YYYY-MM-DD` | Gemini 작업 | 독해 자료 |
| `ES_inbox_practice_YYYY-MM-DD` | Gemini 작업 | 작문·말하기 과제, 표현 |
| `ES_inbox_feedback_YYYY-MM-DD` | Gemini 작업 | 작문 피드백 |
| `ES_outbox_writing_YYYY-MM-DD` | Apps Script | 내가 쓴 작문 |
| `ES_status` | Apps Script | 학습 현황 |
| `ES_done_…` | Apps Script | 수거 완료 표시 (이름 변경) |

### 문서 내용 형식 (기계가 읽을 부분)
Gemini가 자유롭게 써도 되지만, 반드시 아래 두 표시 줄 사이에 JSON을 넣게 함. Apps Script는 이 부분만 읽음.
```
===ES-JSON-BEGIN===
{ ... }
===ES-JSON-END===
```
JSON이 깨져 있으면 수거하지 않고 문서 이름을 `ES_error_…`로 바꿔 표시 → 다음날 확인.

---

## 3. Gemini 작업 목록 (5개, 한도 15개 이내)

| # | 이름 | 일정 | 읽는 것 | 만드는 것 |
|---|---|---|---|---|
| T1 | 독해 자료 수집 | 매일 05:00 | `ES_status` | `ES_inbox_reading_날짜` |
| T2 | 오늘의 연습 과제 | 매일 05:30 | `ES_status` | `ES_inbox_practice_날짜` |
| T3 | 작문 피드백 | 매일 21:00 | `ES_outbox_writing_*` (미처리) | `ES_inbox_feedback_날짜` |
| T4 | 아침 리마인드 | 매일 07:30 | `ES_status` | 알림 메시지 (파일 없음) |
| T5 | 주간 리뷰 | 일요일 20:00 | `ES_status` | `ES_inbox_weekly_날짜` |

> 각 작업 프롬프트는 아래 4절. Gemini 앱 → "작업" → 새 작업 → 프롬프트 붙여넣기 → 일정 설정.

---

## 4. 작업 프롬프트 (복사해서 사용)

### T1. 독해 자료 수집
```
너는 물리·광학(레이저/포토닉스) 연구자의 영어 독해 자료 담당이다.

1) Google Drive에서 제목이 "ES_status"인 문서를 읽고, 사용자의 현재 주차·약점·관심 키워드를 파악해.
   (없으면 기본값: 주차 1, 키워드 ultrafast laser, nonlinear optics, photonics)
2) 오늘 읽을 영어 원문 3편을 웹에서 찾아:
   - 연구: arXiv physics.optics 최근 초록 1편 (키워드 관련)
   - 과학 뉴스: Phys.org / Nature News / Optica 뉴스 등 1편
   - 일상: 일반 영어 기사·에세이 1편 (CEFR B2~C1)
3) 각 글마다: 제목, URL, 원문 발췌(오픈 라이선스가 아니면 400단어 이하), 한국어 3줄 요약,
   핵심 표현 5개(영어 표현, 뜻, 원문 예문), 이해도 질문 3개(정답 포함), CEFR 레벨.
4) Google Drive의 "english-study/inbox" 폴더에 Google 문서로 저장해. 폴더 저장이 안 되면 내 드라이브 최상위에 저장.
   문서 제목: ES_inbox_reading_{오늘 날짜 YYYY-MM-DD}
5) 문서 맨 끝에 아래 형식의 JSON을 정확히 넣어. 표시 줄은 글자 그대로 유지.

===ES-JSON-BEGIN===
{"type":"reading","date":"YYYY-MM-DD","items":[
 {"category":"research|news|daily","title":"","url":"","cefr":"B2","excerpt":"","summary_ko":"",
  "phrases":[{"en":"","ko":"","example":""}],
  "questions":[{"q":"","a":""}]}
]}
===ES-JSON-END===
```

### T2. 오늘의 연습 과제
```
너는 연구자 대상 영어 코치다.
1) Google Drive 문서 "ES_status"에서 현재 주차와 약점을 확인해.
2) 요일에 맞는 과제를 만들어:
   월=이메일 작문, 화=일상 회화, 수=논문 작문, 목=학회 Q&A, 금=한→영 영작, 토=발표 연습, 일=휴식(복습만).
   - 작문: 상황 설명, 요구 조건, 참고 표현 5개, 모범 답안 1개
   - 회화/Q&A: 역할극 시나리오, 상대 대사 5개, 내가 쓸 만한 표현 5개
   - 발표: 1분 분량 스크립트 주제와 구조
   - 한→영: 한국어 문장 5개와 모범 영작
3) 연구 주제 표현 3개를 추가해 (논문 섹션별).
4) "english-study/inbox"에 Google 문서로 저장 (안 되면 최상위). 제목: ES_inbox_practice_{YYYY-MM-DD}
5) 문서 끝에:
===ES-JSON-BEGIN===
{"type":"practice","date":"YYYY-MM-DD","mode":"email|conversation|paper|qa|translate|presentation|rest",
 "task":{"title":"","instructions_ko":"","reference_phrases":[{"en":"","ko":""}],"model_answer":""},
 "extra_phrases":[{"en":"","ko":"","section":"abstract|intro|method|result|discussion|email|talk"}]}
===ES-JSON-END===
```

### T3. 작문 피드백
```
Google Drive에서 제목이 "ES_outbox_writing_"으로 시작하고 아직 "[reviewed]" 표시가 없는 문서를 모두 찾아.
각 문서의 내 영작에 대해:
- 교정본, 수정 이유(한국어), 오류 유형(관사/시제/전치사/수일치/어순/어휘/톤), 더 자연스러운 대안 2개, 격식도 평가(1~5)
"english-study/inbox"에 Google 문서로 저장 (안 되면 최상위). 제목: ES_inbox_feedback_{YYYY-MM-DD}
처리한 outbox 문서 제목 끝에는 " [reviewed]"를 붙여 (가능하면).
문서 끝에:
===ES-JSON-BEGIN===
{"type":"feedback","date":"YYYY-MM-DD","items":[
 {"source_doc":"","original":"","corrected":"","reasons_ko":[""],"error_types":[""],"alternatives":["",""],"formality":3}
]}
===ES-JSON-END===
```

### T4. 아침 리마인드
```
Google Drive 문서 "ES_status"를 읽고, 오늘 할 일을 5줄 이내 한국어로 알려줘:
오늘 복습 카드 수, 3일 이상 안 한 모듈, 오늘 과제 종류, 연속 학습일, 응원 한 줄.
파일은 만들지 마.
```

### T5. 주간 리뷰
```
"ES_status"를 읽고 이번 주를 분석해: 모듈별 학습 일수, 반복 오류 Top 3, 다음 주 집중 포인트 3개,
다음 주 학습 난이도 조정 제안. "english-study/inbox"에 ES_inbox_weekly_{YYYY-MM-DD}로 저장하고 끝에 JSON:
===ES-JSON-BEGIN===
{"type":"weekly","date":"YYYY-MM-DD","top_errors":[""],"focus_next_week":[""],"difficulty_adjust":"up|keep|down","note_ko":""}
===ES-JSON-END===
```

---

## 5. Apps Script 다리 (Drive ↔ GitHub) — 역할만, 코드는 Phase 1에서 작성

| 함수 | 트리거 | 동작 |
|---|---|---|
| `collectInbox()` | 매시간 | Drive 전체에서 `ES_inbox_*` 검색 → Markdown으로 내보내기 → JSON 추출·검증 → GitHub `data/{type}/{date}.json`에 커밋 → 문서를 `ES_done_*`로 이름 변경 + `processed/`로 이동 |
| `exportStatus()` | 매일 04:30 | GitHub `progress/` 읽기 → 복습 예정 수, 미학습 모듈, 약점, streak 계산 → Drive `ES_status` 문서 덮어쓰기 |
| `exportOutbox()` | 매일 20:30 | GitHub `progress/writing/pending/*` → Drive `outbox/ES_outbox_writing_*` 생성 |

- GitHub 토큰: **fine-grained personal access token**, 대상 repo = `english-study` 하나, 권한 = Contents: Read and write. Apps Script의 "스크립트 속성"에 저장 (코드에 직접 쓰지 않음).
- Apps Script 하루 실행 한도 존재 (현재 수치 미확인) → 매시간 1회 정도면 여유 있을 것으로 **[추정]**.

---

## 6. API 없이 생기는 제약과 대응

| 제약 | 영향 | 대응 |
|---|---|---|
| 실시간 피드백 불가 (작업은 예약 실행) | 작문 피드백이 당일 저녁에 도착 | ① 비동기(T3) 기본 ② 앱의 "Gemini로 보내기" 버튼: 프롬프트+내 글을 복사하고 Gemini 앱을 열어 즉시 피드백 |
| 말하기 실시간 대화 | 앱 안에서 AI와 대화 불가 | 앱이 시나리오를 보여주고 → Gemini 앱 **Live**로 대화 → 대화 후 Gemini에 "교정 리포트 JSON으로" 요청 → 붙여넣기로 앱에 저장 |
| 출력 형식이 가끔 깨짐 | 수거 실패 | `ES_error_` 표시 + 앱 대시보드에 경고, 다음 실행에서 재생성 |
| 폴더 지정이 안 될 수 있음 | 문서가 최상위에 쌓임 | 제목 검색으로 수거 후 자동 이동 |

---

## 7. Phase 0 테스트 순서 (사용자 직접, 약 30분)

1. [ ] Drive에 `english-study/inbox`, `processed`, `outbox` 폴더 생성
2. [ ] Drive 최상위에 `ES_status` 문서 생성, 내용: `주차: 1 / 키워드: ultrafast laser, nonlinear optics / 약점: 미정`
3. [ ] Gemini 앱 → 작업 → **T1 프롬프트를 일정 없이 1회 실행**
4. [ ] 확인 후 결과를 알려주면 됨:
   - (a) 문서가 생성됐는가? (b) inbox 폴더에 들어갔는가, 최상위인가? (c) `ES_status`를 읽었는가? (d) JSON 블록이 형식대로 들어갔는가?
5. [ ] 결과에 따라 프롬프트·Apps Script 수정 → 일정 등록
