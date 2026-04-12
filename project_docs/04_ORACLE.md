# 04. Oracle 서버 (Python — Rule-based + AI 검증)

## 1. 역할 한 줄
EAP가 발행한 검사 결과를 **1차 Rule-based로 실시간 판정**하고(필수), LOT 완료 후 **2차 AI로 동적 임계값을 학습**(후순위)하여 ORACLE_ANALYSIS 이벤트를 발행한다.

## 2. ⚠️ 핵심 — 1차와 2차의 분리

| 구분 | 1차 검증 | 2차 검증 |
|---|---|---|
| **방식** | Rule-based if-else (R01~R37) | EWMA+MAD / Isolation Forest |
| **트리거** | INSPECTION_RESULT 수신 즉시 | LOT_END 수신 후 비동기 |
| **응답 시간** | < 100ms (실시간) | ~수초 (LOT 완료 후) |
| **데이터 소스** | MQTT 직접 수신 | Historian TSDB read-only |
| **의존성** | Broker만 있으면 동작 | Historian 필수 |
| **우선순위** | **P1 (필수)** | P2 (후순위) |
| **목적** | Pass/Warning/Critical 즉시 판정 | 회색지대(MARGINAL) 검출 + 임계값 갱신 |

> **중요**: 1차 Rule-based 검증은 데이터 플로우의 **반드시 동작해야 하는 핵심 컴포넌트**다. EAP가 단순히 PASS/FAIL을 발행하는 것을 넘어, 38개 Rule로 WARNING/CRITICAL 등급을 매겨 모바일에 즉시 전달한다. 이게 없으면 모바일은 PASS/FAIL 이분법밖에 못 본다.

## 3. 책임

### 3.1 1차 Rule-based (P1, 필수)
- ✅ INSPECTION_RESULT 실시간 구독
- ✅ Rule R01~R37 if-else 평가 (외부 Rule DB 조회)
- ✅ 판정 결과를 ORACLE_ANALYSIS 토픽으로 즉시 발행 (또는 알람 토픽 trigger)
- ✅ Rule DB 임계값 변경 즉시 적용 (다음 INSPECTION_RESULT부터)
- ✅ HW_ALARM 이벤트 트리거 조건 검출 (R26~R34)

### 3.2 2차 AI 검증 (P2, 후순위)
- ✅ LOT_END 트리거 → Historian TSDB 조회
- ✅ 레시피별 독립 EWMA+MAD 동적 임계값 계산
- ✅ Isolation Forest 복합 이상 감지 (LOT 10개 이상 시)
- ✅ ORACLE_ANALYSIS 발행 (NORMAL/WARNING/DANGER)
- ✅ threshold_proposal 생성 (오퍼레이터 승인 대기)
- ❌ Rule DB 자동 갱신 금지 (오퍼레이터 승인 후만)

## 4. 기술 스택
| 항목 | 선택 |
|---|---|
| 언어 | **Python 3.11+** |
| MQTT | paho-mqtt 2.x |
| AI | scikit-learn (Isolation Forest), numpy, pandas |
| DB | PostgreSQL pgvector (Vector DB), Chroma DB (대안) |
| Rule DB | PostgreSQL (RDBMS, RDB 테이블 형태) |
| Historian 조회 | psycopg2 read-only 사용자 |

## 5. 1차 검증 — Rule DB 구조

Rule을 코드에 하드코딩하지 않고 **DB 테이블에서 읽어** 동적 적용. 오퍼레이터 승인 시 DB만 갱신하면 다음 INSPECTION_RESULT부터 자동 적용.

### 5.1 rules 테이블
```sql
CREATE TABLE rules (
  rule_id            TEXT PRIMARY KEY,        -- R01, R02, ..., R37
  recipe_id          TEXT NOT NULL DEFAULT 'DEFAULT',
  metric             TEXT NOT NULL,           -- 'prs_x_offset', 'yield_pct', ...
  normal_max         REAL,
  warning_max        REAL,
  critical_max       REAL,
  comparison_op      TEXT NOT NULL,           -- '>', '<', 'abs>', ...
  updated_at         TIMESTAMPTZ DEFAULT NOW(),
  approved_by        TEXT,
  lot_basis          INTEGER                  -- 몇 LOT 학습값인지
);
```

### 5.2 rule_history 테이블 (audit)
```sql
CREATE TABLE rule_history (
  changed_at   TIMESTAMPTZ NOT NULL,
  rule_id      TEXT NOT NULL,
  recipe_id    TEXT NOT NULL,
  metric       TEXT NOT NULL,
  old_warning  REAL,
  new_warning  REAL,
  old_critical REAL,
  new_critical REAL,
  approved_by  TEXT,
  ai_basis     TEXT                           -- '8 LOT EWMA μ=68.2%, σ=4.1%'
);
```

## 6. 핵심 모듈 구조

```
oracle/
├── src/
│   ├── main.py
│   ├── primary/                          ← 1차 Rule-based (P1)
│   │   ├── subscriber.py                 ← INSPECTION_RESULT 구독
│   │   ├── rule_engine.py                ← R01~R37 if-else
│   │   ├── rule_loader.py                ← Rule DB 캐시 (10초마다 refresh)
│   │   └── publisher.py                  ← ORACLE_ANALYSIS 발행
│   ├── secondary/                        ← 2차 AI 검증 (P2)
│   │   ├── lot_end_listener.py
│   │   ├── historian_query.py            ← TSDB read-only
│   │   ├── ewma_mad.py                   ← 동적 임계값 계산
│   │   ├── isolation_forest.py
│   │   └── threshold_proposer.py
│   ├── shared/
│   │   ├── mqtt_client.py
│   │   ├── rule_db.py                    ← rules / rule_history CRUD
│   │   └── models.py
│   └── operator/
│       └── approval_handler.py           ← 오퍼레이터 승인 처리
└── requirements.txt
```

## 7. 1차 Rule Engine 핵심 로직

```python
# rule_engine.py (개념)
class RuleEngine:
    def __init__(self, rule_loader: RuleLoader):
        self.rules = rule_loader  # 10초마다 refresh

    def evaluate(self, inspection: InspectionResult) -> Judgment:
        violations = []
        recipe = inspection.recipe_id

        # R02: PRS XOffset
        for slot in inspection.prs_result:
            rule = self.rules.get('R02', recipe)
            if abs(slot.x_offset) > rule.critical_max:
                violations.append(('R02', 'CRITICAL', slot))
            elif abs(slot.x_offset) > rule.warning_max:
                violations.append(('R02', 'WARNING', slot))

        # R09: SIDE ET=52 비율
        et52_count = sum(1 for s in inspection.side_result if s.error_type == 52)
        et52_ratio = et52_count / len(inspection.side_result)
        rule = self.rules.get('R09', recipe)
        if et52_ratio > rule.critical_max:
            violations.append(('R09', 'CRITICAL', et52_ratio))
        # ... R01~R37 전체 평가

        return Judgment(
            level=max([v[1] for v in violations] or ['NORMAL']),
            violations=violations
        )
```

## 8. 2차 검증 핵심 로직

```python
# ewma_mad.py (개념)
def calculate_dynamic_threshold(recipe_id: str, metric: str) -> Threshold:
    # 동일 레시피의 최근 LOT 이력만 조회 (다른 레시피 참조 금지)
    lots = historian.query_lots_by_recipe(recipe_id, limit=30)

    if len(lots) < 5:
        return Threshold.from_seeding(recipe_id, metric)  # 오퍼레이터 시딩값

    values = [lot.yield_pct for lot in lots]
    ewma = calculate_ewma(values, alpha=0.3)
    mad = calculate_mad(values)

    return Threshold(
        warning_min=ewma - 2 * mad,
        warning_max=ewma + 2 * mad,
        danger_threshold=ewma - 3 * mad,
        lot_basis=len(lots)
    )
```

## 9. 판정 3단계 (2차 검증)
| 등급 | EWMA 조건 | Isolation Forest | 액션 |
|---|---|---|---|
| NORMAL | μ±2σ 이내 | 점수 < 0.5 | 보고서 생성, 경보 없음 |
| WARNING | 2σ~3σ 초과 | 0.5 ~ 0.85 | 모바일 주의 알림 + 오퍼레이터 확인 |
| DANGER | 3σ 초과 | > 0.85 | 즉시 경보 + 작업 중단 권고 |

## 10. 데이터 플로우

### 10.1 1차 검증 (실시간)
```
EAP → INSPECTION_RESULT → Broker → Oracle Subscriber
                                       ↓
                                  RuleEngine (R01~R37)
                                       ↓
                              Rule DB (10초 캐시)
                                       ↓
                            ORACLE_ANALYSIS 발행
                                       ↓
                              Broker → Mobile (<100ms)
```

### 10.2 2차 검증 (비동기)
```
LOT_END → Broker → Oracle LotEndListener
                        ↓
            Historian TSDB read-only 조회
                        ↓
            EWMA+MAD + Isolation Forest
                        ↓
            ORACLE_ANALYSIS 발행 (threshold_proposal 포함)
                        ↓
            Mobile / MES 수신 → 오퍼레이터 승인
                        ↓
            Rule DB 갱신 → 다음 1차 검증부터 새 임계값 적용
```

## 11. 오퍼레이터 승인 구조 (필수)

AI는 **자동으로 임계값을 갱신하지 않는다.** 모든 변경은 오퍼레이터 승인 필요.

```
AI 기준 조정 제안 (threshold_proposal)
        ↓
모바일 알림 (제안 근거 + lot_basis 표시)
        ↓
오퍼레이터 수락 → ApprovalHandler → Rule DB UPDATE → rule_history INSERT
오퍼레이터 거부 → 기존 기준 유지
```

이유:
1. 불량 LOT 연속 시 불량 상태를 "정상"으로 학습하는 것 방지
2. 현장 엔지니어 도메인 지식이 AI 판단 보정 가능

## 12. 실패 처리
| 상황 | 대응 |
|---|---|
| Historian 다운 (2차) | 2차 검증만 일시 중단, 1차는 정상 동작 |
| Rule DB 다운 | 마지막 캐시값으로 평가 + 알람 |
| MQTT 단절 | 백오프 재연결, INSPECTION_RESULT는 QoS 1이라 일부 손실 가능 (수용) |
| Isolation Forest 학습 실패 | EWMA만으로 fallback |

## 13. 우선순위
- **1차 Rule-based: P1 (필수)** — EAP/Broker/Mobile 다음 4번째로 개발
- **2차 AI 검증: P2 (후순위)** — Historian 안정화 + 5 LOT 이상 데이터 축적 후

## 14. 검증 시나리오
1. INSPECTION_RESULT(ET=11 PRS XOffset=350) → R02 CRITICAL 판정 → ORACLE_ANALYSIS 100ms 이내 발행
2. Rule DB의 R02 critical_max를 300→250으로 변경 → 10초 후 다음 INSPECTION_RESULT부터 새 기준 적용
3. LOT_END(yield 68.5%, recipe Carsem_4X6) → 8개 LOT 학습 결과로 WARNING 판정 + threshold_proposal 발행
4. 오퍼레이터가 모바일에서 threshold_proposal 수락 → Rule DB 갱신 + history 기록

## 15. 참조
- 정식 명세: `명세서/DS_EAP_MQTT_API_명세서.md` v3.4 §11 (Rule 38개)
- 2차 기획안: `문서/오라클 2차 검증 기획안.md`
- ORACLE_ANALYSIS 페이로드: `99_EVENT_CATALOG.md` §3
