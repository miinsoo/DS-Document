# DS 프로젝트 문서 디렉토리

> Claude Code가 각 서버 개발 시 컨텍스트로 로드할 분할 명세 모음.
> **Mobile은 별도 프론트엔드 팀이 담당**하므로 본 디렉토리에 포함되지 않음.
> Mobile 팀 Contract는 `MOBILE_FRONTEND_CONTRACT.md` 별도 파일.

## 파일 구성 (v1.1)

| 파일 | 용도 | 항상 로드 |
|---|---|---|
| `00_PROJECT_OVERVIEW.md` | 프로젝트 전체 정의 | ✅ |
| `99_EVENT_CATALOG.md` | 8개 이벤트 페이로드 사전 | ✅ |
| `98_DATA_FLOWS.md` | 7개 시나리오 end-to-end | ⚠️ 검증 시 |
| `01_BROKER.md` | Mosquitto 설정 | |
| `02_EAP_PUBLISHER.md` | C# 가상 비전 머신 | |
| `03_HISTORIAN.md` | Node.js + TimescaleDB | |
| `04_ORACLE.md` | Python (1차 P1 + 2차 P2) | |
| `05_MES.md` | C# 중앙 제어 + RDBMS | |
| `06_DISPATCHER.md` | Node.js DMZ 게이트웨이 | |

## 서버 우선순위 (개발 순서)

| Step | 서버 | 우선순위 | 의존성 |
|---|---|---|---|
| 1 | Broker | P0 | 없음 |
| 2 | EAP Publisher | P0 | Broker |
| 3 | **Oracle 1차 (Rule-based)** | **P1 필수** | Broker |
| 4 | Historian | P1 | Broker |
| 5 | MES | P1 | Broker |
| 6 | Oracle 2차 (AI) | P2 | Historian + 5+ LOT |
| 7 | Dispatcher | P2 | Historian + Online 팀 |

> **Oracle 1차는 P1 필수**입니다. 모바일이 PASS/FAIL 이분법을 넘어 38개 Rule 기반 WARNING/CRITICAL 등급을 받으려면 반드시 동작해야 합니다. Oracle 2차 AI는 5 LOT 이상 데이터 축적 후 후순위로 개발합니다.

## Claude Code 컨텍스트 로딩 패턴

| 작업 | 로드할 파일 |
|---|---|
| Broker 설정 | `00` + `01` |
| EAP Publisher 개발 | `00` + `99` + `02` |
| Historian 개발 | `00` + `99` + `03` |
| Oracle 1차 (P1) | `00` + `99` + `04` |
| Oracle 2차 (P2) | `00` + `99` + `04` + `오라클 2차 검증 기획안.md` |
| MES 개발 | `00` + `99` + `05` |
| Dispatcher 개발 | `00` + `06` |
| 통합 검증 | `00` + `99` + `98` + 관련 서버 |

## Mobile 앱 협업

Mobile은 별도 프론트엔드 팀이 .NET MAUI로 개발합니다. 본 디렉토리는 **백엔드 6종**만 다룹니다. Mobile 팀에 전달할 인터페이스 Contract는 다음 파일:

- `MOBILE_FRONTEND_CONTRACT.md` (단일 파일, 외부 팀 전달용)

## 외부 원본 우선

충돌 시 항상 원본이 우선:
- `명세서/DS_EAP_MQTT_API_명세서.md` v3.4
- `명세서/DS_이벤트정의서.md`
- `문서/기획안.md`
- `문서/오라클 2차 검증 기획안.md`

## 버전
v1.1 — 2026-04-12 (Mobile 제외, Oracle 1차/2차 분리, 6개 서버 분할 완료)
