# 애플리케이션 설계 통합본 (Application Design)

**단계**: INCEPTION / Application Design
**작성일**: 2026-08-20
**프로젝트**: 묵담 (Mukdam) — 기도제목·묵상 기록 iOS 앱

이 문서는 아래 상세 문서를 통합한 개요다. 각 항목의 근거와 세부는 링크한 문서에 있다.

- [`components.md`](components.md) — 컴포넌트 정의와 책임
- [`component-methods.md`](component-methods.md) — 메서드 시그니처와 입출력
- [`services.md`](services.md) — 서비스 조정과 주요 흐름
- [`component-dependency.md`](component-dependency.md) — 의존 관계, 통신 패턴, 데이터 흐름

---

## 1. 설계 결정 요약

| 항목 | 결정 | 출처 |
|---|---|---|
| 프레젠테이션 패턴 | 혼합. 상태 전이 있는 화면만 ViewModel, 목록 조회는 `@Query` | Q1=C |
| 폴더 구조 | 하이브리드. 기반은 계층별, 화면은 기능별 | Q2=C |
| 저장소 추상화 | 프로토콜 없음. `ModelContext`를 `EntryStore` 한 타입에 모음 | Q3=B |
| 66권 데이터 | Swift 소스에 정적 정의 | Q4=A |
| 도메인 서비스 | 역할별 전용 타입 4종 | Q5=A |
| 오류 표현 | 도메인 오류 타입 + 표시 문구 변환 분리 | Q6=A |
| 작성 중 보존 | 명시적 저장 + 초안 자동 보존 | Q7=A |
| 테스트 | 단위 Swift Testing, UI XCUITest | Q8=A |
| 타깃 이름 | `Mukdam` (표시 이름 `묵담`) | Q9=A |
| 번들 식별자 | `com.octo.mukdam` | Q10 |

---

## 2. 계층 구조

```
Presentation  Features/    SwiftUI 뷰, ViewModel, 오류 문구 변환
     |
Domain        Domain/      검증, 참조 문자열화, 공유 텍스트, 백업 코덱/임포터, 정경 데이터
     |
Persistence   Persistence/ EntryStore, 컨테이너 팩토리, 초안 저장소
     |
Models        Models/      SwiftData 모델과 값 타입

Support       Support/     AppLogger (횡단, 상위 계층 참조 없음)
```

의존은 아래 방향으로만 흐르며 순환이 없다. 검증 결과 상세는
[`component-dependency.md`](component-dependency.md) 3절의 의존 매트릭스에 있다.

---

## 3. 컴포넌트 한눈에 보기

### Models

| 컴포넌트 | 역할 |
|---|---|
| `Entry` | 기록 한 건. CloudKit 전방 호환 제약 적용 |
| `ScriptureReference` | 성경 구절 참조 한 건 |
| `EntryKind` | 종류 열거형 (prayer, devotion) |
| `ScriptureReferenceValue` | 계층 간 참조 전달용 값 타입 |

### Persistence

| 컴포넌트 | 역할 |
|---|---|
| `EntryStore` | 생성·수정·삭제, 월 범위 집계, 백업 반영. 트랜잭션 경계 |
| `ModelContainerFactory` | 앱용·테스트용 컨테이너 생성, 파일 보호 속성 적용 |
| `DraftStore` | 작성 중 초안 보존 |

### Domain

| 컴포넌트 | 역할 |
|---|---|
| `EntryValidator` | 입력 검증과 정규화 |
| `EntryLimits` | 길이·개수·범위 상한 상수 |
| `BibleCanon` / `BibleBook` | 66권 정적 데이터와 약칭 검색 |
| `ScriptureReferenceFormatter` | 참조 문자열화 |
| `ShareTextComposer` | 공유 텍스트 조립 |
| `BackupCodec` | 백업 인코딩·디코딩·검증 |
| `BackupImporter` | 가져오기 파이프라인과 롤백 |
| Errors 4종 | 실패 원인을 값으로 표현 |

### Presentation

| 컴포넌트 | ViewModel | 역할 |
|---|---|---|
| `RootTabView` | 없음 | 2탭 구성, 백업 진입점 배치 |
| `TodayView` | 없음 | 오늘 기록 + 작성 버튼 2개 |
| `EntryListView` | 없음 | 전체 목록, 필터, 보기 전환 |
| `EntryDetailView` | 없음 | 상세, 수정 진입, 삭제, 공유 |
| `EntryEditorView` | `EntryEditorViewModel` | 작성·수정, 검증 표시, 참조 편집, 초안 |
| `ScripturePickerView` | 없음 | 66권 선택과 약칭 검색 |
| `CalendarView` | `CalendarViewModel` | 월 격자, 표식, 날짜 선택 |
| `BackupView` | `BackupViewModel` | 내보내기, 가져오기 다단계 흐름 |
| `ErrorMessageMapper` | — | 도메인 오류 → 사용자 문구 |

---

## 4. 핵심 흐름 요약

상세 흐름도는 [`services.md`](services.md) 2절에 있다.

| 흐름 | 요약 |
|---|---|
| 기록 저장 | 입력 → `EntryValidator` → `ValidatedEntryDraft` → `EntryStore` → 초안 삭제 |
| 초안 보존 | 입력 변화·백그라운드 전환 시 `DraftStore` 저장, 진입 시 복원, 저장 성공 시 삭제 |
| 목록 조회 | `@Query`로 직접 조회. 필터는 프레디케이트에 반영 |
| 캘린더 | 표시 월 범위만 조회해 `[Date: DayMarker]` 생성. 월 전환 시 재조회 |
| 공유 | `ShareTextComposer` 순수 함수 → 공유 시트 |
| 백업 내보내기 | 전체 조회 → JSON 인코딩 → 사용자 위치 저장. 평문 안내 표시 |
| 백업 가져오기 | 디코딩 → 검증 → (여기까지 저장소 무변경) → 단일 트랜잭션 반영 → 실패 시 롤백 |

---

## 5. 요구사항 커버리지

FR·SEC·NFR과 담당 컴포넌트의 대응은 [`components.md`](components.md) 8절에 표로 있다.
누락 없이 FR-01~FR-13, SEC-01~SEC-11이 배정되었다.

특히 다음 세 요구사항은 구조로 보장한다.

1. **SEC-04 (검증 우회 금지)** — `ValidatedEntryDraft`의 초기화를 `EntryValidator`만
   호출할 수 있게 제한한다. 검증하지 않은 값이 저장소에 도달하는 경로가 타입 수준에서
   존재하지 않는다.
2. **SEC-03 (콘텐츠 미로깅)** — `AppLogger`가 자유 문자열을 받지 않고 사건 열거형만
   받는다. 본문을 로그에 넣을 방법이 컴파일 시점에 없다.
3. **FR-12 (전부 성공 또는 전부 실패)** — 검증 완료 전 저장소를 건드리지 않고, 반영은
   단일 트랜잭션이며 실패 시 롤백한다.

---

## 6. CloudKit 전방 호환 제약

다음 버전에 iCloud 동기화를 추가하기로 했으므로(D-18) MVP 모델부터 아래를 지킨다.

| 제약 | 적용 방식 |
|---|---|
| 모든 저장 프로퍼티에 기본값 또는 옵셔널 | `Entry`, `ScriptureReference` 전 프로퍼티에 기본값 부여 |
| 유니크 제약 사용 금지 | `@Attribute(.unique)` 미사용. `id` 유일성은 저장 시 조회로 보장 |
| 관계는 옵셔널 + 역참조 | `references: [ScriptureReference]?`, `entry: Entry?` |
| 열거형 직접 저장 회피 | `kindRawValue: String` 저장, `kind`는 계산 프로퍼티 |

이 제약을 어기면 다음 버전에서 스키마 마이그레이션이 필요하다. Unit 1 Functional
Design에서 다시 검증한다.

---

## 7. 설계 단계에서 내린 판단과 근거

| 판단 | 근거 |
|---|---|
| 백업 화면을 `기록` 탭 툴바 메뉴에 배치 | 탭 2개 결정(D-10, D-27) 유지. 백업은 상시 노출이 필요한 기능이 아님 |
| `EntryStore`에 목록 조회 메서드를 두지 않음 | `@Query`와 중복되면 읽기 경로가 두 개로 갈라짐 |
| `DraftStore`를 `UserDefaults`로 구현하지 않음 | 초안에 기도제목 본문이 담기므로 파일 보호 속성 적용 가능한 파일 저장이 적절 (SEC-01) |
| 초안을 SwiftData에 저장하지 않음 | 저장되지 않은 초안이 목록에 나타나면 안 됨 |
| `BackupCodec`의 `decode`와 `validate` 분리 | 파싱 실패와 내용 무효는 사용자 안내가 다르고, 검증만 따로 테스트 가능 |
| `BackupEntryRecord.kind`를 문자열로 정의 | 알 수 없는 값이 와도 디코딩이 통째로 실패하지 않고 레코드 인덱스와 함께 오류 보고 가능 |
| 오류 타입을 `Equatable`로 정의 | 단위 테스트가 표시 문구가 아니라 오류 값을 단정하게 함 |
| 알림 체계(NotificationCenter 등) 미도입 | 데이터 변경 전파는 SwiftData와 `@Query`가 담당. 경로를 하나로 유지 |
| `EntryLimits`를 주입 가능한 값으로 정의 | 경계값 테스트에서 작은 상한으로 대체 가능 |
| `EntryValidator`에 `Calendar` 주입 | 시간대 의존 동작을 테스트에서 고정 |

---

## 8. 미확정 사항 (Functional Design에서 확정)

- [ ] `EntryLimits` 구체 수치 확정 (제안값은 `component-methods.md` 3.3절)
- [ ] 데이터 보호 속성 수준 선택 (`.complete` vs `.completeUnlessOpen`)
- [ ] 백업 파일 크기 상한 구체 수치
- [ ] 백업 스키마 버전 정책 (호환 범위, 상위 버전 발견 시 동작)
- [ ] 공유 텍스트 정확한 서식
- [ ] 백업 인코딩·디코딩의 비동기 처리 방식
- [ ] 66권 약칭 목록 확정 (검색 대상 문자열)
- [ ] 캘린더 월 범위 프레디케이트 구현 방식 (날짜 경계 처리)

---

## 9. 보안 규칙 준수 요약 (Application Design 단계)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-01 | 준수 | `ModelContainerFactory`, `DraftStore`에 파일 보호 속성 적용 책임 배정. 수준 선택은 NFR Design에서 확정 |
| SECURITY-02 | N/A | 네트워크 중개 장비 없음 |
| SECURITY-03 | 준수 | `AppLogger`가 사건 열거형만 수용. 콘텐츠 로깅 경로 부재 |
| SECURITY-04 | N/A | HTML 제공 엔드포인트 없음 |
| SECURITY-05 | 준수 | `EntryValidator`(사용자 입력), `BackupCodec`(외부 파일) 두 검증 지점 배치. 크기 상한·타입·범위 검증 책임 명시 |
| SECURITY-06 | N/A | 클라우드 IAM 리소스 없음 |
| SECURITY-07 | N/A | 네트워크 리소스 없음 |
| SECURITY-08 | N/A | 서버 엔드포인트 없음, 단일 사용자, 타인 데이터 개념 없음 |
| SECURITY-09 | 준수 | `ErrorMessageMapper`로 내부 정보 노출 차단. 기본 자격 증명 없음 |
| SECURITY-10 | 준수 | 서드파티 의존성 없는 설계. 전부 표준 프레임워크 |
| SECURITY-11 | 준수 | 계층 분리로 보안 관련 로직(검증·직렬화·저장) 격리. 오용 시나리오 7건을 `services.md` 2.7절에 명시 |
| SECURITY-12 | 부분 적용 | 인증 기능 없음. 자격 증명 하드코딩 대상 코드 부재 |
| SECURITY-13 | 준수 | 백업 역직렬화를 신뢰 불가 입력으로 취급. `decode` → `validate` 2단계 방어 |
| SECURITY-14 | N/A | 중앙 로그·알림 인프라 없음 |
| SECURITY-15 | 준수 | 모든 실패를 값 또는 `throws`로 표현. 저장 실패 시 롤백(fail closed). 초안 저장 실패는 흐름 차단하지 않음 |

**Blocking Security Findings**: 없음
