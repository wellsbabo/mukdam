# 묵담 요구사항 확인 질문

**단계**: INCEPTION / Requirements Analysis
**작성일**: 2026-08-20
**입력 문서**: `requirements/requirements.md`, `requirements/constraints.md`

각 질문의 `[Answer]:` 뒤에 알파벳 하나를 적어주세요. 보기 중에 맞는 게 없으면
마지막 `Other`를 고르고 뒤에 원하는 내용을 적어주시면 됩니다.
답변이 끝나면 채팅으로 알려주세요.

이미 확정된 내용(Swift/SwiftUI, 앱 이름 묵담, 계정 없음, 성경 본문 미포함,
묵상당 다중 구절 참조, 내보내기·필터 MVP 포함)은 다시 묻지 않습니다.

---

## A. 기능 및 화면

## Question 1
기록을 작성할 때 기도제목과 묵상에 어떻게 진입하고 싶으신가요?

A) 하나의 작성 화면에서 종류(기도제목 / 묵상)를 선택한다

B) 기도제목 작성과 묵상 작성 버튼을 처음부터 분리한다

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 2
앱을 열었을 때 처음 보이는 화면은 무엇이어야 하나요?

A) 날짜별로 묶인 기록 목록 (최신 날짜가 위)

B) 오늘 날짜 중심의 화면(오늘 기록 + 바로 작성 버튼), 전체 목록은 별도 탭

C) 월간 캘린더에서 기록이 있는 날짜를 표시

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 3
기록에 제목 필드가 필요한가요?

A) 필요 없다. 내용 본문만 있으면 된다 (목록에는 내용 앞부분을 미리보기로 표시)

B) 제목을 선택 입력으로 제공한다 (비워두면 내용 앞부분을 제목처럼 사용)

C) 제목을 필수 입력으로 한다

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 4
같은 날짜에 기록을 여러 건 남길 수 있어야 하나요?

A) 제한 없이 여러 건 가능

B) 하루에 종류별 1건씩만 (기도제목 1건, 묵상 1건)

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 5
기록을 삭제할 때 어떻게 처리하는 게 좋을까요?

A) 확인 후 즉시 완전 삭제

B) `최근 삭제함`에 일정 기간(예: 30일) 보관 후 자동 삭제, 복구 가능

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 6
내보내기(공유)는 어떤 단위로 지원해야 하나요?

A) 기록 1건 단위만

B) 특정 날짜의 기록 전체 단위만

C) 1건 단위와 날짜 단위 모두

D) 1건, 날짜, 기간 선택(예: 이번 주) 모두

X) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## B. 성경 구절 참조

## Question 7
성경 책 이름을 어떻게 입력하고 싶으신가요?

A) 앱에 내장된 66권 목록에서 선택 (오타 없음, 표기 통일)

B) 직접 텍스트로 자유 입력 (약칭이나 개인 표기 자유)

C) 텍스트 입력 + 자동완성 추천 (내장 목록 기반, 목록에 없는 표기도 허용)

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 8
성경 구절 참조에서 절(verse) 입력을 어디까지 허용할까요?

A) 장만 기록해도 된다 (예: `시편 23편` — 절 없이 저장 가능)

B) 장과 절을 모두 입력해야 한다

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 9
성경 책 이름 표기를 무엇에 맞출까요? (본문은 포함하지 않고 책 이름 표기만
결정합니다)

A) 개역개정 기준 한글 정식 명칭 (예: `요한복음`, `사도행전`)

B) 정식 명칭 + 약칭 병기 및 약칭 검색 지원 (예: `요한복음` / `요`)

X) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## C. 데이터 보관 및 기기 변경

## Question 10
기기 변경 시 데이터 이전 문제를 MVP에서 어떻게 다룰까요? (가장 중요한 결정입니다)

A) MVP에 iCloud 동기화 포함 — SwiftData + CloudKit 개인 데이터베이스 사용.
   로그인 화면 없이 기기의 애플 계정을 그대로 쓰므로 계정 기능이 필요 없고,
   서버와 운영비도 발생하지 않으며 기기 변경 시 자동으로 데이터가 따라옵니다.
   대신 유료 개발자 계정과 CloudKit 컨테이너 설정이 필요합니다.

B) MVP는 로컬 저장만. 데이터 이전 수단은 다음 버전에서 검토 (지금은 기기를
   바꾸면 기록이 사라짐)

C) MVP는 로컬 저장만 하되, 백업 파일 내보내기/가져오기를 제공해 수동으로
   이전할 수 있게 한다

D) 로컬 저장 + 백업 파일(C)로 시작하고, iCloud 동기화(A)는 다음 버전에서 추가

X) Other (please describe after [Answer]: tag below)

[Answer]:D

## Question 11
백업 파일을 제공하는 경우(Question 10에서 C 또는 D 선택 시) 어떤 형식이 좋을까요?

A) JSON (앱으로 되돌려 가져오기 정확, 사람이 읽기는 불편)

B) 마크다운/텍스트 (사람이 바로 읽을 수 있음, 되돌려 가져오기는 제한적)

C) 둘 다 제공 (JSON은 복원용, 텍스트는 열람용)

D) 해당 없음 (Question 10에서 A 또는 B를 선택)

X) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## D. 기술 및 개발 환경

## Question 12
iOS 최소 지원 버전을 어떻게 할까요?

A) iOS 17 이상 — SwiftData를 쓸 수 있어 저장 코드가 가장 단순해집니다.
   현재 대다수 기기가 해당됩니다.

B) iOS 16 이상 — 지원 범위는 넓어지지만 SwiftData를 못 쓰고 Core Data를 직접
   다뤄야 해서 코드가 늘어납니다.

C) iOS 18 이상 — 최신 API를 자유롭게 쓸 수 있습니다.

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 13
Xcode 프로젝트를 처음 만드는 방식을 정해야 합니다. 저는 Xcode GUI를 직접 조작할
수 없어서, 프로젝트 파일 생성 방법에 따라 진행 방식이 달라집니다.

A) 직접 Xcode에서 새 iOS 앱 프로젝트를 만들고, 저는 그 안에 들어갈 Swift 소스
   파일과 구조를 작성한다 (가장 확실하고 흔한 방식)

B) 저(AI)가 XcodeGen 설정 파일(`project.yml`)과 소스를 모두 작성하고, 명령
   한 번으로 프로젝트를 생성한다 (Homebrew로 xcodegen 설치 필요)

C) 저(AI)가 Swift Package로 앱 로직과 뷰를 모두 작성하고, 얇은 앱 셸
   프로젝트만 직접 만들어 연결한다 (로직 테스트가 쉬움)

X) Other (please describe after [Answer]: tag below)

[Answer]:A. 다만 어떤식으로 프로젝트를 만들어야하는지는 알려줘야함

> **검토 메모 (2026-08-20) — A 권장**
>
> 확인된 개발 환경: Xcode 26.0.1, Swift 6.2, Homebrew 설치됨,
> xcodegen·tuist 미설치.
>
> - **A 권장 근거**: Xcode 16부터 프로젝트가 디스크 폴더를 그대로 참조하는
>   방식(synchronized file system group)을 기본으로 사용한다. 앱 폴더에 `.swift`
>   파일을 추가하면 `.pbxproj` 수정 없이 빌드에 포함되므로, AI가 Xcode GUI를
>   조작할 수 없다는 제약이 사실상 해소된다. 빌드·테스트 검증은 커맨드라인
>   `xcodebuild`로 수행 가능하다.
> - **B 비권장 근거**: XcodeGen 자체는 성숙한 도구지만 해결 대상 문제(다인 개발
>   시 `.pbxproj` 머지 충돌, 다수 타깃 관리)가 이 프로젝트에 없다. 파일 추가마다
>   `xcodegen generate` 재실행이 필요하고, Xcode에서 직접 추가한 변경은 재생성 시
>   유실된다. 또한 `.pbxproj`를 파싱하는 서드파티 도구는 Apple의 포맷 변경에
>   뒤따라간다(Xcode 16 synchronized group 도입 시 CocoaPods 등 다수 도구가
>   `unknown ISA PBXFileSystemSynchronizedRootGroup` 오류로 중단된 사례).
> - **C 참고**: 로직을 시뮬레이터 없이 `swift test`로 검증할 수 있다는 장점이
>   있으나, 이 규모의 앱에 모듈 경계를 추가하는 관리 비용이 이득보다 크다.B

## Question 14
테스트는 어디까지 작성할까요?

A) 핵심 로직 단위 테스트만 (성경 구절 파싱/검증, 날짜 묶음, 내보내기 텍스트 생성)

B) 단위 테스트 + 주요 화면 UI 테스트

C) 테스트 없이 진행

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 15
수익 모델은 나중에 결정하기로 했습니다. MVP 코드에서 이를 어떻게 다룰까요?

A) 전혀 고려하지 않는다. 완전 무료 앱으로 단순하게 만든다

B) 나중에 유료 기능을 붙이기 쉽도록 기능 경계만 정리해 둔다 (결제 코드는 넣지
   않음)

X) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## E. 확장 규칙 적용 여부

> AI-DLC에는 선택 적용 가능한 확장 규칙이 있습니다. 적용하면 해당 규칙이
> 이후 모든 단계에서 반드시 지켜야 하는 제약으로 취급됩니다.

## Question 16: 보안 확장 규칙
이 프로젝트에 보안 확장 규칙(Security Baseline)을 적용할까요?

A) 예 — 모든 보안 규칙을 반드시 지켜야 하는 제약으로 적용 (운영 서비스 수준
   애플리케이션에 권장)

B) 아니오 — 보안 규칙을 적용하지 않음 (PoC, 프로토타입, 실험용 프로젝트에 적합)

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 17: 복원력(Resiliency) 확장 규칙
이 프로젝트에 복원력 기준(Resiliency Baseline)을 적용할까요?

**이 확장이 무엇인지**: 적용하면 AWS Well-Architected Framework 신뢰성 축과
복원력 검토 지침에서 도출된 **설계 시점의 방향성 있는 모범 사례**가 적용됩니다.
장애 내성, 고가용성, 관측성, 복구 가능성 관점에서 요구사항·설계·코드를
유도합니다.

**이 확장이 아닌 것**: 적용해도 워크로드가 운영 준비 완료 상태가 되는 것은
아니며, 특정 가용성·RTO·RPO 목표를 보증하지도 않습니다. 정식 AWS
Well-Architected 리뷰를 대체하지 않는 출발점입니다.

A) 예 — 설계 시점 지침으로 복원력 기준을 적용 (업무상 중요한 워크로드에 권장)

B) 아니오 — 복원력 기준을 적용하지 않음 (PoC, 프로토타입, 빠른 반복이 더 중요한
   프로젝트에 적합)

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 18: 속성 기반 테스트(Property-Based Testing) 확장 규칙
이 프로젝트에 속성 기반 테스트 규칙을 적용할까요?

A) 예 — 모든 PBT 규칙을 반드시 지켜야 하는 제약으로 적용 (비즈니스 로직, 데이터
   변환, 직렬화, 상태를 다루는 컴포넌트가 있는 프로젝트에 권장)

B) 부분 적용 — 순수 함수와 직렬화 왕복 검증에만 PBT 규칙을 적용 (알고리즘 복잡도가
   제한적인 프로젝트에 적합)

C) 아니오 — PBT 규칙을 적용하지 않음 (단순 CRUD 앱, UI 중심 프로젝트, 비즈니스
   로직이 거의 없는 얇은 연동 계층에 적합)

X) Other (please describe after [Answer]: tag below)

[Answer]:C

---

## 답변 확인

- [ ] Question 1 ~ 18 모두 답변했다
- [ ] Question 10(데이터 이전)과 Question 13(Xcode 프로젝트 생성 방식)은 이후
      설계와 구현 방식을 크게 바꾸므로 특히 확인했다
