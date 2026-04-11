# CLAUDE.md — DS 비전 검사 장비 모바일 모니터링 프로젝트 지시 명세서

> **작성자**: 수석 아키텍트
> **수신자**: Claude Code
> **버전**: v1.0 (2026-04-11)
> **작업 성격**: 문서·Mock 데이터 패치 only (코드 작성 없음)

---

## 0. 프로젝트 컨텍스트

### 0.1 너의 역할
너는 15년 차 제조 IT(MES/스마트 팩토리) 도메인의 수석 아키텍트로서, 디에스(DS) 주식회사 비전 검사 장비의 모바일 모니터링 시스템 명세서를 다듬는다. **이번 작업은 코드를 작성하지 않는다.** 오직 문서(`.md`)와 Mock 데이터(`.json`)만 수정한다.

### 0.2 프로젝트의 본질
- 망 분리된 반도체 후공정 공장 현장에서, N대의 비전 검사 장비(EAP)를 한 대의 모바일 앱에서 모니터링
- 통신: MQTT (Eclipse Mosquitto) over Local Wi-Fi
- 핵심 가치: 데이터 병목 없는 파이프라인 + 현장 엔지니어 즉시성

### 0.3 작업 대상 저장소 구조
```
.
├── CLAUDE.md                              ← 이 파일
├── EAP_mock_data/
│   ├── README.md
│   ├── 01_heartbeat.json ~ 25_oracle_danger.json
│   └── (G1에서 신규 디렉토리 추가)
├── 명세서/
│   ├── DS_EAP_MQTT_API_명세서.md          ← v3.3 (메인 수정 대상)
│   └── DS_이벤트정의서.md                  ← (보조 수정 대상)
└── 문서/
    ├── 기획안.md                           ← 읽기 전용 (참조만)
    ├── 기업소개 및 요구사항.md              ← 읽기 전용 (참조만)
    └── 오라클 2차 검증 기획안.md            ← 읽기 전용 (참조만)
```

### 0.4 작업 시작 전 필독 문서
작업을 시작하기 전에 **반드시 아래 4개 파일을 순서대로 읽어서 컨텍스트를 머릿속에 적재**한다. 이걸 건너뛰면 G2/G3 패치의 의도를 잘못 이해할 위험이 있다.

1. `문서/기업소개 및 요구사항.md` — 5대 목표 확인
2. `문서/기획안.md` — 시스템 구조와 서버 역할 확인
3. `명세서/DS_EAP_MQTT_API_명세서.md` — v3.3 현재 상태 전체 확인
4. `명세서/DS_이벤트정의서.md` — 이벤트 분류 체계 확인

---

## 1. 작업 원칙 (모든 Task 공통)

### 1.1 절대 금지 사항
- ❌ **실로그 기반 Mock(01~17)의 수치 변경 금지** — 이들은 Carsem 14일 실측값이다. `equipment_id`만 G1에서 치환하는 경우를 제외하고 손대지 말 것.
- ❌ **Rule 38개 번호 재배치 금지** — R01~R38c는 외부에서 참조될 수 있으니 새 Rule은 R39부터 부여.
- ❌ **기존 토픽 구조 변경 금지** — `ds/{eq}/heartbeat` 등 8개 토픽 패턴은 그대로 유지.
- ❌ **PascalCase ↔ snake_case 변환 금지** — `inspection_detail` 내부는 PascalCase, 그 외는 snake_case 규칙 유지.
- ❌ **`saw_process` 필드 부활 금지** — README.md에 명시된 제외 정책 준수.

### 1.2 필수 준수 사항
- ✅ 모든 신규 Mock JSON에는 `_source` 또는 `_purpose` 메타 필드를 포함해 출처/목적을 명시.
- ✅ 합성 데이터(실로그 기반이 아닌 데이터)에는 `_synthetic: true` 필드 추가.
- ✅ 명세서 수정 시 변경 이력을 부록 C 개정 이력 표에 한 줄 추가 (v3.4로 버전 올림).
- ✅ 모든 Task 완료 후 `git diff --stat`로 변경 파일 수와 라인 수를 보고할 것.

### 1.3 검증 체크포인트
각 Task 끝에 **자기 검증 체크리스트**가 있다. 한 Task의 체크리스트를 모두 통과하지 못한 채로 다음 Task로 넘어가지 말 것.

---

## 2. Task 실행 순서 (G1 → G2 → G3 → G4 → G5)

이 순서는 의존성에 따라 결정되었다. **G2(Retained)와 G5(ALARM_ACK)는 한 쌍이지만, G2를 먼저 정의해야 G5의 ACK 대상이 명확**해지므로 순서를 분리했다.

| 순서 | Task ID | 제목 | 우선순위 | 예상 |
|---|---|---|---|---|
| 1 | G1 | Multi-Equipment 시나리오 추가 | P0 | 1일 |
| 2 | G2 | MQTT Retained Message 정책 신설 | P0 | 0.5일 |
| 3 | G3 | STATUS_UPDATE 진행률 3필드 추가 | P1 | 0.5일 |
| 4 | G4 | Mobile Subscriber 세션 정책 신설 (부록 A.7) | P1 | 0.5일 |
| 5 | G5 | 알람 ACK 메커니즘 (CONTROL_CMD ALARM_ACK) | P2 | 0.5일 |

---

## 3. Task G1 — Multi-Equipment 시나리오 Mock 추가

### 3.1 배경
현재 25개 Mock의 `equipment_id`가 모두 `DS-VIS-001` 단일값이라, **N:1 다설비 모니터링 본질**(기획서 §3 1번)을 검증할 수 없다. 모바일 타일 대시보드의 N대 동시 렌더링·정렬·동시 알람 처리 로직을 개발 단계에서 검증할 데이터가 필요하다.

### 3.2 작업 목표
기존 Mock을 **수정하지 않고**, 시나리오 정의 파일 1개와 시뮬레이터가 참조할 장비 프로파일 4개를 새로 만든다. 시뮬레이터(추후 별도 작업)는 이 시나리오를 읽어 기존 Mock의 `equipment_id`만 치환해 발행한다.

### 3.3 작업 항목

**3.3.1 신규 디렉토리 생성**
```
EAP_mock_data/scenarios/
```

**3.3.2 신규 파일 생성: `EAP_mock_data/scenarios/multi_equipment_4x.json`**

```json
{
  "_purpose": "N=4 동시 시나리오 — 모바일 N:1 타일 대시보드 검증용",
  "_synthetic": true,
  "_note": "기존 Mock 01~25를 equipment_id만 치환하여 4 토픽 트리에 동시 발행. 시뮬레이터가 이 파일을 읽어 routing.",
  "scenario_id": "MULTI-4X-001",
  "scenario_name": "4대 장비 동시 운영 — 정상/이상/대기/STOP 혼합",
  "duration_sec": 600,
  "concurrent_alarms": true,
  "equipments": [
    {
      "equipment_id": "DS-VIS-001",
      "display_name": "비전 #1 (정상 양산)",
      "site": "Carsem-A",
      "scenario": "RUN_NORMAL",
      "scenario_desc": "Carsem_3X3 정상 양산. 96.2% 수율 유지.",
      "mock_sequence": [
        "01_heartbeat",
        "02_status_run",
        "04_inspection_pass",
        "09_lot_end_normal",
        "23_oracle_normal"
      ],
      "tile_color_hint": "GREEN"
    },
    {
      "equipment_id": "DS-VIS-002",
      "display_name": "비전 #2 (Teaching 미완성)",
      "site": "Carsem-A",
      "scenario": "RUN_DEGRADED",
      "scenario_desc": "Carsem_4X6 신규 레시피 투입 직후 SIDE ET=52 폭주. WARNING 알람 발행.",
      "mock_sequence": [
        "01_heartbeat",
        "02_status_run",
        "19_recipe_changed_new_4x6",
        "05_inspection_fail_side_et52",
        "15_alarm_side_vision_fail",
        "24_oracle_warning"
      ],
      "tile_color_hint": "YELLOW"
    },
    {
      "equipment_id": "DS-VIS-003",
      "display_name": "비전 #3 (대기)",
      "site": "Carsem-A",
      "scenario": "IDLE",
      "scenario_desc": "직전 LOT 정상 완료 후 IDLE 상태. 다음 LOT 투입 대기.",
      "mock_sequence": [
        "01_heartbeat",
        "03_status_idle"
      ],
      "tile_color_hint": "GRAY"
    },
    {
      "equipment_id": "DS-VIS-004",
      "display_name": "비전 #4 (정지)",
      "site": "Carsem-A",
      "scenario": "STOP_CRITICAL",
      "scenario_desc": "CAM_TIMEOUT_ERR 발생으로 STOP. CRITICAL 알람 + Will 메시지 시뮬레이션.",
      "mock_sequence": [
        "11_alarm_cam_timeout",
        "17_alarm_eap_disconnected"
      ],
      "tile_color_hint": "RED"
    }
  ],
  "validation_criteria": {
    "expected_topic_count": 4,
    "expected_simultaneous_tiles": 4,
    "expected_alarm_levels": ["CRITICAL", "WARNING", "INFO"],
    "expected_status_distribution": {
      "RUN": 2,
      "IDLE": 1,
      "STOP": 1
    }
  }
}
```

**3.3.3 README 업데이트**

`EAP_mock_data/README.md`의 끝부분에 **신규 섹션 추가**:

```markdown
---

## 🌐 N:1 다설비 시나리오 (scenarios/)

기존 Mock 01~25는 단일 장비(DS-VIS-001) 기준 데이터입니다. 모바일 N:1 타일 대시보드 검증을 위해 시나리오 파일을 별도 디렉토리에 둡니다.

### 파일 목록

| 파일 | 장비 수 | 시나리오 | 검증 목적 |
| :--- | :--- | :--- | :--- |
| `scenarios/multi_equipment_4x.json` | 4대 | RUN+RUN+IDLE+STOP 혼합 | 타일 정렬, equipment_id 라우팅, 동시 알람, 색상 분기 |

### 사용 방법

시뮬레이터(별도 구현)는 이 파일을 읽어 `equipments[].equipment_id`로 기존 Mock의 `equipment_id` 필드를 치환한 뒤, 각 장비의 토픽 트리(`ds/DS-VIS-001/...`, `ds/DS-VIS-002/...`)에 동시 발행합니다. 시나리오 파일 자체는 발행되지 않으며, 시뮬레이터의 routing 입력으로만 사용됩니다.
```

### 3.4 검증 체크리스트
- [ ] `EAP_mock_data/scenarios/` 디렉토리 생성됨
- [ ] `multi_equipment_4x.json` 생성됨, JSON 파싱 에러 없음 (`python -m json.tool`로 확인)
- [ ] 시나리오 안의 모든 `mock_sequence` 항목이 실존하는 파일명과 일치 (확장자 `.json` 제외, prefix 일치)
- [ ] `equipment_id` 4개가 서로 다름 (DS-VIS-001~004)
- [ ] `tile_color_hint` 4개가 GREEN/YELLOW/GRAY/RED로 모두 다름
- [ ] `EAP_mock_data/README.md`에 N:1 시나리오 섹션 추가됨

### 3.5 Git 커밋 메시지
```
feat(mock): N:1 다설비 시나리오 추가 (G1)

- EAP_mock_data/scenarios/ 디렉토리 신설
- multi_equipment_4x.json: 4대 장비(RUN/RUN/IDLE/STOP) 동시 시나리오
- 기존 Mock 01~25를 equipment_id 치환하여 재활용하는 라우팅 구조
- README.md에 시나리오 사용법 섹션 추가

기획서 §3 "N:1 다설비 모니터링" 본질 요구사항 매핑.
시뮬레이터 routing 코드 작성 전 사전 작업.
```

---

## 4. Task G2 — MQTT Retained Message 정책 신설

### 4.1 배경
망 분리 현장에서 모바일 앱을 처음 켜거나 절전 후 복귀했을 때, **마지막 STATUS_UPDATE가 6초 후에야 도착**한다. 그 동안 빈 타일 화면이고, 30초 전에 발생한 CRITICAL 알람도 모른다. **MQTT 표준의 Retained Message 기능**으로 이 빈틈을 해결한다. 새 구독자가 토픽을 구독하는 순간 브로커가 마지막 retained 메시지를 즉시 푸시해주는 1급 기능이다.

이 정책은 단순 문서 한 줄이 아니라, **EAP Publisher 코드가 `MqttApplicationMessage.Retain = true`를 어디에 켜야 하는지** 결정짓는 사항이다.

### 4.2 작업 목표
API 명세서 §1.1 토픽 표에 **Retained 컬럼을 추가**하고, 토픽별 정책을 확정한다. Will 메시지와의 상호작용도 함께 명시한다.

### 4.3 작업 항목

**4.3.1 `명세서/DS_EAP_MQTT_API_명세서.md` §1.1 토픽 구조 표 수정**

기존 표에 **Retained 컬럼을 QoS 컬럼 뒤에 삽입**한다. 변경 후 표:

```markdown
### 1.1 토픽 구조 (v3.4)

| 토픽 패턴 | 이벤트 타입 | QoS | Retained | 방향 | 주기/조건 | 설명 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| ds/{eq}/heartbeat | HEARTBEAT | 1 | ❌ | Pub | 3초 | 온라인 감지 경량 신호 (3초마다 갱신, retained 시 stale 위험) |
| ds/{eq}/status | STATUS_UPDATE | 1 | ✅ | Pub | 6초 | 장비 상태 보고 (앱 시작 시 즉시 복원 필수) |
| ds/{eq}/result | INSPECTION_RESULT | 1 | ❌ | Pub | takt ~1,620ms | 매 takt 갱신, retained 무의미 |
| ds/{eq}/lot | LOT_END | 2 | ✅ | Pub | Lot 완료 1회 | 마지막 LOT 결과 즉시 복원 |
| ds/{eq}/alarm | HW_ALARM | 2 | ✅ | Pub | 이벤트 즉시 | 미해결 알람 즉시 복원 (ACK 시 빈 retained로 clear, §6.6 참조) |
| ds/{eq}/recipe | RECIPE_CHANGED | 2 | ✅ | Pub | 변경 즉시 | 현재 활성 레시피 즉시 복원 |
| ds/{eq}/control | CONTROL_CMD | 2 | ❌ | Sub | 명령 즉시 | 명령은 1회성, retained 금지 |
| ds/{eq}/oracle | ORACLE_ANALYSIS | 2 | ✅ | Pub | LOT_END 후 비동기 | 마지막 LOT 분석 결과 복원 |
```

**4.3.2 §1.1 토픽 구조 표 직후에 §1.1.1 신규 절 추가**

```markdown
### 1.1.1 Retained Message 정책 (신규 — v3.4)

망 분리 환경에서 모바일 앱이 새로 구독하거나 재연결할 때, 마지막 상태를 즉시 복원하기 위해 MQTT Retained Message를 활용합니다. EAP Publisher는 위 표의 Retained ✅ 토픽에 발행할 때 `MqttApplicationMessage.Retain = true`를 반드시 설정해야 합니다.

#### 정책 원칙

1. **Retained ✅ 토픽**: 새 구독자에게 즉시 마지막 메시지를 푸시. EAP는 발행 시 Retain 플래그 켜기.
2. **Retained ❌ 토픽**: 주기 발행 또는 1회성 명령. Retain 켜면 stale 메시지 위험 (특히 heartbeat).
3. **HW_ALARM clear 메커니즘**: 알람이 해소되면 모바일이 ALARM_ACK를 보내고, EAP는 동일 토픽에 **빈 페이로드 + Retain=true**를 발행하여 retained 메시지를 비웁니다 (§6.6 참조).
4. **Will 메시지와의 관계**: Will은 자동으로 Retain되지 않습니다. EAP_DISCONNECTED Will 메시지는 `WillRetain = true`로 명시 설정하여, 비정상 종료 후에도 새 구독자가 STOP 상태를 즉시 인지하도록 합니다.

#### 새 구독자 시나리오 예시

```
[시각 T+0]    EAP가 STATUS_UPDATE(RUN, Carsem_3X3) 발행 (Retain=true)
[시각 T+5]    EAP가 HW_ALARM(CRITICAL, CAM_TIMEOUT) 발행 (Retain=true)
[시각 T+10]   모바일 앱 신규 실행 → ds/+/status, ds/+/alarm 구독
[시각 T+10.05] Broker가 retained 메시지 2건을 즉시 푸시:
              - STATUS_UPDATE (T+0 시점, 5초 전 데이터)
              - HW_ALARM CRITICAL (T+5 시점, 5초 전 알람)
              → 모바일 타일이 즉시 RUN+CRITICAL 상태로 렌더링
```

retained 정책 없이는 모바일이 다음 STATUS_UPDATE(최대 6초 대기)를 받기 전까지 빈 화면을 표시합니다.

#### EAP Publisher 구현 가이드 (참고)

```csharp
// MQTTnet 예시 — 실제 구현은 별도 작업
var statusMessage = new MqttApplicationMessageBuilder()
    .WithTopic($"ds/{equipmentId}/status")
    .WithPayload(jsonPayload)
    .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
    .WithRetainFlag(true)   // ← 핵심
    .Build();
```
```

**4.3.3 §부록 A.4 Will 메시지 절 수정**

기존 Will 메시지 페이로드 JSON 위에 **한 줄 추가**:

```markdown
> ⚠️ Will 메시지는 반드시 `WillRetain = true`로 설정해야 합니다 (§1.1.1 참조). 이를 통해 EAP 비정상 종료 후 새로 구독하는 모바일이 STOP 상태를 즉시 인지할 수 있습니다.
```

**4.3.4 부록 C 개정 이력 표에 v3.4 행 추가**

```markdown
| v3.4 | 2026-04-11 | Retained Message 정책 신설 (§1.1.1): status/lot/alarm/recipe/oracle 토픽 retained 의무화. Will 메시지 WillRetain=true 명시. STATUS_UPDATE 진행률 3필드 추가. Mobile Subscriber 세션 정책 부록 A.7 신설. CONTROL_CMD에 ALARM_ACK 명령 추가. |
```

문서 상단의 버전 표기도 `v3.3 (최종)` → `v3.4`로 변경.

### 4.4 검증 체크리스트
- [ ] §1.1 표에 Retained 컬럼이 추가되었고, 8개 토픽 모두 ✅/❌ 표시
- [ ] §1.1.1 신규 절이 §1.1 직후, §1.2 직전에 위치
- [ ] §1.1.1에 EAP Publisher 코드 예시 포함 (참고용 C# 스니펫)
- [ ] §부록 A.4에 WillRetain=true 한 줄 추가됨
- [ ] 부록 C에 v3.4 개정 이력 추가됨 (단, 이 시점에선 아직 G3~G5 내용도 함께 미리 적어두기)
- [ ] 문서 헤더의 버전 표기가 v3.4로 갱신됨
- [ ] markdown 표 문법 깨짐 없음 (파이프 정렬 확인)

### 4.5 Git 커밋 메시지
```
feat(spec): MQTT Retained Message 정책 신설 (G2)

- §1.1 토픽 표에 Retained 컬럼 추가 (5개 토픽 retained 활성)
- §1.1.1 신규 절: Retained 정책 원칙 + 시나리오 + Publisher 가이드
- §부록 A.4 Will 메시지에 WillRetain=true 명시
- 부록 C 개정 이력에 v3.4 추가, 문서 버전 v3.3 → v3.4

망 분리 환경에서 모바일 앱 신규 구독 시 6초 빈 화면 문제 해결.
기획서 §3 "현장 보조 시스템" 즉시성 본질 매핑.
```

---

## 5. Task G3 — STATUS_UPDATE 진행률 3필드 추가

### 5.1 배경
LOT는 평균 82~370분 걸린다. 오퍼레이터가 모바일 타일을 보며 가장 자주 묻는 질문은 "지금 어디까지 갔어?"인데, 현재 STATUS_UPDATE에는 `uptime_sec`만 있고 **현재 LOT의 진행률을 알 수 있는 필드가 없다**.

INSPECTION_RESULT의 unit_id를 카운트해서 추정할 수도 있지만, 명세서가 강제하는 **PASS drop 정책**(모바일이 PASS는 detail 파싱 스킵) 때문에 이 방법은 불가능하다. 두 정책이 충돌하므로, **STATUS_UPDATE에 진행률을 얹어 PASS drop을 유지하면서도 진행률 표시가 가능**하게 한다.

### 5.2 작업 목표
`명세서/DS_EAP_MQTT_API_명세서.md` §3.1 STATUS_UPDATE 페이로드 필드 표에 3개 필드를 추가하고, Mock 02번 파일도 함께 업데이트한다.

### 5.3 작업 항목

**5.3.1 §3.1 페이로드 필드 표에 3개 행 추가**

기존 `uptime_sec` 행 다음에 아래 3행 삽입:

```markdown
| current_unit_count | integer | 1247 | N | 현재 LOT에서 검사 완료된 unit 누적 수. RUN 상태에서만 의미. IDLE/STOP 시 마지막 LOT 종료값 유지. |
| expected_total_units | integer | 2792 | N | 레시피 이력 기반 예상 총 unit 수. Historian에서 동일 레시피 직전 LOT 평균값 사용. 레시피 신규 투입 시 null. |
| current_yield_pct | float | 95.8 | N | 현재 LOT의 실시간 수율(%). pass_count / current_unit_count × 100. RUN 상태에서만 의미. |
```

3개 필드 모두 **N(선택)**으로 지정한 이유: 레시피 신규 투입 시점에는 expected_total_units을 알 수 없고, IDLE/STOP 상태에서는 진행률이 의미 없기 때문.

**5.3.2 §3.1 표 직후에 진행률 계산 규칙 박스 추가**

```markdown
> **[진행률 계산 규칙 — v3.4]**
> - `current_unit_count`: EAP가 LOT 시작 시 0으로 초기화하고, INSPECTION_RESULT 발행할 때마다 +1. LOT_END 후 IDLE 전환 시 마지막 값 유지.
> - `expected_total_units`: EAP가 LOT 시작 시 Historian에 직전 동일 레시피 LOT 3개의 total_units 평균값을 조회하여 설정. 이력 없으면 null. 레시피 변경 시 재계산.
> - `current_yield_pct`: EAP가 INSPECTION_RESULT 발행할 때마다 갱신. 모바일은 이 값으로 타일에 진행률 게이지를 그릴 수 있어 INSPECTION_RESULT를 한 건도 파싱하지 않아도 됨 (PASS drop 정책과 양립).
> - **모바일 타일 표시 예**: "Carsem_3X3 (1,247 / 2,792 unit · 44.7% · 수율 95.8%)"
```

**5.3.3 §3.2 페이로드 예시 갱신**

기존 RUN 예시에 3개 필드 추가:

```json
// RUN — Lot in progress (v3.4)
{
  "event_type": "STATUS_UPDATE",
  "equipment_status": "RUN",
  "lot_id": "LOT-20260122-001",
  "recipe_id": "Carsem_3X3",
  "operator_id": "ENG-KIM",
  "uptime_sec": 1528,
  "current_unit_count": 1247,
  "expected_total_units": 2792,
  "current_yield_pct": 95.8
}

// IDLE — after LOT_END (v3.4)
{
  "event_type": "STATUS_UPDATE",
  "equipment_status": "IDLE",
  "lot_id": "LOT-20260122-001",
  "recipe_id": "Carsem_3X3",
  "operator_id": "ENG-KIM",
  "uptime_sec": 6420,
  "current_unit_count": 2792,
  "expected_total_units": 2792,
  "current_yield_pct": 96.2
}
```

**5.3.4 Mock `EAP_mock_data/02_status_run.json` 수정**

기존 파일을 **읽고**, JSON 객체에 3개 필드를 추가한다. `_source`나 `_note` 같은 메타 필드는 그대로 유지하고, 이 변경에 대한 설명을 `_note`에 한 줄 덧붙인다:

추가할 필드 (uptime_sec 다음에):
```json
"current_unit_count": 1247,
"expected_total_units": 2792,
"current_yield_pct": 95.8,
```

`_note` 또는 신규 `_v34_note` 필드에 추가:
```
"v3.4 진행률 3필드 추가 — Carsem_3X3 LOT 실측 기반 (1,247/2,792, 수율 95.8%는 LOT 중간 시점 추정값)"
```

**5.3.5 Mock `EAP_mock_data/03_status_idle.json` 수정**

03번도 동일하게 진행률 필드 추가하되, IDLE 상태이므로 마지막 LOT 종료값을 사용:
```json
"current_unit_count": 2792,
"expected_total_units": 2792,
"current_yield_pct": 96.2,
```

### 5.4 검증 체크리스트
- [ ] §3.1 표에 3개 필드(current_unit_count, expected_total_units, current_yield_pct) 추가됨
- [ ] 3개 필드 모두 필수=N으로 표시
- [ ] §3.1 표 직후에 진행률 계산 규칙 박스 추가됨
- [ ] §3.2 RUN/IDLE 페이로드 예시 모두 새 필드 포함
- [ ] `02_status_run.json` JSON 파싱 에러 없음, 3개 필드 추가됨
- [ ] `03_status_idle.json` JSON 파싱 에러 없음, 3개 필드 추가됨
- [ ] 02와 03의 expected_total_units 값이 일치 (2792)
- [ ] PASS drop 정책(§4 박스)을 건드리지 않음 — 그 정책은 그대로 유지

### 5.5 Git 커밋 메시지
```
feat(spec): STATUS_UPDATE 진행률 3필드 추가 (G3)

- §3.1 페이로드 표에 current_unit_count, expected_total_units,
  current_yield_pct 3필드 추가
- §3.1 직후 진행률 계산 규칙 박스 추가
- §3.2 RUN/IDLE 페이로드 예시 갱신
- Mock 02_status_run.json, 03_status_idle.json 동기화

LOT 진행률 표시 요구를 PASS drop 정책과 양립하게 해결.
모바일 타일이 INSPECTION_RESULT 파싱 없이도 진행률 게이지 렌더 가능.
기획서 §목표 "진행 중인 Lot 정보 표시" 본질 매핑.
```

---

## 6. Task G4 — Mobile Subscriber 세션 정책 신설 (부록 A.7)

### 6.1 배경
API 부록 A.6 백오프 표는 **EAP→Broker 재연결 정책**만 명세하고 있다. **Mobile→Broker 재연결**은 같은 백오프를 쓰는지, `clean_session=false`로 끊긴 동안 큐를 보존하는지, persistent client_id는 어떻게 부여하는지 등이 미확정이다. 이건 기획서 §3 4번 "고가용성 확보" 그 자체에 해당하는 항목이다.

### 6.2 작업 목표
API 명세서 부록 A에 **A.7 Mobile Subscriber 세션 정책** 절을 신설한다.

### 6.3 작업 항목

**6.3.1 `명세서/DS_EAP_MQTT_API_명세서.md` 부록 A.6 직후에 신규 절 A.7 추가**

```markdown
### A.7 Mobile Subscriber 세션 정책 (신규 — v3.4)

망 분리 현장의 Wi-Fi는 자주 끊깁니다. 모바일 앱이 다시 붙었을 때 끊긴 동안 발생한 CRITICAL 알람을 받을지 못 받을지가 이 세션 정책에서 결정됩니다. EAP Publisher와 별도로, 모바일 Subscriber는 다음 정책을 따라야 합니다.

#### A.7.1 MQTT 연결 파라미터

| 항목 | 권장 값 | 이유 |
| :--- | :--- | :--- |
| protocol_version | MQTT v5.0 | session_expiry_interval 사용 위해 v5 필수 |
| clean_session (v3) / clean_start (v5) | **false** | 끊긴 동안 QoS 1/2 메시지 큐 보존 |
| session_expiry_interval (v5) | **3600s (1시간)** | 절전 모드 1시간까지 메시지 큐 유지. 1시간 초과 시 세션 폐기 |
| client_id | **`ds_mobile_{userId}_{deviceId}`** | 같은 사용자 재로그인 시 세션 승계. deviceId는 ANDROID_ID 또는 iOS identifierForVendor |
| keep_alive | **30s** | Wi-Fi 변동 빠른 감지. EAP는 60s지만 모바일은 더 짧게 |
| 재연결 백오프 | **§A.6과 동일** | EAP/Mobile 백오프 통일 (1s→2s→5s→15s→30s, max 60s, jitter ±20%) |

#### A.7.2 구독 토픽 권장 패턴

| 패턴 | 용도 | QoS |
| :--- | :--- | :--- |
| `ds/+/heartbeat` | 모든 장비의 ONLINE/OFFLINE 감지 | 1 |
| `ds/+/status` | 모든 장비 상태 + retained로 즉시 복원 | 1 |
| `ds/+/lot` | 모든 장비 LOT 완료 알림 | 2 |
| `ds/+/alarm` | 모든 장비 알람 + retained로 미해결 알람 복원 | 2 |
| `ds/+/recipe` | 모든 장비 레시피 변경 | 2 |
| `ds/+/oracle` | 모든 장비 Oracle 분석 결과 | 2 |

> INSPECTION_RESULT(`ds/+/result`)는 **모바일이 구독하지 않습니다**. 모바일은 §3.1의 STATUS_UPDATE 진행률 필드로 충분합니다 (§4 PASS drop 정책 + G3 진행률 정책).

#### A.7.3 재연결 시퀀스

```
[T+0]    Wi-Fi 단절 감지 → MqttClient.IsConnected = false
[T+1s]   1차 재시도 (jitter 0.8~1.2s)
[T+2s]   2차 재시도 (1.6~2.4s)
[T+5s]   3차 재시도 (4~6s)
[T+15s]  4차 재시도 (12~18s)
[T+30s]  5차 재시도 (24~36s)
[T+60s+] 6차 이후 max 60s 간격 유지

연결 성공 시:
1. clean_start=false 덕분에 끊긴 동안 큐된 QoS 1/2 메시지 자동 수신
2. retained 메시지로 모든 장비의 마지막 status/alarm/lot/recipe/oracle 즉시 복원
3. 모바일 UI는 1초 이내 전체 N개 타일 최신 상태로 렌더링 완료
```

#### A.7.4 세션 만료 후 처리

`session_expiry_interval=3600s`을 초과하여 세션이 폐기된 경우(예: 야간 보관 후 다음 날 출근), 모바일 앱은 새 세션을 생성하지만 retained 메시지로 모든 토픽의 최신 상태를 즉시 복원할 수 있습니다 (§1.1.1 참조). 누락된 알람이 있을 수 있으므로, **앱 시작 시 사용자에게 "마지막 세션이 만료되었습니다. 누락된 이력은 Historian 웹 대시보드에서 확인하세요" 안내 토스트를 표시**할 것을 권장합니다.

#### A.7.5 보안

| 항목 | 정책 |
| :--- | :--- |
| 인증 | `mobile_app` 계정 (§A.3 ACL 참조). userId/deviceId는 클라이언트 측 식별일 뿐, 브로커 인증은 공유 계정 |
| 권한 | Subscribe: `ds/#` 전체 / Publish: `ds/+/control` 한정 (EMERGENCY_STOP, STATUS_QUERY, ALARM_ACK만 ACL 허용) |
| TLS | 망 분리 로컬 Wi-Fi 환경에선 선택. 외부 노출 시 필수 |
```

### 6.4 검증 체크리스트
- [ ] 부록 A에 A.7 신규 절이 A.6 직후에 위치
- [ ] A.7.1~A.7.5 5개 하위 절 모두 존재
- [ ] A.7.1 표에 6개 파라미터 모두 권장값 포함
- [ ] A.7.2 표에 6개 토픽 패턴 (result 제외)
- [ ] A.7.2에 "INSPECTION_RESULT는 구독하지 않음" 명시
- [ ] A.7.3에 재연결 시퀀스 다이어그램 (텍스트) 포함
- [ ] A.7.5에서 ALARM_ACK가 모바일 허용 명령 목록에 포함됨 (G5와 일관성)

### 6.5 Git 커밋 메시지
```
feat(spec): Mobile Subscriber 세션 정책 신설 — 부록 A.7 (G4)

- A.7.1 MQTT 연결 파라미터 (clean_start=false, session_expiry=1h)
- A.7.2 구독 토픽 패턴 (INSPECTION_RESULT 제외 명시)
- A.7.3 재연결 시퀀스 (A.6 백오프 통일)
- A.7.4 세션 만료 후 사용자 안내 가이드
- A.7.5 보안 정책 (모바일 ACL 권한)

기획서 §3 "고가용성 확보" 본질 매핑.
망 분리 Wi-Fi 단절 시 알람 누락 방지.
```

---

## 7. Task G5 — 알람 ACK 메커니즘 (CONTROL_CMD ALARM_ACK)

### 7.1 배경
G2에서 알람을 retained로 발행하기로 했으니, **알람이 해소되었을 때 retained 메시지를 비울 메커니즘**이 반드시 필요하다. 그렇지 않으면 한 번 발생한 알람이 영원히 retained로 남아 모바일에 계속 표시된다.

### 7.2 작업 목표
1. CONTROL_CMD command enum에 `ALARM_ACK` 추가
2. ACK 시 EAP가 빈 페이로드 + Retain=true로 alarm 토픽을 발행하여 clear하는 시퀀스 명세
3. Mock 파일 1개 신규 생성 (`26_control_alarm_ack.json`)

### 7.3 작업 항목

**7.3.1 `명세서/DS_EAP_MQTT_API_명세서.md` §8.2 command 코드 목록 표에 1행 추가**

기존 `STATUS_QUERY` 행 다음에 추가:

```markdown
| ALARM_ACK | MES / 모바일 | 알람 확인 처리. EAP가 ds/{eq}/alarm 토픽에 빈 페이로드+Retain=true 발행하여 retained 메시지 clear. burst_id 그룹 단위 ACK 가능 | Y (확인) |
```

**7.3.2 §8.1 페이로드 필드 표에 burst_id 행 추가** (선택 필드로)

기존 `target_lot_id` 행 다음에 추가:

```markdown
| target_burst_id | string (UUID) | 8d9e1f2a-... | N | ALARM_ACK 명령 시 대상 burst_id. 동일 burst_id 그룹 전체를 한 번에 ACK. null이면 단독 알람 ACK |
```

**7.3.3 §6 HW_ALARM 절 끝에 §6.6 신규 절 추가**

```markdown
### 6.6 알람 ACK 및 Retained Clear 시퀀스 (신규 — v3.4)

§1.1.1에서 HW_ALARM 토픽이 Retained로 발행되도록 정책을 정의했습니다. 알람이 해소되거나 오퍼레이터가 확인한 경우, retained 메시지를 비우지 않으면 모바일에 영원히 알람이 표시됩니다. ALARM_ACK 명령으로 이 문제를 해결합니다.

#### 6.6.1 ACK 시퀀스

```
[T+0]    EAP가 HW_ALARM(CRITICAL, CAM_TIMEOUT_ERR) 발행
         → ds/DS-VIS-001/alarm topic에 retained=true로 저장됨
         → 모든 모바일 구독자에게 즉시 푸시

[T+10s]  오퍼레이터가 모바일 앱에서 알람 확인 버튼 탭
         → 모바일이 CONTROL_CMD(ALARM_ACK) 발행:
           ds/DS-VIS-001/control topic에 다음 페이로드:
           {
             "event_type": "CONTROL_CMD",
             "command": "ALARM_ACK",
             "issued_by": "MOBILE_APP",
             "target_burst_id": null,
             "reason": "Operator acknowledged"
           }

[T+10.1s] EAP가 CONTROL_CMD(ALARM_ACK) 수신
         → ds/DS-VIS-001/alarm topic에 빈 페이로드 + Retain=true 발행:
           Topic:   ds/DS-VIS-001/alarm
           Payload: (empty / zero-byte)
           Retain:  true
         → Broker가 retained 메시지 삭제
         → 신규 구독자는 더 이상 이 알람을 받지 않음

[T+10.2s] 기존 모바일 구독자도 빈 메시지 수신
         → 모바일은 빈 페이로드를 알람 dismiss 신호로 해석
         → 타일 알람 배지 제거
```

#### 6.6.2 burst_id 그룹 ACK

`burst_id`로 묶인 연속 알람(§6.1 burst_id/burst_count 참조)은 한 번의 ACK로 그룹 전체를 dismiss할 수 있습니다. 모바일이 `target_burst_id`를 명시하면 EAP는 동일 burst_id의 모든 retained 알람을 clear합니다. AggregateException 41건 연속 발생 케이스에 대응하기 위함입니다.

```json
// 모바일이 발행하는 burst 그룹 ACK
{
  "event_type": "CONTROL_CMD",
  "command": "ALARM_ACK",
  "issued_by": "MOBILE_APP",
  "target_burst_id": "8d9e1f2a-aggex-4abc-b100-000000000001",
  "reason": "Operator acknowledged AggregateException burst (41 alarms)"
}
```

#### 6.6.3 자동 ACK 시나리오

다음 경우 EAP는 모바일 명령 없이 자동으로 retained alarm을 clear할 수 있습니다.

| 자동 clear 트리거 | 대상 알람 |
| :--- | :--- |
| `auto_recovery_attempted=true` 알람의 복구 성공 확인 | LIGHT_PWR_LOW 등 |
| 동일 hw_error_code의 정상 STATUS_UPDATE 6회 연속 (36초) 수신 | CAM_TIMEOUT_ERR 복구 후 |
| 새 RECIPE_CHANGED 발생 시 이전 레시피 관련 VISION_SCORE_ERR | Teaching incomplete 알람 |

자동 clear 시에도 빈 페이로드 + Retain=true 발행 패턴은 동일합니다.

#### 6.6.4 ACL 권한

`mobile_app` 계정은 `ds/+/control` 토픽에 ALARM_ACK 명령만 발행할 수 있습니다 (EMERGENCY_STOP, STATUS_QUERY와 함께 모바일 허용 3종). EAP만 `ds/+/alarm` 토픽에 빈 페이로드 retained를 발행할 수 있습니다 (§부록 A.3 ACL 참조).
```

**7.3.4 부록 A.3 ACL 파일 갱신**

기존 ACL 주석 부분에 ALARM_ACK 추가 명시:

```text
# Mobile app — Read + limited Publish (control only)
user mobile_app
topic read ds/#
topic write ds/+/control
# Allowed commands: EMERGENCY_STOP, STATUS_QUERY, ALARM_ACK
```

**7.3.5 신규 Mock 파일 생성: `EAP_mock_data/26_control_alarm_ack.json`**

```json
{
  "_source": "CONTROL_CMD §8.2 ALARM_ACK 신규 명령 — v3.4",
  "_synthetic": true,
  "_note": "모바일 앱이 11번 CAM_TIMEOUT_ERR 알람을 ACK하는 시나리오. EAP는 이 명령 수신 후 ds/DS-VIS-001/alarm 토픽에 빈 페이로드+Retain=true 발행하여 retained clear (§6.6).",
  "message_id": "7f3a1200-ctrl-4abc-b100-000000000026",
  "event_type": "CONTROL_CMD",
  "timestamp": "2026-01-23T13:30:00.000Z",
  "command": "ALARM_ACK",
  "issued_by": "MOBILE_APP",
  "reason": "Operator acknowledged CAM_TIMEOUT_ERR alarm",
  "target_lot_id": null,
  "target_burst_id": null
}
```

burst 그룹 ACK 케이스도 함께 만든다.

**7.3.6 신규 Mock 파일 생성: `EAP_mock_data/27_control_alarm_ack_burst.json`**

```json
{
  "_source": "CONTROL_CMD §8.2 ALARM_ACK + burst_id 그룹 ACK — v3.4",
  "_synthetic": true,
  "_note": "16번 AggregateException 41건 burst를 한 번에 ACK. EAP는 동일 burst_id 그룹의 모든 retained alarm을 clear (§6.6.2).",
  "message_id": "7f3a1200-ctrl-4abc-b100-000000000027",
  "event_type": "CONTROL_CMD",
  "timestamp": "2026-01-26T11:15:00.000Z",
  "command": "ALARM_ACK",
  "issued_by": "MOBILE_APP",
  "reason": "Operator acknowledged AggregateException burst (41 alarms)",
  "target_lot_id": null,
  "target_burst_id": "8d9e1f2a-aggex-4abc-b100-000000000001"
}
```

**7.3.7 부록 B Mock 인덱스 표 업데이트**

부록 B의 Mock 인덱스 표 끝에 2행 추가:

```markdown
| 26_control_alarm_ack | CONTROL_CMD | §8 | ALARM_ACK / MOBILE_APP / 단독 알람 dismiss | v3.4 신규 |
| 27_control_alarm_ack_burst | CONTROL_CMD | §8 | ALARM_ACK / target_burst_id / 그룹 dismiss | v3.4 신규 |
```

**7.3.8 이벤트 정의서 동기화**

`명세서/DS_이벤트정의서.md`의 부록 Mock 데이터 인덱스 표에도 동일하게 26, 27 두 행을 추가한다 (열 구조는 그 문서의 기존 형식을 따를 것).

### 7.4 검증 체크리스트
- [ ] §8.2 command 표에 ALARM_ACK 행 추가됨, 모바일 허용=Y
- [ ] §8.1 페이로드 표에 target_burst_id 행 추가됨, 필수=N
- [ ] §6.6 신규 절이 §6.5 페이로드 예시 직후에 위치
- [ ] §6.6.1 ACK 시퀀스 다이어그램 포함
- [ ] §6.6.2 burst_id 그룹 ACK 예시 JSON 포함
- [ ] §6.6.3 자동 ACK 시나리오 표 (3개 트리거)
- [ ] §6.6.4 ACL 참조 포함
- [ ] 부록 A.3 ACL 파일에 ALARM_ACK 주석 추가됨
- [ ] `26_control_alarm_ack.json` JSON 파싱 OK
- [ ] `27_control_alarm_ack_burst.json` JSON 파싱 OK
- [ ] 27번의 target_burst_id가 §6.1 burst_id 예시 형식과 일치
- [ ] 부록 B Mock 인덱스에 26, 27 추가됨
- [ ] 이벤트 정의서 부록 Mock 인덱스에도 26, 27 추가됨

### 7.5 Git 커밋 메시지
```
feat(spec): 알람 ACK 메커니즘 신설 — CONTROL_CMD ALARM_ACK (G5)

- §8.2 command enum에 ALARM_ACK 추가 (모바일 허용)
- §8.1 페이로드에 target_burst_id 선택 필드 추가
- §6.6 신규 절: ACK 시퀀스, burst 그룹 ACK, 자동 ACK, ACL
- §부록 A.3 ACL에 ALARM_ACK 권한 주석 추가
- Mock 26_control_alarm_ack.json (단독 ACK)
- Mock 27_control_alarm_ack_burst.json (burst 그룹 ACK)
- 부록 B + 이벤트 정의서 Mock 인덱스 동기화

G2 retained 정책의 짝. 알람이 영원히 retained로 남는 문제 해결.
AggregateException 41건 burst 케이스 대응.
```

---

## 8. 최종 통합 검증 (모든 Task 완료 후)

### 8.1 문서 일관성 검증
모든 Task가 끝난 후, 다음 명령으로 전체 일관성을 점검한다.

```bash
# 1. 모든 JSON 파일 파싱 검증
find EAP_mock_data -name "*.json" -exec python3 -m json.tool {} \; > /dev/null

# 2. 신규 Mock 파일 존재 확인
ls -la EAP_mock_data/scenarios/multi_equipment_4x.json
ls -la EAP_mock_data/26_control_alarm_ack.json
ls -la EAP_mock_data/27_control_alarm_ack_burst.json

# 3. 명세서 버전 확인
grep -n "v3.4" 명세서/DS_EAP_MQTT_API_명세서.md | head -20

# 4. 변경 통계
git diff --stat
```

### 8.2 교차 참조 무결성 체크
다음 항목이 모두 일치하는지 확인:

| 체크 항목 | 위치 1 | 위치 2 | 일치해야 |
|---|---|---|---|
| ALARM_ACK 명령 | API §8.2 | Mock 26, 27 | 명령 코드 |
| target_burst_id 형식 | API §8.1 | Mock 27 | UUID 형식 |
| Retained 정책 | API §1.1.1 | API §6.6 | clear 메커니즘 |
| 진행률 필드 3개 | API §3.1 | Mock 02, 03 | 필드명 |
| Multi-Equipment Mock 시퀀스 | scenarios/ | 기존 Mock 파일명 | prefix 매칭 |
| 모바일 ACL | API §A.3 | API §A.7.5 | EMERGENCY_STOP/STATUS_QUERY/ALARM_ACK |

### 8.3 최종 보고 형식
모든 작업 완료 후 다음 형식으로 보고:

```
## DS 프로젝트 명세서 패치 v3.3 → v3.4 완료 보고

### 변경 통계
- 수정 파일: N개
- 신규 파일: M개
- 추가 라인: +X
- 삭제 라인: -Y

### Task 완료 현황
- [x] G1 Multi-Equipment 시나리오 (P0)
- [x] G2 Retained Message 정책 (P0)
- [x] G3 STATUS_UPDATE 진행률 3필드 (P1)
- [x] G4 Mobile Subscriber 세션 정책 (P1)
- [x] G5 알람 ACK 메커니즘 (P2)

### 검증 결과
- JSON 파싱: PASS (N/N 파일)
- 교차 참조: PASS (6/6 항목)
- 버전 갱신: v3.3 → v3.4

### 발견된 이슈 (있을 경우)
...

### 다음 단계 권고
이 패치 완료 후 진행 가능한 작업:
1. C# MQTTnet 가상 EAP Publisher 스켈레톤 (Retained 정책 + Multi-Equipment routing 반영)
2. .NET MAUI Mobile Subscriber 스켈레톤 (A.7 세션 정책 반영)
3. Oracle 1차 검증 if-else 엔진 (Rule R01~R38c)
```

---

## 9. 작업 시 주의사항 (실수 방지)

### 9.1 자주 하는 실수
- ❌ Markdown 표에서 파이프(`|`) 정렬을 망가뜨림 → 표 렌더링 깨짐. 각 행의 컬럼 수가 헤더와 일치하는지 매번 확인.
- ❌ 한국어 본문에서 영문 코드명을 백틱(`` ` ``) 없이 적음. `VISION_SCORE_ERR` 같은 코드는 항상 백틱 처리.
- ❌ JSON에 trailing comma 추가 → 파싱 에러. 각 파일 저장 전 `python3 -m json.tool`로 검증.
- ❌ 한 Task의 부록 C 개정 이력만 추가하고 다른 Task의 변경 사항을 빠뜨림. **부록 C는 G2 시점에 v3.4 통합 이력으로 한 번에 적기**.

### 9.2 도움이 되는 작업 패턴
- ✅ Task 시작 전에 해당 Task가 수정할 파일을 먼저 view로 읽어 현재 상태 확인
- ✅ str_replace로 정확한 위치를 짚어 수정 (광범위한 다시쓰기 금지)
- ✅ 각 Task 끝에 검증 체크리스트의 모든 항목을 한 번에 점검 후 다음 Task로
- ✅ Git 커밋은 Task 단위로 5번 분리. 한 커밋에 여러 Task 섞지 말 것

### 9.3 막혔을 때
- 명세서 구조나 본질이 모호하면 `문서/기획안.md`를 다시 읽고 결정의 근거를 찾는다
- 필드명/값 컨벤션이 모호하면 기존 v3.3 명세서와 Mock 01~25의 패턴을 따른다
- 두 가지 해석이 가능한 경우, **이 CLAUDE.md의 §0~§1 원칙**으로 돌아가서 본질에 더 부합하는 쪽을 선택

---

## 10. 최종 확인

이 명세서를 받았다면, 작업을 시작하기 전에 아래 4가지를 너 자신에게 확인한다.

1. ✅ 5개 Task의 우선순위와 의존성을 이해했는가? (G1 → G2 → G3 → G4 → G5)
2. ✅ 코드는 작성하지 않고 문서·Mock JSON만 수정한다는 점을 이해했는가?
3. ✅ 실로그 기반 Mock 01~17의 수치는 절대 변경하지 않는다는 원칙을 기억하는가?
4. ✅ 각 Task 끝에 검증 체크리스트를 모두 통과해야 다음 Task로 넘어간다는 규칙을 따를 것인가?

모두 ✅이면 **§0.4 필독 문서 4개를 먼저 read한 후**, Task G1부터 시작한다.

작업 진행 중 §0~§9 중 어느 절이라도 모순되거나 막막한 부분이 있다면, 추측으로 진행하지 말고 멈춰서 사용자에게 질문한다.

---

**End of CLAUDE.md**