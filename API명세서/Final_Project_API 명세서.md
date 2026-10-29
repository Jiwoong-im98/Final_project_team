# Final\_Project\_API 명세서

Oct 29, 2026 · @혜현

## 1. 개요

이 문서는 프런트엔드(Next.js)와 백엔드(FastAPI)가 주고받는 REST API 64개와 면접 실시간 메시지 22종을 한곳에 정리한 명세다. 경로·요청·응답·오류를 여기서 맞추고, 칸 하나하나의 DB 정의는 테이블 정의서를 따른다.

| 항목 | 내용 |
| --- | --- |
| 기준 문서 | TRD v1.5(5장·4-3·14장), PRD v1.3, 화면정의서 v0.2(§12 추가 API) |
| 화면 주소 | `http://127.0.0.1:8080` — 같은 PC 브라우저에서만 접속 |
| REST 기본 주소 | `http://127.0.0.1:8080/api` — frontend가 Next.js rewrites로 api(8000)에 넘긴다. 이 문서의 경로는 `/api`를 뺀 값 |
| 실시간 주소 | WebSocket `ws://127.0.0.1:8000/ws/interview/{session_id}` · WebRTC 음성 UDP 40000\~40019 |
| 인증 | 로그인 쿠키(HttpOnly, SameSite=Lax). 화면과 API가 같은 주소라 쿠키가 그대로 쓰인다 |
| 표기 | 🔒 지원자 로그인 · 🛡 관리자(`require_admin`) · 🌐 로그인 불필요 · ➕ TRD 5-1에 없어 이 문서에서 추가한 API |

➕ 표시 API는 화면정의서 §12에서 필요하다고 정리한 것을 TRD 규칙에 맞춰 경로를 정한 것이다. TRD 다음 개정 때 5-1에 옮겨 적어야 한다.

## 2. 공통 규칙

모든 API는 JSON을 주고받고, 오류는 `{"code", "message"}` 한 가지 모양이다(TRD 4-3). 화면은 `code`로 안내 문구를 고른다.

### 2-1. 데이터 형식

| 규칙 | 내용 |
| --- | --- |
| 형식 정의 | 요청·응답·WebSocket 메시지 모두 Pydantic 모델. 모델에 없는 칸이 오면 거절(`extra="forbid"`, 422) |
| id | UUID 문자열 |
| 시각 | 한국 시간이 붙은 ISO 8601 — `2026-11-03T10:00:00+09:00` |
| 점수 | 소수 오차를 막기 위해 문자열 — `"70.15"` (소수점 둘째 자리) |
| 목록 | `?page=1&size=20`, 응답은 `{"items": [...], "total": 0, "page": 1, "size": 20}` |
| 파일 업로드 | `multipart/form-data`. 서버 보호 한도 `upload_max_mb`(50MB)는 화면에 표시하지 않는다 |
| 개인정보 | 이름·연락처는 화면 응답에만 넣고, LLM 입력과 Redis 메시지에는 넣지 않는다 |

### 2-2. 인증·권한

- **로그인 유지**: 쿠키 세션. 마지막 요청 후 2시간이 지나면 만료되고, 면접 중에는 계속 늘린다. `cookie_secure`는 지금 끔(http 실행).
- **관리자 API**(`/admin/*`): `require_admin`. 지원자가 부르면 403.
- **지원자 API**: `require_owner`. 모든 조회에 `applicant_id = 로그인한 사람` 조건을 넣는다. 다른 사람 데이터 번호로 부르면 있는지도 알 수 없게 404.
- **면접 API**(`/interview/{sid}/*`, `/coding/{sid}/*`, WebSocket): 면접 시작 때 받은 세션 토큰(`sid`)과 로그인 쿠키를 함께 확인한다. WebSocket은 `Origin` 헤더가 `http://127.0.0.1:8080`이 아니면 끊는다.
- **관리자 초기 비밀번호**: 바꾸기 전에는 `/auth/password-change`와 `/auth/logout` 말고는 403 `PASSWORD_CHANGE_REQUIRED`.
- **결과·결정**: 관리자 API에서도 읽기만 된다. 바꿀 수 있는 것은 메모·설정·다시 돌리기뿐이고, 판정을 고치는 주소는 없다(AC-14).

### 2-3. HTTP 상태 코드

| 코드 | 뜻 | 주로 쓰는 곳 |
| --- | --- | --- |
| 200 · 201 · 202 · 204 | 성공 · 만듦 · 뒤에서 처리 시작 · 내용 없음 | 202는 이력서 추출·AI 초안·재처리처럼 worker가 이어서 하는 일 |
| 400 | 요청 값이 규칙에 맞지 않음 | 비밀번호 규칙, 가중치 합계 |
| 401 | 로그인 필요 | 쿠키 없음·만료 |
| 403 | 권한 없음 | 지원자가 `/admin` 호출, 이메일 미인증 |
| 404 | 없음(또는 보여 줄 수 없음) | 남의 데이터, 공개 전 결과 |
| 409 | 지금 상태에서는 안 됨 | 잠긴 기준 수정, 마감 뒤 시작·이어가기, 두 번째 응시 |
| 413 | 서버 보호 한도 초과 | 이력서 파일 > `upload_max_mb` |
| 422 | 형식 오류 | 필수 칸 빠짐, 모델에 없는 칸 |
| 423 | 잠김 | 로그인 5번 실패 후 10분 |
| 429 | 자리가 꽉 참 | 면접 시작(응시 횟수 안 씀) |
| 500 · 503 | 서버·AI 서버 오류 | 면접 중이면 장애 후보(TRD 7-2) |

그 밖에 410(보관 기간이 지난 녹화·음성)과 502(Open API가 답하지 않음)도 쓴다. 오류 응답 예:

```json
{ "code": "CRITERIA_LOCKED", "message": "첫 응시가 시작되어 평가 기준을 바꿀 수 없습니다." }
```

### 2-4. 응시 상태 값

여러 API가 같이 쓰는 `attempt_status` 값이다. '대기'는 저장하는 상태가 아니라 429 안내 화면일 뿐이다(TRD 7-1).

| 값 | 화면 표시 |
| --- | --- |
| `drafting` | 작성 중 (화면정의서 G-01) |
| `ready` | 준비 중 |
| `interviewing` | 면접 진행 중 |
| `coding` | 코딩테스트 진행 중 |
| `interrupted` | 장애 중단 |
| `completed` | 응시 완료 |
| `ended` | 응시 종료 (창 닫기·권한 끄기) |
| `ended_closed` | 응시 종료(마감) |
| `cancelled` | 응시 취소 (화면 이탈 2번) |
| `not_taken_closed` | 미응시(마감) |

분석 상태 `analysis_status`: `pending`(분석 대기) · `running`(분석 중) · `decided`(확정) · `needs_review`(확인 필요). 공개 상태 `release_status`: `scheduled` · `delayed`(채점이 늦어 다음 날 10:00 공개) · `released` · `purged`.

## 3. 인증·계정 API

가입은 지원자만 하고, 관리자 계정 1개는 설치할 때 만든다(N8). 비밀번호 규칙은 가입·변경·재설정 세 API가 같은 함수로 확인한다.

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `POST /auth/signup` | 🌐 | 지원자 가입 | S-C02 |
| `POST /auth/login` | 🌐 | 로그인, 쿠키 발급 | S-C01 |
| `POST /auth/logout` ➕ | 🔒🛡 | 로그아웃, 쿠키 삭제 | 공통 머리글 |
| `GET /auth/me` ➕ | 🔒🛡 | 지금 로그인한 사람·역할 (라우트 가드용) | 전체 |
| `POST /auth/password-reset` | 🌐 | 재설정 링크 메일 요청(30분 유효) | S-C03 ① |
| `POST /auth/password-reset/confirm` ➕ | 🌐 | 링크 토큰으로 새 비밀번호 저장 | S-C03 ② |
| `POST /auth/password-change` ➕ | 🔒🛡 | 현재 비밀번호로 확인 후 변경 (관리자 초기 변경 포함) | S-C04, S-A13 |
| `PATCH /me` ➕ | 🔒 | 이름·휴대전화 수정 (이메일은 못 바꿈) | S-A13 계정 |
| `DELETE /me` ➕ | 🔒 | 탈퇴: 진행 중 응시 취소, 응시 데이터·계정 즉시 삭제 | S-A13 계정 |
| `POST /auth/email-verification` ➕ | 🔒 | 인증 메일 다시 보내기(1분에 1번). 나중에 붙이는 기능 | 추후 |
| `POST /auth/email-verification/confirm` ➕ | 🌐 | 24시간 토큰으로 인증, `email_verified_at` 기록 | 추후 |

### POST /auth/signup

요청

```json
{
  "name": "홍길동",
  "email": "hong@example.com",
  "password": "Hire#2026ai",
  "phone": "010-1234-5678",
  "agreements": { "age_14_plus": true, "terms": true, "privacy": true }
}
```

응답 201 — `{ "user_id": "9f1c…", "email": "hong@example.com" }`. 가입만 하고 로그인은 따로 한다(화면은 S-C01로 이동).

| 오류 | code | 조건 |
| --- | --- | --- |
| 400 | `PASSWORD_POLICY` | 8\~64자, 영문·숫자·특수문자 각 1자, 이메일 아이디 포함 금지, 같은 문자 4번 연속 금지 중 하나라도 어김. `message`에 어긴 규칙 |
| 400 | `AGREEMENT_REQUIRED` | 필수 동의 3개 중 빠진 것 |
| 409 | `EMAIL_TAKEN` | 이미 가입된 이메일 |

### POST /auth/login

요청 — `{ "email": "hong@example.com", "password": "…", "next": "/postings/3a7e…" }`

응답 200 (`Set-Cookie: session=…; HttpOnly; SameSite=Lax`)

```json
{
  "user_id": "9f1c…",
  "role": "applicant",
  "password_change_required": false,
  "redirect": "/postings/3a7e…"
}
```

- `role`: `applicant` · `admin`. 관리자 첫 로그인이면 `password_change_required: true`, `redirect: "/admin/password"`.
- `next`는 `/postings/` · `/applications/` · `/me`로 시작할 때만 받는다. `//` · `/\` · 다른 도메인이면 `/me`로 바꾼다(TRD 11장). 관리자는 `/admin`으로 보낸다.
- 오류: 401 `INVALID_CREDENTIALS`(이메일·비밀번호 중 무엇이 틀렸는지 알리지 않음) · 423 `ACCOUNT_LOCKED`(5번 틀리면 10분, `message`에 풀리는 시각).

### POST /auth/password-reset · /confirm

- 요청: `{ "email": "…" }` → 항상 202. 가입 여부를 드러내지 않는다.
- 확정: `{ "token": "…", "new_password": "…" }` → 204. 오류 400 `TOKEN_EXPIRED`(30분 지남) · 400 `PASSWORD_POLICY`.

### POST /auth/password-change

요청 — `{ "current_password": "…", "new_password": "…" }` → 204. 오류 400 `WRONG_PASSWORD` · 400 `PASSWORD_POLICY` · 400 `SAME_AS_OLD`. 바꾸면 관리자의 `password_change_required`가 풀린다.

### PATCH /me · DELETE /me

- `PATCH /me` 요청: `{ "name": "홍길동", "phone": "010-1234-5678" }`(바꿀 칸만) → 200 회원 정보.
- `DELETE /me` 요청: `{ "password": "…", "confirm_text": "탈퇴" }` → 204, 쿠키 삭제. 진행 중 면접이 있으면 먼저 `cancelled`로 끝내고 자리를 돌려준 뒤 지운다. 지운 기록은 `deletion_logs`에 남는다.

## 4. 공고 API

공개 조회는 `/postings`, 관리는 모두 `/admin/postings` 아래에 둔다. TRD 5-1은 관리용을 `POST /postings`처럼 적고 "/admin 아래"라고 덧붙였는데, 이 문서는 주소에 `/admin`을 붙이는 쪽으로 맞춘다.

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `GET /postings` | 🌐 | 게시 중 공고 목록 | S-A01 |
| `GET /postings/{id}` | 🌐 | 공고 상세. 로그인했으면 내 지원 상태도 | S-A02 |
| `GET /job-codes` ➕ | 🌐 | 직무 목록(준비 중 직무 포함) | S-A01, S-R03 |
| `GET /admin/postings?status=` ➕ | 🛡 | 관리자 공고 목록: `draft` · `published` · `closed` | S-R01 |
| `POST /admin/postings` | 🛡 | 임시저장 공고 만들기 | S-R03 |
| `GET /admin/postings/{id}` ➕ | 🛡 | 편집용 상세(채점 기준·잠금 여부 포함) | S-R03, S-R04 |
| `PATCH /admin/postings/{id}` | 🛡 | 공고 내용·채점 기준 고치기. 기준이 잠기면 기준 칸만 409 | S-R03 |
| `POST /admin/postings/{id}/publish` | 🛡 | 게시. 마감 시각 예약 작업을 건다 | S-R03 3단계 |
| `POST /admin/postings/{id}/close` | 🛡 | 지금 마감. 진행 중 면접도 바로 끝남(INT-10) | S-R01 |
| `DELETE /admin/postings/{id}` ➕ | 🛡 | 지원자 0명일 때만 삭제 | S-R01 |
| `POST /admin/postings/import` | 🛡 | Open API·더미 JSON/CSV에서 불러오기 → 임시저장 | S-R01 모달 |
| `POST /admin/postings/{id}/draft-ai` | 🛡 | AI 공고 초안 만들기(채점용 LLM, 뒤에서) | S-R02 |
| `GET /admin/postings/{id}/draft-ai/status` ➕ | 🛡 | 초안 만들기 진행 상태 | S-R02 |

### 공고 객체

```json
{
  "posting_id": "3a7e…",
  "status": "published",
  "source": "manual",
  "title": "AI/AX 엔지니어 (신입·경력)",
  "job_code": "AI_AX",
  "company_name": "테스트컴퍼니",
  "company_intro": "…",
  "duties": "…",
  "requirements": "…",
  "preferred": "…",
  "employment_type": "full_time",
  "location": "서울 강남구",
  "headcount": 3,
  "open_at": "2026-11-02T09:00:00+09:00",
  "close_at": "2026-11-10T18:00:00+09:00",
  "release_at": "2026-11-11T10:00:00+09:00",
  "contact_email": "recruit@example.com",
  "has_coding_test": true
}
```

- `source`: `manual` · `open_api` · `dummy` · `ai_draft`(화면정의서 §12 #18 추가값).
- `contact_email`은 화면정의서 D-03에서 추가한 칸이다. 기본값은 관리자 이메일.
- `GET /postings/{id}`는 로그인한 지원자에게 `my_application: { "application_id", "attempt_status" }`(없으면 `null`)를 더 준다. 카드 버튼은 화면정의서 §3.4 표를 따른다.
- `GET /postings`는 `?job_code=&page=&size=`. 게시 중(`published`)만 내보낸다.

### 채점 기준 객체 (관리자 응답에만)

```json
"criteria": {
  "version": 1,
  "locked": false,
  "weights": { "expertise": 30, "logic": 25, "relevance": 20, "delivery": 10, "culture_fit": 15 },
  "coding_enabled": true,
  "coding_ratio": 30,
  "pass_score": 70
}
```

- 가중치는 5 단위, 항목마다 5 이상, 합계 100. 틀리면 400 `WEIGHT_SUM`.
- `coding_ratio`는 10 단위(기본 30 = 면접 70 : 코딩 30). `coding_enabled: false`면 무시한다.
- `pass_score`는 0\~100 정수, 기본 70.
- 첫 응시가 시작되면 `locked: true`. 그 뒤 `PATCH`에 `criteria`가 있으면 409 `CRITERIA_LOCKED`, 공고 내용 칸은 그대로 바뀜다(JOB-07).

### POST /admin/postings/{id}/publish · /close

- 게시 조건: 필수 칸이 채워짐, `open_at ≤ close_at`, `close_at`이 지금 뒤, 직무가 AI/AX 엔지니어. 아니면 400 `POSTING_INCOMPLETE` 또는 400 `JOB_NOT_OPEN`.
- `close` 응답 200 — `{ "closed_at": "…", "not_taken": 4, "ended_by_close": 1 }`. 시작 전 지원은 `not_taken_closed`, 진행 중·장애 중단은 `ended_closed`가 되고 채점 작업이 들어간다(TRD 10장).
- `DELETE`: 지원자가 있으면 409 `HAS_APPLICANTS`.

### POST /admin/postings/import

요청 — Open API는 `{ "source": "open_api", "query": { "keyword": "AI", "limit": 20 } }`(JSON), 더미는 `multipart/form-data`로 `source=dummy`와 `file`(JSON·CSV).

응답 200

```json
{ "imported": 12, "skipped_duplicate": 3, "needs_job_check": 2, "posting_ids": ["…"] }
```

같은 `external_id`는 건너뛴다. 모두 `draft`로 들어오고, 직무가 AI/AX가 아니면 `needs_job_check`로 센다. Open API가 답하지 않으면 502 `IMPORT_SOURCE_FAILED`.

### POST /admin/postings/{id}/draft-ai

빈 임시저장 공고를 먼저 만들고(`POST /admin/postings`), 그 id로 부른다. 화면을 떠나도 만들기는 계속되고, 결과는 그 공고에 채워진다(G-06).

```json
{
  "job_code": "AI_AX",
  "career_level": "junior", "career_years": 2,
  "employment_type": "full_time",
  "location": "서울 강남구",
  "headcount": 3,
  "open_at": "…", "close_at": "…",
  "duties": ["모델 개발", "데이터 파이프라인"],
  "skills": ["Python", "PyTorch"],
  "preferred": "…",
  "company_name": "테스트컴퍼니", "company_intro": "…",
  "emphasis": "…",
  "coding_test": true
}
```

응답 202 — `{ "job_id": "…", "status": "running" }`. `status` 조회는 `{ "status": "running" | "done" | "failed", "posting_id", "bias_warnings": [ { "text": "만 30세 이하", "reason": "나이 제한" } ] }`. 고용 형태·근무지·인원·기간·회사명·코딩테스트 여부는 AI를 거치지 않고 그대로 복사한다. `bias_warnings` 판단 방식은 미정(O-05).

## 5. 지원·동의·이력서 API

지원은 `지원 만들기 → 동의 → 이력서 올리기(선택) → 이력서 확정` 순서다. 확정하는 순간 상태가 `ready`가 되고, 서버는 면접 질문 10개 만들기를 뒤에서 시작한다(QST-03).

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `POST /applications` | 🔒 | 지원 만들기(`drafting`). 작성 중인 지원이 있으면 그것을 돌려줌 | S-A02 |
| `POST /applications/{id}/consent` | 🔒 | 필수 동의 5개 기록 | S-A03 |
| `POST /applications/{id}/consent/withdraw` ➕ | 🔒 | 동의 철회: 진행 중 응시 취소, 이 지원의 데이터 즉시 삭제 | S-A13 계정 |
| `POST /applications/{id}/resume/upload` ➕ | 🔒 | 이력서 파일 올리기 → 내용 뽑기 시작(OCR 포함) | S-A04 |
| `GET /applications/{id}/resume/extract-status` ➕ | 🔒 | 뽑기 진행 단계·예상 시간·결과·실패 이유 (2초마다 조회) | S-A04 |
| `DELETE /applications/{id}/resume/upload` ➕ | 🔒 | 처리 중 취소, 올린 파일 삭제 | S-A04 \[취소\] |
| `POST /applications/{id}/resume` | 🔒 | 이력서 확정(사본 저장). 이후 수정 불가 | S-A05 |
| `GET /me/resume` ➕ | 🔒 | 마이페이지 등록 이력서(1개) | S-A04, S-A13 |
| `PUT /me/resume` ➕ | 🔒 | 등록 이력서 저장(통째로) | S-A13 이력서 |
| `DELETE /me/resume` ➕ | 🔒 | 등록 이력서 삭제 | S-A13 이력서 |
| `POST /me/resume/upload` ➕ | 🔒 | 등록 이력서용 파일 올리기(같은 추출 과정) | S-A13 이력서 |
| `GET /me/resume/extract-status` ➕ | 🔒 | 위 추출 진행 상태 | S-A13 이력서 |

### POST /applications

요청 — `{ "posting_id": "3a7e…" }`

응답 201 (작성 중인 지원이 있으면 200) — `{ "application_id": "7b2e…", "attempt_status": "drafting", "next_step": "consent" }`. `next_step`은 `consent` · `resume` 중 하나로, 이어서 쓸 때 어느 화면으로 보낼지 알려 준다.

| 오류 | code | 조건 |
| --- | --- | --- |
| 409 | `ALREADY_APPLIED` | 같은 공고에 `drafting`이 아닌 지원이 있음(1회 응시) |
| 409 | `POSTING_CLOSED` | 접수 기간이 아님 |
| 400 | `JOB_NOT_OPEN` | 준비 중 직무 공고 |
| 403 | `EMAIL_NOT_VERIFIED` | `email_verification_required=true`인데 미인증(지금은 꺼져 있음) |

### POST /applications/{id}/consent

```json
{
  "consent_version": "v1.3",
  "items": {
    "data_collection": true,
    "ai_decision": true,
    "share_with_admin": true,
    "retention": true,
    "exam_rules": true
  }
}
```

응답 201 — `{ "consent_id": "…", "agreed_at": "…" }`. 하나라도 `false`면 400 `AGREEMENT_REQUIRED`. 문구 버전은 서버의 현재 버전과 같아야 하고, 다르면 409 `CONSENT_VERSION_CHANGED`(화면이 새 문구를 다시 보여 줌).

### POST /applications/{id}/resume/upload

`multipart/form-data`의 `file` 하나. PDF·JPG·PNG만 받고, 확장자와 파일 앞부분의 실제 형식을 함께 본다.

응답 202

```json
{
  "upload_id": "c41d…",
  "file_name": "이력서_홍길동.pdf",
  "pages": { "text": 1, "scanned": 8 },
  "estimated_seconds": 85
}
```

`estimated_seconds` = 글자 있는 쪽 수 × 0.5 + 스캔 쪽 수 × `ocr_sec_per_page` + `resume_llm_sec`(TRD 7-6 ①). 오류: 400 `UNSUPPORTED_FILE`(형식 불일치 포함) · 413 `FILE_TOO_LARGE`. 화면은 둘 다 "이력서를 불러오는 데 실패했습니다" 모달로 보여 준다(D-05).

### GET /applications/{id}/resume/extract-status

```json
{
  "upload_id": "c41d…",
  "stage": "ocr",
  "estimated_seconds": 85,
  "elapsed_seconds": 18,
  "method": null,
  "result": null,
  "ocr_confidence": null,
  "ocr_low_conf_fields": [],
  "fail_reason": null
}
```

- `stage`: `uploaded` → `checking` → `ocr`(스캔 쪽·사진이 있을 때만) → `structuring` → `done` | `failed`.
- `done`이면 `method`(`text` · `ocr` · `mixed`), `result`(아래 이력서 객체), `ocr_confidence`(평균, 예 `"0.86"`), `ocr_low_conf_fields`(예 `["career[0].company"]`, 신뢰도 0.80 미만)가 채워진다.
- `failed`면 `fail_reason`: `TIMEOUT`(예상의 2배, 최소 60초) · `NO_TEXT` · `LOW_CONFIDENCE`(평균 0.5 미만) · `INTERNAL`. 화면은 "직접 입력하시겠어요?" 모달을 띄운다(APP-04).

### POST /applications/{id}/resume (확정)

```json
{
  "method": "upload",
  "upload_id": "c41d…",
  "resume": {
    "education": [ { "school": "○○대학교", "major": "컴퓨터공학", "degree": "bachelor", "start": "2018-03", "end": "2024-02" } ],
    "career": [ { "company": "△△", "role": "ML 엔지니어", "start": "2024-03", "end": null, "description": "…" } ],
    "projects": [ { "name": "…", "period": "2025-01~2025-06", "description": "…", "skills": ["PyTorch"] } ],
    "skills": ["Python", "PyTorch", "SQL"],
    "certificates": [ { "name": "정보처리기사", "acquired": "2023-11" } ],
    "cover_letter": null
  },
  "edited_fields": ["career[0].company"]
}
```

- `method`: `upload` · `manual` · `saved`(등록본). `upload`일 때만 `upload_id`가 필요하다.
- `edited_fields`는 지원자가 고친 칸으로, 관리자 이력서 탭에 표시한다.
- 응답 200 — `{ "attempt_status": "ready", "close_at": "…" }`. 필수 칸이 빠지면 422 `RESUME_REQUIRED_MISSING`(`message`에 칸 이름), 이미 확정했으면 409 `RESUME_LOCKED`.
- `/me/resume`의 `GET`·`PUT`은 같은 `resume` 객체를 주고받는다. 등록본을 바꿔도 이미 확정한 지원의 사본은 바뀌지 않는다(APP-05).

## 6. 면접 시작·점검 API

면접은 `장치 점검 → 시작(입장 관리 통과) → WebSocket 연결` 순서로 들어간다. 응시 횟수는 시작 API가 아니라 첫 질문이 나갈 때(`attempt_consumed_at`) 쓴다.

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `POST /applications/{id}/precheck` | 🔒 | 장치·인터넷·AI 서버 점검, 통과 표시 발급 | S-A06, S-A15 |
| `GET /applications/{id}/interview/state` ➕ | 🔒 | 지금 응시 상태, 질문 준비 정도, 장애 판단 결과, 이어가기 기한 | `/exam` 가드, S-A15 |
| `POST /applications/{id}/interview/start` | 🔒 | 입장 관리(N) 통과 시 면접 세션 발급 | S-A07 |
| `POST /applications/{id}/interview/resume` ➕ | 🔒 | 장애 중단 면접 이어가기(N 다시 안 봄) | S-A15 |
| `POST /interview/{sid}/events` ➕ | 세션 | 창 닫기·이탈 신호를 `sendBeacon`으로 남김 | S-A09, S-A11 |
| `POST /interview/{sid}/recording` ➕ | 세션 | 카메라 녹화 조각 올리기 | S-A09, S-A11 |

### POST /applications/{id}/precheck

요청

```json
{
  "browser": { "name": "chrome", "version": "141" },
  "devices": { "mic": true, "camera": true, "speaker": true },
  "network_mbps": "12.4"
}
```

응답 200

```json
{
  "passed": true,
  "checks": { "browser": "ok", "mic": "ok", "camera": "ok", "speaker": "ok", "network": "ok", "ai_server": "ok" },
  "valid_until": "2026-11-03T10:15:00+09:00"
}
```

- 마이크·카메라·스피커는 화면이 확인한 결과를 그대로 기록하고, 서버는 브라우저(Chrome·Edge 최신 2개 버전)·인터넷·AI 서버(llm-rt·stt 상태)를 직접 본다.
- 각 항목은 `ok` · `fail`. 하나라도 `fail`이면 `passed: false`. 통과 표시는 15분 동안 유효하며, 시작·이어가기 API가 이를 확인한다.

### GET /applications/{id}/interview/state

```json
{
  "attempt_status": "interrupted",
  "phase": "interview",
  "current_seq": 4,
  "questions": { "ready": true, "composed": 10, "total": 10 },
  "interruption": {
    "judged": "fault",
    "reason": "heartbeat_timeout",
    "at": "2026-11-03T10:07:42+09:00"
  },
  "resume_until": "2026-11-10T18:00:00+09:00",
  "server_time": "2026-11-03T10:09:01+09:00"
}
```

- `phase`: `interview` · `coding` (어느 화면으로 이어 갈지).
- `interruption.judged`: `fault`(장애 → 이어가기 가능) · `not_fault`(장애 아님 → `ended`). 서버는 끊긴 뒤 15초가 되는 순간 판단하므로(TRD 7-2), 이 API는 그 결과를 읽기만 한다.
- `reason`: `heartbeat_timeout` · `ws_abnormal_close` · `offline` · `device_error` · `upload_failed` · `server_error` · `unknown` · `unload` · `permission_revoked`.
- `resume_until`은 공고 마감 시각이다(RES-03). `server_time`으로 화면 카운트다운을 맞춘다.

### POST /applications/{id}/interview/start

요청 본문 없음. 응답 200

```json
{
  "session_id": "5d1c2f0e-…",
  "ws_url": "ws://127.0.0.1:8000/ws/interview/5d1c2f0e-…",
  "question_total": 10,
  "questions_ready": false
}
```

- `questions_ready: false`면 화면은 S-A08 '질문 준비 중'을 띄우고, WebSocket `preparing` 메시지를 받다가 첫 `question`이 오면 면접을 시작한다. 기다림 상한은 60초(`question_compose_budget_sec`)이고, 넘으면 서버가 질문 DB·예비 질문으로 채운다.
- 시작 성공 시 `interview:active`를 1 올리고 `holds_seat=true`. 상태는 첫 질문이 나갈 때 `interviewing`이 된다.

| 오류 | code | 조건 |
| --- | --- | --- |
| 429 | `CAPACITY_FULL` | 동시 면접 N이 찼거나 자동 보호(그래픽카드 메모리 92%·다음 질문 시간)가 켜짐. 응시 횟수 안 씀. 화면은 '다시 시도'를 10초 막는다 |
| 409 | `PRECHECK_REQUIRED` | 점검 통과 표시 없음 또는 15분 지남 |
| 409 | `NOT_READY` | 상태가 `ready`가 아님(이력서 미확정 등) |
| 409 | `ALREADY_ATTEMPTED` | 이미 응시함(1회 제한) |
| 409 | `POSTING_CLOSED` | 마감 시각이 지남(INT-10) |

### POST /applications/{id}/interview/resume

요청 본문 없음. 응답은 start와 같은 모양에 `"phase"`와 `"resume_seq"`(다시 낼 질문 번호)를 더한다.

- N을 다시 확인하지 않는다. 자리를 이미 돌려줬으면(끊긴 뒤 10분) 1을 다시 더한다(TRD 5-3).
- 면접은 끊긴 질문을 처음부터 새 답변 시간으로, 코딩테스트는 마지막 자동 저장 코드와 남은 시간으로 이어 간다.
- 오류: 409 `PRECHECK_REQUIRED` · 409 `NOT_INTERRUPTED`(상태가 `interrupted`가 아님) · 409 `POSTING_CLOSED`.

### POST /interview/{sid}/events

탭 닫기·새로고침·뒤로 가기 때 `beforeunload`·`pagehide`에서 `navigator.sendBeacon`으로 보낸다. 창이 닫혀도 전송되도록 본문은 문자열 JSON이다.

```json
{ "type": "unload", "at": "2026-11-03T10:07:30+09:00", "detail": "pagehide" }
```

- `type`: `unload` · `permission`(마이크·카메라 권한 거부). 연결이 살아 있을 때의 이탈·장치 신호는 WebSocket으로 보낸다(7장).
- 응답 204. 서버는 `session_events`에 남기고, 끊기기 직전 30초(`unload_lookback_sec`) 안에 이 기록이 있으면 '장애 아님'으로 본다.

### POST /interview/{sid}/recording

`multipart/form-data` — `chunk_no`(0부터), `phase`(`interview` · `coding`), `file`(WebM 조각). 응답 204. 실패하면 화면이 1·2·4초 간격으로 3번(`upload_retry_limit`) 다시 올리고, 그래도 안 되면 장애 후보가 된다. 답변 음성은 이 API가 아니라 WebRTC로 보낸다.

## 7. 면접 실시간 통신 (WebSocket · WebRTC)

면접 중에는 연결 두 개를 열어 둔다. 답변 음성은 WebRTC로, 그 밖의 모든 것(질문·자막·진행 상태·연결 확인)은 WebSocket으로 주고받는다. 코딩테스트 중에도 같은 WebSocket을 유지해 heartbeat와 이탈 신호를 보낸다.

### 7-1. 연결

- 주소: `ws://127.0.0.1:8000/ws/interview/{session_id}`. 로그인 쿠키와 `Origin`(`http://127.0.0.1:8080`)을 확인하고, 아니면 종료 번호 4401로 끊는다.
- 모든 메시지는 `type` 칸이 있는 JSON이다.
- WebRTC는 같은 PC 안이라 STUN·TURN을 쓰지 않는다. aiortc는 UDP 40000\~40019를 쓰고, 브라우저에 알리는 주소는 `127.0.0.1`이다.
- 정상 종료는 1000. 서버가 면접을 끝낼 때는 `terminate`를 보낸 뒤 1000으로 닫는다. 1000·1001로 닫혀도 창 닫기 기록이 없으면 장애로 본다(TRD 7-2).

### 7-2. 질문 하나가 오가는 순서

1. 연결 직후 화면이 `rtc_offer`를 보내고, 서버가 `rtc_answer`로 답한다. 후보 주소는 양쪽이 `rtc_ice`로 주고받는다.
2. 질문이 다 안 만들어졌으면 서버가 `preparing`을 1초마다 보낸다.
3. 서버가 `question`을 보낸다. 화면은 질문 음성을 틀고(없으면 자막만), 3-2-1 뒤 녹음을 시작한다.
4. 답변 중 서버가 `caption`(임시 자막)을 보낸다.
5. 화면이 `answer_done`을 보내거나 답변 시간이 끝난다.
6. 서버가 최종 글자(`caption`, `is_final: true`)를 보내고, 꼬리질문을 판단한 뒤 다음 `question`을 보낸다. 목표는 `next_question_target_sec` 안이다.
7. 10번째 기본 질문(꼬리질문 포함)이 끝나면 `phase`로 코딩테스트에 넘어간다. 코딩테스트를 쓰지 않는 공고면 `terminate`(`completed`)로 끝난다.

### 7-3. 화면 → 서버 메시지

| type | 칸 | 보내는 때 |
| --- | --- | --- |
| `rtc_offer` | `sdp` | 연결 직후 |
| `rtc_ice` | `candidate` | 후보 주소가 생길 때 |
| `heartbeat` | `sent_at` | 5초마다. 3번 연속(15초) 안 오면 장애 판단 |
| `answer_done` | `seq`, `answer_no` | 지원자가 \[답변 완료\]를 누름 |
| `visibility` | `state`(`hidden`), `at` | 탭이 숨겨짐 |
| `blur` | `at`, `duration_ms` | 창이 포커스를 잃고 2초(`blur_threshold_sec`) 넘게 안 돌아옴. 2초 미만은 보내지 않음 |
| `fullscreen` | `state`(`exit`), `at` | 전체 화면 해제 |
| `navigate` | `at`, `href` | 다른 주소로 가려 함 |
| `device_error` | `device`(`mic` · `camera`), `kind`(`ended` · `silent`) | 트랙 `ended` 이벤트, 마이크 입력 10초 넘게 0 |
| `warn_ack` | `seq` | 1번째 이탈 경고 창에서 \[확인\]을 누름 |
| `unload` | `at`, `detail` | 창 닫기·새로고침. 연결이 살아 있으면 여기로, 아니면 `POST /interview/{sid}/events`로 |

### 7-4. 서버 → 화면 메시지

| type | 칸 | 보내는 때 |
| --- | --- | --- |
| `rtc_answer` | `sdp` | `rtc_offer`에 대한 답 |
| `rtc_ice` | `candidate` | 서버 쪽 후보 주소 |
| `preparing` | `composed`, `total`, `elapsed_sec` | 질문 준비 중(최대 60초) |
| `question` | `seq`, `kind`(`base` · `follow_up`), `text`, `audio_url`(없으면 `null`), `answer_sec`, `progress`(`"3 / 10"`) | 질문을 낼 때. 면접 중 LLM·TTS가 늦으면 `audio_url: null` |
| `caption` | `seq`, `answer_no`, `chunk_no`, `text`, `is_final` | `caption_interval_sec`마다 임시 자막, 답변 끝에 최종 글자. 화면은 더 작은 `chunk_no`나 지난 `seq`를 버린다 |
| `warn` | `count`(1), `seq`, `remaining_sec`, `phase` | 1번째 이탈. 시간을 멈추고 경고 창을 띄운다 |
| `resume` | `seq`, `remaining_sec`, `phase` | `warn_ack` 뒤 또는 장애 이어가기 뒤 다시 시작 |
| `paused` | `reason`, `resume_until` | 장애로 멈춤(연결이 살아 있을 때만 도착). 화면은 OV-01 |
| `phase` | `to`(`coding`), `coding` → 8장 문제 목록 | AI 면접이 끝남 |
| `terminate` | `reason`, `message` | 면접이 끝남. `reason`: `completed` · `cancelled`(2번째 이탈) · `closed`(마감) · `ended`(장애 아님) · `withdrawn`(동의 철회) |
| `error` | `code`, `message` | 형식이 틀린 메시지 등. 연결은 유지 |

### 7-5. 메시지 예

```json
{ "type": "question", "seq": 3, "kind": "base",
  "text": "최근 프로젝트에서 모델 성능을 높인 방법을 설명해 주세요.",
  "audio_url": "/media/questions/5d1c…/3.wav", "answer_sec": 90, "progress": "3 / 10" }

{ "type": "caption", "seq": 3, "answer_no": 1, "chunk_no": 7,
  "text": "저는 이전 프로젝트에서 데이터 불균형을", "is_final": false }

{ "type": "warn", "count": 1, "seq": 3, "remaining_sec": 47, "phase": "interview" }

{ "type": "terminate", "reason": "closed", "message": "마감이 되어 면접을 종료합니다." }
```

- 1번째 이탈 뒤에는 같은 질문을 멈춘 시점의 남은 시간부터 이어 답하고, 이탈 전후 음성을 합쳐 하나의 답변으로 채점한다(TRD 7-3). PRD v1.3 CHT-02(답변 시간 새로 부여)와 다르므로 12장에 적었다.
- 장애로 판단된 구간에 생긴 이탈 신호는 세지 않는다. 면접과 코딩테스트의 이탈 횟수는 합쳐 센다.

## 8. 코딩테스트 API

코딩테스트는 면접과 같은 세션(`sid`)을 쓴다. 2문제를 합쳐 60분이고, 시간은 서버가 잰다. 시간이 끝나면 서버가 마지막 저장 코드를 자동 제출한다.

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `GET /coding/{sid}` ➕ | 세션 | 문제 2개, 남은 시간, 마지막 저장 코드 | S-A10, S-A11 |
| `POST /coding/{sid}/start` ➕ | 세션 | 코딩테스트 시작(타이머 시작). 안내 화면에서 60초 뒤 자동으로도 부름 | S-A10 |
| `POST /coding/{sid}/run` | 세션 | 공개 예시로 실행 | S-A11 |
| `POST /coding/{sid}/autosave` | 세션 | 30초마다 자동 저장 | S-A11 |
| `POST /coding/{sid}/submit` | 세션 | 최종 제출 → 응시 완료 | S-A11 |

### GET /coding/{sid}

```json
{
  "started": true,
  "remaining_sec": 2841,
  "problems": [
    {
      "problem_no": 1,
      "title": "구간 합 최댓값",
      "statement": "…",
      "signature": "def solution(nums: list[int], k: int) -> int:",
      "examples": [ { "input": "[1, 3, 2], 2", "output": "5" } ],
      "limits": { "time_sec": 10, "memory_mb": 256 },
      "saved_code": "def solution(nums, k):\n    ...",
      "saved_at": "2026-11-03T10:31:30+09:00"
    }
  ]
}
```

숨겨진 테스트와 정답은 어떤 응답에도 넣지 않는다(TRD 9장). 장애로 끊긴 시간은 빼지 않으므로, 이어가기 뒤에도 `remaining_sec`는 서버 시각으로 계산한 값이다.

### POST /coding/{sid}/run

요청 — `{ "problem_no": 1, "code": "…" }`

응답 200

```json
{
  "results": [ { "example_no": 1, "passed": true, "stdout": "5", "stderr": "", "time_ms": 41 } ],
  "status": "ok"
}
```

`status`: `ok` · `timeout`(10초) · `memory_exceeded`(256MB) · `runtime_error` · `syntax_error`. 매번 새 실행 상자(인터넷 없음)에서 돌린다. 실행 상자가 바쁘면 503 `SANDBOX_BUSY`.

### POST /coding/{sid}/autosave

요청 — `{ "problem_no": 1, "code": "…", "paste_blocked": 2 }` → 204. `paste_blocked`는 마지막 저장 뒤 새로 막은 외부 붙여넣기 시도 수로, 서버가 누적한다. 실패하면 화면이 3번 다시 보내고, 그래도 안 되면 장애 후보다(`upload_retry_limit`).

### POST /coding/{sid}/submit

요청 — `{ "codes": [ { "problem_no": 1, "code": "…" }, { "problem_no": 2, "code": "…" } ] }`

응답 200 — `{ "attempt_status": "completed", "release_at": "2026-11-11T10:00:00+09:00" }`. 서버는 숨겨진 테스트로 채점하고 최종 채점·AI 결정 작업을 넣는다. 시간이 지났으면 409 `TIME_OVER`(서버가 이미 자동 제출함). 화면은 이어서 S-A12로 간다.

## 9. 마이페이지·결과·알림 API

지원자에게는 공개 시각 이후에만 결과를 주고, 점수·근거·이유는 어느 응답에도 넣지 않는다(RPT-03, AC-16).

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `GET /me/applications` ➕ | 🔒 | 내 지원 내역과 응시·공개 상태 | S-A13 지원 내역·결과 |
| `GET /me/results/{application_id}` | 🔒 | 공개된 결과(`public_view`). 공개 전 404 | S-A14 |
| `GET /notifications` ➕ | 🔒🛡 | 사이트 알림 최근 20건, 안 읽은 수 | 머리글 🔔 |
| `POST /notifications/{id}/read` ➕ | 🔒🛡 | 읽음 표시 | 머리글 🔔 |

### GET /me/applications

```json
{
  "items": [
    {
      "application_id": "7b2e…",
      "posting": { "posting_id": "3a7e…", "title": "AI/AX 엔지니어", "close_at": "2026-11-10T18:00:00+09:00" },
      "attempt_status": "interrupted",
      "next_action": "resume",
      "resume_until": "2026-11-10T18:00:00+09:00",
      "release_status": "scheduled",
      "release_at": "2026-11-11T10:00:00+09:00",
      "result_until": null,
      "applied_at": "2026-11-03T09:40:00+09:00"
    }
  ],
  "total": 1, "page": 1, "size": 20
}
```

- `next_action`: `continue_draft` · `precheck` · `resume` · `view_result` · `null`. 화면은 이 값으로 카드 버튼을 그린다(화면정의서 §3.4).
- 정렬은 서버가 한다: `interrupted` → `ready` → `drafting` → 최신 지원순.
- `release_status`가 `delayed`면 "다음 날 10:00 공개"를 보여 준다(EVL-08). `result_until`은 공개 후 열람 끝 시각(공개 + 7일).
- 보관 기간이 지나 지워진 지원은 목록에서 빠진다.

### GET /me/results/{application\_id}

```json
{
  "application_id": "7b2e…",
  "result": "fail",
  "attempt_status": "completed",
  "status_reason": null,
  "strengths": ["직무 개념을 정확히 설명함"],
  "improvements": ["경험을 수치와 함께 말하면 좋음"],
  "ai_generated": true,
  "contact_email": "recruit@example.com",
  "result_until": "2026-11-18T10:00:00+09:00",
  "status": "released"
}
```

- `result`: `pass` · `fail`. `strengths`·`improvements`는 각 최대 3개.
- `status_reason`: 응시 취소면 `cheating_cancelled`, 마감 종료면 `closed`, 장애 아님 종료면 `ended`.
- 공개 전·남의 지원·보관 만료는 모두 404 `NOT_FOUND`다. 화면은 `GET /me/applications`의 `release_at`으로 "아직 공개되지 않았습니다"와 "보관 기간이 지났습니다"를 구분한다.

### GET /notifications

```json
{
  "unread": 2,
  "items": [
    { "notification_id": "…", "kind": "result_released", "title": "1차 결과가 공개되었습니다",
      "link": "/me/results/7b2e…", "read": false, "created_at": "…" }
  ]
}
```

`kind` — 지원자: `result_released` · `deadline_reminder` · `result_expiring`. 관리자: `new_application` · `posting_closing` · `data_purge_soon` · `needs_review`. 메일 보내기가 실패해도 사이트 알림은 항상 남는다(TRD 10장).

## 10. 관리자 API

관리자는 결과와 결정을 읽기만 한다. 바꿀 수 있는 것은 메모·설정·다시 돌리기 세 가지다(EVL-05, AC-14). 주소의 `{aid}`는 지원 1건의 `application_id`다.

| 방식 · 주소 | 권한 | 하는 일 | 화면 |
| --- | --- | --- | --- |
| `GET /admin/postings/{id}/applicants` | 🛡 | 공고별 지원자 목록·점수·판정 | S-R05 |
| `GET /admin/applicants/{aid}` | 🛡 | 지원자 상세: 리포트·질문별 답변·이력서·코딩·기록·메모 | S-R06 |
| `GET /admin/applicants/compare?ids=` | 🛡 | 2\~3명 비교 | S-R07 |
| `GET /admin/applicants/{aid}/media` ➕ | 🛡 | 답변 음성·녹화 재생용 5분짜리 주소. 감사 로그 | S-R06 |
| `POST /admin/applicants/{aid}/memo` | 🛡 | 내부 메모 쓰기 | S-R06 |
| `PATCH /admin/memos/{memo_id}` ➕ | 🛡 | 메모 고치기 | S-R06 |
| `DELETE /admin/memos/{memo_id}` ➕ | 🛡 | 메모 지우기 | S-R06 |
| `GET /admin/postings/{id}/passed` ➕ | 🛡 | 1차 합격자 목록·연락처 | S-R08 |
| `POST /admin/analysis/{aid}/reprocess` | 🛡 | '확인 필요' 건 다시 돌리기 | S-R05, S-R06, S-R09 |
| `GET /admin/stats` ➕ | 🛡 | 집계 통계(개인 정보 없음) | S-R09 |
| `GET /admin/ops-status` ➕ | 🛡 | 확인 필요 목록, 메일 실패, 예비 질문 수 | S-R09 |
| `GET /admin/settings` ➕ | 🛡 | 현재 N, 진행 중 수, 최근 변경 | S-R10 |
| `PUT /admin/settings/{key}` | 🛡 | 설정 바꾸기(감사 로그) | S-R10 |

### GET /admin/postings/{id}/applicants

쿼리 — `result`(`undecided` · `pass` · `fail`) · `attempt_status` · `analysis_status` · `sort`(`total_score` · `interview_score` · `coding_score` · `applied_at`, 앞에 `-`면 내림차순) · `page` · `size`

```json
{
  "items": [
    {
      "application_id": "7b2e…",
      "name": "홍길동",
      "attempt_status": "completed",
      "analysis_status": "decided",
      "interview_score": "72.50",
      "coding_score": "60.00",
      "total_score": "68.75",
      "decision": "fail",
      "records": { "resumed": 1, "exits": 0, "ended_by_close": false },
      "purge_at": "2026-11-18T10:00:00+09:00"
    }
  ],
  "has_coding_test": true,
  "total": 24, "page": 1, "size": 20
}
```

분석 전 행은 점수·`decision`이 `null`이다. `not_taken_closed` 행도 `null`이며 통계에서 빠진다.

### GET /admin/applicants/{aid}

```json
{
  "application_id": "7b2e…",
  "applicant": { "name": "홍길동", "email": "hong@example.com", "phone": "010-1234-5678" },
  "criteria_version": 1,
  "result": {
    "decision": "fail",
    "reason": "직무 이해는 좋지만 구체적인 근거가 부족했다. …",
    "quotes": ["먼저 원인을 확인하고 그다음 방법을 골랐습니다"],
    "item_scores": { "expertise": "80.00", "logic": "65.00", "relevance": "70.00", "delivery": "75.00", "culture_fit": "60.00" },
    "interview_score": "72.50", "coding_score": "60.00", "total_score": "68.75", "pass_score": "70.00",
    "strengths": ["…"], "improvements": ["…"], "summary": "…",
    "model_version": "qwen3-8b-q4_k_m@sha256:…", "prompt_version": "eval-v3"
  },
  "answers": [
    {
      "seq": 3, "kind": "base", "category": "직무", "question": "…", "replaced": false,
      "transcript": "…",
      "scores": [ { "item": "expertise", "score": 4, "quote": "클래스 가중치를 조정해 재현율을 높였습니다", "quote_verified": true } ],
      "follow_ups": []
    }
  ],
  "highlights": [ { "seq": 3, "kind": "good", "start_ms": 12400, "end_ms": 19800, "item": "expertise" } ],
  "resume": { "snapshot": { }, "method": "ocr", "ocr_confidence": "0.86", "edited_fields": ["career[0].company"] },
  "coding": { "score": "60.00", "problems": [ { "problem_no": 1, "passed": 8, "total": 10, "code": "…", "paste_blocked": 2, "feedback": "…" } ] },
  "events": [ { "kind": "disconnect", "at": "…", "duration_sec": 42, "judged": "fault", "reason": "heartbeat_timeout" } ],
  "recording_until": "2026-11-06T10:40:00+09:00",
  "memos": [ { "memo_id": "…", "body": "…", "created_at": "…", "updated_at": null } ]
}
```

- 분석이 끝나지 않았으면 `result`가 `null`이고 `analysis_status`를 함께 준다. 이력서·기록은 그대로 보인다.
- `highlights.kind`: `good`(4\~5점) · `deducted`(1\~2점). 각 최대 3개(TRD 8-4).
- 코딩 `feedback`은 점수에 반영하지 않는다. 숨겨진 테스트 내용은 넣지 않는다.
- 이 API를 부르면 '결과 보기' 감사 로그가 남는다.

### GET /admin/applicants/{aid}/media

쿼리 — `kind`(`answer` · `recording`), `seq`(`answer`일 때). 응답 — `{ "url": "…", "expires_at": "…" }`(MinIO 5분 주소). 녹화는 응시 끝 + 3일, 답변 음성은 공개 + 7일이 지나면 410 `MEDIA_EXPIRED`. 부를 때마다 '녹화 다시 보기' 감사 로그가 남는다.

### 메모

- `POST /admin/applicants/{aid}/memo` — `{ "body": "…" }` → 201 메모 객체.
- `PATCH /admin/memos/{memo_id}` — `{ "body": "…" }` → 200. `DELETE` → 204.
- 1\~2000자. 지원자 응답에는 절대 넣지 않는다(ADM-05).

### POST /admin/analysis/{aid}/reprocess

요청 본문 없음. 응답 202 — `{ "analysis_status": "running" }`. 같은 응시 자료로 채점·결정을 처음부터 다시 돌리고, 판정을 사람이 고치지는 않는다. `needs_review`가 아니면 409 `NOT_NEEDS_REVIEW`.

### GET /admin/stats · /ops-status

- `stats` 쿼리 — `posting_id`(없으면 전체). 응답은 지원 수, 응시 완료율, 합격률, 항목별 평균 점수, 점수 분포(10점 구간), 장애·이탈 건수. 이름·연락처는 없다.
- `ops-status` 응답:

```json
{
  "needs_review": [ { "application_id": "…", "posting_title": "…", "since": "…", "reason": "rejudge_failed" } ],
  "mail_failed": { "count": 3, "items": [ { "to": "ho***@example.com", "kind": "result_released", "reason": "550 no such user", "at": "…" } ] },
  "mail_deferred_today": 0,
  "reserve_questions": [ { "category": "경험·프로젝트", "count": 4, "below_minimum": true } ]
}
```

### GET /admin/settings · PUT /admin/settings/{key}

`GET` 응답

```json
{
  "max_concurrent_interviews": 2,
  "active_interviews": 1,
  "admission_guard": "open",
  "history": [ { "key": "max_concurrent_interviews", "from": 3, "to": 2, "at": "…" } ]
}
```

- `PUT` 요청 — `{ "value": 3 }` → 200 `{ "key": "max_concurrent_interviews", "value": 3 }`. 다음 시작 요청부터 바로 적용되고, 이미 진행 중인 면접은 끊지 않는다(TRD 5-3).
- 화면에서 바꾸는 키는 `max_concurrent_interviews` 하나다. 그 밖의 키(TRD 14장)는 서버가 받더라도 화면에 두지 않는다. 없는 키는 404, 범위 밖(1 미만)은 400 `INVALID_SETTING`.
- `admission_guard`: `open` · `blocked`(자동 보호로 새 입장을 막는 중).

## 11. 오류 코드 목록

화면은 아래 `code`로 안내 문구를 고른다. 문구는 화면정의서의 문구를 기준으로 하고, 서버의 `message`는 개발자 확인용이다.

| 분야 | code | HTTP | 화면 안내 |
| --- | --- | --- | --- |
| 공통 | `UNAUTHORIZED` | 401 | 로그인 화면으로(현재 주소를 `next`로) |
| 공통 | `FORBIDDEN` | 403 | 접근할 수 없는 페이지입니다 |
| 공통 | `NOT_FOUND` | 404 | 찾을 수 없는 정보입니다 |
| 공통 | `VALIDATION_ERROR` | 422 | 입력값을 확인해 주세요 (`message`에 칸 이름) |
| 공통 | `SERVER_ERROR` | 500 | 잠시 후 다시 시도해 주세요 |
| 공통 | `AI_UNAVAILABLE` | 503 | AI 서버를 확인하지 못했습니다 |
| 계정 | `PASSWORD_POLICY` | 400 | 비밀번호 규칙 체크리스트에 ✕ 표시 |
| 계정 | `AGREEMENT_REQUIRED` | 400 | 필수 항목에 모두 동의해 주세요 |
| 계정 | `EMAIL_TAKEN` | 409 | 이미 가입된 이메일입니다 |
| 계정 | `INVALID_CREDENTIALS` | 401 | 이메일 또는 비밀번호가 올바르지 않습니다 |
| 계정 | `ACCOUNT_LOCKED` | 423 | 5회 실패로 10분간 로그인할 수 없습니다 |
| 계정 | `TOKEN_EXPIRED` | 400 | 링크가 만료되었습니다 |
| 계정 | `WRONG_PASSWORD` | 400 | 현재 비밀번호가 맞지 않습니다 |
| 계정 | `SAME_AS_OLD` | 400 | 이전과 다른 비밀번호를 입력해 주세요 |
| 계정 | `PASSWORD_CHANGE_REQUIRED` | 403 | 초기 비밀번호 변경 화면(S-C04)으로 |
| 계정 | `EMAIL_NOT_VERIFIED` | 403 | 이메일 인증 후 지원할 수 있습니다 |
| 공고 | `CRITERIA_LOCKED` | 409 | 🔒 첫 응시가 시작되어 평가 기준을 바꿀 수 없습니다 |
| 공고 | `WEIGHT_SUM` | 400 | 합계가 100%가 되어야 합니다 |
| 공고 | `POSTING_INCOMPLETE` | 400 | 빈 필수 칸으로 스크롤 |
| 공고 | `JOB_NOT_OPEN` | 400 | 아직 준비 중인 직무입니다 |
| 공고 | `HAS_APPLICANTS` | 409 | 지원자가 있는 공고는 삭제할 수 없고 마감만 가능합니다 |
| 공고 | `IMPORT_SOURCE_FAILED` | 502 | 공고를 불러오지 못했습니다 |
| 지원 | `ALREADY_APPLIED` | 409 | 이미 지원한 공고입니다 |
| 지원 | `POSTING_CLOSED` | 409 | 마감이 되어 면접을 진행할 수 없습니다 |
| 지원 | `CONSENT_VERSION_CHANGED` | 409 | 동의 문구가 바뀌었습니다. 다시 확인해 주세요 |
| 이력서 | `UNSUPPORTED_FILE` | 400 | 이력서를 불러오는 데 실패했습니다 (실패 모달) |
| 이력서 | `FILE_TOO_LARGE` | 413 | 이력서를 불러오는 데 실패했습니다 (실패 모달, 용량은 말하지 않음) |
| 이력서 | `RESUME_REQUIRED_MISSING` | 422 | 빠진 항목으로 스크롤 + 빨간 테두리 |
| 이력서 | `RESUME_LOCKED` | 409 | 제출한 이력서는 수정할 수 없습니다 |
| 면접 | `CAPACITY_FULL` | 429 | 동시 면접 인원이 찼습니다. 응시 기회는 그대로입니다 + \[다시 시도\] |
| 면접 | `PRECHECK_REQUIRED` | 409 | 환경 점검(S-A06)으로 |
| 면접 | `NOT_READY` | 409 | 지원 절차를 마쳐 주세요 |
| 면접 | `ALREADY_ATTEMPTED` | 409 | 이미 응시한 면접입니다 |
| 면접 | `NOT_INTERRUPTED` | 409 | 이어갈 수 있는 면접이 없습니다 |
| 코딩 | `SANDBOX_BUSY` | 503 | 잠시 후 다시 실행해 주세요 |
| 코딩 | `TIME_OVER` | 409 | 시간이 끝나 자동 제출되었습니다 |
| 관리자 | `MEDIA_EXPIRED` | 410 | 보관 기간 만료 |
| 관리자 | `NOT_NEEDS_REVIEW` | 409 | 재처리 대상이 아닙니다 |
| 관리자 | `INVALID_SETTING` | 400 | 1 이상의 정수를 입력해 주세요 |

## 12. 확인 필요 사항

문서끼리 말이 다르거나 어느 문서에도 없는 것은 아래처럼 임시로 정했다. 팀이 정하면 이 문서와 해당 문서를 같이 고친다.

| # | 항목 | 문서별 내용 | 이 문서의 선택 | 고칠 문서 |
| --- | --- | --- | --- | --- |
| A-01 | 기준 PRD 버전 | TRD v1.5는 PRD v1.6 기준, 팀에 공유된 PRD는 v1.3 | TRD v1.5를 우선하고, PRD v1.3과 다른 곳은 이 표에 적음 | PRD v1.6 공유 후 다시 대조 |
| A-02 | 1번째 화면 이탈 뒤 답변 시간 | PRD v1.3 CHT-02: 같은 질문, 답변 시간 새로 / TRD v1.5 7-3: 남은 시간부터, 음성 합침 | TRD(`warn`·`resume`에 `remaining_sec`) | PRD CHT-02, 화면정의서 OV-02 |
| A-03 | 음성 글자 바꾸기 최종 실패 | 화면정의서 D-02: 면접 화면에서 같은 질문 다시 답변(OV-04, `stt_retry` 메시지) / TRD v1.5 7-2·7-5: 2번 다시 해도 안 되면 장애 | TRD(`paused`). `stt_retry`는 넣지 않음 | TRD 또는 화면정의서 중 하나 |
| A-04 | 자리 꽉 참 오류 코드 | TRD 4-3 ① `SEATS_FULL` / 5-3 `CAPACITY_FULL` | `CAPACITY_FULL` | TRD 4-3 |
| A-05 | 관리자 공고 경로 | TRD 5-1은 `POST /postings` 표기 + "/admin 아래" 설명 | `/admin/postings/…`로 통일 | TRD 5-1, 화면정의서 S-R01·S-R03 |
| A-06 | 답변 음성 올리기 | 화면정의서 S-A09: `POST /interview/{sid}/answers/{seq}/audio` / TRD v1.5: WebRTC 실시간 전송, 파일 업로드 없음 | TRD(파일 API 없음) | 화면정의서 S-A09 |
| A-07 | 장애 판단 시점 | 화면정의서 S-A15: 들어오면 '확인 중' 후 판단 / TRD 7-2: 끊긴 뒤 15초에 바로 판단 | TRD. `interview/state`는 결과를 읽기만 함 | 화면정의서 S-A15 ① |
| A-08 | 응시 상태 값 이름 | TRD에는 `not_taken_closed`·`ended_closed`·`needs_review`만, 화면정의서는 `drafting`만 | 2-4의 영문 값 제안 | 테이블 정의서 |
| A-09 | 결과 화면 문의 이메일 | 화면정의서 D-03: 공고에 `contact_email` / PRD N7·RPT-03에는 없음 | 공고·결과 응답에 `contact_email` 넣음 | PRD N7·RPT-03, TRD 4-1 |
| A-10 | AI 초안 출처·차별 표현 | TRD `source`는 `manual`·`open_api`·`dummy`만, 차별 표현 방식 미정(O-05, T-08) | `ai_draft` 추가, `bias_warnings` 칸만 둠 | TRD 4-1·15장 |
| A-11 | 녹화 올리기 | TRD 2-2는 "녹음·녹화를 조각내 올리기"만 있고 경로·조각 크기 없음 | `POST /interview/{sid}/recording`, WebM 조각 | TRD 5-1 |
| A-12 | 코딩테스트 시작 | 화면정의서 G-03: 안내 화면 + 60초 뒤 자동 시작 / TRD에 없음 | `POST /coding/{sid}/start` 추가 | TRD 9장 |
| A-13 | 접속 범위 | TRD: 127.0.0.1만(지원자도 서버 PC에서 응시) | 그대로. 다른 PC 접속으로 바꾸면 HTTPS, `Origin`, `cookie_secure`, WebRTC 알리는 주소를 바꿔야 함 | PRD 10-2(전제) |
