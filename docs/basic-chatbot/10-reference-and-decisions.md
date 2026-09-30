# 10. 참고 자료·설계 결정·확장 지침

[목차](./README.md) · 이전: [운영](./09-operations.md)

## 1. 문서의 근거와 해석

다음 네 문서는 기존 시스템의 동작을 설명하는 1차 입력이다. 가이드에 기록된 커밋은 해당 가이드의 기준이며 이번 작업에서 로컬 스냅샷이 정확히 같은 커밋인지 새로 확인한 것은 아니다.

| 입력 문서 | 기록된 기준 | 신규 설계에 사용한 내용 |
|---|---|---|
| [DEVELOPER_GUIDE_1](../DEVELOPER_GUIDE_1.md) | LogisticsKorea `5b49a31` | 문서→임베딩→검색→생성, hybrid 검색, 스트림·준비 상태 |
| [DEVELOPER_GUIDE_2](../DEVELOPER_GUIDE_2.md) | LSE `fe1485cfe2d24707a482e1d2b43fccf18b0743a5` | 구조화 매뉴얼, 표·그림, 정확 명칭 보강, 인덱스 manifest·검증 |
| [DEVELOPER_GUIDE_3](../DEVELOPER_GUIDE_3.md) | mysbiz `f616318` | 문서/폴더 범위, OCR·변환, 다중 서비스 운영의 경계 |
| [DEVELOPER_GUIDE_4](../DEVELOPER_GUIDE_4.md) | 김비서 `8eed86f` | 봇 관리, Q&A, 봇별 Chroma, 공개 채팅, 미답변 개선 |

LSE 가이드의 Base64 내장 자료는 일반 설계 본문과 구별한다. 새 시스템의 RAG 입력으로 개발 가이드와 그 내장 ZIP을 자동 재귀 색인하지 않는다. 고객 지식, 개발 문서, 코드 참고 자료는 서로 다른 목적의 데이터셋이다.

## 2. 확인한 로컬 소스와 활용 범위

아래 절대 경로는 이번 작성 환경에서 확인한 소스다. 다른 개발자의 PC에서는 해당 저장소 위치에 맞춰 찾는다. 파일 전체를 그대로 가져오라는 지시가 아니라 필요한 구현을 대조하는 출처다.

| 소스 | 확인 내용 | 새 설계의 보완 |
|---|---|---|
| `C:\Projects\secretaryKim\app\services\extractor.py` | PDF/DOCX/CSV/텍스트 추출, 문자 청킹 | 원래 문서 순서·위치·최종 청크 상한 유지 |
| `C:\Projects\secretaryKim\app\services\knowledge.py` | Q&A/본문 해시 중복 제거, DB·벡터 쓰기 | 원본·버전 보존, 세대 활성화·실패 복구 |
| `C:\Projects\secretaryKim\app\services\vector_store.py` | 봇별 컬렉션, cosine, upsert/search | 검색 중 빈 컬렉션 자동 생성 금지 |
| `C:\Projects\secretaryKim\app\services\retriever.py` | 0.65 벡터+0.35 키워드+Q&A 0.10 | exact 후보 보강, 점수와 정답 확률 구분 |
| `C:\Projects\secretaryKim\app\services\ollama_client.py` | embed/chat 어댑터, 재시도, 스트림 | 입력 truncate 금지, digest·차원·deadline 검사 |
| `C:\Projects\secretaryKim\requirements.txt` | 기존 Chroma 0.6.3 등 버전 | 신규 1.x 설정과 기존 API 구별 |
| `C:\Projects\LSE\app\manual\retrieval.py` | 한글 명칭/영문 식별자 검색, 조각 재정렬 | 후보 합칠 때 exact 정보 보존, 범용 정책화 |
| `C:\Projects\LSE\tools\index_rag_corpus.py` | 새 대상에 구축, incomplete 상태, manifest, 차원·건수·smoke | ID 집합 검증·봇 세대 활성화·revision 충돌 검사 |
| `C:\Projects\LSE\preprocess\build_ixp3_chunks.py` | 목차·표·그림 추출 함수와 문서별 정책 | 특정 PDF 좌표를 범용 기본값으로 사용하지 않음 |
| `C:\Projects\LSE\requirements.txt` | FastAPI 0.127.0, Chroma 1.3.7 등의 기록 | 신규 의존성 후보 범위와 잠금 절차 |
| `C:\Projects\mysbiz\app\utils\document_processor.py` | 형식별 추출·OCR·HWP 변환 진입점 | 어댑터 격리, 지원 확인·품질 승인 |
| `C:\Projects\mysbiz\app\utils\chunking.py` | Q&A 구조를 인지하는 분할 경로 | Q&A 질문 반복·원본 위치 보존 |

소스 확인은 정적 읽기와 함수·설정 대조다. 기존 앱을 기동하거나 운영 DB·비밀 설정·모델을 가져와 실행하지 않았다. 일부 큰 파일은 관련 함수·진입점 중심으로 확인했으며 전체 코드 감사를 수행했다는 의미가 아니다.

## 3. 기존 시스템을 합치지 않고 통일한 결정

| 결정 | 선택 이유 | 바꿀 때 필요한 작업 |
|---|---|---|
| FastAPI가 화면도 제공 | 가장 작은 실행 구성, 프록시 주소 혼선 감소 | 별도 UI 도입 시 API·CORS·SSE 유지 |
| SQLite+Chroma | 기존 봇·지식 흐름과 친숙하며 로컬 시작 용이 | 서버형 DB·벡터 저장소는 마이그레이션·동시성 시험 |
| 원본/추출/청크 모두 보존 | 파서·OCR·모델 변경 때 재구축 가능 | 개인정보·용량별 보존 정책 설정 |
| 문서 UUID·불변 버전 | 같은 파일명 충돌과 과거 출처 혼동 방지 | 외부 데이터셋도 이 ID 체계로 매핑 |
| 봇별 전체 인덱스 세대 | SQLite/벡터 비원자성의 공개 영향을 줄임 | 대규모에서는 증분 처리·membership 필터 설계 |
| 영속 작업+단일 API worker | 재시작 복구와 로컬 큐 일관성 확보 | 공유 큐·모델 슬롯·서버형 Chroma 도입 |
| 검증 후 SSE 전달 | 미답변·인용·저장 결과를 확정한 뒤 표시 | 실시간 토큰은 사후 수정·partial 계약 추가 |
| 기본 OCR 비활성 | GPU·변환기 없는 환경에서도 구축 시작 | 엔진·모델·장치·검토 UI를 추가한 뒤 지원 |
| 공개 링크 기본 비활성 | 소유자가 지식 공개 범위를 결정 | 조직 ACL·로그인 공유는 별도 기능 |
| 모델 digest·manifest 기록 | 이름이 같은 다른 모델 및 재색인 혼동 방지 | 모델 변경 시 새 평가·세대 작성 |

다음 항목은 기존 코드에 이미 완성되어 있는 기능으로 주장하지 않는다: Alembic 마이그레이션, 전처리 위치 공통 DTO, 세대별 원자적 활성화, 영속 작업 lease, 최종 JSON 답변 검증, 서버 방문 세션, request_id 멱등, 통합 인수 기준. 모두 신규 시스템의 요구사항이다.

## 4. 공식 문서로 확인한 외부 계약

2026-09-18 확인. 링크의 최신 예시와 설치한 패키지 버전이 다르면 실제 잠금 버전에 맞는 API를 확인하고 계약 시험을 추가한다.

- [Ollama embed API](https://docs.ollama.com/api/embed): input 배열, embeddings 배열, truncate 옵션.
- [Ollama chat API](https://docs.ollama.com/api/chat): messages, format schema, stream, done, 모델 응답 필드.
- [Chroma 컬렉션 설정](https://docs.trychroma.com/docs/collections/configure): 생성 시 configuration으로 cosine 지정.
- [Chroma Python client](https://docs.trychroma.com/reference/python): PersistentClient 인터페이스.
- [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/): 시작·종료 자원 관리.
- [pypdf 텍스트 추출](https://pypdf.readthedocs.io/en/stable/user/extract-text.html): 텍스트 추출의 한계, 스캔과 OCR 구분.
- [Ollama qwen3:4b](https://ollama.com/library/qwen3:4b), [bge-m3](https://ollama.com/library/bge-m3): 기본 개발 예시 모델의 등록 여부.

외부 문서를 조회했다는 사실은 이 문서의 전체 dependency 조합을 설치·보안 검증했다는 뜻이 아니다. 모델 라이선스, 장치별 성능, 한국어 답변 품질, 운영 인증서·방화벽은 실제 구축 시 확인한다.

## 5. 확장 순서

| 필요 | 권장 확장 | 유지할 공통 계약 |
|---|---|---|
| 스캔·사진·한글 문서 | OCR/HWPX/변환 어댑터, 검토 UI | ExtractedDocument·위치·원본 버전 |
| 표의 세부 수치·그림 | 행 단위 검색, asset 카탈로그 | chunk_id·evidence_id·출처 권한 |
| 긴 제품 코드·한국어 검색 | exact 사전, BM25, 형태소 분석 | 봇·세대 필터, 평가 데이터 |
| 답변 정확도 부족 | cross encoder, 모델 비교 | 점수 분리·인용 검사·미답변 |
| 외부 생성 API | provider 어댑터, 키·비용·시간 제한 | AnswerDraft/AnswerResult, 개인정보 정책 |
| 여러 운영 서버 | PostgreSQL, 공유 작업 큐, 서버형 벡터 DB | 영속 상태·멱등·전역 슬롯·세대 |
| 회사별 사용자 권한 | tenant·membership·document ACL | 권한을 검색 전후 모두 적용 |
| 전문 코드 생성 | 도메인 엔진·정적 검증·승인 절차 | 일반 답변과 모드 분리, 도구 실행 금지 기본값 |

기능을 추가할 때 `PROJECT_DECISIONS.md`에 문제·대안·선택·영향·인수시험을 적는다. 모든 도메인 앱의 기능을 한 번에 넣는 것보다 기본 RAG의 자료·권한·복구 계약을 유지하는 것을 우선한다.

## 6. 문서 관리

API와 DTO 변경은 해당 장, 예시, golden set, UI 파서를 같은 변경 단위로 갱신한다. 설정 기본값의 단일 기준은 02장이고 도메인별 재정의는 별도 프로젝트 결정 파일에 둔다. 구축 경험이 생기면 “계획값”을 지우는 대신 측정 환경과 결과를 추가한다.

이번 산출물은 Markdown 설계·구현 지침이다. 실제 신규 앱 구현, dependency 설치, 모델 다운로드·추론, DB 생성, 브라우저 종단 간 시험은 수행 범위에 포함하지 않았다. 문서 구조·링크·예제의 정적 점검과 작은 독립 예제 검증 결과는 함께 제공하는 검증 기록을 따른다.
