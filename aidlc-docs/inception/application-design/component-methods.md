# 컴포넌트 메서드 (Component Methods)

**단계**: INCEPTION / Application Design
**작성일**: 2026-08-20

이 문서는 메서드 시그니처와 입출력 타입을 정의한다. 상세 비즈니스 규칙(경계값, 정규화
순서, 스키마 버전 정책)은 CONSTRUCTION 단계의 Functional Design에서 확정한다.

시그니처는 Swift 표기를 쓰되 확정된 계약이 아니라 설계 의도를 나타낸다. 구현 시
세부는 조정될 수 있다.

---

## 1. Models

### 1.1 Entry

```swift
@Model
final class Entry {
    var id: UUID = UUID()
    var kindRawValue: String = EntryKind.prayer.rawValue
    var date: Date = Date()
    var title: String?
    var body: String = ""
    @Relationship(deleteRule: .cascade, inverse: \ScriptureReference.entry)
    var references: [ScriptureReference]?
    var createdAt: Date = Date()
    var updatedAt: Date = Date()

    var kind: EntryKind { get }   // kindRawValue 변환. 저장 프로퍼티 아님
    var orderedReferences: [ScriptureReferenceValue] { get }   // order 정렬 후 값 변환
}
```

**CloudKit 전방 호환 이유로 지켜지는 사항**
- 모든 저장 프로퍼티에 기본값 또는 옵셔널
- `@Attribute(.unique)` 없음
- 관계는 옵셔널 배열 + 역참조
- `kind`를 열거형 저장 프로퍼티가 아니라 문자열로 저장하고 계산 프로퍼티로 노출

### 1.2 ScriptureReference

```swift
@Model
final class ScriptureReference {
    var bookID: String = ""
    var chapter: Int = 1
    var verseStart: Int?
    var verseEnd: Int?
    var order: Int = 0
    var entry: Entry?

    var value: ScriptureReferenceValue { get }
}
```

### 1.3 EntryKind

```swift
enum EntryKind: String, Codable, CaseIterable, Sendable {
    case prayer
    case devotion
}
```

### 1.4 ScriptureReferenceValue

```swift
struct ScriptureReferenceValue: Equatable, Hashable, Sendable {
    let bookID: String
    let chapter: Int
    let verseStart: Int?
    let verseEnd: Int?
}
```

---

## 2. Persistence

### 2.1 EntryStore

```swift
@MainActor
final class EntryStore {
    init(context: ModelContext)

    // 생성 / 수정 / 삭제
    func insert(_ draft: ValidatedEntryDraft) throws -> Entry
    func update(_ entry: Entry, with draft: ValidatedEntryDraft) throws
    func delete(_ entry: Entry) throws

    // 단건 조회
    func entry(id: UUID) throws -> Entry?

    // 캘린더용 월 범위 집계 (NFR-12)
    func dayMarkers(inMonthContaining date: Date,
                    filter: EntryListFilter) throws -> [Date: DayMarker]
    func entries(on day: Date, filter: EntryListFilter) throws -> [Entry]

    // 백업
    func allEntries() throws -> [Entry]
    func existingIDs() throws -> Set<UUID>
    func applyImport(_ records: [ValidatedBackupRecord],
                     strategy: BackupMergeStrategy) throws -> BackupImportSummary
}
```

**메서드 목적**

| 메서드 | 목적 | 실패 시 |
|---|---|---|
| `insert(_:)` | 검증된 초안으로 기록 생성 | `StoreError.saveFailed`, 컨텍스트 롤백 |
| `update(_:with:)` | 기존 기록 갱신. 종류는 변경하지 않음 | `StoreError.saveFailed`, 컨텍스트 롤백 |
| `delete(_:)` | 기록 완전 삭제. 참조는 cascade 제거 | `StoreError.deleteFailed` |
| `entry(id:)` | 식별자로 단건 조회 | `StoreError.fetchFailed` |
| `dayMarkers(inMonthContaining:filter:)` | 표시 월 범위만 조회해 날짜별 종류 유무 집계 | `StoreError.fetchFailed` |
| `entries(on:filter:)` | 캘린더에서 선택한 날짜의 기록 조회 | `StoreError.fetchFailed` |
| `allEntries()` | 내보내기용 전체 조회 | `StoreError.fetchFailed` |
| `existingIDs()` | 가져오기 중복 판별용 식별자 집합 | `StoreError.fetchFailed` |
| `applyImport(_:strategy:)` | 검증 완료 레코드 일괄 반영. 단일 트랜잭션, 실패 시 전체 롤백 | `StoreError.saveFailed` |

**목록 조회 메서드가 없는 이유**: `TodayView`와 `EntryListView`는 SwiftUI `@Query`로
직접 조회한다 (Q1=C). 저장소에 같은 조회를 중복 정의하면 두 경로가 갈라진다.

### 2.2 ModelContainerFactory

```swift
enum ModelContainerFactory {
    static func makeAppContainer() throws -> ModelContainer
    static func makeInMemoryContainer() throws -> ModelContainer
}
```

| 메서드 | 목적 |
|---|---|
| `makeAppContainer()` | 앱 실행용. 저장 파일 경로 지정과 데이터 보호 속성 적용 (SEC-01) |
| `makeInMemoryContainer()` | 테스트용. 디스크에 쓰지 않는 컨테이너 |

### 2.3 DraftStore

```swift
final class DraftStore {
    init(directoryURL: URL)

    func save(_ draft: EntryDraft, for target: DraftTarget) throws
    func load(for target: DraftTarget) -> EntryDraft?
    func clear(for target: DraftTarget)
}

enum DraftTarget: Hashable, Codable {
    case newEntry(kind: EntryKind)
    case existingEntry(id: UUID)
}
```

| 메서드 | 목적 | 실패 처리 |
|---|---|---|
| `save(_:for:)` | 작성 중 내용 보존 (NFR-04) | 던짐. 호출측은 무시하고 작성 흐름을 막지 않음 |
| `load(for:)` | 앱 재시작 후 초안 복원 | 실패 시 nil 반환. 초안 없음과 동일 취급 |
| `clear(for:)` | 저장 완료 또는 사용자 폐기 시 초안 제거 | 실패를 던지지 않음 |

**설계 판단**: `load`는 던지지 않는다. 초안 복원 실패는 사용자가 새로 쓰면 되는
상황이며, 오류를 노출할 이유가 없다. `save` 실패도 작성 자체를 막지 않는다.

---

## 3. Domain

### 3.1 EntryDraft / ValidatedEntryDraft

```swift
struct EntryDraft: Codable, Equatable, Sendable {
    var kind: EntryKind
    var date: Date
    var title: String
    var body: String
    var references: [ScriptureReferenceValue]

    var isEmpty: Bool { get }   // 초안 저장 필요 여부 판단
}

struct ValidatedEntryDraft: Equatable, Sendable {
    let kind: EntryKind
    let date: Date                 // 일 단위 정규화 완료
    let title: String?             // 공백만이면 nil
    let body: String               // 트림 완료, 1자 이상 보장
    let references: [ScriptureReferenceValue]   // 검증 완료, 순서 확정
}
```

**초기화 접근 제어**: `ValidatedEntryDraft`의 초기화는 `EntryValidator`만 호출할 수
있도록 제한한다. 검증을 우회한 생성 경로를 막는다 (SEC-04).

### 3.2 EntryValidator

```swift
enum EntryValidator {
    static func validate(_ draft: EntryDraft,
                         limits: EntryLimits = .default,
                         calendar: Calendar = .current)
        -> Result<ValidatedEntryDraft, EntryValidationError>

    static func validate(_ reference: ScriptureReferenceValue,
                         limits: EntryLimits = .default)
        -> Result<ScriptureReferenceValue, ScriptureReferenceError>
}
```

**검증 항목 (상세 규칙은 Functional Design에서 확정)**
- 내용: 트림 후 1자 이상, 최대 길이 이내
- 제목: 최대 길이 이내. 공백만이면 nil로 정규화
- 참조 개수: 상한 이내. 종류가 `prayer`면 참조가 없어야 함
- 참조별: 책 식별자가 정경에 존재, 장 범위, 절 범위, 끝 절 단독 금지, 역순 금지
- 날짜: 지정 캘린더 기준 일 단위 정규화

**`calendar` 매개변수 이유**: 시간대 의존 동작을 테스트에서 고정할 수 있게 한다.

### 3.3 EntryLimits

```swift
struct EntryLimits: Sendable {
    let bodyMaxLength: Int
    let titleMaxLength: Int
    let maxReferencesPerEntry: Int
    let chapterRange: ClosedRange<Int>
    let verseRange: ClosedRange<Int>

    static let `default`: EntryLimits
}
```

**초기 제안값** (Functional Design에서 확정)

| 항목 | 제안값 | 근거 |
|---|---|---|
| `bodyMaxLength` | 20,000 | 긴 묵상도 수용. 저장·표시 성능 방어선 |
| `titleMaxLength` | 100 | 한 줄 제목 용도 |
| `maxReferencesPerEntry` | 20 | 실사용 상한을 크게 웃도는 방어값 |
| `chapterRange` | 1...150 | 최대 장 수(시편 150편) 기준 전역 상한 |
| `verseRange` | 1...200 | 최대 절 수(시편 119편 176절) 기준 전역 상한 |

장·절 상한은 책별 검증이 아니라 전역 방어값이다 (요구사항 명세 4.2절 결정 유지).

### 3.4 BibleCanon / BibleBook

```swift
struct BibleBook: Identifiable, Equatable, Sendable {
    let id: String
    let name: String
    let abbreviations: [String]
    let canonicalOrder: Int
    let testament: Testament
}

enum Testament: String, Sendable { case old, new }

enum BibleCanon {
    static let books: [BibleBook]                     // 66권, 정경 순서
    static func book(id: String) -> BibleBook?
    static func search(_ query: String) -> [BibleBook]
}
```

| 메서드 | 목적 | 반환 |
|---|---|---|
| `book(id:)` | 식별자로 조회. 검증과 문자열화에 사용 | 없으면 nil |
| `search(_:)` | 정식 명칭과 약칭 대상 검색 (D-17) | 정경 순서 정렬. 빈 질의는 전체 |

### 3.5 ScriptureReferenceFormatter

```swift
enum ScriptureReferenceFormatter {
    static func string(for value: ScriptureReferenceValue) -> String
    static func string(for values: [ScriptureReferenceValue]) -> String
}
```

| 입력 | 출력 |
|---|---|
| 절 없음 | `요한복음 3장` |
| 단일 절 | `요한복음 3:16` |
| 절 범위 | `요한복음 3:16-17` |
| 여러 참조 | `요한복음 3:16, 시편 23장` |

알 수 없는 책 식별자는 검증 단계에서 걸러지므로 여기서는 나타나지 않는다. 방어적으로
빈 문자열을 반환하고 `AppLogger`에 사건만 기록한다.

### 3.6 ShareTextComposer

```swift
enum ShareTextComposer {
    static func text(for entry: Entry,
                     dateFormatter: DateFormatter = .shareDate) -> String
}
```

**출력 구성 (FR-10)**: 날짜 / 종류 / 제목(있는 경우) / 참조 문자열(묵상) / 내용 전문.
정확한 서식은 Unit 2 Code Generation에서 확정한다.

### 3.7 BackupDocument

```swift
struct BackupDocument: Codable, Sendable {
    let schemaVersion: Int
    let exportedAt: Date
    let entries: [BackupEntryRecord]
}

struct BackupEntryRecord: Codable, Sendable {
    let id: UUID
    let kind: String
    let date: Date
    let title: String?
    let body: String
    let references: [BackupReferenceRecord]
    let createdAt: Date
    let updatedAt: Date
}

struct BackupReferenceRecord: Codable, Sendable {
    let bookID: String
    let chapter: Int
    let verseStart: Int?
    let verseEnd: Int?
    let order: Int
}
```

`kind`를 열거형이 아니라 문자열로 두는 이유: 알 수 없는 값이 들어와도 디코딩 자체가
실패하지 않고 검증 단계에서 레코드 인덱스와 함께 오류를 낼 수 있다 (SEC-05).

### 3.8 BackupCodec

```swift
enum BackupCodec {
    static let currentSchemaVersion: Int
    static let supportedSchemaVersions: Set<Int>
    static let maxFileSizeBytes: Int

    static func encode(_ entries: [Entry], exportedAt: Date) throws -> Data
    static func decode(_ data: Data) throws -> BackupDocument
    static func validate(_ document: BackupDocument) throws -> [ValidatedBackupRecord]
}

struct ValidatedBackupRecord: Sendable {
    let id: UUID
    let draft: ValidatedEntryDraft
    let createdAt: Date
    let updatedAt: Date
}
```

| 메서드 | 목적 | 실패 |
|---|---|---|
| `encode(_:exportedAt:)` | 전체 기록을 JSON으로 직렬화 (FR-11) | `BackupError.encodingFailed` |
| `decode(_:)` | 크기 상한 검사 → JSON 파싱 → 스키마 버전 확인 | `fileTooLarge`, `malformedJSON`, `unsupportedSchemaVersion` |
| `validate(_:)` | 레코드별 타입·범위·필수 필드 검증. 하나라도 실패하면 전체 실패 | `invalidRecord(index:reason:)` |

**설계 판단**: `decode`와 `validate`를 분리한다. 파싱 실패와 내용 무효는 사용자에게
다른 안내가 필요하고, 검증만 따로 단위 테스트할 수 있다.

### 3.9 BackupImporter

```swift
struct BackupImporter {
    init(store: EntryStore)

    func performImport(data: Data,
                       strategy: BackupMergeStrategy) throws -> BackupImportSummary
}

enum BackupMergeStrategy: Sendable {
    case appendMissing   // 기존 유지, 없는 식별자만 추가
    case replaceAll      // 전체 교체
}

struct BackupImportSummary: Sendable {
    let totalInFile: Int
    let inserted: Int
    let skippedDuplicates: Int
    let strategy: BackupMergeStrategy
}
```

**처리 순서 (FR-12, SEC-05, SEC-07)**

```
1. BackupCodec.decode(data)          실패 -> 중단, 저장소 변경 없음
2. BackupCodec.validate(document)    실패 -> 중단, 저장소 변경 없음
3. store.existingIDs()               중복 판별 집합 확보
4. store.applyImport(records:strategy:)  단일 트랜잭션 반영
5. 실패 시 컨텍스트 롤백, BackupError.persistenceFailed
6. 성공 시 BackupImportSummary 반환
```

1~2단계가 끝나기 전에는 저장소를 건드리지 않는다. 부분 반영이 발생할 수 없다.

### 3.10 Errors

```swift
enum EntryValidationError: Error, Equatable {
    case emptyBody
    case bodyTooLong(max: Int)
    case titleTooLong(max: Int)
    case tooManyReferences(max: Int)
    case referencesNotAllowedForPrayer
    case invalidReference(index: Int, reason: ScriptureReferenceError)
}

enum ScriptureReferenceError: Error, Equatable {
    case unknownBook
    case chapterOutOfRange(allowed: ClosedRange<Int>)
    case verseOutOfRange(allowed: ClosedRange<Int>)
    case verseEndWithoutStart
    case verseRangeReversed
}

enum BackupError: Error, Equatable {
    case fileTooLarge(maxBytes: Int)
    case notReadable
    case malformedJSON
    case unsupportedSchemaVersion(found: Int)
    case invalidRecord(index: Int, reason: BackupRecordIssue)
    case encodingFailed
    case persistenceFailed
}

enum BackupRecordIssue: Error, Equatable {
    case unknownKind(String)
    case validationFailed(EntryValidationError)
    case duplicateIDInFile
}

enum StoreError: Error, Equatable {
    case saveFailed
    case deleteFailed
    case fetchFailed
}
```

모든 오류 타입이 `Equatable`인 이유: 단위 테스트에서 실패 원인을 값으로 단정한다.
표시 문구에 의존하는 테스트를 만들지 않는다.

---

## 4. Presentation

### 4.1 EntryEditorViewModel

```swift
@Observable
@MainActor
final class EntryEditorViewModel {
    init(mode: EditorMode, store: EntryStore, draftStore: DraftStore)

    var draft: EntryDraft { get set }
    var validationError: EntryValidationError? { get }
    var canSave: Bool { get }

    func onAppear()
    func addReference(_ value: ScriptureReferenceValue)
    func removeReference(at index: Int)
    func moveReference(from: Int, to: Int)
    func save() -> Bool
    func discardDraft()
    func persistDraftIfNeeded()
}

enum EditorMode {
    case create(kind: EntryKind, date: Date)
    case edit(entry: Entry)
}
```

| 메서드 | 목적 |
|---|---|
| `onAppear()` | 초안 복원 시도 (NFR-04) 또는 편집 대상 값 적재 |
| `addReference(_:)` | 참조 추가. 개별 참조를 즉시 검증해 즉각 피드백 |
| `removeReference(at:)` | 참조 삭제 |
| `moveReference(from:to:)` | 참조 순서 변경 (표시 순서가 곧 저장 순서) |
| `save()` | 전체 검증 → 저장 → 초안 삭제. 성공 여부 반환 |
| `discardDraft()` | 사용자가 작성 취소 시 초안 제거 |
| `persistDraftIfNeeded()` | 백그라운드 전환·입력 변화 시 초안 저장 |

### 4.2 CalendarViewModel

```swift
@Observable
@MainActor
final class CalendarViewModel {
    init(store: EntryStore, calendar: Calendar = .current)

    var displayedMonth: Date { get }
    var selectedDay: Date? { get }
    var markers: [Date: DayMarker] { get }
    var entriesOnSelectedDay: [Entry] { get }
    var filter: EntryListFilter { get set }
    var loadError: StoreError? { get }

    func showMonth(containing date: Date)
    func showPreviousMonth()
    func showNextMonth()
    func showToday()
    func selectDay(_ day: Date)
    func reload()
}

struct DayMarker: Equatable, Sendable {
    let hasPrayer: Bool
    let hasDevotion: Bool
}
```

`markers`는 표시 중인 달 범위만 담는다. 월 전환 시 해당 달만 다시 조회한다 (NFR-12).

### 4.3 BackupViewModel

```swift
@Observable
@MainActor
final class BackupViewModel {
    init(store: EntryStore, importer: BackupImporter)

    var exportState: ExportState { get }
    var importState: ImportState { get }
    var pendingImportData: Data? { get }

    func prepareExport() -> Data?
    func selectImportFile(_ url: URL)
    func confirmImport(strategy: BackupMergeStrategy)
    func cancelImport()
}

enum ExportState: Equatable { case idle, preparing, ready(Data), failed(BackupError) }
enum ImportState: Equatable {
    case idle
    case awaitingStrategy
    case importing
    case completed(BackupImportSummary)
    case failed(BackupError)
}
```

`awaitingStrategy` 상태가 병합 방식 확인 단계(FR-12)를 표현한다.

### 4.4 EntryListFilter

```swift
enum EntryListFilter: String, CaseIterable, Sendable {
    case all
    case prayer
    case devotion

    var entryKinds: [EntryKind] { get }
}
```

목록 보기와 캘린더 보기가 같은 타입을 공유한다 (FR-07 필터 공통 적용).

### 4.5 ErrorMessageMapper

```swift
enum ErrorMessageMapper {
    static func message(for error: EntryValidationError) -> String
    static func message(for error: ScriptureReferenceError) -> String
    static func message(for error: BackupError) -> String
    static func message(for error: StoreError) -> String
}
```

반환 문구는 지역화 문자열이며 내부 정보를 담지 않는다 (SEC-06).

---

## 5. Support

### 5.1 AppLogger

```swift
enum AppLogger {
    static func log(_ event: AppEvent)
}

enum AppEvent {
    case entrySaved(id: UUID, kind: EntryKind)
    case entryUpdated(id: UUID)
    case entryDeleted(id: UUID)
    case validationRejected(reason: String)      // 오류 케이스 이름만
    case draftRestored(target: String)           // 대상 종류만
    case draftPersistFailed
    case backupExported(count: Int)
    case backupImportStarted(strategy: String)
    case backupImportFinished(inserted: Int, skipped: Int)
    case backupImportFailed(reason: String)      // 오류 케이스 이름만
    case storeOperationFailed(operation: String)
    case unknownBookIDEncountered
}
```

**SEC-03 준수 방식**: 사건 열거형의 연관값은 식별자, 개수, 오류 케이스 이름으로
제한한다. 기록 제목·내용·참조 문자열을 받는 케이스가 존재하지 않으므로 본문이 로그에
들어갈 경로가 컴파일 시점에 없다.
