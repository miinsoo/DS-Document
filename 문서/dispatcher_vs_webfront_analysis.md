# Dispatcher 전송 데이터 vs 웹 프론트 필요 데이터 분석

> **작성일**: 2026-05-03
> **분석 기준**: capstone zip 소스코드 + 피그마 웹 프론트 디자인
> **목적**: Dispatcher가 AI 서버로 전송하는 데이터로 웹 프론트 화면을 정상 구현할 수 있는지 확인

---

## 1. 현재 Dispatcher 전송 구조

### 1-1. 조회 테이블 (현재 2개만)

| 테이블 | 용도 |
|:---|:---|
| lot_ends | LOT 요약 (수율, 생산량, 상태) |
| inspection_results | 개별 검사 결과 (치수, 불량 유형) |

### 1-2. DispatchBatch 구조

```typescript
interface DispatchBatch {
  batchId: string;          // UUID v4
  dispatchedAt: string;     // ISO 8601 UTC
  lotHash: string;          // lot_id HMAC 해시
  equipmentHash: string;    // equipment_id HMAC 해시
  totalRecords: number;
  records: AnonymizedInspectionRecord[];  // inspection_results 비식별화
  lotSummary: AnonymizedLotRecord;        // lot_ends 비식별화
}
```

### 1-3. 비식별화 규칙

| 원본 필드 | 처리 방식 |
|:---|:---|
| equipment_id | HMAC-SHA256 해시 |
| lot_id | HMAC-SHA256 해시 |
| operator_id | 필드 삭제 |
| strip_id | 순번 대체 (1, 2, 3...) |
| unit_id | 순번 대체 (1, 2, 3...) |
| recipe_id | 원본 유지 (비식별화 대상 아님) |

---

## 2. 웹 프론트 화면별 데이터 매핑

### 2-1. 공장 현황 대시보드 (KPI Scorecard)

| KPI | 필요 데이터 | Dispatcher 전송 | 상태 |
|:---|:---|:---|:---|
| 총 생산량 (24,563 units) | lot_ends.total_units 집계 | O (lotSummary.total_units) | 정상 |
| 종합 수율 (96.4%) | lot_ends.yield_pct 평균 | O (lotSummary.yield_pct) | 정상 |
| Fail 수량 (342건) | lot_ends.fail_count 합산 | O (lotSummary.fail_count) | 정상 |
| Marginal 수량 (128건) | marginal_count | X | **누락 - 스키마에 없음** |
| UPH (289) | total_units / lot_duration_sec | O (간접 계산 가능) | 정상 |
| 재작업률 (0.5%) | rework 추적 필드 | X | **누락 - 스키마에 없음** |
| 라인 장비 가동률 (87.3%) | status_updates RUN/STOP 비율 | X | **Dispatcher 미조회** |
| 총 다운타임 (257분) | status_updates STOP 시간 합산 | X | **Dispatcher 미조회** |
| 가동 장비 수 (4/5대) | status_updates 최신 상태 | X | **Dispatcher 미조회** |
| MTBF (12.5h) | hw_alarms 간 interval 계산 | X | **Dispatcher 미조회** |
| 평균 Cpk (1.52) | inspection_detail JSONB 치수 | O (records[].inspection_detail) | 정상 |
| 치수 합격률 (98.7%) | inspection_detail 기반 계산 | O (간접 계산 가능) | 정상 |
| 관리 이탈 (8건) | inspection_detail Z-score | O (간접 계산 가능) | 정상 |
| 전일 대비 변화율 | 이전 날짜 데이터 비교 | 별도 쿼리 필요 | 웹 백엔드 처리 |

### 2-2. 장비 현황 테이블

| 컬럼 | 필요 데이터 | Dispatcher 전송 | 상태 |
|:---|:---|:---|:---|
| 장비 ID (VELOCE-G7.01) | equipment_id 원본 | X (HMAC 해시만 전송) | **비식별화로 복원 불가** |
| 현재 레시피 (PKG_DICE_A12) | recipe_id | O (lotSummary.recipe_id) | 정상 |
| 상태 (정상/경고/정지) | equipment_status | X | **Dispatcher 미조회** |
| 수율 (48.0%) | yield_pct | O (lotSummary.yield_pct) | 정상 |
| Cpk (0.85) | inspection_detail 치수 | O (간접 계산 가능) | 정상 |
| UPH (245) | total_units / duration | O (간접 계산 가능) | 정상 |
| 조치 알람 (긴급/주의/정보) | hw_alarms + oracle judgment | 부분 | **hw_alarms 미전송** |

### 2-3. AI 분석 및 권고사항

| 항목 | 필요 데이터 | Dispatcher 전송 | 상태 |
|:---|:---|:---|:---|
| 치수 데이터 기반 원인 추론 | oracle_analyses.ai_comment | X | **Dispatcher 미조회** |
| 예상 수율 영향 | oracle_analyses.yield_status | X | **Dispatcher 미조회** |
| 추천 액션 우선순위 | oracle_analyses.violated_rules | X | **Dispatcher 미조회** |
| judgment (NORMAL/WARNING/DANGER) | oracle_analyses.judgment | X | **Dispatcher 미조회** |

### 2-4. 일일 리포트 - 장비별 상세 분석

| 항목 | 필요 데이터 | Dispatcher 전송 | 상태 |
|:---|:---|:---|:---|
| 수율 추이 차트 (LOT별) | lot_ends 시계열 | O (lotSummary) | 정상 |
| 불량 위치 히트맵 | inspection_detail 내부 좌표 | O (records[].inspection_detail) | 정상 |
| SPC 관리도 (Width) | inspection_detail 치수 | O (records[].inspection_detail) | 정상 |
| 파라미터 통계 (평균/표준편차/Z-score) | inspection_detail 치수 + USL/LSL | O (간접 계산 가능) | 정상 |
| 불량 유형별 (Chipping/Burr/Crack) | fail_reason_code | O (records[].fail_reason_code) | 정상 |
| AI 패턴 분류 및 추정 원인 | oracle_analyses 전체 | X | **Dispatcher 미조회** |

### 2-5. 일일 리포트 - 가동/알람 섹션

| 항목 | 필요 데이터 | Dispatcher 전송 | 상태 |
|:---|:---|:---|:---|
| 24시간 가동 타임라인 | status_updates 시계열 | X | **Dispatcher 미조회** |
| 장비 정지 사유 (렌즈 오염 40% 등) | hw_alarms.hw_error_code 분류 | X | **Dispatcher 미조회** |
| 경보 요약 및 조치 현황 | hw_alarms + 조치 이력 | X | **Dispatcher 미조회** |
| 조치 결과 (렌즈 세정 by 김민수) | 작업자 조치 이력 | X | **스키마 자체 없음** |
| 미조치 경보 | hw_alarms 중 미처리 건 | X | **Dispatcher 미조회** |

---

## 3. 누락 항목 분류

### 3-1. Dispatcher가 조회하지 않는 테이블 (3개) - 추가 필요

현재 Dispatcher는 lot_ends + inspection_results만 조회합니다.
웹 프론트를 위해 아래 3개 테이블을 추가 조회해야 합니다.

| 테이블 | 웹 프론트 사용 영역 | 주요 필드 |
|:---|:---|:---|
| oracle_analyses | AI 분석 코멘트, 판정 등급, 권고사항 | judgment, ai_comment, yield_status, violated_rules |
| status_updates | 장비 가동률, 다운타임, 24시간 타임라인 | equipment_status, uptime_sec, current_unit_count |
| hw_alarms | 알람 이력, 정지 사유, 경보 요약 | hw_error_code, alarm_level, burst_id, auto_recovery_attempted |

### 3-2. 스키마 자체에 없는 데이터 (3개) - 설계 결정 필요

| 데이터 | 피그마 위치 | 현재 상태 | 해결 방안 |
|:---|:---|:---|:---|
| Marginal 수량 (128건) | 대시보드 KPI | lot_ends에 marginal_count 컬럼 없음 | Oracle 판정의 marginal_units 카운트를 lot_ends에 추가하거나, oracle_analyses에서 집계 |
| 재작업률 (0.5%) | 대시보드 KPI | 재작업 추적 필드/테이블 없음 | MES 영역 데이터 → 웹 백엔드에서 별도 관리 또는 피그마에서 제거 |
| 조치 이력 (렌즈 세정 by 김민수 15:20) | 리포트 Page 2, 장비 상세 | 작업자 조치 기록 테이블 없음 | 웹 프론트에서 직접 입력 -> 웹 백엔드 DB 저장 (로컬 서버 범위 밖) |

### 3-3. 비식별화로 인해 복원 불가 (1개) - 정책 결정 필요

| 데이터 | 피그마 위치 | 현재 상태 | 해결 방안 |
|:---|:---|:---|:---|
| 장비 ID 원본 (VELOCE-G7.01) | 장비 테이블, 리포트 전체 | HMAC 해시로 비식별화됨 | 방안 A: AI 서버에 해시->장비명 매핑 테이블 보유 / 방안 B: equipment_id 비식별화 제외 (보안 정책 재검토) |

---

## 4. 전송 가능 vs 전송 불가 요약

### 4-1. 현재 Dispatcher로 커버 가능한 화면 요소

- 총 생산량 / 종합 수율 / Fail 수량 / UPH
- 레시피 정보
- 수율 추이 차트 (LOT별)
- 불량 위치 히트맵
- SPC 관리도 (Width 등 치수)
- 파라미터 상세 통계 (평균, 표준편차, Z-score, USL/LSL, 판정)
- 불량 유형별 카운트 (Chipping / Burr / Crack / Others)
- Cpk 계산

### 4-2. Dispatcher 수정으로 커버 가능한 화면 요소

oracle_analyses / status_updates / hw_alarms 테이블을 추가 조회하면:

- AI 분석 및 권고사항 전체
- 장비 상태 (정상/경고/정지)
- 장비 가동률 / 다운타임 / MTBF
- 24시간 가동 타임라인
- 장비 정지 사유
- 경보 요약 (긴급/주의/정보)

### 4-3. 로컬 서버 범위 밖 (웹 백엔드에서 처리)

- 전일 대비 변화율 (이전 데이터 비교는 웹 백엔드 DB에서)
- 조치 이력 (웹 프론트 입력 -> 웹 백엔드 저장)
- 재작업률 (MES 영역)
- 담당/검토 체크박스 (웹 프론트 UI 기능)
- 리포트 설정/다운로드/인쇄/공유 (웹 프론트 기능)

---

## 5. Dispatcher 수정 권장안

### 5-1. DispatchBatch 타입에 3개 필드 추가

```typescript
export interface DispatchBatch {
  // 기존 필드
  batchId: string;
  dispatchedAt: string;
  lotHash: string;
  equipmentHash: string;
  totalRecords: number;
  records: AnonymizedInspectionRecord[];
  lotSummary: AnonymizedLotRecord;

  // 추가 필드
  oracleAnalysis?: {
    judgment: string;           // NORMAL / WARNING / DANGER
    ai_comment: string;
    yield_status: object;
    violated_rules: object[];
    isolation_forest_score: number | null;
  };
  statusHistory?: {
    time: string;
    equipment_status: string;   // RUN / IDLE / STOP
    uptime_sec: number;
    current_unit_count: number | null;
    current_yield_pct: number | null;
  }[];
  alarmHistory?: {
    time: string;
    alarm_level: string;        // CRITICAL / WARNING / INFO
    hw_error_code: string;
    hw_error_detail: string;
    auto_recovery_attempted: boolean;
    burst_id: string | null;
  }[];
}
```

### 5-2. queries.ts에 3개 쿼리 함수 추가

```typescript
// 1. Oracle 판정 결과 조회
export async function fetchOracleAnalysis(
  lotId: string
): Promise<OracleAnalysisRecord | null>

// 2. 해당 LOT 시간대의 status_updates 조회
export async function fetchStatusHistory(
  equipmentId: string,
  startTime: string,
  endTime: string
): Promise<StatusRecord[]>

// 3. 해당 LOT 시간대의 hw_alarms 조회
export async function fetchAlarmHistory(
  equipmentId: string,
  startTime: string,
  endTime: string
): Promise<AlarmRecord[]>
```

### 5-3. 비식별화 규칙 적용

추가 데이터에도 기존 비식별화 규칙을 동일하게 적용:
- oracle_analysis의 lot_id, equipment_id -> HMAC 해시
- status/alarm의 equipment_id -> HMAC 해시
- operator_id -> 삭제
- hw_error_detail 중 개인정보 포함 가능성 -> 검토 필요

---

## 6. 우선순위

| 우선순위 | 항목 | 영향 |
|:---|:---|:---|
| P0 | oracle_analyses 추가 조회 | AI 분석 섹션 전체 불가 |
| P0 | status_updates 추가 조회 | 가동률/다운타임/타임라인 전체 불가 |
| P1 | hw_alarms 추가 조회 | 알람/정지사유/경보 섹션 불가 |
| P1 | equipment_id 비식별화 정책 결정 | 장비명 표시 불가 |
| P2 | Marginal 수량 스키마 추가 | 대시보드 KPI 1개 누락 |
| P2 | 재작업률 데이터 소스 결정 | 대시보드 KPI 1개 누락 |
| P3 | 조치 이력 테이블 설계 | 리포트 조치 섹션 빈칸 |
