# DS 주식회사 EAP MQTT 브로커 API 명세서

Ver 3.3 (최종) · 2026-04-07 · 대외비  
Carsem 실가동 로그 14일 분석 반영 | v3.2(재혁) + v3.0(팀원) 통합 최종본

| 문서 번호 | DS-EAP-MQTT-API-001 v3.3 |
| :--- | :--- |
| 이전 버전 | v3.2 (재혁) + v3.0 (팀원) 통합 |
| 이벤트 종류 | 8종: HEARTBEAT / STATUS_UPDATE / INSPECTION_RESULT / LOT_END / HW_ALARM / RECIPE_CHANGED / CONTROL_CMD / ORACLE_ANALYSIS |
| Rule 기준 | 38개 (R01~R38) — CRITICAL / WARNING / NORMAL 3단계 |
| v3.3 병합 내용 | v3.2 inspection_detail 구조 + v3.0 Rule 38개 판정표 / 상태전환 / 레시피 판정 / 버전 비교 |
| 실측 기준 | Carsem Inc. / GVisionWpf / 2026-01-16~29 \| takt=1,620ms / total_units=2,792 / yield=96.2% |
| 네트워크 | 망 분리 공장 현장 로컬 Wi-Fi / BLE Eclipse Mosquitto 2.x |

## 1. 개요
본 문서는 DS 주식회사 반도체 후공정 비전 검사 장비(EAP / GVisionWpf)가 Eclipse Mosquitto MQTT 브로커와 교환하는 이벤트 메시지 API 명세서입니다. v3.3은 Carsem 현장 실가동 로그(2026-01-16~29, 14일)를 완전 반영하였습니다.

> [v3.3] inspection_detail 서브오브젝트: prs_result / side_result 배열, PascalCase 키명 (GVisionWpf 로그 원본 구조)
> [v3.3] VISION_SCORE_ERR 단일 코드 통합 — hw_error_detail로 3케이스 구분 / LIGHT_PWR_LOW 교체
> [v3.3] Rule 38개 종합 판정표 — Oracle 서버 if-else 구현 직결 (§11)
> [v3.3] 상태 전환 규칙 6케이스 / 레시피 전환 판정 5종 / 수율 5단계 / LOT 소요시간 판정
> [v3.3] ORACLE_ANALYSIS 이벤트: 레시피별 독립 학습(EWMA+MAD / Isolation Forest), 오퍼레이터 승인 구조

### 1.1 토픽 구조 (v3.3)

| 토픽 패턴 | 이벤트 타입 | QoS | 방향 | 주기/조건 | 설명 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ds/{eq}/heartbeat | HEARTBEAT | 1 | Pub | 3초 | 온라인 감지 경량 신호 (실측 확인) |
| ds/{eq}/status | STATUS_UPDATE | 1 | Pub | 6초 | 장비 상태 보고 |
| ds/{eq}/result | INSPECTION_RESULT | 1 | Pub | takt ~1,620ms | 단위 검사 결과 (PRS/SIDE 8슬롯) |
| ds/{eq}/lot | LOT_END | 2 | Pub | Lot 완료 1회 | 수율 집계 (실측 82~400분) |
| ds/{eq}/alarm | HW_ALARM | 2 | Pub | 이벤트 즉시 | HW/SW 알람 (Will 포함) |
| ds/{eq}/recipe | RECIPE_CHANGED | 2 | Pub | 변경 즉시 | 레시피 전환 |
| ds/{eq}/control | CONTROL_CMD | 2 | Sub | 명령 즉시 | 원격 제어 (ACL 필수) |
| ds/{eq}/oracle | ORACLE_ANALYSIS | 2 | Pub | LOT_END 후 비동기 | Oracle 2차 검증 결과 |

### 1.2 공통 헤더 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID v4) | e7026e09-477c-43c3-8ba5-35b7b7f8a659 | Y | 메시지 고유 식별자. RFC 4122 UUID v4 |
| event_type | string (enum) | STATUS_UPDATE | Y | 이벤트 종류 (8종) |
| timestamp | string (ISO 8601) | 2026-01-22T16:41:42.123Z | Y | UTC 밀리초(.fffZ) 필수. takt 1,620ms 환경 대응 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 식별자 (Genesem VELOCE-G7) |
| equipment_status | string (enum) | RUN | Y | RUN / IDLE / STOP. HEARTBEAT·CONTROL·ORACLE 제외 |

---

## 2. HEARTBEAT — 온라인 감지 경량 신호

```text
TOPIC       ds/{equipment_id}/heartbeat
EVENT TYPE  HEARTBEAT  
QoS         QoS 1  
방향        EAP → Broker
```
3초 주기 생존 신호. 4필드만 포함. 장비 RUN/IDLE 무관 항상 발행.

> [실측] Carsem 현장 14일간 3초 주기 정상 확인. Heartbeat 정상 수신 중에도 장비 비정상 가능 — STATUS_UPDATE 병행 확인 필수

### 2.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | e7026e09-477c-43c3-8ba5-35b7b7f8a659 | Y | 공통 헤더 |
| event_type | string | HEARTBEAT | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T16:17:10.921Z | Y | UTC 밀리초 필수 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |

### 2.2 페이로드 예시

```json
{ 
  "message_id": "e7026e09-477c-43c3-8ba5-35b7b7f8a659",
  "event_type": "HEARTBEAT",
  "timestamp": "2026-01-22T16:17:10.921Z",
  "equipment_id": "DS-VIS-001" 
}
```

### 2.3 판정 기준 (Rule R01)

| 상태 | 조건 | 조치 |
| :--- | :--- | :--- |
| ONLINE | 최근 9초 이내 수신 | 정상 |
| WARNING | 9~30초 미수신 | 네트워크 상태 확인 |
| OFFLINE | 30초 초과 미수신 | 모바일 앱 오프라인 배지 전환 + Will 메시지 확인 |

---

## 3. STATUS_UPDATE — 장비 상태 보고

```text
TOPIC       ds/{equipment_id}/status
EVENT TYPE  STATUS_UPDATE  
QoS         QoS 1  
방향        EAP → Broker
```
6초 주기. RUN(검사 진행) / IDLE(대기) / STOP(정지). Lot ID / Recipe ID / 운영자 / uptime 포함.

### 3.1 페이로드 필드 (RUN / IDLE 비교)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | f31b70a2-... / 05d3415c-... | Y | 공통 헤더 |
| event_type | string | STATUS_UPDATE | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T16:41:41.000Z / 17:39:13.000Z | Y | 발행 시각 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |
| equipment_status | string (enum) | RUN / IDLE | Y | 현재 장비 상태 |
| lot_id | string | LOT-20260122-001 | Y | 현재(또는 마지막) Lot ID |
| recipe_id | string | Carsem_3X3 | Y | 활성(또는 마지막) Recipe ID |
| recipe_version | string | v1.0 | Y | Recipe 버전 |
| operator_id | string | ENG-KIM | Y | 운영 엔지니어 ID |
| uptime_sec | integer | 1528 / 6420 | Y | 기동 후 경과(초). 실측: 시작 1528s / IDLE 6420s |

### 3.2 페이로드 예시 (실측 데이터)

```json
// RUN — Lot in progress
{ 
  "event_type": "STATUS_UPDATE", 
  "equipment_status": "RUN",
  "lot_id": "LOT-20260122-001", 
  "recipe_id": "Carsem_3X3",
  "operator_id": "ENG-KIM", 
  "uptime_sec": 1528 
}

// IDLE — after LOT_END
{ 
  "event_type": "STATUS_UPDATE", 
  "equipment_status": "IDLE",
  "lot_id": "LOT-20260122-001", 
  "recipe_id": "Carsem_3X3",
  "operator_id": "ENG-KIM", 
  "uptime_sec": 6420 
}
```

### 3.3 상태 전환 규칙

| 이전 상태 | 이후 상태 | 트리거 이벤트 | 정상 여부 |
| :--- | :--- | :--- | :--- |
| IDLE | RUN | LOT 시작 | 정상 |
| RUN | IDLE | LOT_END (COMPLETED) | 정상 |
| RUN | IDLE | LOT_END (ABORTED) | 비정상 — 원인 조사 필요 |
| RUN | STOP | HW_ALARM (CRITICAL) | 비정상 — 즉시 조치 |
| RUN | RUN | STATUS 계속 발행 (6초 주기) | 정상 |
| STOP | STOP | HW_ALARM 후 미복구 | 비정상 — 수동 개입 필요 |

---

## 4. INSPECTION_RESULT — 단위 검사 결과

```text
TOPIC       ds/{equipment_id}/result
EVENT TYPE  INSPECTION_RESULT  
QoS         QoS 1  
방향        EAP → Broker
```
단위 부품 1개 검사 완료 후 발행. takt 실측 ~1,620ms. 8슬롯(ZAxisNum 0~7) 기준.

> [실측] 모바일 드롭 필터 의무: overall_result=PASS AND fail_count=0 → inspection_detail / detail 파싱 스킵

### 4.1 필드 그룹

* [구조] summary 그룹: `overall_result` / `fail_count` 등 — 모바일 앱 필수 파싱
* [구조] inspection_detail 그룹: `prs_result[]` / `side_result[]` — GVisionWpf 로그 원본 PascalCase 구조
* [구조] detail 그룹: `geometric` / `bga` / `surface` / `singulation` — 캡스톤 AI 분석용
* [구조] process 그룹: `takt_time_ms` / `inspection_duration_ms`

### 4.2 최상위 필드 (PASS / FAIL 비교)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | b84183e2-... / acc17ae8-... | Y | 공통 헤더 |
| event_type | string | INSPECTION_RESULT | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T16:41:42.123Z / 2026-01-27T11:36:52.000Z | Y | 밀리초 필수 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |
| equipment_status | string | RUN | Y | 항상 RUN |
| **-- summary 그룹 --** | | | N | **모바일 앱 필수 파싱** |
| lot_id | string | LOT-20260122-001 / LOT-20260127-001 | Y | 소속 Lot ID |
| unit_id | string | UNIT-0001 | Y | 단위 부품 ID |
| strip_id | string | STRIP-001 | Y | 소속 Strip ID |
| recipe_id | string | Carsem_3X3 / Carsem_4X6 | Y | 적용 Recipe ID |
| recipe_version | string | v1.0 | Y | Recipe 버전 |
| operator_id | string | ENG-KIM | Y | 운영자 ID |
| overall_result | string (enum) | PASS / FAIL | Y | PASS / FAIL |
| fail_reason_code | string | null / SIDE_VISION_FAIL | N | FAIL 시 코드. PASS이면 null (§4.6) |
| fail_count | integer | 0 / 8 | Y | 불량 슬롯 수. PASS이면 0 |
| total_inspected_count | integer | 8 | Y | 슬롯 수. ZAxisNum 0~7 고정 = 8 |
| **-- inspection_detail 그룹 --** | | | N | **실측 로그 원본 구조** |
| inspection_detail | object | { prs_result, side_result } | Y | 슬롯별 검사 원시 데이터 (§4.3) |
| **-- detail 그룹 --** | | | N | **캡스톤 AI용** |
| geometric | object | { ... } | N | 외형 치수 |
| bga | object | { ... } | N | BGA 볼 품질 |
| surface | object | { ... } | N | 표면 결함 |
| singulation | object | { ... } | N | 절삭 품질 (§4.4) |
| process | object | { ... } | Y | 검사 성능 메타 (§4.5) |

### 4.3 inspection_detail 서브오브젝트
GVisionWpf 로그 원본 키명 그대로. PascalCase. CameraId=1(PRS)/5(SIDE)

#### 4.3.1 prs_result[] — PRS 픽앤소트 슬롯 (CameraId=1, InspectionType=10)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| ZAxisNum | integer (0~7) | 0 | Y | 슬롯 번호 |
| InspectionResult | integer (0/1) | 1 | Y | 1=PASS / 0=FAIL |
| ErrorType | integer | 1 | Y | 오류 코드 (§4.7). 1=정상 |
| XOffset | integer | 121 | Y | X축 오프셋. 정상 ±300. 실측: -25~121 (PASS), 73~102 (FAIL ET=11) |
| YOffset | integer | -94 | Y | Y축 오프셋. 정상 ±300. 실측: -94~143 (PASS), -130~-50 (FAIL ET=11) |
| TOffset | integer | -115 | Y | 회전 오프셋. 정상 ±10,000 |

#### 4.3.2 prs_result 판정 기준 (Rule R02~R07)

| Rule # | 파라미터 | 정상 | WARNING | CRITICAL | 근거 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| R02 | PRS XOffset | \|x\|≤300 | 250<\|x\|≤300 | \|x\|>300 (ET=11 FAIL) | 07 |
| R03 | PRS YOffset | \|y\|≤300 | 250<\|y\|≤300 | \|y\|>300 (ET=11 FAIL) | 07 |
| R04 | PRS TOffset | \|t\|≤10,000 | 8,000<\|t\|≤10,000 | \|t\|>10,000 | 07 |
| R05 | PRS ET=30 발생률 | <1% | 1~3% | >3% → CAM_TIMEOUT | 11 |
| R06 | PRS Pass율 | ≥95% | 90~95% | <90% | 07 |
| R07 | PRS ET=11 동시 슬롯 | 0개 | 1~2개 | ≥3개 동시 (정렬 이상) | 07 |

#### 4.3.3 side_result[] — SIDE 측면 슬롯 (CameraId=5, Side1+Side2 동시)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| ZAxisNum | integer (0~7) | 0 | Y | 슬롯 번호 |
| InspectionResult | integer (0/1) | 1 | Y | 1=PASS / 0=FAIL |
| ErrorType | integer | 1 | Y | 오류 코드 (§4.7) |
| XOffset | integer | 0 | Y | 항상 0 — SIDE는 외관 판정만, 위치 보정 없음 (실측 전수 확인) |
| YOffset | integer | 0 | Y | 항상 0 |
| TOffset | integer | 0 | Y | 항상 0 |

#### 4.3.4 side_result 판정 기준 (Rule R08~R12)

| Rule # | 파라미터 | 정상 | WARNING | CRITICAL | 근거 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| R08 | SIDE Pass율 (Carsem_3X3) | ≥96% | 90~96% | <90% | 05,06,08 |
| R09 | SIDE ET=52 비율 | <5% | 5~50% | >50% (Teaching 미완성) | 05 |
| R10 | SIDE ET=52 연속 Request | 0건 | 5~9건 | ≥10건 연속 | 15 |
| R11 | SIDE ET=12 발생 | 없음 | 산발 발생 | 전 슬롯 ET=12 | 06 |
| R12 | SIDE ET=30 연속 | 0건 | 1~2건 | ≥3건 → CAM_TIMEOUT | 11 |

#### 4.3.5 inspection_detail 예시

```json
// PASS — prs_result (real log 22nd)
"inspection_detail": { 
  "prs_result": [
    {"ZAxisNum":0,"InspectionResult":1,"ErrorType":1,"XOffset":-4,"YOffset":-47,"TOffset":51},
    {"ZAxisNum":5,"InspectionResult":1,"ErrorType":1,"XOffset":121,"YOffset":-94,"TOffset":-115},
    ... // ZAxisNum 0~7 total 8 slots
  ], 
  "side_result": [
    {"ZAxisNum":0,"InspectionResult":1,"ErrorType":1,"XOffset":0,"YOffset":0,"TOffset":0},
    ... // XOffset/YOffset/TOffset always 0
  ]
}

// FAIL — side_result ET=52 all (27th, 1,253 cases)
"side_result": [
  {"ZAxisNum":0,"InspectionResult":0,"ErrorType":52,"XOffset":0,"YOffset":0,"TOffset":0},
  ... // all ZAxisNum 0~7 are ET=52
]

// FAIL — prs_result ET=11 mixed (21st, 268 cases)
"prs_result": [
  {"ZAxisNum":0,"InspectionResult":0,"ErrorType":11,"XOffset":73,"YOffset":-130,"TOffset":166},
  {"ZAxisNum":4,"InspectionResult":0,"ErrorType":11,"XOffset":102,"YOffset":-50,"TOffset":143},
  {"ZAxisNum":2,"InspectionResult":1,"ErrorType":1,"XOffset":28,"YOffset":-40,"TOffset":88},
  ...
]
```

### 4.4 detail 그룹 서브오브젝트 (캡스톤 AI / Oracle 분석용)
8종 로그 미수집 필드 — 캡스톤 Rule-based / AI 분석 목적으로 유지. 실장비 연동 시 DS 측 파이프라인 확인 필요

**geometric — 외형 치수 및 정렬**

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| dimension_w_mm | float | 10.02 (PASS) / 10.01 (ET=52) | N | 패키지 가로 치수 (mm) |
| dimension_l_mm | float | 10.01 (PASS) / 9.99 (ET=52) | N | 패키지 세로 치수 (mm) |
| dimension_h_mm | float | 1.45 (PASS) / 1.46 (ET=52) | N | 패키지 두께 Z-height (mm) |
| x_offset_um | float | 121.0 (PASS) / 102.0 (ET=11) | N | X축 중심 오프셋 (μm). PRS XOffset 실측값 반영 |
| y_offset_um | float | -94.0 (PASS) / -130.0 (ET=11) | N | Y축 중심 오프셋 (μm). PRS YOffset 실측값 반영 |
| theta_deg | float | -0.12 (PASS) / -0.55 (ET=11) | N | 회전 각도 편차 (°). 허용 범위 ±0.5° |
| kerf_width_um | float | 52.3 (PASS) / 51.5 (ET=11) | N | 블레이드 절삭 폭 (μm). 마모 간접 지표 |

**bga — BGA 볼 품질**

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| available | boolean | true | N | BGA 검사 활성 여부 |
| ball_count_nominal | integer | 256 | N | 설계 볼 개수 |
| ball_count_actual | integer | 256 (PASS) / 256 (ET=52) | N | 실측 볼 개수. nominal과 차이 시 FAIL |
| ball_diameter_avg_mm | float | 0.498 (PASS) / 0.497 (ET=52) | N | 볼 평균 지름 (mm) |
| coplanarity_mm | float | 0.065 (PASS) / 0.122 (ET=52) | N | 볼 높이 편차 최댓값 (mm). 기준 ≤ 0.1mm |
| pitch_deviation_um | float | 4.2 (PASS) / 6.8 (ET=52) | N | 볼 간격 기준값 대비 편차 (μm) |
| max_ball_offset_um | float | 18.7 (PASS) / 24.3 (ET=52) | N | 패드 중심 대비 최대 볼 위치 편차 (μm) |
| avg_ball_offset_um | float | 6.1 (PASS) / 9.5 (ET=52) | N | 패드 중심 대비 평균 볼 위치 편차 (μm) |

**surface — 표면 결함**

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| foreign_material_size_um | float | 0 (PASS) / 28.5 (ET=52) | N | 이물질 최대 크기 (μm). 0이면 없음. 실측 FAIL: 28.5μm |
| scratch_area_mm2 | float | 0.003 (PASS) / 0.008 (ET=52) | N | 스크래치 면적 (mm²) |
| marking_quality_grade | string | A (PASS) / C (ET=52) | N | 마킹 품질 등급: A(최우수) ~ F(불량). FAIL 케이스: B, C |

### 4.5 singulation — 절삭 품질 (Rule R13~R15)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| chipping_top_um | float | 22.0 (PASS) / 65.2 (FAIL) | N | 상면 칩핑 크기(μm). 정상<40 / WARNING 40~50 / CRITICAL>50 (ET=12) |
| chipping_bottom_um | float | 18.5 (PASS) / 58.1 (FAIL) | N | 하면 칩핑 크기(μm). 정상<35 / WARNING 35~45 / CRITICAL>45 |
| burr_height_um | float | 3.2 (PASS) / 9.8 (FAIL) | N | 버 높이(μm). 정상<5 / WARNING 5~8 / CRITICAL>8 |

### 4.6 process — 검사 성능 메타 (Rule R22, R37)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| inspection_duration_ms | integer | 1200 | Y | 순수 검사 시간(ms). 정상<1,500 / WARNING 1,500~2,000 / CRITICAL>2,000 |
| takt_time_ms | integer | 1620 | Y | 투입~배출 전체(ms). 실측 ~1,620ms. 정상<2,000 / WARNING 2,000~3,000 |
| algorithm_version | string | v4.3.1 | Y | 비전 알고리즘 버전 |

### 4.7 fail_reason_code 코드 목록 (실측 기반)

| 코드 값 | 관련 슬롯 | ErrorType | 설명 | 실측 발생 |
| :--- | :--- | :--- | :--- | :--- |
| SIDE_VISION_FAIL | side_result | ET=52 | SIDE 알고리즘 종합 실패 (Teaching 미완성) | 1,253건/일 |
| CHIPPING_EXCEED | side_result / singulation | ET=12 | 측면 치수/칩핑 기준 미달 | 680건/일 |
| DIMENSION_OUT_OF_SPEC | prs_result | ET=11 | PRS 픽업 위치 오프셋 허용치 초과 | 268건/일 |
| PRS_X_AXIS_EXCEED | prs_result | ET=15 | X축 과편차 (XOffset>300) | 산발 |
| PRS_MULTI_AXIS_EXCEED | prs_result | ET=17 | 복합 편차 (x/y/t 동시 초과) | 산발 |
| IMAGE_ACQUISITION_FAIL | — | ET=30 | 이미지 미취득 (CAM_TIMEOUT 전조) | <1% |
| FOREIGN_MATERIAL | surface | ET=50 | 이물질 크기 기준 초과 | 산발 |
| BGA_BALL_MISSING | bga | — | 볼 개수 설계값 미달 | — |

### 4.8 ErrorType 코드북 (실측 기반)

| ET | 모듈 | 의미 (추론) | Offset 패턴 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | MAP/PRS/SIDE | 정상 PASS | 실측값/0 | 공통 정상 |
| 3 | MAP/SIDE | 패키지 미감지 | 모두 0 | 연속 3회→CAM_TIMEOUT |
| 11 | PRS | 픽업 오프셋 허용치(±300) 초과 | 실측값 있음 | XOffset 73~102, YOffset -50~-130 |
| 12 | SIDE | 측면 치수/칩핑 기준 미달 | 모두 0 | Teaching 미완성 시 전수. chipping 52μm |
| 15 | PRS | X축 편차 과도 | XOffset 큰 값 | 수평 정렬 불량 |
| 17 | PRS | 복합 편차 (회전+위치) | 다양한 값 | |
| 30 | MAP/PRS/SIDE | 이미지 미취득 / 타임아웃 | 모두 0 | 연속 3회→CAM_TIMEOUT_ERR |
| 50 | SIDE | 측면 이물질·오염 | 모두 0 | |
| 52 | SIDE | 측면 알고리즘 종합 실패 | 모두 0 | 신규 레시피 전수. 실측 1,253건 |
| 55 | SIDE | Burr 높이 기준 초과 | 모두 0 | |
| 62 | PRS | Carsem_4X6 연관 신종 (미확인) | XO±200/YO±100 | DS P2 확인 필요 |

---

## 5. LOT_END — Lot 완료 이벤트

```text
TOPIC       ds/{equipment_id}/lot
EVENT TYPE  LOT_END  
QoS         QoS 2  
방향        EAP → Broker
```
Lot 완료 직후 1회 발행. QoS 2 수율 유실 방지. LOT_END 수신 시 Oracle 2차 검증 트리거.

### 5.1 페이로드 필드 (COMPLETED / ABORTED 비교)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | 9060db98-... / f3058be0-... | Y | 공통 헤더 |
| event_type | string | LOT_END | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T17:39:13.646Z / 2026-01-28T14:18:38.227Z | Y | 완료 시각 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |
| equipment_status | string | IDLE | Y | 항상 IDLE |
| lot_id | string | LOT-20260122-001 / LOT-20260128-001 | Y | 완료된 Lot ID |
| lot_status | string (enum) | COMPLETED / ABORTED | Y | COMPLETED / ABORTED / ERROR |
| total_units | integer | 2792 / 656 | Y | 전체 단위 수. 실측: 2,792 (349 Strip×8슬롯) |
| pass_count | integer | 2687 / 618 | Y | 합격 수 |
| fail_count | integer | 105 / 38 | Y | 불합격 수 |
| yield_pct | float | 96.2 / 94.2 | Y | 수율(%). 실측: COMPLETED=96.2% / ABORTED=94.2% |
| lot_duration_sec | integer | 4923 / 2918 | Y | 소요시간(초). 실측: 82분(정상) / 49분(ABORTED) |

### 5.2 수율 판정 기준 (Rule R23)

| 수율 구간 | 판정 | 조치 |
| :--- | :--- | :--- |
| ≥ 98% | EXCELLENT | 정상 양산. 기록 보존 |
| 95% ~ 98% | NORMAL | 정상. Carsem_3X3 기준 96.2% 실측 |
| 90% ~ 95% | WARNING | Oracle 1차 검증 트리거. 불량 패턴 분석 |
| 80% ~ 90% | MARGINAL | Oracle MARGINAL 판정. 생산 중단 검토 |
| < 80% | CRITICAL | 즉시 생산 중단. 현장 점검 + 레시피 재확인 |

### 5.3 Lot 소요시간 판정 기준 (Rule R24)

| 소요시간 | 판정 | 조치 |
| :--- | :--- | :--- |
| < 24,000s (400분) | NORMAL | 정상. 실측 최대 370분 (22,200s) |
| ≥ 24,000s | LOT_END 미수신 의심 | CRITICAL — LOT_END_MISSING 알람 발행 |

### 5.4 Lot 누적 불균형 판정 (Rule R25)

| LOT Start/End 차이 | 판정 | 조치 |
| :--- | :--- | :--- |
| 0 차이 | NORMAL | 정상 |
| 1~3 차이 누적 | WARNING | AggregateException 여부 확인 |
| ≥ 5 차이 누적 | CRITICAL | EAP 재시작 + 수동 LOT_END 처리 |

---

## 6. HW_ALARM — 하드웨어/소프트웨어 알람

```text
TOPIC       ds/{equipment_id}/alarm
EVENT TYPE  HW_ALARM  
QoS         QoS 2  
방향        EAP → Broker
```
장비 HW/SW 이상 즉시 발행. QoS 2 알람 누락 방지. Will 메시지(EAP_DISCONNECTED) 포함.

### 6.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | 995a93ef-... | Y | 공통 헤더 |
| event_type | string | HW_ALARM | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-23T13:22:52.297Z | Y | 알람 발생 시각 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |
| equipment_status | string | STOP / RUN | Y | CRITICAL이면 STOP. WARNING이면 RUN 가능 |
| alarm_level | string (enum) | CRITICAL | Y | CRITICAL / WARNING / INFO |
| hw_error_code | string | CAM_TIMEOUT_ERR | Y | 오류 코드 (§6.3) |
| hw_error_source | string (enum) | CAMERA | Y | CAMERA / LIGHTING / VISION / PROCESS / MOTION / COOLANT |
| hw_error_detail | string | GrabLinkGrabber failed... | Y | 오류 상세. GVisionWpf 클래스 경로 포함 |
| exception_detail | object | { ... } or null | N | SW 예외 상세 (§6.2). 없으면 null |
| auto_recovery_attempted | boolean | false | Y | 자동 복구 시도 여부 |
| requires_manual_intervention | boolean | true | Y | 수동 처리 필요. true이면 모바일 알림 강조 |

### 6.2 exception_detail 서브오브젝트

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| module | string | GVisionWpf.Cameras.Grabber.GrabLinkGrabber | Y | GVisionWpf 네임스페이스.클래스 |
| exception_type | string | OperationCanceledException | Y | 예외 타입명 (아래 실측 목록 참조) |
| stack_trace_hash | string | b3f1a2c4 | Y | 스택 트레이스 앞 8자리 해시 |

> 실측 예외 타입: OperationCanceledException(카메라) / HOperatorException(HALCON #3142,#4056) / AggregateException(LotController) / WrongValueException(조명) / NullReferenceException(UI)

### 6.3 hw_error_code 코드 목록 (실측 기반)

| hw_error_code | 소스 | 레벨 | 자동복구 | 실측 빈도 | 설명 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| CAM_TIMEOUT_ERR | CAMERA | CRITICAL | N | 17건 | ET=30 연속 3회. GrabLinkGrabber. OperationCanceledException |
| WRITE_FAIL | VISION | CRITICAL | N | 20건/일 | HALCON #3142. 디스크 포화/권한. HOperatorException |
| VISION_SCORE_ERR | VISION | WARNING | N | 3케이스 | 비전 오류 통합 코드 — hw_error_detail로 케이스 구분 (§6.4) |
| LIGHT_PWR_LOW | LIGHTING | WARNING | Y | 산발 | WrongValueException. CameraIlluminator.SetBrightness |
| EAP_DISCONNECTED | PROCESS | CRITICAL | N | 6회 크래시 | Will 메시지 전용. AggregateException 후 크래시 패턴 |
| STAGE_HOME_FAIL | MOTION | CRITICAL | N | 미발생(예약) | X축 홈 복귀 3회 실패 |
| WATER_FLOW_LOW | COOLANT | WARNING | Y | 미발생(예약) | 절삭수 유량 1.5 lpm 미만 |

### 6.4 VISION_SCORE_ERR 케이스 구분 (hw_error_detail 기준)

| 케이스 | hw_error_detail 키워드 | exception_type | 실측 | 파일 |
| :--- | :--- | :--- | :--- | :--- |
| Teaching 이미지 NULL 객체 | HALCON error #4056 / NULL in smallest_rectangle2 | HOperatorException | 23일 16건 연속 | 13 |
| 신규 레시피 Teaching 미완성 | SIDE ET=52 fail rate exceeded 50% threshold | null | Carsem_4X6 Pass 48% | 15 |
| LOT 시작 실패 / LOT_END 미발행 | UnobservedTaskException in LotController.StartNewLot | AggregateException | 26~29일 41건 | 16 |

### 6.5 페이로드 예시

```json
// CAM_TIMEOUT_ERR
{ 
  "hw_error_code": "CAM_TIMEOUT_ERR", 
  "alarm_level": "CRITICAL",
  "hw_error_source": "CAMERA", 
  "equipment_status": "STOP",
  "hw_error_detail": "GrabLinkGrabber failed to dequeue image within timeout.",
  "exception_detail": { 
    "module": "GVisionWpf.Cameras.Grabber.GrabLinkGrabber",
    "exception_type": "OperationCanceledException", 
    "stack_trace_hash": "b3f1a2c4" 
  },
  "auto_recovery_attempted": false, 
  "requires_manual_intervention": true 
}

// VISION_SCORE_ERR (Teaching incomplete)
{ 
  "hw_error_code": "VISION_SCORE_ERR", 
  "alarm_level": "WARNING",
  "equipment_status": "RUN",
  "hw_error_detail": "SIDE ET=52 fail rate exceeded 50% threshold (Pass: 48.0%).",
  "exception_detail": null, 
  "auto_recovery_attempted": false 
}

// EAP_DISCONNECTED (Will message)
{ 
  "hw_error_code": "EAP_DISCONNECTED", 
  "alarm_level": "CRITICAL",
  "equipment_status": "STOP",
  "hw_error_detail": "EAP process terminated unexpectedly.",
  "exception_detail": null, 
  "requires_manual_intervention": true 
}
```

---

## 7. RECIPE_CHANGED — 레시피 전환

```text
TOPIC       ds/{equipment_id}/recipe
EVENT TYPE  RECIPE_CHANGED  
QoS         QoS 2  
방향        EAP → Broker
```
레시피 변경 즉시 발행. QoS 2 이력 유실 방지. 신규 레시피 투입 시 Oracle 독립 학습 시작.

### 7.1 페이로드 필드 (정상 / 신규 비교)

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | 62e13776-... / 3a646f6f-... | Y | 공통 헤더 |
| event_type | string | RECIPE_CHANGED | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T16:16:19.033Z / 2026-01-16T11:01:28.600Z | Y | 변경 시각 |
| equipment_id | string | DS-VIS-001 | Y | 장비 고유 ID |
| equipment_status | string | IDLE | Y | 항상 IDLE |
| previous_recipe_id | string | ATC_1X1 / Carsem_3X3 | Y | 변경 전 Recipe ID |
| previous_recipe_version | string | v1.0 | Y | 변경 전 버전 |
| new_recipe_id | string | Carsem_3X3 / Carsem_4X6 | Y | 변경 후 Recipe ID. 이름형 또는 숫자형(446275) |
| new_recipe_version | string | v1.0 | Y | 변경 후 버전 |
| changed_by | string | ENG-KIM | Y | 엔지니어 ID 또는 MES_AUTO |

### 7.2 실측 레시피 목록 (Carsem 현장 기반)

| recipe_id | 형식 | 최초 등장 | 용도 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| Carsem_3X3 | 이름형 | 01-14 | 주력 양산 | 정상 수율 96.2%. MAP 1,216건/Strip. 28 LOT 기반 EWMA 학습 |
| ATC_1X1 | 이름형 | 01-14 | 교정/검증 | Sequence=9999 더미 스트립 사용. 실제 양산 아님 |
| Carsem_4X6 | 이름형 | 01-16 | 신규 양산 (Teaching 미완성) | ET=52 전수 실패 1,253건 유발. 8 LOT 기반 EWMA. 수율 68%대 |
| 446275 | 숫자형 | 01-23 | 미확인 | EMAP 181개 이상치 (정상 46개의 3.93배). DS 측 확인 필요 (P1) |
| 640022 | 숫자형 | 01-23 | 미확인 | AggregateException 집중 발생 구간과 연관. DS 측 확인 필요 |
| Carsem_4X6 training | 이름+용도형 | 01-16 | Teaching 전용 추정 | 레시피명에 'training' 포함. 양산 레시피와 별도 관리 여부 확인 |

> 숫자형 recipe_id(446275, 640022) 수신 시 Rule R31 트리거 → DS 측 확인 알림 자동 발행

### 7.3 레시피 전환 판정 기준

| 구분 | 판정 조건 | 조치 |
| :--- | :--- | :--- |
| F-1 정상 전환 | Historian에 이력 있는 recipe_id | 즉시 양산 가능. Oracle 기존 EWMA 이력 로드 |
| F-2 신규 투입 | Historian에 이력 없는 recipe_id | E-5 Teaching 모니터링 50 Strip 자동 시작. Oracle 독립 학습 시작 |
| 숫자형 ID 경보 | recipe_id가 숫자형 (446275, 640022 등) | 정체 미확인 → DS 측 확인 알림 발행 |
| EMAP 이상치 | EMAP 크기 > 100개 (정상 46개 기준) | 레시피 설정 오류 의심 → 검토 후 양산 |
| 비정상 전환 | equipment_status ≠ IDLE | 비정상 전환 경보. LOT 진행 중 전환 불가 |

---

## 8. CONTROL_CMD — 원격 제어 명령 [Subscribe 전용]

```text
TOPIC       ds/{equipment_id}/control
EVENT TYPE  CONTROL_CMD  
QoS         QoS 2  
방향        Broker → EAP
```
MES 서버 또는 모바일 앱 명령 수신. 모바일은 EMERGENCY_STOP·STATUS_QUERY만 허용. MQTT ACL 필수.

### 8.1 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | 7f3a1200-ctrl-4abc-b100-000000000010 | Y | 명령 고유 식별자 |
| event_type | string | CONTROL_CMD | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T10:30:00.000Z | Y | 명령 발행 시각 |
| command | string (enum) | EMERGENCY_STOP | Y | 제어 명령 코드 (§8.2) |
| issued_by | string | MOBILE_APP | Y | MES_SERVER / MOBILE_APP / OPERATOR |
| reason | string | Yield drop alert | N | 명령 사유. 감사 로그용 |
| target_lot_id | string | LOT-20260122-001 | N | 특정 Lot 대상 명령. 없으면 null |

### 8.2 command 코드 목록

| 명령 코드 | 발행 주체 | 설명 | 모바일 허용 |
| :--- | :--- | :--- | :--- |
| EMERGENCY_STOP | MES / 모바일 | 즉시 장비 정지. 진행 중 Lot 중단 | Y (긴급) |
| ALARM_CLEAR | MES | 알람 해제 및 복구 시도 | N |
| RECIPE_LOAD | MES | 지정 Recipe 로드 | N |
| LOT_ABORT | MES | 현재 Lot 강제 종료. LOT_END 유발 | N |
| STATUS_QUERY | MES / 모바일 | 즉시 STATUS_UPDATE 발행 요청 | Y (읽기) |

---

## 9. ORACLE_ANALYSIS — Oracle 2차 검증 결과

```text
TOPIC       ds/{equipment_id}/oracle
EVENT TYPE  ORACLE_ANALYSIS  
QoS         QoS 2  
방향        Oracle 서버 → Broker
```
LOT_END 수신 후 비동기 2차 검증. EWMA+MAD / Isolation Forest. 레시피별 독립 학습. 오퍼레이터 승인 구조.

### 9.1 1차/2차 검증 비동기 분리

```text
[1st — Real-time: Rule-based]
INSPECTION_RESULT -> Rule DB lookup -> instant judgment -> mobile (<100ms)

[2nd — Async: Oracle]
LOT_END -> Oracle server
  -> Query Historian TSDB for LOT INSPECTION_RESULT
  -> Dynamic threshold calc per recipe (EWMA+MAD, N>=5)
  -> Multi-metric anomaly detection (Isolation Forest, N>=10)
  -> Publish ORACLE_ANALYSIS (ds/{eq}/oracle, QoS 2)
  -> Mobile app / MES receive
Operator approval -> Rule DB update -> new threshold next cycle
```

### 9.2 레시피별 독립 학습 구조

| LOT 누적 | 활성 모델 | 임계값 기준 | 비고 |
| :--- | :--- | :--- | :--- |
| 0~4 LOT | 오퍼레이터 시딩값 | 엔지니어 입력 초기값 | 셋업 구간 오염 방지 |
| 5~9 LOT | EWMA+MAD | 실측 데이터 동적 계산 | 레시피별 독립 |
| 10+ LOT | EWMA+MAD + Isolation Forest | 복합 지표 이상 감지 추가 | 이상도 점수 0~1 병행 |

### 9.3 판정 3단계 구조

| 등급 | EWMA 조건 | Isolation Forest | 액션 |
| :--- | :--- | :--- | :--- |
| NORMAL | μ±2σ 이내 | 점수<0.5 | LOT 보고서 생성. 경보 없음 |
| WARNING | 2σ~3σ 초과 | 점수 0.5~0.85 | 보고서 + 모바일 주의 알림 + 오퍼레이터 확인 |
| DANGER | 3σ 초과 | 점수>0.85 | 즉시 경보 + 작업 중단 권고 |

### 9.4 페이로드 필드

| 필드명 | 타입 | 예시 값 | 필수 | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| message_id | string (UUID) | a1b2c3d4-... | Y | 공통 헤더 |
| event_type | string | ORACLE_ANALYSIS | Y | 고정값 |
| timestamp | string (ISO 8601) | 2026-01-22T17:40:15.123Z | Y | 분석 완료 시각 |
| equipment_id | string | DS-VIS-001 | Y | 장비 ID |
| lot_id | string | LOT-20260122-001 | Y | 분석 대상 Lot ID |
| recipe_id | string | Carsem_3X3 | Y | 레시피 ID |
| judgment | string (enum) | NORMAL | Y | NORMAL / WARNING / DANGER |
| yield_status | object | { actual, dynamic_threshold, lot_basis } | Y | 수율 분석 결과 |
| ai_comment | string | 28 LOT basis: range [94.9%-98.7%]. Stable. | Y | AI 자연어 요약 코멘트 |
| threshold_proposal | object | { metric, current, proposed, basis } or null | N | 임계값 변경 제안 |

### 9.5 페이로드 예시

```json
// NORMAL (Carsem_3X3, 28 LOT basis)
{ 
  "event_type": "ORACLE_ANALYSIS", 
  "judgment": "NORMAL",
  "recipe_id": "Carsem_3X3",
  "yield_status": { 
    "actual": 96.2,
    "dynamic_threshold": { "normal_min": 94.9, "normal_max": 98.7 },
    "lot_basis": 28 
  },
  "ai_comment": "28 LOT basis: range [94.9%-98.7%]. Stable.",
  "threshold_proposal": null 
}

// WARNING + threshold proposal (Carsem_4X6, 8 LOT)
{ 
  "judgment": "WARNING",
  "yield_status": { 
    "actual": 68.5,
    "dynamic_threshold": { "warning_min": 65.0, "warning_max": 70.0 },
    "lot_basis": 8 
  },
  "ai_comment": "8 LOT analysis: range [65-72%]. 68.5% near boundary.",
  "threshold_proposal": {
    "metric": "yield_pct", "current_warning": 90.0, "proposed_warning": 65.0,
    "basis": "8 LOT EWMA mean=68.2%, std=4.1%" 
  } 
}
```

---

## 10. 이벤트 흐름 시나리오 (v3.3)

| 순서 | 이벤트 타입 | QoS | status | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | HEARTBEAT | 1 | — | 3초 주기. 온라인 확인 (실측 확인) |
| 2 | RECIPE_CHANGED | 2 | IDLE | Lot 투입 전 Recipe 확정 |
| 2a | HW_ALARM (E-5 Teaching) | 2 | RUN | 신규 레시피 시 Teaching 모니터링 50 Strip 자동 시작 |
| 3 | STATUS_UPDATE | 1 | RUN | Lot 시작. 6초 주기 반복 |
| 4~N | INSPECTION_RESULT | 1 | RUN | Unit 완료 시마다 (실측 2,792회). 모바일 드롭 필터 적용 |
| N+1 | LOT_END | 2 | IDLE | Lot 수율 집계. Oracle 2차 검증 트리거 |
| N+2 | ORACLE_ANALYSIS | 2 | — | Oracle 서버 비동기 발행. 보고서 + 임계값 제안 |
| * | HW_ALARM | 2 | * | 이상 발생 즉시. VISION_SCORE_ERR 3케이스 포함. Will 포함 |
| * | CONTROL_CMD | 2 | * | MES/모바일 명령 수신 (Sub). ACL 필수 |

---

## 11. Rule-based 종합 판정 기준표 (38개)
Carsem 실가동 로그 14일 분석 기반. Oracle 서버 1차 검증 if-else 구현에 직접 사용.

| Rule # | 파라미터 | 정상 | WARNING | CRITICAL | 근거 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| R01 | Heartbeat 간격 | ≤9s | 9~30s | >30s | 01 |
| R02 | PRS XOffset | \|x\|≤300 | 250<\|x\|≤300 | \|x\|>300 (ET=11) | 07 |
| R03 | PRS YOffset | \|y\|≤300 | 250<\|y\|≤300 | \|y\|>300 (ET=11) | 07 |
| R04 | PRS TOffset | \|t\|≤10,000 | 8,000<\|t\|≤10,000 | \|t\|>10,000 | 07 |
| R05 | PRS ET=30 발생률 | <1% | 1~3% | >3% →CAM_TIMEOUT | 11 |
| R06 | PRS Pass율 | ≥95% | 90~95% | <90% | 07 |
| R07 | PRS ET=11 동시 슬롯 | 0개 | 1~2개 | ≥3개 동시 | 07 |
| R08 | SIDE Pass율 | ≥96% | 90~96% | <90% | 05,06,08 |
| R09 | SIDE ET=52 비율 | <5% | 5~50% | >50% Teaching의심 | 05 |
| R10 | SIDE ET=52 연속 | 0건 | 5~9건 | ≥10건 연속 | 15 |
| R11 | SIDE ET=12 발생 | 없음 | 산발 | 전 슬롯 ET=12 | 06 |
| R12 | SIDE ET=30 연속 | 0건 | 1~2건 | ≥3건 | 11 |
| R13 | chipping_top_um | <40μm | 40~50μm | >50μm (ET=12) | 06,08 |
| R14 | chipping_bottom_um | <35μm | 35~45μm | >45μm (ET=12) | 06,08 |
| R15 | burr_height_um | <5μm | 5~8μm | >8μm | 06,08 |
| R22 | takt_time_ms | <2,000ms | 2,000~3,000ms | >3,000ms | 04~08 |
| R37 | inspection_duration_ms | <1,500ms | 1,500~2,000ms | >2,000ms | 04~08 |
| R23 | yield_pct | ≥95% | 90~95% | <90% | 09,10 |
| R24 | lot_duration_sec | <24,000s | - | ≥24,000s LOT_END미수신 | 09,10,16 |
| R25 | LOT Start/End 차이 | 0 | 1~3 누적 | ≥5 누적 | 16 |
| R19 | MAP 응답시간 P95 | <2,500ms | 2,500~3,000ms | >3,000ms | report |
| R20 | MAP 카메라 fps | 8~16fps | 6~8fps | <6fps | report,12 |
| R21 | MapQue 잔여 | 0개 | 15~30개 | >30개 | 12 |
| R26 | CAM_TIMEOUT_ERR/일 | 0건 | 1~2건 | >3건 | 11 |
| R27 | WRITE_FAIL 발생 | 0건 | 1건 이상 | 연속 5건 이상 | 12 |
| R28 | VISION_NULL_OBJ | 0건 | 산발 | 연속 10건 이상 | 13 |
| R29 | LIGHT_PWR_LOW | 0건 | 1건 | 연속 3건 이상 | 14 |
| R30 | 신규레시피 Fail율 | <10% | 10~30% | >30% 지속 | 15,19 |
| R31 | 숫자형 레시피 ID | 문자형 | - | 숫자형(446275) | 20 |
| R32 | EMAP 크기 | ≤100개 | 100~150개 | >150개 | 20 |
| R33 | AggregateException | 0건 | 1~5건/일 | >5건/일 | 16 |
| R34 | EAP_DISCONNECTED | 0건 | 1건/주 | >2건/주 | 17 |
| R35 | 동일레시피 ABORTED | 0회 | 1회 | ≥2회 연속 | 10 |
| R36 | blade_usage_count | <20,000회 | 20,000~25,000회 | >25,000회 | 04~08 |
| R38a | Isolation Forest 점수 | <0.5 | 0.5~0.85 | >0.85 | Oracle |
| R38b | EWMA 이탈 | μ±2σ 이내 | 2σ~3σ 초과 | 3σ 초과 | Oracle |
| R38c | status 비정상 전환 | 정상 패턴 | - | RUN→STOP 무경고 | 02,03 |

---

## 12. 연결 규격 및 운영 정책

| 항목 | v2.0 | v2.1 | v2.2 | v3.0 | v3.3 (최종) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 프로토콜 | MQTT v5.0 | 동일 | 동일 | 동일 | 동일 |
| Timestamp | 밀리초 강제 | 동일 | 동일 | 동일 | 동일 |
| QoS LOT_END | QoS 2 | 동일 | 동일 | 동일 | 동일 |
| QoS RECIPE | QoS 2 | 동일 | 동일 | 동일 | 동일 |
| QoS ORACLE | — | — | — | QoS 2 | 동일 |
| Heartbeat | 별도 토픽(3s) | 동일 | 동일 | 실측 검증 | 동일 |
| STATUS 주기 | 10s | 6s 타협 | 동일 | 동일 | 동일 |
| RESULT Takt | ~3,850ms | 동일 | 동일 | ~1,620ms 실측 | 동일 |
| RESULT 슬롯 | 필드 그룹화 | 드롭 필터 | 동일 | PRS/SIDE 명확화 | inspection_detail PascalCase |
| HW_ALARM 코드 | 7종 | 동일 | Will 명시 | 10종 실측 재정의 | VISION_SCORE_ERR 통합 |
| Rule-based | — | — | — | 38개 추가 | 38개 종합 판정표 |
| Oracle 2차 | — | — | — | 인터페이스 추가 | 레시피별 독립학습 상세 |
| 재연결 | 지수 백오프 | 동일 | 동일 | 동일 | 동일 |

---

## 부록 A. MQTT ACL 설정 가이드

### A.1 mosquitto.conf
```text
allow_anonymous false
password_file /etc/mosquitto/passwd
acl_file /etc/mosquitto/acl
```

### A.2 ACL 파일
```text
# EAP device — Publish only
user eap_vis_001
topic write ds/DS-VIS-001/#
topic read ds/DS-VIS-001/control

# Oracle server — Subscribe LOT_END/RESULT, Publish ORACLE_ANALYSIS
user oracle_server
topic read ds/+/lot
topic read ds/+/result
topic write ds/+/oracle

# MES server — full access
user mes_server
topic readwrite ds/#

# Mobile app — Read + limited Publish (control only)
user mobile_app
topic read ds/#
topic write ds/+/control

# Historian — Read only
user historian
topic read ds/#
```

### A.3 클라이언트별 권한 요약

| 클라이언트 | 계정 ID | Publish 토픽 | Subscribe 토픽 |
| :--- | :--- | :--- | :--- |
| EAP 장비 | eap_vis_{id} | ds/{id}/heartbeat·status·result·lot·alarm·recipe | ds/{id}/control |
| Oracle 서버 | oracle_server | ds/+/oracle | ds/+/lot ds/+/result |
| MES 서버 | mes_server | ds/# (전체) | ds/# (전체) |
| 모바일 앱 | mobile_app | ds/+/control (긴급STOP·STATUS_QUERY만) | ds/# (전체) |
| Historian | historian | 없음 | ds/# (전체) |

### A.4 Will 메시지 고정값

```json
{ 
  "event_type": "HW_ALARM", 
  "equipment_status": "STOP",
  "alarm_level": "CRITICAL", 
  "hw_error_code": "EAP_DISCONNECTED",
  "hw_error_source": "PROCESS",
  "hw_error_detail": "EAP process terminated unexpectedly.",
  "auto_recovery_attempted": false, 
  "requires_manual_intervention": true 
}
```

> 실측: Carsem 14일간 6회 크래시. AggregateException 연속 후 2분 이내 발생 패턴 확인.

---

## 부록 B. Mock 데이터 파일 인덱스 (real_log_based v3.3)

| 파일 | 이벤트 | 분류 | 대표 수치 | 실측 근거 |
| :--- | :--- | :--- | :--- | :--- |
| 01_heartbeat | HEARTBEAT | A-1 | 3초 주기 | info.log HeartBeatResponse |
| 02_status_run | STATUS_UPDATE | B-1 | RUN / Carsem_3X3 / 1528s | info.log LOT Start 16:17:10 |
| 03_status_idle | STATUS_UPDATE | B-2 | IDLE / 6420s | info.log LOT End 17:39:13 |
| 04_inspection_pass | INSPECTION_RESULT | C-1 | PASS / ET=1 전체 | prs.log + side.log 22일 |
| 05_inspection_fail_side_et52 | INSPECTION_RESULT | C-2 | FAIL / ET=52 8/8 / Carsem_4X6 | side.log 27일 1,253건 |
| 06_inspection_fail_side_et12 | INSPECTION_RESULT | C-3 | FAIL / ET=12 8/8 / chipping 52μm | side.log 21일 680건 |
| 07_inspection_fail_prs_offset | INSPECTION_RESULT | C-4 | FAIL / ET=11 3/8 / XOffset 102 | prs.log 21일 268건 |
| 08_inspection_fail_side_mixed | INSPECTION_RESULT | C-2 | FAIL / ET=52+12 혼재 | side.log 27일 Pass 62.3% |
| 09_lot_end_normal | LOT_END | D-1 | COMPLETED / 96.2% / 2792 units | info.log 22일 4,923s |
| 10_lot_end_aborted | LOT_END | D-2 | ABORTED / 94.2% / 656 units | info.log 28일 2,918s |
| 11_alarm_cam_timeout | HW_ALARM | E-1 | CRITICAL / CAM_TIMEOUT_ERR | error.log 17건 |
| 12_alarm_write_image_fail | HW_ALARM | E-2 | CRITICAL / WRITE_FAIL | error.log 27일 20건 연속 |
| 13_alarm_vision_null_object | HW_ALARM | E-3 | WARNING / VISION_SCORE_ERR (NULL) | error.log 23일 16건 |
| 14_alarm_light_param_err | HW_ALARM | E-4 | WARNING / LIGHT_PWR_LOW | error.log 23일 산발 |
| 15_alarm_side_vision_fail | HW_ALARM | E-5 | WARNING / VISION_SCORE_ERR (ET=52) | side.log 27일 Pass 48% |
| 16_alarm_lot_start_fail | HW_ALARM | D-3 | WARNING / VISION_SCORE_ERR (AggEx) | error.log 26~29일 41건 |
| 17_alarm_eap_disconnected | HW_ALARM | A-2 | CRITICAL / EAP_DISCONNECTED / Will | lifecycle.log 6회 크래시 |
| 18_recipe_changed_normal | RECIPE_CHANGED | F-1 | ATC_1X1 → Carsem_3X3 | info.log 22일 16:16:19 |
| 19_recipe_changed_new_4x6 | RECIPE_CHANGED | F-2 | → Carsem_4X6 (신규) | info.log 16일 첫 투입 |
| 20_recipe_changed_446275 | RECIPE_CHANGED | F-2 | → 446275 (EMAP 181개 이상) | info.log 23일 |

---

## 부록 C. 개정 이력

| 버전 | 일자 | 변경 내용 |
| :--- | :--- | :--- |
| v1.0 | 2026-03-29 | 최초 작성. 5종 이벤트 명세 |
| v2.0 | 2026-03-29 | QoS 상향 / Timestamp 밀리초 강제 / HEARTBEAT·CONTROL 신규 |
| v2.1 | 2026-03-29 | STATUS 주기 6s 타협 / MQTT ACL 부록 추가 |
| v2.2 | 2026-03-29 | Will 메시지 페이로드 고정값 명세 |
| v3.0 (팀원) | 2026-04-07 | Carsem 로그 반영 / PRS/SIDE 8슬롯 / HW_ALARM 10종 / Rule 38개 / ORACLE_ANALYSIS |
| v3.2 (재혁) | 2026-04-04 | real_log_based 완전 반영: inspection_detail PascalCase / VISION_SCORE_ERR 통합 / ORACLE_ANALYSIS 상세 |
| v3.3 (최종) | 2026-04-07 | v3.2 주축 + v3.0 흡수 통합: ① inspection_detail PascalCase 구조 유지 ② Rule 38개 종합 판정표 (색상 구분, 구현 직결) ③ 상태 전환 규칙 6케이스 ④ 수율 5단계 / LOT 소요시간 / 레시피 전환 판정 5종 ⑤ exception_detail 실측 예외 타입 목록 ⑥ fail_reason_code 실측 발생 빈도 ⑦ 페이로드 예시 RUN/IDLE 병렬 ⑧ 버전 비교표 v2.0~v3.3 |