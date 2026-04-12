# 04. Historian (Node.js + TimescaleDB)

## 1. 역할
모든 EAP 이벤트를 시계열 DB에 영속 적재. Oracle/AI 서버의 분석 원천 데이터 공급.

## 2. 책임
- ✅ `ds/#` 전체 구독 (read-only)
- ✅ TSDB 적재 (PostgreSQL TimescaleDB 권장)
- ✅ ABORTED LOT의 partial 데이터를 별도 태그로 저장
- ❌ 비즈니스 판정 (Oracle 담당)
- ❌ 외부 노출 (Dispatcher 담당)

## 3. 기술 스택
- Node.js + TypeScript
- mqtt.js v5+
- pg (PostgreSQL) + TimescaleDB extension
- 대안: InfluxDB

## 4. DB 스키마 (8개 하이퍼테이블)

| 테이블 | 주요 컬럼 | 인덱스 |
|---|---|---|
| heartbeats | timestamp, equipment_id | (eq, time) |
| status_updates | timestamp, eq, status, lot_id, recipe_id, current_unit_count, current_yield_pct | (eq, time) |
| inspection_results | timestamp, eq, lot_id, unit_id, overall_result, fail_reason, prs_jsonb, side_jsonb | (eq, lot, time) |
| lot_ends | timestamp, eq, lot_id, lot_status, total_units, yield_pct | (eq, recipe, time) |
| hw_alarms | timestamp, eq, level, error_code, burst_id | (eq, time, level) |
| recipe_changes | timestamp, eq, prev, new, changed_by | (eq, time) |
| control_cmds | timestamp, eq, command, issued_by | (eq, time) |
| oracle_analyses | timestamp, eq, lot_id, judgment, recipe_id | (recipe, time) |

## 5. 핵심 모듈
- **Subscriber**: `ds/#` wildcard, QoS 2, clean_start=false, persistent client_id
- **Writer**: 배치 INSERT (1초 또는 1000건 단위)
- **RetentionPolicy**: 90일 후 압축, 1년 후 삭제

## 6. 우선순위
P1 — EAP/Mobile/Broker 다음.

---

# 05. MES (C#)

## 1. 역할
N대 가상 EAP를 중앙에서 일괄 제어. Lot 시작/종료, Recipe 변경, EMERGENCY_STOP 발행.

## 2. 책임
- ✅ MES → EAP CONTROL_CMD 발행
- ✅ Lot 이력 RDBMS 저장 (audit)
- ✅ 실패 레시피 경고 (오퍼레이터 휴먼 에러 방지)
- ✅ Oracle 알림 수신 시 작업 차단 검토
- ❌ 시계열 데이터 적재 (Historian 담당)

## 3. 기술 스택
- C# .NET 8
- MQTTnet v4.3+
- PostgreSQL/MSSQL

## 4. 핵심 모듈
- **CommandIssuer**: RECIPE_LOAD/LOT_ABORT/EMERGENCY_STOP 발행
- **LotHistoryRepository**: Lot 시작/종료 RDBMS 저장
- **RecipeWarningService**: 실패 이력 레시피 선택 시 경고
- **OracleAlertReceiver**: ds/+/oracle 구독 → 위험 장비 작업 차단

---

# 06. Oracle (Python, 후순위)

## 1. 역할
1차 Rule-based 검증 + 2차 AI 검증 (EWMA+MAD / Isolation Forest). 레시피별 독립 학습.

## 2. 책임
- ✅ 1차: 38개 Rule for INSPECTION_RESULT (실시간 if-else)
- ✅ 2차: LOT_END 후 Historian TSDB 조회 → 동적 임계값 → ORACLE_ANALYSIS 발행
- ✅ 오퍼레이터 승인 구조 (Rule DB 자동 업데이트 금지, 제안만)
- ✅ 레시피별 독립 EWMA (다른 레시피 데이터 참조 금지)

## 3. 기술 스택
- Python 3.11+
- paho-mqtt
- scikit-learn (Isolation Forest)
- PostgreSQL pgvector / Chroma DB
- pandas (Historian 조회)

## 4. 데이터 플로우
```
[1차 실시간]
INSPECTION_RESULT → Rule DB 조회 → PASS/WARNING/CRITICAL → 모바일 (<100ms)

[2차 비동기]
LOT_END → Historian TSDB 조회 → EWMA+MAD/IF → ORACLE_ANALYSIS 발행
       → 모바일/MES → 오퍼레이터 승인 → Rule DB 갱신
```

## 5. 판정 3단계
- NORMAL: μ±2σ 이내 + IF<0.5
- WARNING: 2σ~3σ 또는 IF 0.5~0.85
- DANGER: 3σ 초과 또는 IF>0.85

## 6. 우선순위
P2 — 후순위. EAP/Mobile/Historian 안정화 후 착수.

---

# 07. Dispatcher (Node.js)

## 1. 역할
Local Area TSDB에서 read-only로 데이터를 조회 → 비식별화 → Online AI 서버로 단방향 push.

## 2. 책임
- ✅ TSDB read-only 권한만
- ✅ 비식별화 (operator_id, lot_id 마스킹)
- ✅ 단방향 push (AI 서버는 응답 없음)
- ✅ 1시간 단위 배치 스케줄
- ❌ 양방향 통신 (보안 침투 경로 차단)

## 3. 기술 스택
- Node.js + TypeScript
- pg (read-only 사용자)
- axios (HTTPS push)

## 4. 보안
- DMZ 구성, 단방향 firewall
- TLS mutual auth
- 비식별화 로그 별도 audit

## 5. 우선순위
P2 — Online Area 협업 시점에 착수.
