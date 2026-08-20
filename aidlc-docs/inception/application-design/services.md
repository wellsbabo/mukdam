# 서비스 정의와 오케스트레이션 (Services)

**단계**: INCEPTION / Application Design
**작성일**: 2026-08-20

---

## 1. 서비스 관점 정리

이 앱에는 네트워크 서비스나 서버 계층이 없다. 여기서 `서비스`는 여러 컴포넌트를
조정해 하나의 사용자 목적을 완수하는 조정자를 뜻한다.

Q5=A 결정에 따라 도메인 동작을 하나의 파사드에 모으지 않고 역할별로 나눴다. 따라서
조정 책임은 다음 세 곳에 분산된다.

| 조정자 | 위치 | 조정 범위 |
|---|---|---|
| ViewModel | Presentation | 화면 단위 흐름. 검증 → 저장 → 초안 정리, 캘린더 월 전환, 백업 다단계 흐름 |
| BackupImporter | Domain | 가져오기 파이프라인. 디코딩 → 검증 → 반영 → 롤백 |
| EntryStore | Persistence | 저장소 트랜잭션 경계 |

**의도적으로 두지 않은 것**: `EntryService` 같은 범용 파사드. 그런 타입을 두면 Unit 1에서
만든 파일을 Unit 2, Unit 3에서 계속 수정하게 되고, 검증·공유·백업이라는 무관한 책임이
한 타입에 모인다.

---

## 2. 주요 흐름

### 2.1 기록 저장 (FR-02, FR-03)

```
EntryEditorView
    |
    | 사용자 입력 -> draft 갱신
    v
EntryEditorViewModel.save()
    |
    | 1. EntryValidator.validate(draft)
    |      실패 -> validationError 설정, 저장 중단, 화면 유지
    v
    | 2. EntryStore.insert / update (ValidatedEntryDraft)
    |      실패 -> StoreError, 컨텍스트 롤백, 입력 내용 유지
    v
    | 3. DraftStore.clear(target)
    | 4. AppLogger.log(.entrySaved)
    v
저장 성공 -> 화면 닫기
```

**보장 사항**
- 검증을 통과하지 않은 값은 `EntryStore`에 도달할 수 없다. 타입으로 강제된다 (SEC-04).
- 저장 실패 시 입력 내용이 사라지지 않는다 (FR-02 예외 처리).
- 초안은 저장 성공 후에만 제거된다.

### 2.2 작성 중 내용 보존과 복원 (NFR-04, Q7=A)

```
화면 진입
    |
    v
EntryEditorViewModel.onAppear()
    |
    +-- 신규 작성: DraftStore.load(.newEntry(kind:))
    |                발견 -> draft에 적재, AppLogger.log(.draftRestored)
    |                없음 -> 빈 draft
    |
    +-- 기존 수정: DraftStore.load(.existingEntry(id:))
                     발견 -> 초안 우선 적재
                     없음 -> Entry 값 적재

입력 변화 또는 백그라운드 전환
    |
    v
EntryEditorViewModel.persistDraftIfNeeded()
    |
    | draft.isEmpty == true  -> 저장하지 않음
    | draft.isEmpty == false -> DraftStore.save
    |                            실패 -> AppLogger.log(.draftPersistFailed), 흐름 계속
    v
저장 완료 또는 작성 취소 -> DraftStore.clear
```

**설계 판단**: 초안 저장 실패가 작성을 막지 않는다. 초안은 편의 기능이며, 실패했을 때
사용자에게 오류를 띄우면 작성 흐름만 방해한다.

### 2.3 목록 조회 (FR-06, FR-07)

```
TodayView / EntryListView
    |
    | @Query(predicate, sort)
    v
SwiftData ModelContext (SwiftUI 환경에서 주입)
    |
    v
Entry 배열 -> 뷰에서 날짜별 그룹화 및 표시
```

`EntryStore`를 경유하지 않는 유일한 읽기 경로다. `@Query`가 데이터 변경 시 화면을 자동
갱신하므로 별도 알림 체계가 필요하지 않다.

**필터 적용 방식**: `EntryListFilter`를 `@Query` 프레디케이트에 반영한다. 메모리에서
걸러내지 않는다.

### 2.4 캘린더 표시와 월 전환 (FR-13, NFR-12)

```
CalendarView 진입
    |
    v
CalendarViewModel.showMonth(containing: today)
    |
    | EntryStore.dayMarkers(inMonthContaining:filter:)
    |   표시 월의 시작~끝 범위만 조회
    v
markers: [Date: DayMarker] -> 격자에 종류별 표식 렌더링

사용자가 날짜 선택
    |
    v
CalendarViewModel.selectDay(day)
    |
    | EntryStore.entries(on:filter:)
    v
선택 날짜 기록 목록 표시 -> 항목 선택 시 EntryDetailView

사용자가 월 이동 또는 필터 변경
    |
    v
markers 재조회 (해당 월 범위만)
```

**보장 사항**: 전체 기록을 메모리에 올리지 않는다. 조회 범위는 항상 표시 중인 달이다.

### 2.5 기록 1건 공유 (FR-10)

```
EntryDetailView 공유 버튼
    |
    v
ShareTextComposer.text(for: entry)
    |
    v
ShareLink / UIActivityViewController -> 외부 앱
```

순수 함수 하나를 호출하는 단순 경로다. 조정자가 필요하지 않다.

### 2.6 백업 내보내기 (FR-11)

```
BackupView 내보내기
    |
    v
BackupViewModel.prepareExport()
    |
    | 1. EntryStore.allEntries()
    | 2. BackupCodec.encode(entries, exportedAt: now)
    |      실패 -> exportState = .failed(.encodingFailed)
    v
exportState = .ready(Data)
    |
    v
파일 저장 위치 선택 (사용자) -> 파일 쓰기
    |
    v
AppLogger.log(.backupExported(count:))
```

**화면 책임**: 평문 안내 문구를 내보내기 전에 표시한다 (SEC-02).

### 2.7 백업 가져오기 (FR-12, SEC-05, SEC-07, SEC-13)

```
BackupView 가져오기
    |
    v
파일 선택 (사용자)
    |
    v
BackupViewModel.selectImportFile(url)
    |
    | 파일 읽기 실패 -> importState = .failed(.notReadable)
    v
importState = .awaitingStrategy
    |
    | 사용자가 병합 방식 선택 (기존 유지 추가 / 전체 교체)
    v
BackupViewModel.confirmImport(strategy)
    |
    v
BackupImporter.performImport(data:strategy:)
    |
    | 1. BackupCodec.decode
    |      크기 초과 / JSON 손상 / 스키마 버전 미지원 -> 중단
    | 2. BackupCodec.validate
    |      레코드 무효 / 파일 내 중복 식별자 -> 중단
    |      * 여기까지 저장소는 전혀 변경되지 않음
    | 3. EntryStore.existingIDs()
    | 4. EntryStore.applyImport(records:strategy:)
    |      단일 트랜잭션. 실패 -> 컨텍스트 롤백 -> .persistenceFailed
    v
importState = .completed(summary)
    |
    v
AppLogger.log(.backupImportFinished(inserted:skipped:))
```

**보장 사항 (전부 성공 또는 전부 실패)**
- 검증이 끝나기 전에는 저장소를 변경하지 않는다.
- 반영은 단일 트랜잭션이며 실패 시 롤백한다.
- 손상된 파일을 가져와도 기존 데이터는 그대로다 (FR-12 완료 조건).

**오용 시나리오 대응 (SEC-11)**: 악의적으로 조작된 백업 파일을 가정한다.

| 조작 유형 | 방어 지점 |
|---|---|
| 거대한 파일로 메모리 소진 시도 | `BackupCodec` 크기 상한 검사 |
| 중첩 구조로 파싱 부하 유발 | JSON 디코딩 실패 처리 + 크기 상한 |
| 알 수 없는 `kind` 값 주입 | `validate`에서 `unknownKind` 오류 |
| 극단적 장·절 값 주입 | `EntryValidator` 범위 검증 |
| 과도한 길이의 본문 주입 | `EntryValidator` 길이 상한 |
| 파일 내 동일 식별자 반복으로 중복 생성 유도 | `validate`에서 `duplicateIDInFile` 오류 |
| 부분 반영 유도 (중간 레코드만 무효) | 전량 검증 후 일괄 반영 |

---

## 3. 의존성 주입 방식

| 대상 | 주입 방법 |
|---|---|
| `ModelContainer` | `MukdamApp`에서 `ModelContainerFactory.makeAppContainer()`로 생성해 `.modelContainer(_:)` 적용 |
| `ModelContext` | SwiftUI 환경(`@Environment(\.modelContext)`)에서 획득 |
| `EntryStore` | 화면에서 환경의 `ModelContext`로 생성해 ViewModel 초기화 시 전달 |
| `DraftStore` | 앱 진입점에서 생성해 환경 값으로 전달 |
| `BackupImporter` | `BackupViewModel` 초기화 시 `EntryStore`와 함께 생성 |

**테스트 시 대체 방법**: `ModelContainerFactory.makeInMemoryContainer()`로 컨테이너를
만들어 `EntryStore`에 주입한다. 가짜 구현을 만들지 않는다 (Q3=B).

---

## 4. 동시성 방침

- `EntryStore`, ViewModel은 `@MainActor`로 고정한다. SwiftData `ModelContext`를 여러
  스레드에서 다루지 않는다.
- 순수 도메인 타입(`EntryValidator`, `ScriptureReferenceFormatter`, `BackupCodec`,
  `ShareTextComposer`)은 `Sendable`이며 액터에 묶이지 않는다.
- 백업 인코딩·디코딩은 큰 데이터에서 시간이 걸릴 수 있다. 필요 시 별도 태스크로
  수행하되, 저장소 반영은 메인 액터에서 한다. 정확한 처리 방식은 Unit 3 Functional
  Design에서 확정한다.

---

## 5. 서비스 수준 보장 요약

| 보장 | 구현 지점 |
|---|---|
| 검증 우회 저장 불가 | `ValidatedEntryDraft` 초기화 접근 제어 |
| 저장 실패 시 입력 유지 | ViewModel이 실패를 값으로 받아 화면 상태 유지 |
| 삭제 즉시 완전 반영 | `EntryStore.delete` + cascade 삭제 규칙 |
| 가져오기 전부 성공 또는 전부 실패 | 검증 완료 후 단일 트랜잭션 반영 + 롤백 |
| 캘린더 조회 범위 제한 | `dayMarkers(inMonthContaining:)` 월 범위 프레디케이트 |
| 사용자 콘텐츠 미로깅 | `AppEvent` 열거형에 콘텐츠 케이스 부재 |
| 오류 문구에 내부 정보 미포함 | `ErrorMessageMapper` 단일 변환 지점 |
