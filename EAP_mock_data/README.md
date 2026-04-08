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
