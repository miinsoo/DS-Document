# 통합 Mock 데이터 (real_log_based only)

## 📊 데이터 구성

**실제 로그 기반 데이터 20개** (eap_mock_final + EAP_mock_data 통합)

---

## 🔍 통합 내용

### eap_mock_final (실제 로그)
- ✅ PRS/SIDE 실제 Offset 값
- ✅ ErrorType 패턴 (1/11/12/52)
- ✅ 실제 timestamp

### EAP_mock_data (상세 파라미터)
- ✅ geometric
- ✅ bga
- ✅ surface
- ✅ singulation
- ✅ process

### ❌ 제외된 필드
- ❌ **saw_process** (사용하지 않음)

---

## 📁 파일 목록

```
real_log_based/
├── 01_heartbeat.json
├── 02_status_run.json
├── 03_status_idle.json
├── 04_inspection_pass.json
├── 05_inspection_fail_side_et52.json
├── 06_inspection_fail_side_et12.json
├── 07_inspection_fail_prs_offset.json
├── 08_inspection_fail_side_mixed.json
├── 09_lot_end_normal.json
├── 10_lot_end_aborted.json
├── 11_alarm_cam_timeout.json
├── 12_alarm_write_image_fail.json
├── 13_alarm_vision_null_object.json
├── 14_alarm_light_param_err.json
├── 15_alarm_side_vision_fail.json
├── 16_alarm_lot_start_fail.json
├── 17_alarm_eap_disconnected.json
├── 18_recipe_changed_normal.json
├── 19_recipe_changed_new_4x6.json
└── 20_recipe_changed_446275.json
```

---

## 📋 JSON 구조

```json
{
  "message_id": "...",
  "event_type": "INSPECTION_RESULT",
  "timestamp": "...",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "RUN",
  
  "lot_id": "...",
  "overall_result": "PASS | FAIL",
  "fail_count": 0,
  "total_inspected_count": 8,
  
  "inspection_detail": {
    "prs_result": [...],
    "side_result": [...]
  },
  
  "geometric": {...},
  "bga": {...},
  "surface": {...},
  "singulation": {...},
  "process": {...}
  
  // ❌ saw_process 제외
}
```

---

## ✅ 검증 완료

- ✅ 20개 파일 생성
- ✅ saw_process 필드 전부 제거
- ✅ 실제 로그값 유지
- ✅ 상세 파라미터 추가

---

**생성일**: 2026-04-04  
**버전**: v2.0 (saw_process 제외)  
**파일 수**: 20개

---

## 🌐 N:1 다설비 시나리오 (scenarios/)

기존 Mock 01~25는 단일 장비(DS-VIS-001) 기준 데이터입니다. 모바일 N:1 타일 대시보드 검증을 위해 시나리오 파일을 별도 디렉토리에 둡니다.

### 파일 목록

| 파일 | 장비 수 | 시나리오 | 검증 목적 |
| :--- | :--- | :--- | :--- |
| `scenarios/multi_equipment_4x.json` | 4대 | RUN+RUN+IDLE+STOP 혼합 | 타일 정렬, equipment_id 라우팅, 동시 알람, 색상 분기 |

### 사용 방법

시뮬레이터(별도 구현)는 이 파일을 읽어 `equipments[].equipment_id`로 기존 Mock의 `equipment_id` 필드를 치환한 뒤, 각 장비의 토픽 트리(`ds/DS-VIS-001/...`, `ds/DS-VIS-002/...`)에 동시 발행합니다. 시나리오 파일 자체는 발행되지 않으며, 시뮬레이터의 routing 입력으로만 사용됩니다.
