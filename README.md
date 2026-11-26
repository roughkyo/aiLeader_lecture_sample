# 수준별 학습자료 변환기 — GPT의 어머니, 트랜스포머

2026 인공지능 활용 선도교사 후속전문연수(4차시: 인공지능 API를 활용한 AI 기능 구현) 샘플 웹 앱입니다.  
하나의 원문 학습자료를 **기초·보통·심화·다문화** 4가지 수준으로 제공하고, **집중 모드**, **본문 중간 활동 및 형성평가**, **Gemini 기반 소크라테스식 힌트(학생 질문 포함)** 기능을 지원합니다.

## 주요 기능

- **수준별 학습자료 읽기**: 기초 · 보통 · 심화 수준 전환, 글자 크기 조절(보통·크게·아주 크게), 브라우저 음성 읽어 주기(TTS)
- **다문화 모드**: 쉬운 한국어 본문 + 밑줄 친 핵심 낱말 클릭 시 쉬운 뜻·모국어 뜻(베트남어·중국어) 말풍선 및 발음 듣기
- **집중 모드**: 전체 화면 전환, 집중 시간 측정, 다른 탭 이동·화면 이탈 감지 알림
- **학습 활동 및 형성평가**: 본문 중간 활동(하나 고르기·골라 넣기·빈칸 채우기·중요 단어 찾기) 및 형성평가 자동 채점
- **소크라테스식 AI 힌트 (`POST /api/feedback`)**:
  - 서술형 문항에서 학생이 궁금한 점을 직접 입력하거나 바로 힌트 요청(최대 3단계: 되묻기 → 낱말 단서 → 본문 위치 안내)
  - 정답 직접 요구 시 정답을 알려 주지 않고 스스로 생각해 보도록 되묻기
  - 최신 **Gemini Interactions API**(`gemini-3.5-flash-lite` 기본, `gemini-3.8-flash` 지원) 사용 및 `store: false`로 서버 미저장

## 폴더 구조

```text
.
├─ index.html              # 학생 화면(수준 선택 · 읽기 · 집중 모드 · 형성평가)
├─ css/style.css           # 스타일시트
├─ js/
│  ├─ app.js               # 화면 전환 · 마크다운 로드
│  ├─ reader-tools.js      # 글자 크기 · 읽어 주기 · 다문화 낱말 말풍선
│  ├─ focus.js             # 집중 모드(전체 화면 · 이탈 감지 · 타이머)
│  └─ activities.js        # 본문 활동 · 형성평가 · AI 힌트 요청 UI
├─ materials/              # 수준별 학습자료(기초·보통·심화·다문화.md) 및 activities.json
├─ api/feedback.js         # 서버 함수 — Gemini 소크라테스식 힌트
├─ dev-server.mjs          # 로컬 개발 서버(Node.js 20+, 외부 패키지 없음)
├─ package.json
└─ .env.example            # 환경변수 템플릿
```

## 로컬 실행 방법

Node.js 20 이상이 필요합니다(외부 npm 패키지 설치 불필요).

1. `.env.example`을 복사해 `.env.local` 파일을 만들고 Google AI Studio에서 발급한 API 키를 입력합니다.
   ```env
   GEMINI_API_KEY=여기에_API_키_입력
   # 선택: 비워 두면 gemini-3.5-flash-lite (gemini-3.8-flash 등으로 변경 가능)
   GEMINI_MODEL=
   # 선택: 비워 두면 low (low / medium / high)
   GEMINI_THINKING_LEVEL=
   ```
2. 로컬 개발 서버를 실행합니다.
   ```bash
   npm run dev
   ```
3. 브라우저에서 `http://127.0.0.1:5173` 으로 접속합니다.

## 배포 및 연계 (GitHub → Supabase)

- 이 저장소는 GitHub에서 **Supabase**와 직접 연계·배포하도록 구성되어 있습니다.
- 배포 환경변수(Secrets)에 아래 항목을 등록해 사용합니다.
  - `GEMINI_API_KEY` (필수)
  - `GEMINI_MODEL` (선택, 기본값 `gemini-3.5-flash-lite`, 예: `gemini-3.8-flash`)
  - `GEMINI_THINKING_LEVEL` (선택, 기본값 `low`)

## 보안 및 개인정보 원칙

- `.env.local` 등 비밀값이 포함된 파일은 `.gitignore`로 제외하며 절대 저장소에 커밋하지 않습니다.
- Gemini API 키는 서버 함수(`api/feedback.js`)의 환경변수로만 읽으며 브라우저에 전달하지 않습니다.
- 학생 답안·질문·본문은 서버 로그에 남기지 않으며 Gemini Interactions API 호출 시 `store: false`를 적용합니다.
