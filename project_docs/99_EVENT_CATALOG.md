# 99. 이벤트 카탈로그 (8개 이벤트)

> **모든 서버가 공통으로 참조하는 이벤트 사전.** 정식 페이로드는 `명세서/DS_EAP_MQTT_API_명세서.md` v3.4 참조.

## 1. 토픽 / 이벤트 / QoS / Retained 매트릭스

| # | 토픽 | 이벤트 | QoS | Retained | 방향 | 주기 | Mock |
|---|---|---|---|---|---|---|---|
| 1 | `ds/{eq}/heartbeat` | HEARTBEAT | 1 | ❌ | EAP→Broker | 3초 | 01 |
| 2 | `ds/{eq}/status` | STATUS_UPDATE | 1 | ✅ | EAP→Broker | 6초 | 02, 03 |
| 3 | `ds/{eq}/result` | INSPECTION_RESULT | 1 | ❌ | EAP→Broker | takt ~1620ms | 04~08 |
| 4 | `ds/{eq}/lot` | LOT_END | 2 | ✅ | EAP→Broker | LOT 완료 1회 | 09, 10 |
| 5 | `ds/{eq}/alarm` | HW_ALARM | 2 | ✅ | EAP→Broker | 즉시 | 11~17 |
| 6 | `ds/{eq}/recipe` | RECIPE_CHANGED | 2 | ✅ | EAP→Broker | 변경 즉시 | 18~20 |
| 7 | `ds/{eq}/control` | CONTROL_CMD | 2 | ❌ | MES/Mobile→EAP | 명령 즉시 | 21, 22, 26, 27 |
| 8 | `ds/{eq}/oracle` | ORACLE_ANALYSIS | 2 | ✅ | Oracle→Broker | LOT_END 후 비동기 | 23, 24, 25 |

## 2. 공통 헤더 (모든 페이로드 필수)

| 필드 | 타입 | 예시 |
|---|---|---|
| message_id | UUID v4 | `e7026e09-477c-43c3-8ba5-35b7b7f8a659` |
| event_type | enum | `STATUS_UPDATE` |
| timestamp | ISO 8601 ms | `2026-01-22T16:41:42.123Z` |
| equipment_id | string | `DS-VIS-001` |
| equipment_status | enum | `RUN`/`IDLE`/`STOP` (HEARTBEAT/CONTROL/ORACLE 제외) |

## 3. 이벤트별 핵심 필드 요약

### HEARTBEAT (4필드)
공통 헤더 4개만 (equipment_status 제외). 3초 주기 생존 신호.

### STATUS_UPDATE (v3.4 진행률 포함)
- `lot_id`, `recipe_id`, `recipe_version`, `operator_id`, `uptime_sec`
- **(v3.4 신규)** `current_unit_count`, `expected_total_units`, `current_yield_pct`

### INSPECTION_RESULT (PASS drop 의무)
- summary: `lot_id`, `unit_id`, `strip_id`, `overall_result`(PASS/FAIL), `fail_reason_code`, `fail_count`, `total_inspected_count`
- `inspection_detail`: `prs_result[]` / `side_result[]` (PascalCase: ZAxisNum, InspectionResult, ErrorType, XOffset, YOffset, TOffset)
- `geometric` / `bga` / `surface` / `singulation` / `process` (캡스톤 AI용)
- **모바일은 PASS 시 inspection_detail/detail 그룹 파싱 스킵 의무**

### LOT_END
- `lot_status`(COMPLETED/ABORTED), `total_units`, `pass_count`, `fail_count`, `yield_pct`, `lot_duration_sec`

### HW_ALARM (v3.4 burst 대응)
- `alarm_level`(CRITICAL/WARNING/INFO), `hw_error_code`, `hw_error_source`, `hw_error_detail`
- `exception_detail`(module, exception_type, stack_trace_hash), `auto_recovery_attempted`, `requires_manual_intervention`
- **(v3.4 신규)** `burst_id`, `burst_count`

### RECIPE_CHANGED
- `previous_recipe_id`, `previous_recipe_version`, `new_recipe_id`, `new_recipe_version`, `changed_by`

### CONTROL_CMD (v3.4 ALARM_ACK 추가)
- `command`(EMERGENCY_STOP / ALARM_CLEAR / RECIPE_LOAD / LOT_ABORT / STATUS_QUERY / **ALARM_ACK**), `issued_by`, `reason`, `target_lot_id`, **`target_burst_id`(v3.4)**
- **모바일 허용**: EMERGENCY_STOP, STATUS_QUERY, ALARM_ACK

### ORACLE_ANALYSIS
- `lot_id`, `recipe_id`, `judgment`(NORMAL/WARNING/DANGER), `yield_status`, `ai_comment`, `threshold_proposal`

## 4. hw_error_code 코드북 (v3.4 통합)

| 코드 | 소스 | 레벨 | 설명 |
|---|---|---|---|
| CAM_TIMEOUT_ERR | CAMERA | CRITICAL | GrabLinkGrabber 타임아웃 |
| WRITE_FAIL | VISION | CRITICAL | HALCON #3142 디스크/권한 |
| **VISION_SCORE_ERR** | VISION | WARNING | **통합 코드** — hw_error_detail로 3케이스 구분 (NULL 객체 / Teaching 미완성 / LOT 시작 실패) |
| LIGHT_PWR_LOW | LIGHTING | WARNING | 조명 파라미터 이탈 |
| EAP_DISCONNECTED | PROCESS | CRITICAL | Will 메시지 전용 |
| STAGE_HOME_FAIL | MOTION | CRITICAL | 예약 |
| WATER_FLOW_LOW | COOLANT | WARNING | 예약 |

## 5. ErrorType 코드북 (INSPECTION_RESULT)

| ET | 모듈 | 의미 |
|---|---|---|
| 1 | 공통 | 정상 PASS |
| 11 | PRS | 픽업 오프셋 ±300 초과 |
| 12 | SIDE | 측면 치수/칩핑 기준 미달 |
| 30 | 공통 | 이미지 미취득 (CAM_TIMEOUT 전조) |
| 52 | SIDE | 측면 알고리즘 종합 실패 (Teaching 미완성) |

## 6. Rule 38개 요약

- **R01~R12**: PRS/SIDE 슬롯 판정
- **R13~R18**: 절삭 품질 (chipping, burr, blade, spindle, water)
- **R19~R22**: MAP / takt
- **R23~R25**: LOT 수율 / 소요시간 / 누적 불균형
- **R26~R34**: 알람 발생률
- **R35~R37**: ABORTED / blade_usage / inspection_duration
- **R38a/b/c**: Oracle 복합 (Isolation Forest / EWMA / 비정상 전환)

상세는 API 명세서 §11 참조.

## 7. Mock 파일 인덱스 (27 + scenarios)

01~17 실로그 / 18~25 합성 / 26~27 ALARM_ACK / scenarios/multi_equipment_4x.json (N=4 시나리오)
