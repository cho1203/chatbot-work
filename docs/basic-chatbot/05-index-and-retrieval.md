# 05. 임베딩·인덱스 구축·검색

[목차](./README.md) · 이전: [전처리](./04-document-pipeline.md) · 다음: [답변](./06-answer-and-conversation.md)

## 1. 임베딩 어댑터

`embed(texts: list[str]) -> list[list[float]]`를 기본 인터페이스로 한다. 문서·질의는 같은 모델, digest, 전처리, 정규화, 차원을 사용한다. 모델명이 같아도 digest가 바뀌면 기존 인덱스와 다른 의미 공간일 수 있으므로 새 세대를 만든다.

Ollama 요청:

```json
{"model":"bge-m3","input":["문서 제목\n청크 본문"],"truncate":false}
```

`truncate=false`로 길이 초과를 오류로 받는다. 벡터를 만들기 전에 문서 끝을 자동으로 잘라버리는 동작을 허용하지 않는다. 공식 [embed API](https://docs.ollama.com/api/embed)를 따른다.

응답 검사 순서:

1. HTTP 성공과 `embeddings` 배열 존재 확인.
2. 입력 수와 벡터 수 동일, 모든 벡터 차원 동일 및 `EMBED_DIM` 일치.
3. 모든 값이 유한한 숫자이며 벡터 norm이 0보다 큰지 확인.
4. 앱에서 L2 정규화하고 이 정책을 manifest에 기록. 질의에도 같은 정규화 적용.
5. 빈 입력은 빈 배열로 반환하되 비어 있는 개별 텍스트는 청킹 오류로 거부.

배치는 기본 16개다. 메모리 부족이면 배치를 줄여 재시도하되 모델 차원·본문을 바꾸어 성공처럼 처리하지 않는다. 연결 실패·429·일시적 5xx만 1초·3초·6초+jitter로 재시도하고 총 기한을 지킨다. 400 입력 오류·모델 미등록은 즉시 설정 오류로 종료한다.

캐시를 구현하면 키는 `bot_id + model_digest + normalization_policy + embed_text_sha256`이다. 캐시에 모델명만 쓰지 않는다. 캐시는 선택 사항이고 삭제·보존 정책을 따르며 다른 봇의 본문을 결과로 반환하지 않는다.

## 2. Chroma 어댑터

기본 생성 예시(서비스 내부에서 사용):

```python
import chromadb

def create_generation(path: str, collection_name: str):
    client = chromadb.PersistentClient(path=path)
    return client.create_collection(
        name=collection_name,
        embedding_function=None,
        configuration={"hnsw": {"space": "cosine"}},
    )
```

Chroma의 기본 거리 설정을 가정하지 않고 cosine을 명시한다. 새 문서는 Chroma 1.x의 `configuration` 형식을 기준으로 하며, 기존 김비서의 0.6.x 메타데이터 설정을 그대로 복사하지 않는다. [Chroma 컬렉션 설정](https://docs.trychroma.com/docs/collections/configure).

생성 시만 `create_collection`을 쓴다. 질의는 DB manifest에 등록된 이름을 `get_collection`으로 가져오며 없으면 `index_missing` 장애다. 조회 코드가 `get_or_create_collection`으로 빈 컬렉션을 만들어 장애를 숨기지 않게 한다. 앱이 계산한 embeddings만 사용하고 Chroma 기본 임베딩 모델을 자동 로드하지 않는다.

## 3. 전체 세대 구축 알고리즘

기본판은 구현·복구를 단순하게 하기 위해 봇별 전체 스냅샷을 만든다. 벡터 캐시로 계산을 재사용할 수 있지만 활성 컬렉션을 직접 수정하지 않는다.

1. 짧은 DB 트랜잭션에서 봇의 `corpus_revision`, 삭제되지 않은 문서의 approved 희망 버전, 청크 ID 목록을 읽는다.
2. `index_generations(status=building)` 행을 만들고 새로운 컬렉션을 생성한다. 동일 이름이 이미 있으면 새 ID를 사용한다.
3. 구성 청크를 정렬하여 `index_members`로 기록한다. 청크가 다른 봇·버전 소속인지 검사한다.
4. 배치별 임베딩→upsert를 수행한다. 벡터 수·차원·유한성 검사 후에만 적재한다.
5. 전체 expected ID 집합과 실제 ID 집합을 비교한다. 같은 개수만으로 통과하지 않는다.
6. 비어 있지 않은 세대는 대표 청크 자기 검색과 정답 질의 smoke test를 수행한다. `n_results=min(k,count)`를 사용한다.
7. manifest를 임시 파일로 작성하고 검증된 위치로 rename한 뒤 세대를 validated로 만든다.
8. 활성화 직전 revision·모델 digest·검토 상태를 재검사한다. 중간에 변경되면 활성화하지 않고 최신 작업을 예약한다.
9. 하나의 SQLite 트랜잭션에서 새 세대를 active, 이전 것을 retired, 봇 포인터를 새 ID로 바꾼다.
10. 성공 상태를 저장한다. 이전 컬렉션은 진행 중 요청과 보존 기간을 확인한 뒤 정리한다.

문서가 모두 삭제된 경우 청크 수 0인 정상 빈 세대를 활성화할 수 있다. 이 세대의 검색은 정상적인 빈 결과이고 새 답변은 미답변이다. 컬렉션 누락·모델 불일치는 빈 결과와 구별되는 장애다.

중간 실패 시 이전 active는 유지하고 새 세대는 failed로 남긴다. 시작 후 앱이 죽어 컬렉션만 남으면 manifest·DB를 비교해 고아 후보로 분류한다. 재시도는 새 세대를 만들며 운영자가 원인을 볼 수 있어야 한다.

## 4. manifest 필수 필드

```json
{
  "schema_version": "1",
  "generation_id": "세대 UUID",
  "bot_id": "봇 UUID",
  "collection_name": "서버에서 생성한 컬렉션 이름",
  "corpus_revision": 7,
  "embedding": {
    "provider": "ollama",
    "model": "bge-m3",
    "digest": "실제 모델 digest",
    "dimension": 1024,
    "normalization": "l2-app-v1",
    "distance": "cosine",
    "document_prefix": "title-section-v1",
    "query_prefix": "none"
  },
  "chunk_policy": "chunk-v1",
  "source_versions": ["문서 버전 UUID"],
  "chunk_count": 12,
  "sorted_chunk_ids_sha256": "정렬된 ID 목록 해시",
  "corpus_sha256": "청크 ID와 embed_text 해시 목록의 해시",
  "dependency_lock_sha256": "잠금 파일 해시",
  "built_at": "2026-09-18T00:00:00Z",
  "checks": {"id_set_match": true, "dimension_match": true, "smoke_passed": true}
}
```

예시 해시 문구는 실제 빌드에서 계산한다. `embedding_fingerprint`는 embedding 객체의 정렬된 정규 JSON을 SHA-256한 값이다. 앱은 준비 검사 및 검색 시작 때 활성 세대와 현재 embedder fingerprint를 대조한다. 다르면 503과 재색인 안내를 반환한다.

## 5. 검색 파이프라인

검색 API는 생성 모델 없이도 사용 가능해야 한다. 초기 문제를 전처리·검색·생성으로 나눠 진단하기 위한 핵심 도구다.

1. 서버에서 bot_id와 접근 주체를 결정한다. 활성 generation_id를 한 번 읽어 요청에 고정한다.
2. 질문의 공백만 정리하고 고유명·숫자·단위를 보존한다. 2,000자 이상이면 입력 검증으로 거부한다.
3. 같은 embedder로 질의 벡터를 만든다. Chroma에서 상위 20개 dense 후보를 받는다.
4. 따옴표 안 명칭, 제품 코드, 오류 코드, 영문 식별자에서 최대 4개의 exact term을 추출한다. 일반 단어·문장 전체를 exact term으로 쓰지 않는다.
5. 같은 봇·세대의 청크에 한정하여 해당 명칭이 실제 등장하는 후보를 최대 20개 보강한다. SQL을 쓴다면 파라미터 바인딩과 LIKE의 `%`, `_` 이스케이프를 적용한다.
6. dense/exact 후보를 chunk_id로 합치고 두 경로의 증거를 보존한다. 이미 dense로 들어온 청크가 exact에도 있으면 `exact_match=true`로 갱신한다.
7. SQLite 조인으로 활성 세대 membership, 문서 tombstone, 봇 공개 상태를 검사한다. 잘못된 ID는 반환하지 않는다.
8. 키워드 겹침과 Q&A 가산점으로 재정렬하고 상위 4개를 선택한다. 청크·출처가 과도하게 반복되면 같은 문서의 인접 청크를 묶되 사용 위치를 보존한다.
9. 문맥 예산 안의 실제 사용 근거만 answerer로 전달한다.

봇별 컬렉션만 믿고 권한 검사를 생략하지 않는다. 결과 점수의 의미를 혼합하지 않도록 아래 필드를 독립적으로 유지한다.

| 필드 | 의미 |
|---|---|
| distance | Chroma cosine distance 원값 |
| similarity | `1 - distance`, 이론상 -1~1 |
| keyword_overlap | 질의 검색 토큰 중 본문에도 있는 비율, 0~1 |
| exact_match | 명확한 식별자/인용 명칭의 경계 일치 여부 |
| rank_score | 아래 기본식 또는 reranker 결과 |
| retrieval_methods | dense/exact/bm25 등 후보 유입 경로 |

기본 점수:

```text
rank_score = 0.65 * max(0, similarity)
           + 0.35 * keyword_overlap
           + (0.10 if kind == "qa" and keyword_overlap > 0 else 0)
```

이 계수는 김비서의 단순 재정렬 방식에서 가져온 시작점이다. 정답 확률이 아니며 1을 넘을 수 있다. `similarity=0.65`를 “65% 정확”으로 표시하지 않는다. exact_match는 후보 보강에 사용하고 무조건 정답으로 채택하는 신호로 쓰지 않는다.

한국어 기본 검색 토큰은 `[0-9A-Za-z가-힣_]+`로 추출하되 한 글자 단위 질문·제품 기호 등 손실을 fixture로 확인한다. 조사 때문에 겹침이 낮은 경우 벡터 검색이 보완한다. 실제 실패가 누적되면 형태소 분석·별칭 사전을 추가하고 그 버전을 기록한다.

## 6. Evidence 출력 계약

```json
{
  "generation_id": "세대 UUID",
  "query": "고객센터 운영 시간이 어떻게 되나요?",
  "items": [
    {
      "evidence_id": "S1",
      "chunk_id": "청크 UUID",
      "document_id": "문서 UUID",
      "version_id": "버전 UUID",
      "title": "고객센터",
      "text": "고객센터는 평일 09:00~18:00에 운영합니다.",
      "location": {"page_start": 1, "page_end": 1},
      "similarity": 0.78,
      "rank_score": 0.66,
      "exact_match": false,
      "retrieval_methods": ["dense"]
    }
  ],
  "timing_ms": {"embedding": 120, "search": 15, "rerank": 2}
}
```

위 점수·시간은 응답 형식 예시다. 공개 응답에는 내부 경로·원본 DB 키·민감한 검색 디버그 정보를 필요 이상 제공하지 않는다. 상세 점수는 관리자 검색 시험 화면에서만 볼 수 있다.

## 7. 문맥 조립과 근거 부족

기본 문맥 상한은 6,000자다. 시스템 프롬프트·현재 질문·최근 대화·출력 예약 공간을 포함해 `OLLAMA_NUM_CTX=8192` 안에 들어가는지 별도 확인한다. 문자 수를 토큰 수라고 부르지 않는다. 기본판은 총 프롬프트 9,000자 상한을 추가하고 실제 `prompt_eval_count`로 보수성을 시험한다. 선택 모델의 토크나이저를 사용할 수 있으면 토큰 단위 예산으로 바꾼다.

우선순위는 현재 질문→필수 시스템 지침→관련 근거→대화 참고다. 초과 시 오래된 대화와 낮은 순위 근거부터 제거한다. 핵심 수치가 있는 표 행 중간을 잘라 문장을 바꾸지 않는다. 제거된 자료의 ID는 인용 목록에서 제외한다.

기본판의 “확실한 근거 부족”은 허용된 후보가 0개인 경우다. 점수 임계값은 도메인 평가셋으로 보정하기 전에는 하드코딩하지 않는다. 후보가 있어도 질문을 답하지 못하면 06장의 answerer가 unanswered/clarification으로 판정한다. 모델 판정도 오류가 가능하므로 부정 질의 평가가 필수다.

## 8. 고급 검색 확장

BM25를 추가하면 봇·세대의 청크에서 별도 인덱스를 생성하고 같은 활성화 시점에 교체한다. 벡터 유사도와 BM25 원점수를 그대로 더하지 않는다. RRF 예시는 `sum(1/(60+rank))`로 융합하고 각 경로의 순위·후보 수를 기록한다.

Cross encoder는 후보 query/text 쌍을 재정렬한다. 큰 부모 문서 전체 대신 실제 정답이 있는 검색 조각을 넣는다. 모델 ID·revision·최대 입력 길이·batch·장치를 기록하고 `RERANKER_ENABLED=false`이면 로드·호출을 모두 건너뛰는지 시험한다.

확장 전후 같은 golden set으로 Recall@20, MRR, 답변 품질, 검색 p95를 비교한다. 품질이 좋아져도 지연·메모리가 허용 범위를 넘으면 도입하지 않는다.
