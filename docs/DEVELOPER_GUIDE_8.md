# 삼광 설비 챗봇 — 다이어그램 모음

작성일: 2026-10-12
기준 문서: DEVELOPER_GUIDE_5.md(핵심 아키텍처), DEVELOPER_GUIDE_6.md 3번(그림·표 구현), DEVELOPER_GUIDE_7.md(업무 요청 통합)
상태: 1·2번은 각 문서의 확정된 다이어그램을 그대로 옮긴 것. 3번은 GUIDE_7의 글로만 적힌 전체 프로덕션 요구사항을 그림으로 새로 그린 **초안**이라 구현 전 확인이 필요하다. 4번은 **1차 데모 범위**로, 기존 설계 재사용만으로 바로 구현 가능한 수준까지 정리했다.

## 1. 기존 아키텍처 (GUIDE_5)

### 1.1 구현을 구성하는 계층 구조 (0.2절)

```mermaid
flowchart TB
  U["정적 화면 index.html"] --> A["FastAPI REST"]
  A --> R["질의 라우팅·문서 종류 선택·검색·재순위·생성"]
  R --> V[("PostgreSQL pgvector manual_chunks 테이블")]
  R --> G["GITC"]
  A --> P["page_align 번역 짝 PDF만"]

  PDF["PDF 원본 2종"] --> PDFE["bge-m3 임베딩"]
  PDF2["PDF 번역 2종"] --> P
  EXCEL["엑셀 3종 수리보고서 표준서"] --> EXCE["텍스트 추출 청킹 임베딩"]

  PDFE --> V
  EXCE --> V

  A --> M["요청별 상태 페이지 잉크 필터 캐시"]
  S["venv env 단일 프로세스"] -. 운영 .-> A
  S -. 운영 .-> G
```

### 1.2 색인 준비 순서 — PDF (0.5절)

```mermaid
sequenceDiagram
  participant D as 개발자
  participant B as rebuild_originals.py
  participant J as merged_ko.json / merged_en.json
  participant I as index_to_pgvector.py
  participant V as PostgreSQL
  D->>B: 원본 PDF 두 권 입력 실행
  B->>B: 샘플 그림·표·화면은 유지, 나머지 쪽은 텍스트 추출·청킹
  B->>J: 조각과 110 dpi 페이지 이미지 저장
  D->>I: 조각이 바뀐 뒤에만 실행
  I->>I: bge-m3로 정규화 임베딩
  I->>V: manual_chunks 전체 교체 적재
```

### 1.3 색인 준비 순서 — 엑셀 (0.5b절)

```mermaid
sequenceDiagram
  participant D as 개발자
  participant E as ingest_excel_data.py
  participant X as repair_logs.json / work_standard_*.json
  participant I as index_to_pgvector.py
  participant V as PostgreSQL
  D->>E: 엑셀 3종 입력 실행
  E->>E: 행·절차 읽기, 키워드 6개 추출, 80~90% 샘플링
  E->>X: 청크 JSON 저장
  D->>I: PDF 조각과 함께 색인 실행
  I->>I: bge-m3로 정규화 임베딩
  I->>V: manual_chunks에 PDF 청크와 함께 적재
```

### 1.4 질문 한 건의 실행 순서 (0.6절)

```mermaid
sequenceDiagram
  participant U as index.html
  participant A as FastAPI
  participant Q as 질의 라우팅
  participant V as PostgreSQL
  participant G as 생성 분기
  U->>A: POST /ask 와 POST /search 를 동시에
  A->>Q: is_followup, choose_document_type, choose_lang, requested_kind
  Q->>Q: 문서 종류 선택: manual | repair_log | work_standard
  Q->>V: SQL WHERE 문서 필터 조건 더하기 ORDER BY embedding 코사인거리
  V-->>Q: 후보 청크 (메타데이터 포함, 문서별로 정렬됨)
  Q->>Q: 키워드 보강
  Q-->>G: rank_manual 상위 3개 (또는 repair_log_rank)
  G-->>U: 답 JSON
  A-->>U: 근거 카드 JSON
```

## 2. 그림·표 구현 아키텍처 (GUIDE_6 3.5절)

본문은 왼쪽 분기(살아있으면 그대로, 깨지면 bbox+OCR), 그림·표는 오른쪽 분기(bbox 위치 → 원문 또는 OCR·VLM)로 처리된다. 모든 후보는 VLM 1차 검증을 거쳐 QA 대기열로 들어간다.

```mermaid
flowchart TD
  Z["문서 청킹"] --> Z1["본문<br/>text 타입"]
  Z --> A["그림 표 타입"]
  Z1 --> Z2["PyMuPDF<br/>텍스트 추출"]
  Z2 --> Z4{"한글 비율<br/>20퍼센트 이상"}
  Z4 -->|살아 있음| Z3["그대로<br/>청크 생성"]
  Z4 -->|깨짐| Z5["bbox 위치로<br/>영역 잡기<br/>OCR로 다시 읽기"]
  A --> B["영역 탐지<br/>그림 80pt 이상<br/>표 가로선 4 세로선 2"]
  B --> C["겹침 검사<br/>좌표 복사<br/>빈 영역 버림"]
  C --> D["읽기 범위 확정<br/>고정 여백<br/>다음 그림 제목"]
  D --> E{"원문 문장<br/>있는가"}
  E -->|있음| F["원문 문장<br/>그대로 사용"]
  E -->|그림 없음| G["OCR 캡션<br/>EasyOCR 다음 PaddleOCR<br/>다음 VLM"]
  E -->|표 없음| H["칸별 OCR<br/>HTML 재구성"]
  F --> V["VLM 1차 검증<br/>캡션과 그림이 맞는가"]
  G --> V
  H --> V
  Z5 --> V
  V -->|불일치| I["QA 대기열<br/>VLM 불일치 표시"]
  V -->|일치| I2["QA 대기열"]
  I --> J["우선순위 분류<br/>과다캡처 등"]
  I2 --> J
  J --> K{"경고 주의<br/>위험 문구"}
  K -->|예| L["2차 승인"]
  K -->|아니오| M["1차 승인"]
  L --> N["검수 통과"]
  M --> N
  N --> O["merged 파일 반영"]
  O --> P["index_to_pgvector.py<br/>pgvector 색인"]
  Z3 --> P
```

## 3. 신규 요구사항 아키텍처 (GUIDE_7 기반, 초안)

GUIDE_7에 글로만 적혀 있던 4가지 요구사항을 그림으로 옮긴 것이다. 세부 API 경로·인증 방식은 아직 미정이라, 아래 그림은 **흐름의 뼈대만** 보여준다.

### 3.1 전체 시스템 구조도 (GUIDE_7 4.1·4.9절 기반)

```mermaid
flowchart TB
  PC["PC 클라이언트"] --> MAIN["메인 API 서버"]
  PC -. 장애 시 .-> BACKUP["예비 API 서버"]
  MAIN --> DB[("DB<br/>대화 보고서 pgvector")]
  BACKUP --> DB
  MAIN --> GEN["GITC"]
  BACKUP --> GEN
  MAIN --> FILES["파일 저장소<br/>원본 생성파일 사진"]
  BACKUP --> FILES
```

**확인 필요**: DB·파일 저장소를 메인·예비가 공유하는지, 각자 따로 두고 동기화하는지는 GUIDE_7 4.9절에서도 미정으로 남아있다. 위 그림은 "공유 DB" 가정으로 그렸다.

### 3.2 PC별 대화 구분 흐름 (GUIDE_7 4.2절 기반)

```mermaid
sequenceDiagram
  participant PC as PC 클라이언트
  participant S as API 서버
  participant DB as DB
  PC->>S: 최초 실행, 설치ID 전송
  S->>DB: 식별자 신규 등록
  DB-->>S: 등록 완료
  S-->>PC: 자격증명 발급
  PC->>S: 재실행 시 같은 식별자로 재연결
  S->>DB: 식별자로 대화 조회
  DB-->>S: 대화 목록·메시지
  S-->>PC: 기존 대화 이어서 표시
```

**확인 필요**: "식별자"가 설치 ID인지 네이티브 인증서인지는 GUIDE_7 4.2절에서 AI 팀 회신 대기 중인 사항이다. 위 그림은 식별자 종류와 무관하게 성립하는 공통 흐름만 그렸다.

### 3.3 메인/예비 전환 흐름 (GUIDE_7 4.9절 기반)

```mermaid
flowchart LR
  A["메인 운영 중"] -->|상태 신호 끊김| B["장애 감지"]
  B -->|판정 기준 충족| C["예비로 전환 중"]
  C -->|전환 완료| D["예비 운영 중"]
  D -->|메인 신호 복귀| E["메인 복귀 판단"]
  E -->|복귀 확정| A
  A -->|양쪽 장애| F["서비스 불가"]
  D -->|양쪽 장애| F
```

**확인 필요**: "장애 감지" 판정 기준(시간, 신호 종류)과 "메인 복귀 판단" 기준은 GUIDE_7 4.9절에서 전부 미정이다.

### 3.4 보고서 작성·저장·엑셀생성 흐름 (GUIDE_7 4.6절 기반)

```mermaid
sequenceDiagram
  participant PC as PC 클라이언트
  participant S as API 서버
  participant DB as DB
  PC->>S: 보고서 작성 내용 전송
  S->>DB: 신규 버전 저장
  DB-->>S: 저장 완료, 버전ID
  S-->>PC: 저장 완료 응답
  PC->>S: 엑셀 생성 요청, 버전ID
  S->>DB: 해당 버전 조회
  DB-->>S: 본문·작업행·사진 정보
  S->>S: 서식에 반영, 사진 삽입
  S-->>PC: 생성 완료, 다운로드 링크
  PC->>S: 다운로드 요청
  S-->>PC: 엑셀 파일 전송
```

**확인 필요**: "저장"과 "검색(pgvector) 반영"이 같은 요청에서 같이 일어나는지, 별도 색인 스크립트를 다시 돌려야 하는지는 GUIDE_7 6번 절(남은 작업 4번)에서 아직 결정되지 않았다. 위 그림은 저장까지만 그렸고, 검색 반영 단계는 의도적으로 뺐다.

## 4. 1차 데모 범위

3번 절은 메인/예비 이중화·대화 영속 저장·보고서 작성 같은 **전체 프로덕션** 요구사항이다. 1차 데모는 그 전부가 필요하지 않다. 기술 정확도가 우선이고 화면은 깔끔한 수준이면 되는 데모 기준으로 범위를 다시 잡는다. 기존 1~3번 절은 그대로 두고, 데모는 이 절 기준으로만 간다.

### 4.1 데이터 현황

| 구분 | 데이터 | 상태 |
|---|---|---|
| 매뉴얼 — 한글 원본 | `manual_1_ko` (B-64484KO-2 취급설명서) | 준비됨, 검색 대상 |
| 매뉴얼 — 영어 원본 | `manual_1_en` (B-63874EN-2 MANUAL GUIDE i) | 준비됨, 검색 대상 |
| 매뉴얼 — 한글→영어 번역본 | `manual_2_en` | **미준비** — 파일 자체가 없음(GUIDE_5 0.4절엔 "지원 문서"로 설계만 있고 실물 없음) |
| 매뉴얼 — 영어→한글 번역본 | `manual_2_ko` | **미준비** — 위와 동일 |
| 작업지시서 | 작업표준서(SOP) 엑셀 | 회사로부터 수령 완료, 준비됨 |
| 고장수리내역 | 오류·고장 사례 및 조치내역 엑셀 | 회사로부터 수령 완료, 준비됨 |

매뉴얼 4종 중 번역본 2종은 파일이 없어서 데모에 못 넣는다. 다만 이건 원래 설계(GUIDE_5)에서도 번역본은 검색 대상이 아니라 언어 전환 버튼 전용이었으므로, **번역본이 빠져도 매뉴얼 탭(한글·영어 원본 검색)은 정상 동작한다** — 언어 전환 버튼만 데모에서 빠지는 것.

### 4.2 탭 구성 (3개)

| 탭 | `document_type` 값 | 데이터 소스 |
|---|---|---|
| 매뉴얼 | `manual` | `manual_1_ko`, `manual_1_en` |
| 작업지시서 | `work_standard` | 작업표준서(SOP) 엑셀 |
| 고장수리내역 | `repair_log` | 오류·고장 사례 엑셀 |

이 구분은 새로 만드는 게 아니라 GUIDE_5 5.2절의 기존 `document_type` 필드를 그대로 재사용한다.

### 4.3 데모용 API (최소 구성, 기존 라우트 재사용)

메인/예비 이중화·인증·PC별 대화 구분(GUIDE_7 4장)은 데모 범위가 아니다. 1차 데모는 기존 GUIDE_5 11.1절 라우트만으로 충분하다 — 새 API를 만드는 게 아니라 탭마다 `document_type` 값만 다르게 보내면 된다.

```
GET  /                       정적 화면, 탭 3개
POST /ask                    {query, document_type, prior_query}  →  답 JSON
POST /search                 {query, document_type, prior_query}  →  근거 카드 배열
GET  /manuals/{name}.pdf      매뉴얼 원문 서빙 (매뉴얼 탭 전용)
GET  /filters                 작업지시서·고장수리내역 탭 드롭다운 옵션 (GUIDE_5 10.6절)
```

### 4.4 UI 요구사항 (최소)

- 탭 3개(매뉴얼/작업지시서/고장수리내역) 전환만 되면 됨
- 매뉴얼 탭은 한글/영어 책 선택만 있으면 됨 — 언어 전환 버튼은 번역본이 없어서 데모에서 뺌
- 디자인은 깔끔한 수준이면 충분 — 이번 데모의 우선순위는 검색 정확도·근거 표시 같은 기술 쪽이지 화면 꾸미기가 아님

### 4.5 데모 아키텍처

```mermaid
flowchart LR
  T1["매뉴얼 탭"] --> Q["질문 하나<br/>document_type 포함"]
  T2["작업지시서 탭"] --> Q
  T3["고장수리내역 탭"] --> Q
  Q --> R["기존 검색 파이프라인<br/>GUIDE_5 0.6절"]
  R --> V[("pgvector<br/>manual_chunks")]
  R --> ANS["답 + 근거 카드"]
  ANS --> T1
  ANS --> T2
  ANS --> T3
```

데모는 탭이 3개로 나뉘어 보여도, 뒤에서는 **기존 검색 파이프라인 하나를 `document_type`만 바꿔서 재사용**하는 구조다. 탭별로 서버나 API를 따로 만드는 게 아니다.

## 5. 다음에 할 일

**데모(4번 절)** 는 기존 설계 재사용만으로 가능하다 — 새로 확인할 게 거의 없다. 작업지시서·고장수리내역 엑셀 파일도 이미 회사로부터 수령 완료라 바로 구현 들어갈 수 있다.

**전체 프로덕션(3번 절)** 은 아직 멀었다. "확인 필요" 네 가지가 정해지면 다이어그램을 그만큼 구체화한다. DB 스키마(ERD)와 API 요청·응답 예시는 이 문서가 아니라 GUIDE_7에 별도로 추가하는 걸 권장한다 — 다이어그램은 흐름을, GUIDE_7은 세부 계약을 담당하는 역할 분리를 유지한다.
