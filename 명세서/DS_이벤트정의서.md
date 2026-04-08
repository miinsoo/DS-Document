# DS 주식회사 EAP 이벤트 정의 및 Rule-based 판정 기준

Ver 1.0 · 2026-04-04 · 대외비  
Carsem 현장 실가동 로그 기반 Mock 데이터 20종 전수 분석

| 분류 | 내용 |
| :--- | :--- |
| 분석 대상 | EAP Mock 데이터 20종 (EAP_mock_data 5종 + eap_mock_final 20종 통합) |
| 장비 정보 | Genesem VELOCE-G7 / GVisionWpf (C# WPF + HALCON + MQTTnet) |
| 현장 정보 | Carsem Inc. / 분석 기간: 2026-01-16 ~ 01-29 (14일) |
| 이벤트 분류 | 5개 대분류 / 15개 소분류 이벤트 정의 |
| Rule 기준 수 | 총 38개 판정 규칙 (CRITICAL 12개 / WARNING 18개 / NORMAL 8개) |

---

## 1. 이벤트 분류 체계
20종 Mock 데이터에서 도출된 이벤트를 5개 대분류·15개 소분류로 정의합니다.

| 대분류 | 소분류 | 이벤트 타입 | 관련 파일 | 심각도 |
| :--- | :--- | :--- | :--- | :--- |
| A. 장비 연결 | A-1. 정상 연결 | HEARTBEAT | 01 | INFO |
| A. 장비 연결 | A-2. 비정상 종료 | HW_ALARM | 17 | CRITICAL |
| B. 장비 상태 | B-1. 검사 진행 | STATUS_UPDATE | 02 | INFO |
| B. 장비 상태 | B-2. 대기 | STATUS_UPDATE | 03 | INFO |
| C. 검사 결과 | C-1. 유닛 정상 | INSPECTION_RESULT | 04 | PASS |
| C. 검사 결과 | C-2. 유닛 불량 (SIDE ET=52) | INSPECTION_RESULT | 05,08 | FAIL |
| C. 검사 결과 | C-3. 유닛 불량 (SIDE ET=12) | INSPECTION_RESULT | 06 | FAIL |
| C. 검사 결과 | C-4. 유닛 불량 (PRS ET=11) | INSPECTION_RESULT | 07 | FAIL |
| D. Lot 관리 | D-1. Lot 정상 완료 | LOT_END | 09 | INFO |
| D. Lot 관리 | D-2. Lot 강제 종료 | LOT_END | 10 | WARNING |
| D. Lot 관리 | D-3. Lot End 누락 | HW_ALARM | 16 | WARNING |
| E. 장비 오류 | E-1. 카메라 타임아웃 | HW_ALARM | 11 | CRITICAL |
| E. 장비 오류 | E-2. 이미지 저장 실패 | HW_ALARM | 12 | CRITICAL |
| E. 장비 오류 | E-3. 비전 NULL 객체 | HW_ALARM | 13 | WARNING |
| E. 장비 오류 | E-4. 조명 파라미터 오류 | HW_ALARM | 14 | WARNING |
| E. 장비 오류 | E-5. Teaching 미완성 경보 | HW_ALARM | 15 | WARNING |
| F. 레시피 | F-1. 일반 레시피 전환 | RECIPE_CHANGED | 18 | INFO |
| F. 레시피 | F-2. 신규 레시피 투입 | RECIPE_CHANGED | 19,20 | WARNING |

---

## 2. 대분류 A — 장비 연결 상태

### A-1. 정상 연결 (HEARTBEAT)

> **A-1. 정상 연결 신호**  
> EAP 프로세스가 살아있음을 3초 주기로 브로커에 알리는 경량 신호. 페이로드 4필드만 포함(message_id / event_type / timestamp / equipment_id). 장비 RUN/IDLE 상태와 무관하게 항상 발행.  
> 근거 파일: `01_heartbeat.json`

**판정 기준**
* **ONLINE:** 최근 9초 이내 HEARTBEAT 수신 (3회 주기 × 3s = 9s)
* **OFFLINE:** 마지막 수신 후 9초 초과 → 모바일 앱 오프라인 배지 전환 + EAP_DISCONNECTED Will 메시지 확인

> 주의: 앱 미종료 상태에서도 장비 비정상일 수 있음 — STATUS_UPDATE 병행 확인 필수 (13번 케이스)

### A-2. EAP 비정상 종료 (EAP_DISCONNECTED)

> **A-2. EAP 프로세스 비정상 종료**  
> EAP 프로세스가 exitApplication 호출 없이 강제 종료될 때 MQTT Broker가 자동 발행하는 Will 메시지. 실측 14일간 6회 발생. AggregateException 연속 후 크래시 패턴이 주원인.  
> 근거 파일: `17_alarm_eap_disconnected.json`

**판정 기준**
* **트리거:** equipment_status=STOP + hw_error_code=EAP_DISCONNECTED
* **조치:** 즉시 현장 확인. 모바일 최우선 알림 (requires_manual_intervention=true)
* **선행 패턴:** AggregateException 알람(16번) 수신 후 2분 이내 발생 가능성 높음
* HEARTBEAT 수신 중에도 발생 가능 → 9s 오프라인 판단 로직과 별도 처리 필요

---

## 3. 대분류 B — 장비 운영 상태

### B-1. 검사 진행 (STATUS_UPDATE / RUN)

> **B-1. Lot 검사 진행 중**  
> Lot 검사가 진행 중인 상태. 6초 주기 발행. 현재 Lot ID / Recipe ID / 운영자 / 가동 시간 포함. 실측: Lot 시작 후 uptime 1,528s 기준.  
> 근거 파일: `02_status_run.json`

### B-2. 대기 (STATUS_UPDATE / IDLE)

> **B-2. Lot 완료 후 대기**  
> Lot 완료(LOT_END) 직후 IDLE로 전환. uptime은 계속 누적(6,420s). lot_id와 recipe_id는 마지막 Lot 값 유지.  
> 근거 파일: `03_status_idle.json`

**상태 전환 규칙**

| 이전 상태 | 이후 상태 | 트리거 이벤트 | 정상 여부 |
| :--- | :--- | :--- | :--- |
| IDLE | RUN | LOT 시작 | 정상 |
| RUN | IDLE | LOT_END (COMPLETED) | 정상 |
| RUN | IDLE | LOT_END (ABORTED) | 비정상 — 원인 조사 |
| RUN | STOP | HW_ALARM (CRITICAL) | 비정상 — 즉시 조치 |
| RUN | RUN | STATUS 계속 발행 | 정상 (6초 주기) |
| * | 없음 | HW_ALARM 후 STOP 지속 | 비정상 — EAP 미복구 |

---

## 4. 대분류 C — 단위 검사 결과 (INSPECTION_RESULT)
8슬롯(ZAxisNum 0~7) 기준. prs_slots(PRS 픽앤소트) 또는 side_slots(SIDE 측면) 배열로 슬롯별 result/error_type/offset 포함. PASS 시 fail_count=0 → 모바일 앱 detail 파싱 스킵(드롭 필터) 의무.

### C-1. 유닛 정상 (PASS)

> **C-1. 유닛 검사 정상 통과**  
> 8슬롯 전체 ET=1(정상). PRS XOffset 범위 -25~121 / YOffset -94~143 (±300 이내). SIDE XOffset/YOffset/TOffset 항상 0. chipping_top=22μm / blade_wear=0.31 (정상 마모 구간).  
> 근거 파일: `04_inspection_result_pass.json`

**판정 기준**
* **PASS 조건:** overall_result=PASS AND fail_count=0
* **PRS 정상:** 모든 슬롯 error_type=1 AND |x_offset| ≤ 300 AND |y_offset| ≤ 300 AND |t_offset| ≤ 10,000
* **SIDE 정상:** 모든 슬롯 error_type=1 (XOffset/YOffset/TOffset은 항상 0, 판정 불필요)
* **예지보전 정상 구간:** blade_wear_index < 0.70 AND chipping_top_um < 40 AND spindle_load_pct < 70

### C-2. 유닛 불량 — SIDE ET=52 (Teaching 미완성)

> **C-2. 유닛 불량: SIDE 알고리즘 전수 실패 (ET=52)**  
> 측면 비전 알고리즘 종합 실패. 8슬롯 전수 ET=52 발생 시 Teaching 미완성 신호. 패키지 자체는 인식(XOffset=0)하지만 알고리즘 판정 탈락. 실측: Carsem_4X6 투입 후 1,253건 동일 패턴 연속 발생.  
> 근거 파일: `05_inspection_fail_side_et52.json`

> **C-2b. 유닛 불량: SIDE ET=52+ET=12 혼재 (과도기)**  
> ET=52(알고리즘 실패)와 ET=12(치수 기준 미달)가 같은 Unit에 혼재. ZA=0~3은 ET=52, ZA=4~7은 ET=12 집중 패턴. 신규 레시피 Teaching 불완전 진행 중 과도기 구간. chipping_top=55μm (기준 50μm 초과).  
> 근거 파일: `08_inspection_fail_side_mixed.json`

**판정 기준**
* **C-2 단순 FAIL:** side_slots 내 error_type=52 슬롯 수 ≥ 1 → fail_reason_code=SIDE_VISION_FAIL
* **C-2 전수 FAIL(CRITICAL):** side_slots 전 슬롯(8개) error_type=52 → Teaching 미완성 강력 의심
* **C-2 Pass율 경보:** SIDE Pass율 < 90% → WARNING / Pass율 < 50% → CRITICAL (Teaching 미완성 확정)
* 연속 10 Request 동안 ET=52 Fail율 > 50% 지속 → RECIPE_TEACHING_INCOMPLETE 알람 발행
* **조치:** 해당 레시피 Teaching 재수행 전 생산 중단 권고

### C-3. 유닛 불량 — SIDE ET=12 (칩핑 기준 초과)

> **C-3. 유닛 불량: 측면 치수/칩핑 기준 미달 (ET=12)**  
> 측면 치수·형상 기준 미달. 전수 발생 시 패키지 치수 설정 미완성 신호. chipping_top=65.2μm (기준 50μm 초과), chipping_bottom=58.1μm (기준 45μm 초과). burr_height=9.8μm. 실측: 21일 Teaching 직후 680건 연속.  
> 근거 파일: `06_inspection_fail_side_et12.json`

**판정 기준**
* **C-3 단순 FAIL:** side_slots 내 error_type=12 슬롯 수 ≥ 1 → fail_reason_code=CHIPPING_EXCEED
* **C-3 전수 FAIL:** 전 슬롯 ET=12 → 패키지 치수 설정 재확인 필요
* **singulation 임계값:** chipping_top_um > 50 OR chipping_bottom_um > 45 OR burr_height_um > 8 → FAIL
* Oracle WARNING 기준: chipping_top_um 40~50 (임박 경고) / burr_height_um 6~8 (주의)

### C-4. 유닛 불량 — PRS ET=11 (픽업 오프셋 초과)

> **C-4. 유닛 불량: 픽업 위치 오프셋 허용치 초과 (ET=11)**  
> 패키지 인식은 성공하나 PRS 픽앤소트 위치 오프셋이 허용치(±300) 초과. 실측: XOffset 범위 10~102, YOffset -50~-130. 3슬롯 FAIL / 5슬롯 PASS 혼재 패턴. blade_wear=0.18 (초기 마모 구간).  
> 근거 파일: `07_inspection_fail_prs_offset.json`

**판정 기준**
* **C-4 단순 FAIL:** prs_slots 내 error_type=11 슬롯 수 ≥ 1 → fail_reason_code=DIMENSION_OUT_OF_SPEC
* **PRS 오프셋 임계값:** |x_offset| > 300 OR |y_offset| > 300 → ET=11 (FAIL)
* Oracle WARNING 기준: 동일 Strip 내 ET=11 슬롯 ≥ 3개 동시 발생 → 정렬 이상 의심
* Oracle MARGINAL 기준: |x_offset| 250~300 OR |y_offset| 250~300 → 임박 경고
* ET=15(X축 과편차) / ET=17(복합 편차): x_offset > 300 절대값 → 즉시 FAIL + 정렬 보정 요구

---

## 5. 대분류 D — Lot 관리 이벤트 (LOT_END)

### D-1. Lot 정상 완료 (COMPLETED)

> **D-1. Lot 정상 완료**  
> 전체 2,792 유닛 검사 완료. 수율 96.2% (실측). 소요 시간 4,923s (약 82분). lot_status=COMPLETED. LOT_END 발행 후 STATUS_UPDATE가 IDLE로 전환됨.  
> 근거 파일: `09_lot_end_normal.json`

**수율 판정 기준**

| 수율 구간 | 판정 | 조치 |
| :--- | :--- | :--- |
| ≥ 98% | EXCELLENT | 정상 양산. 기록 보존 |
| 95% ~ 98% | NORMAL | 정상. Carsem_3X3 기준 96.2% 실측 |
| 90% ~ 95% | WARNING | Oracle 1차 검증 트리거. 불량 패턴 분석 |
| 80% ~ 90% | MARGINAL | Oracle MARGINAL 판정. 생산 중단 검토 |
| < 80% | CRITICAL | 즉시 생산 중단. 현장 점검 + 레시피 재확인 |

### D-2. Lot 강제 종료 (ABORTED)

> **D-2. Lot 강제 종료**  
> AggregateException 연속 발생으로 LOT 강제 종료. 656 유닛 처리 후 중단. 수율 94.2%. 소요 시간 2,918s (약 49분). 실측: 28일 15 Start / 1 End 불균형 중 1 End가 이 케이스.  
> 근거 파일: `10_lot_end_aborted.json`

**판정 기준**
* **트리거:** lot_status=ABORTED
* **조치:** 중단 원인 파악 — HW_ALARM 이력 확인 (16번 LOT_END_MISSING 알람과 연계)
* **재투입 가능 조건:** 중단 원인 해소 + 수율 기준 충족 + 레시피 Teaching 완료 확인
* MES 연동: ABORTED Lot의 partial 데이터는 Historian에 별도 태그 저장 권장

### D-3. Lot End 누락 감지 (LOT_END_MISSING)

> **D-3. LOT_END 미발행 감지**  
> LotController.StartNewLot에서 AggregateException 발생 시 LOT_END가 발행되지 않을 수 있음. 실측: 26일 이후 Start 누적 > End 누적 불균형 심화(28일 15 Start / 1 End). 앱은 종료되지 않아 Heartbeat는 정상 수신.  
> 근거 파일: `16_alarm_lot_end_missing.json`

**판정 기준**
* **트리거 로직:** LOT_START 이벤트 후 (정상 최대 소요시간 + 여유시간) 초과 시 LOT_END 미수신
* 실측 기준: 정상 Lot 최대 370분 → 400분(24,000s) 이내 LOT_END 미수신 시 LOT_END_MISSING 발행
* 보조 감지: HW_ALARM(hw_error_code=LOT_END_MISSING) 수신 즉시 — MES Start/End 카운트 불균형 확인
* **조치:** EAP 재시작 또는 수동 LOT_END 처리 필요

---

## 6. 대분류 E — 장비 오류 (HW_ALARM)

### E-1. 카메라 타임아웃 (CAM_TIMEOUT_ERR)

> **E-1. 카메라 이미지 취득 타임아웃**  
> GrabLinkGrabber가 지정 시간 내 이미지 디큐 실패. CancellationToken 트리거. ET=30 연속 3회 이상 발생 시 트리거. equipment_status=STOP. 실측: error.log에서 17건 확인.  
> 근거 파일: `11_alarm_cam_timeout.json`

**판정 기준**
* **트리거:** hw_error_code=CAM_TIMEOUT_ERR AND alarm_level=CRITICAL
* 선행 패턴: ET=30(이미지 미취득) 연속 3회 → 카메라 타임아웃 예비 감지
* ET=30 발생률 경보: < 1% 정상 / 1~3% WARNING / > 3% CRITICAL
* **조치:** auto_recovery_attempted=false → 수동 조치 필수 (카메라 케이블 / 드라이버 확인)
* MAP fps 연동: fps < 6 동반 시 I/O 포화 또는 카메라 드라이버 문제 의심

### E-2. 이미지 저장 실패 (WRITE_FAIL)

> **E-2. 이미지 파일 저장 실패**  
> HALCON #3142 — write_image 연산자에서 파일 쓰기 실패. 디스크 포화 또는 권한 문제. 실측: 27일 20건 연속 발생, MAP fps 5.3→2.9fps 저하, MapQue 잔여 28개 누적. subscribeUnhandledException 20건과 1:1 매핑.  
> 근거 파일: `12_alarm_write_image_fail.json`

**판정 기준**
* **트리거:** hw_error_code=WRITE_FAIL AND alarm_level=CRITICAL
* 선행 경보: MAP fps < 5.3 (정상 8~16fps의 절반 이하) → 저장 병목 의심
* MapQue 잔여 경보: MapQue 잔여 ≥ 30개 → 즉시 처리 병목 경보
* **조치:** 디스크 여유 공간 확인 / 이미지 저장 경로 권한 확인 / 불필요 이미지 파일 정리
* 연쇄 영향: WRITE_FAIL 지속 시 LOT_END 미발행 가능성 → D-3 로직 함께 모니터링

### E-3. 비전 NULL 객체 (VISION_NULL_OBJ)

> **E-3. Teaching 이미지 미획득 — NULL 객체 참조**  
> HALCON #4056 — smallest_rectangle2 연산자에서 객체 ID NULL. Teaching 이미지 획득 전 검사 실행 시 발생. 앱 미종료(Heartbeat 정상 수신)지만 비전 검사 불가 상태. 실측: 23일 16건 연속 — Teaching 화면 연속 클릭 패턴.  
> 근거 파일: `13_alarm_vision_null_object.json`

**판정 기준**
* **트리거:** hw_error_code=VISION_NULL_OBJ AND alarm_level=WARNING
* 주의: Heartbeat 정상 수신 중에도 장비 비정상 — 모바일 앱에서 별도 상태 표시 필요
* 선행 조건: Teaching 미완료 상태에서 검사 실행 시도 → 레시피 전환 직후 집중 모니터링
* **조치:** 해당 레시피 Teaching 재수행 필수. auto_recovery_attempted=false → 수동 처리
* 연쇄 위험: 반복 발생 시 onDispatcherUnhandledException → 앱 강제 종료 가능

### E-4. 조명 파라미터 오류 (LIGHT_PARAM_ERR)

> **E-4. 조명 파라미터 유효 범위 이탈**  
> CameraIlluminator.SetBrightness에서 WrongValueException. 조명 파라미터 유효 범위 이탈. equipment_status=RUN 유지(검사 중단 없음). auto_recovery_attempted=true — 자동 복구 시도. 실측: SIDE ET=52 급증의 전조 신호 패턴 확인.  
> 근거 파일: `14_alarm_light_param_err.json`

**판정 기준**
* **트리거:** hw_error_code=LIGHT_PARAM_ERR AND alarm_level=WARNING
* 자동 복구: auto_recovery_attempted=true → 일정 시간 후 복구 여부 재확인
* 전조 패턴: LIGHT_PARAM_ERR 발생 후 SIDE Pass율 하락 감지 시 → SIDE 검사 품질 저하 연동
* Oracle 연동: LIGHT_PARAM_ERR 발생 이후 50 Strip 동안 SIDE ET=52 비율 모니터링
* **조치:** 복구 실패(동일 알람 재발) 시 수동 조명 파라미터 재설정 필요

### E-5. Teaching 미완성 경보 (RECIPE_TEACHING_INCOMPLETE)

> **E-5. 신규 레시피 Teaching 미완성 감지**  
> 신규 레시피 투입 후 50 Strip 모니터링 로직 트리거. SIDE ET=52 Fail율 48% (정상 96~98%). 연속 10 Request Fail율 > 50% 지속 시 발행. equipment_status=RUN 유지. 실측: Carsem_4X6 투입 직후 즉시 발생.  
> 근거 파일: `15_alarm_recipe_teaching_incomplete.json`

**판정 기준**
* **트리거 로직:** RECIPE_CHANGED 수신 후 50 Strip(= 400 슬롯) 동안 SIDE ET=52/12 Fail율 모니터링
* WARNING 기준: SIDE Fail율 > 30% 지속 (10 Request 연속)
* CRITICAL 기준: SIDE Fail율 > 50% 지속 (10 Request 연속) → RECIPE_TEACHING_INCOMPLETE 발행
* 정상 종료: 50 Strip 이내 Fail율 < 10% 안정화 시 모니터링 종료
* **조치:** 신규 레시피 Teaching 완료 확인 후 양산 재개. 완료 전 생산 중단 권고

---

## 7. 대분류 F — 레시피 전환 (RECIPE_CHANGED)

### F-1. 일반 레시피 전환

> **F-1. 정상 레시피 전환**  
> 기존에 사용하던 레시피로 전환. equipment_status=IDLE에서만 발생. 실측: ATC_1X1 → Carsem_3X3 전환 (검증된 레시피 간 전환). changed_by=ENG-KIM (수동 전환).  
> 근거 파일: `18_recipe_changed_normal.json`

### F-2. 신규 레시피 투입

> **F-2. 신규 레시피 최초 투입 — Teaching 미완성 위험**  
> 이력 없는 신규 레시피 투입. 실측: Carsem_4X6 투입 후 즉시 ET=52 전수 실패(1,253건). 446275 레시피는 EMAP 181개 이상치(정상 46개의 3.93배) 발생. 신규 레시피 도입 = 자동 Teaching 완료 확인 프로세스 필요.  
> 근거 파일: `19_recipe_changed_new_4x6.json` `20_recipe_changed_446275.json`

**레시피 전환 판정 기준**
* F-1 정상 전환: 기존 이력 있는 레시피 → 즉시 양산 가능
* F-2 신규 투입: Historian에 이력 없는 recipe_id → 자동으로 E-5 Teaching 모니터링 50 Strip 시작
* 레시피 ID 형식 경보: 숫자형 ID(446275, 640022 등) → 정체 미확인 레시피 → DS 측 확인 알림 발행
* EMAP 이상치 감지: EMAP 크기 > 100개 (정상 46개 기준) → 레시피 설정 오류 의심
* equipment_status 검증: RECIPE_CHANGED 수신 시 equipment_status ≠ IDLE → 비정상 전환 경보

---

## 8. Rule-based 종합 판정 기준표
모든 판정 기준을 한 곳에서 참조할 수 있도록 정리합니다.

### 8.1 수치 임계값 — 즉시 판정 (38개 Rule)

| # | 파라미터 | 정상 범위 | WARNING | CRITICAL | 출처 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| R01 | Heartbeat 수신 간격 | ≤ 9s | 9~30s | 30s 초과 | 01 |
| R02 | PRS x_offset | \|x\| ≤ 300 | \|x\| 250~300 | \|x\| > 300 (ET=11) | 07 |
| R03 | PRS y_offset | \|y\| ≤ 300 | \|y\| 250~300 | \|y\| > 300 (ET=11) | 07 |
| R04 | PRS t_offset | \|t\| ≤ 10,000 | \|t\| 8,000~10,000 | \|t\| > 10,000 | 07 |
| R05 | PRS ET=30 발생률 | < 1% | 1~3% | > 3% (CAM_TIMEOUT_ERR) | 11 |
| R06 | PRS Pass율 | ≥ 95% | 90~95% | < 90% | 07 |
| R07 | PRS ET=11 동시 슬롯 수 | 0개 | 1~2개 | ≥ 3개 동시 | 07 |
| R08 | SIDE Pass율 (Carsem_3X3) | ≥ 96% | 90~96% | < 90% | 05,06,08 |
| R09 | SIDE ET=52 비율 | < 5% | 5~50% | > 50% (Teaching 의심) | 05 |
| R10 | SIDE ET=52 연속 Request | 0건 | 5~9건 연속 | ≥ 10건 연속 | 15 |
| R11 | SIDE ET=12 발생 여부 | 없음 | 산발 발생 | 전수(전 슬롯 ET=12) | 06 |
| R12 | SIDE ET=30 연속 발생 | 0건 | 1~2건 | ≥ 3건 연속 | 11 |
| R13 | chipping_top_um | < 40μm | 40~50μm | > 50μm (ET=12) | 06,08 |
| R14 | chipping_bottom_um | < 35μm | 35~45μm | > 45μm (ET=12) | 06,08 |
| R15 | burr_height_um | < 5μm | 5~8μm | > 8μm | 06,08 |
| R16 | blade_wear_index | < 0.70 | 0.70~0.85 | > 0.85 (교체 권고) | 04~08 |
| R17 | spindle_load_pct | < 70% | 70~80% | > 80% | 04~08 |
| R18 | cutting_water_flow_lpm | ≥ 1.5 L/min | 1.2~1.5 | < 1.2 | 04 |
| R19 | MAP 응답시간 P95 | < 2,500ms | 2,500~3,000ms | > 3,000ms | 보고서 |
| R20 | MAP 카메라 fps | 8~16fps | 6~8fps | < 6fps | 보고서 |
| R21 | MapQue 잔여 수 | 0개 | 15~30개 | > 30개 | 12 |
| R22 | takt_time_ms (정상) | < 2,000ms | 2,000~3,000ms | > 3,000ms | 04~08 |
| R23 | yield_pct | ≥ 95% | 90~95% | < 90% | 09,10 |
| R24 | lot_duration_sec | < 24,000s (400분) | — | > 24,000s (LOT_END_MISSING) | 09,10,16 |
| R25 | LOT Start/End 불균형 | 0 차이 | 1~3 차이 누적 | ≥ 5 차이 누적 | 16 |
| R26 | CAM_TIMEOUT_ERR 발생 | 0건/일 | 1~2건/일 | > 3건/일 | 11 |
| R27 | WRITE_FAIL 발생 | 0건 | 1건 이상 | 연속 5건 이상 | 12 |

---

## 9. Oracle 서버 2 차 검증 — 이벤트 연동 인터페이스
Oracle 서버는 LOT_END 이벤트를 수신하여 레시피별 동적 임계값 학습 및 복합 지표 분석을 수행하고, 분석 결과를 ORACLE_ANALYSIS 이벤트로 발행합니다. 1 차 검증(Rule-based)은 실시간 즉시 판정을 담당하고, 2 차 검증(Oracle)은 비동기로 장기 트렌드와 학습 기반 판정을 수행합니다.

### 9.1 입력 이벤트 (MQTT Subscribe)

| 이벤트 타입 | 토픽 | QoS | 용도 |
| :--- | :--- | :--- | :--- |
| LOT_END | ds/{eq}/lot | 2 | 2 차 검증 트리거. 수율, 총 유닛 수, 소요시간 포함 |
| INSPECTION_RESULT | ds/{eq}/result | 1 | Historian TSDB 경유. 유닛별 상세 계측값 참조 |

Oracle 서버는 LOT_END 를 직접 구독하며, INSPECTION_RESULT 는 Historian DB 를 통해 간접 참조합니다. 이는 Oracle 서버가 실시간 처리 부하에서 자유로워 복잡한 분석을 수행할 수 있도록 합니다.

### 9.2 출력 이벤트 (MQTT Publish)

**ORACLE_ANALYSIS — 2 차 검증 분석 결과**

| 필드명 | 타입 | 설명 |
| :--- | :--- | :--- |
| message_id | UUID | 메시지 고유 식별자 |
| event_type | string | ORACLE_ANALYSIS (고정값) |
| timestamp | ISO 8601 | 분석 완료 시각 |
| equipment_id | string | 장비 ID |
| lot_id | string | 분석 대상 LOT ID |
| recipe_id | string | 레시피 ID |
| judgment | enum | NORMAL / WARNING / DANGER |
| yield_status | object | 수율 분석 결과 (actual, threshold, basis) |
| ai_comment | string | AI 분석 코멘트 (자연어 요약) |
| threshold_proposal | object | 임계값 변경 제안 (optional) |

토픽: `ds/{equipment_id}/oracle` | QoS: 2 (정확히 1 회 전달 보장)

### 9.3 이벤트 흐름도

**[실시간 경로] 1 차 검증**
INSPECTION_RESULT → Rule DB 조회 → PASS/WARNING/CRITICAL
→ 모바일 즉시 전달 (반응 속도: <100ms)

**[비동기 경로] 2 차 검증**
LOT_END → Oracle 서버 Subscribe
→ Historian TSDB 에서 해당 LOT 의 INSPECTION_RESULT 조회
→ 레시피별 동적 임계값 계산 (EWMA+MAD)
→ 복합 지표 이상 감지 (Isolation Forest)
→ ORACLE_ANALYSIS 이벤트 발행
→ 모바일 앱 / MES 수신

### 9.4 판정 등급 정의

| 등급 | 조건 | 액션 |
| :--- | :--- | :--- |
| NORMAL | 동적 경계 정상 범위 내 | LOT 보고서 생성, 경보 없음 |
| WARNING | 동적 경계 주의 구간 | LOT 보고서 + 모바일 주의 알림 |
| DANGER | 동적 경계 위험 구간 | 즉시 경보 + 작업 중단 권고 |

동적 경계는 레시피별 LOT 이력을 기반으로 Oracle 서버가 자동 계산합니다. 세부 알고리즘(EWMA, MAD, Isolation Forest 등)은 별도 문서를 참조하십시오.

### 9.5 페이로드 예시

**ORACLE_ANALYSIS 이벤트 예시 (NORMAL 판정):**
```json
{ 
  "message_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", 
  "event_type": "ORACLE_ANALYSIS",
  "timestamp": "2026-01-22T17:40:15.123Z", 
  "equipment_id": "DS-VIS-001", 
  "lot_id": "LOT-20260122-001", 
  "recipe_id": "Carsem_3X3", 
  "judgment": "NORMAL", 
  "yield_status": { 
    "actual": 96.2, 
    "dynamic_threshold": { "normal_min": 94.9, "normal_max": 98.7 },
    "lot_basis": 28 
  }, 
  "ai_comment": "28 LOT 학습 기준 정상 범위 [94.9%, 98.7%] 내. 추세 안정.", 
  "threshold_proposal": null 
}
```

**ORACLE_ANALYSIS 이벤트 예시 (WARNING 판정 + 임계값 제안):**
```json
{ 
  "message_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901", 
  "event_type": "ORACLE_ANALYSIS",
  "timestamp": "2026-01-27T12:30:22.456Z", 
  "equipment_id": "DS-VIS-001", 
  "lot_id": "LOT-20260127-003", 
  "recipe_id": "Carsem_4X6", 
  "judgment": "WARNING", 
  "yield_status": { 
    "actual": 68.5, 
    "dynamic_threshold": { "warning_min": 65.0, "warning_max": 70.0 },
    "lot_basis": 8 
  }, 
  "ai_comment": "8 LOT 분석 결과 정상 범위 [65~72%]. 현재 68.5% 경계 근접.", 
  "threshold_proposal": { 
    "metric": "yield_pct", 
    "current_warning": 90.0, 
    "proposed_warning": 65.0, 
    "basis": "8 LOT 기반 동적 계산 (EWMA 평균 68.2%, 표준편차 4.1%)" 
  } 
}
```

---

## 부록. Mock 데이터 파일 인덱스

| 파일 | 이벤트 타입 | 소분류 | 대표 수치 |
| :--- | :--- | :--- | :--- |
| 01_heartbeat | HEARTBEAT | A-1 | 3 초 주기 |
| 02_status_run | STATUS_UPDATE | B-1 | RUN / Carsem_3X3 / uptime 1528s |
| 03_status_idle | STATUS_UPDATE | B-2 | IDLE / uptime 6420s |
| 04_inspection_result_pass | INSPECTION_RESULT | C-1 | PASS / ET=1 전체 / blade_wear 0.31 |
| 05_inspection_fail_side_et52 | INSPECTION_RESULT | C-2 | FAIL / ET=52 8/8 / Carsem_4X6 |
| 06_inspection_fail_side_et12 | INSPECTION_RESULT | C-3 | FAIL / ET=12 8/8 / chipping 65.2μm |
| 07_inspection_fail_prs_offset | INSPECTION_RESULT | C-4 | FAIL / ET=11 3/8 / x_offset 102 |
| 08_inspection_fail_side_mixed | INSPECTION_RESULT | C-2 | FAIL / ET=52+12 혼재 / 2/8 PASS |
| 09_lot_end_normal | LOT_END | D-1 | COMPLETED / 96.2% / 2792 units |
| 10_lot_end_aborted | LOT_END | D-2 | ABORTED / 94.2% / 656 units |
| 11_alarm_cam_timeout | HW_ALARM | E-1 | CRITICAL / CAM_TIMEOUT_ERR / STOP |
| 12_alarm_write_image_fail | HW_ALARM | E-2 | CRITICAL / WRITE_FAIL / HALCON#3142 |
| 13_alarm_vision_null_object | HW_ALARM | E-3 | WARNING / VISION_NULL_OBJ / HALCON#4056 |
| 14_alarm_light_param_err | HW_ALARM | E-4 | WARNING / LIGHT_PARAM_ERR / 자동복구 |
| 15_alarm_recipe_teaching_inc | HW_ALARM | E-5 | WARNING / RECIPE_TEACHING_INCOMPLETE |
| 16_alarm_lot_end_missing | HW_ALARM | D-3 | WARNING / LOT_END_MISSING / AggEx |
| 17_alarm_eap_disconnected | HW_ALARM | A-2 | CRITICAL / EAP_DISCONNECTED / Will |
| 18_recipe_changed_normal | RECIPE_CHANGED | F-1 | ATC_1X1 → Carsem_3X3 |
| 19_recipe_changed_new_4x6 | RECIPE_CHANGED | F-2 | Carsem_3X3 → Carsem_4X6 (신규) |
| 20_recipe_changed_446275 | RECIPE_CHANGED | F-2 | ATC_1X1 → 446275 (숫자형 ID) |