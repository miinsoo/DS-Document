# 02. EAP 가상 비전 머신 (Publisher)

## 1. 역할
실제 비전 검사 장비(Genesem VELOCE-G7 / GVisionWpf)를 대체하는 Mock Publisher. 27개 Mock + multi_equipment_4x.json을 routing하여 N대 장비를 동시 시뮬레이션.

## 2. 책임
- ✅ 8개 토픽 발행 (HEARTBEAT, STATUS, RESULT, LOT_END, ALARM, RECIPE, ORACLE은 다른 서버 담당)
- ✅ Retained 정책 준수 (status/lot/alarm/recipe만 Retain=true)
- ✅ Will 메시지 등록 (EAP_DISCONNECTED, WillRetain=true)
- ✅ N대 장비 routing (multi_equipment_4x.json 기반)
- ✅ 백오프 재연결 (1s→2s→5s→15s→30s, max 60s, jitter ±20%)
- ✅ CONTROL_CMD 수신 처리 (ALARM_ACK 시 빈 페이로드 retained 발행으로 clear)
- ❌ 자체 판정 로직 (Oracle 담당)
- ❌ 데이터 저장 (Historian 담당)

## 3. 기술 스택
- C# .NET Standard 2.1+
- MQTTnet v4.3+
- WinForms 또는 Console UI (시뮬레이터 제어용)

## 4. 핵심 모듈

### 4.1 ScenarioLoader
- `EAP_mock_data/scenarios/multi_equipment_4x.json` 파싱
- 4대 장비 routing 테이블 생성

### 4.2 MockPublisher
- 시나리오의 mock_sequence 따라 Mock JSON 로드
- `equipment_id` 치환 후 발행
- Retain 플래그 토픽별 분기

### 4.3 MqttConnectionManager
- 백오프 재연결 (`명세서 부록 A.6`)
- Will 메시지 등록 (`WillRetain=true`)
- KeepAlive 60s

### 4.4 ControlCmdHandler
- `ds/{eq}/control` 구독
- ALARM_ACK 수신 시 → 해당 alarm 토픽에 빈 페이로드 + Retain=true 발행
- target_burst_id 그룹 ACK 지원

## 5. 발행 시나리오 (4대 장비)

| 장비 | 상태 | 발행 시퀀스 |
|---|---|---|
| DS-VIS-001 | RUN_NORMAL | heartbeat → status_run → inspection_pass → lot_end → oracle_normal |
| DS-VIS-002 | RUN_DEGRADED | heartbeat → status_run → recipe_changed → inspection_fail → alarm → oracle_warning |
| DS-VIS-003 | IDLE | heartbeat → status_idle |
| DS-VIS-004 | STOP | alarm_cam_timeout → eap_disconnected (Will) |

## 6. 데이터 플로우
```
Mock JSON 27개 ──┐
                 ├──→ ScenarioLoader ──→ MockPublisher ──→ Broker
multi_eq_4x.json ┘                          ↑
                                            │
                       Broker ──→ ControlCmdHandler (ALARM_ACK 처리)
```

## 7. 우선순위 작업
1. ScenarioLoader 구현 (G1 시나리오 파싱)
2. MqttConnectionManager + Will 등록
3. MockPublisher (Retain 분기 핵심)
4. ControlCmdHandler (ALARM_ACK clear 로직)
5. WinForms UI (시나리오 시작/정지 버튼)

## 8. 참조
- 페이로드: `99_EVENT_CATALOG.md`
- API 정식: `명세서/DS_EAP_MQTT_API_명세서.md` v3.4
- Mock: `EAP_mock_data/01~25.json`, `scenarios/multi_equipment_4x.json`
