# 🏢 api_llm - 엔터프라이즈 AI 에이전트 시스템

## 📋 프로젝트 개요

기존 `api.py`(2346줄)를 **모듈화**하고 **구조화**하여 유지보수와 확장이 용이한 엔터프라이즈 시스템으로 재편성했습니다.

### 주요 특징

✅ **모듈화**: 기능별로 분리된 독립적인 패키지 구조  
✅ **LLM 유연성**: Google Gemini, Ollama, OpenAI 모두 지원 (설정 한 곳에서만 변경)  
✅ **로컬 모델 학습 지원**: JSONL 형식 학습 데이터 자동 수집 (`logs/training_data.jsonl`)  
✅ **사용자 피드백 시스템**: 👍/👎 버튼으로 즉각적인 만족도 평가 (-1/0/1)  
✅ **자동 차트 생성**: AI 기반 데이터 시각화 (PIE/BAR/LINE/TABLE)  
✅ **ChromaDB 벡터 DB**: Google Drive 대신 실제 벡터 DB 사용  
✅ **기능 손실 없음**: 기존 모든 기능 100% 보존  

---

## 📦 폴더 구조

```
api_llm/
├── __init__.py                 # 패키지 진입점
├── config.py                   # 모든 설정값 중앙 관리 ⭐
├── app.py                      # Streamlit 메인 앱 (차트 렌더링 지원)
│
├── models/                     # LLM & Vector DB 관리
│   ├── __init__.py
│   ├── llm_config.py          # LLM 모델 설정 (Google/Ollama/OpenAI)
│   └── vector_db.py           # ChromaDB 관리
│
├── sql/                        # SQL 처리 모듈
│   ├── __init__.py
│   ├── sql_validator.py       # SQL 검증 (구문/컬럼/JOIN/안전성 등)
│   ├── sql_generator.py       # LLM 기반 SQL 생성
│   └── sql_executor.py        # DB 연결 및 쿼리 실행 + 학습 로그 기록
│
├── rag/                        # 검색 증강 생성 (RAG)
│   ├── __init__.py
│   └── retrievers.py          # 6가지 RAG 검색 (하이브리드 + 리랭킹)
│
├── agent/                      # LangGraph 에이전트
│   ├── __init__.py
│   ├── state.py               # AgentState 상태 정의
│   └── nodes.py               # 7가지 처리 노드 (차트 타입 자동 추론 포함)
│
├── cache/                      # 쿼리 캐싱
│   ├── __init__.py
│   └── query_cache.py         # TTL 기반 캐시 관리
│
├── logging/                    # 로깅 설정
│   ├── __init__.py
│   └── logger_config.py       # 메인 + 학습 로그 설정
│
└── utils/                      # 유틸리티
    ├── __init__.py
    ├── helpers.py             # 차트/설명/메모리 관리
    └── training_logger.py     # 🆕 학습 데이터 자동 수집 (JSONL 형식)
```

---

## 🎯 주요 개선 사항 (v2.0)

### 1️⃣ 학습 데이터 자동 수집 시스템
**파일**: `utils/training_logger.py`

사용자의 질문, 생성된 SQL, 실행 결과를 JSONL 형식으로 자동 수집합니다.

```python
{
  "timestamp": "2025-02-27T10:30:45.123456",
  "question": "23년 매출액은?",
  "generated_sql": "SELECT SUM(sale_amount) FROM sales WHERE year=2023",
  "schema_snapshot": "sales(sale_id, sale_amount, year), ...",
  "execution_success": true,
  "execution_result_summary": "1행 반환: $1,234,567",
  "user_satisfaction": null,  # 사용자 평가 전
  "error_details": null,
  "metadata": {"retry_count": 0, "cache_hit": false}
}
```

**위치**: `<project>/logs/training_data.jsonl`  
**자동 기록**: SQL 실행 시 (user_satisfaction=null)  
**업데이트**: 사용자 피드백 시 (user_satisfaction=1 또는 -1)

### 2️⃣ 사용자 피드백 UI (👍 / 👎)
**파일**: `app.py` (render_response 함수)

Streamlit 응답 아래에 만족도 버튼을 표시합니다:

```
📊 [차트/테이블 표시]
SQL: SELECT ...

👍 좋아요  👎 싫어요
```

- **👍 좋아요**: user_satisfaction = +1
- **👎 싫어요**: user_satisfaction = -1
- **평가 안 함**: user_satisfaction = null

학습 로그에 즉시 기록되어 로컬 모델 학습에 활용됩니다.

### 3️⃣ 자동 데이터 시각화
**파일**: `agent/nodes.py` (visualization_node)

AI가 질문과 데이터를 분석하여 최적 차트 타입을 자동 선택:

| 차트 타입 | 트리거 | 예시 |
|---------|--------|------|
| **PIE** | 소수 항목 비율 | "상위 5개 고객사 비중" |
| **BAR** | 카테고리별 비교 | "부서별 판매량" |
| **LINE** | 시계열/추이 | "월별 매출 추이" |
| **TABLE** | 상세 데이터 | "거래처 목록" |

**자동 감지 로직**:
- Datetime 컬럼 있음 → LINE (추이)
- 행 수 < 5 → PIE (비율)
- 행 수 > 100 → LINE (트렌드)
- Otherwise → BAR (비교)

### 4️⃣ 간소화된 SQL 검증
**파일**: `sql/sql_validator.py`

SELECT/WITH 제한 제거 → 더 많은 쿼리 지원 + 안전성 유지:

**제거된 제한**:
```python
# ❌ 제거됨
if first_token not in ("SELECT", "WITH"):
    return False
```

**유지된 검증** (다층 방어):
1. ✅ sqlglot 구문 분석 (진정한 SQL 에러 감지)
2. ✅ SELECT * 감지 및 확장
3. ✅ GROUP BY 검증 (집계 함수 오류 차단)
4. ✅ Cartesian Product 감지 (JOIN ON 누락)
5. ✅ 테이블/컬럼 존재 확인
6. ✅ JOIN 키 매핑 검증
7. ✅ 자동 LIMIT 추가 (대용량 결과 방지)

### 5️⃣ 표준화된 RAG 반환값
**파일**: `rag/retrievers.py`

6가지 RAG 컬렉션이 일관된 형식으로 반환:

| 컬렉션 | 반환 형식 | 예시 |
|-------|---------|------|
| **FEWSHOT** | doc + meta | "SELECT ..." + 설명 |
| **ERROR** | doc + solution | 에러 + 해결책 |
| **SYNONYM** | doc + canonical | "고가" + "high_price" |
| **BIZTERM** | doc + description | 용어 + 뜻 |
| **SCHEMA** | meta only | 테이블 구조만 |
| **KEYWORD** | meta only | 의도 + 테이블만 |

---

## 🔧 사용 방법

### 0️⃣ 환경변수 설정 (.env 파일)

```bash
# .env 파일 생성
cp .env.example .env

# 필수 API 키 설정
GOOGLE_API_KEY=your_google_api_key          # Google Gemini API
LANGSMITH_API_KEY=your_langsmith_api_key    # LangChain 추적
HF_TOKEN=your_huggingface_token             # Hugging Face 모델

# 선택사항 (기본값 사용 가능)
CHROMA_PROVIDER=persistent                  # 로컬 벡터 DB
CHROMA_PATH=/content/local_vdata            # ChromaDB 경로
DB_USER=name                                # DB 사용자
DB_PASSWORD=1234                            # DB 비밀번호
DB_HOST=127.0.0.1                          # DB 호스트
DB_PORT=5432                               # DB 포트
LOG_LEVEL=INFO                             # 로그 레벨
```

### 1️⃣ 설정 (config.py)

모든 설정이 **한 파일**에 집중:

```python
from config import DEFAULT_LLM_CONFIG, CHROMA_CONFIG

# LLM 변경 (현재: Gemini 2.5 Flash)
DEFAULT_LLM_CONFIG = LLMConfig(
    provider="google",                    # "google" / "ollama" / "openai"
    model_name="gemini-2.5-flash",
    temperature=0.0,
)

# ChromaDB 경로
CHROMA_CONFIG = {
    "PATH": "/content/local_vdata",
    "EMBEDDING_MODEL": "BAAI/bge-m3",
}
```

### 2️⃣ Streamlit 앱 실행

```bash
# .env 설정 완료 후
streamlit run api_llm/app.py

# 포트 변경
streamlit run api_llm/app.py --server.port 8000
```

### 3️⃣ 학습 데이터 조회

```python
from utils.training_logger import get_training_log_stats, export_training_dataset

# 학습 로그 통계
stats = get_training_log_stats()
print(stats)  # {'total_logs': 42, 'success_rate': 0.95, 'avg_satisfaction': 0.8, ...}

# 학습 데이터 내보내기
dataset = export_training_dataset(
    success_only=True,                    # 성공한 쿼리만
    with_satisfaction_only=True,          # 평가된 것만
)
```

### 4️⃣ RAG 검색

```python
from rag import retrieve_parallel

# 6가지 RAG 병렬 검색
results = retrieve_parallel(
    question="매출액에 따른 고객사 순위",
    error_msg="",
)

# 결과 구조
{
    "fewshot": {"results": [...], "score": 0.92},      # doc + meta
    "synonym": {"results": [...], "score": 0.85},      # doc + canonical
    "bizterm": {"results": [...], "score": 0.88},      # doc + desc
    "schema": {"results": [...], "score": 0.91},       # meta only
    "error": {"results": [...], "score": 0.79},        # doc + solution
    "keyword": {"results": [...], "score": 0.84},      # meta only
}
```

### 5️⃣ SQL 생성 및 실행

```python
from sql import SQLGenerator, execute_query

# SQL 생성
sql_gen = SQLGenerator(llm=llm)
sql = sql_gen.generate(
    question="23년 매출은?",
    data_stats={"min_date": "2023-01-01", "max_date": "2025-12-31"},
)

# 실행 + 자동 학습 로그 기록
success, df, error_msg = execute_query(
    sql=sql,
    question="23년 매출은?",
    schema_snapshot="sales(sale_id, amount, year), ...",
    enable_training_log=True,  # 학습 로그 기록
)
```

---

## 🔄 워크플로우

```
[입력: 사용자 질문]
        ↓
[1] entity_linking_node: 오타 보정 + 대명사 해석
        ↓
[2] router_node: 의도 분류 (INVENTORY/TECH_SALES/CHIT_CHAT)
        ↓
    ├─→ CHIT_CHAT → [7] answer_node → 일상 답변
    ├─→ TECH_SALES → [7] answer_node → 기술 답변
    └─→ INVENTORY
        ↓
    [3] sql_generation_node: LLM + RAG로 SQL 생성
        ↓
    [4] db_execution_node: SQL 검증 + DB 실행
        │   └─→ 자동 학습 로그 기록 (logs/training_data.jsonl)
        │       - question, sql, schema_snapshot, success, result_summary
        │       - user_satisfaction = null (사용자 평가 대기)
        │   └─→ 에러? → [3] 재시도 (MAX_RETRY_COUNT까지)
        │   └─→ 성공
        ↓
    [5] result_validation_node: 데이터 이상 검증
        ├─→ 이상? → [3] 재시도
        └─→ 정상
        ↓
    [6] visualization_node: 🆕 차트 타입 자동 추론
        │   - 질문/데이터 분석
        │   - 최적 차트 타입 선택 (PIE/BAR/LINE/TABLE)
        │   - X/Y 축 자동 지정
        ↓
    [7] answer_node: LLM으로 최종 답변 생성
        ↓
[출력: 최종 답변 + 테이블 + 차트]
        ↓
[UI 렌더링 (app.py)]
    - 컨텐츠 표시
    - 📊 차트/테이블 렌더링  👈 chart_info 처리
    - 👍 / 👎 피드백 버튼
        ↓
[사용자 평가]
    - 👍 클릭 → 학습 로그 업데이트 (user_satisfaction = 1)
    - 👎 클릭 → 학습 로그 업데이트 (user_satisfaction = -1)
```

---

## 📊 학습 데이터 흐름

```
SQL 실행
    ↓
[logs/training_data.jsonl 에 기록]
    {
        "timestamp": "2025-02-27T10:30:45.123456",
        "question": "23년 매출은?",
        "generated_sql": "SELECT SUM(amount) FROM sales WHERE year=2023",
        "schema_snapshot": "...",
        "execution_success": true,
        "execution_result_summary": "1행 반환",
        "user_satisfaction": null,  ← 대기 중
        "error_details": null,
        "metadata": {...}
    }
    ↓
[사용자 피드백 클릭]
    👍 좋아요 → user_satisfaction = 1  ✅
    👎 싫어요 → user_satisfaction = -1 ❌
    ↓
[학습 로그 업데이트]
    {
        "timestamp": "2025-02-27T10:30:45.123456",
        ...
        "user_satisfaction": 1,  ← 업데이트됨
        ...
    }
    ↓
[로컬 모델 학습 시 사용]
    from utils.training_logger import export_training_dataset
    dataset = export_training_dataset(with_satisfaction_only=True)
    # → 사용자가 평가한 42개 샘플로 파인튜닝
```

---

## 💾 로깅 시스템

### 메인 로그 (app.log)
```
2025-02-27 10:30:45 [INFO] llm_config - 🔄 Google Gemini 초기화: gemini-2.5-flash
2025-02-27 10:30:46 [INFO] sql_executor - ✅ 데이터베이스 엔진 초기화 완료
2025-02-27 10:31:02 [INFO] nodes - 질문 정제: '23년 매출은?' → '2023년 매출은?'
2025-02-27 10:31:03 [INFO] sql_generator - ✅ SQL 생성 완료
2025-02-27 10:31:04 [INFO] nodes - 📊 차트 타입 선택: LINE (추이)
```

### 학습 로그 (logs/training_data.jsonl)
```jsonl
{"timestamp":"2025-02-27T10:31:04.123456","question":"23년 매출은?","generated_sql":"SELECT SUM(amount) FROM sales WHERE year=2023","schema_snapshot":"...","execution_success":true,"execution_result_summary":"1행 반환: $1M","user_satisfaction":null,"error_details":null,"metadata":{"retry_count":0,"cache_hit":false}}
{"timestamp":"2025-02-27T10:31:15.456789","question":"23년 매출은?","generated_sql":"SELECT SUM(amount) FROM sales WHERE year=2023","schema_snapshot":"...","execution_success":true,"execution_result_summary":"1행 반환: $1M","user_satisfaction":1,"error_details":null,"metadata":{"retry_count":0,"cache_hit":false}}
```

---

## 🔐 SQL 검증 계층 (방어 심화)

7개의 검증 레이어로 안전성 확보:

| 순서 | 검증 항목 | 목적 | 예시 |
|------|---------|------|------|
| 1 | sqlglot 구문 분석 | 진정한 SQL 문법 에러 감지 | "SELEXT ..." → 거부 |
| 2 | SELECT * 감지 | 대용량 반환 방지 | 자동 EXPAND |
| 3 | GROUP BY 검증 | 집계 함수 오류 감지 | "SUM(x) GROUP BY" |
| 4 | Cartesian Product | 누락된 JOIN ON 감지 | 테이블 3개+, JOIN ON 0 → 거부 |
| 5 | 테이블/컬럼 존재 | 존재하지 않는 참조 차단 | `invalid_col` → 거부 |
| 6 | JOIN 키 매핑 | 올바른 JOIN 조건 검증 | 테이블 간 매핑 확인 |
| 7 | 자동 LIMIT | 결과 폭발 방지 | 항상 LIMIT 1000 추가 |

**⚠️ 더 이상 SELECT/WITH 전용이 아님** → 더 유연한 쿼리 지원

---

## 🎯 주요 기능

### 모듈별 기능

| 모듈 | 파일 | 기능 |
|------|------|------|
| **models** | llm_config.py | LLM 관리 (Google/Ollama/OpenAI) |
| | vector_db.py | ChromaDB 벡터 DB 관리 |
| **sql** | sql_validator.py | SQL 7단계 검증 (SELECT/WITH 제한 제거) |
| | sql_generator.py | LLM + RAG 기반 SQL 생성 |
| | sql_executor.py | DB 연결 + 학습 로그 자동 기록 |
| **rag** | retrievers.py | 6가지 컬렉션 하이브리드 검색 + 리랭킹 |
| **agent** | nodes.py | 7개 워크플로우 노드 |
| | state.py | AgentState 상태 정의 |
| **cache** | query_cache.py | TTL 기반 쿼리 캐싱 |
| **logging** | logger_config.py | 메인/학습 로그 설정 |
| **utils** | helpers.py | 차트/설명/메모리 관리 |
| | **training_logger.py** | 🆕 **학습 데이터 JSONL 자동 수집** |

---

## 📈 성능 및 통계

### 학습 로그 통계

```python
stats = get_training_log_stats()
# {
#     'total_logs': 42,                    # 총 쿼리 수
#     'success_rate': 0.952,               # 성공률 95.2%
#     'avg_satisfaction': 0.8,             # 평균 평가 0.8
#     'satisfaction_count': {'1': 25, '-1': 5, 'null': 12},
#     'file_path': '/content/.../training_data.jsonl'
# }
```

### 데이터 내보내기

```python
# 성공 + 평가받은 샘플만
dataset = export_training_dataset(
    success_only=True,
    with_satisfaction_only=True,
)
# → 로컬 모델 파인튜닝에 사용
```

---

## 🧪 테스트

```bash
# SQL 검증
python -c "
from sql import validate_sql_static
sql = 'SELECT part_number, SUM(sale_quantity) FROM sales_orders GROUP BY part_number'
is_valid, errors, strategy = validate_sql_static(sql, {})
print(f'Valid: {is_valid}, Errors: {errors}')
"

# 학습 로그
python -c "
from utils.training_logger import get_training_log_stats
stats = get_training_log_stats()
print(f'Total logs: {stats[\"total_logs\"]}')
print(f'Success rate: {stats[\"success_rate\"]:.2%}')
print(f'Avg satisfaction: {stats[\"avg_satisfaction\"]}')
"

# LLM 테스트
python -c "
from models import get_default_llm
llm = get_default_llm()
response = llm.invoke('안녕하세요')
print(response.content)
"

# RAG 검색
python -c "
from rag import retrieve_parallel
results = retrieve_parallel('매출액에 따른 고객사 순위', '')
for key, val in results.items():
    print(f'{key}: {val[\"score\"]:.3f}')
"
```

---

## 📝 설정 변경 가이드

### 1. LLM 모델 변경

**파일**: `config.py`

```python
# Gemini → Ollama 변경
DEFAULT_LLM_CONFIG = LLMConfig(
    provider="ollama",
    model_name="mistral",
    temperature=0.3,
)
```

### 2. ChromaDB 경로 변경

```python
CHROMA_CONFIG = {
    "PATH": "/new/path/to/vdata",
    "EMBEDDING_MODEL": "BAAI/bge-m3",
}
```

### 3. DB 연결 설정

```python
DATABASE_CONFIG = {
    "USER": "postgres",
    "PASSWORD": "secure_password",
    "HOST": "192.168.1.100",  # 원격 DB
    "PORT": "5432",
}
```

---

## 📚 데이터 구조

### 학습 데이터 JSON 스키마

```json
{
  "timestamp": "2025-02-27T10:30:45.123456",
  "question": "23년 매출은?",
  "generated_sql": "SELECT SUM(sale_amount) FROM sales WHERE year=2023",
  "schema_snapshot": "sales(sale_id INT, sale_amount DECIMAL, year INT), ...",
  "execution_success": true,
  "execution_result_summary": "1행 반환: $1,234,567",
  "user_satisfaction": 1,
  "error_details": null,
  "metadata": {
    "retry_count": 0,
    "cache_hit": false,
    "execution_time_ms": 245,
    "chart_type": "bar"
  }
}
```

### 차트 정보 구조

```python
chart_info = {
    "type": "bar",           # "pie", "bar", "line", "table", "none"
    "x": "month",            # X축 컬럼
    "y": "amount",           # Y축 컬럼 또는 리스트
    "title": "월별 매출 추이" # 질문 기반
}
```

---

## 🔗 의존성

```
langchain-google-genai          # Google Gemini API
langchain-core                  # LangChain core
langchain-community             # LangChain community
langgraph                       # LangGraph workflow
sqlalchemy                      # SQL toolkit
psycopg2-binary                # PostgreSQL driver
chromadb                       # Vector database
sentence-transformers          # Embedding models
rapidfuzz                       # Fuzzy matching
rank-bm25                       # BM25 ranking
FlagEmbedding                   # BGE embeddings
sqlglot                         # SQL parsing
streamlit                       # Web UI
pandas                          # Data processing
python-dotenv                   # Environment variables
```

---

## 📞 주요 파일 참고

- **설정**: [config.py](config.py)
- **메인 앱**: [app.py](app.py)
- **LLM 설정**: [models/llm_config.py](models/llm_config.py)
- **SQL 처리**: [sql/sql_validator.py](sql/sql_validator.py), [sql/sql_executor.py](sql/sql_executor.py)
- **RAG 검색**: [rag/retrievers.py](rag/retrievers.py)
- **워크플로우**: [agent/nodes.py](agent/nodes.py)
- **학습 로거**: [utils/training_logger.py](utils/training_logger.py) ✨ NEW

---

## 📊 버전 히스토리

| 버전 | 날짜 | 주요 기능 | 상태 |
|------|------|---------|------|
| **1.0** | 2025-02-26 | 초기 모듈화 | ✅ |
| **2.0** | 2025-02-27 | 학습 로거 + 피드백 UI + 자동 차트 | ✅ |

---

**Version**: 2.0.0  
**Created**: 2025-02-26  
**Last Updated**: 2025-02-27
