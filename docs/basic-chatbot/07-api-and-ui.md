# 07. API·SSE·관리 화면·채팅 화면

[목차](./README.md) · 이전: [답변](./06-answer-and-conversation.md) · 다음: [구현·인수](./08-build-and-acceptance.md)

## 1. 공통 HTTP 계약

모든 업무 API는 `/api/v1`을 사용한다. 기본 origin은 `http://127.0.0.1:8000` 하나이며 프록시를 거쳐도 API 경로를 바꾸지 않는다. JSON은 UTF-8, 필드명은 snake_case, 날짜는 UTC ISO 8601이다. multipart 업로드 외에는 JSON을 쓴다.

인증 쿠키를 사용하는 변경 요청에는 `X-CSRF-Token`을 요구한다. 로그인은 정확한 Origin 검사와 JSON Content-Type 검사로 login CSRF를 차단한다. 로그인 이후 세션마다 발급한 CSRF 토큰을 사용한다. API는 임의 `owner_id`, 컬렉션명, 모델 URL을 받지 않는다.

오류 공통 형태:

```json
{
  "error": {
    "code": "index_not_ready",
    "message": "자료를 준비하고 있습니다. 잠시 후 다시 시도해 주세요.",
    "retryable": true
  },
  "request_id": "요청 UUID"
}
```

Pydantic의 기본 422도 exception handler에서 위 envelope로 통일하고 field_errors를 선택적으로 추가한다. 내부 경로·스택·SQL·모델 키는 사용자 응답에 포함하지 않는다. request_id는 서버가 발급하고 `X-Request-ID`에도 싣는다.

목록 응답은 `{"items":[],"next_cursor":null}`. limit 기본 20, 최대 100, cursor는 `(created_at,id)`의 불투명 인코딩으로 구현한다. 소유권 필터를 적용한 뒤 페이지를 만든다.

## 2. 인증과 봇 관리 API

| 메서드·경로 | 권한 | 요청 → 성공 응답 |
|---|---|---|
| POST `/api/v1/auth/login` | 익명, Origin 검사 | email/password → 200 user/csrf_token + 세션 쿠키 |
| GET `/api/v1/auth/me` | 로그인 | 200 user/csrf_token |
| POST `/api/v1/auth/logout` | 로그인+CSRF | 204, 서버 세션 폐기 |
| GET `/api/v1/bots` | 로그인 | 본인 봇 목록 |
| POST `/api/v1/bots` | 로그인+CSRF | name/description → 201 Bot |
| GET `/api/v1/bots/{bot_id}` | 소유자 | 200 Bot |
| PATCH `/api/v1/bots/{bot_id}` | 소유자+CSRF | 표시·공개 설정 일부 → 200 Bot |
| DELETE `/api/v1/bots/{bot_id}` | 소유자+CSRF | 202 job_id, 즉시 접근 차단 |

봇 생성 body:

```json
{"name":"샘플 안내봇","description":"이용 안내를 답변합니다."}
```

Bot 응답 필수 필드: id, public_id, name, description, tone, greeting, fallback_message, is_public, active_generation_id, corpus_revision, knowledge_state, public_url, created_at. `knowledge_state=empty|building|ready|failed`. 기본 tone은 `concise`, is_public은 false다. 이름 1~80자, 소개 0~1,000자, 인사말 500자, 대체 문구 500자, 말투는 `friendly|concise|professional` 허용 목록으로 제한한다. 자유 시스템 프롬프트 편집은 기본판에서 제공하지 않는다.

다른 소유자의 봇은 404로 응답한다. 같은 공개 ID라도 비공개 상태이면 공개 API에서 404를 반환한다.

## 3. 문서·작업·검색 관리 API

| 메서드·경로 | 요청 → 응답 |
|---|---|
| GET `/api/v1/bots/{bot_id}/documents` | 문서와 서비스/대기 버전 목록 |
| POST `/api/v1/bots/{bot_id}/documents` | multipart `file`, 선택 display_name → 202 document_id/version_id/job_id |
| POST `/api/v1/bots/{bot_id}/qa` | question/answer → 202 document_id/version_id/job_id |
| POST `/api/v1/bots/{bot_id}/documents/{document_id}/versions` | file 또는 QA JSON, 기존 kind 유지 → 202 새 버전·작업 |
| GET `/api/v1/bots/{bot_id}/documents/{document_id}/versions/{version_id}/preview` | 추출본·경고·위치, 소유자만 |
| POST `/api/v1/bots/{bot_id}/documents/{document_id}/versions/{version_id}/approve` | 승인, 선택 comment → 202 job_id |
| DELETE `/api/v1/bots/{bot_id}/documents/{document_id}` | tombstone → 202 job_id |
| POST `/api/v1/bots/{bot_id}/reindex` | reason → 202 job_id |
| GET `/api/v1/bots/{bot_id}/jobs/{job_id}` | 작업 상태·단계·오류 |
| POST `/api/v1/bots/{bot_id}/jobs/{job_id}/retry` | 재시도 가능 failed 작업 → 202 새 job_id |
| POST `/api/v1/bots/{bot_id}/retrieve` | question/top_k → 200 Evidence 결과 |

모두 소유자 권한이며 변경 요청에는 CSRF를 요구한다. JSON Q&A는 question 1~2,000자, answer 1~20,000자. 형식에 따라 `/versions`가 받는 Content-Type을 분기하고 다른 kind로 바꾸려면 새 문서를 만든다. approval은 `needs_review` 추출을 승인할 때 사용하며 이미 승인된 버전은 멱등 응답한다.

`GET /api/v1/bots/{bot_id}/health`는 소유자에게 해당 봇의 knowledge_state, active_generation_id, chunk_count, latest_job을 반환한다. 전체 서버의 내부 경로·타 봇 현황은 포함하지 않는다.

202 예시:

```json
{
  "document_id":"문서 UUID",
  "version_id":"버전 UUID",
  "job_id":"작업 UUID",
  "state":"queued",
  "poll_after_seconds":2
}
```

작업 GET 예시:

```json
{
  "id":"작업 UUID",
  "kind":"ingest",
  "state":"running",
  "stage":"embedding",
  "progress":{"processed":16,"total":40},
  "error":null,
  "created_at":"2026-09-18T00:00:00Z",
  "finished_at":null
}
```

동일 봇에서 queued reindex가 여러 개면 최신 corpus_revision 하나로 병합할 수 있다. 병합된 원래 작업도 `superseded_by`를 명시해 UI 폴링이 영원히 queued에 머물지 않게 한다.

## 4. 공개 및 소유자 채팅 API

| 메서드·경로 | 역할 |
|---|---|
| GET `/api/v1/public/bots/{public_id}` | 안전한 공개 설정만 반환 |
| POST `/api/v1/public/bots/{public_id}/sessions` | Origin 검사 후 방문 세션 쿠키·CSRF 발급 |
| POST `/api/v1/public/bots/{public_id}/conversations` | 방문 세션+CSRF로 새 대화 생성 |
| POST `/api/v1/public/bots/{public_id}/chat/stream` | 자신의 대화에서 SSE 질의 |
| POST `/api/v1/public/bots/{public_id}/chat` | 같은 계약의 최종 JSON 질의 |
| GET `/api/v1/public/bots/{public_id}/runs/{run_id}` | 자신의 실행 상태·저장 결과 조회 |
| GET `/api/v1/public/bots/{public_id}/runs/{run_id}/sources/{evidence_id}` | 그 실행에 실제 사용한 인용문 조회 |
| POST `/api/v1/public/bots/{public_id}/runs/{run_id}/feedback` | rating/reason 저장 |
| POST `/api/v1/bots/{bot_id}/conversations` | 소유자 테스트 대화 생성 |
| POST `/api/v1/bots/{bot_id}/chat/stream` | 비공개 상태에서도 소유자 시험 |
| POST `/api/v1/bots/{bot_id}/chat` | 소유자 JSON 질의 |
| GET `/api/v1/bots/{bot_id}/runs/{run_id}` | 소유자 실행 조회 |
| GET `/api/v1/bots/{bot_id}/runs/{run_id}/sources/{evidence_id}` | 소유자 출처 조회 |
| POST `/api/v1/bots/{bot_id}/runs/{run_id}/cancel` | 소유자 진행 중 실행 취소 |
| POST `/api/v1/public/bots/{public_id}/runs/{run_id}/cancel` | 방문자 자신의 실행 취소 |

공개 설정에는 name, greeting, tone, knowledge_state만 넣는다. 소유자 ID·운영 설정·비공개 원본 경로는 제외한다. 방문자 쿠키는 서버가 발급한 난수 토큰이고 DB에는 해시만 저장한다. 브라우저가 임의로 제출하는 session_id만으로 남의 대화에 접근할 수 없어야 한다. 기본 방문 세션 만료는 24시간이다.

방문 세션은 봇별 쿠키 이름 `visitor_<public_id>`로 구분해 여러 봇을 열 때 충돌하지 않게 한다. `/sessions`는 유효한 쿠키가 있으면 기존 방문 세션과 CSRF를 재사용하고 없거나 만료된 경우에만 새로 만든다. 새 대화 생성 응답은 201 `{"conversation_id":"새 UUID"}`이며 세션 생성 응답은 201 또는 재사용 시 200 `{"csrf_token":"토큰","expires_at":"UTC 시각"}`이다. CSRF는 `base64url(HMAC-SHA256(SESSION_SECRET, "csrf:" + session_id))`로 생성하여 다른 탭과 GET me에서도 같은 값을 반환한다. 원문은 DB에 저장하지 않고 SHA-256 해시를 csrf_hash로 저장한다. 세션 토큰 해싱과 CSRF 파생의 입력 영역을 구별한다.

기본판은 동일 origin 채팅으로 시작한다. iframe·QR은 나중에 추가하며, iframe을 사용하면 제3자 쿠키·CSP·허용 origin 정책을 새로 검증한다.

채팅 요청:

```json
{
  "conversation_id":"대화 UUID",
  "client_request_id":"클라이언트 생성 UUID",
  "question":"고객센터 운영 시간이 어떻게 되나요?"
}
```

서버는 대화가 경로의 봇과 인증 주체에 속하는지 검사한다. 모델·temperature·top_k·owner_id는 공개 요청에서 받지 않는다. 클라이언트 request_id는 네트워크 재시도 때 동일하게 재사용하고 새 질문에는 새 값을 만든다.

JSON 완료 응답:

```json
{
  "run_id":"실행 UUID",
  "conversation_id":"대화 UUID",
  "status":"answered",
  "answer":"고객센터는 평일 09:00~18:00에 운영합니다. [S1]",
  "sources":[{"id":"S1","title":"이용 안내","location":{"page_start":1},"excerpt":"고객센터는 평일 09:00~18:00에 운영합니다.","url":"서버가 생성한 해당 실행의 출처 API 경로"}],
  "timing_ms":{"total":3200},
  "persisted":true
}
```

`chat_runs.state=completed`와 `AnswerResult.status=unanswered`는 함께 가능하다. 실행 성공과 지식 답변 성공을 구별한다. 동기 JSON 요청도 동일 deadline을 쓰고 180초 이내 실패하면 오류 응답한다. 폴링 가능한 run_id가 발급되었다면 중복 생성하지 않는다.

## 5. SSE 규격

HTTP 헤더는 `Content-Type: text/event-stream; charset=utf-8`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`. 이벤트는 `event` 이름과 JSON `data`를 사용한다. 정상 실행에는 정확히 하나의 done, 실패에는 error 후 정확히 하나의 done을 보낸다. 연결 자체가 끊어졌으면 전송은 불가능하므로 서버 저장 상태를 기준으로 한다.

```text
event: start
data: {"schema_version":"1","run_id":"run-1","conversation_id":"conv-1"}

event: status
data: {"stage":"retrieving","message":"자료를 찾고 있습니다."}

event: sources
data: {"items":[{"id":"S1","title":"이용 안내","location":{"page_start":1}}]}

event: token
data: {"text":"고객센터는 평일 09:00~18:00에 운영합니다. [S1]"}

event: done
data: {"status":"answered","persisted":true,"timing_ms":{"total":3200}}

```

기본판 sources는 최종 검증·저장된 출처 목록이고 token은 검증 완료 후 전달한다. 대기 중에는 15초마다 `: heartbeat\n\n` 주석을 보낼 수 있다. SSE 시작 전 인증·입력·admission 오류는 HTTP 4xx/5xx JSON이다. 시작 후 오류는 error 이벤트로 표현한다.

```text
event: error
data: {"code":"model_unavailable","message":"답변 서비스를 사용할 수 없습니다.","retryable":true}

event: done
data: {"status":"error","persisted":false}

```

SSE 자동 재연결은 구현하지 않는다. POST 요청을 EventSource로 보내려 하지 말고 fetch 스트림을 사용한다. EOF가 왔는데 done을 받지 못했다면 실패/연결 끊김으로 표시하고 run 조회로 실제 완료 여부를 확인한다.

## 6. 브라우저 SSE 파서 예시

아래는 `app/static/chat.js`의 파서로 사용할 수 있는 독립 함수다. UTF-8 문자, 줄, 프레임은 네트워크 조각마다 끊길 수 있으므로 각각 버퍼링한다.

```javascript
export async function readSSE(response, onEvent) {
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  if (!response.headers.get("content-type")?.includes("text/event-stream")) {
    throw new Error("Unexpected content type");
  }
  if (!response.body) throw new Error("Empty response body");
  const reader = response.body.getReader();
  const decoder = new TextDecoder("utf-8");
  let buffer = "", eventName = "message", data = [], ended = false;
  function line(value) {
    if (value === "") {
      if (data.length) {
        const payload = JSON.parse(data.join("\n"));
        if (eventName === "done") ended = true;
        onEvent(eventName, payload);
      }
      eventName = "message";
      data = [];
      return;
    }
    if (value.startsWith(":")) return;
    const colon = value.indexOf(":");
    const field = colon < 0 ? value : value.slice(0, colon);
    let content = colon < 0 ? "" : value.slice(colon + 1);
    if (content.startsWith(" ")) content = content.slice(1);
    if (field === "event") eventName = content;
    if (field === "data") data.push(content);
  }
  function drain() {
    let pos;
    while ((pos = buffer.indexOf("\n")) >= 0) {
      let value = buffer.slice(0, pos);
      buffer = buffer.slice(pos + 1);
      if (value.endsWith("\r")) value = value.slice(0, -1);
      line(value);
      if (ended) break;
    }
  }
  try {
    while (!ended) {
      const { value, done } = await reader.read();
      if (done) {
        buffer += decoder.decode();
        drain();
        break;
      }
      buffer += decoder.decode(value, { stream: true });
      drain();
      if (buffer.length > 1048576) throw new Error("SSE frame too large");
    }
    if (!ended) throw new Error("Stream ended without done");
  } finally {
    await reader.cancel().catch(() => {});
    reader.releaseLock();
  }
}
```

호출은 ES module `<script type="module">`에서 수행한다. 송신 요청에는 `credentials:'same-origin'`, JSON Content-Type, `X-CSRF-Token`, AbortController.signal을 넣는다. HTTP 오류는 파서 호출 전에 JSON envelope를 읽어 사용자 메시지로 바꾸면 된다. UI 이벤트 핸들러는 알 수 없는 이벤트를 무시하고 error·done은 반드시 처리한다.

## 7. 미답변·기록 API

| 메서드·경로 | 계약 |
|---|---|
| GET `/api/v1/bots/{bot_id}/inbox?status=open` | 빈도·질문·최근 시간 목록 |
| POST `/api/v1/bots/{bot_id}/inbox/{item_id}/answer` | answer → Q&A·색인 작업, 해결은 색인 성공 후 |
| POST `/api/v1/bots/{bot_id}/inbox/{item_id}/ignore` | 선택 reason → 200 상태 |
| GET `/api/v1/bots/{bot_id}/conversations` | 소유자만 대화 목록 |
| GET `/api/v1/bots/{bot_id}/conversations/{conversation_id}` | 소유자만 메시지·출처·실행 상태 |
| DELETE `/api/v1/bots/{bot_id}/conversations/{conversation_id}` | 204, 기록 보존 정책 적용 |

개인정보가 들어갈 수 있는 검색·대화·미답변에는 마스킹과 접근 로그를 적용한다. 공개 방문자가 소유자의 전체 대화 목록을 읽는 API는 없다.

## 8. HTTP 코드

| 코드 | 경우 |
|---|---|
| 400 | 잘못된 cursor, 상태와 맞지 않는 동작 |
| 401 | 로그인·방문 세션 필요 또는 만료 |
| 403 | CSRF/Origin 오류 |
| 404 | 대상 없음·권한 밖 대상·비공개 공개 링크 |
| 409 | 동일 ID의 다른 본문, 실행 중 중복, 버전 충돌 |
| 413 | 파일·요청 크기 초과 |
| 415 | 지원하지 않는 파일/Content-Type |
| 422 | 입력 schema, 빈 질문, 추출 불가능한 자료 |
| 429 | 요청 제한·대기열 포화, Retry-After 포함 |
| 503 | 모델·인덱스·의존성 준비 안 됨 |
| 504 | 전체 처리 기한 초과 |
| 500 | 미분류 내부 오류, request_id로 추적 |

## 9. 화면 구현 순서와 상태

관리 화면 `/admin/login`, `/admin/bots`, `/admin/bots/{bot_id}`를 만든다. 봇 상세 화면은 설정, 지식, 검색 시험, 미답변, 대화 기록의 다섯 탭이다. 지식 탭에서 업로드 직후 “처리 중”을 표시하고, 현재 서비스 버전·대기 버전·실패 이유·재시도·추출 미리보기·승인을 제공한다. 성공 메시지는 활성 세대 반영 후에 표시한다.

채팅 화면은 소유자 `/admin/bots/{bot_id}/chat`, 공개 `/c/{public_id}`로 분리한다. 기본 흐름은 설정 조회→세션 발급/확인→대화 생성→질문→검색/생성 진행→본문·출처→피드백이다. 새 대화는 서버에서 새 conversation_id를 만든다.

입력 상태는 idle/submitting/waiting/generating/completed/error/cancelled를 구별한다. 중복 클릭을 막고 AbortController와 cancel API를 연결한다. 네트워크 재시도는 동일 client_request_id로 처리한다. 429는 Retry-After를 표시하고 자동 무한 재시도하지 않는다.

초기 본문은 `textContent`로 렌더한다. Markdown이 필요하면 로컬 고정 버전 파서와 sanitizer를 함께 추가하고 위험 URL scheme·raw HTML을 막는다. 출처 제목도 HTML로 삽입하지 않는다. 코드 복사는 텍스트만 복사한다. 로딩·오류 상태에 aria-live를 적용하고 Enter 전송, Shift+Enter 줄바꿈, 키보드 포커스를 지원한다.
