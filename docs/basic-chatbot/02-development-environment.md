# 02. 개발 환경과 프로젝트 구성

[목차](./README.md) · 이전: [시스템](./01-system-definition.md) · 다음: [데이터 계약](./03-data-contracts.md)

## 1. 준비할 도구

필수는 Python 3.11의 최신 보안 패치 버전, Git, Ollama, 편집기, 브라우저다. Windows PowerShell 개발을 기본으로 설명하고 Linux 명령을 함께 제공한다. Python·Ollama는 각 공식 설치 경로를 이용한다. [Python 설치](https://www.python.org/downloads/), [Ollama 설치](https://ollama.com/download).

개발 장치의 계획값은 RAM 16GB 이상, 여유 디스크 20GB 이상이다. GPU는 기본 문서 처리에 필수가 아니며 CPU 생성은 느릴 수 있다. 실제 메모리 요구량은 모델과 문맥 길이에 따라 달라진다. GPU가 있더라도 PyTorch/CUDA를 기본 앱에 먼저 설치할 필요는 없다. 이 설계의 모델 실행은 Ollama에 위임한다. OCR·cross encoder를 추가할 때 별도 의존성을 설치한다.

사내 프록시·오프라인 환경에서는 Python wheel, 모델 파일, 로컬 JS/CSS 자산을 승인된 경로로 준비한다. 모델·파서·폰트의 라이선스와 재배포 조건은 실제 채택 버전에 대해 기록한다.

## 2. 빈 프로젝트 생성

다음은 사용자가 선택한 개발 위치에서 실행하는 PowerShell 예시다. 기존 프로젝트가 있는 경로에는 다시 만들지 않는다.

```powershell
New-Item -ItemType Directory -Path C:\Projects\base-chatbot
Set-Location C:\Projects\base-chatbot
git init
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip pip-tools
$folders = @('app/routers','app/services','app/repositories','app/templates','app/static','migrations','tests/fixtures','tests/integration','tests/unit','eval','deploy','scripts','data/raw','data/derived','data/chroma','data/manifests','reports')
foreach ($folder in $folders) { New-Item -ItemType Directory -Force -Path $folder | Out-Null }
```

Linux에서는 새 디렉터리에서 `python3.11 -m venv .venv`를 실행하고 이후 명령의 `.\.venv\Scripts\python.exe`를 `.venv/bin/python`으로 바꾼다. 문서의 `python`은 항상 이 가상환경의 실행 파일을 뜻한다. 활성화가 필요하면 Windows는 `& .\.venv\Scripts\Activate.ps1`, Linux는 `source .venv/bin/activate`를 사용한다. PowerShell 실행 정책 때문에 활성화가 안 되면 실행 정책을 전역 변경하지 말고 실행 파일 전체 경로를 사용한다.

## 3. 의존성 정의와 잠금

다음 내용을 `requirements.in`으로 저장한다. 버전 범위는 **신규 개발용 후보 범위**이며 이 문서 작성 중 설치 시험한 잠금 파일은 아니다. 기존 앱의 서로 다른 requirements를 합치지 않는다.

```text
fastapi>=0.127,<1
uvicorn[standard]>=0.40,<1
pydantic>=2.10,<3
pydantic-settings>=2.7,<3
sqlalchemy>=2.0.36,<2.1
alembic>=1.14,<2
chromadb>=1.3.7,<2
httpx>=0.28,<1
jinja2>=3.1.5,<4
python-multipart>=0.0.21,<1
argon2-cffi>=23.1,<26
pypdf>=6,<7
python-docx>=1.1.2,<2
```

`requirements-dev.in`:

```text
-r requirements.in
pytest>=8,<10
pytest-asyncio>=0.25,<2
ruff>=0.9,<1
```

설치 및 잠금:

```powershell
.\.venv\Scripts\python.exe -m piptools compile --generate-hashes --output-file requirements.lock.txt requirements.in
.\.venv\Scripts\python.exe -m piptools compile --generate-hashes --output-file requirements-dev.lock.txt requirements-dev.in
.\.venv\Scripts\python.exe -m pip install --require-hashes -r requirements-dev.lock.txt
.\.venv\Scripts\python.exe -m pip check
```

Linux 운영에서 동일 절차로 `requirements-linux.lock.txt`를 생성·검증한다. 운영 이미지는 그 파일로 설치한다. OS가 다른 잠금 파일을 무조건 재사용하지 않는다. 개발용 잠금 파일의 실행 의존성과 운영 잠금 파일의 버전이 같은지 비교하고 차이를 해소한다. 설치 실패 시 임의로 패키지를 지우거나 버전을 느슨하게 풀지 말고 Python ABI, wheel 제공 여부, 충돌 패키지를 기록해 후보 범위와 잠금을 함께 수정한다.

최종 배포에는 실제 사용한 Python 패치 버전·잠금 파일 해시·Ollama 버전·모델 digest를 `reports/environment.json`에 기록한다. 이후 환경은 이 기록으로 재현한다. 최신 패키지라는 이유만으로 검증된 잠금을 매 실행마다 갱신하지 않는다.

## 4. 프로젝트 파일 책임

```text
base-chatbot/
  app/
    __init__.py
    main.py                 # create_app, lifespan, 미들웨어, 라우터
    config.py               # Settings와 시작 검증
    database.py             # engine, 세션, SQLite PRAGMA
    models.py               # ORM: 03장 계약
    schemas.py              # Pydantic 요청·응답·추출·청크
    auth.py                 # 비밀번호, DB 세션, CSRF, 소유권
    cli.py                  # 08장 관리 명령
    routers/                # auth, bots, documents, jobs, chat, inbox, health
    repositories/           # DB 조회·트랜잭션
    services/
      extractors.py         # 기본 형식별 추출
      normalizer.py
      chunker.py
      ingestion.py          # 검토·버전 선택·작업 등록
      jobs.py               # 영속 작업 루프·재시도
      ollama.py             # embed, generate, probe
      vector_store.py       # Chroma 접근의 유일한 통로
      indexing.py           # 세대 생성·검증·교체
      retrieval.py
      prompting.py
      answering.py
      conversations.py
      inbox.py
      traffic.py
    templates/              # login, bots, bot_detail, chat
    static/                 # app.css, admin.js, chat.js
  migrations/versions/
  tests/fixtures/
  eval/golden.jsonl
  deploy/Dockerfile
  deploy/compose.yaml
  deploy/nginx.conf
  requirements*.txt
  .env.example
  .gitignore
  PROJECT_DECISIONS.md
```

위 트리는 작성할 파일 목록이다. 초기 단계에 모든 파일을 빈 껍데기로 채우고 구현 완료로 보지 않는다. 각 기능은 08장의 작업 단계와 시험을 함께 완성한다. Python 패키지 디렉터리에는 `__init__.py`를 둔다.

`.gitignore` 최소 내용:

```gitignore
.venv/
__pycache__/
.pytest_cache/
.ruff_cache/
.env
data/
reports/private/
*.log
```

시험 자료는 실제 고객자료 대신 `tests/fixtures/`의 합성 자료를 커밋한다. 모델 가중치, 업무 DB, 원본 업로드는 Git에 넣지 않는다.

## 5. 환경 변수의 단일 기준

다음을 `.env.example`에 저장한 뒤 `.env`로 복사한다. 모든 상대 경로는 `config.py`의 프로젝트 루트 기준으로 절대 경로화한다. 현재 작업 디렉터리에 따라 DB 위치가 달라져서는 안 된다.

```dotenv
APP_ENV=development
HOST=127.0.0.1
PORT=8000
BASE_URL=http://127.0.0.1:8000
DATA_DIR=./data
DATABASE_URL=sqlite:///./data/app.sqlite3
VECTOR_DB_PATH=./data/chroma
SESSION_SECRET=REPLACE_WITH_RANDOM_SECRET
COOKIE_SECURE=false
ALLOWED_ORIGINS=["http://127.0.0.1:8000"]
OLLAMA_URL=http://127.0.0.1:11434
CHAT_MODEL=qwen3:4b
EMBED_MODEL=bge-m3
EMBED_DIM=1024
OLLAMA_NUM_CTX=8192
MAX_OUTPUT_TOKENS=1024
CONNECT_TIMEOUT_SECONDS=5
READ_TIMEOUT_SECONDS=120
TOTAL_TIMEOUT_SECONDS=180
MAX_UPLOAD_BYTES=20971520
MAX_PAGES=200
CHUNK_MAX_CHARS=1000
CHUNK_OVERLAP_CHARS=150
EMBED_BATCH_SIZE=16
RETRIEVAL_FETCH_K=20
RETRIEVAL_TOP_K=4
CONTEXT_MAX_CHARS=6000
GENERATION_CONCURRENCY=1
MAX_WAITING=8
QUEUE_TIMEOUT_SECONDS=15
OCR_ENABLED=false
RERANKER_ENABLED=false
CONVERSATION_RETENTION_DAYS=30
```

`EMBED_DIM=1024`는 초기 기대값이다. 실제 `/api/embed` 결과가 일치하는지 검사하고 다른 모델이면 설정과 새 인덱스 manifest를 함께 변경한다. 일치하지 않는 벡터를 잘라서 맞추지 않는다.

`SESSION_SECRET`은 다음 명령의 결과를 로컬 `.env`에 저장한다. 키를 문서나 로그에 복사하지 않는다.

```powershell
.\.venv\Scripts\python.exe -c "import secrets; print(secrets.token_urlsafe(48))"
```

`Settings`는 Pydantic Settings로 작성하고 `env_file`을 프로젝트 루트의 `.env`로 고정한다. OS 환경변수가 `.env`보다 우선한다. 설정 이름의 오탈자는 시작 오류로 처리한다. `ALLOWED_ORIGINS`는 JSON 배열이며 쉼표 문자열과 혼용하지 않는다. 최소 검증:

- `0 <= CHUNK_OVERLAP_CHARS < CHUNK_MAX_CHARS`, `1 <= TOP_K <= FETCH_K`.
- 비밀키 기본 문구·빈 문자열 거부, 운영에서 `COOKIE_SECURE=true`, HTTPS `BASE_URL` 필수.
- 모델명·Ollama URL은 서버 설정만 허용. 사용자 요청으로 임의 URL을 선택하지 않음.
- 데이터 디렉터리 쓰기 권한, DB·벡터 경로의 예상 위치 확인.
- 선언되지 않은 OCR 엔진·거리 함수·실행 worker 수는 오류.

## 6. Ollama 준비와 실제 점검

Ollama를 설치·실행한다. 데스크톱 앱이 이미 서비스를 실행 중이면 또 `ollama serve`를 띄우지 않는다. 서비스가 없을 때만 별도 터미널에서 실행한다.

```powershell
ollama --version
ollama pull qwen3:4b
ollama pull bge-m3
ollama list
Invoke-RestMethod http://127.0.0.1:11434/api/tags
$embedPayload = @{ model='bge-m3'; input=@('테스트 문서'); truncate=$false } | ConvertTo-Json
$embedResult = Invoke-RestMethod -Method Post -Uri http://127.0.0.1:11434/api/embed -ContentType 'application/json; charset=utf-8' -Body ([Text.Encoding]::UTF8.GetBytes($embedPayload))
$embedResult.embeddings[0].Count
$chatPayload = @{ model='qwen3:4b'; messages=@(@{role='user';content='한국어로 안녕하세요라고 답해 주세요.'}); stream=$false; think=$false; options=@{num_ctx=8192;num_predict=64;temperature=0} } | ConvertTo-Json -Depth 6
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:11434/api/chat -ContentType 'application/json; charset=utf-8' -Body ([Text.Encoding]::UTF8.GetBytes($chatPayload))
```

`/api/tags`의 모델 digest, `ollama --version`, 응답 차원, 소요시간을 기록한다. 모델 목록 조회 성공과 실제 임베딩·생성 성공을 구별한다. API 형태는 공식 [embed](https://docs.ollama.com/api/embed), [chat](https://docs.ollama.com/api/chat) 문서를 기준으로 한다.

## 7. 첫 서버 구현

`main.py`에는 `create_app()`을 두고 설정→DB 연결→모델 어댑터→작업 루프 순으로 초기화한다. [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/)을 사용해 종료 시 작업 접수를 멈추고 HTTP 클라이언트와 실행 작업을 정리한다. 모듈 import 시 테이블 생성·모델 다운로드·대량 색인을 실행하지 않는다.

마이그레이션과 최소 health 라우터 구현 후:

```powershell
.\.venv\Scripts\python.exe -m alembic upgrade head
.\.venv\Scripts\python.exe -m uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000 --workers 1
```

설정의 `HOST`·`PORT`는 CLI 실행 래퍼에서 사용하도록 구현한다. Uvicorn을 직접 실행하면 위 명령의 인자가 우선하며 `.env`의 `PORT`만 바꿔도 Uvicorn이 바뀌는 것으로 가정하지 않는다. 개발 재시작은 수동으로 먼저 검증하고 `--reload`는 영속 작업 복구 시험 이후에 사용한다.

## 8. 환경 단계의 통과 조건

`pip check` 성공, 모델 두 개 실제 호출 성공, SQLite 외래키 활성화, 마이그레이션 버전 일치, `/health/live` 200, `/health/ready`의 의존성 상태 정상까지 확인한다. 빈 DB의 지식 수 0은 서버 준비 실패가 아니라 봇 자료 미준비 상태다. 구체적인 봇 질문에서는 별도 `index_not_ready` 처리를 한다.

`reports/environment.json`에는 설치 검증 일시·OS·CPU/GPU·RAM·잠금 해시·모델 digest·테스트 결과를 남긴다. 설치만 성공했을 때 검색 품질이나 서비스 처리량을 검증했다고 쓰지 않는다.
