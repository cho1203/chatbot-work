# 03. 데이터 모델과 저장 계약

[목차](./README.md) · 이전: [환경](./02-development-environment.md) · 다음: [문서 처리](./04-document-pipeline.md)

## 1. 세 가지 저장소

SQLite는 권한·지식 버전·작업·대화의 기준 저장소다. 파일 저장소는 업로드 원본과 재처리 가능한 중간 산출물을 보관한다. Chroma는 SQLite 및 중간 산출물로 재생성할 수 있는 검색용 사본이다. **SQLite와 Chroma 사이에 분산 트랜잭션이 있다고 가정하지 않는다.**

모든 ID는 UUID 문자열, 모든 시각은 UTC ISO 8601로 통일한다. `public_id`는 128비트 이상 무작위 값으로 별도 생성한다. 화면에서만 한국 시간으로 변환한다. 파일명, 배열 순번, 모델 생성값을 권한 식별자로 쓰지 않는다.

## 2. ORM과 마이그레이션에 구현할 테이블

표의 공통 필드 `created_at`은 모든 테이블에, `updated_at`은 변경 가능한 테이블에 넣는다. JSON 필드는 SQLite TEXT로 저장하고 Pydantic으로 형식을 검증한다. FK는 SQLite `PRAGMA foreign_keys=ON`에서 실제 강제되어야 한다.

| 테이블 | 필수 필드 | 제약·인덱스 |
|---|---|---|
| users | id, email, password_hash, disabled_at | email 소문자 정규화 UNIQUE |
| sessions | id, user_id FK, token_hash, csrf_hash, expires_at, revoked_at | token_hash UNIQUE, user_id·expires_at 인덱스 |
| bots | id, owner_id FK, public_id, name, description, tone, greeting, fallback_message, is_public, active_generation_id nullable, corpus_revision, deleted_at | public_id UNIQUE, owner_id 인덱스 |
| documents | id, bot_id FK, kind(file/qa), display_name, desired_version_id nullable, deleted_at | bot_id 인덱스. 파일명 UNIQUE 금지 |
| document_versions | id, document_id FK, version_no, original_sha256, raw_path nullable, mime, byte_size, qa_json nullable, extraction_status, review_status, parser_version, chunk_policy, derived_path, chunk_count, error_code | UNIQUE(document_id,version_no) |
| chunks | id, version_id FK, ordinal, kind, text, embed_text, text_sha256, location_json, metadata_json | UNIQUE(version_id,ordinal), version_id 인덱스 |
| index_generations | id, bot_id FK, collection_name, status, corpus_revision, manifest_path, embedding_fingerprint, chunk_count, activated_at | collection_name UNIQUE, bot_id·status 인덱스 |
| index_members | generation_id FK, chunk_id FK | 복합 PRIMARY KEY |
| jobs | id, bot_id FK, document_version_id nullable, kind, state, stage, payload_json, progress, attempt, lease_until, error_code, started_at, finished_at | state·created_at 인덱스 |
| visitor_sessions | id, bot_id FK, token_hash, csrf_hash, expires_at, revoked_at | token_hash UNIQUE, bot_id 인덱스 |
| conversations | id, bot_id FK, visitor_session_id nullable FK, owner_id nullable FK, last_at, deleted_at | 두 소유 주체 중 정확히 하나, bot_id·last_at 인덱스 |
| chat_runs | id, conversation_id FK, client_request_id, request_sha256, state, generation_id nullable FK, retrieval_json, prompt_version, model_digest, error_code, started_at, finished_at | UNIQUE(conversation_id,client_request_id) |
| messages | id, conversation_id FK, run_id FK, role, content, answer_status nullable, sources_json, is_partial | UNIQUE(run_id,role), conversation_id·created_at 인덱스 |
| inbox_items | id, bot_id FK, question_key, question_redacted, count, status, resolution_document_id nullable FK, last_at | UNIQUE(bot_id,question_key) |
| feedback | id, run_id FK, actor_key, rating, reason nullable | UNIQUE(run_id,actor_key), rating=up/down |
| audit_events | id, actor_id nullable, action, bot_id nullable, target_id nullable, request_id, detail_redacted_json | created_at·action 인덱스 |

`bots.active_generation_id`와 `documents.desired_version_id`의 순환 FK는 첫 삽입 시 NULL로 두고 자식 행 삽입 후 연결한다. 활성 세대가 같은 봇, 희망 버전이 같은 문서 소속인지 서비스와 테스트에서 검사한다. 가능하면 복합 FK로도 강제한다. 다른 봇의 청크가 `index_members`에 포함될 수 없게 인덱스 검증 시 조인 검사한다.

`documents.desired_version_id`는 다음 색인에 반영할 승인 버전이다. 실제 검색 자료는 **활성 세대의 index_members**가 결정한다. 희망 버전으로 교체 중이어도 실패하면 기존 버전이 계속 검색된다. UI에는 “현재 서비스 버전”과 “반영 대기 버전”을 별도로 표시한다.

회원·봇 삭제는 서비스에서 순서를 정해 처리한다. 파일·Chroma 삭제를 DB의 `ON DELETE CASCADE`가 처리한다고 가정하지 않는다. 외부 저장소 정리 전에는 감사·복구에 필요한 관계를 보존한다.

문자열 ID는 `String(36)`, 해시는 `String(64)`, 본문·JSON은 `Text`, 카운트·revision은 `Integer`, 플래그는 `Boolean`으로 매핑한다. NULL 허용은 표에 nullable이 있거나 아직 일어나지 않은 시각·실패 값·삭제 값에만 적용한다. 상태 값은 CHECK 또는 ORM enum으로 제한하고 전체 상태 전이도 서비스에서 검사한다. `progress`는 07장 객체 형태를 JSON TEXT로 보관한다. 미답변·대화의 개인정보 보존 정책과 FK 삭제 정책을 초기 migration에 명시한다.

SQLite engine 연결 시 `foreign_keys=ON`, `busy_timeout=5000`을 적용하고 초기화 때 WAL 모드를 설정한다. SQLAlchemy 세션은 요청/작업 단계마다 새로 만들고 쓰기 트랜잭션을 짧게 유지한다. `check_same_thread=False`가 동시 쓰기를 무제한 안전하게 만들어 주는 설정은 아니다. lifespan에서 API/CLI 공용 파일 잠금을 유지해 기본판의 단일 프로세스 저장소 사용을 강제한다.

## 3. 파일 저장 규칙

```text
data/
  app.sqlite3
  raw/<bot_id>/<document_id>/<version_id>/original.<검증한 확장자>
  derived/<version_id>/extracted.json
  derived/<version_id>/chunks.jsonl
  derived/<version_id>/assets/<asset_id>.png
  manifests/<generation_id>.json
  chroma/                   # Chroma 전용, 직접 내부 파일 수정 금지
  tmp/<job_id>/             # 임시 작업 결과
```

실제 사용자가 올린 이름은 DB 표시 필드에만 보관한다. 저장 경로는 서버 생성 ID로 만든다. 경로를 resolve한 결과가 데이터 루트 안에 있는지 확인한다. 임시 저장→해시·크기 검증→동일 파일시스템 내 원자적 rename을 적용한다. 파일 저장 후 DB 등록 실패로 고아 파일이 남으면 24시간 이상 지난 참조 없는 임시 파일만 정리한다.

원본·파생 자료는 정적 디렉터리에 노출하지 않는다. 출처 API가 봇·대화 소유권, 문서 tombstone, 공개 상태를 다시 확인한 뒤 제공한다. 공개 봇은 기본적으로 인용에 사용한 텍스트 조각만 제공하며 원본 다운로드는 관리자만 허용한다.

## 4. 추출 산출물 스키마

`extracted.json`의 필수 형태:

```json
{
  "schema_version": "1",
  "version_id": "11111111-1111-4111-8111-111111111111",
  "source_sha256": "실제 파일의 64자리 SHA-256",
  "parser": {"name": "pypdf", "version": "실제 설치 버전", "policy": "extract-v1"},
  "pages": 2,
  "blocks": [
    {
      "block_id": "p1-b1",
      "type": "text",
      "text": "고객센터는 평일 09:00~18:00에 운영합니다.",
      "location": {"page_start": 1, "page_end": 1, "section_path": ["이용 안내"]},
      "method": "native",
      "confidence": null
    }
  ],
  "warnings": [],
  "quality": {"requires_review": false, "empty_pages": [], "replacement_char_ratio": 0.0}
}
```

이 JSON은 형식 예시이고 SHA 문자열·버전 값은 실행 시 채운다. PDF 페이지는 1부터 시작하는 실제 파일 페이지 번호다. 인쇄된 쪽수는 `page_label`로 별도 저장한다. DOCX에는 실제 페이지 번호를 추측해 넣지 않고 `paragraph_index` 또는 `table_index`, CSV에는 `row_start`·`row_end`, TXT·MD에는 `line_start`·`line_end`를 넣는다. 표·그림의 추가 필드는 04장을 따른다.

## 5. 청크 스키마와 결정적 ID

```json
{
  "schema_version": "1",
  "chunk_id": "결정적으로 생성한 UUID",
  "version_id": "11111111-1111-4111-8111-111111111111",
  "ordinal": 0,
  "kind": "passage",
  "text": "고객센터는 평일 09:00~18:00에 운영합니다.",
  "embed_text": "이용 안내 > 고객센터\n고객센터는 평일 09:00~18:00에 운영합니다.",
  "text_sha256": "정규화 텍스트의 64자리 SHA-256",
  "location": {"page_start": 1, "page_end": 1, "block_ids": ["p1-b1"]},
  "metadata": {"title": "고객센터", "language": "ko", "extraction_method": "native", "policy": "chunk-v1"}
}
```

새 원본 또는 새 청킹 정책은 새 문서 버전으로 처리한다. 같은 버전의 재시도에서는 정규화·정렬 순서를 고정해 동일 청크 ID가 생성되어야 한다.

```python
import hashlib
import uuid

def chunk_id(version_id: str, policy: str, ordinal: int, text: str) -> str:
    digest = hashlib.sha256(text.encode("utf-8")).hexdigest()
    return str(uuid.uuid5(uuid.UUID(version_id), f"{policy}:{ordinal}:{digest}"))
```

정확한 중복은 같은 봇 안에서만 판단한다. 원본 파일 SHA가 같은 재업로드는 기본적으로 기존 문서·작업을 반환한다. 같은 이름·다른 내용은 새 문서로 만들거나 사용자가 명시한 문서의 새 버전으로 저장한다. 파일명으로 대량 삭제하지 않는다. 청크 내용이 같더라도 서로 다른 문서의 위치·권한을 삭제해 하나로 합치지 않는다.

## 6. Chroma 저장 계약

컬렉션 이름은 `bot_<botUUIDhex>_g_<generationUUIDhex>`로 서버에서 생성한다. `ids`는 chunk_id, `documents`는 embed_text, `embeddings`는 명시적 벡터다. 원문·위치의 최종 기준은 SQLite의 청크다. 메타데이터는 기본판에서 문자열·숫자·불리언의 평탄한 값으로 제한한다.

```json
{
  "bot_id": "봇 UUID",
  "document_id": "문서 UUID",
  "version_id": "버전 UUID",
  "chunk_id": "청크 UUID",
  "kind": "passage",
  "title": "고객센터",
  "page_start": 1,
  "policy": "chunk-v1"
}
```

알 수 없는 페이지를 0으로 꾸미지 않고 필드를 생략한다. 복잡한 표·위치 목록은 SQLite JSON과 추출 파일에 보관한다. 검색 후 반환된 ID를 SQLite에 조인해 허용된 청크인지 검증한다.

## 7. 작업과 인덱스 상태

문서 추출 상태: `uploaded → extracting → extracted → chunked` 또는 `failed`.

검토 상태: `pending → approved` 또는 `needs_review → approved/rejected`. 명백한 기본 텍스트 자료는 품질 검사를 통과하면 자동 approved로 전환할 수 있다. OCR 결과는 기본적으로 사람이 승인한다.

작업 상태: `queued → running → succeeded/needs_review/failed/cancelled/superseded`; needs_review는 추출 검토 대기 종료 상태이며 승인하면 새 작업을 만들어 청킹·색인을 진행한다. succeeded는 그 작업의 목표가 완료된 경우에만 사용하며 ingest/reindex는 세대 활성화까지 요구한다. 오래된 revision 또는 병합된 작업은 superseded로 종료하고 `payload_json.superseded_by`에 후속 작업 ID를 기록해 응답에도 제공한다. 관리자가 실패 작업을 재시도하면 동일 대상을 가진 **새 작업 ID**를 만들고 `payload_json.retry_of`로 연결한다. 진행 단계 `stage`는 extracting/chunking/embedding/validating/activating 등이다. `progress`는 처리 건수 기반이며 알 수 없는 시간의 가짜 백분율을 만들지 않는다.

인덱스 상태: `building → validated → active → retired` 또는 `failed`. 활성 상태는 봇별 하나다. 바꾸는 트랜잭션에서 이전 세대를 retired로, 새 세대를 active로 만들고 봇 포인터를 교체한다.

작업 루프는 DB에서 하나씩 접수하여 짧은 트랜잭션으로 lease를 획득한다. 실행 중 10초마다 갱신하고 60초 lease를 시작값으로 사용한다. 재시작 시 lease 만료 작업은 중간 산출물을 확인해 실패·재시도 대상으로 복구한다. 신규 시도는 새 generation을 사용하며 불완전 컬렉션을 활성화하지 않는다. 기본판은 전역 인덱싱 작업 1개로 제한한다.

동기 파서·Chroma 호출은 제한된 thread executor에서 실행하고 네트워크는 async client를 사용해 이벤트 루프와 lease heartbeat를 막지 않는다. 강제 종료가 필요한 변환/OCR은 별도 프로세스에서 처리한다. 작업의 재시도 전체 횟수는 최대 3회로 제한하고 영구적인 입력·설정 오류는 자동 재시도하지 않는다.

## 8. 불일치 방지와 삭제

`corpus_revision`은 승인 버전 변경·삭제 시 증가한다. 인덱싱 시작 시 revision과 문서 버전 집합을 캡처한다. 활성화 직전 값이 바뀌었으면 stale 작업으로 종료하고 최신 revision의 작업을 만든다. 오래 걸린 과거 작업이 최신 자료를 덮는 일을 방지한다.

삭제는 먼저 SQLite에 `deleted_at`을 기록하고 revision을 증가시킨다. 검색 및 출처 제공은 즉시 해당 자료를 차단한다. 새 세대를 구축한 다음 이전 벡터·원본을 보존 정책에 맞춰 정리한다. 벡터 삭제 실패에도 사용자가 삭제한 문서가 답변에 나와서는 안 된다.

대화에는 사용한 문서 버전·청크·인덱스 세대·인용문 사본을 저장한다. 문서 삭제 후 과거 대화의 출처 링크는 “삭제된 자료”로 표시한다. 개인정보 완전 삭제 요청에는 과거 인용문·로그·백업의 보존 만료까지 처리 범위를 기록한다.

## 9. 마이그레이션 절차

`alembic init migrations` 후 `env.py`가 Settings의 DB URL과 `models.Base.metadata`를 사용하게 작성한다. SQLite는 `render_as_batch=True`를 검토하고 생성된 마이그레이션을 사람이 확인한다.

```powershell
.\.venv\Scripts\python.exe -m alembic revision --autogenerate -m "initial chatbot schema"
.\.venv\Scripts\python.exe -m alembic upgrade head
.\.venv\Scripts\python.exe -m alembic current
```

빈 DB 생성 시험과 이전 버전 사본 업그레이드 시험을 따로 수행한다. 앱 시작 시 임의 `ALTER TABLE`을 실행하거나 실패를 무시하지 않는다. 운영 downgrade는 파괴적일 수 있으므로 DB 백업과 앱 버전을 함께 복원하는 절차를 먼저 준비한다.
