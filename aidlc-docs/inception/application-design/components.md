# 컴포넌트 정의 (Components)

**단계**: INCEPTION / Application Design
**작성일**: 2026-08-20
**설계 결정 근거**: `aidlc-docs/inception/plans/application-design-plan.md` Q1~Q10

---

## 1. 계층 구조

의존은 위에서 아래로만 흐른다. 아래 계층은 위 계층을 알지 못한다.

```
Presentation  (Features/)      SwiftUI 뷰 + ViewModel + 오류 문구 변환
      |
      v
Domain        (Domain/)        검증, 참조 문자열화, 공유 텍스트, 백업 코덱/임포터, 성경 정경 데이터
      |
      v
Persistence   (Persistence/)   SwiftData 접근 전용 타입, 컨테이너 구성, 초안 저장소
      |
      v
Models        (Models/)        SwiftData @Model 타입과 값 타입
```

**Support** (`Support/`)는 계층에 속하지 않는 횡단 유틸리티다. 로깅만 포함하며 어떤
계층에서도 호출할 수 있다. 단 상위 계층 타입을 참조하지 않는다.

---

## 2. 폴더 구조 (Q2=C 하이브리드)

```
Mukdam/
  MukdamApp.swift
  Models/
    Entry.swift
    ScriptureReference.swift
    EntryKind.swift
    ScriptureReferenceValue.swift
  Persistence/
    EntryStore.swift
    ModelContainerFactory.swift
    DraftStore.swift
  Domain/
    Validation/
      EntryDraft.swift
      ValidatedEntryDraft.swift
      EntryValidator.swift
      EntryLimits.swift
    Scripture/
      BibleBook.swift
      BibleCanon.swift
      ScriptureReferenceFormatter.swift
    Sharing/
      ShareTextComposer.swift
    Backup/
      BackupDocument.swift
      BackupCodec.swift
      BackupImporter.swift
      BackupMergeStrategy.swift
      BackupImportSummary.swift
    Errors/
      EntryValidationError.swift
      ScriptureReferenceError.swift
      BackupError.swift
      StoreError.swift
  Features/
    Root/
      RootTabView.swift
    Today/
      TodayView.swift
    EntryList/
      EntryListView.swift
      EntryListFilter.swift
      EntryRowView.swift
    Calendar/
      CalendarView.swift
      CalendarViewModel.swift
      CalendarMonth.swift
      DayMarker.swift
    EntryDetail/
      EntryDetailView.swift
    EntryEditor/
      EntryEditorView.swift
      EntryEditorViewModel.swift
      ScripturePickerView.swift
      ScriptureReferenceEditorView.swift
    Backup/
      BackupView.swift
      BackupViewModel.swift
    Shared/
      ErrorMessageMapper.swift
      EmptyStateView.swift
  Support/
    AppLogger.swift
  Resources/
    Localizable.xcstrings
```

작업 단위와의 대응은 다음과 같다.

| 폴더 | 담당 단위 |
|---|---|
| `Models/`, `Persistence/`, `Domain/Validation/`, `Domain/Scripture/`, `Domain/Errors/`, `Support/` | Unit 1 |
| `Features/Root/`, `Features/Today/`, `Features/EntryList/`, `Features/EntryDetail/`, `Features/EntryEditor/`, `Domain/Sharing/`, `Features/Shared/` | Unit 2 |
| `Features/Calendar/`, `Features/Backup/`, `Domain/Backup/` | Unit 3 |

---

## 3. Models 계층

### 3.1 Entry

**목적**: 기도제목 또는 묵상 한 건을 저장하는 SwiftData 모델.

**책임**
- 기록의 영속 상태 보관 (종류, 날짜, 제목, 내용, 참조, 생성·수정 시각)
- 참조 컬렉션과의 관계 소유

**인터페이스**: SwiftData `@Model` 저장 프로퍼티와 초기화. 비즈니스 로직을 갖지 않는다.

**설계 제약 (CloudKit 전방 호환)**
- 모든 저장 프로퍼티에 기본값을 부여하거나 옵셔널로 선언한다.
- `@Attribute(.unique)`를 사용하지 않는다. `id`의 유일성은 저장 시점 조회로 보장한다.
- `references` 관계는 옵셔널 배열로 선언하고 역참조(`entry`)를 정의한다.
- 삭제 규칙은 cascade로 지정해 기록 삭제 시 참조가 함께 제거되도록 한다.

### 3.2 ScriptureReference

**목적**: 묵상에 붙는 성경 구절 참조 한 건을 저장하는 SwiftData 모델.

**책임**
- 책 식별자, 장, 절 범위, 표시 순서 보관
- 소유 기록에 대한 역참조 유지

**인터페이스**: `@Model` 저장 프로퍼티. 문자열화는 Domain의 Formatter가 담당한다.

### 3.3 EntryKind

**목적**: 기록 종류 열거형 (`prayer`, `devotion`).

**책임**
- 종류 구분값 제공
- 백업 직렬화를 위한 안정적 문자열 표현 제공 (`Codable`, `RawValue`는 영문 고정)

**주의**: 화면 표시용 한글 명칭은 이 타입이 아니라 Presentation 계층에서 지역화
문자열로 처리한다. 모델에 표시 문자열을 넣으면 지역화와 저장이 결합된다.

### 3.4 ScriptureReferenceValue

**목적**: 참조를 계층 간에 전달하기 위한 값 타입. 저장 모델과 검증 결과, 백업 레코드,
문자열화 입력이 모두 같은 형태를 공유한다.

**책임**
- `bookID`, `chapter`, `verseStart`, `verseEnd` 보관
- `ScriptureReference` 모델과의 상호 변환 제공

**존재 이유**: 모델을 도메인 함수에 직접 넘기면 순수 함수 테스트를 위해 SwiftData
컨테이너가 필요해진다. 값 타입을 경유하면 Formatter와 Validator를 컨테이너 없이
테스트할 수 있다 (NFR-08).

---

## 4. Persistence 계층

### 4.1 EntryStore

**목적**: SwiftData `ModelContext` 접근을 한곳에 모으는 전용 타입 (Q3=B).

**책임**
- 기록 생성, 수정, 삭제
- 캘린더용 월 범위 집계 조회 (NFR-12)
- 백업 내보내기용 전체 조회, 가져오기용 일괄 반영
- 실패 시 컨텍스트 롤백 (SEC-07)

**책임이 아닌 것**
- 목록 화면용 일반 조회. 목록 화면은 SwiftUI `@Query`를 직접 사용한다 (Q1=C).
- 입력 검증. 검증된 값만 받는다.

**인터페이스**: `@MainActor` 클래스. `ModelContext`를 주입받는다. 실패는 `StoreError`로
던진다.

**존재 이유**: 프로토콜 추상화를 두지 않는 대신(Q3=B), 저장소 접근 지점을 한 타입에
모아 다음 버전의 CloudKit 전환 시 수정 범위를 국소화한다.

### 4.2 ModelContainerFactory

**목적**: 앱용 컨테이너와 테스트용 인메모리 컨테이너 생성.

**책임**
- 스키마 정의와 저장소 URL 결정
- 저장 파일에 데이터 보호 속성 적용 (SEC-01)
- 테스트용 인메모리 컨테이너 제공 (Q3=B의 테스트 전략 전제)

**인터페이스**: 정적 팩토리 메서드 2개 (`makeAppContainer`, `makeInMemoryContainer`).

### 4.3 DraftStore

**목적**: 작성 중 내용을 앱 종료 후에도 복원할 수 있게 보관 (Q7=A, NFR-04).

**책임**
- 편집 대상별 초안 저장, 로드, 삭제
- 저장 파일에 데이터 보호 속성 적용 (SEC-01)

**인터페이스**: 초안 저장·로드·삭제 3개 메서드. 저장 매체는 Application Support
디렉터리의 JSON 파일.

**설계 판단**: `UserDefaults`를 쓰지 않는다. 초안에는 기도제목 본문이 담기므로
SEC-01의 파일 보호 속성을 적용할 수 있는 파일 저장이 적절하다. 정식 기록과 섞이지
않도록 SwiftData 저장소에도 넣지 않는다. 빈 기록이 목록에 나타나면 안 된다.

---

## 5. Domain 계층

### 5.1 EntryDraft / ValidatedEntryDraft

**목적**: 검증 전 입력값과 검증 통과값을 타입으로 구분한다.

**책임**
- `EntryDraft`: 화면에서 들어온 원본 입력 보관 (종류, 날짜, 제목, 내용, 참조 입력 배열)
- `ValidatedEntryDraft`: 정규화 완료값 보관 (날짜 일 단위 정규화, 제목 공백 → nil,
  내용 트림, 참조 검증 완료)

**존재 이유**: `EntryStore`가 검증되지 않은 값을 받을 수 없게 타입으로 강제한다.
검증을 우회한 저장 경로가 생기지 않는다 (SEC-04).

### 5.2 EntryValidator

**목적**: 입력 검증과 정규화를 담당하는 순수 타입 (SEC-04).

**책임**
- 내용 비어 있음 검사 (트림 후 1자 이상)
- 내용·제목 최대 길이 검사
- 참조 개수 상한 검사
- 참조별 검증: 책 식별자 존재, 장 범위, 절 범위, 끝 절 단독 입력 금지, 역순 범위 금지
- 날짜 일 단위 정규화, 제목 공백 정규화

**인터페이스**: 정적 메서드 하나. `Result<ValidatedEntryDraft, EntryValidationError>` 반환.
예외를 던지지 않고 값으로 반환해 화면이 실패 원인을 그대로 표현할 수 있게 한다.

### 5.3 EntryLimits

**목적**: 검증 상한값을 한곳에 모은 상수 집합.

**책임**: 내용 최대 길이, 제목 최대 길이, 참조 최대 개수, 장 상한, 절 상한 제공.

**주의**: 장·절 상한은 책별 실제 값이 아니라 전역 방어 상한이다. 책별 검증을 하지
않는다는 요구사항 결정(요구사항 명세 4.2절)을 유지한다.

### 5.4 BibleBook / BibleCanon

**목적**: 성경 66권 정적 데이터와 검색 (FR-04, Q4=A).

**책임**
- `BibleBook`: 식별자, 정식 명칭, 약칭 배열, 정경 순서, 구약/신약 구분
- `BibleCanon`: 66권 배열 제공, 식별자로 조회, 명칭·약칭 검색 (D-17)

**인터페이스**: Swift 소스에 정적 배열로 정의. 검색은 정경 순서로 정렬된 결과 반환.

**책임이 아닌 것**: 성경 본문 제공. 본문 텍스트는 어떤 형태로도 포함하지 않는다 (D-05).

### 5.5 ScriptureReferenceFormatter

**목적**: 참조를 사람이 읽는 문자열로 변환 (FR-05).

**책임**
- 절 없음 → `요한복음 3장`
- 단일 절 → `요한복음 3:16`
- 절 범위 → `요한복음 3:16-17`
- 여러 참조 → 순서대로 `, ` 연결

**인터페이스**: 값 타입 입력, 문자열 출력. 정적 메서드 2개(단건, 복수).

### 5.6 ShareTextComposer

**목적**: 기록 1건을 공유용 텍스트로 구성 (FR-10).

**책임**: 날짜, 종류, 제목, 참조 문자열, 내용 전문을 정해진 형식으로 조립.

**인터페이스**: 기록 스냅샷 입력, 문자열 출력. 순수 함수.

### 5.7 BackupDocument / BackupCodec

**목적**: 백업 파일 형식 정의와 인코딩·디코딩 (FR-11, FR-12, Q11=A JSON).

**책임**
- `BackupDocument`: 스키마 버전, 내보낸 시각, 기록 레코드 배열
- `BackupCodec`: 인코딩, 디코딩, 스키마 버전 확인, 파일 크기 상한 검사, 레코드 검증

**보안 책임 (SEC-05, SEC-13)**: 가져오기 입력을 신뢰하지 않는다. 크기 상한 초과, JSON
파싱 실패, 스키마 버전 미지원, 필수 필드 누락, 타입 불일치, 값 범위 초과를 모두
차단하고 `BackupError`로 실패 원인을 표현한다.

### 5.8 BackupImporter

**목적**: 검증 통과 레코드를 저장소에 반영하는 오케스트레이션 (FR-12).

**책임**
- 병합 방식 적용 (기존 유지 추가 / 전체 교체)
- 기존 식별자와 대조해 중복 삽입 방지
- 전량 검증 후 일괄 반영. 실패 시 전체 롤백 (SEC-07)
- 반영 결과 요약 반환

**인터페이스**: `EntryStore` 주입. `import(data:strategy:)` 메서드가
`BackupImportSummary` 반환 또는 `BackupError` 던짐.

### 5.9 Errors

**목적**: 도메인 실패 원인을 값으로 표현 (Q6=A).

**구성**
- `EntryValidationError`: 내용 없음, 내용 길이 초과, 제목 길이 초과, 참조 개수 초과,
  참조 오류(인덱스 + 원인)
- `ScriptureReferenceError`: 알 수 없는 책, 장 범위 초과, 끝 절 단독 입력, 절 역순,
  절 범위 초과
- `BackupError`: 파일 과대, 읽기 실패, JSON 손상, 미지원 스키마 버전, 레코드 무효,
  저장 실패
- `StoreError`: 저장 실패, 삭제 실패, 조회 실패

**주의**: 이 타입들은 사용자 표시 문자열을 갖지 않는다. 문구 변환은 Presentation의
`ErrorMessageMapper`가 담당한다 (SEC-06).

---

## 6. Presentation 계층

### 6.1 RootTabView

**목적**: 2탭 구조 구성 (`오늘`, `기록`).

**책임**: 탭 정의와 각 탭의 `NavigationStack` 구성. 백업 화면 진입점 배치.

**설계 판단**: 설정 탭을 만들지 않는다. 백업 화면은 `기록` 탭 툴바의 추가 메뉴에서
진입한다. 탭이 2개라는 결정(D-10, D-27)을 유지하면서 백업 진입점을 확보하기 위한
선택이며, 백업은 자주 쓰는 기능이 아니라 상시 노출이 필요하지 않다.

### 6.2 TodayView

**목적**: 오늘 화면 (FR-06, FR-01).

**책임**
- `@Query`로 오늘 날짜 기록 조회 및 표시
- `기도제목 작성`, `묵상 작성` 두 버튼 제공
- 빈 상태 표시

**ViewModel 없음**: 조건에 맞는 데이터를 나열하기만 하므로 `@Query` 직접 사용 (Q1=C).

### 6.3 EntryListView

**목적**: 기록 탭 (FR-07).

**책임**
- `@Query`로 전체 기록 조회, 날짜별 그룹화, 날짜 역순 정렬
- 종류 필터 제공 및 상태 유지
- 목록 / 캘린더 보기 전환 제공
- 백업 화면 진입점 노출

**ViewModel 없음**: 필터와 보기 방식은 단일 값 상태이므로 뷰 상태로 충분하다.

### 6.4 CalendarView / CalendarViewModel

**목적**: 월간 캘린더 보기 (FR-13).

**책임**
- `CalendarViewModel`: 표시 월, 선택 날짜, 월별 표식 데이터, 필터 반영, 월 이동,
  오늘로 이동
- `CalendarView`: 월 격자 렌더링, 종류별 표식 표시, 날짜 선택, 선택 날짜 기록 목록

**ViewModel 필요**: 표시 월과 선택 날짜라는 상태 전이가 있고, 월 변경마다 범위 조회를
수행해야 한다 (NFR-12). Q1=C 기준에 해당한다.

**접근성 책임**: 표식을 색상만으로 구분하지 않고, 날짜 접근성 라벨에 기록 유무와 종류를
포함한다 (NFR-06).

### 6.5 EntryDetailView

**목적**: 기록 상세 (FR-08, FR-09, FR-10).

**책임**: 전체 내용 표시, 수정 화면 진입, 삭제 확인 및 실행, 공유 시트 호출.

**ViewModel 없음**: 표시 중심이며 삭제·공유는 단일 동작이다.

### 6.6 EntryEditorView / EntryEditorViewModel

**목적**: 기록 작성·수정 (FR-02, FR-03, FR-08).

**책임**
- `EntryEditorViewModel`: 초안 상태 보관, 참조 추가·삭제, 검증 실행 및 결과 보관,
  저장, 초안 자동 보존과 복원, 저장 성공 시 초안 삭제
- `EntryEditorView`: 입력 폼, 검증 실패 표시, 참조 목록 편집
- `ScripturePickerView`: 66권 선택과 약칭 검색 (FR-04)
- `ScriptureReferenceEditorView`: 장·절 입력

**ViewModel 필요**: 검증 결과 표현, 참조 컬렉션 편집, 초안 보존이라는 상태 전이가 있다.

### 6.7 BackupView / BackupViewModel

**목적**: 백업 내보내기·가져오기 (FR-11, FR-12).

**책임**
- `BackupViewModel`: 내보내기 데이터 생성, 파일 선택 결과 처리, 병합 방식 선택 상태,
  가져오기 실행, 진행·결과·실패 상태 보관
- `BackupView`: 내보내기 버튼과 평문 안내(SEC-02), 가져오기 흐름, 결과 요약 표시

**ViewModel 필요**: 파일 선택 → 병합 방식 확인 → 검증 → 반영 → 결과 표시의 다단계
흐름이다.

### 6.8 ErrorMessageMapper

**목적**: 도메인 오류를 사용자 문구로 변환 (Q6=A, SEC-06).

**책임**
- `EntryValidationError`, `ScriptureReferenceError`, `BackupError`, `StoreError`를
  지역화된 일반 문구로 변환
- 내부 경로, 스택, 원시 오류 설명을 노출하지 않음

---

## 7. Support

### 7.1 AppLogger

**목적**: 진단 로깅 (SEC-03).

**책임**
- 사건 종류와 식별자만 기록. 기록 제목·내용·참조를 로그에 넣지 않음
- 로깅 대상 사건을 열거형으로 고정해 자유 문자열 로깅을 막음

**인터페이스**: 사건 열거형을 받는 단일 로깅 함수. `os.Logger` 기반.

**설계 판단**: 문자열을 받는 범용 로깅 API를 두지 않는다. 범용 API가 있으면 언젠가
본문이 로그에 들어간다. 사건 열거형으로 제한하면 컴파일 시점에 막힌다.

---

## 8. 요구사항 커버리지

| 요구사항 | 담당 컴포넌트 |
|---|---|
| FR-01 | TodayView, RootTabView |
| FR-02 | EntryEditorView/ViewModel, EntryValidator, EntryStore, DraftStore |
| FR-03 | EntryEditorView/ViewModel, ScripturePickerView, ScriptureReferenceEditorView, EntryValidator |
| FR-04 | BibleCanon, BibleBook, ScripturePickerView |
| FR-05 | ScriptureReferenceFormatter |
| FR-06 | TodayView |
| FR-07 | EntryListView, EntryListFilter |
| FR-08 | EntryDetailView, EntryEditorView/ViewModel, EntryStore |
| FR-09 | EntryDetailView, EntryStore |
| FR-10 | ShareTextComposer, EntryDetailView |
| FR-11 | BackupCodec, BackupView/ViewModel, EntryStore |
| FR-12 | BackupCodec, BackupImporter, BackupView/ViewModel, EntryStore |
| FR-13 | CalendarView/ViewModel, CalendarMonth, DayMarker, EntryStore |
| SEC-01 | ModelContainerFactory, DraftStore |
| SEC-02 | BackupView |
| SEC-03 | AppLogger |
| SEC-04 | EntryValidator, EntryLimits, ValidatedEntryDraft |
| SEC-05 | BackupCodec, BackupImporter |
| SEC-06 | ErrorMessageMapper |
| SEC-07 | EntryStore, BackupImporter |
| SEC-08 | 의존성 없음 원칙 (Code Generation 단계 적용) |
| SEC-09 | 자격 증명 미사용 (해당 코드 없음) |
| SEC-10 | 계층 구조 전체 (1절) |
| SEC-11 | BackupCodec, BackupImporter (조작된 백업 파일 오용 시나리오) |
| NFR-04 | DraftStore, EntryEditorViewModel |
| NFR-06 | CalendarView, EntryDetailView, EntryListView |
| NFR-08 | 값 타입 경유 설계, 순수 도메인 타입 |
| NFR-12 | EntryStore 월 범위 집계, CalendarViewModel |
