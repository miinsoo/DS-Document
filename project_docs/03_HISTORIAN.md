# 03. Historian 서버 (Node.js + TimescaleDB)

## 1. 역할 한 줄
모든 EAP 이벤트를 시계열 DB에 영속 적재하여 Oracle 2차 검증과 Dispatcher의 read-only 데이터 공급원이 된다.

## 2. 책임
- ✅ `ds/#` 전체 토픽 read-only 구독
- ✅ 8개 이벤트 JSON → TSDB 적재
- ✅ ABORTED LOT의 partial 데이터를 별도 태그(`is_partial=true`)로 저장
- ✅ Retention 정책 (90일 압축, 1년 삭제)
- ❌ 비즈니스 판정 (Oracle 담당)
- ❌ 외부 노출 (Dispatcher 담당)
- ❌ 데이터 변환·집계 (raw 그대로 저장)

## 3. 기술 스택
| 항목 | 선택 |
|---|---|
| 런타임 | Node.js 20 LTS + TypeScript 5+ |
| MQTT 라이브러리 | mqtt.js v5+ |
| DB | **PostgreSQL 16 + TimescaleDB extension** (1순위) |
| ORM | 사용 안 함 (raw SQL, 배치 INSERT 성능 우선) |
| Connection Pool | `pg` Pool (max 20) |

## 4. MQTT 구독 정책
| 항목 | 값 |
|---|---|
| client_id | `historian_server` (고정) |
| clean_start | **false** |
| session_expiry | **86400s (24시간)** |
| keep_alive | 60s |
| 구독 토픽 | `ds/#` (와일드카드) |

## 5. DB 스키마 (8개 하이퍼테이블)

모든 테이블은 `time` 컬럼 기준 TimescaleDB 하이퍼테이블.

### 5.1 inspection_results (가장 큰 테이블)
```sql
CREATE TABLE inspection_results (
  time              TIMESTAMPTZ NOT NULL,
  equipment_id      TEXT NOT NULL,
  lot_id            TEXT NOT NULL,
  unit_id           TEXT NOT NULL,
  recipe_id         TEXT NOT NULL,
  overall_result    TEXT NOT NULL,    -- PASS/FAIL
  fail_reason_code  TEXT,
  fail_count        SMALLINT,
  inspection_detail JSONB,            -- prs_result + side_result
  geometric         JSONB,
  bga               JSONB,
  surface           JSONB,
  singulation       JSONB,
  process           JSONB
);
SELECT create_hypertable('inspection_results', 'time', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX ix_ir_lot ON inspection_results (lot_id, unit_id);
CREATE INDEX ix_ir_eq_recipe ON inspection_results (equipment_id, recipe_id, time DESC);
```

### 5.2 status_updates / lot_ends / hw_alarms / heartbeats / recipe_changes / control_cmds / oracle_analyses
구조 동일 패턴 (time, equipment_id + 이벤트별 필드). v3.4 신규 필드 모두 포함:
- status_updates: `current_unit_count`, `expected_total_units`, `current_yield_pct`
- hw_alarms: `burst_id`, `burst_count`

## 6. 핵심 모듈

```
historian/
├── src/
│   ├── index.ts                 ← 진입점
│   ├── mqtt/
│   │   ├── subscriber.ts        ← mqtt.js + 백오프
│   │   └── reconnect.ts
│   ├── db/
│   │   ├── pool.ts              ← pg Pool
│   │   ├── batch_writer.ts      ← ★ 1초/1000건 배치 INSERT
│   │   └── schema.sql
│   ├── handlers/                ← 8개 이벤트별 핸들러
│   ├── retention/
│   │   └── policy.ts            ← 90일 압축 cron
│   └── observability/
│       └── logger.ts            ← pino
└── package.json
```

## 7. 배치 Writer (성능 핵심)

INSPECTION_RESULT는 takt 1.62초 × N대 → 단건 INSERT 시 DB 라운드트립 병목.

```typescript
class BatchWriter {
  private buffer: InspectionRow[] = [];
  private timer: NodeJS.Timeout | null = null;

  enqueue(row: InspectionRow) {
    this.buffer.push(row);
    if (this.buffer.length >= 1000) this.flush();
    else if (!this.timer) this.timer = setTimeout(() => this.flush(), 1000);
  }

  async flush() {
    if (this.buffer.length === 0) return;
    const batch = this.buffer.splice(0);
    this.timer = null;
    await this.pool.query(buildBulkInsertSql(batch));
  }
}
```

**규칙**: 1초 OR 1000건 중 먼저 도달하는 시점에 flush. 다른 이벤트는 단건 INSERT.

## 8. 데이터 플로우
```
EAP(N대) ─┐
MES ──────┼──→ Broker ──→ Historian Subscriber → EventRouter
Oracle ───┘                                          ↓
                                              ┌──────┴──────┐
                                              ↓             ↓
                                       BatchWriter   SingleWriter
                                       (results)     (나머지 7종)
                                              ↓             ↓
                                              └→ TimescaleDB ←┘
                                                     ↓
                                          Oracle 2차 (read-only)
                                          Dispatcher (read-only)
```

## 9. 실패 처리
| 상황 | 대응 |
|---|---|
| DB 연결 끊김 | pg Pool 자동 재연결 + 메모리 버퍼 (max 10,000건) |
| 버퍼 오버플로우 | WAL 파일 fallback |
| MQTT 단절 | mqtt.js 백오프 + clean_start=false |
| JSON 파싱 실패 | dead_letter 테이블 + 로그 |

## 10. Retention
```sql
SELECT add_compression_policy('inspection_results', INTERVAL '90 days');
SELECT add_retention_policy('inspection_results', INTERVAL '365 days');
```

## 11. 우선순위
**P1** — EAP/Broker 다음. Oracle 2차 검증의 필수 의존성.

## 12. 검증 시나리오
1. 4대 EAP 동시 발행 → 8개 테이블 모두 적재
2. takt 1.62s × 1시간 부하 → batch flush 지연 < 2초
3. ABORTED LOT → `is_partial=true` 태그 확인
4. burst_id 41건 알람 → burst_id로 그룹 쿼리 가능
