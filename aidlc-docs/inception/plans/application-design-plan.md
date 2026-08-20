# Application Design 계획

**단계**: INCEPTION / Application Design
**작성일**: 2026-08-20
**입력**: `aidlc-docs/inception/requirements/requirements.md`,
`aidlc-docs/inception/plans/execution-plan.md`

이 문서는 두 부분으로 구성됩니다.

- **Part A**: 설계 작업 체크리스트 (질문 답변 후 실행)
- **Part B**: 설계 결정 질문 (지금 답변 필요)

Part B의 `[Answer]:` 뒤에 알파벳 하나를 적어주세요. 보기가 맞지 않으면 마지막
`Other`를 고르고 원하는 내용을 적으시면 됩니다. 각 질문에는 제 권장안과 근거를
붙였습니다. 답변이 끝나면 채팅으로 알려주세요.

---

# Part A: 설계 작업 체크리스트

## A.1 컴포넌트 식별

- [x] 기능 영역별 컴포넌트 도출 (모델, 영속화, 도메인, 프레젠테이션)
- [x] 컴포넌트별 책임 정의
- [x] 컴포넌트 인터페이스 정의
- [x] 계층 경계 및 의존 방향 확정 (SEC-10 준수 확인)

## A.2 필수 산출물 생성

- [x] `application-design/components.md` — 컴포넌트 정의와 책임
- [x] `application-design/component-methods.md` — 메서드 시그니처와 입출력 타입
- [x] `application-design/services.md` — 서비스 정의와 오케스트레이션 패턴
- [x] `application-design/component-dependency.md` — 의존 관계 매트릭스, 통신 패턴, 데이터 흐름
- [x] `application-design/application-design.md` — 위 문서 통합본
- [x] 설계 완전성 및 일관성 검증

## A.3 요구사항 커버리지 확인

- [x] FR-01 ~ FR-13 각각을 담당 컴포넌트에 매핑 (`components.md` 8절)
- [x] SEC-01 ~ SEC-11 중 설계 단계에서 반영해야 하는 항목 배치
- [x] NFR-08(테스트 가능 구조), NFR-12(캘린더 조회 범위 제한) 설계 반영
- [x] CloudKit 전방 호환 제약을 모델 설계에 명시 (`application-design.md` 6절)

## A.4 검증

- [x] 의존 방향에 순환이 없는지 확인 (`component-dependency.md` 3절 매트릭스)
- [x] 도메인 로직이 UI 타입에 의존하지 않는지 확인 (Domain → Presentation 의존 없음)
- [x] 단위 테스트 대상 경계가 명확한지 확인 (`component-dependency.md` 6절)
- [x] 다이어그램 문법 검증 (Mermaid 노드 ID 영숫자, 라벨 인용, ASCII 기본 문자, 텍스트 대안 포함)

---

# Part B: 설계 결정 질문

## Question 1: 프레젠테이션 아키텍처 패턴

SwiftUI 화면과 데이터를 연결하는 방식입니다.

A) **MV 패턴** — 뷰에서 SwiftData `@Query`로 직접 조회하고, 검증·문자열화 같은 로직만
   순수 타입으로 분리한다. 코드량이 가장 적다.

B) **MVVM 전면 적용** — 모든 화면에 `@Observable` ViewModel을 두고 저장소를 주입받아
   조회와 변경을 담당한다. 일관성이 높지만 단순 목록 화면에도 계층이 하나 늘어난다.

C) **혼합 (기준 명시)** — 아래 기준으로 나눈다.
   - **ViewModel을 둔다**: 여러 단계 상태 전이나 검증 결과를 화면에 표현해야 하는 경우
     → 기록 작성·수정 화면, 백업 내보내기·가져오기 화면, 캘린더(선택 날짜·표시 월 상태)
   - **`@Query`를 직접 쓴다**: 저장된 데이터를 조건에 맞게 나열하기만 하는 경우
     → 오늘 화면 목록, 기록 탭 목록, 상세 화면

X) Other (please describe after [Answer]: tag below)

> **권장: C**. A는 작성 화면의 검증 상태와 백업 가져오기의 다단계 흐름(파일 선택 →
> 병합 방식 확인 → 검증 → 반영)을 뷰 안에 밀어넣게 되어 단위 테스트가 어려워집니다
> (NFR-08 위반 위험). B는 단순 목록 화면에서 `@Query`가 제공하는 자동 갱신을 직접
> 다시 구현하는 셈입니다. C는 두 문제를 피하면서 경계 기준이 명확합니다.

[Answer]:C

## Question 2: 소스 폴더 구조

A) **레이어별** — `Models/`, `Persistence/`, `Domain/`, `Views/`, `Resources/`

B) **기능별** — `Entries/`, `Calendar/`, `Backup/`, `Scripture/`, `Shared/`

C) **하이브리드** — 공통 기반은 레이어별(`Models/`, `Persistence/`, `Domain/`),
   화면은 기능별(`Features/Today/`, `Features/EntryList/`, `Features/Calendar/`,
   `Features/Backup/`)

X) Other (please describe after [Answer]: tag below)

> **권장: C**. 작업 단위(Unit 1 기반 / Unit 2 화면 / Unit 3 캘린더·백업)와 폴더가
> 그대로 대응되어 단위별 코드 생성 시 어디에 무엇이 추가되는지 명확합니다. A는 화면이
> 늘어날 때 `Views/`가 평평하게 비대해지고, B는 모델과 저장소를 어느 기능에 둘지
> 애매해집니다.

[Answer]:C

## Question 3: 저장소 계층 추상화

단위 테스트를 위해 저장소를 프로토콜로 추상화할지 여부입니다.

A) **프로토콜 추상화** — `EntryStore` 프로토콜을 정의하고 SwiftData 구현체를 주입한다.
   테스트에서는 가짜(fake) 구현으로 대체한다.

B) **SwiftData 직접 사용** — `ModelContext`를 그대로 사용하고, 테스트에서는 인메모리
   `ModelContainer`를 만들어 실제 저장 동작을 검증한다.

X) Other (please describe after [Answer]: tag below)

> **권장: B**. SwiftData는 인메모리 컨테이너를 공식 지원하므로 프로토콜 없이도 빠른
> 단위 테스트가 가능합니다. 가짜 구현을 쓰면 실제 쿼리·정렬·관계 동작을 검증하지
> 못해 통과하는 테스트와 깨지는 앱이 갈라질 수 있습니다. 검증·문자열화·백업 직렬화는
> 어차피 저장소와 무관한 순수 타입이라 추상화 없이도 테스트됩니다. 다만 다음 버전에서
> CloudKit을 붙일 때 저장소 접근 지점이 흩어져 있으면 곤란하므로, 프로토콜 대신
> **저장소 접근을 전용 타입 한 곳에 모으는 것**은 어느 쪽을 선택해도 적용합니다.

[Answer]:B

## Question 4: 성경 66권 정적 데이터 저장 방식

A) **Swift 소스에 정적 데이터로 작성** — 컴파일 타임에 존재가 보장되고 파일 I/O가 없다.
   수정하려면 코드를 고쳐야 한다.

B) **앱 번들에 JSON 리소스로 포함하고 시작 시 로드** — 데이터만 따로 고칠 수 있지만
   로드 실패 처리 경로가 추가된다.

X) Other (please describe after [Answer]: tag below)

> **권장: A**. 66권 목록은 변하지 않는 데이터입니다. B를 택하면 "리소스 로드 실패"라는
> 실패 경로가 생기고, 그 경우 앱이 무엇을 해야 하는지 정의해야 합니다. 이득 없이
> 복잡도만 늘어납니다.

[Answer]:A

## Question 5: 도메인 서비스 구성 단위

검증, 참조 문자열화, 공유 텍스트 생성, 백업 직렬화를 어떻게 묶을지입니다.

A) **역할별 전용 타입** — `EntryValidator`, `ScriptureReferenceFormatter`,
   `ShareTextComposer`, `BackupCodec`을 각각 둔다.

B) **단일 파사드** — `EntryService` 하나에 모든 도메인 동작을 모은다.

X) Other (please describe after [Answer]: tag below)

> **권장: A**. 네 로직은 서로 의존하지 않고 입력·출력이 명확합니다. 분리하면 각각을
> 순수 함수 수준으로 테스트할 수 있고, 단위별 코드 생성 범위와도 맞습니다
> (Validator·Formatter는 Unit 1, ShareTextComposer는 Unit 2, BackupCodec은 Unit 3).
> B는 Unit 1에서 만든 타입을 Unit 3에서 계속 수정하게 됩니다.

[Answer]:A

## Question 6: 오류 표현 방식

A) **도메인 오류 타입 + 표시 문자열 매핑 분리** — `EntryError`, `BackupError` 같은
   열거형으로 실패 원인을 표현하고, 사용자 문구 변환은 프레젠테이션 계층에서 담당한다.

B) **화면에서 문자열 직접 처리** — 실패 시점에 바로 표시 문구를 만든다.

X) Other (please describe after [Answer]: tag below)

> **권장: A**. SEC-06(내부 정보 노출 금지)을 지키려면 내부 원인과 사용자 문구를 분리하는
> 지점이 필요합니다. 또 검증 실패 조건을 단위 테스트로 검증할 때 문자열이 아니라 오류
> 값으로 단정할 수 있어야 테스트가 문구 변경에 깨지지 않습니다.

[Answer]:A

## Question 7: 작성 중 내용 보존 방식 (NFR-04)

앱이 강제 종료되었을 때 작성 중이던 내용을 어떻게 다룰지입니다.

A) **명시적 저장 + 초안 자동 보존** — 저장 버튼으로만 기록이 확정되지만, 작성 중 내용은
   초안으로 자동 보관해 앱을 다시 열면 이어서 쓸 수 있다.

B) **명시적 저장만** — 저장하지 않고 종료하면 내용이 사라진다. NFR-04를 완화해야 한다.

C) **자동 저장** — 입력하는 즉시 기록으로 저장된다. 저장 버튼이 없다.

X) Other (please describe after [Answer]: tag below)

> **권장: A**. NFR-04가 이미 "강제 종료되어도 작성 중 내용 복원"을 요구합니다. C는
> 빈 기록이 목록에 쌓이고 삭제가 즉시 완전 삭제(FR-09)라는 결정과 어울리지 않습니다.
> B를 택하려면 NFR-04를 수정해야 하는데, 기도제목을 쓰다 전화를 받는 상황이 흔하다는
> 점을 고려하면 손실이 큽니다.

[Answer]:A

## Question 8: 테스트 프레임워크

A) **단위 테스트는 Swift Testing, UI 테스트는 XCUITest** — Swift 6 환경의 현재 표준
   조합이다.

B) **단위 테스트도 XCTest** — 익숙한 방식이지만 새 프로젝트에서 선택할 이유는 적다.

X) Other (please describe after [Answer]: tag below)

> **권장: A**. Xcode 26 / Swift 6.2 환경이 확인되었고, Swift Testing은 이 버전에서
> 기본 제공됩니다. UI 테스트는 Swift Testing 대상이 아니므로 XCUITest를 씁니다.

[Answer]:A

## Question 9: Xcode 타깃 이름 (영문)

한글 앱 표시 이름은 `묵담`으로 확정되었습니다. Xcode 타깃과 모듈 이름은 영문이어야
합니다.

A) `Mukdam`

B) `MukdamApp`

X) Other (please describe after [Answer]: tag below)

> **권장: A**. 짧고 모듈 이름으로도 자연스럽습니다. 표시 이름은 `Info.plist`에서
> `묵담`으로 따로 지정합니다.

[Answer]:A

## Question 10: 번들 식별자

App Store와 기기에서 앱을 식별하는 문자열입니다. 나중에 바꾸면 별개의 앱이 되므로
지금 정하는 편이 좋습니다. 보유한 도메인을 거꾸로 쓰는 것이 관례이며, 도메인이 없어도
고유하면 됩니다.

A) `com.jungkwan.mukdam` 형태 (개인 이름 기반)

B) 보유 도메인 기반 (예: `com.example.mukdam`) — `Other`에 실제 값을 적어주세요

X) Other (please describe after [Answer]: tag below)

> **권장: A 또는 실제 보유 도메인**. A를 고르시면 `com.jungkwan.mukdam`으로
> 진행하겠습니다. 다른 문자열을 원하시면 `Other`에 정확한 값을 적어주세요.

[Answer]:com.octo.mukdam

---

## 답변 확인

- [ ] Question 1 ~ 10 모두 답변했다
- [ ] Question 1(아키텍처 패턴)과 Question 7(작성 중 내용 보존)은 화면 구조와
      요구사항에 직접 영향이 있으므로 특히 확인했다
