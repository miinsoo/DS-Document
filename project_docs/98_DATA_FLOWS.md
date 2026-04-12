# 98. 데이터 플로우 (End-to-End 시나리오)

> 7개 핵심 시나리오로 모든 서버 간 상호작용을 예시. 코드 작성 시 이 흐름을 따라 검증.

## 시나리오 1 — 정상 양산 LOT (Happy Path)

```
[T+0]    EAP: HEARTBEAT (3초마다 반복) → Broker → Mobile, Historian
[T+1]    EAP: STATUS_UPDATE(IDLE) Retain ✅ → Broker → Mobile 타일 회색
[T+5]    MES: CONTROL_CMD(RECIPE_LOAD, Carsem_3X3) → Broker → EAP
[T+10]   EAP: RECIPE_CHANGED Retain ✅ → Broker → Mobile, Historian
[T+15]   EAP: STATUS_UPDATE(RUN, current_unit_count=0) Retain ✅
         → Mobile 타일 RUN 배지로 전환, 진행률 0%
[T+15~]  EAP: INSPECTION_RESULT (takt 1.62s마다 2792회 반복)
         → Historian만 적재, Mobile은 PASS drop으로 파싱 안 함
         → EAP가 STATUS_UPDATE 6초마다 progress 갱신
[T+82분] EAP: LOT_END(COMPLETED, yield 96.2%) Retain ✅ QoS 2
         → Mobile 토스트, Historian 적재, Oracle 트리거
[T+82분+α] Oracle: Historian 조회 → ORACLE_ANALYSIS(NORMAL) Retain ✅
         → Mobile/MES 수신
[T+82분+1s] EAP: STATUS_UPDATE(IDLE) → Mobile 타일 회색 복귀
```

## 시나리오 2 — Teaching 미완성 + Burst 알람

```
[T+0]    EAP: RECIPE_CHANGED(Carsem_4X6 신규)
[T+5]    EAP: INSPECTION_RESULT FAIL ET=52 (1,253건 연속)
[T+30s]  EAP가 자체 모니터링: SIDE Pass율 48% < 50% threshold
         → HW_ALARM(VISION_SCORE_ERR, WARNING, burst_id=X) Retain ✅
[T+31s~] EAP: 동일 burst_id로 알람 41건 연속, burst_count++ 갱신
         → Mobile은 burst_id로 de-dup, 1개 알람 + count 표시
[T+5분]  오퍼레이터 ACK → Mobile: CONTROL_CMD(ALARM_ACK, target_burst_id=X)
         → Broker → EAP
[T+5분+1s] EAP: ds/{eq}/alarm에 빈 페이로드 + Retain=true 발행
         → Broker retained 삭제 → Mobile alarm dismiss
[T+82분] LOT_END(ABORTED, yield 68%) → Oracle WARNING 분석
```

## 시나리오 3 — 모바일 앱 신규 실행 (Retained 복원)

```
[T+0]    EAP 4대가 각자 STATUS/LOT/ALARM/RECIPE/ORACLE 발행 중
         → Broker에 5종 retained 메시지 누적
[T+10s]  사용자가 모바일 앱 실행
[T+10.1s] Mobile: Broker 연결, ds/+/status, ds/+/alarm, ds/+/lot, ds/+/recipe, ds/+/oracle 구독
[T+10.2s] Broker가 retained 메시지 즉시 푸시:
         - 4대 STATUS_UPDATE
         - 미해결 ALARM (있다면)
         - 마지막 LOT_END
         - 활성 RECIPE
         - 마지막 ORACLE_ANALYSIS
[T+10.3s] Mobile UI: 4개 타일 동시 렌더링, 진행률 게이지 표시
         (대기 시간 0.3초, retained 없으면 6초 빈 화면)
```

## 시나리오 4 — Wi-Fi 단절 후 자동 재연결

```
[T+0]    Mobile 정상 구독 중 (clean_start=false, session_expiry=3600s)
[T+10s]  Wi-Fi 단절
[T+11s]  Broker가 KeepAlive 30s 만료 감지, 세션은 1시간 보존
[T+11s~] EAP가 CRITICAL ALARM 5건 발행 → Broker가 Mobile session 큐에 적재
[T+25s]  Wi-Fi 복구
[T+26s]  Mobile: 1차 재시도(1s±jitter) → 연결 성공
         → clean_start=false → 큐된 5건 즉시 수신
         → retained 5종도 즉시 수신
[T+27s]  Mobile UI: 5건 알람 팝업 + 4개 타일 최신 상태
```

## 시나리오 5 — EAP 비정상 종료 (Will)

```
[T+0]    EAP: HEARTBEAT 정상 발행 중, Will 등록 (WillRetain=true)
[T+5s]   EAP 프로세스 크래시 (AggregateException)
[T+6s]   Broker: KeepAlive 60s 대기 시작
[T+66s]  Broker: Will 메시지 자동 발행 → ds/{eq}/alarm
         (EAP_DISCONNECTED, CRITICAL, retained=true)
[T+66.1s] Mobile: Will 수신 → Heartbeat 9s OFFLINE 타이머 무시
         → 즉시 STOP 배지 전환, CRITICAL 알람 팝업
[T+66.2s] Historian: Will 적재
[T+T]    엔지니어 EAP 재시작 → 새 STATUS_UPDATE(RUN) 발행
         → Mobile 자동 복구
```

## 시나리오 6 — 1차 + 2차 검증 비동기 분리

```
[1차 실시간]
INSPECTION_RESULT → Oracle Subscriber → Rule DB if-else (R01~R37)
                  → 100ms 이내 판정 → 모바일 즉시 표시

[2차 비동기]
LOT_END(COMPLETED) → Oracle Subscriber → Historian TSDB 조회
                   → 레시피별 EWMA+MAD 동적 임계값 계산
                   → Isolation Forest (LOT 10개 이상)
                   → ORACLE_ANALYSIS 발행 (NORMAL/WARNING/DANGER)
                   → 모바일 + MES 수신
                   → 오퍼레이터 승인 → Rule DB 임계값 자동 갱신
                   → 다음 LOT부터 새 임계값 적용
```

## 시나리오 7 — Online Area로 데이터 push

```
[1시간 단위 배치]
Dispatcher: Historian TSDB read-only 조회
          → 비식별화 (operator_id, lot_id 마스킹)
          → HTTPS push → Online Area AI 서버
          → AI가 RAG 임베딩 → 일/주간 보고서 생성
          → Web Backend → Web Front

(Local Area는 일절 응답 받지 않음 — 단방향 보장)
```

## 검증 매트릭스

| 시나리오 | 검증 대상 서버 | 핵심 본질 |
|---|---|---|
| 1 정상 LOT | EAP, Mobile, Historian, Oracle | Happy path |
| 2 Burst 알람 | EAP, Mobile | burst_id de-dup, ALARM_ACK |
| 3 앱 시작 복원 | Broker, Mobile | Retained Message |
| 4 재연결 | Broker, Mobile | clean_start, session_expiry |
| 5 Will | Broker, EAP, Mobile | Will + WillRetain |
| 6 비동기 검증 | Oracle, Historian | 1차/2차 분리 |
| 7 비식별화 push | Dispatcher, AI | DMZ 단방향 |
