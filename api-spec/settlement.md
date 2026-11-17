# 정산 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [정산 준비 API](#정산-준비-api)
  - [SETTLE-01 정산 대상 조회](#settle-01-정산-대상-조회)
  - [SETTLE-02 정산 준비 시작](#settle-02-정산-준비-시작)
  - [SETTLE-03 정산 준비 조회](#settle-03-정산-준비-조회)
  - [SETTLE-04 준비 잠금 연장](#settle-04-준비-잠금-연장)
  - [SETTLE-05 정산 준비 취소](#settle-05-정산-준비-취소)
- [계산과 발행 API](#계산과-발행-api)
  - [SETTLE-06 정산 미리보기](#settle-06-정산-미리보기)
  - [SETTLE-07 정산 확정과 발행](#settle-07-정산-확정과-발행)
- [정산서 조회 API](#정산서-조회-api)
  - [SETTLE-08 여행 정산 목록](#settle-08-여행-정산-목록)
  - [SETTLE-09 발행 정산 상세](#settle-09-발행-정산-상세)
  - [SETTLE-10 참여자별 정산서](#settle-10-참여자별-정산서)
  - [SETTLE-11 내 정산 목록](#settle-11-내-정산-목록)
  - [SETTLE-12 내 정산 상세](#settle-12-내-정산-상세)
- [수령 확인과 취소 API](#수령-확인과-취소-api)
  - [SETTLE-13 수령 확인](#settle-13-수령-확인)
  - [SETTLE-14 발행 정산 취소](#settle-14-발행-정산-취소)
- [공통 모델](#공통-모델)
  - [Candidate](#candidate)
  - [PreparationInput](#preparationinput)
  - [Preparation](#preparation)
  - [PreparationView](#preparationview)
  - [LockExpiry](#lockexpiry)
  - [MethodInput](#methodinput)
  - [SettlementPreview](#settlementpreview)
  - [CurrencyTotal](#currencytotal)
  - [Balance](#balance)
  - [TransferPreview](#transferpreview)
  - [SettlementDetail](#settlementdetail)
  - [Transfer](#transfer)
  - [SettlementSummary](#settlementsummary)
  - [Statement](#statement)
  - [StatementExpense](#statementexpense)
  - [StatementSummary](#statementsummary)
  - [ReceiptConfirmation](#receiptconfirmation)
  - [SettlementCancellation](#settlementcancellation)
- [오류 코드](#오류-코드)

## 문서 범위

정산 준비 취소와 발행된 정산 취소를 분리한다. 송금은 앱 밖에서 진행하며 앱은 실제 송금 실행 API를 제공하지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

이 문서는 API 스펙 검토용 초안이다. API 식별자는 화면 정의서 식별자와 별개다. 구체적인 경로, 요청과 응답 필드 및 오류 코드는 아직 팀 검토를 마치지 않았다. 각 API의 `논의 필요` 인용문 안에 제안 모델과 예시를 구분한다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [정산 준비 시작](#settle-02-정산-준비-시작) | `POST` | `/api/v1/trips/{tripId}/settlements` | 여행 관리자 (여행 확정 후) |
| [정산 준비 조회](#settle-03-정산-준비-조회) | `GET` | `/api/v1/trips/{tripId}/settlements/{settlementId}/preparation` | 정산 준비를 시작한 여행 관리자 |
| [준비 잠금 연장](#settle-04-준비-잠금-연장) | `PATCH` | `/api/v1/trips/{tripId}/settlements/{settlementId}/preparation-lock` | 정산 준비를 시작한 여행 관리자 |
| [정산 준비 취소](#settle-05-정산-준비-취소) | `DELETE` | `/api/v1/trips/{tripId}/settlements/{settlementId}/preparation` | 정산 준비를 시작한 여행 관리자 |
| [정산 미리보기](#settle-06-정산-미리보기) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/previews` | 정산 준비를 시작한 여행 관리자 |
| [정산 확정과 발행](#settle-07-정산-확정과-발행) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/confirm` | 정산 준비를 시작한 여행 관리자 |
| [여행 정산 목록](#settle-08-여행-정산-목록) | `GET` | `/api/v1/trips/{tripId}/settlements` | 여행 참여자 |
| [발행 정산 상세](#settle-09-발행-정산-상세) | `GET` | `/api/v1/trips/{tripId}/settlements/{settlementId}` | 여행 참여자 |
| [참여자별 정산서](#settle-10-참여자별-정산서) | `GET` | `/api/v1/trips/{tripId}/settlements/{settlementId}/statements/{tripMemberId}` | 여행 참여자 |
| [내 정산 목록](#settle-11-내-정산-목록) | `GET` | `/api/v1/me/settlement-statements` | 로그인 필요 |
| [내 정산 상세](#settle-12-내-정산-상세) | `GET` | `/api/v1/me/settlement-statements/{settlementId}` | 로그인 필요 (본인 정산서) |
| [수령 확인](#settle-13-수령-확인) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/transfers/{transferId}/receipt-confirmation` | 로그인 필요 (해당 송금의 수령인) |
| [발행 정산 취소](#settle-14-발행-정산-취소) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/cancel` | 여행 관리자 |

## 정산 준비 API

### SETTLE-01 정산 대상 조회

정산 대상 조회는 [EXP-01 지출 목록](expense.md#exp-01-지출-목록)의 `GET /api/v1/trips/{tripId}/expenses`로 통합한다. 별도의 settlement-candidates API는 제공하지 않는다. 기존 API 식별자는 통합 이력을 위해 유지하며 후속 API 번호를 변경하지 않는다.

인증은 지출 목록 API의 여행 참여자 조건을 따른다. 정산 준비 시작 권한은 여행 확정 후 관리자에게만 허용하는 기존 조건을 유지한다. 조회 시 선택 가능해도 준비 시작 시 권한, 잠금과 정산 상태를 다시 검증한다.

> 구현 제안
>
> [Candidate](#candidate)의 선택 가능 여부와 선택 불가 사유를 지출 목록 응답에 통합한다. 필드명과 포함 조건은 지출 목록의 응답 모델에서 확정한다.

전체 선택은 현재 필터의 선택 가능한 지출을 대상으로 한다. 앱은 전체 ID를 수집하지 않고 필터 조건과 선택 해제한 지출 ID를 정산 준비 요청에 보낸다. 서버는 같은 여행과 필터에 해당하는 대상에서 제외 ID를 빼고 선택 대상을 구한다. page와 size는 전체 선택 범위를 제한하지 않는다. 전체 선택 시점에 확정한 대상 ID 범위를 유지하며, 이후 새로 등록되거나 필터 조건에 새로 들어온 지출은 자동 포함하지 않는다. 선택 대상에서 제외한 ID는 이 범위에서 제거한다. 선택된 ID의 금액과 품목 등 내용이 이후 수정됐더라도 변경 확인을 다시 요구하지 않고 정산 준비 시점의 최신 저장값을 사용한다. 결제일이 바뀌어도 선택 시점의 ID 범위를 다시 필터링하지 않는다. 삭제, 수정 잠금과 다른 정산 포함 여부는 준비 시작 시 다시 검증한다.

> 논의 필요
>
> 결제일의 날짜 옵션 제공 방식, 시간대와 기간 경계, 필터 변경 시 기존 선택 유지 여부, 기본 정렬, 정산 필터와 응답 필드, 전체 선택 필터의 구체적인 필드와 제외 ID 검증, 선택 시점의 대상 범위를 서버에 보존하고 준비 요청에서 식별하는 방식을 결정한다.

### SETTLE-02 정산 준비 시작

`POST /api/v1/trips/{tripId}/settlements`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

여행 확정 후, 로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

개별 선택은 지출 ID 목록으로, 전체 선택은 필터 조건과 제외 ID로 요청한다. 서버는 선택 대상을 구한 뒤 정산 대상으로 잠그고 발행 전 준비 상태로 등록한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [PreparationInput](#preparationinput).
>
> ```json
> {
>   "selectionMode": "EXPLICIT",
>   "expenseIds": [
>     "expense-example-a"
>   ]
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `201 Created`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [Preparation](#preparation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Preparation](#preparation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "status": "DRAFT",
>     "expenseIds": [
>       "expense-example-a"
>     ],
>     "settlementMethod": null,
>     "lockToken": "example-preparation-lock-token",
>     "expiresAt": "2026-11-05T03:00:00Z",
>     "createdAt": null
>   }
> }
> ```
>
> 생성한 대상의 조회 URL을 `Location`에 제공한다. 조회 경로를 아직 정의하지 않은 자원은 해당 경로를 함께 확정해야 한다.

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_EXPENSE_UNAVAILABLE` | 선택한 지출 중 한 건 이상이 수정 중이거나 다른 정산에 포함돼 있음 |

#### 처리 규칙

같은 여행의 미정산 지출을 한 건 이상 선택해야 한다. 개별 선택의 expenseIds에 같은 ID가 여러 번 들어오면 중복을 제거한 고유 ID 집합으로 검증, 잠금과 계산을 진행한다. 중복이라는 이유만으로 요청을 거절하지 않으며 같은 지출을 여러 번 합산하지 않는다. 한 건이라도 수정 중이거나 다른 정산에 포함돼 있으면 전체 요청을 거절한다. 실패한 경우 준비 정산을 만들거나 일부 지출만 잠그지 않는다.

> 논의 필요
>
> - 최대 지출 건수와 잠금 유지 시간을 결정한다. 준비를 시작한 회원과 현재 작업을 확인하는 방법, 준비 시작 응답을 받지 못한 앱이 기존 준비를 조회하는 방법도 정해야 한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-03 정산 준비 조회

`GET /api/v1/trips/{tripId}/settlements/{settlementId}/preparation`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 정산 준비의 선택 지출과 방식, 유효 기간을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [PreparationView](#preparationview)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [PreparationView](#preparationview) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "status": "DRAFT",
>     "expenseIds": [
>       "expense-example-a"
>     ],
>     "settlementMethod": null,
>     "expiresAt": "2026-11-05T03:00:00Z",
>     "createdAt": null
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 만료 또는 취소되었거나 발행 전 준비 상태가 아님 |

#### 처리 규칙

취소되거나 만료되지 않은 준비 정산만 조회한다. 조회만으로 다른 화면에서 시작한 정산 준비를 이어받거나 잠금 토큰을 새로 받지는 않는다. 토큰을 잃어버렸을 때의 처리는 결정해야 한다.

> 논의 필요
>
> - 만료 전 준비 화면으로 돌아오는 방법을 결정한다. 앱이 준비 잠금 토큰을 잃어버렸을 때 기존 준비를 이어갈 수 있는지, 조회에도 토큰을 요구할지 함께 정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-04 준비 잠금 연장

`PATCH /api/v1/trips/{tripId}/settlements/{settlementId}/preparation-lock`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 정산 준비의 유효 기간을 연장한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

##### Header

> 논의 필요
>
> 전달 위치와 헤더 이름을 확정해야 한다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. 헤더 이름과 전달 위치는 제안이다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [LockExpiry](#lockexpiry)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [LockExpiry](#lockexpiry) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "expiresAt": "2026-11-05T03:00:00Z"
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 정산 준비가 취소되거나 만료됐거나 잠금 토큰이 현재 정산 준비과 다름 |

#### 처리 규칙

준비가 취소되거나 발행되지 않았는지, 요청자가 준비를 시작한 회원인지, 토큰이 같은 정산 준비인지, 만료 전인지 확인한다. 조건을 만족하면 선택한 지출 전체의 준비 시간을 연장한다.

> 논의 필요
>
> - 준비 토큰 전달 위치, 잠금 유지 시간과 연장 주기를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-05 정산 준비 취소

`DELETE /api/v1/trips/{tripId}/settlements/{settlementId}/preparation`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행 전 정산 준비를 취소하고 선택 지출의 잠금을 해제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

##### Header

> 논의 필요
>
> 전달 위치와 헤더 이름을 확정해야 한다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. 헤더 이름과 전달 위치는 제안이다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `204 No Content`를 반환하는 안이다. 응답 Body는 없다.

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 요청자가 이 정산 준비를 시작한 회원이 아니거나 이미 취소 또는 발행된 준비임 |

#### 처리 규칙

요청자가 해당 정산 준비를 시작했는지 확인한다. 준비 취소와 선택 지출의 잠금 해제는 함께 성공하거나 함께 취소된다. 이미 발행한 정산은 이 API로 취소할 수 없다.

> 논의 필요
>
> - 만료 후 정상 이탈 허용, 반복 취소 응답과 토큰 전달 위치를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

## 계산과 발행 API

### SETTLE-06 정산 미리보기

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/previews`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

정산 방식에 따른 부담액과 잔액 및 송금 내역를 미리 계산한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

##### Header

> 논의 필요
>
> 전달 위치와 헤더 이름을 확정해야 한다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. 헤더 이름과 전달 위치는 제안이다. |

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [MethodInput](#methodinput).
>
> ```json
> {
>   "settlementMethod": "KRW_UNIFIED"
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [SettlementPreview](#settlementpreview)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [SettlementPreview](#settlementpreview) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "settlementMethod": "KRW_UNIFIED",
>     "expenseCount": 1,
>     "totals": [
>       {
>         "currency": "KRW",
>         "amount": "12000.00"
>       }
>     ],
>     "balances": [
>       {
>         "tripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "paidAmount": "12000.00",
>         "shareAmount": "6000.00",
>         "balance": "6000.00"
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "currency": "KRW",
>         "paidAmount": "0.00",
>         "shareAmount": "6000.00",
>         "balance": "-6000.00"
>       }
>     ],
>     "transfers": [
>       {
>         "senderTripMemberId": "participant-example-b",
>         "receiverTripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "amount": "6000.00"
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 정산 준비가 취소되거나 만료되어 미리볼 수 없음 |

#### 처리 규칙

지출에 저장된 환율로 계산하며 새 환율을 적용하지 않는다. 미리보기만으로 정산서를 발행하거나 송금 내역을 확정 저장하지 않는다. 계산 결과와 그 결과에 대응하는 대상 지출 및 정산 방식을 발행 전 미리보기로 보존한다. 발행 요청은 사용자가 확인한 미리보기를 식별하며, 발행 시 보존 결과와 현재 준비 조건의 일치를 검증한다.

> 논의 필요
>
> - 미리보기 결과의 저장 위치와 식별 및 버전 방식, 송금 횟수를 최소로 보장할지와 계산 결과가 여러 개일 때의 선택 기준, 관리자 부담 자투리와 통화 최소 단위를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-07 정산 확정과 발행

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/confirm`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

확인한 정산 결과를 확정하고 참여자별 정산서를 발행한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

##### Header

> 논의 필요
>
> 전달 위치와 헤더 이름을 확정해야 한다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. 헤더 이름과 전달 위치는 제안이다. |

> 논의 필요
>
> 현재 정산 준비의 잠금 토큰이 필요하다. 어떤 미리보기 결과를 확정하는지 확인할 방법을 정한 뒤 추가 요청 필드와 예시를 작성한다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [SettlementDetail](#settlementdetail)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [SettlementDetail](#settlementdetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "tripId": "trip-example-a",
>     "status": "IN_PROGRESS",
>     "settlementMethod": "KRW_UNIFIED",
>     "expenseCount": 1,
>     "createdAt": "2026-11-05T03:00:00Z",
>     "totals": [
>       {
>         "currency": "KRW",
>         "amount": "12000.00"
>       }
>     ],
>     "tripMemberIds": [
>       "participant-example-a",
>       "participant-example-b"
>     ],
>     "transfers": [
>       {
>         "transferId": "transfer-example-a",
>         "senderTripMemberId": "participant-example-b",
>         "receiverTripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "amount": "6000.00",
>         "receivedAt": null,
>         "isReceivable": true
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 준비가 만료되었거나 취소 또는 발행되어 확정할 수 없음 |

#### 처리 규칙

준비가 취소되거나 만료되지 않았는지, 요청자와 토큰이 현재 정산 준비과 일치하는지, 선택 지출이 여전히 이 정산에 잠겨 있는지 다시 확인한다. 포함 지출과 송금 내역은 함께 저장하며 일부만 성공하지 않는다. 같은 정산을 다시 확정해도 정산서나 송금 내역을 중복 생성하지 않는다. 사용자가 확인한 보존 미리보기와 대상 지출, 정산 방식 및 결과가 일치하는 경우에만 발행한다. 불일치하면 다른 결과로 자동 발행하지 않고 새 미리보기를 확인하도록 한다.

> 논의 필요
>
> - 미리보기 식별 및 버전 전달 필드, 방식 변경 시 이전 미리보기 무효화와 불일치 오류 응답을 결정한 뒤 정산 확정 요청 Body 예시를 작성한다.
> - 중복 발행의 다시 요청했을 때의 응답, 송금 0건과 처음부터 수령 확인 가능 관계 0건인 경우의 완료 시점을 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

## 정산서 조회 API

### SETTLE-08 여행 정산 목록

`GET /api/v1/trips/{tripId}/settlements`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행의 발행된 정산을 상태별로 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Query Parameter

> 논의 필요
>
> 다음 조회 필드는 검토 제안이다. 선택 필터를 생략하면 해당 조건을 적용하지 않는다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `status` | String | 선택 | 전체 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
> | `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 Array&lt;[SettlementSummary](#settlementsummary)&gt;이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | Array&lt;[SettlementSummary](#settlementsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |
>
> 응답 최상위 `page`는 [Page](common.md#page)를 따른다. 페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.
>
> ```json
> {
>   "data": [
>     {
>       "settlementId": "settlement-example-a",
>       "tripId": "trip-example-a",
>       "status": "IN_PROGRESS",
>       "settlementMethod": "KRW_UNIFIED",
>       "expenseCount": 1,
>       "createdAt": "2026-11-05T03:00:00Z",
>       "totals": [
>         {
>           "currency": "KRW",
>           "amount": "12000.00"
>         }
>       ]
>     }
>   ],
>   "page": {
>     "page": 0,
>     "size": 20,
>     "totalElements": 1,
>     "hasNext": false
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

발행 전 준비 상태인 정산은 이 목록에서 제외한다. 발행 후 취소한 정산은 정산취소 필터로 조회한다.

> 논의 필요
>
> - 발행 시각 기준 정렬 방향과 동률, 취소 이력의 상세 보존 범위를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-09 발행 정산 상세

`GET /api/v1/trips/{tripId}/settlements/{settlementId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행 정산의 요약과 송금 상태를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [SettlementDetail](#settlementdetail)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [SettlementDetail](#settlementdetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "tripId": "trip-example-a",
>     "status": "IN_PROGRESS",
>     "settlementMethod": "KRW_UNIFIED",
>     "expenseCount": 1,
>     "createdAt": "2026-11-05T03:00:00Z",
>     "totals": [
>       {
>         "currency": "KRW",
>         "amount": "12000.00"
>       }
>     ],
>     "tripMemberIds": [
>       "participant-example-a",
>       "participant-example-b"
>     ],
>     "transfers": [
>       {
>         "transferId": "transfer-example-a",
>         "senderTripMemberId": "participant-example-b",
>         "receiverTripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "amount": "6000.00",
>         "receivedAt": null,
>         "isReceivable": true
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

탈퇴자의 표시와 기록 삭제는 [회원 탈퇴 정책](../policy/user.md#공동-기록과-미완료-정산)을 따른다. 탈퇴 전에 확인된 수령 상태는 유지하고 양쪽 모두 탈퇴한 관계는 응답에서 제외한다. 기록 삭제 시 해당 항목과 총 송금액을 숨기고 일부 기록 삭제 안내를 제공한다. 조회로 수령 확인을 바꾸지 않는다.

> 논의 필요
>
> - [정산 취소 정책의 제안](../policy/settlement.md#정산-취소) 채택 여부를 확인한다. 채택 시 취소일과 취소 당시 금액 및 송금 관계만 읽기 전용으로 반환하고 결제 목록, 품목 상세와 액션은 제외한다. 원본 수정 및 재정산과 독립된 결과 보존 방법을 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-10 참여자별 정산서

`GET /api/v1/trips/{tripId}/settlements/{settlementId}/statements/{tripMemberId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지정한 여행 참여자의 정산서와 결제 상세 내역를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
| `tripMemberId` | String | 필수 | 없음 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [Statement](#statement)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Statement](#statement) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "tripId": "trip-example-a",
>     "status": "IN_PROGRESS",
>     "settlementMethod": "KRW_UNIFIED",
>     "expenseCount": 1,
>     "createdAt": "2026-11-05T03:00:00Z",
>     "totals": [
>       {
>         "currency": "KRW",
>         "amount": "12000.00"
>       }
>     ],
>     "tripMemberId": "participant-example-a",
>     "balances": [
>       {
>         "tripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "paidAmount": "12000.00",
>         "shareAmount": "6000.00",
>         "balance": "6000.00"
>       }
>     ],
>     "transfers": [
>       {
>         "transferId": "transfer-example-a",
>         "senderTripMemberId": "participant-example-b",
>         "receiverTripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "amount": "6000.00",
>         "receivedAt": null,
>         "isReceivable": true
>       }
>     ],
>     "expenses": [
>       {
>         "expenseId": "expense-example-a",
>         "paidDate": "2026-11-05",
>         "paymentMethod": "CARD",
>         "currency": "KRW",
>         "exchangeRate": "1.00",
>         "amount": "12000.00",
>         "shareAmount": "6000.00"
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

비로그인 참여자 정산서도 조회 가능하다. 결제 상세는 결제 날짜 오름차순이며 저장 환율을 사용한다.

> 논의 필요
>
> - 공유 이미지의 요약 또는 품목별 상세 범위, 취소와 탈퇴 시 표시 및 지출 정렬값이 같은 경우을 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-11 내 정산 목록

`GET /api/v1/me/settlement-statements`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

내 공간에 표시할 본인의 정산서 목록을 조회한다.

#### 요청

##### Query Parameter

> 논의 필요
>
> 다음 조회 필드는 검토 제안이다. 선택 필터를 생략하면 해당 조건을 적용하지 않는다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `status` | String | 선택 | 전체 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
> | `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 Array&lt;[StatementSummary](#statementsummary)&gt;이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | Array&lt;[StatementSummary](#statementsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |
>
> 응답 최상위 `page`는 [Page](common.md#page)를 따른다. 페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.
>
> ```json
> {
>   "data": [
>     {
>       "settlementId": "settlement-example-a",
>       "tripId": "trip-example-a",
>       "status": "IN_PROGRESS",
>       "settlementMethod": "KRW_UNIFIED",
>       "expenseCount": 1,
>       "createdAt": "2026-11-05T03:00:00Z",
>       "totals": [
>         {
>           "currency": "KRW",
>           "amount": "12000.00"
>         }
>       ],
>       "tripMemberId": "participant-example-a"
>     }
>   ],
>   "page": {
>     "page": 0,
>     "size": 20,
>     "totalElements": 1,
>     "hasNext": false
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

관리자여도 다른 사람 정산서를 내 목록에 넣지 않는다. 여행 접근 차단 후에도 정책상 유지하는 내역을 제공한다.

> 논의 필요
>
> - 기본 정렬 기준과 값이 같은 경우의 순서, 여행 나감 후 상태별 목록 노출을 결정한다. 생성자 탈퇴 후에도 본인 정산서를 내 정산 내역에서 제공한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-12 내 정산 상세

`GET /api/v1/me/settlement-statements/{settlementId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 회원은 본인의 정산서만 조회할 수 있다. 여행방에 계속 접근할 수 있는지와 정산서 조회 가능 여부는 별도로 판단한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

내 공간에서 본인의 정산서와 허용된 계산에 사용한 지출 내역를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [Statement](#statement)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Statement](#statement) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "tripId": "trip-example-a",
>     "status": "IN_PROGRESS",
>     "settlementMethod": "KRW_UNIFIED",
>     "expenseCount": 1,
>     "createdAt": "2026-11-05T03:00:00Z",
>     "totals": [
>       {
>         "currency": "KRW",
>         "amount": "12000.00"
>       }
>     ],
>     "tripMemberId": "participant-example-a",
>     "balances": [
>       {
>         "tripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "paidAmount": "12000.00",
>         "shareAmount": "6000.00",
>         "balance": "6000.00"
>       }
>     ],
>     "transfers": [
>       {
>         "transferId": "transfer-example-a",
>         "senderTripMemberId": "participant-example-b",
>         "receiverTripMemberId": "participant-example-a",
>         "currency": "KRW",
>         "amount": "6000.00",
>         "receivedAt": null,
>         "isReceivable": true
>       }
>     ],
>     "expenses": [
>       {
>         "expenseId": "expense-example-a",
>         "paidDate": "2026-11-05",
>         "paymentMethod": "CARD",
>         "currency": "KRW",
>         "exchangeRate": "1.00",
>         "amount": "12000.00",
>         "shareAmount": "6000.00"
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

여행 상세 접근 권한과 별개로 본인 정산서 보존과 접근을 검증한다. 생성자 탈퇴 후에도 기존 여행명과 필요한 송금 관계 및 금액을 조회하고 본인이 수령인인 관계의 수령 확인을 진행할 수 있다. 탈퇴자 기록 삭제 정책은 동일하게 적용한다. 여행 나감 후 진행 중 정산의 조회 범위는 미정이다.

> 논의 필요
>
> - 여행 나감 후 진행 중 정산 및 취소 정산서의 상세와 근거 조회 범위를 결정한다. 생성자 탈퇴 후에는 최소 기록을 제공하는 응답 모델과 일반 정산서의 구분을 설계한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

## 수령 확인과 취소 API

### SETTLE-13 수령 확인

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/transfers/{transferId}/receipt-confirmation`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 회원 중 해당 송금에서 돈을 받는 사람만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

실제 수령인이 송금 내역 한 건의 수령을 확인한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
| `transferId` | String | 필수 | 없음 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [ReceiptConfirmation](#receiptconfirmation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [ReceiptConfirmation](#receiptconfirmation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "transferId": "transfer-example-a",
>     "receivedAt": "2026-11-05T03:00:00Z",
>     "settlementId": "settlement-example-a",
>     "status": "COMPLETED"
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_RECEIPT_NOT_ALLOWED` | 현재 정산 상태 또는 탈퇴 수령인 관계로 수령 확인할 수 없음 |

#### 처리 규칙

정산이 IN_PROGRESS인지, 요청자가 해당 송금의 수령인인지, 아직 수령 확인하지 않았는지 확인한다. 수령 확인은 한 번만 반영한다. 수령 확인 불가 관계를 제외한 모든 송금의 수령이 확인되면 COMPLETED로 바꾼다. 완료 시 탈퇴자의 송금 관계와 금액을 즉시 삭제한다. 최대 보존 기간 만료로 일부 기록이 삭제돼도 남은 관계의 수령 확인은 계속할 수 있다.

> 논의 필요
>
> - 같은 수령 확인의 반복 응답과 완료 후 재시도, 처음부터 확인 가능한 관계가 0건일 때의 완료를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

### SETTLE-14 발행 정산 취소

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/cancel`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행된 정산을 취소하고 포함 지출을 다시 사용할 수 있게 한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [SettlementCancellation](#settlementcancellation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [SettlementCancellation](#settlementcancellation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "settlementId": "settlement-example-a",
>     "status": "CANCELED"
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `SETTLEMENT_CANCELLATION_NOT_ALLOWED` | 이미 수령 확인이 있거나 취소 가능한 정산 상태가 아님 |

#### 처리 규칙

정산이 IN_PROGRESS이고 수령 확인이 한 건도 없어야 한다. 수령 확인과 정산 취소가 동시에 성공하지 않아야 한다. 취소하면 포함 지출의 잠금을 해제한다. 해당 지출로 다시 정산할 때는 새 정산 ID를 사용한다.

> 논의 필요
>
> - 취소된 송금과 지출 연결 및 당시 금액 보존, 취소 반복 응답과 공유 이미지의 취소 안내를 결정한다.
>
> 출처: [정책](../policy/settlement.md) | [처리 설계](../design/settlement.md) | [잠금](../design/locking.md)

## 공통 모델

> 논의 필요
>
> 아래 일반 정산 모델 외에 탈퇴 및 삭제에 따른 응답 축약 계약이 필요하다. 여행명 보존을 위한 `tripName`, 일부 기록 삭제 안내를 위한 `hasDeletedRecords`, 삭제 시 총 송금액 비노출을 위한 별도 총 송금액 필드와 포함 조건을 검토한다. `totals`의 지출 총액과 총 송금액을 혼용하지 않는다. 삭제된 송금 관계는 배열에서 제외하고, 탈퇴자의 표시 이름은 여행 내에서 번호로 구분한다. 취소 정산서 전용 모델은 취소 노출 제안 채택 후 확정한다.

모델 이름과 필드 구성은 모두 검토 제안이다. 요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### Candidate

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |
> | `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
> | `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 결제 날짜. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
> | `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
> | `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
> | `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |
> | `isSelectable` | Boolean | 아니요 | 항상 | 조회 시점의 선택 가능 여부. |
> | `unavailableReason` | String | 예 | 항상 | 선택 가능하면 null. 불가 사유 코드 집합은 논의 필요. |

### PreparationInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `selectionMode` | String | 필수 | 없음 | EXPLICIT은 ID 개별 선택, FILTER는 필터 전체 선택. 이름은 제안 |
> | `expenseIds` | Array&lt;String&gt; | 조건부 | 없음 | EXPLICIT일 때 같은 여행의 지출 ID 1개 이상. FILTER에서는 보내지 않음 |
> | `filter` | Object | 조건부 | 없음 | FILTER일 때 지출 목록과 동일한 필터 조건. 구체적인 필드는 목록 API와 함께 확정. 페이지 조건 제외 |
> | `excludedExpenseIds` | Array&lt;String&gt; | 조건부 | 없음 | FILTER일 때 선택 해제한 지출 ID. 제외할 것이 없으면 빈 배열. EXPLICIT에서는 보내지 않음 |
>
> FILTER 방식도 준비 요청 시 필터를 새로 실행해 대상을 확장하지 않는다. 전체 선택 시 확정한 범위를 식별할 값이 추가로 필요하며, 선택 토큰 또는 서버 선택 기록 등 구체적인 계약은 별도 설계한다. 필터와 제외 ID만으로 시점 보존이 보장된다고 가정하지 않는다.
>
> 두 선택 방식을 혼합하지 않는다. 필드명과 enum은 제안이며, 필터 전체 선택이라는 동작은 확정했다. 개별 선택의 중복 ID는 제거해 처리한다. 최대 건수는 별도로 정한다.

### Preparation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `status` | String | 아니요 | 항상 | 발행 전 준비 상태인 `DRAFT`를 반환한다. |
> | `expenseIds` | Array&lt;String&gt; | 아니요 | 항상 | 현재 준비 중인 정산의 대상으로 잠근 지출 ID 목록. |
> | `settlementMethod` | String | 예 | 항상 | KRW_UNIFIED는 원화 통합, BY_CURRENCY는 결제 통화별 정산이다. 아직 선택하지 않았으면 null. |
> | `lockToken` | String | 아니요 | 항상 | 현재 정산 준비의 잠금 토큰. 같은 정산 준비의 연장과 확정, 취소 요청에 사용한다. |
> | `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 준비 만료 시각. 예시 시각은 잠금 유지 시간을 확정한 값이 아니다. |
> | `createdAt` | String | 예 | 항상 | ISO 8601 UTC. 발행 전이므로 null. |

### PreparationView

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `status` | String | 아니요 | 항상 | 발행 전 준비 상태인 `DRAFT`를 반환한다. |
> | `expenseIds` | Array&lt;String&gt; | 아니요 | 항상 | 현재 준비 중인 정산의 대상으로 잠근 지출 ID 목록. |
> | `settlementMethod` | String | 예 | 항상 | KRW_UNIFIED는 원화 통합, BY_CURRENCY는 결제 통화별 정산이다. 아직 선택하지 않았으면 null. |
> | `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 준비 만료 시각. 예시 시각은 잠금 유지 시간을 확정한 값이 아니다. |
> | `createdAt` | String | 예 | 항상 | ISO 8601 UTC. 발행 전이므로 null. |

### LockExpiry

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 연장된 작업 만료 시각. |

### MethodInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `settlementMethod` | String | 필수 | 없음 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |

### SettlementPreview

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
> | `expenseCount` | Integer | 아니요 | 항상 | 선택 지출 건수, 1 이상. |
> | `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 정산 통화별 합계. |
> | `balances` | Array&lt;[Balance](#balance)&gt; | 아니요 | 항상 | 참여자와 통화별 잔액. |
> | `transfers` | Array&lt;[TransferPreview](#transferpreview)&gt; | 아니요 | 항상 | 미리보기 송금 내역. 0원 관계는 제외. |

### CurrencyTotal

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 해당 통화의 총액. 소수점 이하 2자리 문자열로 전달한다. |

### Balance

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `paidAmount` | String | 아니요 | 항상 | 해당 참여자의 결제 합계. 소수점 이하 2자리 문자열로 전달한다. |
> | `shareAmount` | String | 아니요 | 항상 | 해당 참여자의 부담 합계. 소수점 이하 2자리 문자열로 전달한다. |
> | `balance` | String | 아니요 | 항상 | 결제액에서 부담액을 뺀 정산 잔액. 받을 돈은 양수, 보낼 돈은 음수이며 계좌 잔고를 뜻하지 않는다. 소수점 이하 2자리 문자열로 전달한다. |

### TransferPreview

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `senderTripMemberId` | String | 아니요 | 항상 | 돈을 보내야 하는 사람의 여행 참여자 ID. |
> | `receiverTripMemberId` | String | 아니요 | 항상 | 돈을 받아야 하는 사람의 여행 참여자 ID. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 0보다 큰 송금액. 소수점 이하 2자리 문자열로 전달한다. |

### SettlementDetail

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
> | `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
> | `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
> | `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
> | `tripMemberIds` | Array&lt;String&gt; | 아니요 | 항상 | 정산서 조회 대상 참여자. |
> | `transfers` | Array&lt;[Transfer](#transfer)&gt; | 아니요 | 항상 | 발행된 송금 내역. 0원 관계 없음. |

### Transfer

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `transferId` | String | 아니요 | 항상 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |
> | `senderTripMemberId` | String | 아니요 | 항상 | 돈을 보내야 하는 사람의 여행 참여자 ID. |
> | `receiverTripMemberId` | String | 아니요 | 항상 | 돈을 받아야 하는 사람의 여행 참여자 ID. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 0보다 큰 송금액. 소수점 이하 2자리 문자열로 전달한다. |
> | `receivedAt` | String | 예 | 항상 | ISO 8601 UTC. 미확인이면 null. |
> | `isReceivable` | Boolean | 아니요 | 항상 | 이 송금 내역의 수령 확인이 가능한지 표시한다. 요청자가 확인 버튼을 누를 권한이 있다는 뜻은 아니다. 수령인이 탈퇴했으면 false. |

### SettlementSummary

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
> | `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
> | `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
> | `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |

### Statement

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
> | `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
> | `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
> | `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
> | `balances` | Array&lt;[Balance](#balance)&gt; | 아니요 | 항상 | 해당 참여자의 결제액과 부담액 및 잔액. |
> | `transfers` | Array&lt;[Transfer](#transfer)&gt; | 아니요 | 항상 | 해당 참여자가 보내거나 받는 관계. |
> | `expenses` | Array&lt;[StatementExpense](#statementexpense)&gt; | 아니요 | 항상 | 결제 날짜 오름차순, 동률 지출 ID 기준 제안. |

### StatementExpense

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |
> | `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 결제 날짜, 오름차순. |
> | `paymentMethod` | String | 아니요 | 항상 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `exchangeRate` | String | 아니요 | 항상 | 저장 당시 환율. 소수점 이하 2자리 문자열로 전달한다. |
> | `amount` | String | 아니요 | 항상 | 지출 총액. 소수점 이하 2자리 문자열로 전달한다. |
> | `shareAmount` | String | 아니요 | 항상 | 해당 정산서 참여자의 부담액. 소수점 이하 2자리 문자열로 전달한다. |

### StatementSummary

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
> | `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
> | `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
> | `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
> | `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

### ReceiptConfirmation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `transferId` | String | 아니요 | 항상 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |
> | `receivedAt` | String | 아니요 | 항상 | ISO 8601 UTC. 수령 확인 시각. |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `status` | String | 아니요 | 항상 | 아직 확인할 송금이 남으면 `IN_PROGRESS`, 완료 조건을 충족하면 `COMPLETED`다. |

### SettlementCancellation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 해당 여행의 정산이어야 한다. |
> | `status` | String | 아니요 | 항상 | 발행 후 취소 상태인 `CANCELED`를 반환한다. |

## 오류 코드

> 논의 필요
>
> 도메인 오류 코드의 이름, HTTP 대응과 안내 문구는 제안이다. 공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다. 개별 API의 오류 조건과 도메인 코드 이름은 팀 검토 후 확정한다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 409 | `SETTLEMENT_CANCELLATION_NOT_ALLOWED` | 이미 수령 확인이 있거나 취소 가능한 정산 상태가 아님 |
> | 409 | `SETTLEMENT_EXPENSE_UNAVAILABLE` | 선택한 지출 중 한 건 이상이 수정 중이거나 다른 정산에 포함돼 있음 |
> | 409 | `SETTLEMENT_PREPARATION_INVALID` | 준비가 만료되었거나 취소 또는 발행되어 확정할 수 없음 |
> | 409 | `SETTLEMENT_RECEIPT_NOT_ALLOWED` | 현재 정산 상태 또는 탈퇴 수령인 관계로 수령 확인할 수 없음 |
