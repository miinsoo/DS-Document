# 05. MES 서버 (C# — 중앙 제어 + 생산 이력)

## 1. 역할 한 줄
N대 가상 EAP를 중앙에서 일괄 제어(Lot 시작/종료, Recipe 변경, 긴급 정지)하고 Lot 이력과 레시피 사용 이력을 RDBMS에 저장하여 오퍼레이터 휴먼 에러를 사전 방지한다.

## 2. 책임
- ✅ EAP에 CONTROL_CMD 발행 (RECIPE_LOAD / LOT_ABORT / EMERGENCY_STOP / ALARM_CLEAR)
- ✅ Lot 시작/종료 이력 RDBMS 저장 (audit)
- ✅ Recipe 사용 이력 + 실패 이력 저장
- ✅ 실패 이력 있는 레시피 선택 시 오퍼레이터에게 경고
- ✅ Oracle ORACLE_ANALYSIS 구독 → 위험 등급 장비에 신규 작업 차단
- ✅ N대 장비 일괄 제어 UI (테스트 환경 QA 사이클 단축)
- ❌ 시계열 적재 (Historian 담당)
- ❌ 비전 검사 판정 (Oracle 담당)

## 3. 기술 스택
| 항목 | 선택 |
|---|---|
| 언어 | **C# .NET 8** |
| MQTT | MQTTnet v4.3+ |
| DB | PostgreSQL 16 또는 MSSQL 2022 |
| ORM | Entity Framework Core 8 (또는 Dapper) |
| UI | WinForms 또는 WPF (테스트 환경 제어용) |

## 4. MQTT 정책
| 항목 | 값 |
|---|---|
| client_id | `mes_server` |
| 인증 | `mes_server` 계정 |
| ACL | `ds/#` 전체 readwrite |
| 구독 | `ds/+/lot`, `ds/+/oracle`, `ds/+/recipe`, `ds/+/alarm` |
| 발행 | `ds/+/control` (모든 명령 가능) |

## 5. RDBMS 스키마

### 5.1 lot_history
```sql
CREATE TABLE lot_history (
  id              SERIAL PRIMARY KEY,
  lot_id          VARCHAR(64) NOT NULL UNIQUE,
  equipment_id    VARCHAR(32) NOT NULL,
  recipe_id       VARCHAR(64) NOT NULL,
  recipe_version  VARCHAR(16),
  operator_id     VARCHAR(32),
  started_at      TIMESTAMPTZ NOT NULL,
  ended_at        TIMESTAMPTZ,
  lot_status      VARCHAR(16),         -- COMPLETED/ABORTED/ERROR/RUNNING
  total_units     INTEGER,
  pass_count      INTEGER,
  fail_count      INTEGER,
  yield_pct       REAL,
  abort_reason    TEXT
);
CREATE INDEX ix_lh_eq ON lot_history (equipment_id, started_at DESC);
CREATE INDEX ix_lh_recipe ON lot_history (recipe_id, started_at DESC);
```

### 5.2 recipe_usage
```sql
CREATE TABLE recipe_usage (
  recipe_id          VARCHAR(64) PRIMARY KEY,
  total_lots         INTEGER DEFAULT 0,
  failed_lots        INTEGER DEFAULT 0,
  last_failed_at     TIMESTAMPTZ,
  warning_threshold  REAL DEFAULT 0.10,  -- 실패율 10% 이상 시 경고
  is_blocked         BOOLEAN DEFAULT FALSE
);
```

### 5.3 control_audit (감사 로그)
```sql
CREATE TABLE control_audit (
  id            SERIAL PRIMARY KEY,
  issued_at     TIMESTAMPTZ NOT NULL,
  equipment_id  VARCHAR(32) NOT NULL,
  command       VARCHAR(32) NOT NULL,
  issued_by     VARCHAR(64) NOT NULL,
  reason        TEXT,
  target_lot_id VARCHAR(64),
  result        VARCHAR(16)              -- ACK/NACK/TIMEOUT
);
```

## 6. 핵심 모듈

```
mes/
├── MES.Core/
│   ├── Mqtt/
│   │   ├── MesMqttClient.cs           ← 백오프, KeepAlive
│   │   ├── CommandPublisher.cs        ← CONTROL_CMD 발행
│   │   └── EventSubscriber.cs         ← lot/oracle/recipe/alarm 구독
│   ├── Services/
│   │   ├── LotHistoryService.cs       ← LOT_END 수신 → DB 저장
│   │   ├── RecipeWarningService.cs    ← 실패 레시피 경고
│   │   ├── EquipmentBlocker.cs        ← Oracle DANGER 시 작업 차단
│   │   └── BatchControlService.cs     ← N대 일괄 제어
│   ├── Repositories/
│   │   ├── LotRepository.cs
│   │   ├── RecipeRepository.cs
│   │   └── AuditRepository.cs
│   └── Models/
└── MES.UI/                             ← WinForms (선택)
    ├── DashboardForm.cs
    ├── ControlPanelForm.cs
    └── HistoryViewerForm.cs
```

## 7. 핵심 기능 동작

### 7.1 Lot 시작 워크플로우
```
1. 오퍼레이터가 UI에서 [Lot 시작] 클릭 (recipe_id 선택)
2. RecipeWarningService.CheckRecipe(recipe_id):
   - 실패율 ≥ 10% → 경고 모달 "이 레시피는 최근 12회 중 4회 실패. 진행하시겠습니까?"
   - is_blocked=true → 시작 차단
3. 통과 시 CommandPublisher.Publish(RECIPE_LOAD)
4. 통과 시 lot_history INSERT (status=RUNNING)
5. control_audit INSERT
```

### 7.2 Oracle 위험 감지 → 작업 차단
```
1. EventSubscriber가 ds/+/oracle 수신
2. judgment == "DANGER" 감지
3. EquipmentBlocker.Block(equipment_id, reason)
4. 해당 장비의 신규 RECIPE_LOAD 차단 (UI에 빨간 경고)
5. 오퍼레이터가 수동 해제 시까지 유지
```

### 7.3 N대 일괄 제어
```
- [전체 EMERGENCY_STOP] 버튼 → 4대 장비에 CONTROL_CMD 동시 발행
- [전체 STATUS_QUERY] → 즉시 STATUS_UPDATE 받아서 대시보드 갱신
- [Recipe 일괄 변경] → 같은 recipe로 N대 동시 RECIPE_LOAD
```

## 8. 데이터 플로우
```
오퍼레이터 ─→ MES UI ─→ CommandPublisher ─→ Broker ─→ EAP
                              ↓
                         control_audit (DB)

EAP ─→ Broker ─→ EventSubscriber ─→ LotHistoryService ─→ lot_history (DB)
                                           ↓
                                    RecipeWarningService ─→ recipe_usage 갱신

Oracle ─→ ORACLE_ANALYSIS ─→ EventSubscriber ─→ EquipmentBlocker
                                                      ↓
                                            UI 빨간 경고
```

## 9. 우선순위
**P1** — EAP/Broker 다음. Oracle보다 먼저 또는 동시 개발 가능.

## 10. 검증 시나리오
1. 4대 EAP에 RECIPE_LOAD 일괄 발행 → 4대 모두 RECIPE_CHANGED 회신
2. 실패율 12%인 Carsem_4X6 선택 → 경고 모달 표시
3. Oracle DANGER 알림 → 해당 장비 RECIPE_LOAD 시도 → 차단됨
4. LOT_END(ABORTED) → lot_history에 abort_reason 저장
5. control_audit 테이블에 모든 명령 발행 이력 적재
