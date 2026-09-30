# 09. 배포·보안·관측·백업·복구

[목차](./README.md) · 이전: [구현·인수](./08-build-and-acceptance.md) · 다음: [출처·확장](./10-reference-and-decisions.md)

## 1. 배포 전제

기본 운영 구성은 Linux 호스트, Nginx HTTPS, API 컨테이너 1개, Ollama 컨테이너 1개, 영속 data 볼륨이다. Windows 로컬 개발은 02장의 직접 실행을 사용한다. Docker Desktop CPU 실행은 기능 확인용이고 Linux GPU Compose는 드라이버·NVIDIA Container Toolkit이 준비된 별도 프로필로 검증한다.

아래 파일은 08장의 앱과 마이그레이션이 구현된 후 새 프로젝트의 `deploy/`에 작성할 예시다. 도메인·인증서·장치·이미지 digest는 실제 환경에 맞춰 정한다. 배포 명령이 기존 네 앱의 운영 프로세스를 재시작하거나 종료하지 않게 프로젝트명·포트를 분리한다.

## 2. 이미지 만들기

프로젝트 루트 `.dockerignore`:

```text
.git
.venv
.env
data
reports
tests
__pycache__
```

`deploy/Dockerfile`:

```dockerfile
FROM python:3.11-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements-linux.lock.txt /app/requirements-linux.lock.txt
RUN pip install --no-cache-dir --require-hashes -r requirements-linux.lock.txt
RUN useradd --uid 10001 --create-home appuser && mkdir -p /app/data && chown appuser:appuser /app/data
COPY --chown=appuser:appuser app /app/app
COPY --chown=appuser:appuser migrations /app/migrations
COPY --chown=appuser:appuser alembic.ini /app/alembic.ini
USER 10001:10001
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "app.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

운영판은 검증 후 FROM 이미지를 실제 digest로 고정한다. `requirements-linux.lock.txt`는 02장에서 Linux 환경으로 생성한 설치 가능 잠금 파일이어야 한다. 정적 파일은 `app/static`에 있으므로 앱과 함께 복사된다. 이미지에 업무 데이터·`.env`·모델 키를 넣지 않는다.

## 3. Compose 예시

`deploy/compose.yaml`:

```yaml
name: base-chatbot
services:
  ollama:
    image: ${OLLAMA_IMAGE:?Set OLLAMA_IMAGE to a tested image tag or digest}
    restart: unless-stopped
    volumes:
      - ollama-data:/root/.ollama
  api:
    build:
      context: ..
      dockerfile: deploy/Dockerfile
    env_file:
      - ../.env
    environment:
      APP_ENV: production
      HOST: 0.0.0.0
      PORT: '8000'
      DATA_DIR: /app/data
      DATABASE_URL: sqlite:////app/data/app.sqlite3
      VECTOR_DB_PATH: /app/data/chroma
      OLLAMA_URL: http://ollama:11434
      COOKIE_SECURE: 'true'
    ports:
      - '127.0.0.1:8000:8000'
    volumes:
      - app-data:/app/data
    depends_on:
      - ollama
    restart: unless-stopped
    stop_grace_period: 45s
    healthcheck:
      test: ['CMD', 'python', '-c', 'import urllib.request; urllib.request.urlopen("http://127.0.0.1:8000/health/live", timeout=3)']
      interval: 30s
      timeout: 5s
      retries: 3
volumes:
  app-data:
  ollama-data:
```

`OLLAMA_IMAGE`는 배포 쉘 환경변수로 설정한다. 프로젝트 `.env`는 Pydantic 앱 설정 전용이며 Compose 전용 변수를 넣으면 `extra=forbid` 설정 검증에 걸릴 수 있다. 이미지 태그는 배포 시 검증하고 digest를 `reports/environment.json`에 기록한다. Compose의 depends_on은 모델 준비를 보장하지 않는다.

새 named volume은 이미지의 `/app/data` 소유권을 확인한다. 기존 볼륨을 사용한다면 데이터 백업 후 UID 10001 쓰기 권한을 확인하고 임의로 전체 프로젝트 권한을 바꾸지 않는다. SQLite·Chroma를 네트워크 공유 파일시스템에 두지 않는다.

최초 배포 순서(Linux 프로젝트 루트):

```bash
docker compose -f deploy/compose.yaml config
docker compose -f deploy/compose.yaml build api
docker compose -f deploy/compose.yaml up -d ollama
docker compose -f deploy/compose.yaml exec ollama ollama pull qwen3:4b
docker compose -f deploy/compose.yaml exec ollama ollama pull bge-m3
docker compose -f deploy/compose.yaml run --rm api python -m alembic upgrade head
docker compose -f deploy/compose.yaml run --rm api python -m app.cli create-admin --email admin@example.test
docker compose -f deploy/compose.yaml run --rm api python -m app.cli doctor
docker compose -f deploy/compose.yaml up -d api
```

`docker compose config`는 실제 환경변수 값이 출력될 수 있으므로 결과를 공개 로그에 첨부하지 않는다. BASE_URL과 ALLOWED_ORIGINS는 배포 전 `.env`에서 실제 HTTPS 도메인으로 바꾼다. 비밀번호 입력용 create-admin은 대화형 터미널에서 실행한다. API 시작 전 마이그레이션·bootstrap 명령을 실행하므로 동일 Chroma 경로를 여러 프로세스가 동시에 열지 않는다.

GPU가 필요하면 Ollama 서비스에 GPU device 예약 설정을 별도 override로 추가한다. 컨테이너 안의 모델 실행이 실제 GPU를 쓰는지 확인하고 CPU에서도 성공했던 기능 시험을 반복한다. Windows와 Linux에 같은 GPU 설정이 통용된다고 가정하지 않는다.

## 4. Nginx와 SSE

Nginx는 호스트에서 API의 loopback 포트로 연결하는 예시다. 실제 도메인과 인증서 경로를 넣어야 한다.

```nginx
server {
    listen 443 ssl;
    server_name chat.example.com;
    ssl_certificate /etc/ssl/chat/fullchain.pem;
    ssl_certificate_key /etc/ssl/chat/privkey.pem;
    client_max_body_size 22m;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 240s;
        proxy_send_timeout 240s;
    }
}
```

80 포트는 443으로 리다이렉트하고 인증서 갱신을 구성한다. API 컨테이너는 로컬 포트만 공개하고 Ollama 포트는 인터넷에 공개하지 않는다. 앱의 실제 파일 제한은 20MiB, 프록시 22MiB는 multipart 오버헤드를 포함한다.

신뢰할 프록시 IP/네트워크만 forwarded 헤더를 적용하도록 Uvicorn·앱 설정을 맞춘다. 어떤 인터넷 요청의 X-Real-IP도 그대로 신뢰하면 속도 제한을 우회할 수 있다. Docker에서 API가 보는 호스트 주소를 확인해 신뢰 목록에 지정한다. 프록시가 IP 헤더를 덮어쓰는지 실제 요청으로 시험한다.

프록시 경유 curl의 상태 이벤트·heartbeat·done 수신 시점을 검사한다. 200 응답과 최종 본문 표시만으로 SSE 정상 여부를 판정하지 않는다.

## 5. 인증·권한·입력 보안 구현

비밀번호는 Argon2id 등 검증된 라이브러리로 해시한다. 직접 암호 알고리즘을 구현하지 않는다. 초기 계정 생성 시 충분한 길이의 비밀번호를 요구하고 로그인 실패 메시지는 계정 존재 여부를 드러내지 않는다.

관리 세션·방문 세션은 256비트 이상 난수 토큰을 발급하고 DB에는 `HMAC-SHA256(SESSION_SECRET, "session:" + token)`만 저장한다. 쿠키는 HttpOnly, SameSite=Lax, 운영 Secure, Path=/를 설정한다. 관리 세션 만료는 기본 12시간, 방문 세션은 24시간이다. 로그아웃·계정 비활성화 시 DB 세션을 취소한다. CSRF 토큰은 07장의 별도 입력 영역으로 파생하고 해시를 저장한다. 유효성은 상수시간 비교한다. 비밀키 교체 시 기존 세션은 모두 만료시키고 재로그인을 요구한다.

권한 검사는 라우터 dependency와 서비스 계층에 둔다. 전달받은 document_id·version_id·job_id·conversation_id·run_id가 해당 봇에 속하는지 모두 확인한다. 목록·검색·출처·원본 다운로드·삭제·재시도도 예외가 아니다.

초기 속도 제한: 로그인 IP당 분당 5회, 공개 채팅 IP당 분당 20회, 봇·방문 세션당 10초에 1회, 업로드 사용자당 분당 5회. NAT 사용자 영향과 실제 사용량을 측정해 조정한다. 제한 상태는 단일 프로세스 메모리에 둘 수 있으나 다중 서버로 확장하면 Redis 등 공유 저장소가 필요하다.

업로드 크기·파서 제한·ZIP 검사·경로 이탈 차단은 04장을 적용한다. 임의 URL 다운로드·원격 문서 크롤링은 기본 API에 넣지 않는다. 추가하면 SSRF·리다이렉트·사설망·허용 도메인 정책을 별도 설계한다.

웹 출력은 기본 텍스트 렌더링이다. 로컬 정적 자산을 쓰고 CSP에서 `default-src 'self'`, 허용하지 않은 frame/스크립트/연결을 제한한다. CSP·Origin·CSRF·소유권 검사는 서로 대체하지 않는다.

## 6. health와 관측

| 경로 | 공개 범위 | 판정 |
|---|---|---|
| GET `/health/live` | 최소 공개 | 이벤트 루프·프로세스 응답, 200 |
| GET `/health/ready` | 최소 공개 | DB·schema·저장소·모델 준비, 실패 503 |
| offline CLI `doctor` | OS/배포 관리자 | 모델·인덱스·저장소 상세 검사 |
| GET `/api/v1/bots/{bot_id}/health` | 소유자 | 해당 봇 active 세대·청크 수·최근 작업 |

기본판에는 시스템 상세 관리 HTTP API를 노출하지 않는다. 전체 큐·자원 현황은 접근이 제한된 운영 로그·모니터링과 offline CLI로 확인한다. 향후 운영 API를 추가할 때는 별도 운영 역할 또는 키를 설계해야 하며 일반 봇 소유자 계정 전체에 시스템 상세를 공개하지 않는다.

준비 검사에는 실제 임베딩·짧은 생성 probe의 최근 성공 결과를 캐시한다. 매 5초 health 요청마다 큰 모델을 로드하지 않는다. 초기·복구 후 실제 probe, 이후 60초 캐시를 시작점으로 한다. 개별 비공개 봇의 빈 자료나 색인 중 상태가 앱 전체 ready를 실패시키지는 않는다. 활성 인덱스 손상은 해당 봇 질의를 차단하고 관리 상세에 표시한다.

JSON 로그 필수 필드: timestamp, level, request_id, run_id/job_id, bot_id, stage, duration_ms, error_code. 질문 전문 대신 마스킹본 또는 해시를 사용하고 디버그에서도 비밀번호·세션·원본 파일 전문을 남기지 않는다.

수집할 지표: 요청 수·오류율, 임베딩/검색/생성 시간, 큐 길이·대기 시간, 취소 건수, 모델 429/5xx, ingestion 성공/실패, 청크·벡터 수 차이, 디스크 여유, 미답변율, 인용 검증 실패율. 미답변율 상승은 자료 부족·전처리 문제·검색 변경을 나누어 조사한다.

## 7. 백업 정책

기본 목표는 RPO 24시간, RTO 2시간으로 시작하고 운영자가 실제 요구에 맞춰 확정한다. 백업 대상은 같은 시점의 업무 DB, 원본, 추출·청크·assets, manifests, Chroma, 환경·마이그레이션·모델 digest 기록이다. 세션 비밀 등은 암호화한 별도 비밀 저장소로 보관한다. 모델 가중치는 다시 내려받을 수 있는지와 보존 비용에 따라 별도 백업한다.

기본판은 **짧은 정지 백업**을 사용한다.

1. 점검 안내 후 새 요청·작업 접수를 멈춘다.
2. 실행 중 작업·질의를 종료 또는 취소하고 API를 완전히 정지한다.
3. SQLite connection·Chroma 핸들이 모두 닫힌 상태에서 전용 backup 명령을 실행한다.
4. SQLite는 backup API로 사본을 생성하고 `PRAGMA integrity_check`를 실행한다. DB가 살아 있는 상태에서 app.sqlite3만 복사하여 WAL을 빠뜨리지 않는다.
5. raw/derived/manifests/chroma를 같은 스냅샷 디렉터리에 복사한다.
6. 파일별 상대 경로·크기·SHA-256, 환경 기록, 생성 시각을 backup manifest로 쓴다.
7. 새 백업 경로에서 verify-backup을 실행하고 암호화·오프호스트 사본을 만든다.
8. API를 다시 시작하고 ready 및 대표 질문을 확인한다.

초기 보존 예시는 일간 7개·주간 4개다. 고객 데이터 보존 요구와 저장 용량에 맞춰 확정한다. 백업이 성공했다는 로그만으로 복구 가능하다고 판단하지 않는다.

## 8. 복원·롤백 절차

1. 복원 대상 시점과 앱 이미지·migration head·모델 digest를 선택한다.
2. 별도 새 디렉터리/볼륨에 백업을 복원한다. 실행 중인 원본을 덮어쓰지 않는다.
3. 해시·SQLite 무결성·원본과 청크 ID·manifest를 검사한다.
4. 동일 모델을 준비한다. 다른 임베딩 모델만 사용 가능하면 새 세대로 재색인한다.
5. API 정지 상태에서 doctor·verify-index를 실행한다. 벡터가 없으면 raw/derived에서 다시 구축한다.
6. 별도 로컬 포트로 기동하고 권한·검색·출처·미답변·실제 생성 시험을 수행한다.
7. 검증 후 유지보수 창에서 트래픽을 전환한다. 실패하면 이전 볼륨과 앱 버전으로 되돌린다.

인덱스만 롤백할 때 과거 세대의 문서가 현재 삭제·비공개된 것은 아닌지 검사한다. 임베딩 fingerprint가 현재 모델과 일치해야 한다. 검증되지 않은 과거 포인터를 단순 대입하지 않는다. DB schema 변경이 포함된 앱 롤백은 앱 이미지만 되돌려서는 안 된다.

## 9. 배포 변경 순서

코드·설정·prompt·청킹·모델 변경을 구분한다. prompt나 검색 계수는 golden set을 다시 실행한다. 청킹·임베딩 변경은 새 인덱스 세대를 만든다. 생성 모델만 변경해도 인용·미답변·한국어 품질과 지연을 다시 평가한다.

변경 전 백업→스테이징 설치·평가→필요한 마이그레이션→새 세대 검증→제한된 실제 질의 확인→전환 순으로 진행한다. 배포 기록에는 앱 버전, 설정 차이, 잠금 해시, 모델 digest, 평가 결과, 롤백 대상을 남긴다.

## 10. 문제 해결 표

| 증상 | 먼저 확인 | 조치 |
|---|---|---|
| 화면은 뜨는데 답변 불가 | ready 상세·봇 active 세대·모델 probe | 준비 안 된 의존성을 복구 |
| 문서 업로드 후 검색 안 됨 | 작업 state/stage·검토·서비스 버전 | 승인/실패 수정 후 재시도 |
| 잘못된 숫자·순서 | extracted.json·원본 표·OCR | 전처리 수정 및 새 버전 색인 |
| 자료에 있는데 검색 실패 | retrieve 후보·exact term·청킹 | 검색 단계 개선 후 평가 |
| 벡터 차원 오류 | 모델 digest·EMBED_DIM·manifest | 동일 모델 복원 또는 전체 새 세대 |
| 답변에 없는 출처 | prompt evidence와 sources ID | 최종 검증·출처 조립 버그 수정 |
| 삭제 문서가 검색됨 | tombstone filter·과거 세대 캐시 | 즉시 노출 차단 후 인덱스 정리 |
| SSE가 끝에 한꺼번에 옴 | 기본 버퍼 답변인지, status도 지연되는지 | 기본 답변 버퍼는 정상, status 지연은 프록시 확인 |
| SQLite locked | 긴 쓰기 트랜잭션·중복 프로세스 | 네트워크 호출 밖에서 commit, writer 단일화 |
| 429 반복 | 요청률·큐·모델 지연 | 제한 우회보다 병목 측정·용량 조정 |
| GPU OOM | 문맥·batch·동시성·OCR 공존 | batch/문맥 축소, 서비스 자원 분리 |
| 재시작 뒤 작업 멈춤 | lease 만료·작업 복구 루프 | 재시도 기록·중간 세대 정리 |

운영 완료는 HTTPS 접속, 권한 격리, 실제 문서 검색·답변, 프록시 SSE, 장애 복구, 백업 복원 시험까지 통과한 상태다.
