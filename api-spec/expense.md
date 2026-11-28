# 지출 API 스펙 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [지출 조회와 저장 API](#지출-조회와-저장-api)
  - [EXP-01 지출 목록](#exp-01-지출-목록)
  - [EXP-02 지출 상세](#exp-02-지출-상세)
  - [EXP-03 지출 생성](#exp-03-지출-생성)
- [수정 잠금과 변경 API](#수정-잠금과-변경-api)
  - [EXP-04 수정 잠금 획득](#exp-04-수정-잠금-획득)
  - [EXP-05 수정 잠금 연장](#exp-05-수정-잠금-연장)
  - [EXP-06 수정 잠금 해제](#exp-06-수정-잠금-해제)
  - [EXP-07 지출 수정](#exp-07-지출-수정)
  - [EXP-08 지출 삭제](#exp-08-지출-삭제)
- [영수증 인식 API](#영수증-인식-api)
  - [EXP-09 영수증 인식](#exp-09-영수증-인식)
- [공통 모델](#공통-모델)
  - [ExpenseSummary](#expensesummary)
  - [ExpenseDetail](#expensedetail)
  - [ExpenseItem](#expenseitem)
  - [Share](#share)
  - [ExpenseInput](#expenseinput)
  - [ExpenseItemInput](#expenseiteminput)
  - [EditLock](#editlock)
  - [LockExpiry](#lockexpiry)
  - [RecognitionResult](#recognitionresult)
  - [RecognizedItem](#recognizeditem)
- [오류 코드](#오류-코드)

## 문서 범위

입력과 계산 결과 확인, 최종 저장을 구분한다. 결제자와 기록자, 부담자는 서로 다를 수 있다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/expense.md) | [설계](../design/expense.md)

API 식별자는 화면 정의서 식별자와 별개다.

## API 목록

| 기능 | Method | Path | 인증 | operationId |
| --- | --- | --- | --- | --- |
| [지출 목록](#exp-01-지출-목록) | `GET` | `/api/v1/trips/{tripId}/expenses` | 여행 참여자 | `listExpenses` |
| [지출 상세](#exp-02-지출-상세) | `GET` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 여행 참여자 | `getExpense` |
| [지출 생성](#exp-03-지출-생성) | `POST` | `/api/v1/trips/{tripId}/expenses` | 여행 참여자 (여행 확정 후) | `createExpense` |
| [수정 잠금 획득](#exp-04-수정-잠금-획득) | `POST` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자 | `acquireExpenseEditLock` |
| [수정 잠금 연장](#exp-05-수정-잠금-연장) | `PATCH` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 | `extendExpenseEditLock` |
| [수정 잠금 해제](#exp-06-수정-잠금-해제) | `DELETE` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 | `releaseExpenseEditLock` |
| [지출 수정](#exp-07-지출-수정) | `PUT` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 | `updateExpense` |
| [지출 삭제](#exp-08-지출-삭제) | `DELETE` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 지출 기록자 또는 여행 관리자 | `deleteExpense` |
| [영수증 인식](#exp-09-영수증-인식) | `POST` | `/api/v1/trips/{tripId}/receipt-recognitions` | 여행 참여자 (여행 확정 후) | `recognizeReceipt` |

operationId는 백엔드 OpenAPI [`expense.yml`](https://github.com/JECT-Study/Billit-5th-Server/blob/develop/src/main/resources/api-docs/domains/expense.yml)의 이름이다.

서버가 최종 결제액, 참여자별 부담액과 원화 환산액을 계산하며, 저장 요청에는 계산 결과를 넣지 않는다. 근거: [Issue #82 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/82)
계산 기준은 [분배와 할인](../policy/expense.md#분배와-할인)과 [환산과 자투리](../policy/expense.md#환산과-자투리)를 따른다. 1차 MVP에서는 계산된 개인 부담액의 수기 수정을 제공하지 않는다. 직접 수정한 환율의 자동 저장은 [환율 정책](../policy/exchange-rate.md#지출-환율)을 따른다.

탈퇴자의 결제 목록은 표시 이름과 금액만 제공하고 상세 품목과 개인 리시트는 노출하지 않는다.

1차 MVP에서는 입력 중인 지출을 앱이 기기에만 저장하므로 임시 저장 API를 두지 않는다([입력과 저장](../policy/expense.md#입력과-저장)).
지출 생성 API는 입력을 마친 지출만 저장하며 불완전한 입력의 임시 저장에 쓰지 않는다.

## 지출 조회와 저장 API

### EXP-01 지출 목록

`GET /api/v1/trips/{tripId}/expenses`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행의 날짜, 기간 조건에 맞는 지출 목록을 조회한다. 정산 대상 조회도 이 API로 통합하며 별도의 정산 대상 목록 API를 제공하지 않는다. 정산 준비 시작은 별도 API에서 관리자 권한과 선택한 지출의 현재 상태를 다시 검증한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `fromDate` | String | 선택 | 없음 | YYYY-MM-DD. 현지 결제 날짜 기준이며 이 날짜를 포함한다. `toDate`보다 뒤면 400. |
| `toDate` | String | 선택 | 없음 | YYYY-MM-DD. 현지 결제 날짜 기준이며 이 날짜를 포함한다. |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[ExpenseSummary](#expensesummary)&gt;이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[ExpenseSummary](#expensesummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.

```json
{
  "data": [
    {
      "expenseId": "expense-example-a",
      "name": "식사",
      "paidDate": "2026-11-05",
      "currency": "KRW",
      "amount": "12000.00",
      "payerTripMemberId": "participant-example-a",
      "settlementId": null,
      "isEditLocked": false
    }
  ],
  "page": {
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "hasNext": false
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

서버는 여행 기간 밖의 지출도 반환한다.
서버는 삭제한 지출을 반환하지 않는다.
서버는 결제 날짜가 늦은 지출부터, 결제 날짜가 같으면 지출 ID가 큰 지출부터 반환한다.
정렬 기준은 고정이며 요청으로 바꿀 수 없다. 근거: [Issue #80 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/80)
확정 전 여행에는 지출이 없으므로 서버는 빈 목록을 반환한다.

### EXP-02 지출 상세

`GET /api/v1/trips/{tripId}/expenses/{expenseId}`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출의 결제 정보와 품목, 최종 부담액, 저장 환율을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "expenseId": "expense-example-a",
    "name": "식사",
    "paidDate": "2026-11-05",
    "currency": "KRW",
    "amount": "12000.00",
    "payerTripMemberId": "participant-example-a",
    "settlementId": null,
    "isEditLocked": false,
    "tripId": "trip-example-a",
    "splitType": "EQUAL",
    "createdByTripMemberId": "participant-example-a",
    "paymentMethod": "CARD",
    "originalAmount": "12000.00",
    "discountAmount": "0.00",
    "exchangeRate": "1.00",
    "exchangeRateSource": "MANUAL",
    "exchangeRateProvider": null,
    "exchangeRateBaseDate": null,
    "amountKrw": "12000.00",
    "items": [],
    "shares": [
      {
        "tripMemberId": "participant-example-a",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      },
      {
        "tripMemberId": "participant-example-b",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      }
    ]
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 여행이나 지출이 없거나 삭제됐거나, 지출이 이 여행의 지출이 아님 |

#### 처리 규칙

지출을 수정하는 중이면 서버는 기존 저장값을 반환한다.
회원은 발행된 정산에 포함된 지출도 조회할 수 있다.
서버는 품목을 `sortOrder` 오름차순으로, 참여자별 부담액과 품목 부담자를 여행 참여자 ID 오름차순으로 반환한다.
탈퇴자의 품목과 금액은 [공동 기록과 미완료 정산](../policy/user.md#공동-기록과-미완료-정산)에 따라 표시한다.

### EXP-03 지출 생성

`POST /api/v1/trips/{tripId}/expenses`

#### 인증

여행 확정 후 로그인한 여행 참여자가 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

사용자가 입력값과 최종 부담액을 확인한 지출을 저장한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | String | 필수 | 없음 | UUID 형식(8-4-4-4-12자리 16진수)의 지출 생성 고유키. 앱은 저장 시도마다 새 키를 만들고, 응답을 받지 못해 같은 저장을 다시 보낼 때만 같은 키를 사용한다. 없거나 형식이 다르면 400. |

##### Body

Content-Type: `application/json`

요청 필드: [ExpenseInput](#expenseinput).

```json
{
  "name": "식사",
  "splitType": "EQUAL",
  "currency": "KRW",
  "paidDate": "2026-11-05",
  "paymentMethod": "CARD",
  "payerTripMemberId": "participant-example-a",
  "originalAmount": "12000.00",
  "discountAmount": "0.00",
  "exchangeRate": "1.00",
  "exchangeRateSource": "MANUAL",
  "tripMemberIds": [
    "participant-example-a",
    "participant-example-b"
  ]
}
```

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "expenseId": "expense-example-a",
    "name": "식사",
    "paidDate": "2026-11-05",
    "currency": "KRW",
    "amount": "12000.00",
    "payerTripMemberId": "participant-example-a",
    "settlementId": null,
    "isEditLocked": false,
    "tripId": "trip-example-a",
    "splitType": "EQUAL",
    "createdByTripMemberId": "participant-example-a",
    "paymentMethod": "CARD",
    "originalAmount": "12000.00",
    "discountAmount": "0.00",
    "exchangeRate": "1.00",
    "exchangeRateSource": "MANUAL",
    "exchangeRateProvider": null,
    "exchangeRateBaseDate": null,
    "amountKrw": "12000.00",
    "items": [],
    "shares": [
      {
        "tripMemberId": "participant-example-a",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      },
      {
        "tripMemberId": "participant-example-b",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      }
    ]
  }
}
```

HTTP `201 Created`와 생성한 지출의 주소(`/api/v1/trips/{tripId}/expenses/{expenseId}`)를 담은 `Location` 헤더를 반환한다.
같은 회원이 같은 여행에서 같은 고유키로 다시 요청해도 처음과 같이 `201 Created`와 `Location`을 반환하며, `data`에는 처음 만든 지출의 현재 상세를 담는다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `Idempotency-Key`가 없거나 UUID 형식이 아님 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 400 | `COMMON_INVALID_REQUEST` | `splitType`에 맞지 않게 `tripMemberIds`나 `items`를 보냄 |
| 400 | `COMMON_INVALID_REQUEST` | 최소 단위보다 작은 금액, KRW 지출의 `1.00`이 아닌 환율, 품목 금액보다 큰 상품 할인, 0 미만인 최종 결제액이나 정수부가 10자리를 넘는 원화 환산액 |
| 400 | `EXPENSE_INVALID_SHARES` | 결제자가 참여 중인 로그인 참여자가 아니거나, 부담자가 없거나 겹치거나 참여 중인 로그인 참여자나 비로그인 참여자가 아님 |
| 400 | `EXPENSE_AMOUNT_MISMATCH` | 품목별 지출에서 상품 할인 전 품목 금액의 합계가 `originalAmount`와 다름 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 요청한 회원이 이 여행에 참여 중이 아님 |
| 404 | `COMMON_NOT_FOUND` | 여행이 없거나 삭제됨 |
| 409 | `EXPENSE_TRIP_NOT_CONFIRMED` | 여행 확정 전에 지출을 등록하려 함 |
| 409 | `EXPENSE_IDEMPOTENCY_KEY_REUSED` | 다른 회원이나 다른 여행에서 이미 사용한 고유키이거나, 그 키로 만든 지출이 삭제됨 |

#### 처리 규칙

요청한 회원이 지출의 기록자가 된다.
결제자가 이 여행에 참여 중인 로그인 참여자가 아니면 서버는 요청을 거절한다.
부담자가 이 여행에 참여 중인 로그인 참여자나 비로그인 참여자가 아니면 서버는 요청을 거절한다.
서버는 [분배와 할인](../policy/expense.md#분배와-할인)과 [환산과 자투리](../policy/expense.md#환산과-자투리)의 기준으로 최종 결제액, 참여자별 부담액과 원화 환산액을 계산하고 지출, 품목과 부담액을 한 번에 저장한다.
하나라도 저장에 실패하면 서버는 아무것도 저장하지 않고, 저장 고유키도 성공으로 기록하지 않는다.

서버는 요청 본문의 형식, 여행과 참여 여부를 확인한 뒤 고유키를 조회한다.
같은 회원이 같은 여행에서 같은 고유키로 다시 보낸 요청이면 서버는 여행 확정 여부와 금액을 다시 검증하지 않고 처음 만든 지출을 반환한다.
서버는 같은 고유키로 보낸 요청의 본문을 처음 요청과 비교하지 않는다.
같은 고유키의 요청이 동시에 오면 서버는 지출을 하나만 저장하고 모든 요청에 그 지출을 반환한다.
고유키는 만료되지 않으며, 처리 기준은 [지출 생성 멱등성](../design/expense.md#지출-생성-멱등성)을 따른다.

## 수정 잠금과 변경 API

### EXP-04 수정 잠금 획득

`POST /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

#### 인증

그 여행방에 접근할 수 있는 로그인 회원 중 이 지출을 기록한 회원 또는 여행 관리자만 요청할 수 있다. 결제자라는 이유만으로 수정 권한을 주지는 않는다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출 수정 작업의 독점 권한과 만료 시각을 획득한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [EditLock](#editlock)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [EditLock](#editlock) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "lockToken": "5f0c7a2e-9b1d-4c3e-8a6f-2d4b6c8e0a1f",
    "expiresAt": "2026-11-05T03:05:00Z"
  }
}
```

잠금을 조회하는 API가 없으므로 서버는 `Location` 헤더를 반환하지 않는다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `EXPENSE_EDIT_LOCKED` | 만료되지 않은 수정 잠금이 있음. 요청한 회원의 잠금이어도 같음 |
| 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |

#### 처리 규칙

지출이 정산 대상으로 잠겨 있거나 다른 수정 작업으로 잠겨 있으면 서버는 잠금 획득을 거절한다.
같은 회원이 다시 요청해도 서버는 현재 유효한 수정 잠금을 새 토큰으로 바꾸지 않는다.
만료된 잠금이 있으면 서버는 그 잠금을 새 잠금으로 바꾼다.
잠금의 유지 시간은 [수정과 잠금 정책](../policy/expense.md#수정과-잠금)을 따른다.

### EXP-05 수정 잠금 연장

`PATCH /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

#### 인증

그 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 지출 수정 작업의 유효 기간을 연장한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | [EXP-04 수정 잠금 획득](#exp-04-수정-잠금-획득) 응답의 `lockToken`. UUID 형식(8-4-4-4-12자리 16진수)이며 없거나 형식이 다르면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [LockExpiry](#lockexpiry)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [LockExpiry](#lockexpiry) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "expiresAt": "2026-11-05T03:07:00Z"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 소유자와 토큰이 일치하지 않거나 잠금이 만료됨 |

#### 처리 규칙

서버는 요청자가 현재 잠금을 획득한 회원인지, 토큰이 현재 수정 작업과 같은지, 잠금이 만료되기 전인지 확인한다.
만료된 잠금의 연장을 요청하면 서버는 요청을 거절한다.
유효한 연장 요청이면 서버는 기존 만료 시각에 2분을 더한다.
같은 잠금에 연장 요청이 여러 번 도착하면 서버는 요청마다 2분을 더한다.
응답의 `expiresAt`은 그 요청을 반영한 뒤의 만료 시각이다.

### EXP-06 수정 잠금 해제

`DELETE /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

#### 인증

그 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 지출 수정 작업을 종료하고 잠금을 해제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | [EXP-04 수정 잠금 획득](#exp-04-수정-잠금-획득) 응답의 `lockToken`. UUID 형식(8-4-4-4-12자리 16진수)이며 없거나 형식이 다르면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환한다.
응답 Body는 없다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `EXPENSE_EDIT_LOCK_INVALID` | 다른 토큰이나 다른 회원의 만료되지 않은 잠금이 있음 |

#### 처리 규칙

요청한 회원의 잠금이고 토큰이 같으면 서버는 만료 여부와 관계없이 잠금을 해제한다.
지출이 정산에 포함돼 있어도 서버는 토큰이 맞는 잠금을 해제한다.
잠금이 이미 해제됐거나 만료됐으면 서버는 같은 요청을 다시 보낸 것으로 보고 `204 No Content`로 응답한다.
토큰이 다른 새 작업의 잠금이면 서버는 잠금을 해제하지 않는다.

### EXP-07 지출 수정

`PUT /api/v1/trips/{tripId}/expenses/{expenseId}`

#### 인증

그 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 잠금으로 지출을 수정하고, 서버가 다시 계산한 부담액을 함께 저장한다.
앱은 수정 가능한 입력값 전체를 PUT으로 보내며, 서버는 요청 전체를 검증하고 지출 생성과 동일한 계산 기준을 적용한다.
생략한 필드를 기존 값으로 유지하는 PATCH 방식으로 처리하지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | [EXP-04 수정 잠금 획득](#exp-04-수정-잠금-획득) 응답의 `lockToken`. UUID 형식(8-4-4-4-12자리 16진수)이며 없거나 형식이 다르면 400. |

##### Body

Content-Type: `application/json`

요청 필드: [ExpenseInput](#expenseinput).

```json
{
  "name": "식사",
  "splitType": "EQUAL",
  "currency": "KRW",
  "paidDate": "2026-11-05",
  "paymentMethod": "CARD",
  "payerTripMemberId": "participant-example-a",
  "originalAmount": "12000.00",
  "discountAmount": "0.00",
  "exchangeRate": "1.00",
  "exchangeRateSource": "MANUAL",
  "tripMemberIds": [
    "participant-example-a",
    "participant-example-b"
  ]
}
```

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "expenseId": "expense-example-a",
    "name": "식사",
    "paidDate": "2026-11-05",
    "currency": "KRW",
    "amount": "12000.00",
    "payerTripMemberId": "participant-example-a",
    "settlementId": null,
    "isEditLocked": false,
    "tripId": "trip-example-a",
    "splitType": "EQUAL",
    "createdByTripMemberId": "participant-example-a",
    "paymentMethod": "CARD",
    "originalAmount": "12000.00",
    "discountAmount": "0.00",
    "exchangeRate": "1.00",
    "exchangeRateSource": "MANUAL",
    "exchangeRateProvider": null,
    "exchangeRateBaseDate": null,
    "amountKrw": "12000.00",
    "items": [],
    "shares": [
      {
        "tripMemberId": "participant-example-a",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      },
      {
        "tripMemberId": "participant-example-b",
        "amount": "6000.00",
        "amountKrw": "6000.00"
      }
    ]
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `X-Lock-Token`이 없거나 UUID 형식이 아님 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 400 | `COMMON_INVALID_REQUEST` | `splitType`에 맞지 않게 `tripMemberIds`나 `items`를 보냄 |
| 400 | `COMMON_INVALID_REQUEST` | 최소 단위보다 작은 금액, KRW 지출의 `1.00`이 아닌 환율, 품목 금액보다 큰 상품 할인, 0 미만인 최종 결제액이나 정수부가 10자리를 넘는 원화 환산액 |
| 400 | `EXPENSE_INVALID_SHARES` | 새로 지정한 결제자가 참여 중인 로그인 참여자가 아니거나, 부담자가 없거나 겹치거나 새로 지정한 부담자가 참여 중인 로그인 참여자나 비로그인 참여자가 아님 |
| 400 | `EXPENSE_AMOUNT_MISMATCH` | 품목별 지출에서 상품 할인 전 품목 금액의 합계가 `originalAmount`와 다름 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 요청한 회원이 이 여행에 참여 중이 아니거나, 지출 기록자도 여행 관리자도 아님 |
| 404 | `COMMON_NOT_FOUND` | 여행이나 지출이 없거나 삭제됐거나, 지출이 이 여행의 지출이 아님 |
| 409 | `EXPENSE_EDIT_LOCK_INVALID` | 요청한 회원의 잠금이 아니거나, 토큰이 현재 수정 작업과 다르거나, 잠금이 없거나 만료됨 |
| 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |

#### 처리 규칙

서버는 수정 잠금이 만료되지 않았고 지출이 정산 대상으로 잠기지 않았는지 다시 확인한다.
서버는 지출, 품목과 부담액의 변경과 잠금 해제를 한 번에 반영하며, 하나라도 실패하면 모두 취소한다.
저장에 실패하면 서버는 기존 지출과 수정 잠금을 유지한다.

서버는 새로 지정한 결제자와 부담자를 [EXP-03 지출 생성](#exp-03-지출-생성)과 같은 기준으로 검증한다.
수정 전 지출의 결제자를 결제자로, 부담자를 부담자로 다시 보내면 서버는 그 회원이 여행을 나갔거나 탈퇴했어도 거절하지 않는다.
품목별 지출이면 서버는 수정 전 지출의 부담자를 어느 품목의 부담자로 보내도 거절하지 않는다.
결제자와 부담자의 수정 기준은 [수정과 잠금](../policy/expense.md#수정과-잠금)을 따른다.

서버는 기존 품목, 품목별 부담자와 참여자별 부담액을 모두 삭제하고 요청대로 다시 저장한다. 근거: [Issue #88 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/88)
서버가 품목을 다시 저장하므로 `expenseItemId`는 수정할 때마다 바뀐다.

서버는 지출 수정에 `Idempotency-Key`를 쓰지 않는다. 근거: [Issue #88 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/88)
수정에 성공한 뒤 같은 토큰으로 다시 수정을 요청하면 서버는 잠금이 없으므로 `409 EXPENSE_EDIT_LOCK_INVALID`로 거절한다.
수정 응답을 받지 못하면 앱은 지출 상세를 다시 조회해 수정이 반영됐는지 확인한다.

### EXP-08 지출 삭제

`DELETE /api/v1/trips/{tripId}/expenses/{expenseId}`

#### 인증

그 여행방에 접근할 수 있는 로그인 회원 중 이 지출을 기록한 회원 또는 여행 관리자만 요청할 수 있다. 결제자라는 이유만으로 삭제 권한을 주지는 않는다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출을 삭제한다.
서버는 삭제한 지출을 [EXP-01 지출 목록](#exp-01-지출-목록)과 [EXP-02 지출 상세](#exp-02-지출-상세)에서 반환하지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환한다.
응답 Body는 없다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `tripId`나 `expenseId`가 숫자 ID 형식이 아님 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 요청한 회원이 이 여행에 참여 중이 아니거나, 지출 기록자도 여행 관리자도 아님 |
| 404 | `COMMON_NOT_FOUND` | 여행이 없거나 삭제됐거나, 지출이 없거나 이미 삭제됐거나 이 여행의 지출이 아님 |
| 409 | `EXPENSE_EDIT_LOCKED` | 만료되지 않은 수정 잠금이 있음. 요청한 회원의 잠금이어도 같음 |
| 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |

#### 처리 규칙

수정이나 정산으로 잠긴 지출이면 서버는 삭제하지 않는다.
요청한 회원이 획득한 수정 잠금이 유효해도 서버는 `409 EXPENSE_EDIT_LOCKED`로 거절한다. 근거: [Issue #89 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/89)
서버는 지출 삭제에 `X-Lock-Token`을 받지 않으므로 수정 잠금의 소유자를 구분하지 않는다.
회원이 수정 화면에서 지출을 삭제하면 앱은 삭제를 요청하기 전에 [EXP-06 수정 잠금 해제](#exp-06-수정-잠금-해제)로 수정 잠금을 해제한다.

이미 삭제한 지출을 다시 삭제하면 서버는 `404 COMMON_NOT_FOUND`를 반환한다. 근거: [Issue #89 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/89)
삭제 응답을 받지 못해 다시 보낸 요청에도 서버는 `404 COMMON_NOT_FOUND`를 반환한다.

삭제한 지출의 생성 고유키로 지출 생성을 다시 요청하면 서버는 [EXP-03 지출 생성](#exp-03-지출-생성)의 `409 EXPENSE_IDEMPOTENCY_KEY_REUSED`로 거절한다.

## 영수증 인식 API

### EXP-09 영수증 인식

`POST /api/v1/trips/{tripId}/receipt-recognitions`

#### 인증

여행 확정 후 로그인한 여행 참여자가 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

영수증 한 장에서 지출 입력값을 동기로 추출하고, 같은 HTTP 요청의 응답으로 사용자가 확인하고 수정할 인식 결과를 반환한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Part

Content-Type: `multipart/form-data`

| Part | Content-Type | 필수 | 규칙 |
| --- | --- | --- | --- |
| `image` | `image/jpeg`, `image/png`, `image/heic`, `image/webp` | 필수 | 이미지 1개. 최대 15 MiB(15,728,640바이트). Part의 Content-Type과 파일 내용의 형식이 같아야 한다. |
| `request` | `application/json` | 필수 | 분배 방식(`splitType`)을 전달한다. 정의하지 않은 필드가 있으면 400. |

요청 전체는 최대 16 MiB다.

JSON Part `request`의 필드:

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `splitType` | String | 필수 | 없음 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |

```json
{
  "splitType": "EQUAL"
}
```

#### 성공 응답

인식 처리에 성공하면 HTTP `200 OK`로 결과를 반환한다.

공통 응답의 `data`는 [RecognitionResult](#recognitionresult)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [RecognitionResult](#recognitionresult) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "name": "식사",
    "paidDate": "2026-11-05",
    "currency": "KRW",
    "originalAmount": "12000.00",
    "discountAmount": null,
    "items": []
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 400 | `EXPENSE_NOT_RECEIPT` | 인식 서비스가 이미지를 영수증이 아니라고 판단함 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `EXPENSE_TRIP_NOT_CONFIRMED` | 확정 전 여행 |
| 413 | `EXPENSE_RECEIPT_TOO_LARGE` | 이미지가 15 MiB를 넘거나 요청 전체가 16 MiB를 넘음. 서버 앞단에서 먼저 거절하면 응답 Body 없이 413이 올 수 있음 |
| 415 | `EXPENSE_RECEIPT_UNSUPPORTED` | `image` Part의 Content-Type이 JPEG, PNG, HEIC, WebP가 아니거나 파일 내용이 그 형식이 아니거나 비어 있음 |
| 415 | `COMMON_UNSUPPORTED_MEDIA_TYPE` | 요청이 `multipart/form-data`가 아니거나 `request` Part의 Content-Type이 `application/json`이 아님 |
| 503 | `EXPENSE_RECOGNITION_UNAVAILABLE` | 인식 기능이 꺼져 있거나 외부 인식 서비스의 장애, 시간 초과, 오류 응답이나 해석할 수 없는 응답으로 인식을 마치지 못함. 서버 앞단의 동시 요청 수를 넘었을 때도 같음 |

#### 처리 규칙

서버는 인식만으로 지출을 저장하지 않는다.
인식에 실패하면 회원은 다시 시도하거나 직접 입력할 수 있다.
서버는 작업을 접수한 뒤 따로 조회하는 비동기 방식을 쓰지 않는다.
서버는 외부 인식 서비스 호출을 재시도하지 않는다.
서버는 응답에 카드번호, 카드 승인번호, 영수증 번호, 카드사, 가맹점 사업자번호, 주소와 전화번호를 넣지 않는다.
서버는 원본 이미지를 저장하지 않으며, 이미지 처리와 시간 제한 기준은 [지출 설계](../design/expense.md#영수증-이미지-처리)를 따른다.

`splitType`이 `EQUAL`이면 서버는 `items`를 빈 배열로 반환하고, 품목의 상품 할인을 `discountAmount`에 더한다.
`splitType`이 `BY_ITEM`이면 서버는 인식한 품목과 품목별 할인을 그대로 반환한다.
통화가 KRW, JPY, USD, EUR, CNY가 아니면 서버는 `currency`를 `null`로 반환한다.
금액이 0 미만이거나 정수부가 10자리를 넘거나 소수점 이하가 2자리를 넘으면 서버는 그 금액을 `null`로 반환한다.

일부 필드를 인식하지 못해도 서버는 인식한 부분 결과를 `200 OK`로 반환한다.
서버는 인식하지 못한 단일 필드와 품목의 필드를 `null`로 반환한다.
인식한 품목이 없으면 서버는 `items`를 빈 배열로 반환한다.
서버는 인식하지 못한 값을 0이나 임의의 값으로 채우지 않는다.
회원은 인식 결과를 검수하고 빠진 값을 채운 뒤 따로 지출 저장을 요청한다.

인식을 마쳤지만 아무 필드도 인식하지 못하면 서버는 빈 결과를 `200 OK`로 반환한다.
빈 결과이면 서버는 단일 필드를 모두 `null`로, `items`를 빈 배열로 반환하며 `data` 자체는 `null`로 반환하지 않는다.
외부 인식 서비스의 장애나 시간 초과로 처리를 마치지 못하면 서버는 빈 인식 결과로 대신 응답하지 않는다.

다음은 빈 인식 결과의 예시다.

```json
{
  "data": {
    "name": null,
    "paidDate": null,
    "currency": null,
    "originalAmount": null,
    "discountAmount": null,
    "items": []
  }
}
```

## 공통 모델

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### ExpenseSummary

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |
| `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
| `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. UTC 변환 없음. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
| `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
| `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
| `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |

### ExpenseDetail

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |
| `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
| `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. UTC 변환 없음. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
| `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
| `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
| `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `splitType` | String | 아니요 | 항상 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
| `createdByTripMemberId` | String | 아니요 | 항상 | 지출을 등록한 회원의 여행 참여자 ID. 실제 결제자와 다를 수 있다. |
| `paymentMethod` | String | 아니요 | 항상 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `originalAmount` | String | 아니요 | 항상 | 할인 전 총액. 소수점 이하 2자리 문자열로 전달한다. |
| `discountAmount` | String | 아니요 | 항상 | 전체 할인액. 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeRate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액, 0보다 큼. 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeRateSource` | String | 아니요 | 항상 | `AUTO`, `MANUAL` 중 하나. AUTO는 API에서 가져온 환율, MANUAL은 개인 기록 또는 직접 입력한 환율이다. 제공 기관은 exchangeRateProvider로 구분한다. |
| `exchangeRateProvider` | String | 예 | 항상 | API 환율의 제공 기관. 저장한 제공 기관이 없으면 null. |
| `exchangeRateBaseDate` | String | 예 | 항상 | YYYY-MM-DD. API 환율의 실제 기준일. 저장한 기준일이 없으면 null. |
| `amountKrw` | String | 아니요 | 항상 | 원화 환산 결제액. 1원 단위이며 소수 부분이 `.00`인 소수점 이하 2자리 문자열로 전달한다. |
| `items` | Array&lt;[ExpenseItem](#expenseitem)&gt; | 아니요 | 항상 | 품목별 저장 내역을 `sortOrder` 오름차순으로 담음. EQUAL이면 빈 배열. |
| `shares` | Array&lt;[Share](#share)&gt; | 아니요 | 항상 | 참여자별 최종 부담액을 여행 참여자 ID 오름차순으로 담음. 부담액 합계는 `amount`, 원화 부담액 합계는 `amountKrw`와 같음. |

### ExpenseItem

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expenseItemId` | String | 아니요 | 항상 | 지출 품목 ID. 그 지출에 속한 품목을 구분한다. 지출을 수정할 때마다 바뀐다. |
| `name` | String | 예 | 항상 | 품목명. 미입력이면 null. |
| `amount` | String | 아니요 | 항상 | 상품 할인 전 품목 금액. 소수점 이하 2자리 문자열로 전달한다. |
| `discountAmount` | String | 아니요 | 항상 | 그 품목 할인액. 소수점 이하 2자리 문자열로 전달한다. |
| `sortOrder` | Integer | 아니요 | 항상 | 0부터 시작하는 표시 순서. 서버가 요청한 품목 순서대로 매긴다. |
| `tripMemberIds` | Array&lt;String&gt; | 아니요 | 항상 | 품목 부담자의 여행 참여자 ID. ID 오름차순. |

### Share

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
| `amount` | String | 아니요 | 항상 | 결제 통화 기준 최종 부담액. 소수점 이하 2자리 문자열로 전달한다. |
| `amountKrw` | String | 아니요 | 항상 | 원화 환산 부담액. 1원 단위이며 소수 부분이 `.00`인 소수점 이하 2자리 문자열로 전달한다. |

### ExpenseInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `name` | String | 필수 | 없음 | 지출명. 1~50자이며 공백이 아닌 문자를 하나 이상 포함한다. |
| `splitType` | String | 필수 | 없음 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
| `currency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `paidDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |
| `paymentMethod` | String | 필수 | 없음 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `payerTripMemberId` | String | 필수 | 없음 | 실제 결제자의 여행 참여자 ID. 이 여행에 참여 중인 로그인 참여자만 지정할 수 있다. 지출 수정의 예외는 [EXP-07 지출 수정](#exp-07-지출-수정)의 처리 규칙을 따른다. |
| `originalAmount` | String | 필수 | 없음 | 할인 전 총액. 0 이상이고 정수부 10자리 이하인 소수점 이하 2자리 문자열. 결제 통화의 최소 단위보다 작은 소수는 받지 않으며 KRW와 JPY는 소수 부분이 `.00`이어야 한다. |
| `discountAmount` | String | 선택 | "0.00" | 전체 할인액. 형식은 `originalAmount`와 같다. null은 보낼 수 없다. |
| `exchangeRate` | String | 필수 | 없음 | 결제 통화 1단위의 원화 금액. 0보다 크고 정수부 10자리 이하인 소수점 이하 2자리 문자열. KRW 지출은 `1.00`이어야 한다. |
| `exchangeRateSource` | String | 필수 | 없음 | `AUTO`, `MANUAL` 중 하나. |
| `tripMemberIds` | Array&lt;String&gt; | 조건부 | 없음 | splitType=EQUAL이면 필수이며 1명 이상이고 겹칠 수 없다. 이 여행에 참여 중인 로그인 참여자나 비로그인 참여자만 지정할 수 있다. 지출 수정의 예외는 [EXP-07 지출 수정](#exp-07-지출-수정)의 처리 규칙을 따른다. BY_ITEM이면 보낼 수 없고, 보내면 400. |
| `items` | Array&lt;[ExpenseItemInput](#expenseiteminput)&gt; | 조건부 | 없음 | splitType=BY_ITEM이면 필수이며 1개 이상. 보낸 순서대로 표시 순서를 매긴다. EQUAL이면 보낼 수 없고, 보내면 400. |

### ExpenseItemInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `name` | String | 선택 | 없음 | 품목명. 1~50자이며 공백이 아닌 문자를 하나 이상 포함한다. 생략하거나 null이면 이름 없는 품목이다. |
| `amount` | String | 필수 | 없음 | 상품 할인 전 품목 금액. 형식은 [ExpenseInput](#expenseinput)의 `originalAmount`와 같다. |
| `discountAmount` | String | 필수 | 없음 | 이 품목에만 적용하는 상품 할인액. 형식은 `amount`와 같고 `amount`보다 클 수 없다. |
| `tripMemberIds` | Array&lt;String&gt; | 필수 | 없음 | 품목 부담자의 여행 참여자 ID. 1명 이상이고 겹칠 수 없으며, 이 여행에 참여 중인 로그인 참여자나 비로그인 참여자만 지정할 수 있다. 지출 수정의 예외는 [EXP-07 지출 수정](#exp-07-지출-수정)의 처리 규칙을 따른다. |

### EditLock

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `lockToken` | String | 아니요 | 항상 | 현재 수정 작업의 UUID 잠금 토큰. 다음 수정, 연장과 해제 요청의 `X-Lock-Token`으로 보낸다. |
| `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 수정 잠금 획득 시각에 5분을 더하고 초 단위로 내린 만료 시각. 이 시각부터 잠금은 유효하지 않다. |

### LockExpiry

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 기존 만료 시각에 2분을 더한 만료 시각. 이 시각부터 잠금은 유효하지 않다. |

### RecognitionResult

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `name` | String | 예 | 항상 | 지출명으로 쓸 가맹점 이름. 인식하지 못하면 null. 사용자 확인과 수정 전 데이터이며 지출명 최대 길이(50자)를 넘을 수 있다. |
| `paidDate` | String | 예 | 항상 | YYYY-MM-DD. 영수증의 현지 결제 날짜. 인식 실패 시 null. 시각은 반환하지 않음. |
| `currency` | String | 예 | 항상 | KRW, JPY, USD, EUR, CNY 중 하나. 인식하지 못했거나 그 밖의 통화면 null. |
| `originalAmount` | String | 예 | 항상 | 소수점 이하 2자리 할인 전 결제 총액. 세금과 봉사료를 포함한다. 인식 실패 시 null. |
| `discountAmount` | String | 예 | 항상 | 전체 할인액. `splitType`이 `EQUAL`이면 품목의 상품 할인을 더한 값. 할인이 없거나 인식하지 못하면 null. |
| `items` | Array&lt;[RecognizedItem](#recognizeditem)&gt; | 아니요 | 항상 | 인식된 품목. `splitType`이 `EQUAL`이거나 품목을 인식하지 못하면 빈 배열. |

### RecognizedItem

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `name` | String | 예 | 항상 | 인식된 이름. 실패 시 null. |
| `amount` | String | 예 | 항상 | 소수점 이하 2자리 상품 할인 전 금액. 실패 시 null. |
| `discountAmount` | String | 예 | 항상 | 소수점 이하 2자리 상품 할인액. 할인이 없거나 인식하지 못하면 null. |

## 오류 코드

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Billit-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 415 | `COMMON_UNSUPPORTED_MEDIA_TYPE` | 요청이나 Part의 Content-Type을 지원하지 않음 |
| 400 | `EXPENSE_AMOUNT_MISMATCH` | 품목별 지출에서 상품 할인 전 품목 금액의 합계가 할인 전 총액과 다름 |
| 409 | `EXPENSE_EDIT_LOCKED` | 수정 잠금이 아직 유효함 |
| 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 수정 잠금이 유효하지 않음 |
| 409 | `EXPENSE_IDEMPOTENCY_KEY_REUSED` | 지출 생성 고유키를 다른 회원이나 다른 여행에서 이미 사용했거나, 그 키로 만든 지출이 삭제됨 |
| 400 | `EXPENSE_INVALID_SHARES` | 결제자나 부담자로 지정할 수 없는 참여자가 있음 |
| 400 | `EXPENSE_NOT_RECEIPT` | 인식 서비스가 이미지를 영수증이 아니라고 판단함 |
| 413 | `EXPENSE_RECEIPT_TOO_LARGE` | 허용 용량을 초과한 이미지 |
| 415 | `EXPENSE_RECEIPT_UNSUPPORTED` | 지원하지 않는 영수증 이미지 형식 |
| 503 | `EXPENSE_RECOGNITION_UNAVAILABLE` | 외부 인식 기능을 일시적으로 사용할 수 없음 |
| 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |
| 409 | `EXPENSE_TRIP_NOT_CONFIRMED` | 여행 확정 전에 지출을 등록하려 함 |
