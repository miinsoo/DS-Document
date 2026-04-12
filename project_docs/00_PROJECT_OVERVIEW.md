# 00. DS 프로젝트 마스터 개요

> **이 파일을 먼저 읽어라.** 다른 모든 파일은 이 파일을 전제로 작성됨.

## 1. 프로젝트 한 줄 정의

> **망 분리된 반도체 후공정 공장에서, N대의 비전 검사 장비(EAP)를 한 대의 모바일 앱으로 모니터링하는 Edge 기반 N:1 관제 시스템.**

## 2. 핵심 가치 (이 순서대로 우선)

1. **데이터 병목 없는 파이프라인** — 모든 설계 결정의 1순위 기준
2. **현장 엔지니어 즉시성** — 앱 켜자마자 상태 파악 가능
3. **결함 격리(Fault Isolation)** — 한 서버 장애가 전체 중단으로 이어지지 않음
4. **무중단 확장** — 새 클라이언트 추가 시 기존 시스템 무수정

## 3. 클라이언트 / 고객사

- **고객**: 디에스(DS) 주식회사 (반도체 후공정 비전·딥러닝 검사 솔루션 기업)
- **현장**: Carsem Inc. (실가동 로그 14일 분석 기반 — 2026-01-16~29)
- **장비**: Genesem VELOCE-G7 / GVisionWpf (C# WPF + HALCON + MQTTnet)
- **프로젝트 기간**: 2개월 단기 협업

## 4. 망 분리 환경 제약

- 외부 인터넷 차단
- 현장 내부 무선 공유기(Local Wi-Fi)만 사용
- BLE 또는 로컬 Wi-Fi 직접 통신
- 외부 클라우드 의존 금지

## 5. 시스템 영역 구분

```
┌─────────────────────── Local Area (망 분리) ──────────────────────┐
│                                                                    │
│  EAP(N대) ──┐                                                      │
│             │                                                      │
│  MES ───────┼──→ Broker(Mosquitto) ──┬──→ Mobile App(N:1 타일)    │
│             │                         ├──→ Historian → TSDB        │
│             │                         └──→ Oracle (Rule + AI)      │
│             │                                                      │
│                                       ↓                            │
│                                  Dispatcher                        │
│                                       ↓                            │
└───────────────────────────────────────┼────────────────────────────┘
                                        ↓
┌─────────────────────── Online Area ──────────────────────────────┐
│                       AI Server ←→ Web Backend ←→ Web Front       │
└────────────────────────────────────────────────────────────────────┘
```

## 6. 서버 구성 (8개 서버)

| # | 서버 | 영역 | 우선순위 | 상세 문서 |
|---|---|---|---|---|
| 1 | **Broker (Mosquitto)** | Local | P0 | `01_BROKER.md` |
| 2 | **EAP (가상 비전 머신)** | Local | P0 | `02_EAP_PUBLISHER.md` |
| 3 | **Mobile App (.NET MAUI)** | Local | P0 | `03_MOBILE_APP.md` |
| 4 | **Historian (Node.js)** | Local | P1 | `04_HISTORIAN.md` |
| 5 | **MES** | Local | P1 | `05_MES.md` |
| 6 | **Oracle (Python)** | Local | P2 (후순위) | `06_ORACLE.md` |
| 7 | **Dispatcher** | Local→Online | P2 | `07_DISPATCHER.md` |
| 8 | AI / Web Backend / Front | Online | P3 (별도 팀) | (이 프로젝트 범위 외) |

## 7. 통신 규약 (모든 서버 공통)

| 항목 | 값 |
|---|---|
| 프로토콜 | MQTT v5.0 |
| 직렬화 | JSON (UTF-8) |
| 브로커 | Eclipse Mosquitto 2.x |
| 토픽 패턴 | `ds/{equipment_id}/{event_type}` |
| Timestamp | ISO 8601 UTC 밀리초 (.fffZ) |
| ID 형식 | UUID v4 (RFC 4122) |
| 8개 토픽 / 8개 이벤트 | `99_EVENT_CATALOG.md` 참조 |

## 8. 기술 스택 통일

| 영역 | 언어/프레임워크 |
|---|---|
| 비전 PC 모듈 (EAP Publisher) | C# .NET Standard / MQTTnet |
| 모바일 앱 | C# .NET MAUI (Android/iOS) |
| Historian | Node.js (TypeScript) |
| MES | C# |
| Oracle | Python |
| Dispatcher | Node.js (TypeScript) |
| Broker | Eclipse Mosquitto (오픈소스) |
| TSDB | PostgreSQL TimescaleDB / InfluxDB |

## 9. 파일 구조 (Claude Code 컨텍스트 로딩 가이드)

```
project_docs/
├── 00_PROJECT_OVERVIEW.md         ← 항상 먼저 로드 (이 파일)
├── 01_BROKER.md                   ← Broker 작업 시
├── 02_EAP_PUBLISHER.md            ← EAP 시뮬레이터 작업 시
├── 03_MOBILE_APP.md               ← 모바일 앱 작업 시
├── 04_HISTORIAN.md                ← Historian 작업 시
├── 05_MES.md                      ← MES 작업 시
├── 06_ORACLE.md                   ← Oracle 작업 시 (후순위)
├── 07_DISPATCHER.md               ← Dispatcher 작업 시
├── 98_DATA_FLOWS.md               ← 시나리오별 end-to-end 흐름
├── 99_EVENT_CATALOG.md            ← 8개 이벤트 페이로드 요약 (모든 서버 공통 참조)
└── README.md                       ← 이 디렉토리 사용법
```

**Claude Code 작업 시 권장 컨텍스트 로딩**:
```
서버 X 작업 시 = 00 + 99 + 0X + 98 (필요 시)
```

## 10. 관련 외부 문서

이 프로젝트 문서는 아래 원본 문서를 보완·요약합니다. 충돌 시 원본이 우선:

| 외부 문서 | 역할 |
|---|---|
| `명세서/DS_EAP_MQTT_API_명세서.md` v3.4 | MQTT API 정식 명세 (페이로드 필드, Rule 38개 등) |
| `명세서/DS_이벤트정의서.md` | 이벤트 분류 및 판정 기준 정의 |
| `문서/기획안.md` | 시스템 아키텍처 원본 |
| `문서/오라클 2차 검증 기획안.md` | Oracle 서버 상세 기획 |
| `EAP_mock_data/` | Mock JSON 27개 + scenarios/multi_equipment_4x.json |

## 11. 문서 버전

- **v1.0** (2026-04-12) — 최초 작성. v3.4 명세서 반영.
