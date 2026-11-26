# TRD — 수준별 학습자료 변환기

> 요구사항: [PRD.md](PRD.md) · 근거: [설계명세서.md](설계명세서.md)
> 버전: v1 (2026-11-26) · 상태: 구현 전 · 구현 모델: Claude Code **Sonnet 5.5** (기획·설계는 Opus)

## 1. 기술 선택 요약
| 영역 | 선택 | 이유 |
|---|---|---|
| 화면 | **HTML + CSS + 순수 JavaScript(ES Modules)**, 빌드 도구 없음 | 연수 교사가 파일을 열어 읽을 수 있음. 4차시 슬라이드의 "내가 정한 HTML·CSS·JS" 방향 |
| 서버 | **Vercel Serverless Function** 1개: `api/feedback.js` (Node.js 20+) | 4차시 공통 프롬프트(슬라이드 16)와 같은 파일 이름. 키를 숨기는 유일한 자리 |
| AI | **Gemini API** REST 호출(SDK 없이 `fetch`), JSON 출력 모드 | npm 패키지 설치 없이 동작. 응답을 스키마로 강제 |
| 모델 | 환경변수 `GEMINI_MODEL`, 기본값 `gemini-2.5-flash` | 무료 등급에서 쓸 수 있는 빠른 모델. **구현 시점에 AI Studio에서 무료 등급 지원 모델명을 다시 확인** |
| 자료 저장소 | **Vercel Blob** (Vercel 대시보드 Storage 탭에서 생성) + 서버 함수 `api/materials.js` | 자료를 짧은 ID로 저장 → 링크가 짧아져 QR에 담김. Vercel 안에서 만들고 토큰이 환경변수로 자동 연결됨 |
| 공유 | `learn.html?id={8자 ID}` | 약 60자 링크 → QR로 바로 읽힘 |
| QR | 브라우저에서 생성 (qrcode-generator) | 서버 호출 없음 |
| 기기 저장 | `localStorage` | 교사 임시 저장, 다문화 결과 재사용, 힌트 횟수 |
| 배포 | GitHub 저장소 → Vercel 연결 | 4차시 흐름(폴더 1개 = 저장소 1개 = 배포 주소 1개) |

외부 라이브러리
- 브라우저(CDN, 정확한 버전): `qrcode-generator@1.4.4` — QR 생성
- 서버(npm, `package.json`): `@vercel/blob` — Blob 저장·조회. 이 앱의 유일한 npm 의존성(Vercel이 배포 때 자동 설치). 버전은 구현 시점 최신 안정판으로 고정

## 2. 폴더 구조
```
level-converter/
├─ index.html          # 교사: 원문 입력 → 변환 → 검토·수정 → 저장·공유
├─ learn.html          # 학생: 수준 선택 → 집중 모드 → 확인 질문·힌트 → 결과 카드
├─ css/
│  └─ style.css
├─ js/
│  ├─ api.js           # /api/feedback 호출 래퍼 (모드별 함수, 오류 메시지 변환)
│  ├─ schema.js        # 수준 자료 검증(형식·개수·정답 범위)
│  ├─ share.js         # saveDoc()·loadDoc()(/api/materials 호출), 공유 링크 만들기, QR 그리기
│  ├─ storage.js       # localStorage 읽기·쓰기(try/catch 포함)
│  ├─ teacher.js       # index.html 동작
│  ├─ learner.js       # learn.html 동작
│  └─ focus.js         # 집중 모드(전체 화면·이탈 감지·타이머)
├─ api/
│  ├─ feedback.js      # 서버 함수: mode = convert | multicultural | hint
│  └─ materials.js     # 서버 함수: 자료 저장(POST) · 조회(GET ?id=)
├─ package.json        # 의존성: @vercel/blob 하나
├─ vercel.json         # 함수 최대 실행 시간 설정
├─ .env.local          # (로컬 전용) GEMINI_API_KEY, BLOB_READ_WRITE_TOKEN — 저장소에 올리지 않음
├─ .gitignore          # .env*, .vercel
├─ CLAUDE.md           # 프로젝트 규칙(키 금지, 파일 역할, 이 TRD 링크)
├─ PRD.md
└─ TRD.md
```

## 3. 아키텍처와 데이터 흐름
```
[교사 브라우저 index.html]
   원문 ─▶ js/api.js ─(POST /api/feedback {mode:"convert", level})×3 병렬─▶ [Vercel api/feedback.js]
                                                                    │  GEMINI_API_KEY (환경변수)
                                                                    ▼
                                                             [Gemini API]
   ◀── 수준 자료 JSON ×3 ── 검증(schema.js) ── 화면 그리기·수정 ──▶ [저장]
   ShareDoc ─(POST /api/materials)─▶ [Vercel api/materials.js] ─▶ [Vercel Blob] materials/{id}.json
   ◀── { id } ── learn.html?id={id} ─▶ 링크 복사 · QR

[학생 브라우저 learn.html?id=…]
   GET /api/materials?id=… → ShareDoc → 수준 선택
     ├ 기초·보통·심화: 링크 속 자료 그대로 사용 (AI 호출 없음)
     └ 다문화+언어: localStorage 확인 → 없으면 POST {mode:"multicultural"} → 저장
   집중 모드(focus.js) → 객관식(브라우저 채점) → 서술형 힌트 POST {mode:"hint", step} ≤3회 → 결과 카드
```
- 학생 화면에서 AI를 부르는 경우는 **다문화 변환**과 **힌트** 둘뿐이다.
- 학생 화면은 Blob 주소를 직접 알지 못하고 항상 `api/materials.js`를 거친다(5차시에 Supabase로 바꿀 때 이 함수와 `share.js`만 고치면 됨).

## 4. 데이터 모델
### 4-1. LevelDoc (한 수준의 자료)
```json
{
  "level": "basic",
  "body": "쉬운 말로 다시 쓴 본문(문단은 \\n\\n로 구분)",
  "glossary": [
    { "word": "토큰", "meaning": "글을 잘게 나눈 조각", "native": null }
  ],
  "questions": [
    { "id": "q1", "type": "choice", "q": "질문", "options": ["①", "②", "③", "④"], "answer": 1 },
    { "id": "q2", "type": "choice", "q": "질문", "options": ["①", "②", "③", "④"], "answer": 3 },
    { "id": "q3", "type": "essay",  "q": "질문", "keyPoints": ["확률", "앞 단어"], "evidence": "답이 들어 있는 본문 문장 일부" }
  ]
}
```
| 필드 | 규칙 |
|---|---|
| `level` | `basic` · `normal` · `advanced` · `multicultural` (화면 표시는 기초·보통·심화·다문화) |
| `glossary` | 4~8개. `native`는 다문화에서만 문자열, 나머지는 `null` |
| `questions` | 정확히 3개, 순서 고정: choice, choice, essay |
| `options` | 정확히 4개 |
| `answer` | **0부터 시작하는 보기 번호(0~3)** |
| `keyPoints` | 서술형 채점 대신 힌트 2단계에 쓰는 핵심 낱말 2~4개 |
| `evidence` | 힌트 3단계에서 본문 위치를 짚는 데 쓰는 문장 일부 |

다문화 LevelDoc에는 `"lang": "vi" | "zh" | "ru" | "en" | "mn"`과 `"aiGenerated": true`가 더해진다.

### 4-2. ShareDoc (Blob에 저장하는 전체 자료)
```json
{
  "v": 1,
  "id": "k3Fp9QxA",
  "title": "GPT의 어머니, 트랜스포머",
  "createdAt": "2026-11-26T09:00:00+09:00",
  "levels": { "basic": { }, "normal": { }, "advanced": { } }
}
```
- `id`는 서버가 저장할 때 붙인다(8자, 영문 대소문자+숫자, `crypto.randomBytes` 기반). 교사가 보낸 값은 무시
- 날짜는 ISO 8601(한국 시간 오프셋 포함)
- `v`는 형식 버전. 학생 화면은 `v`가 다르면 "자료 형식이 맞지 않습니다" 안내
- 다문화 자료는 ShareDoc에 넣지 않는다(학생 기기에서 생성)

### 4-3. localStorage 키
| 키 | 값 | 사용 |
|---|---|---|
| `lc:draft` | 교사 작업 중 ShareDoc | 새로고침 복구(P1) |
| `lc:mc:{docId}:{lang}` | 다문화 LevelDoc | 같은 언어 재호출 방지 |
| `lc:hint:{docId}:{level}:{qid}` | `{ "used": 2, "hints": ["…", "…"] }` | 힌트 횟수·내용 유지 |
| `lc:session:{docId}` | 집중 기록·객관식 응답 | 새로고침해도 결과 유지 |

- `docId` = ShareDoc의 `id`
- 모든 읽기·쓰기는 try/catch로 감싸고, 실패해도 화면은 동작해야 한다(시크릿 창 대비)

## 5. 서버 API 명세 — `POST /api/feedback`
공통
- 요청·응답 `Content-Type: application/json`
- 키: `process.env.GEMINI_API_KEY` (없으면 500 + `{"error":"NO_KEY"}`)
- 모델: `process.env.GEMINI_MODEL || "gemini-2.5-flash"`
- Gemini 호출: `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent`, 헤더 `x-goog-api-key`에 키(쿼리 문자열에 키를 넣지 않음)
- `generationConfig`: `responseMimeType: "application/json"`, `responseSchema`(모드별), `temperature`(아래 표)
- 응답 JSON을 서버에서 한 번 더 검증. 형식이 틀리면 **1회 재시도**, 그래도 틀리면 502 `BAD_AI_OUTPUT`
- 서버는 받은 원문·학생 답을 로그에 남기지 않는다(`console.log`에 본문 출력 금지)

### 5-1. `mode: "convert"` — 교사 수준 변환
요청
```json
{ "mode": "convert", "level": "basic", "title": "…", "source": "원문(최대 6,000자)" }
```
응답 `200`: LevelDoc (`level` = 요청값)
| 항목 | 값 |
|---|---|
| temperature | 0.4 |
| 호출 방식 | 교사 화면이 basic·normal·advanced **3개를 병렬 호출**. [이 수준 다시 만들기]는 같은 요청 1개 |
| 입력 검증 | `source` 50~6,000자, `level` 3종 중 하나 |

### 5-2. `mode: "multicultural"` — 학생 다문화 변환
요청
```json
{ "mode": "multicultural", "lang": "vi", "base": { "...기초 LevelDoc..." } }
```
응답 `200`: LevelDoc (`level: "multicultural"`, `lang`, `aiGenerated: true`, glossary마다 `native` 채움)
| 항목 | 값 |
|---|---|
| temperature | 0.3 |
| 입력 | 기초 자료(원문 아님) → 쉬운 한국어로 다시 쓰고 질문은 기초와 같은 뜻 유지 |
| 입력 검증 | `lang` 5종 중 하나, `base`가 LevelDoc 형식 |

### 5-3. `mode: "hint"` — 소크라테스식 힌트
요청
```json
{
  "mode": "hint", "step": 1, "level": "normal", "lang": null,
  "question": "서술형 질문", "keyPoints": ["확률", "앞 단어"], "evidence": "본문 문장 일부",
  "body": "해당 수준 본문", "studentAnswer": "학생이 지금까지 쓴 답(빈칸 가능, 최대 1,000자)",
  "previousHints": ["1단계 힌트"]
}
```
응답 `200`
```json
{ "step": 1, "hint": "모델이 다음 단어를 고를 때 무엇을 보고 고를까요?" }
```
| 단계 | 내용 | 쓰는 재료 |
|---|---|---|
| 1 되묻기 | 학생 답을 보고 생각을 여는 질문 1~2문장 | question, studentAnswer |
| 2 단서 | 핵심 낱말을 넣어 다시 써 보라는 안내 | keyPoints 중 학생 답에 없는 것 |
| 3 본문 위치 | 답이 있는 본문 부분을 짚음(몇 번째 문단, 어떤 낱말 근처) | evidence, body |
- temperature 0.5
- 서버 검증: `step` 1~3, `previousHints` 길이 = `step - 1`
- **정답 누출 방지**: 응답 `hint`에 `evidence` 문장이 20자 이상 그대로 들어 있으면 1회 재생성 요청
- 다문화(`lang` 있음): 쉬운 한국어 + 핵심 낱말 뒤 괄호에 모국어

### 5-4. 오류 응답
| HTTP | `error` | 화면 안내 |
|---|---|---|
| 400 | `BAD_REQUEST` | 입력을 확인해 주세요 |
| 405 | `METHOD_NOT_ALLOWED` | — |
| 429 | `RATE_LIMIT` (Gemini 429) | 요청이 많아요. 30초 뒤 다시 시도해 주세요 |
| 500 | `NO_KEY` | 서버 설정(환경변수)을 확인해 주세요 — 교사 화면에서만 노출 |
| 502 | `AI_ERROR` / `BAD_AI_OUTPUT` | AI 응답에 문제가 있어요. 다시 시도해 주세요 |
| 504 | `TIMEOUT` | 시간이 오래 걸려요. 다시 시도해 주세요 |
- 서버 → Gemini 요청 제한 시간: convert 50초, multicultural 40초, hint 15초 (`AbortController`)
- `vercel.json`에서 `api/feedback.js`의 `maxDuration`을 60초로 설정

### 5-5. `api/materials.js` — 자료 저장·조회
공통: 토큰 `process.env.BLOB_READ_WRITE_TOKEN`(Blob 스토어를 프로젝트에 연결하면 Vercel이 자동 등록). 없으면 500 `NO_STORE`

**저장** `POST /api/materials`
```json
{ "v": 1, "title": "…", "createdAt": "2026-11-26T09:00:00+09:00", "levels": { "basic": {}, "normal": {}, "advanced": {} } }
```
응답 `201`
```json
{ "id": "k3Fp9QxA" }
```
- 서버 검증: 3수준 모두 LevelDoc 형식(`schema.js`와 같은 규칙), 요청 본문 200KB 이하
- 저장 위치: Blob 경로 `materials/{id}.json`, `contentType: application/json`, 무작위 접미사 끄기, 덮어쓰기 금지(같은 ID가 있으면 ID를 새로 뽑아 1회 재시도)
- 저장할 때마다 새 ID 발급. 고친 자료는 새 링크가 됨(PRD F-S3)

**조회** `GET /api/materials?id=k3Fp9QxA`
응답 `200`: ShareDoc
- `id`는 정규식 `^[A-Za-z0-9]{8}$`만 허용(경로 조작 방지), 아니면 400
- Blob에서 `materials/{id}.json`을 찾아 읽음. 없으면 404 `NOT_FOUND` → 학생 화면 "자료를 찾을 수 없어요. 선생님께 링크를 다시 받아 주세요"
- 응답 헤더 `Cache-Control: public, max-age=300`(같은 반 30명이 열어도 함수 호출이 몰리지 않게)

| HTTP | `error` | 화면 안내 |
|---|---|---|
| 400 | `BAD_REQUEST` | 자료 형식을 확인해 주세요 / 링크 주소가 올바르지 않아요 |
| 404 | `NOT_FOUND` | 자료를 찾을 수 없어요 |
| 413 | `TOO_LARGE` | 자료가 너무 커요. 원문을 줄여 주세요 |
| 500 | `NO_STORE` | 저장소 연결(환경변수)을 확인해 주세요 — 교사 화면에서만 노출 |

## 6. 공유 링크와 QR
- 링크 형식: `https://{배포주소}/learn.html?id={8자 ID}` (약 60자)
- 교사 화면: [저장] → ID 받음 → 링크 표시 + [복사] + QR(qrcode-generator, 오류 정정 수준 M, 화면 폭 240px 이상)
- [QR 크게 보기]: 전체 화면으로 QR만 띄워 교실 화면(프로젝터)에 보여 줌
- Blob 접근 방식: **공개 읽기(public)** 스토어. 경로를 모르면 찾을 수 없지만, ID를 아는 사람은 누구나 자료를 볼 수 있음 → 학습 자료라 허용
- 알려진 위험: 객관식 정답(`answer`)이 조회 응답에 들어 있어, 개발자 도구로 정답을 볼 수 있음 → 4차시는 허용, 5차시 "화면에 안 보임 ≠ 접근 불가"(RLS) 수업 사례로 사용

## 7. 집중 모드 (`focus.js`)
| 기능 | 구현 |
|---|---|
| 시작 | [집중 시작] 클릭 이벤트 안에서 `document.documentElement.requestFullscreen()` (사용자 동작 필요) |
| 미지원 | `document.fullscreenEnabled`가 false면 전체 화면 없이 진행(아이폰 사파리 등) + "이 기기는 전체 화면을 지원하지 않아요" 안내 |
| 이탈 감지 | `visibilitychange`(hidden), `window` `blur`, `fullscreenchange`(전체 화면 해제) 중 하나라도 발생하면 이탈 시작. 중복 이벤트는 이미 이탈 중이면 무시 |
| 복귀 | `visible`·`focus` 시 이탈 종료 → 이탈 시간 누적, 횟수 +1, "자리를 비웠어요(N번째)" 알림(3초 뒤 자동 닫힘) |
| 타이머 | `setInterval` 횟수가 아니라 **`Date.now()` 시각 차이**로 계산(백그라운드 탭에서 타이머가 느려지는 문제 방지). 집중 시간 = 경과 시간 − 이탈 시간 누적 |
| 종료 | [제출] 또는 [집중 끝내기] → `document.exitFullscreen()` → 기록 고정 |
| 저장 | 기록을 `lc:session:{docId}`에 저장해 새로고침 후에도 결과 카드에 반영 |

## 8. 프롬프트 설계 (서버 `api/feedback.js` 안의 systemInstruction)
공통 규칙
- 출력은 지정 JSON 스키마만. 설명·마크다운 금지
- 원문에 없는 사실·수치를 지어내지 않음. 고유명사·수치·연도는 원문 그대로
- 대상: 대한민국 고등학생

수준별 규칙(convert)
| 수준 | 본문 | 낱말 풀이 | 질문 |
|---|---|---|---|
| basic | 쉬운 말, 한 문장 40자 안팎, 핵심 개념만, 일상 비유 1~2개 | 어려운 용어를 일상어로 | 본문에서 바로 찾을 수 있는 내용 |
| normal | 원문 수준과 범위 유지, 문장만 다듬음 | 교과 용어 | 개념 이해 확인 |
| advanced | 전문 용어 사용, 원리 설명 + 확장 질문 1개를 본문 끝에 | 용어의 정확한 정의 | 원리 적용·비교 |

multicultural 규칙
- 기초 자료를 쉬운 한국어(짧은 문장, 한자어 최소화)로 다시 씀
- 본문 속 핵심 낱말 뒤에 괄호로 모국어 병기: `예측(dự đoán)`
- glossary의 `native`에 모국어 뜻, 질문·보기는 기초와 같은 뜻 유지(쉬운 한국어)

hint 규칙
- 정답 문장, 모범 답안, `evidence` 문장을 그대로 쓰지 않음
- 학생 답에 이미 있는 내용은 칭찬 1구절 후 다음 단계로
- 1~2문장, 존댓말("~해 볼까요?")

## 9. 보안·비용 통제
| 항목 | 방법 |
|---|---|
| 키 보관 | Vercel 환경변수 `GEMINI_API_KEY`·`BLOB_READ_WRITE_TOKEN`(Production·Preview), 로컬은 `.env.local`(`vercel env pull`로 받음) |
| 저장 남용 | 누구나 `POST /api/materials`를 부를 수 있음 → 형식 검증·200KB 상한으로 쓰레기 저장을 줄임. 교사 인증은 5차시 이후 과제(문서화) |
| 저장소 보호 | `.gitignore`에 `.env*`, `.vercel` 포함. 첫 커밋 전에 확인 |
| 키 노출 점검 | 배포 후 개발자 도구 Sources·Network에서 키 문자열 검색, GitHub 저장소 검색 |
| 입력 크기 | 원문 6,000자, 학생 답 1,000자, 요청 본문 60KB 초과 시 400 |
| 호출 횟수 | 힌트: 질문당 3회(브라우저 `localStorage` 기준), 다문화: 언어별 1회 캐시 |
| 한계(문서화) | 브라우저 기준 제한이라 기록을 지우면 우회 가능. 서버 쪽 제한은 DB가 생기는 5차시 이후 과제. 무료 등급 키 자체의 분당·일일 한도가 최종 안전장치 |
| 화면 출력 | AI 응답·교사 입력을 화면에 넣을 때 `textContent` 사용(`innerHTML` 금지) — 저장소에서 읽은 자료에 스크립트가 섞여도 실행되지 않게 |

## 10. 로컬 개발과 배포
1. `vercel` CLI 설치 후 `vercel link`로 프로젝트 연결
2. Vercel 대시보드 → 프로젝트 → **Storage → Create → Blob**(공개 읽기) → 이 프로젝트에 연결 → `BLOB_READ_WRITE_TOKEN`이 환경변수에 자동 등록됨
3. Vercel 환경변수에 `GEMINI_API_KEY` 등록(키 값은 대화창·코드에 붙여넣지 않음)
4. `vercel env pull .env.local`로 두 값을 로컬에 받기 → `.gitignore`에 `.env*`가 있는지 확인
5. `npm install`(`@vercel/blob`) → 로컬 실행 `vercel dev` → `http://localhost:3000`
6. GitHub 저장소에 push → Vercel 자동 배포
7. 환경변수를 새로 등록·변경했으면 **재배포**(등록만 하면 반영 안 됨, 4차시 최다 오류)

## 11. 테스트 계획
자동 테스트 도구는 두지 않고(빌드 도구 없음 원칙), 아래 시나리오를 수동으로 확인한다. 각 항목은 PRD 8절 체크리스트와 1:1 대응.
| # | 시나리오 | 기대 결과 |
|---|---|---|
| 1 | 트랜스포머 원문 붙여넣기 → [변환] | 30초 안에 3수준 탭이 채워지고 각 수준 질문이 choice·choice·essay |
| 2 | 보통 본문 수정 → 심화만 [다시 만들기] | 보통 수정 내용 유지, 심화만 바뀜 |
| 3 | [저장] → 휴대폰으로 QR 찍기 | 링크 약 60자, QR이 바로 읽히고 같은 제목·본문이 열림. Vercel Storage 화면에 `materials/{id}.json` 생성 확인 |
| 4 | 다문화+베트남어 → 뒤로 → 다시 베트남어 | 두 번째는 네트워크 요청 없음(개발자 도구 Network 확인) |
| 5 | 집중 시작 → 다른 탭 10초 → 복귀 | "자리를 비웠어요(1번째)", 집중 시간이 약 10초 덜 증가 |
| 6 | 서술형 빈칸으로 힌트 3번 | 1→2→3단계 내용이 다르고 정답 문장 없음, 4번째는 버튼 잠김 |
| 7 | 객관식 2개 풀고 [제출] | 결과 카드에 점수·집중 시간·이탈 횟수·힌트 단계 |
| 8 | 환경변수 삭제 후 [변환] | "서버 설정(환경변수)을 확인해 주세요" |
| 9 | `?id=`를 없는 값·이상한 값(`../x`)으로 바꿔 열기 | "자료를 찾을 수 없어요"/"링크 주소가 올바르지 않아요" 안내, 화면 멈춤 없음 |
| 11 | Blob 스토어 연결을 끊은 뒤 [저장] | "저장소 연결(환경변수)을 확인해 주세요" |
| 10 | 키 노출 점검(9절) | 키 문자열 검색 결과 0건 |

## 12. 구현 순서 (Sonnet에게 한 단계씩 요청)
1. 폴더 구조·`.gitignore`·`CLAUDE.md`·`package.json` 만들기, GitHub·Vercel 연결, Blob 스토어 생성(10절)
2. `api/feedback.js` convert 모드 + `vercel dev`로 응답 JSON 확인
3. `index.html` 원문 입력 → 3수준 병렬 변환 → 탭 표시
4. 수정·[다시 만들기]
5. `api/materials.js` 저장·조회 → `share.js` → 링크·QR
6. `learn.html` 수준 선택·본문·객관식 채점
7. 집중 모드(`focus.js`)
8. hint 모드 + 서술형 힌트 3단계
9. multicultural 모드 + 언어 캐시
10. 결과 카드 → 11절 테스트 전체 → 배포·키 노출 점검

## 13. 5차시 확장 지점
| 지금(4차시) | 5차시에 바뀌는 것 |
|---|---|
| 자료를 Blob 파일(`materials/{id}.json`)로 저장 | Supabase `materials` 테이블로 이전(ID·링크 형식은 그대로) |
| 결과 카드만 표시 | `answers`(class_code, student_no, level, question_no, answer, created_at) 저장, 학생 번호 입력 추가 |
| 정답이 조회 응답에 들어 있음 | 정답은 DB에 두고 RLS로 학생 조회 차단 |
| 누구나 자료 저장 가능 | 교사 구분(학급 코드)·자료 목록 관리 |
| 힌트 제한이 브라우저 기준 | DB 기준 제한 가능 |
- **5차시 도입 동기 정리**: Blob은 "파일을 통째로 넣고 꺼내는 상자"라 학생 30명의 답을 행 단위로 모으고, 조건으로 찾고, 접근 권한을 나누기 어렵다 → 표(테이블)와 권한(RLS)이 있는 DB가 필요
- 이를 위해 `js/share.js`의 `saveDoc()`·`loadDoc()`와 `api/materials.js`의 요청·응답 형식을 지켜, 5차시에는 저장소 쪽만 Supabase로 바꾸면 되게 한다.
