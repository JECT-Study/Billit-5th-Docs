# 정산 API 스펙 (1차 MVP)

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

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/settlement.md) | [설계](../design/settlement.md)

API 식별자는 화면 정의서 식별자와 별개다.

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
| [수령 확인](#settle-13-수령-확인) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/transfers/{transferId}/receipt-confirmation` | 로그인 필요 (그 송금의 수령인) |
| [발행 정산 취소](#settle-14-발행-정산-취소) | `POST` | `/api/v1/trips/{tripId}/settlements/{settlementId}/cancel` | 여행 관리자 |

## 정산 준비 API

### SETTLE-01 정산 대상 조회

정산 대상 조회는 [EXP-01 지출 목록](expense.md#exp-01-지출-목록)의 `GET /api/v1/trips/{tripId}/expenses`로 통합한다. 별도의 settlement-candidates API는 제공하지 않는다. 기존 API 식별자는 통합 이력을 위해 유지하며 후속 API 번호를 변경하지 않는다.

인증은 지출 목록 API의 여행 참여자 조건을 따른다. 정산 준비 시작 권한은 여행 확정 후 관리자에게만 허용하는 기존 조건을 유지한다. 조회 시 선택 가능해도 준비 시작 시 권한, 잠금과 정산 상태를 다시 검증한다.

전체 선택은 현재 필터의 선택 가능한 지출을 대상으로 한다. 앱은 전체 ID를 수집하지 않고 필터 조건과 선택 해제한 지출 ID를 정산 준비 요청에 보낸다. 서버는 같은 여행과 필터에 해당하는 대상에서 제외 ID를 빼고 선택 대상을 구한다. page와 size는 전체 선택 범위를 제한하지 않는다. 전체 선택 시점에 확정한 대상 ID 범위를 유지하며, 이후 새로 등록되거나 필터 조건에 새로 들어온 지출은 자동 포함하지 않는다. 선택 대상에서 제외한 ID는 이 범위에서 제거한다. 선택된 ID의 금액과 품목 같은 내용이 이후 수정됐더라도 변경 확인을 다시 요구하지 않고 정산 준비 시점의 최신 저장값을 사용한다. 결제일이 바뀌어도 선택 시점의 ID 범위를 다시 필터링하지 않는다. 삭제, 수정 잠금과 다른 정산 포함 여부는 준비 시작 시 다시 검증한다.

### SETTLE-02 정산 준비 시작

`POST /api/v1/trips/{tripId}/settlements`

#### 인증

여행 확정 후, 로그인한 그 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

개별 선택은 지출 ID 목록으로, 전체 선택은 필터 조건과 제외 ID로 요청한다. 서버는 선택 대상을 구한 뒤 정산 대상으로 잠그고 발행 전 준비 상태로 등록한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

Content-Type: `application/json`

요청 필드: [PreparationInput](#preparationinput).

```json
{
  "selectionMode": "EXPLICIT",
  "expenseIds": [
    "expense-example-a"
  ]
}
```

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [Preparation](#preparation)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [Preparation](#preparation) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "status": "DRAFT",
    "expenseIds": [
      "expense-example-a"
    ],
    "settlementMethod": null,
    "lockToken": "example-preparation-lock-token",
    "expiresAt": "2026-11-05T03:00:00Z",
    "createdAt": null
  }
}
```

생성한 대상의 조회 URL을 `Location`에 제공한다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `SETTLEMENT_EXPENSE_UNAVAILABLE` | 선택한 지출 중 한 건 이상이 수정 중이거나 다른 정산에 포함돼 있음 |

#### 처리 규칙

같은 여행의 미정산 지출을 한 건도 선택하지 않으면 서버는 요청을 거절한다.
개별 선택의 `expenseIds`에 같은 ID가 여러 번 들어오면 서버는 중복을 제거한 ID 집합으로 검증, 잠금과 계산을 진행한다.
서버는 중복만을 이유로 요청을 거절하지 않으며 같은 지출을 여러 번 합산하지 않는다.
선택한 지출 중 한 건이라도 수정 중이거나 다른 정산에 포함돼 있으면 서버는 전체 요청을 거절한다.
요청을 거절하면 서버는 준비 정산을 만들지 않고 일부 지출만 잠그지 않는다.

### SETTLE-03 정산 준비 조회

`GET /api/v1/trips/{tripId}/settlements/{settlementId}/preparation`

#### 인증

로그인한 그 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 정산 준비의 선택 지출과 방식, 유효 기간을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [PreparationView](#preparationview)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [PreparationView](#preparationview) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "status": "DRAFT",
    "expenseIds": [
      "expense-example-a"
    ],
    "settlementMethod": null,
    "expiresAt": "2026-11-05T03:00:00Z",
    "createdAt": null
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
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 만료 또는 취소되었거나 발행 전 준비 상태가 아님 |

#### 처리 규칙

서버는 취소되거나 만료되지 않은 준비 정산만 반환한다.
조회만으로 서버는 다른 화면에서 시작한 정산 준비를 넘겨주거나 잠금 토큰을 새로 발급하지 않는다.

### SETTLE-04 준비 잠금 연장

`PATCH /api/v1/trips/{tripId}/settlements/{settlementId}/preparation-lock`

#### 인증

로그인한 그 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 정산 준비의 유효 기간을 연장한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. |

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
    "expiresAt": "2026-11-05T03:00:00Z"
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
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 정산 준비가 취소되거나 만료됐거나 잠금 토큰이 현재 정산 준비과 다름 |

#### 처리 규칙

서버는 준비가 취소되거나 발행되지 않았는지, 요청자가 준비를 시작한 회원인지, 토큰이 같은 정산 준비인지, 만료 전인지 확인한다.
조건을 모두 만족하면 서버는 선택한 지출 전체의 준비 시간을 연장한다.

### SETTLE-05 정산 준비 취소

`DELETE /api/v1/trips/{tripId}/settlements/{settlementId}/preparation`

#### 인증

로그인한 그 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행 전 정산 준비를 취소하고 선택 지출의 잠금을 해제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. |

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
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 요청자가 이 정산 준비를 시작한 회원이 아니거나 이미 취소 또는 발행된 준비임 |

#### 처리 규칙

서버는 요청자가 그 정산 준비를 시작했는지 확인한다.
서버는 준비 취소와 선택 지출의 잠금 해제를 한 번에 반영하며, 하나라도 실패하면 모두 취소한다.
이미 발행한 정산이면 서버는 이 API로 취소하지 않는다.

## 계산과 발행 API

### SETTLE-06 정산 미리보기

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/previews`

#### 인증

로그인한 그 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

정산 방식에 따른 부담액과 잔액, 송금 내역를 미리 계산한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. |

##### Body

Content-Type: `application/json`

요청 필드: [MethodInput](#methodinput).

```json
{
  "settlementMethod": "KRW_UNIFIED"
}
```

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [SettlementPreview](#settlementpreview)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [SettlementPreview](#settlementpreview) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "settlementMethod": "KRW_UNIFIED",
    "expenseCount": 1,
    "totals": [
      {
        "currency": "KRW",
        "amount": "12000.00"
      }
    ],
    "balances": [
      {
        "tripMemberId": "participant-example-a",
        "currency": "KRW",
        "paidAmount": "12000.00",
        "shareAmount": "6000.00",
        "balance": "6000.00"
      },
      {
        "tripMemberId": "participant-example-b",
        "currency": "KRW",
        "paidAmount": "0.00",
        "shareAmount": "6000.00",
        "balance": "-6000.00"
      }
    ],
    "transfers": [
      {
        "senderTripMemberId": "participant-example-b",
        "receiverTripMemberId": "participant-example-a",
        "currency": "KRW",
        "amount": "6000.00"
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
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 정산 준비가 취소되거나 만료되어 미리볼 수 없음 |

#### 처리 규칙

서버는 지출에 저장된 환율로 계산하며 새 환율을 적용하지 않는다.
미리보기만으로 서버는 정산서를 발행하거나 송금 내역을 확정 저장하지 않는다.
서버는 계산 결과와 그 결과에 대응하는 대상 지출과 정산 방식을 발행 전 미리보기로 보존한다.
발행 요청은 사용자가 확인한 미리보기를 식별한다.
발행할 때 서버는 보존한 결과와 현재 준비 조건이 같은지 검증한다.

### SETTLE-07 정산 확정과 발행

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/confirm`

#### 인증

로그인한 그 여행의 관리자 중 현재 정산 준비를 시작한 사용자만 요청할 수 있다. 잠금 토큰으로 같은 정산 준비인지 확인한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

확인한 정산 결과를 확정하고 참여자별 정산서를 발행한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `X-Lock-Token` | String | 필수 | 없음 | 현재 수정 또는 정산 준비의 잠금 토큰. |

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [SettlementDetail](#settlementdetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [SettlementDetail](#settlementdetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "tripId": "trip-example-a",
    "status": "IN_PROGRESS",
    "settlementMethod": "KRW_UNIFIED",
    "expenseCount": 1,
    "createdAt": "2026-11-05T03:00:00Z",
    "totals": [
      {
        "currency": "KRW",
        "amount": "12000.00"
      }
    ],
    "tripMemberIds": [
      "participant-example-a",
      "participant-example-b"
    ],
    "transfers": [
      {
        "transferId": "transfer-example-a",
        "senderTripMemberId": "participant-example-b",
        "receiverTripMemberId": "participant-example-a",
        "currency": "KRW",
        "amount": "6000.00",
        "receivedAt": null,
        "isReceivable": true
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
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 준비가 만료되었거나 취소 또는 발행되어 확정할 수 없음 |

#### 처리 규칙

서버는 준비가 취소되거나 만료되지 않았는지, 요청자와 토큰이 현재 정산 준비와 같은지, 선택 지출이 여전히 이 정산에 잠겨 있는지 다시 확인한다.
서버는 포함 지출과 송금 내역을 한 번에 저장하며 일부만 저장하지 않는다.
송금 내역이 없으면 서버는 `IN_PROGRESS`를 거치지 않고 정산을 `COMPLETED`로 저장하며, 응답의 `transfers`는 빈 배열이다([확정과 발행](../policy/settlement.md#확정과-발행)과 같은 규칙).
같은 정산을 다시 확정해도 서버는 정산서나 송금 내역을 중복 생성하지 않는다.
사용자가 확인한 보존 미리보기와 대상 지출, 정산 방식과 결과가 같을 때만 서버는 정산을 발행한다.
결과가 다르면 서버는 다른 결과로 자동 발행하지 않는다.
결과가 달라 발행하지 않으면 앱은 새 미리보기를 확인하게 한다.

## 정산서 조회 API

### SETTLE-08 여행 정산 목록

`GET /api/v1/trips/{tripId}/settlements`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행의 발행된 정산을 상태별로 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `status` | String | 선택 | 전체 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[SettlementSummary](#settlementsummary)&gt;이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[SettlementSummary](#settlementsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.

```json
{
  "data": [
    {
      "settlementId": "settlement-example-a",
      "tripId": "trip-example-a",
      "status": "IN_PROGRESS",
      "settlementMethod": "KRW_UNIFIED",
      "expenseCount": 1,
      "createdAt": "2026-11-05T03:00:00Z",
      "totals": [
        {
          "currency": "KRW",
          "amount": "12000.00"
        }
      ]
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

서버는 발행 전 준비 상태인 정산을 이 목록에서 뺀다.
발행 후 취소한 정산은 정산취소 필터로 조회한다.

### SETTLE-09 발행 정산 상세

`GET /api/v1/trips/{tripId}/settlements/{settlementId}`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행 정산의 요약과 송금 상태를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [SettlementDetail](#settlementdetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [SettlementDetail](#settlementdetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "tripId": "trip-example-a",
    "status": "IN_PROGRESS",
    "settlementMethod": "KRW_UNIFIED",
    "expenseCount": 1,
    "createdAt": "2026-11-05T03:00:00Z",
    "totals": [
      {
        "currency": "KRW",
        "amount": "12000.00"
      }
    ],
    "tripMemberIds": [
      "participant-example-a",
      "participant-example-b"
    ],
    "transfers": [
      {
        "transferId": "transfer-example-a",
        "senderTripMemberId": "participant-example-b",
        "receiverTripMemberId": "participant-example-a",
        "currency": "KRW",
        "amount": "6000.00",
        "receivedAt": null,
        "isReceivable": true
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
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

탈퇴자의 표시와 기록 삭제는 [회원 탈퇴 정책](../policy/user.md#공동-기록과-미완료-정산)을 따른다.
서버는 탈퇴 전에 확인된 수령 상태를 유지한다.
송금인과 수령인이 모두 탈퇴한 관계는 서버가 응답에서 뺀다.
기록을 삭제했으면 서버는 그 항목과 총 송금액을 응답에서 숨긴다.
기록을 삭제했으면 앱은 일부 기록이 삭제됐다고 안내한다.
조회해도 서버는 수령 확인 상태를 바꾸지 않는다.

### SETTLE-10 참여자별 정산서

`GET /api/v1/trips/{tripId}/settlements/{settlementId}/statements/{tripMemberId}`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지정한 여행 참여자의 정산서와 결제 상세 내역를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `tripMemberId` | String | 필수 | 없음 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [Statement](#statement)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [Statement](#statement) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "tripId": "trip-example-a",
    "status": "IN_PROGRESS",
    "settlementMethod": "KRW_UNIFIED",
    "expenseCount": 1,
    "createdAt": "2026-11-05T03:00:00Z",
    "totals": [
      {
        "currency": "KRW",
        "amount": "12000.00"
      }
    ],
    "tripMemberId": "participant-example-a",
    "balances": [
      {
        "tripMemberId": "participant-example-a",
        "currency": "KRW",
        "paidAmount": "12000.00",
        "shareAmount": "6000.00",
        "balance": "6000.00"
      }
    ],
    "transfers": [
      {
        "transferId": "transfer-example-a",
        "senderTripMemberId": "participant-example-b",
        "receiverTripMemberId": "participant-example-a",
        "currency": "KRW",
        "amount": "6000.00",
        "receivedAt": null,
        "isReceivable": true
      }
    ],
    "expenses": [
      {
        "expenseId": "expense-example-a",
        "paidDate": "2026-11-05",
        "paymentMethod": "CARD",
        "currency": "KRW",
        "exchangeRate": "1.00",
        "amount": "12000.00",
        "shareAmount": "6000.00"
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
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

회원은 비로그인 참여자의 정산서도 조회할 수 있다.
서버는 결제 상세를 결제 날짜 오름차순으로 반환하며 저장 환율을 쓴다.

### SETTLE-11 내 정산 목록

`GET /api/v1/me/settlement-statements`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

내 공간에 표시할 본인의 정산서 목록을 조회한다.

#### 요청

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `status` | String | 선택 | 전체 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[StatementSummary](#statementsummary)&gt;이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[StatementSummary](#statementsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.

```json
{
  "data": [
    {
      "settlementId": "settlement-example-a",
      "tripId": "trip-example-a",
      "status": "IN_PROGRESS",
      "settlementMethod": "KRW_UNIFIED",
      "expenseCount": 1,
      "createdAt": "2026-11-05T03:00:00Z",
      "totals": [
        {
          "currency": "KRW",
          "amount": "12000.00"
        }
      ],
      "tripMemberId": "participant-example-a"
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

#### 처리 규칙

요청자가 관리자여도 서버는 다른 사람의 정산서를 내 목록에 넣지 않는다.
여행 접근이 차단된 뒤에도 서버는 정책상 유지하는 내역을 반환한다.
생성자가 탈퇴한 뒤에도 서버는 본인 정산서를 내 정산 내역에 반환한다.
여행을 나간 회원에게도 서버는 자신이 포함된 정산이 발행되면 정산 시점과 관계없이 반환한다([나가기와 기록 유지](../policy/trip.md#나가기와-기록-유지)와 같은 규칙).

### SETTLE-12 내 정산 상세

`GET /api/v1/me/settlement-statements/{settlementId}`

#### 인증

로그인한 회원은 본인의 정산서만 조회할 수 있다. 여행방에 계속 접근할 수 있는지와 정산서 조회 가능 여부는 별도로 판단한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

내 공간에서 본인의 정산서와 허용된 계산에 사용한 지출 내역를 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [Statement](#statement)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [Statement](#statement) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "tripId": "trip-example-a",
    "status": "IN_PROGRESS",
    "settlementMethod": "KRW_UNIFIED",
    "expenseCount": 1,
    "createdAt": "2026-11-05T03:00:00Z",
    "totals": [
      {
        "currency": "KRW",
        "amount": "12000.00"
      }
    ],
    "tripMemberId": "participant-example-a",
    "balances": [
      {
        "tripMemberId": "participant-example-a",
        "currency": "KRW",
        "paidAmount": "12000.00",
        "shareAmount": "6000.00",
        "balance": "6000.00"
      }
    ],
    "transfers": [
      {
        "transferId": "transfer-example-a",
        "senderTripMemberId": "participant-example-b",
        "receiverTripMemberId": "participant-example-a",
        "currency": "KRW",
        "amount": "6000.00",
        "receivedAt": null,
        "isReceivable": true
      }
    ],
    "expenses": [
      {
        "expenseId": "expense-example-a",
        "paidDate": "2026-11-05",
        "paymentMethod": "CARD",
        "currency": "KRW",
        "exchangeRate": "1.00",
        "amount": "12000.00",
        "shareAmount": "6000.00"
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
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

서버는 여행 상세 접근 권한과 별개로 본인 정산서의 보존과 접근 권한을 검증한다.
생성자가 탈퇴한 뒤에도 회원은 기존 여행명, 필요한 송금 관계와 금액을 조회하고 본인이 수령인인 관계의 수령을 확인할 수 있다.
서버는 탈퇴자 기록의 삭제 정책을 똑같이 적용한다.
여행을 나간 회원도 자신이 포함된 정산이 발행되면 정산 시점과 관계없이 조회할 수 있다([나가기와 기록 유지](../policy/trip.md#나가기와-기록-유지)와 같은 규칙).

## 수령 확인과 취소 API

### SETTLE-13 수령 확인

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/transfers/{transferId}/receipt-confirmation`

#### 인증

로그인한 회원 중 그 송금에서 돈을 받는 사람만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

실제 수령인이 송금 내역 한 건의 수령을 확인한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `transferId` | String | 필수 | 없음 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ReceiptConfirmation](#receiptconfirmation)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ReceiptConfirmation](#receiptconfirmation) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "transferId": "transfer-example-a",
    "receivedAt": "2026-11-05T03:00:00Z",
    "settlementId": "settlement-example-a",
    "status": "COMPLETED"
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
| 409 | `SETTLEMENT_RECEIPT_NOT_ALLOWED` | 현재 정산 상태 또는 탈퇴하거나 여행을 나간 수령인의 관계로 수령 확인할 수 없음 |

#### 처리 규칙

서버는 정산이 `IN_PROGRESS`인지, 요청자가 그 송금의 수령인인지, 아직 수령을 확인하지 않았는지 확인한다.
여행을 나간 수령인이면 서버는 수령 확인을 거절한다([수령 확인과 완료](../policy/settlement.md#수령-확인과-완료)와 같은 규칙).
서버는 수령 확인을 한 번만 반영한다.
수령 확인 불가 관계를 뺀 모든 송금의 수령이 확인되면 서버는 정산을 `COMPLETED`로 바꾼다.
정산을 완료하면 서버는 탈퇴자의 송금 관계와 금액을 즉시 삭제한다.
최대 보존 기간이 지나 일부 기록이 삭제돼도 회원은 남은 관계의 수령을 계속 확인할 수 있다.

### SETTLE-14 발행 정산 취소

`POST /api/v1/trips/{tripId}/settlements/{settlementId}/cancel`

#### 인증

로그인한 그 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

발행된 정산을 취소하고 포함 지출을 다시 사용할 수 있게 한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `settlementId` | String | 필수 | 없음 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [SettlementCancellation](#settlementcancellation)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [SettlementCancellation](#settlementcancellation) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "settlementId": "settlement-example-a",
    "status": "CANCELED"
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
| 409 | `SETTLEMENT_CANCELLATION_NOT_ALLOWED` | 이미 수령 확인이 있거나 취소 가능한 정산 상태가 아님 |

#### 처리 규칙

정산이 `IN_PROGRESS`가 아니거나 수령 확인이 한 건이라도 있으면 서버는 취소를 거절한다.
서버는 같은 정산의 수령 확인과 정산 취소를 둘 다 성공시키지 않는다.
정산을 취소하면 서버는 포함 지출의 잠금을 해제한다.
취소한 정산의 지출을 다시 정산하면 서버는 새 정산 ID를 쓴다.

## 공통 모델

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### Candidate

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |
| `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
| `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 결제 날짜. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
| `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
| `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
| `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |
| `isSelectable` | Boolean | 아니요 | 항상 | 조회 시점의 선택 가능 여부. |
| `unavailableReason` | String | 예 | 항상 | 선택 가능하면 null. |

### PreparationInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `selectionMode` | String | 필수 | 없음 | EXPLICIT은 ID 개별 선택, FILTER는 필터 전체 선택. |
| `expenseIds` | Array&lt;String&gt; | 조건부 | 없음 | EXPLICIT일 때 같은 여행의 지출 ID 1개 이상. FILTER에서는 보내지 않음 |
| `filter` | Object | 조건부 | 없음 | FILTER일 때 지출 목록과 동일한 필터 조건. 페이지 조건 제외 |
| `excludedExpenseIds` | Array&lt;String&gt; | 조건부 | 없음 | FILTER일 때 선택 해제한 지출 ID. 제외할 것이 없으면 빈 배열. EXPLICIT에서는 보내지 않음 |

FILTER 방식도 준비 요청 시 필터를 새로 실행해 대상을 확장하지 않는다.

두 선택 방식을 혼합하지 않는다.
개별 선택의 중복 ID는 제거해 처리한다.

### Preparation

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `status` | String | 아니요 | 항상 | 발행 전 준비 상태인 `DRAFT`를 반환한다. |
| `expenseIds` | Array&lt;String&gt; | 아니요 | 항상 | 현재 준비 중인 정산의 대상으로 잠근 지출 ID 목록. |
| `settlementMethod` | String | 예 | 항상 | KRW_UNIFIED는 원화 통합, BY_CURRENCY는 결제 통화별 정산이다. 아직 선택하지 않았으면 null. |
| `lockToken` | String | 아니요 | 항상 | 현재 정산 준비의 잠금 토큰. 같은 정산 준비의 연장과 확정, 취소 요청에 사용한다. |
| `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 준비 만료 시각. |
| `createdAt` | String | 예 | 항상 | ISO 8601 UTC. 발행 전이므로 null. |

### PreparationView

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `status` | String | 아니요 | 항상 | 발행 전 준비 상태인 `DRAFT`를 반환한다. |
| `expenseIds` | Array&lt;String&gt; | 아니요 | 항상 | 현재 준비 중인 정산의 대상으로 잠근 지출 ID 목록. |
| `settlementMethod` | String | 예 | 항상 | KRW_UNIFIED는 원화 통합, BY_CURRENCY는 결제 통화별 정산이다. 아직 선택하지 않았으면 null. |
| `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 준비 만료 시각. |
| `createdAt` | String | 예 | 항상 | ISO 8601 UTC. 발행 전이므로 null. |

### LockExpiry

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 연장된 작업 만료 시각. |

### MethodInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `settlementMethod` | String | 필수 | 없음 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |

### SettlementPreview

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
| `expenseCount` | Integer | 아니요 | 항상 | 선택 지출 건수, 1 이상. |
| `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 정산 통화별 합계. |
| `balances` | Array&lt;[Balance](#balance)&gt; | 아니요 | 항상 | 참여자와 통화별 잔액. |
| `transfers` | Array&lt;[TransferPreview](#transferpreview)&gt; | 아니요 | 항상 | 미리보기 송금 내역. 0원 관계는 제외. |

### CurrencyTotal

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 그 통화의 총액. 소수점 이하 2자리 문자열로 전달한다. |

### Balance

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `paidAmount` | String | 아니요 | 항상 | 그 참여자의 결제 합계. 소수점 이하 2자리 문자열로 전달한다. |
| `shareAmount` | String | 아니요 | 항상 | 그 참여자의 부담 합계. 소수점 이하 2자리 문자열로 전달한다. |
| `balance` | String | 아니요 | 항상 | 결제액에서 부담액을 뺀 정산 잔액. 받을 돈은 양수, 보낼 돈은 음수이며 계좌 잔고를 뜻하지 않는다. 소수점 이하 2자리 문자열로 전달한다. |

### TransferPreview

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `senderTripMemberId` | String | 아니요 | 항상 | 돈을 보내야 하는 사람의 여행 참여자 ID. |
| `receiverTripMemberId` | String | 아니요 | 항상 | 돈을 받아야 하는 사람의 여행 참여자 ID. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 0보다 큰 송금액. 소수점 이하 2자리 문자열로 전달한다. |

### SettlementDetail

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
| `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
| `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
| `tripMemberIds` | Array&lt;String&gt; | 아니요 | 항상 | 정산서 조회 대상 참여자. |
| `transfers` | Array&lt;[Transfer](#transfer)&gt; | 아니요 | 항상 | 발행된 송금 내역. 0원 관계 없음. |

### Transfer

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `transferId` | String | 아니요 | 항상 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |
| `senderTripMemberId` | String | 아니요 | 항상 | 돈을 보내야 하는 사람의 여행 참여자 ID. |
| `receiverTripMemberId` | String | 아니요 | 항상 | 돈을 받아야 하는 사람의 여행 참여자 ID. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `amount` | String | 아니요 | 항상 | 0보다 큰 송금액. 소수점 이하 2자리 문자열로 전달한다. |
| `receivedAt` | String | 예 | 항상 | ISO 8601 UTC. 미확인이면 null. |
| `isReceivable` | Boolean | 아니요 | 항상 | 이 송금 내역의 수령 확인이 가능한지 표시한다. 요청자가 확인 버튼을 누를 권한이 있다는 뜻은 아니다. 수령인이 탈퇴했거나 여행을 나갔으면 false. |

### SettlementSummary

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
| `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
| `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |

### Statement

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
| `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
| `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
| `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
| `balances` | Array&lt;[Balance](#balance)&gt; | 아니요 | 항상 | 그 참여자의 결제액과 부담액, 잔액. |
| `transfers` | Array&lt;[Transfer](#transfer)&gt; | 아니요 | 항상 | 그 참여자가 보내거나 받는 관계. |
| `expenses` | Array&lt;[StatementExpense](#statementexpense)&gt; | 아니요 | 항상 | 결제 날짜 오름차순, 동률은 지출 ID 순. |

### StatementExpense

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 그 여행에 속한 지출이어야 한다. |
| `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 결제 날짜, 오름차순. |
| `paymentMethod` | String | 아니요 | 항상 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `exchangeRate` | String | 아니요 | 항상 | 저장 당시 환율. 소수점 이하 2자리 문자열로 전달한다. |
| `amount` | String | 아니요 | 항상 | 지출 총액. 소수점 이하 2자리 문자열로 전달한다. |
| `shareAmount` | String | 아니요 | 항상 | 그 정산서 참여자의 부담액. 소수점 이하 2자리 문자열로 전달한다. |

### StatementSummary

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `status` | String | 아니요 | 항상 | `IN_PROGRESS`, `COMPLETED`, 발행 후 취소 상태인 `CANCELED`를 반환한다. |
| `settlementMethod` | String | 아니요 | 항상 | `KRW_UNIFIED`는 원화 통합 정산, `BY_CURRENCY`는 결제 통화별 정산이다. |
| `expenseCount` | Integer | 아니요 | 항상 | 포함 지출 건수. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 정산 확정과 발행 시각. |
| `totals` | Array&lt;[CurrencyTotal](#currencytotal)&gt; | 아니요 | 항상 | 통화별 총액. |
| `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

### ReceiptConfirmation

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `transferId` | String | 아니요 | 항상 | 정산에 포함된 개별 송금 내역 ID. 송금인과 수령인, 통화와 금액을 구분한다. |
| `receivedAt` | String | 아니요 | 항상 | ISO 8601 UTC. 수령 확인 시각. |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `status` | String | 아니요 | 항상 | 아직 확인할 송금이 남으면 `IN_PROGRESS`, 완료 조건을 충족하면 `COMPLETED`다. |

### SettlementCancellation

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `settlementId` | String | 아니요 | 항상 | 정산 ID. 경로에 tripId가 있으면 그 여행의 정산이어야 한다. |
| `status` | String | 아니요 | 항상 | 발행 후 취소 상태인 `CANCELED`를 반환한다. |

## 오류 코드

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Billit-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 409 | `SETTLEMENT_CANCELLATION_NOT_ALLOWED` | 이미 수령 확인이 있거나 취소 가능한 정산 상태가 아님 |
| 409 | `SETTLEMENT_EXPENSE_UNAVAILABLE` | 선택한 지출 중 한 건 이상이 수정 중이거나 다른 정산에 포함돼 있음 |
| 409 | `SETTLEMENT_PREPARATION_INVALID` | 준비가 만료되었거나 취소 또는 발행되어 확정할 수 없음 |
| 409 | `SETTLEMENT_RECEIPT_NOT_ALLOWED` | 현재 정산 상태 또는 탈퇴하거나 여행을 나간 수령인의 관계로 수령 확인할 수 없음 |
