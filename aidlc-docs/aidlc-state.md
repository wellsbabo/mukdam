# AI-DLC State Tracking

## Project Information
- **Project Name**: 묵담 (mukdam)
- **Project Type**: Greenfield
- **Start Date**: 2026-08-20T11:36:50Z
- **Current Stage**: INCEPTION - Workflow Planning

## Workspace State
- **Existing Code**: No
- **Programming Languages**: 없음 (신규 프로젝트)
- **Build System**: 없음
- **Project Structure**: Empty (문서만 존재)
- **Reverse Engineering Needed**: No
- **Workspace Root**: /Users/jungkwan/Desktop/octo/projects/mukdam

### 스캔 결과 상세
| 경로 | 종류 | 판정 |
|---|---|---|
| `.aidlc-rule-details/` | AI-DLC 규칙 문서 | 애플리케이션 코드 아님 |
| `AGENTS.md`, `README.md` | 워크플로우/안내 문서 | 애플리케이션 코드 아님 |
| `requirements/` | 요구사항 입력 문서 | 애플리케이션 코드 아님 |
| `dlc-example/` | 작성 예시 문서 | 애플리케이션 코드 아님 |

소스 코드 파일(.swift, .ts, .py 등) 및 빌드 파일(Package.swift, *.xcodeproj,
package.json 등) 없음 → Greenfield 확정.

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Pre-Workflow Decisions (요구사항 입력 문서 작성 시 확정)
| 항목 | 결정 |
|---|---|
| 구현 기술 | Swift / SwiftUI 네이티브 iOS |
| 앱 이름 | 묵담 |
| 사용자 유형 | 단일 사용자 (관리자 역할 없음) |
| 계정/서버 | 계정 없음, 자체 서버 없음, 로컬 저장 |
| 성경 번역본 본문 | 저작권 사유로 미포함 (구절 참조만 기록) |
| 묵상당 성경 구절 참조 | 다중 허용 |

입력 요구사항 문서: `requirements/requirements.md`, `requirements/constraints.md`

## Extension Configuration
| Extension | Enabled | Decided At |
|---|---|---|
| Security Baseline | Yes | Requirements Analysis |
| Resiliency Baseline | No | Requirements Analysis |
| Property-Based Testing | No | Requirements Analysis |

- Security Baseline 규칙 전문 로드 완료. 적용 대상 판정은
  `aidlc-docs/inception/requirements/requirements.md` 6.1절 참고.
- Resiliency Baseline, Property-Based Testing 규칙 전문은 opt-out에 따라 로드하지 않음.

## Execution Plan Summary
계획 문서: `aidlc-docs/inception/plans/execution-plan.md`

- **위험도**: Low / 롤백: Easy / 테스트 복잡도: Moderate
- **실행할 단계**: Application Design, Units Generation, Functional Design(Unit 1·3),
  NFR Requirements(앱 전체 1회), NFR Design(앱 전체 1회), Code Generation(단위별),
  Build and Test
- **건너뛸 단계**: Reverse Engineering(Greenfield), User Stories(단일 페르소나),
  Infrastructure Design(인프라 없음), Functional Design(Unit 2),
  NFR Requirements·NFR Design(Unit 2·3 — Unit 1 문서로 커버)
- **작업 단위 초안**: Unit 1 데이터·도메인 기반 → Unit 2 기록 작성·조회 화면 →
  Unit 3 캘린더·백업 (순차)
- **사용자 선행 작업**: 번들 식별자 결정 + Xcode 프로젝트 생성
  (Unit 1 Code Generation 시작 전)

## Confirmed Requirements Decisions
확정 결정 26건(D-01 ~ D-26)은
`aidlc-docs/inception/requirements/requirements.md` 2장에 기록.

주요 항목:
- iOS 17 이상, SwiftData 사용
- 2탭 구조(오늘 / 기록), 작성 진입점은 기도제목·묵상 분리
- 성경 책은 내장 66권 목록에서 선택, 약칭 검색 지원, 본문 미포함
- 삭제는 즉시 완전 삭제, 공유는 기록 1건 단위
- 기기 이전은 JSON 백업 내보내기/가져오기. iCloud 동기화는 다음 버전
- Xcode 프로젝트는 사용자가 직접 생성(AI가 절차 안내)
- 단위 테스트 + UI 테스트 작성

## Stage Progress

### 🔵 INCEPTION PHASE
- [x] Workspace Detection (2026-08-20T11:37:00Z)
- [ ] Reverse Engineering — 건너뜀 (Greenfield)
- [x] Requirements Analysis (2026-08-20T12:25:00Z 승인)
- [ ] User Stories — SKIP (단일 페르소나, FR에 완료 조건 포함)
- [x] Workflow Planning (2026-08-20T12:30:00Z) — 사용자 승인 대기
- [x] Application Design (2026-08-20T13:10:00Z) — 사용자 승인 대기
- [ ] Units Generation — EXECUTE (Part 1 계획 작성 완료, 답변 대기)

### 🟢 CONSTRUCTION PHASE
- [ ] Unit 1 — Functional Design — EXECUTE
- [ ] Unit 1 — NFR Requirements — EXECUTE (앱 전체 범위)
- [ ] Unit 1 — NFR Design — EXECUTE (앱 전체 범위)
- [ ] Unit 1 — Infrastructure Design — SKIP
- [ ] Unit 1 — Code Generation — EXECUTE
- [ ] Unit 2 — Functional Design — SKIP
- [ ] Unit 2 — NFR Requirements / NFR Design — SKIP (Unit 1 문서로 커버)
- [ ] Unit 2 — Infrastructure Design — SKIP
- [ ] Unit 2 — Code Generation — EXECUTE
- [ ] Unit 3 — Functional Design — EXECUTE
- [ ] Unit 3 — NFR Requirements / NFR Design — SKIP (Unit 1 문서로 커버)
- [ ] Unit 3 — Infrastructure Design — SKIP
- [ ] Unit 3 — Code Generation — EXECUTE
- [ ] Build and Test — EXECUTE

### 🟡 OPERATIONS PHASE
- [ ] Operations (placeholder)

## Current Status
- **Lifecycle Phase**: INCEPTION
- **Current Stage**: Units Generation — Part 1 (Planning)
- **Next Stage**: Units Generation — Part 2 (Generation)
- **Status**: 분해 결정 질문 7개 답변 대기

## Application Design 결정 (Q1~Q10)
| 항목 | 결정 |
|---|---|
| 프레젠테이션 패턴 | 혼합 (상태 전이 화면만 ViewModel, 목록은 @Query) |
| 폴더 구조 | 하이브리드 (기반 계층별 + 화면 기능별) |
| 저장소 추상화 | 프로토콜 없음, EntryStore 한 타입에 집중, 테스트는 인메모리 컨테이너 |
| 66권 데이터 | Swift 소스 정적 정의 |
| 도메인 서비스 | 역할별 전용 타입 (Validator, Formatter, ShareTextComposer, BackupCodec) |
| 오류 표현 | 도메인 오류 타입 + ErrorMessageMapper 분리 |
| 작성 중 보존 | 명시적 저장 + DraftStore 초안 자동 보존 |
| 테스트 | 단위 Swift Testing, UI XCUITest |
| 타깃 이름 | Mukdam (표시 이름 묵담) |
| 번들 식별자 | com.octo.mukdam |
