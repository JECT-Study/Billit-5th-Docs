# ERD 변경 목록

관련 정책: [여행](../policy/trip.md), [지출](../policy/expense.md), [환율](../policy/exchange-rate.md), [정산](../policy/settlement.md)

## 반영할 변경

| 대상 | 기존 검토 DDL | 변경 방향 | 확정 범위 |
| --- | --- | --- | --- |
| `trip_member.guest_name` | `VARCHAR(10)`, 비로그인 참여자 이름. `UNIQUE (trip_id, guest_name)` | 로그인 참여자의 여행방 닉네임도 함께 저장하고 기존 제약 유지 | 로그인 참여자와 비로그인 참여자 모두의 여행방 닉네임을 저장한다. 같은 여행에서는 로그인 여부와 관계없이 이름 중복을 금지하며 비로그인 참여자 등록, 이름 수정과 여행 참여에 모두 적용한다. |
| `trip.members_locked_at` | `TIMESTAMPTZ NULL`, NULL이면 참여자 초대 중 | 여행 확정 여부를 `status` 컬럼으로 관리하고, 확정 시각은 `confirmed_at`으로 이름을 바꿔 함께 저장 | `status` 컬럼으로 관리하며, API에서는 `UNCONFIRMED`(확정 전)와 `CONFIRMED`(확정 후)로 전달한다. 확정 시각도 `confirmed_at`으로 함께 저장한다. |
| `trip.country_code` | `CHAR(2) NOT NULL` | `VARCHAR(3) NOT NULL`로 변경. 대표 국가 필수 유지. 기타 국가는 `ETC`를 저장하고 입력한 텍스트는 그대로 저장 | 기타 국가를 국가 코드로 바꾸지 않는 것은 확정했다. `CHAR(2)`에는 기타 국가 텍스트를 담을 수 없어 문자열 컬럼 `custom_country_name`을 새로 추가하고 Flyway 마이그레이션에 넣는다. 기타 국가이면 `country_code`에 API 값과 같은 `ETC`를 저장하고, 세 글자를 담도록 `country_code`를 `VARCHAR(3)`으로 바꾼다. API에서는 `countryCode`를 `ETC`로, 입력한 국가명을 `customCountryName`으로 전달한다. 기타 국가명은 최대 10자다([기본 정보](../policy/trip.md#기본-정보)). |
| `trip_member.role` | `OWNER`, `MEMBER` | `OWNER`를 `ADMIN`으로 변경 | 역할 값 `ADMIN`, `MEMBER` 확정. |
| 참여자 상태 | 컬럼 없음. `member_id`와 `guest_name` 값으로 구분 | `trip_member`에 참여자 상태 enum 컬럼 `trip_member_status` 추가 | `MEMBER`(로그인 참여자), `GUEST`(비로그인 참여자), `LEFT`(나간 회원), `WITHDRAWN`(탈퇴 회원)으로 관리한다. 여행에 참여할 때 `member_id`가 있는지 보고 상태를 정하고, 다른 기능은 이 값으로 구분한다. |
| 나간 시각 | 컬럼 없음 | `trip_member.left_at` 추가 | 여행을 나가도 행을 지우지 않고 `left_at`에 나간 시각을 기록한다. 나갔는지는 참여자 상태 `LEFT`로 판단한다. 나간 회원의 행은 지출과 정산에서 계속 참조하고 ‘나간 회원’으로 표시하므로, 조회에서 제외하는 `deleted_at`을 쓰지 않는다. |
| 회원 중복 참여 | `UNIQUE (trip_id, guest_name)`만 있고 같은 회원의 중복 참여를 막는 제약은 없음 | `UNIQUE (trip_id, member_id)` 추가 | 같은 회원이 한 여행에 두 번 참여하지 않게 한다. `left_at`과 함께 마이그레이션에 추가한다. |
| `trip_member` 행 생성 시점 | 자리마다 행을 미리 만들고, `member_id`와 `guest_name`이 모두 NULL인 행을 빈자리로 둔다 | 빈자리 행을 미리 만들지 않고, 참여나 비로그인 추가 때 행 생성 | 로그인 참여자가 참여하거나 관리자가 비로그인 참여자를 추가할 때 행을 만든다. 나간 회원이 확정 전에 다시 참여하면 새 행을 만들지 않고 기존 행을 다시 쓴다([재참여](../policy/trip.md#재참여)). 비로그인 참여자를 삭제하면 그 행을 지우고, 총인원을 바꿔도 행을 만들거나 지우지 않는다. 빈자리 수는 총인원에서 등록된 참여자 수를 빼서 계산한다([총인원과 빈자리](../policy/trip.md#총인원과-빈자리)). 정원 초과를 막는 방식은 [여행 참여](../api-spec/trip.md#trip-07-여행-참여)의 논의 사항을 따른다. |
| `member.nickname` | `UNIQUE (nickname)` | UNIQUE 제약 제거 | 계정 닉네임 중복 허용 확정. 여행방에서는 [여행방 닉네임](../policy/trip.md#여행방-닉네임)을 사용한다. |
| `member.name`, `member.age` | `name VARCHAR(50)`, `age INTEGER`, NULL이면 미입력 | 두 컬럼 삭제 | 가입할 때 이름과 나이를 입력받지 않아 사용하지 않는다([선택 정보](../policy/user.md#선택-정보)). |
| 여행 생성 고유키 | 없음 | `trip.idempotency_key UUID` 추가, 키 단독 UNIQUE | 여행 생성에만 사용하고 만료 없이 여행과 함께 보관한다. 여행과 같은 트랜잭션에 저장하며 키 단독 UNIQUE 제약을 둔다([여행 생성 멱등성](idempotency.md#여행-생성-멱등성)). |
| `settlement.created_at` | `TIMESTAMPTZ NOT NULL`, 준비 생성 시 기록 | NULL 허용, 준비 시 비워두고 정산 확정 시 기록 | 회의 결정. `issued_at` 추가안은 기각됐다. |
| `settlement.deleted_at` | 컬럼 없음 | `TIMESTAMPTZ NULL` 추가, 만료 또는 정상 이탈한 준비 정산을 논리 삭제 | 후속 설계에서 정리한 방향. 지출 점유 해제와 같은 트랜잭션에서 기록한다. |
| 잠금 식별값 | 소유자와 만료 시각만 존재 | 잠금 작업별 토큰 저장 및 요청 전달 | 토큰 도입은 회의 결정. 테이블별 배치, 이름과 타입은 제안이다. |
| 환율 출처 | `expense.exchange_rate_source`의 `AUTO/MANUAL`만 존재 | 여러 환율 출처를 구분할 정보 추가 | 출처 속성 추가는 회의 결정. 구체적인 저장 후보는 아래 구현 제안에서 다룬다. |
| 지출 저장 고유키 | 없음 | 중복 저장을 방지할 키 보관 구조 추가 | 앱 고유키 사용은 회의 결정. 저장 후보와 제약 범위는 멱등성 설계를 따른다. |

`trip`, `trip_member`와 `member`의 기존 값은 백엔드 `develop`의 초기 마이그레이션 `V20261107063430__create_initial_schema.sql`(커밋 `c857020`)에서 확인했다.

같은 마이그레이션은 `guest_name`을 비로그인 참여자 이름이며 회원이나 빈자리이면 NULL인 컬럼으로 설명한다. 로그인 참여자의 여행방 닉네임도 이 컬럼에 저장하므로, 백엔드 Flyway 마이그레이션의 `guest_name` 주석을 여행방 닉네임 기준으로 고쳐야 한다. 같은 주석의 '여행 시작 전 수정 가능'도 여행 확정 전과 후 모두 수정할 수 있는 기준으로 고쳐야 한다([비로그인 참여자](../policy/trip.md#비로그인-참여자)). 빈자리 행을 만들지 않으므로 `trip_member`의 '자리'와 '빈자리' 주석도 함께 고쳐야 한다.

## 추가 컬럼 후보

> 구현 제안
>
> 아래 표는 추가 컬럼 후보와 타입 선택안이다. 실제 migration과 구분한다.

| 테이블 후보 | 컬럼 후보 | 타입 후보 | 값과 NULL 의미 | 관련 설계 |
| --- | --- | --- | --- | --- |
| `expense` | `edit_lock_token` | `UUID` 또는 문자열, NULL 허용 | 수정 작업 식별값. 잠금 해제 시 소유자와 만료 시각도 함께 정리한다. | [수정 잠금](locking.md) |
| `settlement` | `preparation_lock_token` | `UUID` 또는 문자열 | 준비 작업 식별값 | [준비 잠금](locking.md) |
| `settlement` | `preparation_lock_owner_trip_member_id` | `BIGINT` | 준비 잠금 소유자 후보 | [준비 잠금](locking.md) |
| `settlement` | `deleted_at` | `TIMESTAMPTZ NULL` | NULL이면 미삭제, 값이 있으면 폐기한 준비 정산. 활성 조회와 연장 및 발행에서 제외 | [만료 정리](locking.md#만료된-정산-준비-정리) |
| `expense` | `exchange_rate_provider` | 코드 문자열 또는 enum | API 기관 등 출처 구분 후보 | [환율 보존](expense-calculation.md#저장-데이터) |
| `expense` | `exchange_rate_base_date` | `DATE` | 환율의 실제 기준일 보존 후보 | [환율 보존](expense-calculation.md#저장-데이터) |
| `expense` 또는 요청 기록 | `idempotency_key` | `UUID` 또는 문자열 | 앱 저장 요청의 고유키 | [멱등성](idempotency.md) |
| `expense` 또는 요청 기록 | `request_hash` | 문자열 또는 바이트 타입 | 같은 키의 입력 내용 비교 후보 | [멱등성](idempotency.md) |

기존 `exchange_rate_source`는 API 자동값인지 수동 입력값인지 구분한다. 기관명과 기준일은 이 값과 구분한다.

> 논의 필요
>
> API 기관은 코드 문자열 vs enum, 수동 입력 시 기관 정보 유지 vs 비움
>
> 기준일 컬럼의 필수 조건, 토큰의 타입 및 NULL 조건, 준비 소유권 컬럼 추가 여부는 각 설계의 선택안을 따른다.

### 예시: AUTO만 저장하면 나중에 알 수 없는 것

어떤 달러 지출이 기관 A의 기준일 환율을 사용했다고 가정한다. `exchange_rate_source = AUTO`만 있으면 ‘자동으로 가져왔다’는 사실은 알 수 있지만 어느 기관의 어느 날짜 환율인지 이 값만으로 알 수 없다.

출처와 기준일 컬럼 후보는 그 정보를 보존하기 위한 것이다. 기관 A와 B를 나중에 함께 사용해도 지출 당시 적용한 기준을 설명할 수 있게 한다. 수동 입력의 기관 정보 처리 선택은 위 논의 사항을 따른다.

결제와 환전 날짜는 사용자가 입력한 현지 날짜를 저장하므로 기존 `expense.paid_date DATE`와 `manual_exchange_rate.exchange_date DATE`를 유지한다. 시각과 시간대 컬럼은 추가하지 않는다.

## 인증 테이블과 컬럼

인증 구현에서 기존 ERD에 없는 테이블, 컬럼과 제약을 추가했다. 아래 변경은 구현 PR의 마이그레이션이며 아직 `develop`에 반영되지 않았다. `auth_session`은 [PR #30](https://github.com/JECT-Study/Billit-5th-Server/pull/30), 소셜 계정 UNIQUE는 [PR #31](https://github.com/JECT-Study/Billit-5th-Server/pull/31), `apple_refresh_token`과 `social_display_name`은 [PR #33](https://github.com/JECT-Study/Billit-5th-Server/pull/33)에 있다. 설계 근거는 [인증 세션과 토큰](authentication.md)을 따른다.

| 대상 | 변경 | 용도 |
| --- | --- | --- |
| `auth_session` | 테이블 추가 | 로그인 세션과 현재, 직전 Refresh Token의 해시를 저장한다. 컬럼은 [로그인 세션 저장](authentication.md#로그인-세션-저장)을 따른다. |
| `apple_refresh_token` | 테이블 추가, `member_id` UNIQUE | Apple 연결 해제에 사용할 Apple Refresh Token을 회원당 한 행으로 암호화해 저장한다. |
| `member.social_display_name` | `VARCHAR(100) NULL` 추가 후 삭제 | 소셜 표시 이름을 쓰지 않게 되어 [서버 PR #49](https://github.com/JECT-Study/Billit-5th-Server/pull/49)에서 컬럼을 지운다. |
| `member` | `UNIQUE (social_provider, social_provider_user_id)` 추가 | 같은 소셜 계정으로 회원이 두 번 생기지 않게 한다. 탈퇴 처리로 `social_provider_user_id`가 NULL인 행끼리는 중복을 허용한다. |

## 기존 구조에서 유지할 관계

| 대상 | 역할 및 주의점 |
| --- | --- |
| `expense.settlement_id` | 현재 점유 정산이다. `DRAFT`도 포함하며 과거 포함 이력과 구분한다. |
| `settlement_expense` | 발행 시 포함 지출 연결이다. `(settlement_id, expense_id)` 복합 PK를 유지하고 `expense_id` 단독 UNIQUE를 두지 않는 기존 검토안이다. |
| `expense_share_amount` | `(expense_id, trip_member_id)`별 최종 부담액이다. |
| `expense_item_member` | `(expense_item_id, trip_member_id)`로 품목별 부담자 중복을 막는다. |
| `transfer` | 송금인과 수령인은 `trip_member.id`를 참조한다. 금액 0원 행은 저장하지 않는다. |

기존 연결 구조는 지출 ID를 보존하며 금액 스냅샷은 포함하지 않는다. 취소 후 지출이 수정되면 단순 FK로는 당시 산출 근거를 복원할 수 없다.

> 논의 필요
>
> 취소 정산의 송금 및 포함 지출 이력 보존 범위는 PM 논의 중이다. 보존 범위 결정 후 금액 스냅샷 필요 여부를 정한다.

### 예시: 같은 지출을 취소 후 다시 정산하는 경우

지출 101이 정산 900에 포함됐다가 수령 확인 전에 취소됐고, 이후 정산 901에 다시 포함된다고 가정한다.

현재 점유는 `expense.settlement_id`를 NULL로 해제했다가 새 준비 시 901로 바꾸면 된다. 과거 포함 연결을 보존하는 안에서는 다음 두 행이 함께 존재할 수 있어야 한다.

| `settlement_id` | `expense_id` | 의미 |
| --- | --- | --- |
| 900 | 101 | 취소한 정산에 포함됐던 지출 |
| 901 | 101 | 새 정산에 포함한 지출 |

이 때문에 복합 PK는 유지하되 `expense_id`만의 UNIQUE는 두지 않는다. 다만 101의 금액이 재정산 전에 12,000원에서 15,000원으로 바뀌면, 두 연결 모두 같은 현재 지출을 가리킨다. 연결만 보존해서는 ‘정산 900 당시에는 12,000원이었다’를 복원할 수 없다. 스냅샷 선택은 위 논의 사항을 따른다.

### 예시: 여행방 닉네임이 이미 등록된 경우

같은 여행의 `trip_member` 21에 `guest_name = 민수`가 있다면, 민수를 다시 등록하는 요청은 거절한다. 다른 비회원 이름을 민수로 수정하려는 요청과 회원이 여행방 닉네임 민수로 참여하려는 요청도 같은 제약을 적용한다. 요청이 동시에 들어와도 기존 `(trip_id, guest_name)` UNIQUE로 같은 이름이 둘 다 저장되지 않게 한다.

로그인 참여자의 여행방 닉네임도 `guest_name`에 저장하므로, 기존 복합 UNIQUE 하나로 회원과 비회원 사이의 이름 중복까지 막는다. 회원이 계정 닉네임(`member.nickname`)을 바꿔도 `guest_name`은 바뀌지 않는다.

참여자 연결은 계속 `trip_member.id`를 사용한다.

## 정산 시각과 BaseEntity

[BaseEntity](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/persistence/BaseEntity.kt)는 `@CreatedDate`로 최초 저장 시 `created_at`을 기록하고, 매핑에서 `updatable = false`로 지정한다. 정산 시각 결정과 그대로 호환되지 않는다. 코드 확인 기준은 백엔드 커밋 `09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c`이며, DB 적용이나 배포 동작을 검증한 것은 아니다.

| 충돌 | 구현 시 확인할 내용 |
| --- | --- |
| DRAFT 최초 저장 시 자동 시각 기록 | 정산 `created_at`은 발행 전 NULL이어야 한다. 공통 Auditing이 값을 채우지 않아야 한다. |
| `updatable = false` | 확정 시 NULL에서 발행 시각으로 바꿀 수 있어야 한다. |
| 공통 필드 상속 | 정산만 별도 매핑할지 등 적용 방식을 합의해야 한다. 다른 Entity의 생성 시각 정책까지 변경하지 않는다. |

PM 답변은 발행 시각이 확정 시각임을 확인했다. 저장 컬럼은 기존 개발 결정대로 nullable `created_at`을 사용한다.

> 논의 필요
>
> BaseEntity 상속 제외 vs 정산 전용 Auditing 및 매핑 구성
>
> 준비 시작 시각 별도 보존 여부. 기존에 기각한 issued_at 추가와 구분한다.

정산의 논리 삭제에 `deleted_at`이 필요하더라도 BaseEntity 전체 상속을 자동으로 결정하지 않는다. 공통 생성 시각과 발행 시각의 충돌을 먼저 해소하고, 논리 삭제 필드는 정산 매핑에 명시적으로 반영한다. 조회와 조건부 갱신에서도 `deleted_at IS NULL`을 직접 검사한다.

## 추가 결정이 필요한 스키마

> 논의 필요
>
> | 안건 | 기존 상태와 결정할 내용 |
> | --- | --- |
> | 정산 방식 미선택 | `settlement_method NOT NULL`과 방식 선택 전 `DRAFT` 생성이 충돌한다. nullable 변경 또는 생성 순서 조정 필요 |
> | 최근 저장 환율 | 같은 결제 통화와 created_at 내림차순은 확정. 환전 수단 필터와 동률 정렬 기준, 인덱스 선택 |
> | 금액 타입과 단위 | 원본 통화 최소 단위와 원화 소수 보존, 합계 검증을 먼저 정한 뒤 `BIGINT` 유지 여부 결정 |
> | 지출명 | 입력 길이 제한 없는 정책과 `VARCHAR(50)` 간 차이 해소 |
> | 동일 여행 제약 | 단일 FK 외 서비스 검증 또는 복합 FK 적용 방식 |
> | 취소 이력과 개인정보 | 보존 범위가 정해진 뒤 스냅샷, 삭제와 익명화 구조 검토 |
> | 폐기한 준비 정산 | 논리 삭제한 DRAFT의 보존 기간과 최종 물리 삭제 여부. 발행된 정산 취소 이력과 구분 |
> | 인덱스 | 만료 DRAFT 조회와 멱등성 키 범위를 확정한 뒤 필요한 인덱스 설계. 이름만 먼저 확정하지 않는다. |

실제 반영 시에는 기존 데이터, NULL과 상태 조합, 제약 제거 및 추가 영향과 마이그레이션 후 매핑을 검증한다. 검토 DDL의 변경 제안과 실제 마이그레이션 적용 결과는 구분한다.
