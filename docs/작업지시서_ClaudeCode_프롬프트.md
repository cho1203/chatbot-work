# 작업지시서 검색 — Claude Code 실행 프롬프트

사용법: `작업지시서_검색_설계서.md`를 프로젝트의 `docs/` 폴더에 넣고, 아래 프롬프트를 **위에서부터 하나씩** 복사해 Claude Code에 넣는다.
각 단계 결과를 확인하고 OK한 뒤 다음 프롬프트로 넘어간다.

> ⚠️ 설계서 7장 0번(허정희 이사님 확인)이 끝나기 전에는 **프롬프트 1~2까지만** 실행한다.

---

## 프롬프트 0 — 시작 (규칙 설정 + 현황 파악)

```
docs/작업지시서_검색_설계서.md 를 처음부터 끝까지 읽어 줘.
이 문서가 작업지시서(작업표준서) 검색 기능의 설계 기준이야.
참고로 docs/고장수리내역_검색_설계서.md 도 있으니 같이 읽어 줘. 공통 모듈(정규화, LLM 호출, PDF 변환)을 공유해.

작업 규칙:
1. 한 번에 한 단계만 진행해. 각 단계는 먼저 계획을 보여주고, 내가 OK하면 구현해.
2. 각 단계가 끝나면 테스트 코드를 만들어 실행하고 결과를 요약해 줘.
3. 기존 코드가 있으면 새로 만들기 전에 재사용할 수 있는지 먼저 확인해.
4. 설계서에 없는 기능은 추가하지 마. 특히 작업지시서 저장·전송 DB는 만들지 마 (8장 #3, 미결).
5. 설계서 11.2절에서 "가정"이나 "미검증"으로 표시된 부분은 확정된 것처럼 하드코딩하지 말고 config나 분리된 함수로 만들어 바꿀 수 있게 해 줘.
6. 8장 미결 사항에 걸리거나 양식 정보가 필요하면 추측하지 말고 나한테 물어봐.
7. 값은 LLM이 바꾸거나 요약하지 않고 원문 그대로 보여줘야 해 (Retriever only).

지금은 코드를 수정하지 말고:
- 현재 코드베이스에서 작업지시서/고장수리내역 관련 코드와 이미 있는 공통 기능을 찾아서
- 설계서 2.2절 모듈 구성과 비교해 "있음 / 일부 있음 / 없음" 표로 정리해 줘.
```

---

## 프롬프트 1 — 양식·단계 구조 검증 (설계서 6.1절)

```
설계서 6.1절대로 scripts/check_ws_structure.py 를 만들어 줘.
입력: [작업표준서 엑셀 경로]

시트마다 다음을 reports/ws_structure.md 로 리포트해 줘:
1. 식별자 항목(표준번호, 품번, 모델명, 공정명, 부품명, 작업명)이 있는지와 셀 위치
2. 단계를 어떻게 구분하고 있는지 (번호 열 / 행 블록 / 그림 단위 / 기타) — 판단 근거가 된 셀 범위도 같이
3. 섹션 라벨(그림 캡션, 작업내용, 필요치, 공구, 소요 부품) 위치
4. 조건에 따라 다른 단계로 가는 "분기"가 있는지
5. 병합셀 개수, 이미지 개수와 앵커 셀

마지막에 "단계 분리 규칙 제안"과 "설계서 3.2절과 다른 점"을 정리해 줘.
코드는 이 스크립트만 만들고, 다른 파일은 건드리지 마.
```

> 이 리포트를 들고 이사님께 확인한다. 확인 결과를 설계서 3.2절, 8장에 반영한 뒤 프롬프트 2로 넘어간다.

---

## 프롬프트 2 — 공통 모듈 분리 (설계서 2.2절)

```
설계서 2.2절대로 고장수리내역 코드에서 공통 기능을 common/ 으로 분리해 줘.
- common/normalize.py : 식별자 정규화 (NFKC, 공백·하이픈·언더스코어 제거, 대문자)
- common/llm_client.py : LLMClient 인터페이스 + GITCClient (고장수리내역 설계서 4.8절)
- common/pdf_convert.py : 시트 → PDF 변환 (LibreOffice headless)

주의:
- 고장수리내역 기능이 깨지면 안 돼. 분리 전에 고장수리내역 테스트를 먼저 실행해서 결과를 기록하고,
  분리 후 같은 테스트를 다시 실행해서 결과가 같은지 보여줘.
- 아직 고장수리내역 코드에 해당 기능이 없으면 common/ 에 새로 만들고, 그 사실을 알려줘.
```

---

## 프롬프트 3 — 적재: 문서·단계·섹션 분리 + PDF (설계서 3장)

```
설계서 3.1~3.4절과 reports/ws_structure.md 의 단계 분리 규칙대로
ingest/work_standard/split_sheets.py 와 split_steps.py 를 만들어 줘.

결과물:
- data/ws_documents.jsonl : 문서 단위 (식별자 원본 + 정규화 값)
- data/ws_steps.jsonl     : 단계 단위 (step_no, step_title, caption, content, spec_value, tools, parts, 원본 위치)
- data/ws_chunks.jsonl    : 섹션 단위 청크. text 앞에 "[{작업명}] {step_no}단계 {step_title} - " 맥락을 붙여 줘 (3.3절)
- data/pdf/ws/{doc_id}.pdf

먼저 시트 1개로만 돌려서 결과 3개 파일 내용을 나한테 보여주고, 내가 OK하면 전체에 적용해 줘.
검증: 문서 수 = PDF 수, 시트별 단계 수가 리포트와 일치하는지 출력해 줘.
```

---

## 프롬프트 4 — DB 스키마 + 임베딩 적재 (설계서 3.5절)

```
설계서 3.5절 SQL대로 work_standards, work_steps, work_chunks, work_parts 테이블 마이그레이션을 만들어 줘.
그다음 ingest/work_standard/embed_load.py 로 data/ws_*.jsonl 을 읽어서 적재해 줘.
- work_chunks.text 를 BGE-M3(BAAI/bge-m3, normalize_embeddings=True)로 임베딩
- 문서 1건을 넣는 upsert_ws_document(doc, steps, chunks, parts) 함수를 분리해 줘 (나중에 문서 추가용)

검증: embedding NULL 0건, work_steps 의 (doc_id, step_no) 가 1부터 빠짐없이 이어지는지 확인해 줘.
```

---

## 프롬프트 5 — 질의 분석 (설계서 4.1절)

```
설계서 4.1절대로 search/work_standard/query_parser.py 에 parse_ws_query(query) 를 만들어 줘.
- common/llm_client.py 의 LLMClient 를 사용 (GITC API)
- 출력 JSON: identifiers(6개), intent(8종), step_ref, semantic_query, raw_query
- 시스템 프롬프트와 예시는 설계서 그대로, few-shot 은 intent 별로 1개씩 넣어 줘
- temperature 0, 실패 시 1회 재시도 → 그래도 실패하면 intent=general, 식별자 null
- 식별자는 common/normalize.py 로 정규화

테스트: intent 8종마다 질의 2개씩(총 16개) 만들어서 분류 결과와 응답 시간을 표로 보여줘.
```

---

## 프롬프트 6 — 식별자 검색 + 앵커 찾기 (설계서 4.2, 4.3절)

```
설계서 4.2, 4.3절대로 search/work_standard/retriever.py 에 두 함수를 만들어 줘.
1) ws_identifier_search(parsed) -> 후보 문서 집합 D (WS_CORE_IDS 1개 이상 일치, pg_trgm 사용)
2) find_anchor(parsed, D) -> 앵커 후보 목록
   - 검색 문장: step_ref 가 있으면 step_ref, 없으면 semantic_query
   - intent 별 우선 섹션은 4.3절 표대로 (config 로 분리)
   - 1차 D 안에서 → 임계값 이상 없으면 2차 전체에서
   - 같은 단계의 여러 섹션이 걸리면 단계 단위로 묶어 최고 점수 사용

테스트: "클램프 작동기 전공 확인" 처럼 실제 단계명 3개, 표현을 바꾼 질의 3개로 앵커 결과와 점수를 보여줘.
```

---

## 프롬프트 7 — 의도별 처리 + 판단 + 대화 맥락 (설계서 4.4, 4.5, 4.9절)

```
설계서 4.4, 4.5, 4.9절대로 만들어 줘.
- search/work_standard/intent_handler.py : intent 별 처리 (4.4절 표)
- search/work_standard/pipeline.py : search_work_standard(query, session) 
  - 판단: high / low / doc_only / ambiguous / none (4.5절)
  - next_step/prev_step 은 앵커가 high 일 때만 바로 이동, low 면 "이 단계를 말씀하신 건가요?" + 앵커 단계 표시
  - 서로 다른 문서 앵커의 점수 차이가 AMBIGUITY_GAP 이내면 ambiguous → 문서 선택지 반환
  - 대화 맥락: step_ref 와 식별자가 없고 intent 가 next/prev/tools/parts/spec 이면 세션의 마지막 doc_id, step_no 를 앵커로 사용
- 모든 임계값은 config.py 로 (WS_THRESHOLD, WS_LOW, AMBIGUITY_GAP, CONTEXT_TTL)
- 반환 형식은 4.5절 JSON 그대로

단위 테스트: 설계서 5장 예시 A~F 와 "그 다음은?"(대화 맥락), 같은 단계명이 2개 문서에 있는 경우(ambiguous)를 만들어 실행해 줘.
```

---

## 프롬프트 8 — API + 챗봇 화면 (설계서 4.7, 4.8절)

```
설계서 4.7절 API 4개와 4.8절 화면을 만들어 줘.
- 연동할 챗봇 코드 위치: [챗봇 프론트/백엔드 경로]
- 단계 카드: 작업명(모델/공정), 현재 단계/전체 단계, 섹션 내용 원문, [이전][다음][원본 보기(PDF)]
- confidence 가 low 면 안내 문구, ambiguous 면 문서 선택 버튼
- 이전/다음 버튼으로 이동하면 세션 맥락도 갱신
- LLM 이 내용을 요약하거나 바꾸지 않게 해 줘 (원문 그대로)
```

---

## 프롬프트 9 — 평가 + 임계값 확정 + 청킹 비교 (설계서 6.2절)

```
설계서 6.2절대로 평가를 만들어 줘.
1) tests/eval_ws_queries.jsonl 초안 30개를 만들어서 먼저 보여줘. 내가 정답(doc_id, step_no, section)을 확인할게.
   - intent 별로 골고루, next_step 과 spec 은 많이
   - 대화 맥락 질의 5개 이상, 같은 단계명이 여러 문서에 있는 질의 3개 이상
2) scripts/eval_ws.py 로 아래 지표를 출력해 줘:
   의도 분류 정확도, 앵커 정확도, 최종 정답률, 잘못된 다음 단계 노출률, 확인 질문 비율
3) WS_THRESHOLD 0.40~0.80, WS_LOW 0.30~0.60 을 0.05 단위로 바꿔 가며 표로 출력
4) 청킹 비교: 섹션별 벡터(현재) vs 단계 전체 벡터 1개 — 같은 평가셋으로 비교
결과를 reports/ws_eval.md 로 저장하고, "잘못된 다음 단계 노출률 0" 조건에서 가장 좋은 설정을 추천해 줘.
```

---

## 단계별 확인 체크리스트

| 프롬프트 | 넘어가기 전에 확인할 것 |
|---|---|
| 0 | 현황표가 실제 코드와 맞는지 |
| 1 | 단계 분리 규칙 제안이 실제 시트와 맞는지 → **이사님 확인** |
| 2 | 고장수리내역 테스트 결과가 분리 전후로 같은지 |
| 3 | 시트 1개 결과의 단계·섹션이 원본과 맞는지 |
| 4 | step_no 가 빠짐없이 이어지는지 |
| 5 | intent 분류가 맞는지, 응답 시간 |
| 6 | 표현을 바꾼 질의에서도 앵커를 찾는지 |
| 7 | 예시 A~F, 대화 맥락, ambiguous 테스트 통과 |
| 8 | 화면에서 이전/다음/PDF 동작 |
| 9 | 평가표, 섹션별 vs 단계별 비교 결과 |
