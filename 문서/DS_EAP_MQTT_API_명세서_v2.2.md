# DS 주식회사 — EAP → MQTT 브로커 API 명세서

**버전:** v2.2  
**일자:** 2026-03-29  
**등급:** 대외비  
**대상 시스템:** Eclipse Mosquitto 2.x (localhost:1883 / :9001)

---

## 문서 정보

| 항목 | 내용 |
|---|---|
| 문서 번호 | DS-EAP-MQTT-API-001 v2.2 |
| 이전 버전 | v2.1 (2026-03-29) |
| 이벤트 종류 | 7종: STATUS_UPDATE / INSPECTION_RESULT / LOT_END / HW_ALARM / RECIPE_CHANGED / HEARTBEAT / CONTROL |
| 주요 변경 | Will 메시지 페이로드 고정값 명세 추가 (EAP_DISCONNECTED) / 필드 테이블 예시값 컬럼 너비 확장 (타임스탬프 잘림 수정) |
| 네트워크 | 망 분리 공장 현장 · 로컬 Wi-Fi / BLE · Eclipse Mosquitto 2.x |

---

## 1. 개요

본 문서는 DS 주식회사 반도체 후공정 비전 검사 장비(EAP)가 Eclipse Mosquitto MQTT 브로커로 발행·구독하는 7종의 이벤트 메시지에 대한 API 명세서입니다.  
v2.2에서는 Will 메시지 페이로드 고정값 명세 추가, 필드 테이블 예시값 컬럼 너비 확장으로 타임스탬프 잘림을 수정하였습니다.

---

### 1.1 토픽 구조 (v2.2 전체)

> **v2.0 변경**
> - `LOT_END` / `RECIPE_CHANGED`: QoS 1 → QoS 2 상향 (MES · Historian 정합성 보장)
> - `HEARTBEAT` 토픽 신규 / `CONTROL` 토픽 신규 / `RESULT` 필드 그룹화
>
> **v2.1 변경**
> - `STATUS_UPDATE` 주기 10s → 6s 타협 (5~7s 권장 구간 내 채택)
> - MQTT ACL 설정 가이드 부록 신규 추가 (Control 토픽 보안 적용 필수)
> - `INSPECTION_RESULT`: 모바일 앱 메시지 드롭 필터 구현 의무 명시
>
> **v2.2 변경**
> - Will 메시지 페이로드 고정값 명세 추가 (부록 A.4)
> - 타임스탬프 컬럼 잘림 수정

| 토픽 패턴 | 이벤트 타입 | QoS | 방향 | 주기/조건 | 설명 |
|---|---|:---:|:---:|---|---|
| `ds/{eq}/heartbeat` | `HEARTBEAT` | 1 | Pub | 3초 주기 | 온라인 감지 전용 경량 신호 (신규) |
| `ds/{eq}/status` | `STATUS_UPDATE` | 1 | Pub | 6초 주기 | 장비 상태 상세 보고 (5~7s 타협, 6s 채택) |
| `ds/{eq}/result` | `INSPECTION_RESULT` | 1 | Pub | takt (~3.85s) | 단위 검사 결과 (summary + detail) |
| `ds/{eq}/lot` | `LOT_END` | 2 | Pub | Lot 완료 1회 | Lot 수율 집계 (QoS 2 상향) |
| `ds/{eq}/alarm` | `HW_ALARM` | 2 | Pub | 이벤트 즉시 | 하드웨어 알람 |
| `ds/{eq}/recipe` | `RECIPE_CHANGED` | 2 | Pub | 변경 즉시 | 레시피 전환 (QoS 2 상향) |
| `ds/{eq}/control` | `CONTROL_CMD` | 2 | Sub | 명령 즉시 | 원격 제어 명령 수신 (신규) |

> `{eq}` = `equipment_id` 축약 표기. 예: `DS-VIS-001`

---

### 1.2 공통 헤더 필드

> **v2.0 변경**  
> - `timestamp`: ISO 8601 밀리초 단위 강제. `yyyy-MM-ddTHH:mm:ss.fffZ` 형식 필수.  
> - Takt Time 3.85s 환경에서 ms 단위 없으면 TimescaleDB 시계열 정렬 충돌 가능.

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID v4) | `550e8400-e29b-41d4-a716-446655440001` | Y | 메시지 고유 식별자. RFC 4122 UUID v4 형식. 중복 수신 필터링에 활용 |
| `event_type` | string (enum) | `STATUS_UPDATE` | Y | 이벤트 종류. 7종: STATUS_UPDATE / INSPECTION_RESULT / LOT_END / HW_ALARM / RECIPE_CHANGED / HEARTBEAT / CONTROL_CMD |
| `timestamp` | string (ISO 8601) | `2025-07-11T09:31:55.123Z` | Y | **[v2.0]** 밀리초(`.fffZ`) 필수. Takt Time 3.85s 환경에서 ms 단위 없으면 TimescaleDB 정렬 오류 발생 |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 식별자. 토픽 경로의 `{equipment_id}`와 반드시 일치 |
| `equipment_status` | string (enum) | `RUN` | Y | `RUN`(검사 진행) / `IDLE`(대기) / `STOP`(정지). HEARTBEAT·CONTROL 제외 전 토픽 포함 |

---

## 2. HEARTBEAT — 온라인 감지 경량 신호 [v2.0 신규]

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/heartbeat` |
| **이벤트 타입** | `HEARTBEAT` |
| **QoS** | 1 |
| **방향** | EAP → Broker (Publish) |

장비의 네트워크 연결 상태만 확인하는 초경량 신호입니다. 3초 주기로 발행하며 3필드만 포함합니다.  
모바일 앱의 온라인/오프라인 배지 판단에만 사용하고, 상세 상태는 `STATUS_UPDATE`를 참조합니다.

> **[v2.1]** `STATUS_UPDATE` 주기를 5~7s 타협 구간에서 6s로 채택 — Heartbeat(3s)와 역할 명확히 분리

### 2.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...` | Y | 공통 헤더 참조 |
| `event_type` | string | `HEARTBEAT` | Y | 고정값: `HEARTBEAT` |
| `timestamp` | string (ISO 8601) | `2025-07-11T09:31:55.123Z` | Y | 발행 시각 (UTC, 밀리초 필수) |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |

### 2.2 페이로드 예시

```json
{
  "message_id": "7f3a1200-0001-4abc-b100-000000000001",
  "event_type": "HEARTBEAT",
  "timestamp": "2025-07-11T09:31:55.123Z",
  "equipment_id": "DS-VIS-001"
}
```

> ※ Will 메시지 연계: Heartbeat 3회 미수신(9s) 시 모바일 앱이 오프라인으로 판단 권장

---

## 3. STATUS_UPDATE — 장비 상태 상세 보고

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/status` |
| **이벤트 타입** | `STATUS_UPDATE` |
| **QoS** | 1 |
| **방향** | EAP → Broker (Publish) |

장비의 운영 상태를 **6초 주기**로 발행합니다. v2.0의 10s는 장비 다운 감지가 너무 늦다는 피드백을 반영하여 5~7s 권장 구간 내에서 6s로 타협하였습니다. Heartbeat(3s)가 연결 감지를 담당하므로 역할이 분리됩니다.

> **[v2.1]** 발행 주기 10s → 6s 타협 (5~7s 권장 구간 채택 — 다운 감지 지연 보완)

### 3.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...0001` | Y | 공통 헤더 참조 |
| `event_type` | string | `STATUS_UPDATE` | Y | 고정값: `STATUS_UPDATE` |
| `timestamp` | string (ISO 8601) | `2025-07-11T09:31:55.000Z` | Y | 발행 시각 (UTC, 밀리초 필수) |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |
| `equipment_status` | string (enum) | `RUN` | Y | `RUN` / `IDLE` / `STOP` |
| `lot_id` | string | `LOT-20250711-042` | Y | 현재 진행 중인 Lot ID |
| `recipe_id` | string | `RCP-BGA-0.5mm` | Y | 활성 Recipe ID |
| `recipe_version` | string | `v2.1` | Y | Recipe 버전 |
| `operator_id` | string | `ENG-KIM` | Y | 현재 운영 엔지니어 ID |
| `uptime_sec` | integer | `14520` | Y | 장비 기동 후 경과 시간(초). 재시작 시 0 |

### 3.2 페이로드 예시 (Mock 원본)

```json
{
  "message_id": "550e8400-e29b-41d4-a716-446655440001",
  "event_type": "STATUS_UPDATE",
  "timestamp": "2025-07-11T09:31:55.000Z",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "RUN",
  "lot_id": "LOT-20250711-042",
  "recipe_id": "RCP-BGA-0.5mm",
  "recipe_version": "v2.1",
  "operator_id": "ENG-KIM",
  "uptime_sec": 14520
}
```

---

## 4. INSPECTION_RESULT — 단위 검사 결과

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/result` |
| **이벤트 타입** | `INSPECTION_RESULT` |
| **QoS** | 1 |
| **방향** | EAP → Broker (Publish) |

단위 부품(Unit) 1개 검사 완료 후 즉시 발행합니다. 단일 토픽을 유지하되 필드를 **summary 그룹**(양불 판정, 모바일 앱 구독)과 **detail 그룹**(계측값, Historian 구독)으로 명시합니다.

> **[v2.1]** 토픽 분리 대신 단일 토픽 유지 + 클라이언트 선택적 파싱 권장  
> **[v2.1 의무]** 모바일 앱은 메시지 드롭 필터를 반드시 구현해야 합니다.  
> - 구현 방법: `overall_result == "PASS"` 이고 `fail_count == 0` 인 경우 detail 그룹 파싱 스킵  
> - 미구현 시 Takt 3.85s 환경에서 ~800B × 26msg/min = 약 1.2MB/min 불필요 파싱 발생

### 4.1 필드 그룹 설명

- **summary 그룹**: 모바일 앱이 반드시 파싱해야 하는 핵심 필드 (6개)
- **detail 그룹**: Historian / Oracle 서버가 전량 저장하는 계측 상세 필드 (서브오브젝트 6개)

### 4.2 최상위 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...0002` | Y | 공통 헤더 |
| `event_type` | string | `INSPECTION_RESULT` | Y | 고정값 |
| `timestamp` | string (ISO 8601) | `2025-07-11T09:32:00.123Z` | Y | 밀리초 필수 |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |
| `equipment_status` | string | `RUN` | Y | 검사 중 항상 RUN |
| **— summary 그룹 (모바일 앱 필수 파싱) —** | | | | |
| `lot_id` | string | `LOT-20250711-042` | Y | 소속 Lot ID |
| `unit_id` | string | `UNIT-0034` | Y | 단위 부품 ID |
| `overall_result` | string (enum) | `FAIL` | Y | `PASS` / `FAIL` |
| `fail_reason_code` | string | `CHIPPING_EXCEED` | N | FAIL 시 불량 코드. PASS이면 null |
| `fail_count` | integer | `2` | Y | 불량 개수 |
| `total_inspected_count` | integer | `128` | Y | 총 검사 수 |
| **— detail 그룹 (Historian / Oracle 전량 저장) —** | | | | |
| `strip_id` | string | `STRIP-007` | Y | 소속 Strip ID |
| `recipe_id` | string | `RCP-BGA-0.5mm` | Y | 적용 Recipe |
| `recipe_version` | string | `v2.1` | Y | Recipe 버전 |
| `operator_id` | string | `ENG-KIM` | Y | 운영자 ID |
| `geometric` | object | `{ ... }` | Y | 외형 치수 (§4.3) |
| `bga` | object | `{ ... }` | Y | BGA 볼 품질 (§4.4) |
| `surface` | object | `{ ... }` | Y | 표면 결함 (§4.5) |
| `singulation` | object | `{ ... }` | Y | 절삭 품질 (§4.6) |
| `saw_process` | object | `{ ... }` | Y | 공정 설비 상태 (§4.7) |
| `process` | object | `{ ... }` | Y | 검사 성능 메타 (§4.8) |

### 4.3 geometric — 외형 치수 및 정렬

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `dimension_w_mm` | float | `10.02` | Y | 패키지 가로 치수 (mm) |
| `dimension_l_mm` | float | `10.01` | Y | 패키지 세로 치수 (mm) |
| `dimension_h_mm` | float | `1.45` | Y | 패키지 두께 Z-height (mm) |
| `x_offset_um` | float | `12.5` | Y | X축 중심 오프셋 (μm). 양수=우측 이탈 |
| `y_offset_um` | float | `-8.3` | Y | Y축 중심 오프셋 (μm). 음수=하방 이탈 |
| `theta_deg` | float | `0.15` | Y | 회전 각도 편차 (°). 허용 범위 ±0.5° |
| `kerf_width_um` | float | `52.3` | Y | 블레이드 절삭 폭 (μm). 마모 지표 |

### 4.4 bga — BGA 볼 품질

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `available` | boolean | `true` | Y | BGA 검사 활성 여부 |
| `ball_count_nominal` | integer | `256` | Y | 설계 볼 개수 |
| `ball_count_actual` | integer | `256` | Y | 실측 볼 개수. nominal과 차이 시 FAIL |
| `ball_diameter_avg_mm` | float | `0.498` | Y | 볼 평균 지름 (mm) |
| `coplanarity_mm` | float | `0.082` | Y | 볼 높이 편차 최댓값 (mm). 기준 0.1mm 이하 |
| `pitch_deviation_um` | float | `4.2` | Y | 볼 간격 기준값 대비 편차 (μm) |
| `max_ball_offset_um` | float | `18.7` | Y | 패드 중심 대비 최대 볼 위치 편차 (μm) |
| `avg_ball_offset_um` | float | `6.1` | Y | 패드 중심 대비 평균 볼 위치 편차 (μm) |

### 4.5 surface — 표면 결함

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `foreign_material_size_um` | float | `0` | Y | 이물질(F/M) 최대 크기 (μm). 0이면 없음 |
| `scratch_area_mm2` | float | `0.003` | Y | 스크래치 면적 (mm²) |
| `marking_quality_grade` | string | `A` | Y | 마킹 품질 등급: A(최우수) ~ F(불량) |

### 4.6 singulation — 절삭 품질

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `chipping_top_um` | float | `35.0` | Y | 상면 칩핑 최대 크기 (μm). 기준 50μm 이하 |
| `chipping_bottom_um` | float | `28.5` | Y | 하면 칩핑 최대 크기 (μm). 기준 45μm 이하 |
| `burr_height_um` | float | `5.2` | Y | 버(Burr) 높이 (μm). 블레이드 마모와 상관 |

### 4.7 saw_process — 절삭 설비 상태

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `blade_rpm` | integer | `30000` | Y | 블레이드 회전수 (RPM) |
| `blade_wear_index` | float | `0.72` | Y | 블레이드 마모도 (0.0~1.0). 0.85 이상 교체 권고 |
| `blade_usage_count` | integer | `18450` | Y | 블레이드 누적 사용 횟수 |
| `cutting_water_flow_lpm` | float | `2.5` | Y | 절삭수 유량 (L/min). 기준 1.5 이상 |
| `cutting_water_temp_c` | float | `22.3` | Y | 절삭수 온도 (°C) |
| `kerf_deviation_um` | float | `1.8` | Y | 절삭선 경로 편차 (μm) |
| `spindle_load_pct` | float | `64.2` | Y | 스핀들 부하율 (%). 80% 이상 주의 |

### 4.8 process — 검사 성능 메타

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `inspection_duration_ms` | integer | `1240` | Y | 순수 검사 소요 시간 (ms) |
| `takt_time_ms` | integer | `3850` | Y | 투입~배출 전체 소요 시간 (ms) |
| `algorithm_version` | string | `v4.3.1` | Y | 비전 알고리즘 버전 |

### 4.9 fail_reason_code 코드 목록

| 코드 값 | 관련 서브오브젝트 | 설명 |
|---|---|---|
| `CHIPPING_EXCEED` | singulation | 상/하면 칩핑 크기가 허용 기준 초과 |
| `BGA_BALL_MISSING` | bga | 실측 볼 개수가 설계값 미달 |
| `COPLANARITY_EXCEED` | bga | 볼 평탄도(coplanarity)가 0.1mm 초과 |
| `FOREIGN_MATERIAL_DETECTED` | surface | 이물질(F/M) 크기 기준값 초과 |
| `DIMENSION_OUT_OF_SPEC` | geometric | 패키지 치수가 허용 공차 초과 |
| `MARKING_QUALITY_FAIL` | surface | 마킹 품질 등급 C 이하 |

---

## 5. LOT_END — Lot 완료 이벤트

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/lot` |
| **이벤트 타입** | `LOT_END` |
| **QoS** | 2 |
| **방향** | EAP → Broker (Publish) |

Lot 내 전체 단위 검사 완료 직후 1회 발행합니다. 수율 집계와 Lot 이력은 MES·Historian 정합성에 직결되므로 **QoS 2(정확히 1회 전달)**로 상향하였습니다.

> **[v2.0]** QoS 1 → QoS 2 상향 (수율 데이터 유실 시 MES · Historian 정합성 파괴 방지)

### 5.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...0003` | Y | 공통 헤더 |
| `event_type` | string | `LOT_END` | Y | 고정값: `LOT_END` |
| `timestamp` | string (ISO 8601) | `2025-07-11T10:15:33.500Z` | Y | 완료 시각 (UTC, 밀리초 필수) |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |
| `equipment_status` | string | `IDLE` | Y | Lot 완료 시 항상 `IDLE` |
| `lot_id` | string | `LOT-20250711-042` | Y | 완료된 Lot ID |
| `lot_status` | string (enum) | `COMPLETED` | Y | `COMPLETED` / `ABORTED` / `ERROR` |
| `total_units` | integer | `128` | Y | 전체 검사 단위 수 |
| `pass_count` | integer | `126` | Y | 합격 단위 수 |
| `fail_count` | integer | `2` | Y | 불합격 단위 수 |
| `yield_pct` | float | `98.44` | Y | 수율 (%). `pass_count / total_units × 100` |
| `lot_duration_sec` | integer | `2618` | Y | Lot 소요 시간 (초) |

### 5.2 페이로드 예시 (Mock 원본)

```json
{
  "message_id": "550e8400-e29b-41d4-a716-446655440003",
  "event_type": "LOT_END",
  "timestamp": "2025-07-11T10:15:33.500Z",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "IDLE",
  "lot_id": "LOT-20250711-042",
  "lot_status": "COMPLETED",
  "total_units": 128,
  "pass_count": 126,
  "fail_count": 2,
  "yield_pct": 98.44,
  "lot_duration_sec": 2618
}
```

---

## 6. HW_ALARM — 하드웨어 알람

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/alarm` |
| **이벤트 타입** | `HW_ALARM` |
| **QoS** | 2 |
| **방향** | EAP → Broker (Publish) |

카메라·조명·모션 스테이지 이상 또는 소프트웨어 예외 발생 시 즉시 발행합니다. **QoS 2(정확히 1회)**로 알람 누락을 방지합니다.

### 6.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...0005` | Y | 공통 헤더 |
| `event_type` | string | `HW_ALARM` | Y | 고정값: `HW_ALARM` |
| `timestamp` | string (ISO 8601) | `2025-07-11T10:22:05.789Z` | Y | 알람 발생 시각 (UTC, 밀리초 필수) |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |
| `equipment_status` | string | `STOP` | Y | CRITICAL이면 `STOP`, WARNING이면 `RUN` 가능 |
| `alarm_level` | string (enum) | `CRITICAL` | Y | `CRITICAL` / `WARNING` / `INFO` |
| `hw_error_code` | string | `CAM_TIMEOUT_ERR` | Y | 오류 코드 (§6.3) |
| `hw_error_source` | string (enum) | `CAMERA` | Y | `CAMERA` / `LIGHTING` / `MOTION` / `VISION` / `COOLANT` |
| `hw_error_detail` | string | `Top camera failed...` | Y | 오류 상세 설명 (자유 형식 텍스트) |
| `exception_detail` | object | `{ ... }` | N | SW 예외 상세 (§6.2). 없으면 null |
| `auto_recovery_attempted` | boolean | `false` | Y | 자동 복구 시도 여부 |
| `requires_manual_intervention` | boolean | `true` | Y | 수동 처리 필요 여부. `true`이면 모바일 알림 강조 |

### 6.2 exception_detail 서브오브젝트

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `module` | string | `VisionInspector.Capture` | Y | 예외 발생 C# 네임스페이스.클래스 |
| `exception_type` | string | `TimeoutException` | Y | 예외 타입명 |
| `stack_trace_hash` | string | `a3f9c2e1` | Y | 스택 트레이스 앞 8자리 해시 (로그 연동용) |

### 6.3 hw_error_code 코드 목록

| 오류 코드 | 소스 | 레벨 | 자동복구 | 설명 |
|---|---|---|:---:|---|
| `CAM_TIMEOUT_ERR` | CAMERA | CRITICAL | N | 카메라 응답 없음 (500ms 초과). 수동 처리 필요 |
| `LIGHT_PWR_LOW` | LIGHTING | WARNING | Y | 조명 출력 80% 미만. 자동 복구 시도 |
| `STAGE_HOME_FAIL` | MOTION | CRITICAL | N | X축 홈 복귀 3회 실패. 수동 처리 필요 |
| `VISION_SCORE_ERR` | VISION | WARNING | Y | 알고리즘 신뢰도 0.5 미만 3회 연속 |
| `WATER_FLOW_LOW` | COOLANT | WARNING | Y | 절삭수 유량 1.5 lpm 미만 |
| `EAP_DISCONNECTED` | PROCESS | CRITICAL | N | **Will 메시지 전용 예약 코드.** EAP 비정상 종료 (§부록 A.4) |

### 6.4 페이로드 예시 (Mock 원본)

```json
{
  "message_id": "550e8400-e29b-41d4-a716-446655440005",
  "event_type": "HW_ALARM",
  "timestamp": "2025-07-11T10:22:05.789Z",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "STOP",
  "alarm_level": "CRITICAL",
  "hw_error_code": "CAM_TIMEOUT_ERR",
  "hw_error_source": "CAMERA",
  "hw_error_detail": "Top inspection camera failed to respond within 500ms threshold.",
  "exception_detail": {
    "module": "VisionInspector.Capture",
    "exception_type": "TimeoutException",
    "stack_trace_hash": "a3f9c2e1"
  },
  "auto_recovery_attempted": false,
  "requires_manual_intervention": true
}
```

---

## 7. RECIPE_CHANGED — 레시피 변경 이벤트

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/recipe` |
| **이벤트 타입** | `RECIPE_CHANGED` |
| **QoS** | 2 |
| **방향** | EAP → Broker (Publish) |

레시피 변경 시 즉시 발행합니다. 레시피 이력은 Historian의 공정 이력 추적에 직결되므로 **QoS 2**로 상향하였습니다.

> **[v2.0]** QoS 1 → QoS 2 상향 (레시피 이력 유실 시 Historian 공정 추적 불가)

### 7.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `550e8400-...0004` | Y | 공통 헤더 |
| `event_type` | string | `RECIPE_CHANGED` | Y | 고정값: `RECIPE_CHANGED` |
| `timestamp` | string (ISO 8601) | `2025-07-11T10:17:10.000Z` | Y | 변경 발생 시각 (UTC, 밀리초 필수) |
| `equipment_id` | string | `DS-VIS-001` | Y | 장비 고유 ID |
| `equipment_status` | string | `IDLE` | Y | 변경 시 항상 `IDLE` |
| `previous_recipe_id` | string | `RCP-BGA-0.5mm` | Y | 변경 전 Recipe ID |
| `previous_recipe_version` | string | `v2.1` | Y | 변경 전 Recipe 버전 |
| `new_recipe_id` | string | `RCP-QFN-0.4mm` | Y | 변경 후 Recipe ID |
| `new_recipe_version` | string | `v1.3` | Y | 변경 후 Recipe 버전 |
| `changed_by` | string | `ENG-KIM` | Y | 변경 주체. 엔지니어 ID 또는 `MES_AUTO` |

### 7.2 페이로드 예시 (Mock 원본)

```json
{
  "message_id": "550e8400-e29b-41d4-a716-446655440004",
  "event_type": "RECIPE_CHANGED",
  "timestamp": "2025-07-11T10:17:10.000Z",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "IDLE",
  "previous_recipe_id": "RCP-BGA-0.5mm",
  "previous_recipe_version": "v2.1",
  "new_recipe_id": "RCP-QFN-0.4mm",
  "new_recipe_version": "v1.3",
  "changed_by": "ENG-KIM"
}
```

### 7.3 이벤트 발생 시나리오

| 트리거 | changed_by 값 | equipment_status | 설명 |
|---|---|:---:|---|
| 현장 엔지니어 수동 변경 | `ENG-KIM` | IDLE | 운영자가 HMI에서 직접 Recipe 선택 |
| MES 자동 전환 | `MES_AUTO` | IDLE | MES에서 다음 Lot의 Recipe를 자동 투입 |
| 긴급 Recipe 롤백 | `ENG-PARK` | IDLE | 불량 발생으로 이전 Recipe로 복구 |

---

## 8. CONTROL_CMD — 원격 제어 명령 [v2.0 신규]

| 항목 | 내용 |
|---|---|
| **토픽** | `ds/{equipment_id}/control` |
| **이벤트 타입** | `CONTROL_CMD` |
| **QoS** | 2 |
| **방향** | Broker → EAP (Subscribe) |

MES 서버 또는 모바일 앱이 장비에 명령을 내리는 **Subscribe 전용** 토픽입니다.  
모바일 앱은 읽기 전용(Read-Only)이 원칙이며 긴급 STOP만 허용합니다. 토픽 접근 제어는 MQTT ACL 설정으로 브로커 레벨에서 강제해야 합니다 (부록 A 참조).

> **[v2.1]** MQTT ACL 설정 부록 신규 추가 — 명세서만으로는 보안 적용 불가. 부록 A 필수 적용

### 8.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
|---|---|---|:---:|---|
| `message_id` | string (UUID) | `7f3a1200-...` | Y | 명령 고유 식별자 |
| `event_type` | string | `CONTROL_CMD` | Y | 고정값: `CONTROL_CMD` |
| `timestamp` | string (ISO 8601) | `2025-07-11T10:30:00.000Z` | Y | 명령 발행 시각 (UTC, 밀리초 필수) |
| `command` | string (enum) | `EMERGENCY_STOP` | Y | 제어 명령 코드 (§8.2) |
| `issued_by` | string | `MES_SERVER` | Y | 명령 발행 주체. `MES_SERVER` / `MOBILE_APP` / `OPERATOR` |
| `reason` | string | `Yield drop alert` | N | 명령 사유 (자유 형식). 감사 로그용 |
| `target_lot_id` | string | `LOT-20250711-042` | N | 특정 Lot 대상 명령 시 명시. 없으면 null |

### 8.2 command 코드 목록

| 명령 코드 | 발행 주체 | 설명 | 모바일 허용 |
|---|---|---|:---:|
| `EMERGENCY_STOP` | MES / 모바일 | 즉시 장비 정지. 진행 중 Lot 중단 | Y (긴급) |
| `ALARM_CLEAR` | MES | 알람 해제 및 장비 복구 시도 | N (MES 전용) |
| `RECIPE_LOAD` | MES | 지정 Recipe 로드 (MES 자동 전환) | N (MES 전용) |
| `LOT_ABORT` | MES | 현재 Lot 강제 종료. `LOT_END` 발행 유발 | N (MES 전용) |
| `STATUS_QUERY` | MES / 모바일 | 즉시 `STATUS_UPDATE` 발행 요청 | Y (읽기) |

### 8.3 페이로드 예시

```json
{
  "message_id": "7f3a1200-ctrl-4abc-b100-000000000010",
  "event_type": "CONTROL_CMD",
  "timestamp": "2025-07-11T10:30:00.000Z",
  "command": "EMERGENCY_STOP",
  "issued_by": "MOBILE_APP",
  "reason": "Yield drop detected: 85% below threshold",
  "target_lot_id": "LOT-20250711-042"
}
```

---

## 9. 연결 규격 및 운영 정책

| 항목 | v1.0 | v2.0 | v2.1 | v2.2 변경 내용 |
|---|---|---|---|---|
| 프로토콜 | MQTT v5.0 | 동일 | 동일 | 동일 |
| Timestamp | ISO 8601 | 밀리초 강제 | 동일 | 동일 |
| QoS — LOT_END | QoS 1 | QoS 2 | 동일 | 동일 |
| QoS — RECIPE | QoS 1 | QoS 2 | 동일 | 동일 |
| Heartbeat | STATUS 겸임 | 별도 토픽(3s) | 동일 | 동일 |
| STATUS 주기 | 5s | 10s | 6s 타협 | 동일 |
| Control | 미정의 | 신규 | ACL 부록 추가 | 동일 |
| RESULT | 단일 토픽 | 필드 그룹화 | 드롭 필터 의무 | 동일 |
| Will 메시지 | alarm 발행 | 동일 | 동일 | **페이로드 고정값 명시 (부록 A.4)** |
| 재연결 | 지수 백오프 | 동일 | 동일 | 동일 |

---

## 10. 이벤트 흐름 시나리오 (v2.2)

| 순서 | 이벤트 타입 | 토픽 | QoS | status | 설명 |
|:---:|---|---|:---:|:---:|---|
| 1 | `HEARTBEAT` | `.../heartbeat` | 1 | IDLE | 3초 주기. 장비 온라인 확인 |
| 2 | `RECIPE_CHANGED` | `.../recipe` | 2 | IDLE | Lot 투입 전 Recipe 확정 |
| 3 | `STATUS_UPDATE` | `.../status` | 1 | RUN | Lot 시작. 6초 주기 반복 |
| 4~N | `INSPECTION_RESULT` | `.../result` | 1 | RUN | Unit 완료 시마다 (128회). 모바일 드롭 필터 적용 |
| N+1 | `LOT_END` | `.../lot` | 2 | IDLE | Lot 수율 집계 (QoS 2) |
| * | `HW_ALARM` | `.../alarm` | 2 | * | 이상 발생 즉시 (비동기). Will 포함 |
| * | `CONTROL_CMD` | `.../control` | 2 | * | MES/모바일 명령 수신 (Sub). ACL 필수 |

---

## 부록 A. MQTT ACL 설정 가이드 [v2.1 신규]

Control 토픽은 잘못된 클라이언트가 장비에 명령을 내릴 수 있어 보안이 필수입니다.  
Mosquitto의 ACL 파일(`acl_file`)로 클라이언트별 토픽 접근 권한을 브로커 레벨에서 강제합니다.

### A.1 mosquitto.conf 설정

```conf
# /etc/mosquitto/mosquitto.conf
allow_anonymous false
password_file /etc/mosquitto/passwd
acl_file      /etc/mosquitto/acl
```

### A.2 ACL 파일 규칙 (/etc/mosquitto/acl)

```conf
# EAP device account (DS-VIS-001) — Publish only
user eap_vis_001
topic write ds/DS-VIS-001/#
topic read  ds/DS-VIS-001/control

# MES server account — full Publish/Subscribe
user mes_server
topic readwrite ds/#

# Mobile app account — Subscribe only + limited Publish (control)
user mobile_app
topic read  ds/#
topic write ds/+/control
```

### A.3 클라이언트별 권한 요약

| 클라이언트 | 계정 ID | Publish 가능 토픽 | Subscribe 가능 토픽 |
|---|---|---|---|
| EAP 장비 | `eap_vis_{id}` | `ds/{id}/heartbeat·status·result·lot·alarm·recipe` | `ds/{id}/control` |
| MES 서버 | `mes_server` | `ds/#` (전체) | `ds/#` (전체) |
| 모바일 앱 | `mobile_app` | `ds/+/control` (EMERGENCY_STOP·STATUS_QUERY만) | `ds/#` (전체) |
| Historian | `historian` | 없음 | `ds/#` (전체) |

> ※ 모바일 앱의 Control Publish는 ACL로는 토픽 레벨만 제한 가능. `command` 필드 값 검증은 EAP 측 구현 필요

---

### A.4 Will 메시지 페이로드 고정값 [v2.2 신규]

EAP 프로세스가 비정상 종료될 때 브로커가 자동으로 발행하는 Will 메시지의 페이로드입니다.  
`HW_ALARM` 형식을 그대로 따르므로 모바일 앱은 기존 알람 파싱 코드로 비정상 종료를 처리할 수 있습니다.

> - `alarm_level` 고정값: **`CRITICAL`** — 모바일 앱이 최우선 알림으로 처리
> - `hw_error_code` 고정값: **`EAP_DISCONNECTED`** — 네트워크 단절/프로세스 비정상 종료 전용 코드
> - `requires_manual_intervention` 고정값: **`true`** — 현장 엔지니어 즉시 확인 필요
> - `equipment_status` 고정값: **`STOP`** — 장비가 응답 불가 상태임을 명시

#### Will 메시지 MQTT 클라이언트 설정 (C# EAP 코드 기준)

```csharp
var willMsg = new MqttApplicationMessageBuilder()
    .WithTopic($"ds/{equipmentId}/alarm")
    .WithPayload(JsonSerializer.Serialize(new {
        message_id         = Guid.NewGuid().ToString(),
        event_type         = "HW_ALARM",
        timestamp          = DateTime.UtcNow.ToString("yyyy-MM-ddTHH:mm:ss.fffZ"),
        equipment_id       = equipmentId,
        equipment_status   = "STOP",
        alarm_level        = "CRITICAL",
        hw_error_code      = "EAP_DISCONNECTED",
        hw_error_source    = "PROCESS",
        hw_error_detail    = "EAP process terminated unexpectedly.",
        auto_recovery_attempted      = false,
        requires_manual_intervention = true
    }))
    .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
    .WithRetainFlag(false)
    .Build();
```

#### Will 메시지 페이로드 예시 (JSON)

```json
{
  "message_id": "auto-generated-uuid",
  "event_type": "HW_ALARM",
  "timestamp": "2025-07-11T10:45:00.000Z",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "STOP",
  "alarm_level": "CRITICAL",
  "hw_error_code": "EAP_DISCONNECTED",
  "hw_error_source": "PROCESS",
  "hw_error_detail": "EAP process terminated unexpectedly.",
  "auto_recovery_attempted": false,
  "requires_manual_intervention": true
}
```

> ※ Will 메시지 `timestamp`는 연결 수립 시점 기준으로 고정됩니다. 실제 단절 시각과 최대 KeepAlive(30s) 차이가 발생할 수 있습니다.  
> ※ `hw_error_code: EAP_DISCONNECTED`는 Will 메시지 전용 예약 코드입니다. 정상 알람 흐름에서는 발행되지 않습니다.

---

## 부록 B. 개정 이력

| 버전 | 일자 | 변경 내용 |
|---|---|---|
| v1.0 | 2026-03-29 | 최초 작성. 5종 이벤트 명세 (STATUS / RESULT / LOT_END / HW_ALARM / RECIPE_CHANGED) |
| v2.0 | 2026-03-29 | QoS 상향(LOT_END·RECIPE_CHANGED 1→2) / Timestamp 밀리초 강제 / HEARTBEAT 신규 / CONTROL 신규 / RESULT 필드 그룹화 / STATUS 주기 10s 완화 |
| v2.1 | 2026-03-29 | STATUS 주기 10s→6s 재타협(5~7s 권장 구간) / MQTT ACL 부록 A 신규 추가 / 모바일 드롭 필터 구현 의무 명시 / 연결 규격표 3열 비교 확장 |
| v2.2 | 2026-03-29 | Will 메시지 페이로드 고정값 명세 추가 (alarm_level: CRITICAL, hw_error_code: EAP_DISCONNECTED) / 필드 테이블 예시값 컬럼 너비 확장으로 타임스탬프 잘림 수정 / 연결 규격표 v2.2 열 추가 |
