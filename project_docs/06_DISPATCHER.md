# 06. Dispatcher 서버 (Node.js — DMZ 단방향 게이트웨이)

## 1. 역할 한 줄
Local Area Historian TSDB의 데이터를 read-only로 조회 → 비식별화 처리 → Online Area AI 서버로 단방향 push하여 Local/Online 보안 경계를 유지한다.

## 2. ⚠️ 핵심 보안 원칙
| 원칙 | 의미 |
|---|---|
| **Read-only** | TSDB에 read-only 사용자만 사용. INSERT/UPDATE/DELETE 권한 없음 |
| **단방향(Push only)** | AI 서버는 응답 없음. 통신 방향은 Local → Online 한 방향만 |
| **비식별화 의무** | operator_id, 실명 lot_id 등 PII는 마스킹 후 전송 |
| **DMZ 격리** | Local Area 핵심 자산에 직접 접근 금지. 게이트웨이 역할만 |
| **최소 권한** | 필요 테이블만, 필요 컬럼만 SELECT |

## 3. 책임
- ✅ Historian TSDB read-only SELECT (1시간 단위 배치)
- ✅ 비식별화 처리 (operator_id 해시, lot_id 익명 ID 매핑)
- ✅ HTTPS POST로 AI 서버에 push
- ✅ push 실패 시 재시도 (3회 + 지수 백오프)
- ✅ 전송 이력 audit 로그
- ❌ 양방향 통신 (보안 침투 경로 차단)
- ❌ AI 서버로부터 응답/명령 수신
- ❌ Broker 직접 구독 (Historian 경유만)

## 4. 기술 스택
| 항목 | 선택 |
|---|---|
| 런타임 | Node.js 20 LTS + TypeScript 5+ |
| HTTP 클라이언트 | axios v1+ (HTTPS, mTLS 지원) |
| DB 클라이언트 | pg (PostgreSQL read-only 사용자) |
| 스케줄러 | node-cron |
| 로깅 | pino + 별도 audit 파일 |

## 5. DB 권한 설정 (필수)

```sql
-- Historian DB에 별도 read-only 사용자 생성
CREATE USER dispatcher_readonly WITH PASSWORD '...';
GRANT CONNECT ON DATABASE historian TO dispatcher_readonly;
GRANT USAGE ON SCHEMA public TO dispatcher_readonly;

-- 필요한 테이블만 SELECT 권한 부여
GRANT SELECT ON inspection_results TO dispatcher_readonly;
GRANT SELECT ON lot_ends TO dispatcher_readonly;
GRANT SELECT ON hw_alarms TO dispatcher_readonly;
GRANT SELECT ON oracle_analyses TO dispatcher_readonly;

-- 명시적으로 INSERT/UPDATE/DELETE 거부 (기본값이지만 명시)
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM dispatcher_readonly;
```

## 6. 핵심 모듈

```
dispatcher/
├── src/
│   ├── index.ts                       ← 진입점 + cron
│   ├── config/
│   │   ├── env.ts
│   │   └── ai_server.ts               ← AI 서버 endpoint, mTLS 인증서
│   ├── extractors/                    ← TSDB 조회 (read-only)
│   │   ├── inspection_extractor.ts    ← 1시간치 INSPECTION_RESULT
│   │   ├── lot_extractor.ts
│   │   ├── alarm_extractor.ts
│   │   └── oracle_extractor.ts
│   ├── anonymizer/                    ← ★ 비식별화
│   │   ├── pii_masker.ts
│   │   └── id_mapper.ts               ← lot_id ↔ anon_id 매핑
│   ├── pusher/
│   │   ├── ai_client.ts               ← HTTPS POST + mTLS
│   │   ├── retry.ts                   ← 3회 재시도 + 백오프
│   │   └── circuit_breaker.ts         ← AI 서버 다운 시 차단
│   ├── audit/
│   │   └── audit_logger.ts            ← 전송 이력 별도 파일
│   └── scheduler/
│       └── batch_runner.ts            ← node-cron (매시 정각)
└── package.json
```

## 7. 비식별화 정책

### 7.1 마스킹 대상
| 원본 필드 | 처리 방식 | 예시 |
|---|---|---|
| operator_id | SHA-256 해시 (앞 8자리) | `ENG-KIM` → `a3f9c2e1` |
| lot_id | anon_id 매핑 (id_mapper 테이블) | `LOT-20260122-001` → `ANON-LOT-00042` |
| equipment_id | 그대로 유지 (장비 자체는 식별 OK) | `DS-VIS-001` |
| recipe_id | 그대로 유지 | `Carsem_3X3` |
| timestamp | 그대로 유지 | |

### 7.2 id_mapper 테이블 (Local 측 보존)
```sql
CREATE TABLE id_mapper (
  original_id   TEXT PRIMARY KEY,
  anon_id       TEXT NOT NULL UNIQUE,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```
이 매핑 테이블은 **Local Area에만 존재**. AI 서버는 anon_id만 받아서 원본 추적 불가.

## 8. 배치 실행 흐름 (1시간 단위)

```
[매시 정각]
1. node-cron 트리거
2. 직전 1시간 범위 결정 (e.g., 14:00:00 ~ 14:59:59)
3. inspection_extractor.fetch(start, end)
   → SELECT FROM inspection_results WHERE time BETWEEN ...
4. anonymizer.process(rows)
   → operator_id 해시, lot_id → anon_id
5. ai_client.push(POST /ingest/inspection)
6. 응답 200 OK 시 → audit_logger.success(batch_id, count)
7. 실패 시 → retry (1s → 5s → 30s) → 3회 실패 시 audit_logger.failure
8. lot_extractor / alarm_extractor / oracle_extractor 동일 패턴 반복
```

## 9. 데이터 플로우
```
[Local Area]
Historian TSDB ←(read-only SELECT)─ Dispatcher Extractors
                                          ↓
                                   PII Anonymizer
                                          ↓
                                   id_mapper (Local 보존)
                                          ↓
                                   AI Client (HTTPS POST)
                                          ↓
─────────── 단방향 firewall (DMZ) ───────────
                                          ↓
[Online Area]
                                   AI Server /ingest endpoint
                                          ↓
                                   Vector DB 적재 + RAG
```

## 10. 보안 강화

### 10.1 mTLS (mutual TLS)
```typescript
// ai_client.ts (개념)
const httpsAgent = new https.Agent({
  cert: fs.readFileSync('certs/dispatcher.crt'),
  key: fs.readFileSync('certs/dispatcher.key'),
  ca: fs.readFileSync('certs/ai_server_ca.crt'),
  rejectUnauthorized: true
});
```

### 10.2 Firewall 규칙 (인프라)
- Local Area → Online Area: outbound TCP 443만 허용
- Online Area → Local Area: **모든 inbound 차단**
- Dispatcher 서버에는 inbound 포트 개방 없음

### 10.3 Audit 로그 별도 보관
```
logs/
├── app.log                ← 일반 로그
└── audit.log              ← 전송 이력 (rotation 별도, 1년 보존)
```

## 11. 실패 처리 / Circuit Breaker

```typescript
class CircuitBreaker {
  // AI 서버 5회 연속 실패 → 30분 차단
  // 차단 중에는 추출만 하고 push는 스킵
  // 30분 후 1건 시도, 성공 시 복구
}
```

이유: AI 서버 일시적 다운이 Dispatcher 무한 재시도 폭주로 이어지지 않도록.

## 12. 우선순위
**P2** — Online Area 협업 시점에 착수. Local Area 핵심 컴포넌트(Broker/EAP/Mobile/Historian/Oracle 1차)가 안정화된 후.

## 13. 검증 시나리오
1. 1시간 batch 트리거 → inspection_results 1만 건 SELECT → anonymize → AI 서버 push → 200 OK
2. AI 서버 다운 → 3회 재시도 후 audit failure 기록 → 다음 시간 재시도
3. operator_id 해시 검증: 같은 입력은 같은 해시, 원본 복원 불가
4. lot_id 매핑: 같은 lot_id는 같은 anon_id, id_mapper에 영속 저장
5. read-only 사용자로 INSERT 시도 → 권한 거부 (안전 검증)
6. mTLS 인증서 만료 → 연결 실패 → audit 알람

## 14. 참조
- AI 서버 endpoint 명세: 별도 협의 (Online Area 팀과)
- 비식별화 정책: 회사 내부 보안 가이드라인 준수
