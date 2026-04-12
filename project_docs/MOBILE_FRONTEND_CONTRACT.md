# DS 모바일 앱 — Frontend 팀 Contract 문서

> **수신**: 모바일 앱(.NET MAUI) 개발팀
> **발신**: DS 프로젝트 백엔드/EAP 팀
> **목적**: 모바일 앱이 백엔드 시스템과 통신하기 위한 인터페이스 합의 + 화면 요구사항 정의
> **버전**: v1.0 (2026-04-12) — API 명세서 v3.4 기반
> **상태**: 정식 합의용 (변경 시 양 팀 협의 필수)

---

## 1. 프로젝트 한 줄 정의

> **망 분리된 반도체 후공정 공장에서, N대의 비전 검사 장비(EAP)를 한 모바일 앱으로 모니터링하는 현장 보조 시스템.**

- **고객**: 디에스(DS) 주식회사
- **장비**: Genesem VELOCE-G7 비전 검사기 (Carsem 현장 14일 실로그 기반)
- **현장 환경**: 외부 인터넷 차단, 로컬 Wi-Fi만 사용
- **앱 성격**: **모니터링 + 알람 수신 중심**의 경량 앱 (장비 제어 거의 없음)

---

## 2. 모바일 앱이 해야 할 5가지 (기획서 §목표)

| # | 요구사항 | 우선순위 |
|---|---|---|
| 1 | 현재 장비 상태 표시 (RUN / IDLE / STOP) | P0 |
| 2 | 진행 중인 Lot 정보 표시 (진행률 게이지 포함) | P0 |
| 3 | Lot End 발생 알림 | P0 |
| 4 | Recipe 변경 알림 | P0 |
| 5 | 비전/카메라/조명 오류 또는 Exception 알람 수신 | P0 |

### 모바일 앱이 **하지 말아야 할** 것
- ❌ 데이터 적재 (Historian 서버 담당)
- ❌ 비즈니스 판정 로직 (Oracle 서버 담당)
- ❌ 임의의 장비 제어 (3개 명령만 허용 — §6 참조)
- ❌ INSPECTION_RESULT 토픽 구독 (§4.2 참조)

---

## 3. 통신 규약

### 3.1 프로토콜
| 항목 | 값 |
|---|---|
| 프로토콜 | **MQTT v5.0** (필수, v3는 안 됨) |
| 브로커 | Eclipse Mosquitto 2.x (현장 PC에 위치) |
| 포트 | 1883 (TCP) — TLS는 망 분리 환경이라 선택 |
| 직렬화 | JSON (UTF-8) |
| 라이브러리 권장 | **MQTTnet v4.3+** (NuGet) |

### 3.2 연결 파라미터 (필수 설정)
| 항목 | 값 | 이유 |
|---|---|---|
| `clean_start` (v5) | **false** | Wi-Fi 끊긴 동안 QoS 1/2 메시지 큐 보존 |
| `session_expiry_interval` | **3600s (1시간)** | 절전 모드 1시간까지 메시지 큐 유지 |
| `client_id` | **`ds_mobile_{userId}_{deviceId}`** | 같은 사용자 재로그인 시 세션 승계 |
| `keep_alive` | **30s** | Wi-Fi 변동 빠른 감지 |
| 인증 계정 | `mobile_app` (공유 계정) | ACL은 브로커가 담당 |

### 3.3 재연결 백오프 (필수 구현)
Wi-Fi 단절 감지 시 다음 백오프로 자동 재연결:

| 재시도 | 대기 (jitter ±20%) |
|---|---|
| 1차 | 1초 |
| 2차 | 2초 |
| 3차 | 5초 |
| 4차 | 15초 |
| 5차+ | 30초 (max 60초) |

> jitter 계산: 기준값 × (0.8 ~ 1.2) 무작위. 다수 장비 동시 재연결로 인한 브로커 과부하 방지용.

---

## 4. 구독해야 할 토픽 (6개)

### 4.1 토픽 목록
| 패턴 | 이벤트 | QoS | Retained | 용도 |
|---|---|---|---|---|
| `ds/+/heartbeat` | HEARTBEAT | 1 | ❌ | 장비 ONLINE/OFFLINE 감지 |
| `ds/+/status` | STATUS_UPDATE | 1 | ✅ | 장비 상태 + Lot 진행률 |
| `ds/+/lot` | LOT_END | 2 | ✅ | LOT 완료 알림 |
| `ds/+/alarm` | HW_ALARM | 2 | ✅ | 알람 수신 |
| `ds/+/recipe` | RECIPE_CHANGED | 2 | ✅ | 레시피 전환 알림 |
| `ds/+/oracle` | ORACLE_ANALYSIS | 2 | ✅ | LOT 분석 결과 (NORMAL/WARNING/DANGER) |

### 4.2 ❗ INSPECTION_RESULT는 구독 금지
`ds/+/result`는 **모바일이 절대 구독하지 않습니다.** 이유:
- takt 1.62초마다 발행 → N=4 장비면 초당 ~2.5건 × 2.1KB = 5KB/s 지속 부하
- 모바일 앱의 UI 렌더링 부하 + 배터리 소모 + 데이터 사용량 낭비
- **대안**: STATUS_UPDATE의 진행률 3필드(`current_unit_count`, `expected_total_units`, `current_yield_pct`)로 진행률 게이지를 그리세요. INSPECTION_RESULT를 한 건도 받지 않아도 충분합니다.

### 4.3 Retained Message의 의미 (반드시 이해할 것)
**앱이 새로 실행되거나 재연결되면, 위 ✅ 토픽 5종의 마지막 메시지가 즉시 자동으로 푸시됩니다.** 이 덕분에:
- 앱 켜자마자 0.3초 이내에 N개 타일 전체가 마지막 상태로 렌더링됩니다
- 30초 전에 발생한 미해결 알람도 즉시 화면에 나타납니다
- 따라서 **앱 시작 시 별도 "현재 상태 조회 API"가 없습니다.** MQTT 구독 = 최신 상태 자동 수신

---

## 5. 이벤트 페이로드 명세

> 모든 이벤트는 공통 헤더 5필드를 포함합니다 (`equipment_status`는 HEARTBEAT/CONTROL_CMD/ORACLE_ANALYSIS 제외).

### 5.1 공통 헤더
```json
{
  "message_id": "e7026e09-477c-43c3-8ba5-35b7b7f8a659",   // UUID v4
  "event_type": "STATUS_UPDATE",                          // enum
  "timestamp": "2026-01-22T16:41:42.123Z",                // ISO 8601 ms
  "equipment_id": "DS-VIS-001",                           // 장비 ID
  "equipment_status": "RUN"                               // RUN/IDLE/STOP
}
```

### 5.2 HEARTBEAT (3초 주기)
4필드만 (equipment_status 제외). 장비 ONLINE 판정용.
```json
{
  "message_id": "...",
  "event_type": "HEARTBEAT",
  "timestamp": "...",
  "equipment_id": "DS-VIS-001"
}
```

**판정 규칙**:
- ONLINE: 최근 9초 이내 수신
- WARNING: 9~30초 미수신 → 노란 배지
- OFFLINE: 30초 초과 미수신 → 회색/빨간 배지

### 5.3 STATUS_UPDATE (6초 주기, Retained ✅)
```json
{
  "message_id": "...",
  "event_type": "STATUS_UPDATE",
  "timestamp": "...",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "RUN",
  "lot_id": "LOT-20260122-001",
  "recipe_id": "Carsem_3X3",
  "recipe_version": "v1.0",
  "operator_id": "ENG-KIM",
  "uptime_sec": 1528,
  "current_unit_count": 1247,        // ★ 진행률 (RUN 시)
  "expected_total_units": 2792,      // ★ 예상 총 unit (null 가능)
  "current_yield_pct": 95.8          // ★ 실시간 수율
}
```

**모바일 타일 표시 예**:
> Carsem_3X3 (1,247 / 2,792 unit · 44.7% · 수율 95.8%)

**주의**:
- IDLE/STOP 상태에서는 진행률 3필드가 마지막 LOT 종료값 유지
- `expected_total_units`는 신규 레시피 투입 시 `null` 가능 → UI는 "측정 중" 표시

### 5.4 LOT_END (Retained ✅, QoS 2)
```json
{
  "event_type": "LOT_END",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "IDLE",
  "lot_id": "LOT-20260122-001",
  "lot_status": "COMPLETED",          // COMPLETED / ABORTED / ERROR
  "total_units": 2792,
  "pass_count": 2687,
  "fail_count": 105,
  "yield_pct": 96.2,
  "lot_duration_sec": 4923
}
```

**수율 판정 (UI 색상)**:
| yield_pct | 판정 | 색상 |
|---|---|---|
| ≥ 98% | EXCELLENT | 진한 초록 |
| 95~98% | NORMAL | 초록 |
| 90~95% | WARNING | 노랑 |
| 80~90% | MARGINAL | 주황 |
| < 80% | CRITICAL | 빨강 |

### 5.5 HW_ALARM (Retained ✅, QoS 2)
```json
{
  "event_type": "HW_ALARM",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "STOP",
  "alarm_level": "CRITICAL",          // CRITICAL / WARNING / INFO
  "hw_error_code": "CAM_TIMEOUT_ERR",
  "hw_error_source": "CAMERA",        // CAMERA/LIGHTING/VISION/PROCESS/MOTION/COOLANT
  "hw_error_detail": "GrabLinkGrabber failed to dequeue image within timeout.",
  "exception_detail": {
    "module": "GVisionWpf.Cameras.Grabber.GrabLinkGrabber",
    "exception_type": "OperationCanceledException",
    "stack_trace_hash": "b3f1a2c4"
  },
  "auto_recovery_attempted": false,
  "requires_manual_intervention": true,
  "burst_id": "8d9e1f2a-...",         // ★ 동일 원인 연속 알람 그룹 ID (없으면 null)
  "burst_count": 41                   // ★ 그룹 내 누적 횟수 (단독이면 1)
}
```

### 5.6 RECIPE_CHANGED (Retained ✅, QoS 2)
```json
{
  "event_type": "RECIPE_CHANGED",
  "equipment_id": "DS-VIS-001",
  "equipment_status": "IDLE",
  "previous_recipe_id": "ATC_1X1",
  "previous_recipe_version": "v1.0",
  "new_recipe_id": "Carsem_3X3",
  "new_recipe_version": "v1.0",
  "changed_by": "ENG-KIM"
}
```

### 5.7 ORACLE_ANALYSIS (Retained ✅, QoS 2)
LOT_END 후 Oracle 서버가 비동기 분석 결과 발행. NORMAL/WARNING/DANGER 3단계.
```json
{
  "event_type": "ORACLE_ANALYSIS",
  "equipment_id": "DS-VIS-001",
  "lot_id": "LOT-20260127-003",
  "recipe_id": "Carsem_4X6",
  "judgment": "WARNING",              // NORMAL / WARNING / DANGER
  "yield_status": {
    "actual": 68.5,
    "dynamic_threshold": { "warning_min": 65.0, "warning_max": 70.0 },
    "lot_basis": 8
  },
  "isolation_forest_score": 0.61,
  "ai_comment": "8 LOT 분석 결과 정상 범위 [65~72%]. 현재 68.5% 경계 근접.",
  "threshold_proposal": {
    "metric": "yield_pct",
    "current_warning": 90.0,
    "proposed_warning": 65.0,
    "basis": "8 LOT 기반 동적 계산 (EWMA 평균 68.2%, 표준편차 4.1%)"
  }
}
```

> `threshold_proposal`이 있으면 모바일에 "AI가 새 임계값을 제안합니다. [수락] [거부]" 모달 표시. 사용자 응답은 별도 API로 전송 (이 부분은 추후 협의).

---

## 6. 모바일이 발행하는 명령 (3개만 허용)

### 6.1 토픽
`ds/{equipment_id}/control` (QoS 2, Retained ❌)

### 6.2 허용 명령 (`mobile_app` 계정 ACL)
| 명령 | 용도 | 발행 시점 |
|---|---|---|
| **EMERGENCY_STOP** | 긴급 장비 정지 | 사용자가 위험 감지 시 |
| **STATUS_QUERY** | 즉시 STATUS_UPDATE 발행 요청 | "새로고침" 버튼 탭 시 |
| **ALARM_ACK** | 알람 확인 처리 | 사용자가 알람 ACK 버튼 탭 시 |

> ALARM_CLEAR / RECIPE_LOAD / LOT_ABORT는 **MES 전용**으로 ACL 차단됨. 모바일에서 발행 시도하면 브로커가 거부합니다.

### 6.3 페이로드 예시

#### EMERGENCY_STOP
```json
{
  "message_id": "uuid-v4",
  "event_type": "CONTROL_CMD",
  "timestamp": "2026-01-22T10:30:00.000Z",
  "equipment_id": "DS-VIS-001",
  "command": "EMERGENCY_STOP",
  "issued_by": "MOBILE_APP",
  "reason": "Operator detected smoke",
  "target_lot_id": null,
  "target_burst_id": null
}
```

#### ALARM_ACK (단독)
```json
{
  "message_id": "uuid-v4",
  "event_type": "CONTROL_CMD",
  "timestamp": "...",
  "equipment_id": "DS-VIS-001",
  "command": "ALARM_ACK",
  "issued_by": "MOBILE_APP",
  "reason": "Operator acknowledged",
  "target_lot_id": null,
  "target_burst_id": null
}
```

#### ALARM_ACK (burst 그룹)
```json
{
  "command": "ALARM_ACK",
  "issued_by": "MOBILE_APP",
  "target_burst_id": "8d9e1f2a-aggex-4abc-b100-000000000001",
  "reason": "Operator acknowledged AggregateException burst (41 alarms)"
}
```

### 6.4 ALARM_ACK 후 동작 (반드시 이해)
```
[T+0]  Mobile: CONTROL_CMD(ALARM_ACK) 발행
[T+0.1s] EAP: 수신 → ds/{eq}/alarm 토픽에 빈 페이로드 + Retain=true 발행
[T+0.2s] Broker: retained 메시지 삭제 → 신규 구독자는 더 이상 못 받음
[T+0.3s] Mobile: 빈 페이로드 수신 → 알람 dismiss 신호로 해석 → 타일 알람 배지 제거
```

> **빈 페이로드(zero-byte) 수신 = 알람 dismiss 신호**입니다. 파싱 에러로 처리하지 마세요.

---

## 7. 화면 요구사항

### 7.1 메인 대시보드
- **N개 장비 타일 그리드** (3열 권장, 가로 스크롤 가능)
- 타일 1개 구성:
  - 장비명 (display_name)
  - 상태 배지 (RUN=초록 / IDLE=회색 / STOP=빨강 / OFFLINE=검정)
  - Lot ID + Recipe 이름
  - **진행률 게이지** (current_unit_count / expected_total_units · %)
  - 실시간 수율 (current_yield_pct%)
  - 미해결 알람 배지 (있으면 빨간 점 + count)
- 타일 색상은 알람 우선:
  - CRITICAL 알람 있음 → **빨강**
  - WARNING 알람 있음 → **노랑**
  - 정상 RUN → **초록**
  - IDLE → **회색**
  - OFFLINE → **검정/회색 음영**

### 7.2 장비 상세 페이지
타일 탭 시 진입. 표시:
- 장비 정보 (display_name, equipment_id, site)
- 현재 상태 + 진행률 (위 5.3 표시 예 그대로)
- **최근 알람 이력** (최근 10건)
- **최근 LOT 이력** (최근 5건)
- **최근 Oracle 분석** (최근 1건)
- **버튼**: STATUS_QUERY (새로고침), EMERGENCY_STOP (긴급 정지), ALARM_ACK (알람 일괄 확인)

### 7.3 알람 팝업
| 알람 레벨 | UI 동작 |
|---|---|
| CRITICAL | **풀스크린 모달** (즉시) + 진동 + 사운드 |
| WARNING | 토스트 (3초 자동 닫힘) + 짧은 진동 |
| INFO | 토스트 (조용히) |

알람 모달 구성:
- 장비명 + 시각
- `hw_error_code` + `hw_error_detail`
- burst_count > 1이면 "동일 원인 N건 발생" 표시
- **버튼**: [지금 확인] (=ALARM_ACK 발행), [나중에] (=dismiss 안 함)

### 7.4 알람 De-duplication (필수)
동일 `burst_id`로 묶인 알람이 연속 도착해도 **하나의 알람으로 표시**하세요. `burst_count`만 갱신.
- 잘못된 예: AggregateException 41건 → 41개 모달 팝업 (사용자 폭주)
- 올바른 예: 1개 모달 + "41건 발생" 카운터

### 7.5 Will 메시지 우선 처리
EAP가 비정상 종료되면 Broker가 자동으로 다음을 발행합니다:
```json
{
  "event_type": "HW_ALARM",
  "alarm_level": "CRITICAL",
  "hw_error_code": "EAP_DISCONNECTED",
  "equipment_status": "STOP"
}
```

이 메시지를 수신하면:
1. 해당 장비 타일을 **즉시 STOP 배지**로 전환
2. Heartbeat 9초 OFFLINE 타이머 **무시** (Will 우선)
3. CRITICAL 모달 표시

### 7.6 세션 만료 후 처리 (1시간 초과)
session_expiry 1시간을 초과하여 세션이 폐기되면 새 세션이 생성됩니다. 이 경우:
- 끊긴 동안의 알람은 **재수신 불가** (큐 폐기됨)
- 단, retained 메시지로 5종 토픽의 마지막 상태는 즉시 복원
- **사용자에게 안내 토스트 표시 권장**: "마지막 세션이 만료되었습니다. 누락된 이력은 Historian 웹 대시보드에서 확인하세요."

---

## 8. 데이터 플로우 시나리오

### 시나리오 1: 앱 시작 시 (Retained 복원)
```
1. 사용자 앱 실행
2. MQTT 연결 (clean_start=false, client_id 영속)
3. 6개 토픽 동시 구독 (ds/+/status, lot, alarm, recipe, oracle, heartbeat)
4. Broker가 retained 메시지 5종을 즉시 푸시 (~0.3초)
5. UI 렌더링 완료: N개 타일 + 미해결 알람 + 진행률 게이지
```

### 시나리오 2: Wi-Fi 단절 → 복구
```
1. Wi-Fi 단절 감지 (MqttClient.IsConnected = false)
2. 백오프 재연결 시작 (1s → 2s → 5s → ...)
3. Wi-Fi 복구 시점에 연결 성공
4. clean_start=false 덕분에 끊긴 동안 큐된 QoS 1/2 메시지 자동 수신
5. retained 메시지 5종도 즉시 재수신 → UI 갱신
```

### 시나리오 3: Burst 알람 처리
```
1. EAP가 burst_id=X로 알람 41건 연속 발행
2. 모바일은 첫 알람 수신 시 모달 표시
3. 이후 같은 burst_id 알람은 모달 안 띄우고 burst_count만 갱신
4. 모달의 "지금 확인" 탭 → ALARM_ACK 발행 (target_burst_id=X)
5. EAP가 빈 페이로드 retained 발행
6. 모바일이 빈 페이로드 수신 → 모달 닫고 타일 배지 제거
```

### 시나리오 4: LOT 진행률 실시간 갱신
```
1. EAP가 6초마다 STATUS_UPDATE 발행 (progress 3필드 포함)
2. 모바일은 STATUS_UPDATE만 받아서 진행률 게이지 갱신
3. INSPECTION_RESULT는 한 건도 받지 않음 (구독 안 함)
4. LOT_END 수신 시 게이지를 100%로 채우고 토스트 표시
```

---

## 9. 권장 모듈 구조 (참고)

### 9.1 핵심 서비스
| 모듈 | 역할 |
|---|---|
| `MqttSubscriberService` | 백그라운드 MQTT 연결, 백오프, Will 처리 |
| `EventRouter` | 토픽별 이벤트 → ViewModel dispatch |
| `EquipmentRegistry` | equipment_id별 TileVM 관리 |
| `AlarmDeduplicator` | burst_id 기반 de-dup |
| `AlarmAckPublisher` | CONTROL_CMD ALARM_ACK 발행 |

### 9.2 ViewModel
| VM | 구독 토픽 | 책임 |
|---|---|---|
| `TileVM` | status, heartbeat | 타일 1개 상태 + 진행률 |
| `AlarmVM` | alarm | 알람 큐 + 모달 트리거 |
| `LotEndVM` | lot | LOT 완료 토스트 + 수율 색상 |
| `RecipeVM` | recipe | 레시피 변경 토스트 |
| `OracleVM` | oracle | NORMAL/WARNING/DANGER 표시 + 임계값 제안 모달 |

### 9.3 의사 코드 (백오프 + Will 우선 처리)
```csharp
// MqttSubscriberService.cs (개념 예시)
private async Task ConnectWithBackoffAsync()
{
    int[] delays = { 1000, 2000, 5000, 15000, 30000 };
    int attempt = 0;
    while (!_client.IsConnected)
    {
        try {
            var delay = delays[Math.Min(attempt, delays.Length - 1)];
            var jitter = _random.Next((int)(delay * 0.8), (int)(delay * 1.2));
            await Task.Delay(jitter);
            await _client.ConnectAsync(_options);
        }
        catch { attempt++; }
    }
}

// EventRouter.cs (Will 우선 처리)
private void OnAlarmReceived(AlarmEvent e)
{
    if (e.HwErrorCode == "EAP_DISCONNECTED")
    {
        // Heartbeat 9초 OFFLINE 타이머 무시
        _equipmentRegistry[e.EquipmentId].ForceStop();
        _alarmVM.ShowCriticalModal(e);
        return;
    }
    // 일반 알람: burst_id de-dup
    if (_alarmDedup.IsDuplicate(e.BurstId)) return;
    _alarmVM.Enqueue(e);
}
```

---

## 10. 협업 체크리스트

모바일 앱 개발 시작 전에 백엔드 팀과 다음 항목을 합의하세요:

### 10.1 환경 정보
- [ ] Mosquitto 브로커 IP/포트 (현장 / 개발 / QA 각각)
- [ ] `mobile_app` 계정 비밀번호
- [ ] 테스트용 시뮬레이터(가상 EAP) 가동 여부 — `EAP_mock_data/scenarios/multi_equipment_4x.json` 시나리오로 4대 장비 동시 발행 가능
- [ ] 개발 단계 장비 ID 컨벤션 (DS-VIS-001 ~ DS-VIS-N)

### 10.2 화면 요구 확인
- [ ] 타일 디자인 시안 합의 (색상, 배지 모양, 진행률 게이지 형태)
- [ ] 알람 모달 사운드/진동 정책 (CRITICAL만? WARNING도?)
- [ ] 다국어 지원 여부 (한국어/영어)
- [ ] 다크 모드 여부

### 10.3 추후 협의 항목 (별도 미팅)
- [ ] ORACLE_ANALYSIS의 `threshold_proposal` 수락/거부 응답을 어디로 보낼지 (별도 토픽 or REST API)
- [ ] 사용자 인증 정책 (앱 자체 로그인? SSO?)
- [ ] 알람 이력 조회 API (Historian 웹 대시보드 링크? 별도 REST?)
- [ ] 푸시 알림 정책 (앱 백그라운드 시 OS 푸시 사용 여부)

---

## 11. 참고 자료

| 문서 | 용도 |
|---|---|
| `명세서/DS_EAP_MQTT_API_명세서.md` v3.4 | **정식 API 명세 (충돌 시 이 문서 우선)** |
| `EAP_mock_data/01~27.json` | 개발/테스트용 Mock 데이터 27종 |
| `EAP_mock_data/scenarios/multi_equipment_4x.json` | N=4 시뮬레이션 시나리오 |
| MQTT v5.0 스펙 | https://docs.oasis-open.org/mqtt/mqtt/v5.0/ |
| MQTTnet GitHub | https://github.com/dotnet/MQTTnet |

---

## 12. 변경 이력

| 버전 | 일자 | 내용 |
|---|---|---|
| v1.0 | 2026-04-12 | 최초 작성. API 명세서 v3.4 기반. Retained 정책, 진행률 3필드, ALARM_ACK, A.7 세션 정책 모두 반영. |

---

## 13. 문의

- **백엔드/EAP 팀 연락처**: (별도 안내)
- **프로젝트 PM**: (별도 안내)
- **긴급 이슈**: 백엔드 팀 슬랙 채널

> 본 문서에 모호한 부분이 있으면 추측하지 말고 백엔드 팀에 질문하세요. 추측 구현은 통합 테스트에서 비용이 큽니다.

**End of Contract Document**
