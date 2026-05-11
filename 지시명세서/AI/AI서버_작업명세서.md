# AI 서버 작업명세서

**문서번호:** DS-AI-SPEC-001 v1.0  
**작성일:** 2026-05-11  
**프로젝트:** 반도체 후공정 비전 검사 장비 — AI 분석 서버  
**대외비**

| 항목 | 내용 |
| :--- | :--- |
| 장비 모델 | Genesem VELOCE-G7 Saw Singulation |
| 비전 소프트웨어 | GVisionWpf (C# WPF, HALCON + MQTTnet) |
| 현장 실측 기준 | Carsem Inc. / 2026-01-16~29 (14일) |
| 개발 언어 | Python 3.11+ |
| 프레임워크 | FastAPI |
| LLM 모델 | Claude Sonnet 4.6 (Anthropic API) |
| 임베딩 모델 | `intfloat/multilingual-e5-small` (로컬, CPU 전용, 384차원) |
| DB | PostgreSQL + pgvector (벡터 DB) |
| 배포 환경 | AWS Lightsail (CPU-only, GPU 없음) |
| 네트워크 환경 | Online Area (인터넷 접근 가능) |

---

## 1. 개요

### 1.1 목적

AI 서버는 Dispatcher 서버로부터 비식별화된 LOT 배치 데이터를 수신하여 전처리·벡터 임베딩 후 Vector DB에 적재하고, 웹 백엔드 서버의 요청 시 **RAG + CAG** 방식으로 LLM 기반 자동 리서치 보고서를 생성하는 **Online Area 지능형 분석 서버**이다.

### 1.2 핵심 임무

- **데이터 수신 및 적재**: Dispatcher의 단방향 Push(`POST /api/ingest`)를 수신하여 비식별화된 검사 데이터를 Vector DB에 임베딩
- **자동 리서치 보고서 생성**: 일별/주간 주기로 LOT 시계열 데이터를 분석하여 자연어 보고서 자동 생성
- **RAG 질의 응답**: 웹 백엔드의 요청 시 Vector DB 검색 → Context 조합 → LLM 추론 → 인사이트 응답

### 1.3 시스템 내 위치

```
[Local Area]
  Historian TSDB ──(read-only)──→ Dispatcher 서버
                                      │
                                      │ HTTPS POST /api/ingest
                                      │ (DispatchBatch: 5개 테이블 통합 JSON)
                                      ▼
[Online Area]
  AI 서버 (본 프로젝트) ←── 너가 만드는 것
      │
      ├── Vector DB (pgvector) ── 임베딩 적재/검색
      │
      ├── Anthropic API (Claude Sonnet 4.6) ── LLM 추론
      │
      ├──→ 웹 백엔드 서버 ── 보고서 Push + 원본 배치 전달
      │         │
      │         ├── RDBMS ── 배치 원본 저장 (KPI 가공용)
      │         └──→ 웹 프론트 (클라이언트)
      │
      └── 원본 DispatchBatch 저장 (ingest_batches 테이블)
          → 웹 백엔드가 조회 또는 AI 서버가 Push
```

### 1.4 데이터 흐름

```
[수신 경로 — Dispatcher → AI 서버]
Dispatcher POST /api/ingest
  → API Key 인증
  → DispatchBatch 페이로드 파싱 (5개 테이블 통합)
  → 원본 배치 저장 (ingest_batches 테이블, 웹 백엔드 KPI 가공 소스)
  → 전처리 (통계 집계, 텍스트 청킹)
  → 로컬 임베딩 (sentence-transformers, CPU)
  → pgvector INSERT
  → 웹 백엔드에 배치 수신 알림 Push (POST {BACKEND_URL}/api/batches)

[분석 경로 — 웹 백엔드 → AI 서버]
웹 백엔드 POST /api/query
  → Vector DB 유사도 검색 (RAG)
  → Context 조합
  → Claude Sonnet 4.6 추론
  → 자연어 분석 JSON 응답

[스케줄 경로 — 자동 보고서]
Cron (일별/주간)
  → 저장된 배치 데이터 + Vector DB 기간별 조회
  → LOT/Oracle/Alarm/Status 통계 집계
  → Claude Sonnet 4.6 분석
  → 보고서 생성 → 웹 백엔드 Push (POST {BACKEND_URL}/api/reports)

[배치 조회 경로 — 웹 백엔드 → AI 서버]
웹 백엔드 GET /api/batches?equipmentId=...&from=...&to=...
  → 저장된 원본 DispatchBatch 조회 (KPI 가공용)
  → 웹 프론트 대시보드 데이터 소스
```

### 1.5 단일 책임 원칙

AI 서버는 **"분석과 보고서 생성"만** 수행한다. 데이터 적재(Historian), 비식별화·전송(Dispatcher), 사용자 관리·인증(웹 백엔드), UI 렌더링(웹 프론트)은 다른 서버의 책임이다.

---

## 2. 참조 문서

| 우선순위 | 문서 | 참조 목적 |
| :--- | :--- | :--- |
| 1 | `문서/기획안.md` | AI 서버 목적, RAG/CAG 방식, 데이터 흐름 전체 |
| 2 | `dispatcher/src/types/index.ts` | DispatchBatch 수신 페이로드 구조 (비식별화 필드 정의) |
| 3 | `dispatcher/src/push/aiClient.ts` | Dispatcher→AI 서버 Push 프로토콜, 인증 헤더, 재시도 정책 |
| 4 | `dispatcher/.env.example` | AI_SERVER_URL, API_KEY 규격 |
| 5 | `명세서/DS_EAP_MQTT_API_명세서.md` v3.4 | 원본 이벤트 페이로드 필드 이해 (비식별화 전 구조) |
| 6 | `문서/오라클 2차 검증 기획안.md` | LOT 보고서 구조, 보고서 유형(LOT/일별/주간) |

---

## 3. 기능 요구사항

### 3.1 데이터 수신 API (Ingest Endpoint)

Dispatcher 서버가 LOT 단위 배치를 Push하는 유일한 진입점이다.

| 항목 | 규격 |
| :--- | :--- |
| Endpoint | `POST /api/ingest` |
| Content-Type | `application/json` |
| 인증 | `X-Api-Key` 헤더 (환경변수 `AI_INGEST_API_KEY`) |
| 요청 본문 | `DispatchBatch` JSON (§3.1.1 참조) |
| 성공 응답 | `200 OK` — `{ "status": "accepted", "batchId": "..." }` |
| 클라이언트 오류 | `400 Bad Request` — 페이로드 검증 실패 |
| 인증 실패 | `401 Unauthorized` — API Key 불일치 |
| 서버 오류 | `500 Internal Server Error` — Dispatcher가 재시도 (4xx는 재시도 안 함) |

#### 3.1.1 DispatchBatch 수신 페이로드 구조

Dispatcher의 최종 코드 기준, 5개 테이블(lot_ends, inspection_results, oracle_analyses, status_updates, hw_alarms)을 조회하여 아래 구조로 전송한다. 모든 식별 정보는 HMAC-SHA256으로 비식별화되어 수신된다.

```python
class DispatchBatch(BaseModel):
    batchId: str              # UUID v4
    dispatchedAt: str         # ISO 8601 UTC
    lotHash: str              # lot_id HMAC 해시 (원본 lot_id 없음)
    equipmentHash: str        # equipment_id HMAC 해시
    equipmentId: str          # 장비 원본 ID (plaintext 모드 — 웹 프론트 표시용)
    totalRecords: int
    records: list[AnonymizedInspectionRecord]
    lotSummary: AnonymizedLotRecord
    oracleAnalysis: list[OracleAnalysisRecord]     # oracle_analyses 테이블
    statusHistory: list[StatusHistoryRecord]        # status_updates 테이블
    alarmHistory: list[AlarmHistoryRecord]          # hw_alarms 테이블

# ── inspection_results 테이블 ──
class AnonymizedInspectionRecord(BaseModel):
    message_id: str
    lotHash: str
    equipmentHash: str
    strip_id: int             # 순번(sequence)으로 대체됨
    unit_id: int              # 순번(sequence)으로 대체됨
    overall_result: str       # PASS / FAIL
    time: str
    fail_reason_code: str | None
    fail_count: int
    total_inspected_count: int
    inspection_duration_ms: int
    takt_time_ms: int
    algorithm_version: str
    inspection_detail: dict | None   # PascalCase 원본 (PASS시 null, PASS drop 정책)
    geometric: dict | None
    bga: dict | None
    surface: dict | None
    singulation: dict | None

# ── lot_ends 테이블 ──
class AnonymizedLotRecord(BaseModel):
    lotHash: str
    equipmentHash: str
    lot_status: str           # COMPLETED / ABORTED
    recipe_id: str            # 비식별화 안 됨 (공정 분석에 필수)
    total_units: int
    pass_count: int
    fail_count: int
    yield_pct: float
    lot_duration_sec: int

# ── oracle_analyses 테이블 ──
class OracleAnalysisRecord(BaseModel):
    message_id: str
    time: str
    lot_id_hash: str
    judgment: str             # NORMAL / WARNING / DANGER
    yield_pct: float
    ai_comment: str | None
    violated_rules: dict | None   # 위반 Rule 목록 (JSONB)
    analysis_source: str          # rule_based / ewma_mad / composite

# ── status_updates 테이블 ──
class StatusHistoryRecord(BaseModel):
    time: str
    equipment_status: str     # RUN / IDLE / STOP
    lot_id: str | None
    recipe_id: str | None
    uptime_sec: int
    current_unit_count: int | None
    expected_total_units: int | None
    current_yield_pct: float | None

# ── hw_alarms 테이블 ──
class AlarmHistoryRecord(BaseModel):
    time: str
    alarm_level: str          # CRITICAL / WARNING / INFO
    hw_error_code: str
    hw_error_source: str      # CAMERA / LIGHTING / VISION / PROCESS / MOTION / COOLANT
    hw_error_detail: str
    auto_recovery_attempted: bool
    requires_manual_intervention: bool
    burst_id: str | None
    burst_count: int | None
```

> **보안 주의**: `operator_id` 원본은 Dispatcher에서 제거된다. `equipment_id` 원본은 plaintext 모드로 전달되어 웹 프론트에서 장비명 표시에 사용된다. `lot_id`는 HMAC 해시로만 전달된다.

### 3.2 데이터 전처리 및 임베딩 파이프라인

수신된 `DispatchBatch`를 Vector DB에 적재하기 위한 전처리 파이프라인이다.

#### 3.2.1 전처리 단계

```
DispatchBatch 수신 (5개 테이블 통합)
    │
    ├── 0. 원본 배치 저장 (ingest_batches 테이블 — 웹 백엔드 KPI 가공 소스)
    │       ├── lotSummary, oracleAnalysis, statusHistory, alarmHistory → JSONB
    │       └── records → 집계 통계만 저장 (원본 레코드는 임베딩 전용)
    │
    ├── 1. 유효성 검증 (Pydantic 스키마 검증)
    │
    ├── 2. 통계 집계
    │       ├── [records] 수율, FAIL fail_reason_code 분포
    │       ├── [records] takt_time_ms / inspection_duration_ms P50/P95/P99
    │       ├── [records] singulation 지표 (chipping_top_um, burr_height_um 등)
    │       ├── [records] geometric 지표 (dimension, offset 등)
    │       ├── [oracleAnalysis] judgment 분포 (NORMAL/WARNING/DANGER)
    │       ├── [oracleAnalysis] violated_rules 집계, ai_comment 수집
    │       ├── [statusHistory] RUN/IDLE/STOP 비율, 가동률, 다운타임 합산
    │       └── [alarmHistory] alarm_level 분포, hw_error_code 집계, MTBF 계산
    │
    ├── 3. 텍스트 청킹 (Chunking — 7종)
    │       ├── LOT 요약 청크: "LOT {lotHash} | equip={equipmentId} | recipe={recipe_id} | yield={yield_pct}%"
    │       ├── 불량 분석 청크: "ErrorType 분포: ET=52 {n}건, ET=30 {n}건, ..."
    │       ├── 절삭 품질 청크: "chipping_top P95={x}um, burr_height P95={x}um, ..."
    │       ├── 프로세스 청크: "takt_time P95={x}ms, inspection_duration P95={x}ms"
    │       ├── Oracle 판정 청크: "judgment={judgment} | violated_rules=[...] | ai_comment=..."
    │       ├── 가동 이력 청크: "availability={x}% | downtime={x}min | RUN/STOP 패턴"
    │       └── 알람 이력 청크: "alarm_level={x} | hw_error={code} | burst_count={n} | MTBF={x}h"
    │
    ├── 4. 벡터 임베딩 (로컬 모델: sentence-transformers, CPU 전용)
    │       └── 각 청크 → 384차원 벡터 (intfloat/multilingual-e5-small)
    │
    ├── 5. pgvector INSERT
    │       ├── lot_embeddings 테이블
    │       └── 메타데이터 (lotHash, equipmentId, recipe_id, yield_pct, time)
    │
    └── 6. 웹 백엔드 배치 수신 알림
            POST {BACKEND_URL}/api/batches/notify
            { "batchId": "...", "equipmentId": "...", "yieldPct": 96.2, "judgment": "NORMAL" }
```

#### 3.2.2 PASS drop 정책 반영

Historian → Dispatcher 경로에서 PASS drop 정책이 적용된 상태로 데이터가 도착한다. AI 서버는 이를 인지하고 분석해야 한다.

| 조건 | 수신 데이터 범위 | 분석 가능 범위 |
| :--- | :--- | :--- |
| PASS (fail_count = 0) | summary + process 필드만 | 수율, takt_time, duration 통계만 가능 |
| FAIL (fail_count ≥ 1) | 전체 필드 | 불량 원인, 절삭 품질, BGA, 표면 분석 가능 |

### 3.3 배치 데이터 조회 API (웹 백엔드 KPI 가공용)

웹 백엔드가 프론트엔드 대시보드·리포트에 필요한 원본 배치 데이터를 조회하는 API이다. AI 서버가 Dispatcher로부터 수신한 DispatchBatch 원본(lotSummary, oracleAnalysis, statusHistory, alarmHistory)을 저장해두고 웹 백엔드에 공급한다.

| Endpoint | Method | 설명 | 인증 |
| :--- | :--- | :--- | :--- |
| `/api/batches` | GET | 배치 목록 조회 (기간/장비 필터, 페이징) | 서비스 JWT |
| `/api/batches/{batchId}` | GET | 배치 상세 조회 (원본 전체) | 서비스 JWT |
| `/api/batches/latest` | GET | 장비별 최신 배치 조회 | 서비스 JWT |
| `/api/batches/kpi-summary` | GET | 기간별 KPI 집계 (수율, UPH, 가동률 등) | 서비스 JWT |

**쿼리 파라미터:**

| 파라미터 | 타입 | 설명 | 기본값 |
| :--- | :--- | :--- | :--- |
| `equipmentId` | string | 장비 ID 필터 | 전체 |
| `from` | ISO 8601 | 시작 시각 | 24시간 전 |
| `to` | ISO 8601 | 종료 시각 | 현재 |
| `page` | int | 페이지 번호 | 1 |
| `size` | int | 페이지 크기 | 50 |

**KPI 집계 응답 (`/api/batches/kpi-summary`):**

이 엔드포인트는 웹 백엔드가 프론트엔드 대시보드 KPI Scorecard에 표시할 집계 데이터를 한 번에 제공한다. 검증문서 §2-1의 30개 KPI 항목을 커버한다.

```python
class KpiSummaryResponse(BaseModel):
    period: ReportPeriod
    # ── 생산 KPI ──
    totalUnits: int                    # lotSummary.total_units 합산
    totalInspected: int                # records 배열 길이 합산
    totalFail: int                     # lotSummary.fail_count 합산
    avgYieldPct: float                 # lotSummary.yield_pct 평균
    avgUph: float                      # total_units / lot_duration_sec 평균
    # ── Oracle 판정 KPI ──
    marginalCount: int                 # oracleAnalysis[].violated_rules에서 yield_grade=MARGINAL
    dangerCount: int                   # judgment=DANGER 수
    warningCount: int                  # judgment=WARNING 수
    # ── 장비 가동 KPI ──
    avgAvailabilityPct: float          # statusHistory[] RUN 비율
    totalDowntimeMin: float            # statusHistory[] STOP 구간 합산 (분)
    activeEquipmentCount: int          # 최신 status=RUN 장비 수
    totalEquipmentCount: int           # 전체 장비 수
    avgMtbfHours: float | None         # alarmHistory[] 간 interval 평균
    # ── 불량 분석 KPI ──
    topFailReasons: list[FailReasonCount]   # fail_reason_code 상위 5건
    # ── 장비별 상세 ──
    equipmentDetails: list[EquipmentKpi]    # 장비별 수율/UPH/알람 요약
```

> **설계 의도**: 웹 백엔드가 매번 전체 배치를 순회하여 집계하는 대신, AI 서버가 ingest 시점에 통계를 사전 계산해두고 이 API로 제공한다. 프론트엔드 대시보드 렌더링 시 웹 백엔드→AI 서버 1회 호출로 모든 KPI를 확보할 수 있다.

### 3.4 RAG 질의 응답 API

웹 백엔드 서버가 사용자 질의를 전달하면, Vector DB 검색 → LLM 추론을 수행하여 응답한다.

| 항목 | 규격 |
| :--- | :--- |
| Endpoint | `POST /api/query` |
| 인증 | `Authorization: Bearer {JWT}` (웹 백엔드 서버 간 토큰) |
| 요청 본문 | `{ "question": "...", "filters": { "recipe_id": "...", "date_range": [...] } }` |
| 응답 | `{ "answer": "...", "sources": [...], "confidence": 0.85 }` |

#### 3.3.1 RAG 파이프라인

```
사용자 질의 수신
    │
    ├── 1. 질의 임베딩 (동일 로컬 모델: intfloat/multilingual-e5-small)
    │
    ├── 2. pgvector 유사도 검색
    │       ├── cosine similarity top-K (K=10)
    │       ├── 필터: recipe_id, date_range, equipmentHash
    │       └── 메타데이터 기반 re-ranking
    │
    ├── 3. Context 조합
    │       ├── 검색된 청크 + LOT 요약 통계
    │       └── 토큰 제한 내 최대 context 구성
    │
    ├── 4. Claude Sonnet 4.6 추론
    │       ├── System prompt: 반도체 후공정 비전 검사 도메인 전문가
    │       ├── Context: 검색된 청크들
    │       └── User query: 사용자 질의
    │
    └── 5. 응답 구조화
            ├── answer: 자연어 분석 결과
            ├── sources: 참조 LOT 해시 목록
            └── confidence: 응답 신뢰도
```

### 3.4 자동 리서치 보고서 생성

스케줄러가 일별/주간 주기로 자동 실행하여 보고서를 생성한다.

| 보고서 유형 | 주기 | Endpoint | 트리거 |
| :--- | :--- | :--- | :--- |
| 일별 보고서 | 매일 00:00 UTC | `POST /api/report/daily` | 내부 Cron 또는 웹 백엔드 요청 |
| 주간 보고서 | 매주 월요일 00:00 UTC | `POST /api/report/weekly` | 내부 Cron 또는 웹 백엔드 요청 |

#### 3.4.1 보고서 생성 파이프라인

```
스케줄 트리거 (일별/주간)
    │
    ├── 1. 기간별 LOT 데이터 조회 (pgvector + 메타데이터)
    │
    ├── 2. 통계 집계
    │       ├── 전체 수율 추이 (일별/주별)
    │       ├── 레시피별 수율 비교
    │       ├── 장비별(해시) 성능 비교
    │       ├── 주요 불량 원인 Top-5
    │       └── 절삭 품질 트렌드
    │
    ├── 3. Claude Sonnet 4.6 분석
    │       ├── System prompt: 보고서 생성 전문가
    │       ├── Context: 집계된 통계 + 과거 보고서 참조
    │       └── 출력: 구조화된 보고서 JSON
    │
    └── 4. 웹 백엔드로 Push
            POST {BACKEND_URL}/api/reports
            { "type": "daily|weekly", "generatedAt": "...", "content": {...} }
```

#### 3.4.2 보고서 출력 구조

```python
class AnalysisReport(BaseModel):
    reportId: str                  # UUID
    reportType: str                # "daily" | "weekly"
    generatedAt: str               # ISO 8601
    period: ReportPeriod           # { start, end }
    summary: str                   # 핵심 요약 (1-2 문단)
    metrics: ReportMetrics         # 정량 지표
    insights: list[Insight]        # AI 인사이트 목록
    recommendations: list[str]     # 개선 권고사항
    lotDetails: list[LotBrief]     # 개별 LOT 요약

class ReportMetrics(BaseModel):
    # ── 생산 지표 ──
    totalLots: int
    avgYieldPct: float
    minYieldPct: float
    maxYieldPct: float
    totalFailCount: int
    avgUph: float
    topFailReasons: list[FailReasonCount]
    recipeBreakdown: list[RecipeMetric]
    # ── Oracle 판정 지표 ──
    judgmentDistribution: dict         # { "NORMAL": n, "WARNING": n, "DANGER": n }
    marginalCount: int                 # yield_grade=MARGINAL 수
    # ── 장비 가동 지표 ──
    avgAvailabilityPct: float          # statusHistory 기반 가동률
    totalDowntimeMin: float            # STOP 구간 합산 (분)
    avgMtbfHours: float | None         # alarmHistory 간 interval 평균
    # ── 장비별 상세 ──
    equipmentBreakdown: list[EquipmentMetric]  # 장비별 수율/UPH/알람 요약

class Insight(BaseModel):
    severity: str                  # "info" | "warning" | "critical"
    category: str                  # "yield" | "quality" | "equipment" | "recipe"
    message: str                   # 자연어 인사이트
    evidence: list[str]            # 근거 데이터
```

---

## 4. DB 스키마 설계

### 4.1 PostgreSQL + pgvector

```sql
-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- 4.1.1 LOT 임베딩 테이블
CREATE TABLE lot_embeddings (
    id              SERIAL          PRIMARY KEY,
    batch_id        UUID            NOT NULL,        -- DispatchBatch.batchId
    lot_hash        TEXT            NOT NULL,         -- 비식별화된 LOT 해시
    equipment_hash  TEXT            NOT NULL,         -- 비식별화된 장비 해시
    equipment_id    TEXT            NOT NULL,         -- 장비 원본 ID (plaintext, 검색/필터용)
    recipe_id       TEXT            NOT NULL,         -- 레시피 ID (분석용, 비식별화 안 됨)
    chunk_type      TEXT            NOT NULL,         -- 7종: lot_summary | fail_analysis | quality | process | oracle | availability | alarm
    chunk_text      TEXT            NOT NULL,         -- 임베딩 원본 텍스트
    embedding       vector(384)     NOT NULL,         -- 벡터 임베딩 (intfloat/multilingual-e5-small, CPU 전용)
    yield_pct       DOUBLE PRECISION,                 -- 메타데이터 필터용
    lot_status      TEXT,                             -- COMPLETED / ABORTED
    total_units     INTEGER,
    fail_count      INTEGER,
    ingested_at     TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    dispatched_at   TIMESTAMPTZ     NOT NULL          -- DispatchBatch.dispatchedAt
);

CREATE INDEX idx_lot_emb_recipe ON lot_embeddings (recipe_id, ingested_at DESC);
CREATE INDEX idx_lot_emb_equip ON lot_embeddings (equipment_hash, ingested_at DESC);
CREATE INDEX idx_lot_emb_yield ON lot_embeddings (yield_pct);
CREATE INDEX idx_lot_emb_vector ON lot_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- 4.1.2 보고서 이력 테이블
CREATE TABLE analysis_reports (
    id              SERIAL          PRIMARY KEY,
    report_id       UUID            NOT NULL UNIQUE,
    report_type     TEXT            NOT NULL,         -- 'daily' | 'weekly'
    period_start    TIMESTAMPTZ     NOT NULL,
    period_end      TIMESTAMPTZ     NOT NULL,
    content         JSONB           NOT NULL,         -- AnalysisReport 전체
    generated_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    pushed_to_backend BOOLEAN       NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_report_type_time ON analysis_reports (report_type, generated_at DESC);

-- 4.1.3 수신 배치 이력 테이블 (중복 수신 방지 + 원본 보존)
CREATE TABLE ingest_batches (
    batch_id        UUID            PRIMARY KEY,
    lot_hash        TEXT            NOT NULL,
    equipment_hash  TEXT            NOT NULL,
    equipment_id    TEXT            NOT NULL,         -- plaintext 장비 ID
    total_records   INTEGER         NOT NULL,
    lot_summary     JSONB           NOT NULL,         -- AnonymizedLotRecord 원본
    oracle_analysis JSONB,                            -- OracleAnalysisRecord[] 원본
    status_history  JSONB,                            -- StatusHistoryRecord[] 원본
    alarm_history   JSONB,                            -- AlarmHistoryRecord[] 원본
    records_summary JSONB           NOT NULL,         -- records[] 집계 통계 (원본 레코드는 임베딩만)
    dispatched_at   TIMESTAMPTZ     NOT NULL,
    ingested_at     TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    pushed_to_backend BOOLEAN       NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_ingest_equip_time ON ingest_batches (equipment_id, dispatched_at DESC);
CREATE INDEX idx_ingest_pushed ON ingest_batches (pushed_to_backend) WHERE pushed_to_backend = FALSE;
```

---

## 5. 프로젝트 구조

```
ai-server/
├── pyproject.toml
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── alembic/                          ← DB 마이그레이션
│   └── versions/
├── src/
│   ├── __init__.py
│   ├── main.py                       ← FastAPI 앱 엔트리포인트
│   ├── config.py                     ← pydantic-settings 환경변수
│   ├── api/
│   │   ├── __init__.py
│   │   ├── ingest.py                 ← POST /api/ingest (Dispatcher 수신)
│   │   ├── batches.py                ← GET /api/batches/* (웹 백엔드 KPI 조회용)
│   │   ├── query.py                  ← POST /api/query (RAG 질의)
│   │   ├── report.py                 ← POST /api/report/{type}
│   │   └── health.py                 ← GET /health
│   ├── models/
│   │   ├── __init__.py
│   │   ├── dispatch_batch.py         ← DispatchBatch + 5개 하위 모델 (Pydantic)
│   │   ├── kpi.py                    ← KpiSummaryResponse, EquipmentKpi 모델
│   │   ├── query.py                  ← 질의/응답 모델
│   │   └── report.py                 ← AnalysisReport 모델
│   ├── pipeline/
│   │   ├── __init__.py
│   │   ├── preprocessor.py           ← 전처리 (통계 집계, 청킹)
│   │   ├── embedder.py               ← 벡터 임베딩 (로컬 sentence-transformers)
│   │   └── chunker.py                ← 텍스트 청킹 전략
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── retriever.py              ← pgvector 유사도 검색
│   │   ├── reranker.py               ← 메타데이터 기반 re-ranking
│   │   └── context_builder.py        ← LLM 컨텍스트 조합
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py                 ← Anthropic API 클라이언트
│   │   ├── prompts.py                ← System/User 프롬프트 템플릿
│   │   └── report_generator.py       ← 보고서 생성 로직
│   ├── scheduler/
│   │   ├── __init__.py
│   │   └── report_scheduler.py       ← APScheduler 일별/주간 크론
│   ├── db/
│   │   ├── __init__.py
│   │   ├── pool.py                   ← asyncpg 커넥션 풀
│   │   ├── embeddings.py             ← lot_embeddings CRUD
│   │   ├── batches.py                ← ingest_batches CRUD + KPI 집계 쿼리
│   │   └── reports.py                ← analysis_reports CRUD
│   └── utils/
│       ├── __init__.py
│       ├── logging_config.py         ← structlog 설정
│       └── auth.py                   ← API Key / JWT 검증
└── tests/
    ├── test_ingest.py
    ├── test_preprocessor.py
    ├── test_retriever.py
    └── test_report_generator.py
```

---

## 6. 환경변수

```env
# FastAPI
APP_HOST=0.0.0.0
APP_PORT=8000
APP_ENV=production

# 인증 — Dispatcher 수신용
AI_INGEST_API_KEY=CHANGE_ME

# 인증 — 웹 백엔드 간 통신용
BACKEND_JWT_SECRET=CHANGE_ME_32_CHARS_MIN
BACKEND_SERVER_URL=http://web-backend:8080

# PostgreSQL + pgvector
PG_HOST=localhost
PG_PORT=5432
PG_NAME=ai_server
PG_USER=ai_server
PG_PASSWORD=CHANGE_ME
PG_POOL_MIN=2
PG_POOL_MAX=10

# Anthropic API
ANTHROPIC_API_KEY=CHANGE_ME
ANTHROPIC_MODEL=claude-sonnet-4-6
ANTHROPIC_MAX_TOKENS=4096

# 임베딩 모델 (로컬 sentence-transformers, CPU 전용)
EMBEDDING_MODEL=intfloat/multilingual-e5-small
EMBEDDING_DIMENSION=384
EMBEDDING_DEVICE=cpu                 # AWS Lightsail: GPU 없음, CPU 고정
EMBEDDING_BATCH_SIZE=16              # CPU 메모리 고려 배치 크기
EMBEDDING_MAX_SEQ_LENGTH=512         # 최대 토큰 길이
EMBEDDING_USE_ONNX=true              # ONNX Runtime CPU 최적화 (추론 2~3배 가속)

# 스케줄러
DAILY_REPORT_CRON=0 0 * * *
WEEKLY_REPORT_CRON=0 0 * * 1

# 로깅
LOG_LEVEL=INFO
LOG_FORMAT=json
```

---

## 7. 기술 스택

| 항목 | 기술 | 버전 | 용도 |
| :--- | :--- | :--- | :--- |
| 언어 | Python | 3.11+ | 기획서 확정 스택 |
| 웹 프레임워크 | FastAPI | 0.111+ | 비동기 REST API |
| ASGI 서버 | uvicorn | 0.30+ | 프로덕션 서버 |
| DB 드라이버 | asyncpg | 0.29+ | PostgreSQL 비동기 커넥션 풀 |
| 벡터 검색 | pgvector | 0.7+ | 코사인 유사도 검색 |
| ORM/마이그레이션 | SQLAlchemy + Alembic | 2.0+ | 스키마 관리 |
| LLM 클라이언트 | anthropic | 0.34+ | Claude API 호출 |
| 임베딩 | sentence-transformers | 3.0+ | 로컬 임베딩 모델 (intfloat/multilingual-e5-small) |
| CPU 추론 최적화 | onnxruntime + optimum | 1.18+ | ONNX 변환 CPU 추론 가속 (GPU 없는 Lightsail 환경) |
| 텐서 연산 | torch (CPU-only) | 2.3+ | sentence-transformers 의존성 (CUDA 미포함 빌드) |
| 스케줄러 | APScheduler | 3.10+ | Cron 기반 보고서 생성 |
| HTTP 클라이언트 | httpx | 0.27+ | 웹 백엔드 Push |
| 설정 관리 | pydantic-settings | 2.4+ | .env 타입 안전 설정 |
| 로깅 | structlog | 24.1+ | 구조적 JSON 로깅 |
| 검증 | pydantic | 2.8+ | 페이로드 검증 |
| 테스트 | pytest + pytest-asyncio | 8.2+ | 비동기 테스트 |
| 컨테이너 | Docker + Compose | | 배포 환경 |

---

## 8. Task 분할

### Task A1 — 프로젝트 초기화 및 Ingest API

**목표:** FastAPI 프로젝트 구조 생성, `POST /api/ingest` 엔드포인트 구현, DispatchBatch 수신·검증·저장

**구현 사항:**
- `src/main.py`: FastAPI 앱 + lifespan (DB 풀 초기화/종료, 임베딩 모델 로드)
- `src/config.py`: pydantic-settings 환경변수 관리
- `src/models/dispatch_batch.py`: DispatchBatch + 5개 하위 모델 (AnonymizedInspectionRecord, AnonymizedLotRecord, OracleAnalysisRecord, StatusHistoryRecord, AlarmHistoryRecord)
- `src/api/ingest.py`: POST /api/ingest — API Key 인증, 5개 테이블 통합 페이로드 검증, 중복 batchId 체크, ingest_batches 원본 저장
- `src/api/batches.py`: GET /api/batches, /api/batches/{batchId}, /api/batches/latest, /api/batches/kpi-summary
- `src/api/health.py`: GET /health — DB 연결 + 임베딩 모델 로드 상태 확인
- `src/db/pool.py`: asyncpg 커넥션 풀 (min=2, max=10)
- `src/db/batches.py`: ingest_batches CRUD + KPI 집계 쿼리
- `src/utils/auth.py`: X-Api-Key 헤더 검증 (Dispatcher), 서비스 JWT 검증 (웹 백엔드)
- `docker-compose.yml`: PostgreSQL + pgvector 컨테이너

**검증 체크리스트:**
- [ ] `POST /api/ingest` 5개 테이블 통합 페이로드 → 200 OK + ingest_batches 저장
- [ ] API Key 누락/불일치 → 401 Unauthorized
- [ ] 중복 batchId 전송 → 409 Conflict (멱등성)
- [ ] oracleAnalysis/statusHistory/alarmHistory 누락 → 빈 배열로 허용 (Optional)
- [ ] 잘못된 JSON 스키마 → 400 Bad Request (Pydantic 에러 상세)
- [ ] DB 연결 실패 → 500 + 구조적 에러 로그 (Dispatcher 재시도 유도)
- [ ] GET /api/batches/kpi-summary?equipmentId=DS-VIS-001 → KPI 집계 응답
- [ ] GET /health → DB + 임베딩 모델 상태 포함 200 OK

---

### Task A2 — 전처리 및 임베딩 파이프라인

**목표:** DispatchBatch를 청킹·임베딩하여 pgvector에 적재

**구현 사항:**
- `src/pipeline/preprocessor.py`: 5개 테이블 통합 통계 집계 (yield, fail분포, takt P95, singulation, oracle judgment, 가동률, MTBF)
- `src/pipeline/chunker.py`: 7종 청크 생성 (lot_summary, fail_analysis, quality, process, oracle, availability, alarm)
- `src/pipeline/embedder.py`: 로컬 임베딩 모델 로드 및 추론 (sentence-transformers, 서버 시작 시 모델 1회 로드, 배치 임베딩)
- `src/db/embeddings.py`: lot_embeddings 테이블 CRUD (벌크 INSERT)
- PASS drop 정책: PASS 레코드는 process 청크만 생성, FAIL은 전체 상세 필드 포함
- oracleAnalysis 없는 배치: oracle 청크 생략 (빈 배열 허용)
- statusHistory/alarmHistory 없는 배치: availability/alarm 청크 생략

**임베딩 모델 상세:**

| 항목 | 값 |
| :--- | :--- |
| 모델 | `intfloat/multilingual-e5-small` |
| 차원 | 384 |
| 모델 크기 | ~470MB (메모리) |
| 최대 시퀀스 길이 | 512 토큰 |
| 다국어 지원 | ✅ (한국어/영어 혼합 청크 처리) |
| 로드 방식 | 서버 시작 시 1회 로드 → 메모리 상주 (Singleton) |
| 배치 크기 | 16 청크/배치 (Lightsail CPU 메모리 고려) |
| 디바이스 | CPU 고정 (`torch.device("cpu")`) |
| ONNX 최적화 | ✅ 권장 (`optimum[onnxruntime]`으로 CPU 추론 2~3배 가속) |
| 쿼리 prefix | `"query: "` (검색 시) |
| 패시지 prefix | `"passage: "` (적재 시) |

> **모델 선정 이유 (Lightsail CPU 환경 최적화):**
> - `e5-small`은 `e5-base`(768차원) 대비 절반 차원(384)으로, CPU 추론 속도가 약 2배 빠르고 메모리 사용량이 절반이다.
> - Lightsail 인스턴스(2~4GB RAM 기준)에서 모델 로드 후 서버 운영에 필요한 잔여 메모리를 확보할 수 있다.
> - 한국어·영어 혼합 텍스트 성능은 `e5-base`와 유의미한 차이가 없다 (MTEB 벤치마크 기준 ~2% 차이).
> - ONNX Runtime 변환 시 CPU에서 배치 16 기준 청크당 ~15ms 수준으로 처리 가능하다.
> - Dispatcher가 LOT 단위(4종 청크)로 Push하므로 실시간 대량 임베딩이 아닌 배치 적재 패턴에 적합하다.

**Lightsail 인스턴스 권장 사양:**

| 항목 | 최소 | 권장 |
| :--- | :--- | :--- |
| 인스턴스 | $20/월 (2 vCPU, 4GB RAM) | $40/월 (2 vCPU, 8GB RAM) |
| 디스크 | 60GB SSD | 80GB SSD |
| 메모리 배분 | 모델 ~470MB + PostgreSQL ~512MB + 앱 ~512MB | 여유 확보 |

**검증 체크리스트:**
- [ ] Mock DispatchBatch (yield 96.2%, FAIL 106건) → 4종 청크 생성
- [ ] PASS-only LOT → lot_summary + process 2종 청크만 생성
- [ ] 임베딩 모델 로드 실패 → 서버 시작 중단 + FATAL 로그
- [ ] 배치 16청크 임베딩 → 384차원 벡터 16개 반환 확인
- [ ] CPU 전용 추론 확인 (torch.cuda.is_available() == False 환경)
- [ ] ONNX 변환 모델 로드 시 추론 속도 < 20ms/청크
- [ ] pgvector INSERT 후 cosine similarity 검색 정상 동작

---

### Task A3 — RAG 질의 응답 API

**목표:** `POST /api/query` → Vector DB 검색 → LLM 추론 → 구조화된 응답

**구현 사항:**
- `src/api/query.py`: POST /api/query 엔드포인트
- `src/rag/retriever.py`: pgvector cosine similarity top-K 검색 + 메타데이터 필터
- `src/rag/reranker.py`: 시간 가중치 + 수율 관련도 re-ranking
- `src/rag/context_builder.py`: 검색 결과 → LLM 컨텍스트 조합 (토큰 제한 관리)
- `src/llm/client.py`: Anthropic API 클라이언트 (재시도, 타임아웃, 스트리밍)
- `src/llm/prompts.py`: 도메인 전문가 System 프롬프트 + 질의 템플릿

**검증 체크리스트:**
- [ ] "최근 Carsem_3X3 레시피 수율 추이는?" → 관련 LOT 데이터 기반 응답
- [ ] "ET=52 불량이 가장 많은 장비는?" → equipmentHash 기반 집계 응답
- [ ] 빈 Vector DB 상태 질의 → "분석 가능한 데이터가 없습니다" 안내
- [ ] Anthropic API 타임아웃 → 504 + 에러 로그

---

### Task A4 — 자동 보고서 생성 (일별/주간)

**목표:** Cron 스케줄 기반 자동 보고서 생성 → 웹 백엔드 Push

**구현 사항:**
- `src/llm/report_generator.py`: 일별/주간 보고서 생성 로직
- `src/scheduler/report_scheduler.py`: APScheduler Cron 트리거
- `src/models/report.py`: AnalysisReport / ReportMetrics / Insight Pydantic 모델
- `src/db/reports.py`: analysis_reports CRUD
- 웹 백엔드 Push: `POST {BACKEND_URL}/api/reports` (httpx, 재시도 3회)

**검증 체크리스트:**
- [ ] 일별 보고서 수동 트리거 → AnalysisReport JSON 생성
- [ ] 주간 보고서 → 7일간 LOT 집계 + 레시피별 비교 포함
- [ ] 데이터 없는 기간 → "해당 기간 분석 가능한 데이터 없음" 보고서
- [ ] 웹 백엔드 Push 실패 → 3회 재시도 후 pushed_to_backend=false 유지

---

### Task A5 — 통합 테스트 및 Graceful Shutdown

**목표:** 전체 파이프라인 E2E 테스트, SIGTERM 안전 종료

**구현 사항:**
- E2E 테스트: Dispatcher Mock → Ingest → 임베딩 → Query → 보고서 생성
- Graceful Shutdown: SIGTERM → 진행 중 임베딩 완료 대기 → DB 풀 종료
- 성능 테스트: 2,792건 레코드 DispatchBatch 수신 → 적재 완료 < 10초

**검증 체크리스트:**
- [ ] Dispatcher Mock 배치 5건 연속 전송 → 전체 적재 성공
- [ ] SIGTERM 수신 → 진행 중 작업 완료 후 종료 (데이터 손실 없음)
- [ ] 2,792건 배치 처리 시간 < 10초
- [ ] 동시 Ingest + Query 요청 → 데드락/크래시 없음

---

## 9. 비기능 요구사항

| 항목 | 요구사항 |
| :--- | :--- |
| 응답 시간 | Ingest: < 10초 (2,792건 배치, CPU 임베딩 포함), Query: < 15초 (LLM 포함) |
| 배포 환경 | AWS Lightsail (CPU-only, $20~40/월, 2 vCPU / 4~8GB RAM) |
| 가용성 | Docker Compose 기반 단일 인스턴스 (MVP) |
| 보안 | API Key 인증 (Ingest), JWT 인증 (Query/Report) |
| 로깅 | structlog JSON 포맷, 요청 ID 트레이싱 |
| 모니터링 | GET /health 엔드포인트 (DB + 임베딩 모델 로드 상태) |
| 데이터 보존 | Vector DB 임베딩 365일 보존 |
| 메모리 관리 | 임베딩 모델 상주 ~470MB + 앱 ~512MB (Lightsail 4GB 기준 여유 확보) |
| Graceful Shutdown | SIGTERM → 진행 중 작업 완료 → DB 풀 종료 |

---

## 10. Docker Compose 구성

```yaml
version: "3.9"
services:
  ai-server:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      pgvector:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 2g              # Lightsail 메모리 제한 (모델 ~470MB + 앱)
    volumes:
      - model_cache:/home/appuser/.cache/huggingface  # 모델 캐시 영속화 (재시작 시 재다운로드 방지)

  pgvector:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: ai_server
      POSTGRES_USER: ai_server
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - pgvector_data:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          memory: 512m            # Lightsail DB 메모리 제한
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ai_server"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgvector_data:
  model_cache:                        # HuggingFace 모델 캐시 영속화
```

### 10.1 AWS Lightsail 배포 가이드

```bash
# 1. Lightsail 컨테이너 서비스 또는 인스턴스 생성
#    권장: $40/월 (2 vCPU, 8GB RAM) 또는 최소 $20/월 (2 vCPU, 4GB RAM)

# 2. Docker 설치 (인스턴스 방식)
sudo apt update && sudo apt install -y docker.io docker-compose-plugin

# 3. torch CPU-only 빌드로 이미지 크기 절감 (Dockerfile 내)
#    pip install torch --index-url https://download.pytorch.org/whl/cpu

# 4. 첫 기동 시 모델 다운로드 (~200MB) → model_cache 볼륨에 캐시
docker compose up -d

# 5. 모델 사전 다운로드 (빌드 시간 단축)
#    Dockerfile에서 RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('intfloat/multilingual-e5-small')"
```
