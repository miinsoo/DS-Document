# DS 프로젝트 — Carsem 현장 로그 분석 참조 보고서

**장비:** Genesem VELOCE-G7 Saw Singulation  
**현장:** Carsem Inc.  
**소프트웨어:** GVisionWpf (C# WPF, HALCON + MQTTnet)  
**분석 기간:** 2026-01-16 ~ 2026-01-29 (14일, 셋업 구간)  
**장비 대수:** 단일 장비 1대 (날짜별 로그 분리 구조 — 장비 구분 폴더 없음)

> **중요:** 제공된 로그는 셋업(Teaching) 구간 데이터. 실제 양산 시보다 Fail·Exception이 훨씬 많음. (멘토 직접 확인)

---

## 1. 로그 파일 구조

**경로:** `/[로그종류]/2026/01/YYYY-MM-DD.log` — 날짜당 단일 파일

| 로그명 | 로거 클래스 | 주요 내용 |
|--------|-----------|----------|
| `error` | ErrorLogger | 미처리 예외 전체 스택 트레이스 |
| `fatal` | FatalLogger | 치명적 오류 — **전 기간 0건** |
| `info` | 다수 | LOT·Recipe 변경·EMAP·HeartBeat 이벤트 |
| `lifecycle` | LifecycleLogger | 앱 시작/종료/크래시/예외 핸들러 이벤트 |
| `map` | MapLogger | MAP 검사 Request/Response + 카메라 프레임 |
| `prs` | PrsLogger | PRS 검사 Request/Response + 카메라 프레임 |
| `side` | SideLogger | SIDE 검사 Request/Response (Side1+Side2 동시) |
| `strip` | StripLogger | SmartAlign + StripInspection — **Teaching 시에만 기록** |

---

## 2. 레시피 목록 (전 기간 확인, 7종)

| Recipe ID | 형식 | 최초 등장 | 용도 추정 | 비고 |
|-----------|------|----------|----------|------|
| `Carsem_3X3` | 이름 | 01-14 | 주력 양산 | MAP Req당 Response 1,216건 |
| `ATC_1X1` | 이름 | 01-14 | 교정/검증 | PRS Sequence=9999 더미 스트립 |
| `Carsem_4X4` | 이름 | 01-16 | Teaching 테스트 | 단발 등장, 22일 이후 미사용 |
| `Carsem_4X6` | 이름 | 01-16 | 신규 양산 | Teaching 미완성. ET=52/ET=12 전수 실패 원인 |
| `Carsem_4X6 training` | 이름+용도 | 01-16 | 4X6 Teaching 전용 | 레시피명에 "training" 포함 |
| `446275` | 숫자 | 01-23 | 미확인 | EMAP 181개 이상치 연관 |
| `640022` | 숫자 | 01-23 | 미확인 | AggregateException 집중 구간 연관 |

---

## 3. info.log

### 주요 이벤트 패턴

```
[LOT] [Request] LOT Start
[LOT] [Request] LOT End
[Recipe] The Recipe has been changed. Recipe: {recipe_id}
[EMAP] {N} EMAP stored
[Communication] [Response] Vision Status (HeartBeatResponse)
[SmartAlign] [Request] / [Response]
[STRIP] [Request] / [Response]
```

### 날짜별 LOT Start/End 집계

| 날짜 | LOT Start | LOT End | 주요 레시피 | EMAP 크기 |
|------|----------|--------|-----------|---------|
| 01-16 | 3 | 1 | Carsem_4X6 | 46개 (×2) |
| 01-19 | 0 | 0 | — | — |
| 01-20 | 5 | 1 | — | 46개 (×5) |
| 01-21 | 2~3 | 1 | Carsem_3X3 | — |
| 01-22 | 6~8 | 2~3 | ATC_1X1 / Carsem_4X6 | 46개 |
| 01-23 | 7~9 | 2~3 | ATC_1X1 / 446275 / 640022 | 46개 (×9) |
| 01-26 | 6 | **0** | ATC_1X1 / 446275 | — |
| 01-27 | 14 | 4 | ATC_1X1 / 640022 | — |
| 01-28 | 15 | **1** | Carsem_4X6 | 46개 (×14) |
| 01-29 | 9 | 4~5 | ATC_1X1 | **181개 (×3)** |

### LOT 소요시간 실측

- 정상 양산: 40~180분
- 최대: 370분 (01-28)
- 최소: 9.5분 (01-28 단시간 중단/재시작)
- 기준일(01-22): LOT Start 16:17:10 → End 17:39:13 = **4,923초 (82분)**

### EMAP 이상치

- 정상: **46개** (전 기간 대부분)
- 이상치: **181개** (446275 레시피 사용 시, 01-29 3회 연속)
- 정상 대비 3.93배

### HeartBeat 주기

- 정상: 약 3초 간격
- 예외 핸들러 발생 시에도 HeartBeat는 정상 수신 → 앱 살아있지만 장비 비정상 상태 가능

---

## 4. map.log

### 기본 구조

```
[MapLogger] Mapping onFrameArrived
[MapLogger] [MapInspection]: [Request]
  CommonBody = ( ... CameraId : 0  InspectionType : 1 )
[MapLogger] [MapInspection]: [Response]
  CommonBody = ( ... InspectionResult = {0|1}  ErrorType = {ET} ... )
[MapLogger] Cancel Task
[MapLogger] MapQue remain ({N})
```

### 모듈 기본 정보

| 항목 | 값 |
|------|-----|
| CameraId | 0 |
| InspectionType | 1 |
| Thread | 다수 (멀티스레드, 주로 Thread:16~117) |

### Req당 Response 수 (레시피별)

| 최빈값 | 연관 레시피 | 확인 날짜 |
|--------|-----------|---------|
| **1,216건** | Carsem_3X3 | 이전 세션 21~29일 |
| **672건** | ATC_1X1 / Carsem_4X4 / Carsem_4X6 / 446275 | 이번 세션 19~29일 |
| **540건** | Carsem_4X6 / 640022 추정 | 마지막 세션 16·20·28·29일 |
| **15건** | SmartAlign 모드 (InspectionType=43) | 23일 |

### 날짜별 카메라 fps 및 응답시간

| 날짜 | fps | Req당 Resp 최빈 | 응답시간 avg | 응답시간 이상치 |
|------|-----|--------------|------------|-------------|
| 01-16 | 13.9fps | 540건 | 334ms | — |
| 01-19 | 2.9fps | Req 없음 | — | — |
| 01-20 | 3.5~6.0fps | 540건 / 672건 | 688~851ms | 1,167ms |
| 01-21 | 13.2~13.5fps | 672건 | 1,046ms | 1,609ms |
| 01-22 | 11.8fps | 672건 | 796ms | 1,345ms |
| 01-23 | 12.2fps | **15건 (SmartAlign)** | 200ms | 598ms |
| 01-26 | 2.9fps | Req 없음 | — | — |
| 01-27 | 6.1~14.5fps | 672건 | 1,166ms | 2,004ms |
| 01-28 | 3.5~3.6fps | 540건 | 773~6,191ms | **14,278ms** |
| 01-29 | 3.6~3.8fps | 540건 | 727~1,345ms | 2,159ms |

### MapQue 잔여 이상치

- 01-21: 최대 **44개** (교정 구간)
- 01-22: 최대 1개
- 01-23: 최대 1개
- 01-28: 최대 **36개** (AggregateException 15건 연동)

### 26일 특이 패턴

- 729줄 전체가 `onFrameArrived`만 기록
- Request/Response 완전 없음
- 3개 세션으로 분리된 촬영 구간 (16:08~16:09, 16:12~16:16, 16:22~16:25)
- 16:06~16:29까지 반복적 비활성 구간 발생

---

## 5. prs.log

### 기본 구조

```
[PrsLogger] PRS onFrameArrived
[PrsLogger] [PrsInspection]: [Request]
  CommonBody = ( ... CameraId : 1  InspectionType : 10 )
  StripBarcode : {N}  Sequence : {9999|0|1}  ZAxisNum : {0~7}  HasDevice : {0|1}
  XPickPosition / YPickPosition / ZPickPosition
[PrsLogger] [PrsInspection]: [Response]
  InspectionResult = {0|1}  ErrorType = {ET}
  XOffset = {val}  YOffset = {val}  TOffset = {val}
```

### 모듈 기본 정보

| 항목 | 값 |
|------|-----|
| CameraId | 1 |
| InspectionType | 10 |
| Req당 Response 수 | **8건 고정** (ZAxisNum 0~7) |
| Thread | `.NET Long Running Task` (Response 전용) |

### Sequence 의미

| Sequence 값 | 의미 |
|------------|------|
| 9999 | Teaching 더미 스트립 (ATC_1X1 교정 시) |
| 0 | 초기화/대기 |
| 1 이상 | 실제 LOT 검사 |

### Offset 정상 범위 (실측)

| 지표 | 정상 범위 | ET=11 발생 기준 |
|------|---------|--------------|
| XOffset | ±300 이내 | 초과 시 ET=11 |
| YOffset | ±300 이내 | 초과 시 ET=11 |
| TOffset | ±10,000 이내 | 초과 시 ET=11 |

### 날짜별 집계

| 날짜 | Req | Resp | Req당 Resp | 비고 |
|------|-----|------|----------|------|
| 01-16 | 69 | 552 | 8건 고정 | Response 상세 필드 없음 (헤더만) |
| 01-19 | 0 | 0 | — | — |
| 01-20 | 84~176 | 672~1,408 | 8건 고정 | |
| 01-21 | 85 | 680 | 8건 고정 | ET=11 268건 발생 |
| 01-22 | 0 / 28 / 349 | 0 / 224 / 2,792 | 8건 고정 | 22일 185,523줄은 onFrameArrived만 (fps 21.3) |
| 01-23 | 190~347 | 1,520~2,773 | 8건 고정 | |
| 01-26 | 0 | 0 | — | — |
| 01-27 | 0~297 | 0~2,376 | 8건 고정 | |
| 01-28 | 82~146 | 656~1,168 | 8건 고정 | |
| 01-29 | 409~555 | 3,272~4,440 | 8건 고정 | |

### 카메라 fps

- 일반: 20~40fps
- 고속(픽앤소트 집중): ~120fps

### PRS 22일 특이: 185,523줄 onFrameArrived

- 15:31~17:57 (145분) 동안 Request 0건, onFrameArrived만 21.3fps로 촬영
- 검사 없는 카메라 활성 대기 상태

---

## 6. side.log

### 기본 구조

```
[SideLogger] Side1 onFrameArrived
[SideLogger] Side2 onFrameArrived
[SideLogger] [SideInspection]: [Request] Side
  CommonBody = ( ... CameraId : 5  InspectionType : 50 )
  ZAxisNum : {0~7}  X1Orx2 : {0|1}
[SideLogger] [SideInspection]: [Response] SIDEVISION
  InspectionResult = {0|1}  ErrorType = {ET}
  XOffset = 0  YOffset = 0  TOffset = 0
```

### 모듈 기본 정보

| 항목 | 값 |
|------|-----|
| CameraId | 5 |
| InspectionType | 50 |
| 카메라 구성 | Side1 + Side2 동시 촬영 (하드웨어 트리거) |
| Side1·Side2 동기화 | Δ = 0~1ms |
| Req당 Response 수 | **8건 고정** (ZAxisNum 0~7) |
| XOffset / YOffset / TOffset | **항상 0** (외관 판정만, 위치보정 없음) |

### 날짜별 Pass율 및 ET 분포

| 날짜 | Req | Resp | fps | Pass율 | 주요 ET 분포 |
|------|-----|------|-----|--------|------------|
| 01-16 | 69 | 552 | 7.4fps | 97.3% | ET=1: 537, ET=30: 12, ET=12: 1, ET=11: 2 |
| 01-20 | 5~87 | 40~696 | 4.4~5.1fps | 0~91.8% | ET=3: 18~46, ET=30: 2~21, ET=12: 1~9 |
| 01-21 | 85 | 680 | 4.9fps | **0%** | ET=12: 672건 전수, ET=30: 8건 |
| 01-22 | 28 | 223~876 | 12.7fps | 93.7% | ET=1: 209, ET=30: 6, ET=12: 7, ET=3: 1 |
| 01-23 | 190~341 | 1,520~2,728 | 4.1~9.6fps | 69.6~89.5% | ET=52: 266~448건, ET=30: 14~16 |
| 01-26 | 0 | 0 | 2.9fps | — | — |
| 01-27 | 297~303 | 2,376~2,424 | 5.0fps | 62.3% | ET=52: 767, ET=12: 109, ET=30: 39 |
| 01-28 | 152 | 1,216 | 4.8fps | 99.7% | ET=1: 1,212, ET=30: 4 |
| 01-29 | 303~555 | 2,424~4,440 | 4.8fps | 100% | ET=1: 전수 |

### 21일 ET=12 전수 실패 상세

- 발생 시각: 15:51 (Teaching 직후 구간)
- ZAxisNum 0~7 전수 균등 분포 (83~85건씩)
- 원인: 패키지 치수 설정 미완성 (Teaching 미완료 상태)

### 27일 ET=52/ET=12 혼재 상세

| ZAxisNum | ET=1 (Pass) | ET=12 | ET=30 | ET=52 |
|----------|------------|-------|-------|-------|
| ZA=0 | 193 | 2 | 0 | 108 |
| ZA=1 | 194 | 1 | 2 | 106 |
| ZA=2 | 194 | 1 | 2 | 106 |
| ZA=3 | 191 | 0 | 5 | 107 |
| ZA=4 | 181 | **33** | 5 | 84 |
| ZA=5 | 183 | **25** | 6 | 89 |
| ZA=6 | 182 | **29** | 9 | 83 |
| ZA=7 | 191 | **18** | 10 | 84 |

ET=12는 ZA=4~7에 집중, ET=52는 전 ZAxisNum 균등 분포.

### 23일 ET=52 시간대별 추이

- 10시: 257건 (집중 발생)
- 15시: 9건
- 총 714건 (두 로그 세트 합산)

---

## 7. strip.log

### 기본 구조

```
[StripLogger] [SmartAlignInspection]: [Request]
  CommonBody = ( ... CameraId : 2  InspectionType : 43 )
  StripBarcode : \0\0\0... (null)  StripCount : 1
[StripLogger] [SmartAlignInspection]: [Response]
  InspectionResult = {0|1}  ErrorType = 0
  XOffset = {val}  YOffset = {val}

[StripLogger] [StripInspection]:
  CommonBody = ( ... CameraId : 2  InspectionType : 40 )
[StripLogger] [StripInspection]: Strip Barcode decoding failed (Value is null)
```

### 날짜별 기록 현황

| 날짜 | 이벤트 타입 | 건수 | 결과 |
|------|-----------|------|------|
| 01-16 | SmartAlignInspection | Req 27 / Resp 27 | **전수 Pass**, InspectionResult=1 |
| 01-27 | StripInspection | Req 2 / Resp 2 | **전수 실패** "Value is null" |
| 01-28 | SmartAlignInspection | Req 1 / Resp 1 | **FAIL** (InspectionResult=0, XOffset=0, YOffset=0) |
| 그 외 날짜 | — | 0건 | 정상 양산 중 미기록 |

### SmartAlign Offset 실측값 (16일 27건)

| 지표 | 범위 | 평균 | 방향 |
|------|------|------|------|
| XOffset | -179 ~ +224 | +13 | 양방향 (±) |
| YOffset | +349 ~ +670 | +511 | **단방향 (항상 양수)** |

### 결론

- 정상 양산 중 strip.log 비어있는 이유: SmartAlign은 Teaching 또는 신규 레시피 도입 시에만 실행
- Carsem 현장에서 바코드 스캐너 미사용 (또는 바코드 미인쇄) 확인
- strip_id는 단순 증분으로 Mock 처리

---

## 8. lifecycle.log

### 이벤트 종류

```
[LifecycleLogger] [OnStartup]
[LifecycleLogger] [exitApplication]
[LifecycleLogger] [onDispatcherUnhandledException]
[LifecycleLogger] [subscribeUnhandledException]: UnobservedTaskException
[LifecycleLogger] [onCurrentDomainUnhandledException]
```

### 예외 핸들러 특성

| 핸들러 | 앱 종료 | Heartbeat | 특이사항 |
|--------|--------|----------|---------|
| `onDispatcherUnhandledException` | **안 함** | 정상 | UI 스레드 예외. 앱 살아있지만 장비 비정상 |
| `subscribeUnhandledException` | **안 함** | 정상 | 비동기 Task 미처리 예외. error.log AggregateException과 1:1 매핑 |
| `onCurrentDomainUnhandledException` | **안 함** | 정상 | AppDomain 전체 예외. 카메라 콜백 스레드 |

### 날짜별 이벤트 집계

| 날짜 | Start | Exit | Disp | Task | Crash | 주요 다운타임 |
|------|-------|------|------|------|-------|------------|
| 01-16 | 4 | 3 | 2 | 0 | 1 | — |
| 01-19 | 1 | 0 | 0 | 0 | 0 | — |
| 01-21 | 1 | 1 | 0 | 0 | 0 | 11:18~11:22 (4.3분) |
| 01-22 | 4 | 3 | 0 | 1 | 1 | 09:31~10:52 (**81분**) / 13:35~13:39 (4분) |
| 01-23 | 8 | 8 | 0 | 2 | 0 | 10:46~10:48 / 21:35~21:52 (17분) |
| 01-26 | 2 | 1 | 0 | **4** | 0 | 14:10~14:12 (1분) |
| 01-27 | 1 | 1 | 3 | **10** | 0 | 12:04~14:19 (**135분**) |
| 01-28 | 10 | 8 | 0 | **15** | **2** | 09:44~09:46 등 5회 |
| 01-29 | 9 | 7 | 0 | **12** | **2** | 14:04~16:10 (**127분**) |

### subscribeUnhandledException ↔ error.log AggregateException 1:1 매핑

28일 기준: lifecycle subscribeUnhandledException 15건의 타임스탬프와 error.log AggregateException 15건 타임스탬프 분단위 완전 일치 확인.

---

## 9. error.log

### 예외 타입 전체 목록

| 예외 타입 | 발생 날짜 | 건수 | 발생 위치 |
|----------|---------|------|---------|
| `System.AggregateException` | 22~29일 | **41건** | LotController.StartNewLot / LotService.UpdateLotEnd |
| `System.OperationCanceledException` | 전 기간 | 17건 | GrabLinkGrabber.GrabImage / ImageQueue.Dequeue |
| `HalconDotNet.HOperatorException #3142` | 01-27 | **20건** | BaseController.SaveImage (write_image) |
| `HalconDotNet.HOperatorException #4056` | 01-23 | **16건** | GridQfnTeachingViewModel.inspectPadAndLeads |
| `System.ArgumentNullException` | 22·23·27일 | 8건 | DeviceViewViewModel / GridQfnTeachingWindow |
| `System.NullReferenceException` | 20·23일 | 6건 | SystemInformationViewModel / SawOffsetListPanel |
| `GVisionWpf.Exceptions.VisionNotFoundException` | 16·23일 | 3건 | VisionOperation.Distance |
| `HalconDotNet.HOperatorException #1301` | 01-29 | 1건 | SideTeachingViewModel (dilation_circle) |
| `HalconDotNet.HOperatorException #1310` | 01-23 | 1건 | VisionOperation.TryMeasureSinglePackage |
| `HalconDotNet.HOperatorException #1401` | 01-16 | 1건 | VisionOperation.GetAverageArea (tuple_mean) |
| `GVisionWpf.Exceptions.WrongValueException` | 01-23 | 1건 | CameraIlluminator.SetBrightness |
| `GVisionWpf.Exceptions.BlobNotFoundException` | 01-28 | 1건 | LgaTeachingViewModel |
| `GVisionWpf.Exceptions.DuplicatedLotNumberException` | 01-22 | 1건 | LotController |
| `System.Collections.Generic.KeyNotFoundException` | 01-29 | 2건 | MapDeviceViewViewModel.GetColorOfResult |
| `System.ArgumentOutOfRangeException` | 23·29일 | 2건 | GridQfnInspectionService.GetGridAlignContext |
| `System.IO.IOException` (StripInfo.xml 잠금) | 01-23 | 1건 | CarsemSpcService.Export |

### AggregateException 날짜별 추이

| 날짜 | 건수 | 발생 위치 |
|------|------|---------|
| 01-22 | 1 | LotController.StartNewLot |
| 01-23 | 2 | LotController.StartNewLot |
| 01-26 | 4 | LotController.StartNewLot |
| 01-27 | 10 | LotController.StartNewLot |
| 01-28 | **15** | LotController.StartNewLot |
| 01-29 | 12 | LotController.StartNewLot + **LotService.UpdateLotEnd (4건)** |

### 주요 인과관계

| 날짜 | 트리거 | 결과 |
|------|--------|------|
| 01-22 | ArgumentNullException (DeviceViewViewModel) | Dispatcher 예외 → exitApplication → **257분 다운** |
| 01-23 | HALCON #4056 ×16 | Teaching 화면 연속 클릭 → Dispatcher 예외 × 16 → **132분 다운** |
| 01-23 | WrongValueException (조명) | SIDE ET=52 급증 전조 신호 |
| 01-27 | write_image ×20 (HALCON #3142) | 디스크 포화 → Task 파이널라이저 × 20 → MAP 큐 적체 → fps 저하 |
| 01-26~ | AggregateException 누적 | LOT Start/End 불균형 심화 → **LOT_END 미발행 가능성** |
| 01-27~ | Carsem_4X6 Teaching 미완성 | SIDE ET=52/ET=12 전수 → Pass율 48~62% |

---

## 10. ErrorType 코드북

> **주의:** 번호는 실측값. 의미는 로그 컨텍스트 기반 추론. GVisionWpf 공식 정의 문서 검증 필요.

### 공통

| ET | InspectionResult | 추론 의미 |
|----|-----------------|----------|
| 1 | 1 (PASS) | 정상 검사 성공 |
| 3 | 0 (FAIL) | 패키지 미감지 (빈 위치) |
| 30 | 0 (FAIL) | 이미지 미취득 / 카메라 타임아웃 |

### PRS 전용

| ET | 추론 의미 | Offset 특성 |
|----|----------|-----------|
| 11 | 위치 오프셋 허용치 초과 | XOffset/YOffset 실측값 있음 (패키지 인식 성공) |
| 15 | X축 편차 과도 | XOffset 큰 값 |
| 17 | 복합 편차 | 다양한 Offset 값 |
| 21 | 방향/각도 불량 | — |
| 25 | 레시피 임계값 이탈 | — |
| **62** | **의미 미확인 (신종)** | 01-28 첫 등장. XO±200, YO±100. Carsem_4X6 연관. **DS 확인 필요** |

### SIDE 전용

| ET | 추론 의미 | Offset 특성 | 발생 패턴 |
|----|----------|-----------|---------|
| 12 | 치수/형상 기준 미달 | XOffset=0 | Teaching 미완성 시 ZA 0~7 전수 균등 발생 |
| 50 | 이물질·오염 | XOffset=0 | 22일 28건 |
| **52** | **알고리즘 종합 실패** | XOffset=0 | 조명·임계값 이탈. 신규 레시피 전수 발생. ZA 0~7 균등 분포 |
| 55 | Burr 높이 기준 초과 | XOffset=0 | 22일 10건 |
| 56 | 복합 결함 | XOffset=0 | 22일 5건 |

---

## 11. DS 측 미확인 사항

| 우선순위 | 항목 | 현재 상태 |
|---------|------|----------|
| P1 | GVisionWpf ErrorType 정의 문서 | ET 의미 전부 추론 — 공식 정의 필요 |
| P1 | 레시피 `446275` / `640022` 정체 | EMAP 이상치·AggregateException 연관만 파악 |
| P2 | PRS ET=62 의미 | 01-28 첫 등장. Carsem_4X6 연관. 완전 미확인 |
| P2 | total_units 단위 정의 | Strip 수 vs 패키지 수 vs 슬롯 수 — 미확정 |
| P3 | SmartAlign YOffset 단방향 편차 원인 | +349~+670 항상 양수. 의도적 기구부 오프셋인지 불명 |
| P3 | Carsem_4X6 training 레시피 역할 | Teaching 전용 레시피인지 미확인 |

---

*마지막 업데이트: 2026-04-10*
