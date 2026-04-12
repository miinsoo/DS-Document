# 01. Broker 서버 (Eclipse Mosquitto)

## 1. 역할
모든 서버 간 메시지 중재. EAP↔Mobile↔Historian↔Oracle↔MES의 M:N Loose Coupling 실현.

## 2. 책임 (Single Responsibility)
- ✅ MQTT 메시지 라우팅
- ✅ QoS 1/2 보장 (영속 큐)
- ✅ Retained Message 저장 및 신규 구독자 푸시
- ✅ Will 메시지 자동 발행 (EAP 비정상 종료 시)
- ✅ ACL 기반 권한 제어
- ❌ 비즈니스 로직 (절대 금지)
- ❌ 데이터 변환 (절대 금지)

## 3. 기술 스택
- Eclipse Mosquitto 2.x (오픈소스)
- 설정: `mosquitto.conf` + `passwd` + `acl`
- 배치: 현장 PC 또는 Edge 라우터

## 4. 핵심 설정

### mosquitto.conf
```
listener 1883
protocol mqtt
allow_anonymous false
password_file /etc/mosquitto/passwd
acl_file /etc/mosquitto/acl
persistence true
persistence_location /var/lib/mosquitto/
max_inflight_messages 100
max_queued_messages 10000
retain_available true
```

### ACL (5개 계정)
| 계정 | Subscribe | Publish |
|---|---|---|
| `eap_vis_{id}` | `ds/{id}/control` | `ds/{id}/#` |
| `oracle_server` | `ds/+/lot`, `ds/+/result` | `ds/+/oracle` |
| `mes_server` | `ds/#` | `ds/#` |
| `mobile_app` | `ds/#` | `ds/+/control` (EMERGENCY_STOP/STATUS_QUERY/ALARM_ACK만) |
| `historian` | `ds/#` | (없음) |

## 5. 운영 정책
- **Retained**: status/lot/alarm/recipe/oracle 토픽만 활성 (`99_EVENT_CATALOG.md` §1)
- **Will**: EAP_DISCONNECTED에 `WillRetain=true` 의무
- **세션**: clean_start=false, session_expiry=3600s 지원

## 6. 검증 시나리오
1. EAP 4대 동시 발행 → 모바일 1대가 `ds/+/status` 구독 시 4개 retained 즉시 수신
2. EAP 비정상 종료 → Broker가 자동으로 EAP_DISCONNECTED Will 발행
3. 모바일이 ALARM_ACK 발행 → EAP가 빈 retained로 alarm 토픽 clear

## 7. 다음 단계 작업
이 서버는 Mosquitto 설치/설정만 하면 끝. 코드 작성 없음.
