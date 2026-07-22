# RAG 전처리 가이드

> 규정 PDF를 검색 가능한 조각(청크)으로 만드는 **전처리** 과정 정리.
> 작성 배경: KARI-NA 규정검색(RAG) 프로젝트.

---

## 1. RAG 전체 흐름 — 전처리는 어디에?

```
① 문서(PDF)
② 전처리(청킹)   ← 이 문서가 다루는 부분
③ 임베딩 (글 → 숫자 벡터)
④ 벡터DB 저장 (pgvector)
⑤ 검색 (질문 → 비슷한 청크 찾기)
```

- **전처리 = ②번.** "긴 문서를 검색하기 좋은 작은 조각으로 자르고, 이름표(태그)를 붙이는" 작업.
- 전처리의 결과물 = `chunks.jsonl` (잘린 조각들).
- ③~⑤(임베딩·저장·검색)는 별도 단계이며, DB·임베딩 모델이 필요하다.

---

## 2. 핵심 개념

| 용어 | 뜻 |
|---|---|
| **청크(chunk)** | 긴 문서를 잘라낸 작은 조각. 검색·추천의 기본 단위. |
| **임베딩(embedding)** | 문장을 숫자 벡터로 바꾼 것. 뜻이 비슷하면 벡터도 가깝다. |
| **registry.csv** | 문서 대장. "어떤 PDF를 어떤 분류로 넣을지" 적은 목록. |
| **ingest.py** | 청킹 프로그램. 이걸 실행해야 청크가 생긴다. |

**요리 비유:**
`registry.csv`(주문서) + `PDF`(재료) → `ingest.py`(요리 기계) → `chunks.jsonl`(완성 요리)
→ 주문서만 써놓는다고 요리가 나오지 않는다. **기계(프로그램)를 돌려야** 나온다.

---

## 3. 준비물

1. **PDF 원본** — 자를 규정/상품설명서 파일.
2. **registry.csv (문서 대장)** — 각 PDF의 메타데이터 한 줄씩.

### registry.csv 컬럼

| 컬럼 | 설명 | 예시 |
|---|---|---|
| `doc_id` | 문서 고유 ID | `hana-jutaekdamho` |
| `filename` | 실제 PDF 파일명 (**PDF와 정확히 일치해야 함**) | `주택담보대출 상품설명서.pdf` |
| `title` | 문서 제목 | `주택담보대출 상품설명서` |
| `doc_type` | 문서 종류 | `상품설명서` / `약관` / `핵심설명서` |
| `routing` | 라우팅(1층) | `G`(일반)/`S`(단순)/`E`(긴급) |
| `department` | 부서(2층) | `LON`(대출)/`DEP`(예금)/`FX`(외환)/`INV`(연금) |
| `business_code` | 업무코드(3층) | `loan`/`subscription`/`fx`/`pension` |
| `categories` | 검색 필터용 태그 | `LON;loan` |
| `version` | 버전 | `v1` |
| `effective_date` | 시행일 | `2026-01-01` |
| `status` | 상태 | `active` / `superseded` |

> `filename`이 실제 PDF 파일명과 다르면 "대장에 없다"고 에러가 난다.

---

## 4. 실행 — chunks.jsonl 만들기

`backend/` 폴더에서 터미널로 실행:

```bash
python -m app.rag.ingest  <registry.csv 경로>  <PDF1>  <PDF2> ...  >  chunks.jsonl
```

예:
```bash
cd backend
python -m app.rag.ingest ../database/rag/registry.csv \
  "주택담보대출 상품설명서.pdf" \
  "주택청약종합저축.pdf"  >  chunks.jsonl
```

- 앞의 `registry.csv` = 대장(이름표 정보)
- 뒤의 `*.pdf` = 실제 자를 재료 → **반드시 함께 지정**
- `> chunks.jsonl` = 결과를 이 파일로 저장

> Windows에서 한글 깨짐 방지: `PYTHONIOENCODING=utf-8` 를 앞에 붙이거나 환경변수로 설정.

---

## 5. 청킹 기준 (ingest.py가 자동으로 하는 일)

- **구조로 자르기**: 글자 수로 뚝 자르지 않고 `◇ □ ①②③ 제N조 [소제목]` 같은 **문서 구조 경계**에서 새 조각 시작.
- **표는 따로**: 표는 별도 조각으로 떼서 마크다운 표로 보존 (셀 관계 유지).
- **중복 제거**: 같은 내용(은행용/고객용 중복 페이지)은 해시로 걸러냄.
- **맥락 헤더**: 각 조각 앞에 `[문서명 > 페이지 > 섹션]` 을 붙여 조각만 봐도 출처를 알게.
- **1200자 캡**: 너무 길면 문장 경계로만 분할.
- **엔티티 태깅**: 본문의 법령·서식코드·개정일을 자동 추출.
- **태그 상속**: 대장의 부서·업무코드를 각 조각에 붙임.

---

## 6. 결과물 — chunks.jsonl

조각 하나 = JSON 한 줄:

```json
{"chunk_id":"...", "doc_id":"hana-jutaekdamho", "title":"주택담보대출 상품설명서",
 "page":3, "section":"중도상환수수료", "kind":"text",
 "raw":"중도상환 시 수수료는 ...", "text":"[주택담보대출 > p3 > 중도상환수수료]\n...",
 "department":"LON", "business_code":"loan", "categories":["LON","loan"]}
```

여기까지가 **전처리 완료.**

---

## 7. 스캔본 PDF (이미지) 주의

- 텍스트가 없는 **스캔본(이미지 PDF)**은 `pdfplumber`로 글자가 안 뽑힌다 → 청크 0개.
- 이 경우 **OCR**(이미지→글자)이 먼저 필요. (예: PaddleOCR 한국어)
- 자동 파이프라인 `auto_ingest.py`는 스캔본을 감지해 OCR을 태우도록 설계돼 있다.

---

## 8. 전처리 다음 단계 (참고)

`chunks.jsonl`은 아직 "글 조각"일 뿐. 실제 검색이 되려면:

1. **임베딩** — 각 청크를 bge-m3로 벡터화
2. **pgvector 적재** — `store.upsert_documents_and_chunks()`
3. **검색** — `search_regulations()` (의미검색 + 키워드 하이브리드)

이 단계는 임베딩 모델 + DB(pgvector Postgres)가 있는 환경(온프레미스 인덱스 호스트)에서 수행한다.

---

## 한눈에 정리

```
registry.csv (대장)  +  PDF (재료)
        │
        ▼   python -m app.rag.ingest registry.csv *.pdf > chunks.jsonl
        │
chunks.jsonl (전처리 완료 조각)
        │
        ▼   임베딩 → pgvector 적재 → 검색   (별도 단계)
```
