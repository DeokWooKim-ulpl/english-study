# Progress Notes

## 2026-11-04
**한 일**
- 구축 계획서(docs/01_PLAN.md), 학습 계획(docs/02_STUDY_PLAN.md) 작성

**다음 할 일**
- 사용자: Gemini API 키 발급 → GitHub Secret `GEMINI_API_KEY` 등록, repo 공개 여부·리마인드 수단·관심 분야 키워드 결정
- Phase 1 MVP: PWA 뼈대(홈/독해/복습) + daily-collect Action + FSRS 카드

## 2026-11-04 (2)
**한 일**
- 자동화 엔진 변경: Gemini API → Gemini 앱 "작업(Task)" + Google Drive + Apps Script 다리 (docs/03_GEMINI_TASK_PIPELINE.md)
- 작업 프롬프트 T1~T5, Drive 이름 규칙, JSON 형식 확정(초안)

**다음 할 일**
- 사용자: Phase 0 테스트 (T1 1회 실행 → 문서 생성/폴더/JSON 확인 결과 공유)
- Claude: 테스트 결과 반영 → Apps Script `collectInbox()` 작성 → PWA MVP

## 2026-11-05
**한 일**
- 계획 v2 작성 (docs/04_PLAN_v2.md): 이메일 트리거로 Gemini 작업 자동 피드백, Google Sheets `ES_DB`를 DB로, Apps Script 웹 앱, GitHub 백업, 말하기 = 스크립트 1개/일 + 체크, 정기 복습(일/주/월)
- 학습 계획 요일표 수정 (02)

**다음 할 일**
- 사용자: 04 문서 7절 테스트 A~D 결과 공유
- Claude: 결과 반영 → Phase 1 (ES_DB 생성 스크립트, 웹 앱 뼈대, 백업)
