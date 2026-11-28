# 환율 API 스펙 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [개인 환율 API](#개인-환율-api)
  - [FX-01 내 환율 목록](#fx-01-내-환율-목록)
  - [FX-02 개인 환율 등록](#fx-02-개인-환율-등록)
  - [FX-04 개인 환율 삭제](#fx-04-개인-환율-삭제)
- [환율 조회와 선택 API](#환율-조회와-선택-api)
  - [FX-03 결제일 기준 환율 조회](#fx-03-결제일-기준-환율-조회)
- [공통 모델](#공통-모델)
  - [PersonalRate](#personalrate)
  - [PersonalRateInput](#personalrateinput)
  - [ReferenceRate](#referencerate)
- [오류 코드](#오류-코드)

## 문서 범위

개인 환전 기록과 외부 API 환율을 구분한다. 외부 조회값 자체는 개인 기록으로 자동 저장하지 않는다. 직접 수정한 환율은 지출에 적용한다. 본인 현금 또는 트래블 카드 지출의 수동 수정값은 개인 기록으로 자동 저장한다. 개인 환율 기록은 조회, 등록과 삭제를 제공하고 수정은 제공하지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/exchange-rate.md)

API 식별자는 화면 정의서 식별자와 별개다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [내 환율 목록](#fx-01-내-환율-목록) | `GET` | `/api/v1/trips/{tripId}/exchange-rates` | 여행 참여자 (본인 기록) |
| [개인 환율 등록](#fx-02-개인-환율-등록) | `POST` | `/api/v1/trips/{tripId}/exchange-rates` | 여행 참여자 |
| [개인 환율 삭제](#fx-04-개인-환율-삭제) | `DELETE` | `/api/v1/trips/{tripId}/exchange-rates/{exchangeRateId}` | 여행 참여자 (본인 기록) |
| [결제일 기준 환율 조회](#fx-03-결제일-기준-환율-조회) | `GET` | `/api/v1/reference-exchange-rates` | 로그인한 가입 완료 회원 |

## 개인 환율 API

### FX-01 내 환율 목록

`GET /api/v1/trips/{tripId}/exchange-rates`

#### 인증

로그인한 여행 참여자는 본인이 등록한 환율만 조회할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 여행에 저장한 본인의 환전 기록을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[PersonalRate](#personalrate)&gt;이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[PersonalRate](#personalrate)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.

```json
{
  "data": [
    {
      "exchangeRateId": "rate-example-a",
      "currency": "JPY",
      "rate": "9.12",
      "exchangeMethod": "CASH",
      "exchangeDate": "2026-11-05",
      "createdAt": "2026-11-05T03:00:00Z"
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

서버는 현재 로그인한 회원이 등록한 기록만 반환한다.
회원은 다른 회원의 환율을 조회할 수 없다.
최근 기록은 등록 시각을 기준으로 한다([지출 환율](../policy/exchange-rate.md#지출-환율)).

### FX-02 개인 환율 등록

`POST /api/v1/trips/{tripId}/exchange-rates`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인의 환전 환율과 환전 수단, 날짜를 기록한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

Content-Type: `application/json`

요청 필드: [PersonalRateInput](#personalrateinput).

```json
{
  "currency": "JPY",
  "rate": "9.12",
  "exchangeMethod": "CASH",
  "exchangeDate": "2026-11-05"
}
```

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [PersonalRate](#personalrate)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [PersonalRate](#personalrate) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "exchangeRateId": "rate-example-a",
    "currency": "JPY",
    "rate": "9.12",
    "exchangeMethod": "CASH",
    "exchangeDate": "2026-11-05",
    "createdAt": "2026-11-05T03:00:00Z"
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
| 409 | `EXCHANGE_RATE_DUPLICATED` | 같은 여행에 본인이 통화, 환전 날짜, 환전 수단과 환율이 모두 같은 기록을 이미 등록함 |

#### 처리 규칙

서버는 통화가 `JPY`, `USD`, `EUR`, `CNY` 중 하나인지와 환율이 0보다 크고 정수부 10자리 이하인 소수점 이하 2자리 값인지 검증한다.
서버는 기록자 ID를 직접 정한다.
같은 여행에서 본인이 통화, 일자, 수단과 환율이 모두 같은 기록을 다시 저장하면 서버는 `409 EXCHANGE_RATE_DUPLICATED`로 거절한다.
날짜가 다르면 서버는 별도 기록으로 저장한다.

### FX-04 개인 환율 삭제

`DELETE /api/v1/trips/{tripId}/exchange-rates/{exchangeRateId}`

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원이 본인이 등록한 기록만 삭제할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인이 등록한 개인 환율 기록 하나를 삭제한다.
서버는 삭제한 기록을 [FX-01 내 환율 목록](#fx-01-내-환율-목록)에서 반환하지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `exchangeRateId` | String | 필수 | 없음 | 삭제할 개인 환율 기록 ID. 본인이 경로의 여행에서 등록한 기록이어야 한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환한다.
응답 Body는 없다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `tripId`나 `exchangeRateId`가 숫자 ID 형식이 아님 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 요청한 회원이 이 여행에 참여 중이 아니거나 여행방에 접근할 수 없음 |
| 404 | `COMMON_NOT_FOUND` | 여행이 없거나, 기록이 없거나 이미 삭제됐거나, 본인이 이 여행에서 등록한 기록이 아님 |

#### 처리 규칙

서버는 삭제한 기록을 복구하지 않는다. 근거: [Issue #78 논의 결과](https://github.com/JECT-Study/Billit-5th-Docs/issues/78)
서버는 삭제로 기존 지출과 정산의 환율을 바꾸지 않는다.
다른 회원이 등록한 기록의 ID로 요청하면 서버는 기록의 존재를 알리지 않고 `404 COMMON_NOT_FOUND`로 거절한다.
저장 환율은 등록자 본인에게만 보여 주기 때문이다([환율 정책](../policy/exchange-rate.md#개인-환율-기록)).

이미 삭제한 기록의 삭제를 다시 요청하면 서버는 `404 COMMON_NOT_FOUND`를 반환한다.
삭제 응답을 받지 못해 다시 보낸 요청도 같다.

## 환율 조회와 선택 API

### FX-03 결제일 기준 환율 조회

`GET /api/v1/reference-exchange-rates`

#### 인증

로그인한 가입 완료 회원만 요청할 수 있다. 인증 없이 공개하지 않으며 가입 전용 토큰으로는 호출할 수 없다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출 결제일의 참고 환율과 실제 제공 기관, 기준일을 조회한다.

#### 요청

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `currency` | String | 필수 | 없음 | `JPY`, `USD`, `EUR`, `CNY` 중 하나. `KRW`는 거절한다. |
| `paidDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜를 조회 기준으로 사용. 미래 날짜는 거절. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ReferenceRate](#referencerate)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ReferenceRate](#referencerate) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "currency": "JPY",
    "rate": "9.12",
    "provider": "ECB",
    "baseDate": "2026-11-05"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음. `KRW`와 미래 날짜 요청 포함 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `EXCHANGE_RATE_NOT_PUBLISHED` | 결제일과 그 이전 7일 안에 고시된 환율이 없음 |
| 503 | `EXCHANGE_RATE_PROVIDER_UNAVAILABLE` | 외부 환율을 일시적으로 조회할 수 없음 |

#### 처리 규칙

요청 날짜에 고시 환율이 없으면 서버는 그 날짜 이전 7일 안에 고시된 값 중 가장 최근 환율을 반환한다.
서버는 `baseDate`에 요청 날짜가 아닌 실제 고시 기준일을 담는다.
미래 날짜의 환율을 요청하면 서버는 `400`으로 거절한다.
미래 날짜의 판정 기준은 [환율 정책](../policy/exchange-rate.md#지출-환율)을 따른다.
`KRW` 환율을 요청하면 서버는 `400`으로 거절한다. 원화 지출의 환율은 [EXP-03 지출 생성](expense.md#exp-03-지출-생성)의 규칙에 따라 `1.00`이다.
요청한 `paidDate`와 반환된 `baseDate`가 다르면 앱은 실제 기준일과 이전 고시값을 썼다는 안내를 표시한다.

요청 날짜와 그 이전 7일 안에 고시된 환율이 없으면 서버는 `404 EXCHANGE_RATE_NOT_PUBLISHED`를 반환한다.
서버는 빈 성공 응답이나 임의 환율을 반환하지 않는다.
서버는 외부 서비스의 장애나 타임아웃을 그 날짜의 고시값이 없는 상태로 취급하지 않는다.
외부 조회에 실패하면 서버는 `503 EXCHANGE_RATE_PROVIDER_UNAVAILABLE`을 반환한다.
`404`나 `503`을 받으면 앱은 사용자가 환율을 직접 입력하게 한다.
저장 구조와 외부 조회 순서는 [환율 설계서](../design/exchange-rate.md#결제일-기준-환율-조회)를 따른다.

## 공통 모델

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### PersonalRate

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `exchangeRateId` | String | 아니요 | 항상 | 본인이 등록한 개인 환율 기록 ID. |
| `currency` | String | 아니요 | 항상 | `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `rate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액, 0보다 큼. 정수부 10자리 이하인 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeMethod` | String | 아니요 | 항상 | `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `exchangeDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 환전 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 등록 시각. 최근 기록을 고르는 기준이다. |

### PersonalRateInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `currency` | String | 필수 | 없음 | `JPY`, `USD`, `EUR`, `CNY` 중 하나. `KRW`는 거절한다. |
| `rate` | String | 필수 | 없음 | 외화 1단위당 원화 금액, 0보다 크고 정수부 10자리 이하. 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeMethod` | String | 필수 | 없음 | `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `exchangeDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 환전 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |

### ReferenceRate

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `currency` | String | 아니요 | 항상 | `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `rate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액. 고시 기관의 환율을 소수점 셋째 자리에서 반올림한 소수점 이하 2자리 문자열로 전달한다. |
| `provider` | String | 아니요 | 항상 | 환율을 고시한 기관. `ECB`는 유럽중앙은행이다. |
| `baseDate` | String | 아니요 | 항상 | YYYY-MM-DD. 환율의 실제 기준일. 요청 결제일과 다를 수 있다. |

## 오류 코드

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Billit-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `EXCHANGE_RATE_NOT_PUBLISHED` | 결제일과 그 이전 7일 안에 고시된 환율이 없음 |
| 409 | `EXCHANGE_RATE_DUPLICATED` | 같은 여행에 본인이 통화, 환전 날짜, 환전 수단과 환율이 모두 같은 기록을 이미 등록함 |
| 503 | `EXCHANGE_RATE_PROVIDER_UNAVAILABLE` | 외부 환율을 일시적으로 조회할 수 없음 |
