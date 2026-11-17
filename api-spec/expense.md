# 지출 API 스펙 초안 (1차 MVP)

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
- [계산 미리보기 API 후보](#계산-미리보기-api-후보)
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

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

이 문서는 API 스펙 검토용 초안이다. API 식별자는 화면 정의서 식별자와 별개다. 구체적인 경로, 요청과 응답 필드 및 오류 코드는 아직 팀 검토를 마치지 않았다. 각 API의 `논의 필요` 인용문 안에 제안 모델과 예시를 구분한다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [지출 목록](#exp-01-지출-목록) | `GET` | `/api/v1/trips/{tripId}/expenses` | 여행 참여자 |
| [지출 상세](#exp-02-지출-상세) | `GET` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 여행 참여자 |
| [지출 생성](#exp-03-지출-생성) | `POST` | `/api/v1/trips/{tripId}/expenses` | 여행 참여자 (여행 확정 후) |
| [수정 잠금 획득](#exp-04-수정-잠금-획득) | `POST` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자 |
| [수정 잠금 연장](#exp-05-수정-잠금-연장) | `PATCH` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 |
| [수정 잠금 해제](#exp-06-수정-잠금-해제) | `DELETE` | `/api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 |
| [지출 수정](#exp-07-지출-수정) | `PUT` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 지출 기록자 또는 여행 관리자, 본인의 수정 잠금 필요 |
| [지출 삭제](#exp-08-지출-삭제) | `DELETE` | `/api/v1/trips/{tripId}/expenses/{expenseId}` | 지출 기록자 또는 여행 관리자 |
| [영수증 인식](#exp-09-영수증-인식) | `POST` | `/api/v1/trips/{tripId}/receipt-recognitions` | 여행 참여자 (여행 확정 후) |

전체 할인액은 개인별 결제 부담 비율로 분배하며 1차 MVP에서는 계산된 개인 부담액의 수기 수정을 제공하지 않는다. 할인액의 반올림과 잔여 금액 처리 기준은 미정이다. 직접 수정한 환율의 자동 저장은 [환율 정책](../policy/exchange-rate.md#지출-환율)을 따른다.

탈퇴자의 결제 목록은 표시 이름과 금액만 제공하고 상세 품목과 개인 리시트는 노출하지 않는다. 구체적인 응답 축약 모델은 추가 설계가 필요하다.

> 논의 필요
>
> 명시적 임시 저장의 서버 저장, 앱 일시 이탈의 로컬 저장 및 앱 내 페이지 이탈 시 미저장은 11월 7일 회의의 논의 방향이다. 필수값과 MVP 범위를 확정한 뒤 임시 저장 API 및 복원 계약을 설계한다. 현재 지출 생성 API를 불완전 입력의 임시 저장으로 사용하지 않는다. OCR 마스킹은 보안 추가 논의 대상으로 유지한다.

## 지출 조회와 저장 API

### EXP-01 지출 목록

`GET /api/v1/trips/{tripId}/expenses`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행의 날짜 및 기간 조건에 맞는 지출 목록을 조회한다. 정산 대상 조회도 이 API로 통합하며 별도의 정산 대상 목록 API를 제공하지 않는다. 정산 준비 시작은 별도 API에서 관리자 권한과 선택한 지출의 현재 상태를 다시 검증한다.

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
> | `fromDate` | String | 선택 | 없음 | YYYY-MM-DD. 현지 결제 날짜 기준. 경계 날짜 포함 여부는 논의 필요. |
> | `toDate` | String | 선택 | 없음 | YYYY-MM-DD. 현지 결제 날짜 기준. 경계 날짜 포함 여부는 논의 필요. |
> | `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
> | `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 Array&lt;[ExpenseSummary](#expensesummary)&gt;이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | Array&lt;[ExpenseSummary](#expensesummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |
>
> 응답 최상위 `page`는 [Page](common.md#page)를 따른다. 페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.
>
> ```json
> {
>   "data": [
>     {
>       "expenseId": "expense-example-a",
>       "name": "식사",
>       "paidDate": "2026-11-05",
>       "currency": "KRW",
>       "amount": "12000.00",
>       "payerTripMemberId": "participant-example-a",
>       "settlementId": null,
>       "isEditLocked": false
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

여행 기간 밖의 지출도 조회한다. 정산 선택에 재사용하면 선택 가능 여부와 사유를 포함하는 안을 검토한다.

> 논의 필요
>
> - 기간 경계, 정산 필터와 선택 가능 여부 및 선택 불가 사유의 응답 필드와 포함 조건, 기본 정렬 기준과 값이 같은 경우의 순서 기준을 결정한다. 정산 응답 필드 후보는 [Candidate](settlement.md#candidate)를 참고한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-02 지출 상세

`GET /api/v1/trips/{tripId}/expenses/{expenseId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출의 결제 정보와 품목, 최종 부담액 및 저장 환율을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "expenseId": "expense-example-a",
>     "name": "식사",
>     "paidDate": "2026-11-05",
>     "currency": "KRW",
>     "amount": "12000.00",
>     "payerTripMemberId": "participant-example-a",
>     "settlementId": null,
>     "isEditLocked": false,
>     "tripId": "trip-example-a",
>     "splitType": "EQUAL",
>     "createdByTripMemberId": "participant-example-a",
>     "paymentMethod": "CARD",
>     "originalAmount": "12000.00",
>     "discountAmount": "0.00",
>     "exchangeRate": "1.00",
>     "exchangeRateSource": "MANUAL",
>     "exchangeRateProvider": null,
>     "exchangeRateBaseDate": null,
>     "amountKrw": "12000.00",
>     "items": [],
>     "shares": [
>       {
>         "tripMemberId": "participant-example-a",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
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

수정 중에는 기존 저장값을 반환한다. 발행된 정산에 포함된 지출도 조회할 수 있다. 탈퇴자 품목과 금액 표시 정책을 적용한다.

> 논의 필요
>
> - 품목 ID와 표시 순서, 탈퇴자의 품목 숨김과 정산에 필요한 금액만 표시하는 범위, 지출명 길이, 환율 제공 기관 및 기준일의 null 규칙을 결정한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-03 지출 생성

`POST /api/v1/trips/{tripId}/expenses`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

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

> 논의 필요
>
> 전달 위치와 헤더 이름을 확정해야 한다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `Idempotency-Key` | String | 필수 | 없음 | 같은 저장 재시도에는 같은 키를 유지. 이름과 범위는 검토 제안이다. |

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [ExpenseInput](#expenseinput).
>
> ```json
> {
>   "name": "식사",
>   "splitType": "EQUAL",
>   "currency": "KRW",
>   "paidDate": "2026-11-05",
>   "paymentMethod": "CARD",
>   "payerTripMemberId": "participant-example-a",
>   "originalAmount": "12000.00",
>   "discountAmount": "0.00",
>   "exchangeRate": "1.00",
>   "exchangeRateSource": "MANUAL",
>   "tripMemberIds": [
>     "participant-example-a",
>     "participant-example-b"
>   ]
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `201 Created`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "expenseId": "expense-example-a",
>     "name": "식사",
>     "paidDate": "2026-11-05",
>     "currency": "KRW",
>     "amount": "12000.00",
>     "payerTripMemberId": "participant-example-a",
>     "settlementId": null,
>     "isEditLocked": false,
>     "tripId": "trip-example-a",
>     "splitType": "EQUAL",
>     "createdByTripMemberId": "participant-example-a",
>     "paymentMethod": "CARD",
>     "originalAmount": "12000.00",
>     "discountAmount": "0.00",
>     "exchangeRate": "1.00",
>     "exchangeRateSource": "MANUAL",
>     "exchangeRateProvider": null,
>     "exchangeRateBaseDate": null,
>     "amountKrw": "12000.00",
>     "items": [],
>     "shares": [
>       {
>         "tripMemberId": "participant-example-a",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
>       }
>     ]
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
> | 409 | `EXPENSE_TRIP_NOT_CONFIRMED` | 여행 확정 전에 지출을 등록하려 함 |
> | 400 | `EXPENSE_INVALID_SHARES` | 부담자가 없거나 같은 여행에서 부담자로 지정할 수 있는 참여자가 아님 |
> | 400 | `EXPENSE_AMOUNT_MISMATCH` | 결제액과 품목 또는 부담액의 합계가 검증 기준과 맞지 않음 |

#### 처리 규칙

결제자는 같은 여행의 회원 참여자여야 하며, 부담자는 같은 여행의 회원 또는 비로그인 참여자여야 한다. 금액 합계를 검증한 뒤 지출과 품목, 부담액을 함께 저장한다. 하나라도 실패하면 모두 저장하지 않으며 저장 고유키도 성공으로 기록하지 않는다.

> 논의 필요
>
> - 서버와 앱 중 어디서 부담액을 계산할지, 저장 요청에 계산 결과를 포함할지 결정한다. 전체 할인 배분, 품목 합계 검증, 자투리 금액 배정과 환산 순서도 정해야 한다. 현재 요청 예시는 서버 계산을 가정한 전체 1/N(EQUAL) 방식의 안이며 최종 스펙이 아니다.
> - 저장 고유키를 보낼 위치와 중복으로 판단할 범위, 보존 기간, 진행 중 중복과 같은 키의 다른 내용, 이미 성공한 요청을 다시 보냈을 때의 응답과 삭제 후 재시도를 결정한다.
> - 금액 상한과 통화 최소 단위, 지출명 길이와 환율 출처를 확정해야 한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

## 수정 잠금과 변경 API

### EXP-04 수정 잠금 획득

`POST /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

해당 여행방에 접근할 수 있는 로그인 회원 중 이 지출을 기록한 회원 또는 여행 관리자만 요청할 수 있다. 결제자라는 이유만으로 수정 권한을 주지는 않는다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출 수정 작업의 독점 권한과 만료 시각을 획득한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `201 Created`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [EditLock](#editlock)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [EditLock](#editlock) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "lockToken": "example-edit-lock-token",
>     "expiresAt": "2026-11-05T03:00:00Z"
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
> | 409 | `EXPENSE_EDIT_LOCKED` | 다른 수정 작업의 잠금이 아직 유효함 |
> | 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |

#### 처리 규칙

지출이 정산 대상으로 잠겨 있거나 다른 수정 작업으로 잠겨 있으면 실패한다. 같은 회원이 다시 요청해도 현재 유효한 수정 잠금을 새 토큰으로 바꾸지 않는다.

> 논의 필요
>
> - 잠금 토큰 발급 방법과 형식, 유지 시간을 결정한다. 잠금 획득 응답을 받지 못했을 때 같은 수정 작업을 다시 확인하는 방법도 정해야 한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-05 수정 잠금 연장

`PATCH /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

해당 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 지출 수정 작업의 유효 기간을 연장한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

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
> | 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 소유자와 토큰이 일치하지 않거나 잠금이 만료됨 |

#### 처리 규칙

요청자가 현재 잠금을 획득한 회원인지, 토큰이 현재 수정 작업과 일치하는지, 만료 전인지 확인한다. 만료된 잠금은 연장할 수 없다. 유효한 연장 요청은 기존 만료 시각에 2분을 더한다.

> 논의 필요
>
> - 토큰 헤더 이름과 전달 위치, 연장 상한과 만료 경계를 결정한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-06 수정 잠금 해제

`DELETE /api/v1/trips/{tripId}/expenses/{expenseId}/edit-lock`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

해당 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 지출 수정 작업을 종료하고 잠금을 해제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

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
> | 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 작업 토큰과 일치하지 않는 해제 요청 |

#### 처리 규칙

토큰이 다른 새 작업은 해제하지 않는다. 이미 해제되거나 만료된 요청의 성공 또는 충돌 응답은 미정이다.

> 논의 필요
>
> - 토큰 전달, 이미 해제되었거나 만료된 요청의 성공 또는 충돌 응답을 결정한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-07 지출 수정

`PUT /api/v1/trips/{tripId}/expenses/{expenseId}`

> 논의 필요
>
> 전체 입력값을 교체하는 PUT 메서드는 확정했다. 최종 경로와 요청 및 응답 필드는 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

해당 여행방에 접근할 수 있는 지출 기록자 또는 여행 관리자만 요청할 수 있다. 현재 수정 잠금을 획득한 사용자여야 하며 같은 수정 작업의 잠금 토큰을 보내야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 잠금으로 지출을 수정하고 확정된 부담액을 저장한다. 앱은 수정 가능한 입력값 전체를 PUT으로 보내며, 서버는 요청 전체를 검증하고 지출 생성과 동일한 계산 기준을 적용한다. 생략한 필드를 기존 값으로 유지하는 PATCH 방식으로 처리하지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

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
> 요청 필드: [ExpenseInput](#expenseinput).
>
> ```json
> {
>   "name": "식사",
>   "splitType": "EQUAL",
>   "currency": "KRW",
>   "paidDate": "2026-11-05",
>   "paymentMethod": "CARD",
>   "payerTripMemberId": "participant-example-a",
>   "originalAmount": "12000.00",
>   "discountAmount": "0.00",
>   "exchangeRate": "1.00",
>   "exchangeRateSource": "MANUAL",
>   "tripMemberIds": [
>     "participant-example-a",
>     "participant-example-b"
>   ]
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [ExpenseDetail](#expensedetail)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [ExpenseDetail](#expensedetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "expenseId": "expense-example-a",
>     "name": "식사",
>     "paidDate": "2026-11-05",
>     "currency": "KRW",
>     "amount": "12000.00",
>     "payerTripMemberId": "participant-example-a",
>     "settlementId": null,
>     "isEditLocked": false,
>     "tripId": "trip-example-a",
>     "splitType": "EQUAL",
>     "createdByTripMemberId": "participant-example-a",
>     "paymentMethod": "CARD",
>     "originalAmount": "12000.00",
>     "discountAmount": "0.00",
>     "exchangeRate": "1.00",
>     "exchangeRateSource": "MANUAL",
>     "exchangeRateProvider": null,
>     "exchangeRateBaseDate": null,
>     "amountKrw": "12000.00",
>     "items": [],
>     "shares": [
>       {
>         "tripMemberId": "participant-example-a",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "amount": "6000.00",
>         "amountKrw": "6000.00"
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
> | 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 수정 잠금이 유효하지 않음 |
> | 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 정산 대상으로 잠겨 있음 |
> | 400 | `EXPENSE_AMOUNT_MISMATCH` | 수정 결과의 금액 합계가 검증 기준과 맞지 않음 |

#### 처리 규칙

수정 잠금이 만료되지 않았고 지출이 정산 대상으로 잠기지 않았는지 다시 확인한다. 지출과 품목, 부담액 변경 및 잠금 해제는 모두 성공하거나 모두 취소된다. 저장에 실패하면 기존 지출과 수정 잠금을 유지한다.

> 논의 필요
>
> - 계산 주체와 세부 계산 규칙은 지출 생성과 함께 확정한다. 잠금 토큰 전달 방법과 수정 저장 응답을 받지 못했을 때의 재요청 처리도 결정해야 한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

### EXP-08 지출 삭제

`DELETE /api/v1/trips/{tripId}/expenses/{expenseId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

#### 인증

해당 여행방에 접근할 수 있는 로그인 회원 중 이 지출을 기록한 회원 또는 여행 관리자만 요청할 수 있다. 결제자라는 이유만으로 수정 권한을 주지는 않는다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

삭제 가능한 지출을 삭제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `expenseId` | String | 필수 | 없음 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |

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
> | 409 | `EXPENSE_EDIT_LOCKED` | 수정 잠금이 아직 유효함 |
> | 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |

#### 처리 규칙

수정 또는 정산으로 잠긴 지출을 삭제하지 않는다. 이미 삭제한 지출을 다시 삭제하거나 조회할 때의 응답은 정해야 한다.

> 논의 필요
>
> - 본인의 수정 잠금을 보유한 상태에서 삭제할 수 있는지, 재삭제 응답을 결정한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

## 영수증 인식 API

### EXP-09 영수증 인식

`POST /api/v1/trips/{tripId}/receipt-recognitions`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

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

> 논의 필요
>
> Content-Type: `multipart/form-data`
>
> | Part | Content-Type | 필수 | 규칙 |
> | --- | --- | --- | --- |
> | `image` | `image/*` | 필수 | 이미지 1개. 형식과 용량 상한은 논의 필요. |
> | `request` | `application/json` | 필수 | splitType: EQUAL 또는 BY_ITEM을 전달하는 검토안. |
>
> JSON Part `request`의 필드 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `splitType` | String | 필수 | 없음 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
>
> ```json
> {
>   "splitType": "EQUAL"
> }
> ```

#### 성공 응답

인식 처리에 성공하면 HTTP `200 OK`로 결과를 반환한다.

> 논의 필요
>
> 아래 응답 모델과 필드는 검토 제안이다.
>
> 공통 응답의 `data`는 [RecognitionResult](#recognitionresult)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [RecognitionResult](#recognitionresult) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "name": "식사",
>     "paidDate": "2026-11-05",
>     "currency": "KRW",
>     "originalAmount": "12000.00",
>     "discountAmount": null,
>     "items": []
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
> | 413 | `EXPENSE_RECEIPT_TOO_LARGE` | 허용 용량을 초과한 이미지 |
> | 415 | `EXPENSE_RECEIPT_UNSUPPORTED` | 지원하지 않는 영수증 이미지 형식 |
> | 503 | `EXPENSE_RECOGNITION_UNAVAILABLE` | 외부 인식 기능을 일시적으로 사용할 수 없음 |

#### 처리 규칙

인식만으로 지출을 저장하지 않는다. 실패 시 재시도와 수기 입력이 가능해야 한다. 작업 접수 후 별도 조회하는 비동기 방식은 사용하지 않는다.

일부 필드를 인식하지 못해도 HTTP `200 OK`로 인식한 부분 결과를 반환한다. 인식하지 못한 단일 필드와 품목의 필드는 null로 반환하며, 인식된 품목이 없으면 items는 빈 배열로 반환한다. 누락값을 0이나 임의의 값으로 채우지 않는다. 사용자는 인식 결과를 검수하고 누락값을 보완한 뒤 별도로 지출 저장을 요청한다.

인식 처리가 완료됐지만 아무 필드도 인식하지 못한 경우에도 HTTP `200 OK`로 빈 결과를 반환한다. 단일 필드는 모두 null, items는 빈 배열로 반환하며 data 자체는 null로 반환하지 않는다. 외부 인식 서비스 장애나 타임아웃으로 처리를 완료하지 못한 경우를 빈 인식 결과로 대체하지 않는다.

빈 인식 결과 예시:

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


> 논의 필요
>
> - 외부 인식 호출과 전체 요청의 타임아웃 기준, 시간 초과 시 오류 응답과 앱의 재시도 처리를 결정한다.
> - 마스킹과 외부 전달, 동의 범위, 텍스트 보존과 원본 이미지 삭제 시점을 결정한다.
>
> 출처: [정책](../policy/expense.md) | [계산 설계](../design/expense-calculation.md) | [잠금](../design/locking.md) | [멱등성](../design/idempotency.md)

## 계산 미리보기 API 후보

> 논의 필요
>
> 서버에서 계산하기로 하면 `POST /api/v1/trips/{tripId}/expense-previews`를 검토한다. 입력 필드는 ExpenseInput을 바탕으로 정한다. 저장 요청에 최종 부담액을 포함할지와 서버가 다시 검증할 항목을 먼저 결정한다. 계산 결과 확인만으로 지출을 저장하지 않는다. POC 후 도입 여부와 결과 고정 방식을 결정하므로 API 목록의 확정 대상에 포함하지 않는다.
> 출처: [지출 계산 설계](../design/expense-calculation.md#논의-필요)

## 공통 모델

모델 이름과 필드 구성은 모두 검토 제안이다. 요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### ExpenseSummary

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |
> | `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
> | `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. UTC 변환 없음. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
> | `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
> | `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
> | `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |

### ExpenseDetail

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expenseId` | String | 아니요 | 항상 | 지출 ID. 경로에 tripId가 있으면 해당 여행에 속한 지출이어야 한다. |
> | `name` | String | 아니요 | 항상 | 지출명. 말줄임은 표시 시 처리한다. |
> | `paidDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. UTC 변환 없음. |
> | `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `amount` | String | 아니요 | 항상 | 할인 후 결제액. 소수점 이하 2자리 문자열로 전달한다. |
> | `payerTripMemberId` | String | 아니요 | 항상 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
> | `settlementId` | String | 예 | 항상 | 이 지출을 정산 대상으로 잠근 정산 ID. 준비 중이거나 발행된 정산을 포함한다. 현재 연결된 정산이 없으면 null. |
> | `isEditLocked` | Boolean | 아니요 | 항상 | 응답 시점의 유효 수정 잠금 여부. |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `splitType` | String | 아니요 | 항상 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
> | `createdByTripMemberId` | String | 아니요 | 항상 | 지출을 등록한 회원의 여행 참여자 ID. 실제 결제자와 다를 수 있다. |
> | `paymentMethod` | String | 아니요 | 항상 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
> | `originalAmount` | String | 아니요 | 항상 | 할인 전 총액. 소수점 이하 2자리 문자열로 전달한다. |
> | `discountAmount` | String | 아니요 | 항상 | 전체 할인액. 소수점 이하 2자리 문자열로 전달한다. |
> | `exchangeRate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액, 0보다 큼. 소수점 이하 2자리 문자열로 전달한다. |
> | `exchangeRateSource` | String | 아니요 | 항상 | `AUTO`, `MANUAL` 중 하나. AUTO는 API에서 가져온 환율, MANUAL은 개인 기록 또는 직접 입력한 환율이다. 제공 기관은 exchangeRateProvider로 구분한다. |
> | `exchangeRateProvider` | String | 예 | 항상 | API 제공 기관. 수동 값의 null 규칙과 명칭은 논의 필요. |
> | `exchangeRateBaseDate` | String | 예 | 항상 | YYYY-MM-DD. 실제 기준일. 직접 입력했을 때의 값은 논의 필요. |
> | `amountKrw` | String | 아니요 | 항상 | 원화 환산 결제액. 소수점 이하 2자리 문자열로 전달한다. |
> | `items` | Array&lt;[ExpenseItem](#expenseitem)&gt; | 아니요 | 항상 | 품목별 저장 내역. EQUAL이면 빈 배열. 탈퇴자 마스킹은 논의 필요. |
> | `shares` | Array&lt;[Share](#share)&gt; | 아니요 | 항상 | 참여자별 최종 부담액. |

### ExpenseItem

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expenseItemId` | String | 아니요 | 항상 | 지출 품목 ID. 해당 지출에 속한 품목을 구분한다. |
> | `name` | String | 예 | 항상 | 품목명. 미입력이면 null. |
> | `amount` | String | 아니요 | 항상 | 상품 할인 전 품목 금액. 소수점 이하 2자리 문자열로 전달한다. |
> | `discountAmount` | String | 아니요 | 항상 | 해당 품목 할인액. 소수점 이하 2자리 문자열로 전달한다. |
> | `sortOrder` | Integer | 아니요 | 항상 | 표시 순서. 시작값과 중복 처리 기준은 논의 필요. |
> | `tripMemberIds` | Array&lt;String&gt; | 아니요 | 항상 | 품목 부담자의 여행 참여 ID. |

### Share

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
> | `amount` | String | 아니요 | 항상 | 결제 통화 기준 최종 부담액. 소수점 이하 2자리 문자열로 전달한다. |
> | `amountKrw` | String | 아니요 | 항상 | 원화 환산 부담액. 소수점 이하 2자리 문자열로 전달한다. |

### ExpenseInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 필수 | 없음 | 필수 지출명. 정책의 입력 무제한과 DDL 길이 제한 충돌은 논의 필요. |
> | `splitType` | String | 필수 | 없음 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
> | `currency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `paidDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |
> | `paymentMethod` | String | 필수 | 없음 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
> | `payerTripMemberId` | String | 필수 | 없음 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
> | `originalAmount` | String | 필수 | 없음 | 할인 전 총액. 소수점 이하 2자리 문자열로 전달한다. |
> | `discountAmount` | String | 선택 | "0.00" | 소수점 이하 2자리 전체 할인액. 0 이상. |
> | `exchangeRate` | String | 필수 | 없음 | 외화 1단위당 KRW, 소수점 이하 2자리, 0보다 큼. KRW 처리 최종 확인 필요. |
> | `exchangeRateSource` | String | 필수 | 없음 | `AUTO`, `MANUAL` 중 하나. |
> | `tripMemberIds` | Array&lt;String&gt; | 조건부 | 없음 | splitType=EQUAL이면 필수이며 1명 이상. BY_ITEM이면 전달하지 않는다. |
> | `items` | Array&lt;[ExpenseItemInput](#expenseiteminput)&gt; | 조건부 | 없음 | splitType=BY_ITEM이면 필수. EQUAL이면 전달하지 않는다. |

### ExpenseItemInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 선택 | 없음 | 품목명. 생략 시 이름 없는 품목으로 처리하는 안이며 null 허용과 최대 길이는 결정해야 한다. |
> | `amount` | String | 필수 | 없음 | 품목 할인 전 금액. 소수점 이하 2자리 문자열로 전달한다. |
> | `discountAmount` | String | 필수 | 없음 | 품목 할인액. 전체 할인과 구분. 소수점 이하 2자리 문자열로 전달한다. |
> | `tripMemberIds` | Array&lt;String&gt; | 필수 | 없음 | 같은 여행의 유효 부담자, 1명 이상. |

### EditLock

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `lockToken` | String | 아니요 | 항상 | 현재 수정 작업의 잠금 토큰. 다음 수정, 연장과 해제 요청에 사용한다. |
> | `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 수정 잠금 획득 시각으로부터 5분 뒤의 만료 시각. |

### LockExpiry

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `expiresAt` | String | 아니요 | 항상 | ISO 8601 UTC. 연장된 작업 만료 시각. |

### RecognitionResult

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 예 | 항상 | 인식하지 못하면 null. 사용자 확인과 수정 전 데이터. |
> | `paidDate` | String | 예 | 항상 | YYYY-MM-DD. 영수증의 현지 결제 날짜. 인식 실패 시 null. 시각은 반환하지 않음. |
> | `currency` | String | 예 | 항상 | KRW, JPY, USD, EUR, CNY 또는 인식 실패 시 null. |
> | `originalAmount` | String | 예 | 항상 | 소수점 이하 2자리 총액, 인식 실패 시 null. |
> | `discountAmount` | String | 예 | 항상 | 전체 할인액. 인식 실패 시 null. |
> | `items` | Array&lt;[RecognizedItem](#recognizeditem)&gt; | 아니요 | 항상 | 인식된 품목, 없으면 빈 배열. |

### RecognizedItem

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 예 | 항상 | 인식된 이름. 실패 시 null. |
> | `amount` | String | 예 | 항상 | 소수점 이하 2자리 상품 할인 전 금액. 실패 시 null. |
> | `discountAmount` | String | 예 | 항상 | 소수점 이하 2자리 상품 할인액. 실패 시 null. |

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
> | 400 | `EXPENSE_AMOUNT_MISMATCH` | 수정 결과의 금액 합계가 검증 기준과 맞지 않음 |
> | 409 | `EXPENSE_EDIT_LOCKED` | 수정 잠금이 아직 유효함 |
> | 409 | `EXPENSE_EDIT_LOCK_INVALID` | 현재 수정 잠금이 유효하지 않음 |
> | 400 | `EXPENSE_INVALID_SHARES` | 부담자가 없거나 같은 여행에서 부담자로 지정할 수 있는 참여자가 아님 |
> | 413 | `EXPENSE_RECEIPT_TOO_LARGE` | 허용 용량을 초과한 이미지 |
> | 415 | `EXPENSE_RECEIPT_UNSUPPORTED` | 지원하지 않는 영수증 이미지 형식 |
> | 503 | `EXPENSE_RECOGNITION_UNAVAILABLE` | 외부 인식 기능을 일시적으로 사용할 수 없음 |
> | 409 | `EXPENSE_SETTLEMENT_LOCKED` | 지출이 준비 중이거나 발행된 정산에 포함돼 있음 |
> | 409 | `EXPENSE_TRIP_NOT_CONFIRMED` | 여행 확정 전에 지출을 등록하려 함 |
